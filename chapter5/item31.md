# [Item 31] 한정적 와일드카드를 사용해 API 유연성을 높이라

## 서론

제네릭을 설계하다 보면 매개변수 타입을 너무 딱 고정해버리면, 사용자가 불편해진다.
이때 와일드카드(?) 를 잘 활용하면 API가 더 유연해지고, 여러 타입 계층을 한 메서드에서 처리할 수 있다.

---

## 문제 상황: 고정된 제네릭 메서드

```java
public static <T> void pushAll(Stack<T> stack, Iterable<T> src) {
    for (T t : src) stack.push(t);
}
```

사용:

```java
Stack<Number> numbers = new Stack<>();
Iterable<Integer> ints = List.of(1, 2, 3);

// ❌ 오류: Integer는 Number의 하위 타입인데도 불구하고 안 됨
pushAll(numbers, ints);
```

- Stack<Number> 와 Stack<Integer> 는 서로 다른 타입이다.
- 제네릭은 불공변(invariant) 이라서 하위 타입을 자동 변환하지 않는다.

---

## 해결책 1: ? extends T (producer → 읽기 전용)

```java
public static <T> void pushAll(Stack<T> stack, Iterable<? extends T> src) {
    for (T t : src) stack.push(t);
}
```

사용:

```java
Stack<Number> numbers = new Stack<>();
Iterable<Integer> ints = List.of(1, 2, 3);

// ✅ OK: Integer는 Number의 하위 타입이라서 허용
pushAll(numbers, ints);
```

---

## 문제 상황 2: popAll 메서드

```java
public static <T> void popAll(Stack<T> stack, Collection<T> dst) {
    while (!stack.isEmpty()) {
        dst.add(stack.pop());
    }
}
```

- Stack<Integer> → Collection<Number> 에 담고 싶을 때 불가능하다.

---

## 해결책 2: ? super T (consumer → 쓰기 전용)

```java
public static <T> void popAll(Stack<T> stack, Collection<? super T> dst) {
    while (!stack.isEmpty()) {
        dst.add(stack.pop());
    }
}
```

사용:

```java
Stack<Integer> ints = new Stack<>();
ints.push(1); ints.push(2);

Collection<Number> nums = new ArrayList<>();
popAll(ints, nums); // ✅ OK
```

---

## 핵심 원칙 (PECS)

**PECS: Producer-Extends, Consumer-Super**

- **생산자(Producer)** → 데이터를 꺼내는 쪽이면 `? extends T`
- **소비자(Consumer)** → 데이터를 넣는 쪽이면 `? super T`

👉 **"생산자는 extends, 소비자는 super"** 만 기억하자.

---

## 실무 활용 사례

### Collections.copy() 메서드

```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    // src에서 읽어서(producer) dest에 쓴다(consumer)
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}
```

### 사용 예시

```java
List<Integer> ints = Arrays.asList(1, 2, 3);
List<Number> nums = Arrays.asList(0.0, 0.0, 0.0);

Collections.copy(nums, ints); // ✅ OK
// ints는 producer(extends), nums는 consumer(super)
```

---

## 주의사항

### 1. 반환 타입에는 와일드카드 사용하지 말기

```java
// ❌ 나쁜 예: 사용자가 와일드카드를 신경 써야 함
public static List<? extends Number> getNumbers() { ... }

// ✅ 좋은 예: 구체적인 타입 반환
public static List<Number> getNumbers() { ... }
```

### 2. 메서드 내부에서만 사용할 때는 일반 제네릭

```java
// 메서드 내부에서만 사용 → 일반 제네릭 OK
public static <T> T max(Collection<T> items) {
    // ...
}
```

---

## 정리

1. 와일드카드를 쓰면 API의 유연성이 크게 향상된다.
2. **PECS 규칙**을 적용해서 설계해야 한다.
   - Producer → `? extends T`
   - Consumer → `? super T`
3. 불필요하게 제네릭 타입을 고정하지 말고, 적절히 활용하라.
4. 반환 타입에는 와일드카드를 사용하지 않는다.
5. 실무에서 Collections API들이 모두 이 원칙을 따르고 있다.