---
title: DTO, VO, Entity의 차이가 뭘까 ?
date: 2025-09-04 23:21:21 +0900
categories: [OOP, JAVA]
tags: [oop, dto, vo, entity]     # TAG 이름은 항상 소문자여야 합니다.
---

## 들어가며
현재 우리 회사에서는 DTO, VO, Entity의 구분 없이 하나의 DTO 객체로 DB와 매핑하여 사용하고 있다. <br>
정확히 말하면 DB 테이블과 완전한 1:1 매핑이 아니라, 여러 테이블의 데이터를 하나의 DTO에 합쳐서 담아 쓰고 있는 상황이다. <br>
즉, DB 작업에도 동일한 DTO를 사용하고, 사용자 화면에 데이터를 내려줄 때도 동일한 DTO를 그대로 사용하는 방식이다. <br>
이런 구조로 개발하다 보니 불편한 점이 많아졌다. 그래서 이번 기회에 많은 개발자들이 한 번쯤 들어본 DTO, VO, Entity의 차이와 그 필요성에 대해 정리해보려고 한다.

### 업무를 하면서 불편한점 😮‍💨 
1. 관심사 분리가 되지 않는다.
   - DB 저장용 객체와 화면 응답용 객체가 같으니, 한쪽의 변경이 다른 쪽에 영향을 준다.
   - 예를 들어 화면에서만 쓰는 필드를 추가했는데, DB 매퍼 코드까지 수정해야 하는 상황이 발생한다.
2. 불필요한 필드 노출된다.
   - 여러 테이블을 조합한 DTO를 쓰다 보니, 특정 API에서는 필요 없는 필드까지 항상 포함된다.
   - 반대로 필요한 필드가 없어서 `null` 처리하거나 임시 필드를 추가하는 꼼수가 생긴다.
3. 테스트 및 유지보수가 어렵다.
   - 객체가 너무 비대해서 어디서 어떤 필드가 쓰이는지 추적하기 어렵다.
   - 로그를 찍어도 필요 없는 데이터들의 값이 너무 많아 내가 정확히 필요한 데이터를 확인하기 어렵다.
4. 도메인 개념이 사라진다.
   - Entity가 단순히 DB 레코드를 담는 그릇이 아니라 도메인 규칙을 표현해야 하는데, DTO에 다 섞여 있으니 단순 데이터 컨테이너로만 쓰인다.
   - 결과적으로는 객체지향적 개발이 아니라 단순한 "테이블 -> JSON 변환" 코드만 남는다.
5. 재사용성이 저하된다.
   - 한 객체에 여러 의미가 섞이다 보니, 기능마다 다른 의미로 필드를 사용하거나 중복 정의가 늘어난다.


## DTO, VO, Entity의 차이
### Entity
- 정의: 데이터베이스 테이블과 매핑되는 객체
- 역할: 비즈니스 도메인 규칙을 담고 있으며, 데이터 자체를 표현하는 핵심 모델
- 특징:
  - 보통 JPA 같은 ORM 환경에서 `@Entity`로 선언해 사용한다.
  - 단순히 데이터를 담는 그릇이 아니라, 해당 도메인의 불변 조건과 행위를 함께 표현할 수 있다.
  - DB 스키마 변경 시 Entity도 영향을 받는다.

> 💡 예시
> 
> `User` 테이블 과 매핑되는 `UserEntity` 클래스가 있다고 하면,
> 
> `id`, `username`, `password` 같은 필드와 함께 `changePassword()` 같은 도메인 로직이 들어갈 수 있다.

### VO (Value Object)
- 정의: 값 그 자체를 표현하는 객체
- 역할: 동일성(identity)이 아니라 값의 동등성(equality)으로 비교된다.
- 특징:
  - 대표적인 예: `Money`, `Email`, `Address`
  - 값만 같으면 동일한 객체로 간주되며, 불변하게 설계하는 것이 일반적이다.
  - DB와 직접 매핑되기보다는, 모데인 모델 내부에서 의미 있는 값을 표현하는 데 사용된다.

> 💡 예시
>
> `Money(1000, "KRW")`와 `Money(1000, "KRW")`는 같은 값 객체로 취급된다.
>
> `UserEntity`가 `Email` VO를 필드로 가질 수 있다.

