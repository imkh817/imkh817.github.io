---
title: JAVA 참조(Reference) 종류
date: 2025-04-06 23:04:21 +0900
categories: [JAVA]
tags: [java]     # TAG 이름은 항상 소문자여야 합니다.
---

# JAVA 참조(Reference) 종류 정리
Java는 메모리 관리를 자동으로 해주는 언어지만, 개발자가 GC(Garbage Collector)와의 관계를 좀 더 세밀하게 제어하고 싶을 때, **참조(Reference)** 타입을 적절히 선택하느 것이 중요하다.
이번 글에서는 Java에서 제공하는 네 가지 참조 타입에 대해 알아보고 각각의 사용 예시를 통해 이해해보려고한다.

---

## 1. Strong Reference (강한 참조)
### 설명
자바에서 기본적으로 사용하는 참조 방식이다. 평소에 사용하는 대부분의 객체 참조가 이에 해당한다.<br>
GC는 강한 참조가 존재하는 객체는 절대로 수거하지 않는다.

### 💡 예시
```java
String name = "서울";
Object obj = new Object(); // 강한 참조
```

### 특징 요약
- 기본 참조 방식
- GC가 절대 수거하지 않음(null 처리 or 범위 밖으로 나가야 수거 가능)
- 명확하고 안정적인 객체 관리 기능

---

## 2. Soft Reference
### 설명
**`Soft Reference` 객체는 메모리가 부족할 때 가비지 컬렉터(GC)의 판단에 따라 제거될 수 있는 객체이다.**<br>
**`Soft Reference는`** 주로 **메모리에 민감한 캐시를 구현할때 사용된다.**

### 사전 개념
**`softly reachable` 상태란?**
- 어떤 객체가 `softly reachable` 하다는 건, **그 객체가 하나 이상의 `SoftReference`를 통해 접근 가능하지만, 다른 강한(Strong) 참조가 없는 상태를 의미한다.**
- 즉, 이런 상태라고 할 수 있다.
  - ✔️ **SoftReference가 참조하고 있음**
  - ❌ **Strong Reference(일반적인 변수나 필드)로는 참조되지 않음**
  - ❌ **WeakReference나 PhantomReference도 아님**

**코드로 Softly reachable 상태를 이해해보기**

```java
import java.lang.ref.SoftReference;

String strong = new String("Hello");
SoftReference<String> softRef = new SoftReference<>(strong);

strong = null; // 이제 Strong Reference는 사라짐
```

- 이 시점에서 `"Hello"` 객체는 이제 `softly reachable` 상태가 된다.
- 메모리에 여유가 있으면 -> GC는 이 객체를 그대로 유지
- 메모리가 부족해지면 -> GC는 이 객체를 제거


### 특징
- 어떤 시점에 가비지 컬렉터가 특정 객체를 `soft reachable` 상태라고 판단하면, 그 시점에 해당 객체에 대한 모든 `soft reference`와, 그 객체로부터 강한 참조 체인을 따라
도달할 수 있는 다른 `softly reachable` 객체들의 `soft reference`와, 그 객체로부터 강한 참조 체인을 따라 도달할 수 있는 다른 `softly reachalbe` 객체들의
soft reference들도 한번에 제거할 수 있다. 이와 동시에, 또는 나중에, 참조 큐(reference queue)에 등록되어 있는 해당 `soft reference`들은 큐에 추가된다.

**1번 특징은 글로만 봐서는 이해하기가 힘들다. 이것도 코드로 확인해보자**

