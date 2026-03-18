---
title: "Security Filter 던진 예외가 왜 안잡힐까"
date: 2026-03-02 19:09:21 +0900
categories: [blog-project, 트러블슈팅]
image:
  path: /assets/img/thumbnails/공부가나를때렸어.png
---

## 배경

- Spring Boot + Spring Security 환경에서 JWT를 구현
- 잘못된 JWT 토큰이 들어왔을 때 사용자에게 명확한 에러 메세지를 내려주고 싶었다.

처음에는 단순히 @RestControllerAdvice 기반의 GlobalExceptionController로 해결할 수 있을거라 생각했다. 하지만 결과는 예상과 달랐다.

## 첫 번째 시도 방법: GlobalExceptionHandler 구현

**GlobalExceptionHandler 구현**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
	@ResponseStatus(HttpStatus.UNAUTHORIZED)
	@ExceptionHandler(JwtAuthenticationException.class)
	    public ErrorResponseDto handleJwtAuthenticationException() {
	        return new ErrorResponseDto(AUTHENTICATION_FAILED);
	    }
}
```

**의도:**

- JwtAuthenticationFilter에서 예외 발생 → 전역 예외 핸들러에서 처리 → 클라이언트에게 에러 응답을 전달

**결과:**

- JwtAuthenticationFilter 에서 예외는 발생했지만, GlobalExceptionHandler가 전혀 잡지 못함
- 클라이언트는 단순히 401 Unauthorized 상태 코드만 받고, 커스텀 응답 메시지는 전달되지 않음.

**원인 분석**

- Spring Security의 Filter Chain은 DispatcherServlet 보다 먼저 동작한다.
- @RestControllerAdvice는 Controller 단에서 발생한 예외만 처리 가능하다.
- 따라서 Filter에서 발생한 예외는 GlobalExceptionHandler로는 처리할 수 없다.

## 두 번째 시도 방법: AuthenticationEntryPoint 구현

Spring Security 공식 문서를 참고하니, 인증/인가 과정에서 발생한 예외는 ExceptionTranslationFilter가 가로채고, 최종적으로 `AuthenticationEntryPoint`가 처리한다는 점을 알게 되었다.

따라서 AuthenticationEntryPoint 를 커스터마이징하면, 401 에러 상황에서 클라이언트에게 JSON 형태의 응답을 내려줄 수 있을 것이라 판단했다.

![Spring Security 공식문서 예외 처리 방법](/assets/img/post/image11.png)

Spring Security 공식문서 예외 처리 방법

**CustomAuthenticationEntryPoint 구현**

```java
@Slf4j
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {

        AuthErrorType errorType = (AuthErrorType) request.getAttribute("authErrorType");

        if(errorType == null){
            errorType = AuthErrorType.AUTHENTICATION_FAILED;
        }

        String errorResponse = generateErrorResponse(errorType);

        response.setCharacterEncoding("UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.getWriter().write(errorResponse);
    }

    private String generateErrorResponse(AuthErrorType error){
        String response = String.format("""
                        {
                            "success": false,
                            "error": "UNAUTHORIZED",
                            "message": "%s",
                            "timestamp": "%s"
                        }
                        """,
                error.getMessage(),
                LocalDateTime.now()
        );
        return response;
    }
}

