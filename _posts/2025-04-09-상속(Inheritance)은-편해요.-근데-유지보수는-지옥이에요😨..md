---
title: 상속? 모르고 쓰면 지옥을 맛봅니다..😨
date: 2025-04-09 23:04:21 +0900
categories: [JAVA]
tags: [java]     # TAG 이름은 항상 소문자여야 합니다.
---

## 들어가며
처음 자바를 배울 때는 상속이 거의 필수처럼 느껴졌고, 지저분했던 코드를 상속을 활용해 깔끔하게 정리했을 때는 정말 뿌듯하고 신기했다.
그러나 실무에서 몇 번 상속을 이용해보고 유지보수가 너무 불편한데..? 대충 보면 상속을 사용하는게 맞는 거 같은데 왜 이렇게 유지보수가 불편하지?
라는 궁금점이 생겼다. 그래서 
이번 포스팅에서 자바에서 상속이 가지는 장점과 단점을 확인하고 어떤 대안이 있는지 확인해보자 😛

---

## 상속의 장점 👍
상속은 아래와 같은 장점을 가지고 있다.
1. 코드 재사용성
   - 공통 기능을 부모 클래스에 두고, 자식 클래스에서 재활용할 수 있다.
   - 반복되는 코드를 줄이고, 깔끔한 구조처럼 보여준다.
2. 구조적으로 이해하기 쉬움
   - "A는 B다"라는 is-a 관계를 직관적으로 표현할 수 있다.
3. 다형성 활용
   - 부모 타입으로 자식 객체를 다룰 수 있어, 유연한 코드 구성이 가능하다.

상속의 장점을 보면, 당연히 상속을 사용하는 게 맞다고 느껴진다.<br>
하지만 실제로는 신중하게 고려해야 할 단점들도 존재한다. 이제 그 단점들을 알아보자!

---

## 상속의 단점 👎
상속은 아래와 같은 단점을 가지고 있다.

#### 1. 캡슐화를 깨뜨린다.

> 캡슐화 : 객체의 내부 구현을 외부로부터 숨기고, 필요한 인터페이스만 공개하여 변경에 유연하게 대응할 수 있게 하는 원칙

상속은 자식 클래스가 부모 클래스의 기능을 재사용하게 해주지만, <span style='color: orange'>**부모 클래스의 내부 구현에 자식 클래스가 의존하기 때문에 캡슐화를 깨뜨릴 수 있는 위험**</span>이 있다.

`ArrayList`를 상속하여 리스트에 추가된 전체 원소의 수를 세는 `CountingList` 클래스를 만들고, 이로 인해 캡슐화가 어떻게 깨지는지를 살펴보자.

```java
# CountingHashSet 클래스 정의
public class CountingHashSet<E> extends HashSet<E> {

    private int addCount = 0;

    public CountingHashSet(){

    }

    public CountingHashSet(int initCap, float loadFactor){
        super(initCap,loadFactor);
    }

    @Override
    public boolean add(E e){
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c){
        addCount += c.size();
        return super.addAll(c);
    }
    
    public int getAddCount(){
        return addCount;
    }

}

```

```java
# CountingMain 클래스 정의
public class CountingMain {

    public static void main(String[] args) {
        CountingHashSet<String> countingHashSet = new CountingHashSet<>();
        countingHashSet.addAll(List.of("a","b","c"));

        // 카운트 : 6
        System.out.println("카운트 : " + countingHashSet.getAddCount());
    }
}

```
`System.out.println("카운트 : " + countingHashSet.getAddCount());`의 결과가 6이 나온다.<br>
왜 이런 결과가 나오는 걸까? 🧐<br><br>

자바의 `HashSet` 클래스 내부에서 `addAll()`을 구현할 때, 각 요소를 반복적으로 `add()`를 호출하는 방식으로 처리한다.

```java
# HashSet 클래스의 addAll 함수
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```

여기서 중요한 점은 `add(e)`라는 호출이 런타임 시점에서 오버라이딩된 `CountingList`의 `add()` 메서드를 실행한다는 것이다. 이러한 동작을 **자기 참조(self-reference)**라고 부른다.
부모 클래스 내부에서 자기 자신을 참조하는 코드가 있을 경우, 그 객체가 자식 클래스의 인스턴스라면 오버라이드 된 메서드가 호출된다.

이처럼 상속을 사용할 경우, <span style='color: orange'>**부모 클래스 내부에서 어떤 메서드를 어떻게 호출하는지 자식 클래스가 전부 알아야 제대로 동작**</span>한다. 이는 캡슐화가 완전히 깨졌다는 것을 의미한다.

