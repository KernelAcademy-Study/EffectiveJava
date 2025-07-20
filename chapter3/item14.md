# 📘 Effective Java - Item 14: Comparable을 구현할지 고려하라

## ✅ 요약
> **값 클래스(value class)**를 작성할 때, 그 클래스의 인스턴스가 **자연적인 순서를 가진다면 `Comparable` 인터페이스를 구현**하라.

## 🌟 이유
- 인스턴스를 정렬 가능하게 만들어 컬렉션과의 호환성을 높일 수 있음
- `Collections.sort`, `Arrays.sort`, `TreeSet`, `TreeMap` 등에 자연스럽게 사용 가능

## 📖 자연적인 순서란?
- 클래스 인스턴스 사이의 **직관적인 순서**
- 예시
  - `String`: 사전 순서
  - `Integer`: 숫자 크기 순서
  - `Date`: 시간 순서

## 🛠️ 구현 방법
`Comparable<T>` 인터페이스를 구현하고 `compareTo` 메서드를 작성

### 기본 구현 예시
```java
public class PhoneNumber implements Comparable<PhoneNumber> {
    private final int areaCode;
    private final int prefix;
    private final int lineNumber;

    @Override
    public int compareTo(PhoneNumber pn) {
        int result = Integer.compare(areaCode, pn.areaCode);
        if (result == 0) {
            result = Integer.compare(prefix, pn.prefix);
            if (result == 0) {
                result = Integer.compare(lineNumber, pn.lineNumber);
            }
        }
        return result;
    }
}
```

### `Comparator` 유틸리티 사용 예시 (추천)
```java
public int compareTo(PhoneNumber pn) {
    return Comparator
        .comparingInt(PhoneNumber::getAreaCode)
        .thenComparingInt(PhoneNumber::getPrefix)
        .thenComparingInt(PhoneNumber::getLineNumber)
        .compare(this, pn);
}
```

## ⚠️ 주의사항
- `compareTo`의 반환값 의미
  - 음수: `this` < `other`
  - 0: `this` == `other`
  - 양수: `this` > `other`
- `equals`와 일관성 유지
  - `x.compareTo(y) == 0` 이면 `x.equals(y) == true` 여야 함

## 📝 요약 문장
> **값 클래스라면 자연적인 순서를 정의하고 `Comparable`을 구현하라.**
>  
> 정렬 가능성은 컬렉션과의 호환성과 사용성을 높여준다.