```java
class BigObject {
    private final String name;
    private final List<BigObject> children = new ArrayList<>();

    public BigObject(String name) {
        this.name = name;
    }

    public void addChild(BigObject child) {
        children.add(child);
    }

    public String getName() {
        return name;
    }
}
```
```java
public class SoftReferenceChainExample {
    public static void main(String[] args) {
        // Step 1: 객체 생성 및 연결
        BigObject a = new BigObject("A");
        BigObject b = new BigObject("B");
        BigObject c = new BigObject("C");

        // A -> B -> C (강한 참조 체인)
        a.addChild(b);
        b.addChild(c);

        // Step 2: SoftReference로 각각 감싸기
        SoftReference<BigObject> refA = new SoftReference<>(a);
        SoftReference<BigObject> refB = new SoftReference<>(b);
        SoftReference<BigObject> refC = new SoftReference<>(c);

        // Step 3: Strong Reference 제거
        a = null;
        b = null;
        c = null;

        // Step 4: 메모리 부족 유도
        try {
            List<byte[]> stress = new ArrayList<>();
            for (int i = 0; i < 100; i++) {
                stress.add(new byte[10 * 1024 * 10240]); // 10GB 유도
            }
        } catch (OutOfMemoryError e) {
            System.out.println("💥 메모리 부족 발생!");
        }

        // Step 5: SoftReference 상태 확인
        System.out.println("refA: " + (refA.get() != null ? "살아있음" : "GC됨"));
        System.out.println("refB: " + (refB.get() != null ? "살아있음" : "GC됨"));
        System.out.println("refC: " + (refC.get() != null ? "살아있음" : "GC됨"));
    }
}
/**
 * 실행 결과
 * 
 * 💥 메모리 부족 발생!
 * refA: GC됨
 * refB: GC됨
 * refC: GC됨
 */

```
- **`softly reachable` 객체에 대한 모든 `soft reference`는, 가상 머신이 `OutOfMemoryError`를 던지기 전에 반드시 제거된다.**
- **`soft reference`는 참조 대상 객체(reference)가 강하게 참조되고 있다면 (실제로 사용중이라면), 해당 `soft reference`는 제거되지 않는다.**

**3번 특징도 코드로 알아보자.**
```java
public class Example {

    private Map<String, SoftReference<byte[]>> cache = new HashMap<>();

    // Cache 저장
    public void putCache(String key, byte[] data){
        cache.put(key,new SoftReference<>(data));
    }

    // Cache 값 조회
    public byte[] getCache(String key){
        SoftReference<byte[]> ref = cache.get(key);

        if(ref != null){
            byte[] value = ref.get();

            if(value != null){
                System.out.println("value = " + key);
                return value;
            }else{
                System.out.println("SoftReference가 GC에 의해 제거됨 : " + key);
                cache.remove(key);
            }

        }else{
            System.out.println("캐시에 저장되어 있지 않음");
        }
        return null;
    }
}
```
```java
public class ExampleMain {

    public static void main(String[] args) {

        // soft reference는 참조 대상 객체(reference)가 강하게 참조되고 있다면 (실제로 사용중이라면), 해당 soft reference는 제거되지 않는다.

        Example example = new Example();

        // 데이터 준비
        byte[] strongRef = new byte[10 * 1024 * 1024]; // 10MB
        example.putCache("image", strongRef);

        System.out.println("strong reference 유지한 상태에서 getCache 호출");
        example.getCache("image");

        try{
            byte[][] memoryPressure = new byte[100][];

            for(int i=0; i<memoryPressure.length; i++){
                memoryPressure[i] = new byte[10 * 1024 * 10240];
            }
        }catch (OutOfMemoryError e){
            System.out.println("OutOfMemoryError 발생하여도, 캐시에 image는 남아있어야 한다.");
        }

        System.out.println("다시 캐시 조회");
        example.getCache("image");

        System.out.println("strong reference 제거");
        // string reference 제거
        strongRef = null;

        // GC 유도
        System.out.println("strong reference 제거 후, 메모리 부족 유도");
        try{
            byte[][] memoryPressure = new byte[100][];

            for(int i=0; i<memoryPressure.length; i++){
                memoryPressure[i] = new byte[10 * 1024 * 10240];
            }
        }catch (OutOfMemoryError e){
            System.out.println("OutOfMemoryError 발생 (GC 작동 예상)");
        }

        // 다시 캐시 조회
        System.out.println("다시 캐시 조회");
        example.getCache("image");
    }

  /**
   * 실행 결과
   * 
   * strong reference 유지한 상태에서 getCache 호출
   * value = image
   * OutOfMemoryError 발생하여도, 캐시에 image는 남아있어야 한다.
   * 다시 캐시 조회
   * value = image
   * strong reference 제거
   * strong reference 제거 후, 메모리 부족 유도
   * OutOfMemoryError 발생 (GC 작동 예상)
   * 다시 캐시 조회
   * SoftReference가 GC에 의해 제거됨 : image
   */
}

```

