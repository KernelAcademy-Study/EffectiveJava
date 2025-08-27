# [Item 32] 제네릭과 가변인수를 함께 쓸 때는 신중하라

## 서론

자바에서 가변인수(varargs) 메서드는 호출 시 인수들을 자동으로 배열에 담는다.
문제는 배열과 제네릭은 잘 섞이지 않는다는 점이다(아이템 28).
따라서 제네릭 타입을 가변인수로 쓰면 힙 오염(Heap Pollution) 문제가 생길 수 있다.

---

## 문제 상황: 제네릭 가변인수

```java
@SafeVarargs // 경고 억제 (일단 붙여놓음)
static <T> void dangerous(T... args) {
    Object[] array = args;      // T[] → Object[] 로 취급
    array[0] = "문자열";         // 다른 타입 삽입 가능
    T t = args[0];              // 런타임에 ClassCastException 💥
}
```

사용:

```java
dangerous(42, 100); // Integer 배열
```

- 하지만 내부에서 String을 끼워넣으면, 꺼낼 때 Integer로 캐스팅하다가 예외 발생.
- 즉, 타입 안정성이 깨질 수 있음.

---

## 더 위험한 상황: 제네릭 배열 참조 노출

```java
static <T> T[] toArray(T... args) {
    return args; // 제네릭 배열 참조를 그대로 반환
}

static <T> T[] pickTwo(T a, T b, T c) {
    switch(ThreadLocalRandom.current().nextInt(3)) {
        case 0: return toArray(a, b);
        case 1: return toArray(a, c);
        case 2: return toArray(b, c);
    }
    throw new AssertionError();
}
```

사용:

```java
String[] attributes = pickTwo("좋은", "빠른", "저렴한");
// ClassCastException! Object[]를 String[]로 캐스팅할 수 없음
```

---

## 해결책 1: @SafeVarargs 적절히 사용

메서드 내부에서 절대로 힙 오염이 일어나지 않는 경우에만 사용:

```java
@SafeVarargs
static <T> List<T> asList(T... elements) {
    return Arrays.asList(elements); // 배열 그대로 리스트로 감싸기만 함
}

@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists) {
        result.addAll(list);
    }
    return result; // 안전: 제네릭 배열을 외부로 노출하지 않음
}
```

### @SafeVarargs 사용 조건

1. 제네릭이나 매개변수화 타입의 varargs 매개변수에 값을 저장하지 않는다.
2. 그 배열(또는 복제본)을 신뢰할 수 없는 코드에 노출하지 않는다.

---

## 해결책 2: 배열 대신 List 받기

가능하면 가변인수 제네릭 대신 List로 받는 것이 더 안전하다:

```java
static <T> List<T> safeList(List<T> elements) {
    return new ArrayList<>(elements);
}

static <T> List<T> pickTwo(T a, T b, T c, List<T> list) {
    switch(ThreadLocalRandom.current().nextInt(3)) {
        case 0: return List.of(a, b);
        case 1: return List.of(a, c);
        case 2: return List.of(b, c);
    }
    throw new AssertionError();
}
```

사용:

```java
List<String> attributes = pickTwo("좋은", "빠른", "저렴한", List.of());
// 안전함: 컴파일 타임에 타입 체크
```

---

## 실무 활용 사례

### JDK에서 안전한 사용 예시

```java
// Arrays.asList - 안전함
List<String> list = Arrays.asList("a", "b", "c");

// Collections.addAll - 안전함
Set<String> set = new HashSet<>();
Collections.addAll(set, "x", "y", "z");

// EnumSet.of - 안전함
Set<Color> colors = EnumSet.of(Color.RED, Color.GREEN, Color.BLUE);
```

---

## 정리

1. **제네릭 + 가변인수** = 배열을 거치기 때문에 힙 오염 위험이 있다.
2. 반드시 필요한 경우가 아니라면 **List**를 인자로 받도록 설계하라.
3. 정말 안전한 경우에만 **@SafeVarargs**를 붙여서 경고를 억제할 수 있다.
4. **@SafeVarargs 조건**:
   - varargs 배열에 아무것도 저장하지 않음
   - 배열 참조가 외부로 노출되지 않음
5. JDK의 Arrays.asList, Collections.addAll 같은 API도 이 원칙을 따른다.