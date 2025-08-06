# # 📌 아이템 30: 이왕이면 제네릭 메서드로 만들라

---

## ✅ 개요

- **제네릭 메서드**란?
  - 메서드에 자체적인 타입 매개변수를 선언하여 **더욱 일반적이고 재사용 가능한 로직**을 작성할 수 있게 해줌
  - 정적 유틸리티 메서드는 대부분 제네릭으로 작성됨

---

## 🧪 제네릭 메서드 예시

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

* **장점**:
  * 타입 안전 (type-safe)
  * 경고 없이 컴파일 가능
  * 사용이 간결
* **제약**:
  * 입력 집합 2개와 반환 집합의 타입 E는 모두 같아야 함
  * **한정적 와일드카드**(? extends, ? super)를 통해 개선 가능 (→ 아이템 31 참고)

⠀
# 🧊 불변 객체를 다양한 타입으로 활용하려면?
* 제네릭은 **런타임에 타입 정보가 소거됨** (→ 아이템 28)
* **하나의 객체를 여러 타입으로 매개변수화**할 수 있음
* 타입 매개변수에 맞는 **정적 팩터리 메서드**를 활용

## 🧪 제네릭 싱글턴 팩터리
* 예시: Collections.emptySet(), Collections.reverseOrder() 등
* 대표적으로 **항등 함수 (Identity Function)**

```java
⠀private static UnaryOperator<Object> IDENTITY_FN = t -> t;

@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
}
```

* 항등 함수는 상태가 없으므로 싱글턴으로 재사용 가능
* 자바 제네릭이 **실체화**되지 않으므로 소거 방식에 기반해 타입 안정성 확보 가능

## 🔁 재귀적 타입 한정 (Recursive Type Bound)

```java
public static <E extends Comparable<E>> E max(Collection<E> c);
```

# 💡 관련 아이템 요약
| **아이템** | **내용** |
|:-:|:-:|
| 아이템 28 | 제네릭과 타입 소거 |
| 아이템 31 | 한정적 와일드카드 타입 사용 |
| 아이템 42 | 정적 팩터리 메서드 |
| 아이템 59 | 항등 함수 Function.identity() |
| 아이템 14 | Comparable 인터페이스 |
| 아이템 2 | 생성자 대신 정적 팩터리 메서드 사용 |
##  ✅ 결론
* 메서드를 제네릭으로 만들면 **타입 안전성, 재사용성, 유연성**이 크게 향상됨
* **항등 함수나 불변 객체**는 제네릭 싱글턴 패턴으로 효율적으로 재사용
* 복잡한 타입 한정도 **관용구나 와일드카드**로 충분히 해결 가능