---

## 그럼 실무에서는 SoftReference를 어떻게 언제 사용할까?

### 📌 핵심 포인트
> 다시 만들어도 되는 값 + 메모리를 아끼고 싶을 때

### 1. 이미지나 파일 같은 큰 데이터를 캐시할 때
- 서버나 앱에서 큰 이미지를 자주 보여줘야 되는 상황
- 매번 디스크나 네트워크에서 불러오면 느림 -> 캐시에 저장하고싶은데?
- 그런데 이미지가 너무 많으면 메모리가 부족해져
**해결**

```java
import java.lang.ref.SoftReference;

Map<string, SoftReference<Image>> imageCache;
```
- 👉 메모리 충분하면 캐시 유지
- 👉 메모리 부족하면 GC가 알아서 제거
- 👉 다시 필요하면 디스크에서 다시 로딩

### 2. 복잡한 연산 결과를 캐시할 때
- 어떤 계산 결과나 DB 쿼리 결과를 캐시에 저장하고 싶어
- 하지만 메모리 부족할땐 굳이 유지하지 않아도돼
**예시**
```java
SoftReference<Result> resultCache = new SoftReference<>(heavyCalculation());
```
- 👉 필요할 때 `resultCache.get()`으로 가져옴
- 👉 GC에 의해 제거되었으면, 다시 계산해서 넣음

### 실무에서 SoftReferene 쓸 떄 주의점
- 중요한 데이터에 사용 ❌ -> GC가 언제 제거할지 모름
- 실시간 성능이 중요한 캐시 -> GC로 제거되면 다시 로딩 필요 -> 느릴 수 있음
- 다시 계산/불러올 수 있는 데이터 -> 이 때는 아주 적합 ⭕

---

## 3. Weak Reference
### 간단 설명
`Weak Reference`란 GC가 객체를 회수할 수 있는 상태를 더 빨리 만들어주는 참조 방식이다.
즉, 객체가 `WeakReference`로만 참조되고 있다면, **GC는 메모리가 부족하지 않더라도 즉시 해당 객체를 수거할 수 있다.**

```java
WeakReference<MyObject> weakRef = new WeakReference<>(new MyObject());
```
이렇게 만들면 `MyObject`는 강한 참조가 없을 경우 곧바로 GC의 대상이 된다.

### 공식 문서로 이해하기
> 약한 참조(Weak reference) 객체는 그 참조 대상(referent)이 파이널라이즈 가능(finalizable) 상태가 되고, 파이널라이즈(finalized)된 후, 최종적으로 회수(reclaimed)되는 것을 막지 않습니다. 약한 참조는 주로 **정규화 매핑(canonicalizing mappings)** 을 구현하는 데 가장 자주 사용됩니다.<br><br>
> 가비지 컬렉터(garbage collector)가 특정 시점에 어떤 객체가 **약하게 도달 가능(weakly reachable)** 하다고 판단한다고 가정해 봅시다. 그 시점에 가비지 컬렉터는 다음 참조들을 원자적으로(atomically) 모두 **해제(clear)** 할 것입니다:<br><br>
> - 해당 객체에 대한 모든 약한 참조.
> - 강한 참조(strong reference)와 소프트 참조(soft reference)의 연쇄(chain)를 통해 해당 객체로부터 도달 가능한, 다른 모든 약하게 도달 가능한 객체들에 대한 모든 약한 참조. <br><br>
> 동시에, 이전에 약하게 도달 가능했던 모든 객체들을 파이널라이즈 가능(finalizable) 상태로 선언할 것입니다. 동시에 또는 그 이후의 어떤 시점에, **참조 큐(reference queue)** 에 등록된 약한 참조들 중에서 새롭게 해제된 것들을 해당 큐에 넣을(enqueue) 것입니다.