```

**의도:**

- JWT 필터에서 예외 발생 → ExceptionTranslationFilter 호출 → CustomAuthenticationEntryPoint에서 예외 처리 → 클라이언트에게 JSON 응답 반환

**결과:**

- CustomAuthenticationEntryPoint는 정상적으로 작동했으나, JWT 필터에서 발생한 예외 메시지를 그대로 전달하지는 못함
- 클라이언트는 내가 정의한 에러 포멧을 받았지만, 예외 원인은 내가 지정한 원인으로 받지 못함.

**원인 분석**

- ExceptionTranslationFilter는 내부적으로 예외를 AuthenticationException 혹은 AccessDeniedException으로 변환한 뒤 AuthenticationEntryPoint를 호출한다.
- 따라서 JwtAuthenticationFilter에서 던진 커스텀 예외 메시지는 AuthenticationEntryPoint까지 그대로 전달되지 않았다.

```java
// ExceptionTranslationFilter.java
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws IOException, ServletException {
    try {
        chain.doFilter(request, response);
    }
    catch (IOException ex) {
        throw ex;
    }
    catch (Exception ex) {
        // Try to extract a SpringSecurityException from the stacktrace
        Throwable[] causeChain = this.throwableAnalyzer.determineCauseChain(ex);
        // **👉** AuthenticationException으로 변환
        RuntimeException securityException = (AuthenticationException) this.throwableAnalyzer
                .getFirstThrowableOfType(AuthenticationException.class, causeChain);
        if (securityException == null) {
            // **👉** AccessDeniedException으로 변환
            securityException = (AccessDeniedException) this.throwableAnalyzer
                    .getFirstThrowableOfType(AccessDeniedException.class, causeChain);
        }
        if (securityException == null) {
            rethrow(ex);
        }
        if (response.isCommitted()) {
            throw new ServletException("Unable to handle the Spring Security Exception "
                    + "because the response is already committed.", ex);
        }
        handleSpringSecurityException(request, response, chain, securityException);
    }
}
```

## 최종 해결: JwtExceptionHandlerFilter

그래서 선택한 방법은 예외 처리 전용 `Filter`를 생성하는 것이다.

**흐름:**

```
JwtExceptionHandlerFilter → JwtFilter → 나머지 Security Filters …
```

- 흐름은 ExceptionTranslationFilter의 매커니즘과 동일하게 JwtExceptionHandlerFIlter에서 chain.doFilter()를 감싸고, JwtAuthenticationFilter에서 발생하는 예외를 캐치해서 JSON 응답으로 변환했다.

```java
@Slf4j
public class JwtExceptionHandlerFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try{
            log.info("JwtExceptionHandlerFilter 실행");
            filterChain.doFilter(request,response);
        } catch (MalformedJwtException e) {
            log.info("올바르지 않은 JWT 토큰 형식입니다.");
            handleJwtException(response, INVALID_TOKEN);
            return;
        }catch (ExpiredJwtException e){
            log.info("만료된 JWT 토큰입니다.");
            handleJwtException(response, EXPIRED_TOKEN);
            return;
        } catch (UnsupportedJwtException e) {
            log.info("지원하지 않는 JWT 형식입니다.");
            handleJwtException(response, UNSUPPORTED_TOKEN);
            return;
        } catch (IllegalArgumentException e) {
            log.info("JWT 토큰이 비어있거나 잘못되었습니다.");
            handleJwtException(response, ILLEGAL_TOKEN);
            return;
        } catch (SignatureException e) {
            log.info("JWT 서명이 유효하지 않습니다.");
            handleJwtException(response, SIGNATURE_MISMATCH);
            return;
        }
    }

    private void handleJwtException(HttpServletResponse response, JwtErrorType errorType) throws IOException {
        String errorCode = errorType.name();
        String message = errorType.getMessage();
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(String.format(
                """
                        {
                            "success": false,
                            "error": "%s",
                            "message": "%s",
                            "timestamp": "%s"
                        }
                        """, errorCode, message, LocalDateTime.now()));
    }

}

```

## 배운점

- Spring Security의 Filter Chain은 단순히 “앞에서 막는다” 수준이 아니라, **순서와 책임 분리**를 이해해야 제대로 활용할 수 있다.
- AuthenticationEntryPoint는 스프링에서 제공하는만큼 편하지만, 모든 상황에서 동작하는 것은 아니다. (특히 커스텀 필터 앞단에서는 적용되지 않음)
- 결국 내 프로젝트의 구조와 요구사항에 맞는 예외 처리 전략을 세우는 것이 중요하다.