### DTO (Data Transfer Object)
- 정의: 계층 간 데이터 전달을 위한 객체
- 역할: 주로 Controller <-> Service <-> View(API 응답) 사이에서 사용된다.
- 특징:
  - 로직이 없고 순수하게 데이터만 남는다.
  - API 요청(Request DTO), API 응답(Response DTO) 등으로 나뉘어 쓰는 경우가 많다.
  - Entity를 그대로 노출하면 보안/의존성 문제가 생기므르ㅗ, 외부 노출 전환용으로 반드시 필요하다.
> 💡 예시
>
> 클라이언트에서 회원가입을 요청할 때 `UserRequestDto(username, password, email)` 같은 DTO를 사용하고,
>
> 응답 시에는 `UesrResponseDto(id, username, email)` 같은 DTO로 내려준다.

## DTO 변환 책임은 어디에 두는게 맞을까 ?
위에서 알아봤듯이 DTO는 계층 간 데이터 전달을 위한 객체이다. 그럼 결국 DB에서 받은 Entity -> DTO로 변환해야되는 과정을 거치게 된다.<br>
그럼 이 변환 과정을 어디에서 해야할까?

### Controller에서 변환하는 방식
```java
@PostMapping
public ResponseEntity<UserResponseDto> createUser(@RequestBody UserRequestDto requset){
  UserEntity saved = userService.createUser(request);
  return new ResponseEntity.ok(new UserResponseDto(saved.getId(), saved,getUsername(), saved.getEmail()));
}
```

#### 장점
- 단순해보인다.
- 변환 책임이 분명히 Controller 계층에 있다.

#### 단점
- Controller 코드가 지저분해짐 -> 필드가 많을수록 가독성 저하
- 여러 곳에서 동일 변환을 반복하게 되어 중복 코드 증가
- 테스트/유지보수 어려움

`Controller`는 요청/응답만 담당하고, 변환 책임은 다른 쪽으로 위임하는게 일반적이다.

#### Service에서 반환하는 방식

```java

@Service
@RequiredArgsConstructor
public class UserService {
  private final UserRepository userRepository;

  public UserResponseDto createUser(UserRequestDto request) {
    UserEntity user = new UserEntity(request.getUsernmae(), request.getEmail());
    UserEntity saved = userRepository.save(user);
    
    return new UserResponseDto(saved.getId(), saved.getUsername(), saved.getEmail());
  }
}
```
#### 장점
- Controller가 가벼워진다.
- Service 계층에서 Entity -> DTO 변환을 관리하므로 한 곳에 모인다.

#### 단점
- Service 코드가 비즈니스 로직 + 변환 로직 섞여서 복잡해진다.
- Service 메서드가 점점 DTO 변환기로 변질될 수 있다.

중소규모 프로젝트에서는 흔히 쓰이나, 서비스가 점점 비대해지는 단점이 있다.

#### DTO 클래스 내부에서 변환 (정적 팩토리 메서드)
```java
@Getter
@AllArgsConstructor
public class UserResponseDto{
  private Long id;
  private String username;
  private String email;
  
  public static UserResponseDto fromEntity(UserEntity user){
    return new UserResponseDto(user.getId(), user.getUsername(), user.getEmail());
  }
}
```
```java
return UserResponseDto.fromEntity(saved);
```
#### 장점
- 변환 책임이 DTO 내부로 모여 응집도가 높아진다.
- Service/Controller 코드가 깔끔해진다.
- DTO 변환 로직이 여러 군데 흩어지지 않는다.

#### 단점
- DTO가 Entity를 직접 참조하게 됨 -> 의존성 방향(Entity -> DTO) 역전
- 순수한 데이터 컨테이너 역할을 벗어나 로직이 섞이게 됨

## 정리
- Entity: DB와 매핑되는 도메인 객체 (도메인 규칙 + 상태 보존)
- VO: 값 자체를 의미하는 객체 (불변, 값 동등성)
- DTO: 계층 간 데이터 전달용 객체 (외부 노출, 요청/응답)

이렇게 역할을 나누면, 앞서 말한 문제점들이 자연스럽게 해결된다.
- DB 변경이 화면 DTO에 직접적으로 영향을 주지 않는다.
- 불필요한 필드가 섞이지 않고, API 마다 필요한 DTO만 정의할 수 있다.
- 도메인 규칙은 Entity/VO 안에 담을 수 있어 객체지향적으로 개발할 수 있다.