#### WeakReference 객체는 그 참조 대상이 finalizable 상태가 되고, finalized 된후, 최종적으로 회수 되는 것을 막지 않는다 ❓
- GC로 인해 메모리에서 제거되기 전에 해당 클래스의 `finalize()`라는 함수가 실행되고 GC가 메모리를 회수한다는 의미이다.
- `finalize()` 메서드는 객체에 대한 더 이상 유효한 참조가 없다고 판단되어 GC가 수행되지 직전에 GC에 의해 자동적으로 호출하는 메서드이다.
- 하지만, `finalize()`는 java 9부터 Deprecated 되었고, `phantomReference`나 다른 방법을 사용해야 한다.

#### 약한 참조는 정규화 매핑을 구현하는데 자주 사용된다 ❓
- 정규화 매핑 = 같은 데이터를 나타내는 객체를 여러 개 만들지 않고, 하나만 만들어서 사용하는 것
- 예제로 알아보자
  - 예제 상황 : 문자열 `"hello"`를 담고 있는 객체가 여러 개 생성될 수 있다고 가정
  
  ```java
    String str1 = new String("hello");
    String str2 = new String("hello");
    System.out.println(str1 == str2); // false (주소 다름)
  ```
  
    두 객체는 내용은 같지만 서로 다른 인스턴스이다. 이런 경우, 똑같은 `"hello"`를 계속 객체로 만들면 메모리가 낭비된다. 그래서 다음과 같이 중복된 객체 생성을 막는 캐시를 만든다.
  
    ```java
  Map<String, WeakReference<MyObject>> cache = new HashMap<>();

  public MyObject getOrCreate(String value) {
  WeakReference<MyObject> ref = cache.get(value);
  MyObject obj = (ref != null) ? ref.get() : null;
  
      if (obj == null) {
          obj = new MyObject(value);
          cache.put(value, new WeakReference<>(obj));
      }
  
      return obj;
  }
    ```
  - `value`가 같은 `MyObejct`는 캐시에 한번만 만들어서 저장
  - 그런데 `WeakReference`를 쓰면, 이 `MyObject`가 더 이상 사용되지 않으면 GC가 정리해줌
  - 즉, 캐시는 계속 쌓이지 않고 필요 없어진 객체는 자동으로 정리됨

- 실제 사례 : `String.intern()`

  ```java
  String a = "hello".intern();
  String b = "hello".intern();
  System.out.println(a == b); // true
  ```
  
  - `String.intern()`도 개념이 비슷하다.
  - 만약 JVM의 문자열 풀을 확인하여 해당 문자열이 이미 존재하면 그 풀 안에 있는 문자열 객체의 참조를 반환하고, 없다면 객체를 문자열 풀에 새로 추가하고, 풀에 추가된 그 객체의 참조를 반환한다.

#### **참조 큐(reference queue)** 에 등록된 약한 참조들 중에서 새롭게 해제된 것들을 해당 큐에 넣을(enqueue) 것입니다 ❓
> GC가 어떤 객체를 메모리에서 지우면, 그 객체를 가리키던 약한 참조(WeakReference)를 참조 큐(ReferenceQueue)에 자동으로 넣는다.

즉,
- 어떤 객체가 약한 참조로만 연결되어있다가, 
- GC가 "이거 더 이상 안쓰네?"하고 지우기로 결정하면, 
- 그 참조는 참조 큐에 들어간다 👉 "이 객체는 없어졌어!" 라고 알려주는 역할

- 예제 코드

  ```java
  ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
  java.lang.ref.WeakReference<Object> objectWeakReference = new java.lang.ref.WeakReference<>(new Object(),referenceQueue);

  System.gc(); // GC요청

  // GC는 멀티쓰레드 환경에서 비동기적으로 수행되기 때문에 시간을 줘서 기다리기
  Thread.sleep(1000);

  Reference<?> reference = referenceQueue.poll();
  if(reference != null){
      System.out.println("GC가 객체를 제거했다!");
  }
  ```


