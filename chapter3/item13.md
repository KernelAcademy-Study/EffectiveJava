# [Item 13] clone 재정의는 주의해서 진행하라

---

## Stack 예시로 보는 얕은 복사의 문제

```
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

- clone 메서드에서 `super.clone()`만 호출하면 `elements` 배열은 얕은 복사됨
- 원본과 복제본이 같은 배열 참조 → 불변식 깨짐 가능성
- `elements.clone()`으로 **배열 깊은 복사**가 필요

---

## Stack용 올바른 clone 메서드 예시

```
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone(); // 깊은 복사
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

- 배열의 clone은 런타임 타입과 컴파일타임 타입 모두 유지됨
- final 필드라면 clone에서 새 값 할당 불가 → 구조 변경 필요

---

## HashTable 예시 - 연결 리스트 처리 문제

```
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
}
```

### 잘못된 clone: 얕은 복사만 수행

```
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone(); // 내부 연결 리스트는 공유됨
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

- 원본과 복제본이 같은 연결 리스트 참조 → 예기치 않은 동작 발생

---

## 재귀적 깊은 복사 적용

```
private static class Entry {
    final Object key;
    Object value;
    Entry next;

    Entry deepCopy() {
        return new Entry(key, value,
            next == null ? null : next.deepCopy());
    }
}

@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = new Entry[buckets.length];
        for (int i = 0; i < buckets.length; i++)
            if (buckets[i] != null)
                result.buckets[i] = buckets[i].deepCopy();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

- 스택 깊이가 긴 경우 재귀 호출은 StackOverflow 위험 존재

---

## 반복문 기반 깊은 복사 (재귀 대신)

```
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

---

## 고수준 메서드 기반 복사 (예: put)

- `super.clone()` 후 상태를 put 등으로 재구성하는 방법
- clone 내부에서 하위 클래스가 override한 메서드는 호출 금지

```
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = new Entry[buckets.length];
        for (Entry e : this.entrySet())
            result.put(e.key, e.value); // put은 final이거나 private이어야 안전
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

---

## 기타 고려사항

- clone은 반드시 `public`으로 재정의해야 하며 `CloneNotSupportedException`은 던지지 않아야 사용이 편함
- Cloneable을 구현한 **스레드 안전 클래스**는 `clone` 메서드도 적절히 동기화 필요
- 하위 클래스에서 Cloneable 사용을 막고 싶다면:

```
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

---

## 복사 생성자 / 복사 팩터리 방식이 더 낫다

```
// 복사 생성자
public Yum(Yum original) { ... }

// 복사 팩터리
public static Yum newInstance(Yum original) { ... }
```

### 복사 생성자 / 팩터리의 장점

- 생성자 방식이므로 언어 차원에서 안전
- 문서화된 규약 불필요
- final 필드와도 충돌 없음
- 형변환, 예외 처리 불필요
- 인터페이스 기반 타입 변환 가능 → TreeSet(new HashSet())

---

## 핵심 정리

- `Cloneable`은 많은 결함이 있는 구조이며, 복사에는 복사 생성자나 복사 팩터리를 사용하는 것이 바람직하다.
- `Cloneable`을 구현하는 경우:
  - 반드시 `clone()`을 재정의
  - 내부 가변 객체를 깊은 복사
  - 배열 복사는 `clone()`이 유일한 예외적으로 적절한 케이스
- 새로운 인터페이스나 클래스에 `Cloneable`은 사용하지 말 것