> **자기 참조에 대해 더 궁금하면 "망나니 개발자"님의 블로그로 공부해보자!** 👉 ["망나니 개발자" 블로그](https://mangkyu.tistory.com/193)


#### 2. 강한 결합으로 인한 유연성 저하
상속은 자식 클래스가 부모 클래스의 모든 필드와 메서드를 물려받는 구조이기 때문에, 자식 클래스는 부모 클래스에 강하게 결합된다. 이 말은 자식 클래스의 동작이 부모 클래스의 구현에 크게 의존한다는 뜻이 된다.
문제는 부모 클래스의 코드가 변경되었을 때 들어난다.  <span style='color: orange'>**부모 클래스의 내부 구현이나 메서드의 시그니처가 조금이라도 바뀌면, 이를 상속하고 있는 모든 자식 클래스들이 영향을 받아 코드 수정이 필요하다.**</span>
자식 클래스가 많을수록 이러한 변경의 파급 효과는 커지고, 유지보수는 더욱 어려워진다.

#### 3. 필드와 메서드 찾기가 불편하다.
상속 구조에서는 자식 클래스가 부모 클래스의 필드와 메서드를 모두 물려받기 때문에, 실제조 어떤 기능이 어디에 정의되어 있는지를 파악하기가 어렵다.
예를 들어, 어떤 메서드가 자식 클래스에서 정의된 것인지, 부모 클래스에서 정의된 것인지, 또는 오버라이딩 된 것인지 추적하려면 클래스 계층을 계속 따라가야 한다.
이러한 구조는 코드의 가독성과 이해도를 떨어뜨리고 상속 관계가 복잡한 경우 디버깅과 유지보수를 어렵게 만든다.

#### 4. 상속은 결함도 함께 상속한다.
상속은 부모 클래스의 기능을 재사용할 수 있다는 장점이 있지만, 만약 부모 클래스에 존재하는 결함이나 설계상의 문제점까지 자식 클래스가 그대로 물려받게 된다.
부모 클래스에 숨겨진 버그가 있는 경우, 이를 상속받은 자식 클래스에도 동일한 문제가 발생할 수 있으며, 그 원인을 추적하기 어려워진다.

---

## 그럼 상속은 언제 써 🤮
물론 상속도 쓰면 좋은 경우가 있다. 어떤 경우에 상속을 쓰면 좋을지 알아보자.

#### 공통된 흐름을 강제하고 싶을 때 (비즈니스 로직의 강제 적용)
예를 들어, 결제 서비스를 생각해보자.<br>
결제 플로우는 다음과 같은 흐름으로 이루어있다고 했을 때
- 결제 정보 확인
- 결제 실행
- 결제 결과 로그남기기

이 중에서 **결제 정보 확인**과 **로그 남기기**는 모든 결제 방식에 대해 동일하게 처리할 수 있는 부분이다. 하지만 **결제 실행** 단계는 결제 방식(카카오, 삼성, 네이버 등)에 따라 서로 다르게 구현되어야 한다.
이처럼 전체적인 처리 순서는 동일하지만, **특정 단계만 서브 클래스에서 다르게 구현되길 원한다면,** 이럴 떄 추상 클래스를 상속 받는게 좋다.
예를 들어 `PaymentService`라는 추상 클래스를 만들어서 전체 플로우를 고정하고, 서브 클래스에서는 `executePayment()`만 오버라딩하게 강제하는 식으로 처리하면 된다.

아래의 예제 코드를 보면 수월하게 이해될것이다. 중요한건 단순 코드보다 어떤 상황에서 쓰면 좋을지를 생각해보자.

```java
# 공통 로직을 추상 클래스로 구현
public abstract class CardPaymentService {

    // 결제 흐름을 고정
    public final void processPayment() {
        checkPaymentInfo();      // 공통
        executePayment();        // 추상 - 구현은 자식 클래스에 위임
        logPaymentResult();      // 공통
    }

    private void checkPaymentInfo() {
        System.out.println("결제 정보 확인 중...");
    }

    protected abstract void executePayment();  // 서브 클래스에서 구현

    private void logPaymentResult() {
        System.out.println("결제 결과 로그 남기기");
    }
}
```

```java
# 각각의 결제 방식 구현
public class KakaoCardPayment extends CardPaymentService {
    @Override
    protected void executePayment() {
        System.out.println("카카오 카드 결제 실행");
    }
}

public class SamsungCardPayment extends CardPaymentService {
    @Override
    protected void executePayment() {
        System.out.println("삼성 카드 결제 실행");
    }
}
```

```java
# 실행 예시
public class Main {
    public static void main(String[] args) {
        CardPaymentService kakao = new KakaoCardPayment();
        kakao.processPayment();

        System.out.println();

        CardPaymentService samsung = new SamsungCardPayment();
        samsung.processPayment();
    }
}
```

---

## 그럼 이대로 코드를 지저분하게 써야될까 ❓
상속의 단점이나 한계를 느꼈다면, <span style='color: orange'>대안으로 **조합(Compostion)**을 고민해볼 수 있다.</span>

### 조합(Compostion)
<span style='color: orange'>**필요한 기능을 다른 객체에게 맡기고 조합해서 사용하는 방식**</span>을 의미한다. 쉽게 말해, 상속 대신 <span style='color: orange'>**클래스 안에 다른 객체를 포함시켜서 기능을 사용하는 구조**</span>이다.
이번에 조합을 사용하여 코드를 작성해보자. <br>예제는 위의 추상 클래스를 사용하여 작성한 예제와 같으니 비교해서 봐보자.

```java
# 인터페이스: 개별 결제 방식 정의
public interface PaymentStrategy {
    void executePayment();
}
```

```java
# 결제 전략 1: 카카오
public class KakaoPaymentStrategy implements PaymentStrategy {
  @Override
  public void executePayment() {
    System.out.println("카카오 카드 결제 실행");
  }
}

# 결제 전략 2: 삼성
public class SamsungPaymentStrategy implements PaymentStrategy {
  @Override
  public void executePayment() {
    System.out.println("삼성 카드 결제 실행");
  }
}
```

```java
# 공통 결제 흐름: 결제 전략을 받아서 결제 실행
public class CardPaymentService {

    private final PaymentStrategy strategy;

    public CardPaymentService(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void processPayment() {
        checkPaymentInfo();        // 공통
        strategy.executePayment(); // 전략에 위임
        logPaymentResult();        // 공통
    }

    private void checkPaymentInfo() {
        System.out.println("결제 정보 확인 중...");
    }

    private void logPaymentResult() {
        System.out.println("결제 결과 로그 남기기");
    }
```

```java
# 실행 예시
public class Main {
    public static void main(String[] args) {
        PaymentStrategy kakaoStrategy = new KakaoPaymentStrategy();
        CardPaymentService kakaoService = new CardPaymentService(kakaoStrategy);
        kakaoService.processPayment();

        System.out.println();

        PaymentStrategy samsungStrategy = new SamsungPaymentStrategy();
        CardPaymentService samsungService = new CardPaymentService(samsungStrategy);
        samsungService.processPayment();
    }
}
```

---

## 조합의 장점 👍
### 유연성 (Flexibility)
- 한 번 선언한 클래스를 다양한 전략과 조합할 수 있어 재사용성이 높아짐
- 런타임 시 동적으로 전략 교체도 가능
- 상속은 컴파일 타임에 결정되는 반면, 조합은 실행 시 유연하게 대응 가능
예시:

```java
// 전략 바꾸기
paymentService.setStrategy(new AnotherStrategy());
```

### 낮은 결합도 (Low Coupling)
- 구현보다는 인터페이스에 의존
- 내부 구현 변경 시 다른 클래스에 미치는 영향이 적음 → 유지보수 쉬움

### 재사용성 향상
- 다양한 클래스에서 동일한 기능(구성 요소)을 중복 없이 공유 가능
- has-a 관계이므로 구조가 더 자유롭고, 유연하게 조립 가능

### 단일 책임 원칙(SRP) 준수에 용이
- 기능 단위를 객체로 분리할 수 있어서 각 클래스가 한 가지 일만 하도록 설계 가능
- 유지보수, 테스트, 확장이 더 쉬워짐

### 테스트 용이성
- 외부 객체(mock)를 주입하여 단위 테스트 작성이 쉬움
- 상속 구조보다 테스트 격리가 쉬움

> "상속은 설계 당시엔 편할 수 있지만, 조합은 변화에 강한 구조를 만든다."

---


## 조합의 단점 👎
### 클래스의 수가 많아진다.
- 기능 단위로 클래스를 쪼개고 조합하다 보면 구성 요소가 많아진다.
- 작은 프로젝트에서는 오히려 과도한 분리가 복잡합을 유발할 수 있다.

### 간접적인 관계로 인해 코드 흐름이 불분명할 수 있다.
- 상속은 "위에서 아래로" 흐름이 보이지만, 조합은 "어떤 객체가 어떤 기능을 포함하고 있는지" 파악하기 어려울 때가 있다.
- 특히 새로 보는 코드에서는 `new PaymentService(new KakaoStrategy())` 이런 식의 구조 추적이 번거로울 수 있다.

### 런타임 오류 가능성
- 상속은 컴파일 타임에 문법 오류가 드러난다.
  - (예: 부모 메서드 오버라이드 누락 시 컴파일 오류)
- 반면 조합은 구조가 더 유연한 만큼, 런타임까지 실행해봐야 오류가 들어난다.

```java
CardPaymentService service = new CardPaymentService(null);
service.processPayment(); // -> NullPointerException (런타임 오류 발생)
```

> 특히 코드 흐름을 처음 접하는 사람이 객체를 잘못 조립하거나, 필드를 빠뜨려도 컴파일러는 알려주지 않는다.

---

## 마무리하며 🧘
이 글을 준비하면서 해외 포스팅, 국내 포스팅 등 여러 자료를 찾아봤는데, 대부분은 "상속을 쓰지마라", "무조건 조합을 써라" 같은 의견이 많았다. 
> 물론, 요즘은 조합만을 이용하여 유연하게 가져가는게 트렌드이니까,, 


<br>
하지만 개인적으로 공부하면서 느낀건, 무조건 상속을 배제하는 것보다 <br>
👉 상속이 가진 장점과 단점, 주의할점을 제대로 알고 <br>
👉 어떤 상황에서 써도 되는지를 판단할 수 있는 기준이 더 중요하다 생각이 들었다.

결국 중요한건 **상속 vs 조합,** 뭐가 무조건 좋다기 보단 상황에 따라 트레이드오프를 이해하고 선택하는게 맞는거 같다.

