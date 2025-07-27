# 아이템 11: equals를 재정의하려거든 hashCode도 재정의하라

`equals()`를 재정의한 클래스는 반드시 `hashCode()`도 재정의해야 한다. 그렇지 않으면 `HashMap`, `HashSet` 같은 해시 기반 컬렉션에서 올바르게 동작하지 않으며, 심각한 버그로 이어질 수 있다.

## hashCode 규약 (Object 명세 기준)

1. equals 비교에 사용되는 정보가 변하지 않았다면, 실행 중에는 항상 같은 hashCode를 반환해야 한다.
2. equals(Object)가 true인 두 객체는 반드시 같은 hashCode를 반환해야 한다.
3. equals(Object)가 false인 두 객체는 hashCode가 달라야 할 필요는 없지만, 달라야 해시 성능이 좋아진다.

## 문제 사례

```java
Map<PhoneNumber, String> map = new HashMap<>();
map.put(new PhoneNumber(707, 867, 5309), "제니");
map.get(new PhoneNumber(707, 867, 5309)); // null 반환
```

`PhoneNumber`가 `equals()`는 재정의했지만 `hashCode()`를 재정의하지 않아 발생한 문제다. 논리적으로 같은 두 객체가 다른 해시코드를 반환하여, HashMap 내부에서 같은 키로 인식되지 못한다.

## 잘못된 예시

```java
@Override public int hashCode() { return 42; }
```

모든 객체가 같은 해시코드를 갖게 되어 해시 버킷이 하나로 몰리고, 성능이 O(1) → O(n)으로 저하된다.

## 좋은 구현 요령

1. `int result`를 선언하고 첫 핵심 필드의 해시코드로 초기화한다.
2. 나머지 핵심 필드마다 `result = 31 * result + fieldHash` 방식으로 갱신한다.
3. `result`를 반환한다.

> 핵심 필드란 equals에서 사용하는 필드를 의미한다.

### 예시

```java
@Override
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

### 간편한 한 줄 구현

```java
@Override
public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

> 간단하지만 박싱 및 배열 생성으로 인해 성능은 다소 떨어짐.

### 고급: 지연 초기화 (불변 객체에 한함)

```java
private int hashCode;

@Override
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

> 초기 값 0은 자주 생성되는 해시코드와 겹치지 않도록 주의.

## 정리

- `equals()`를 재정의하면 반드시 `hashCode()`도 재정의해야 한다.
- 핵심 필드만 사용하고, 순서를 고려해 곱셈과 더하기를 활용하자.
- 구현은 번거로워도 하지 않으면 해시 기반 자료구조에서 큰 문제가 생긴다.
- IDE 자동 생성 기능이나 AutoValue 같은 도구를 사용하면 편리하다.