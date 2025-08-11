# 📌 Item 38: 확장할 수 있는 열거 타입이 필요하면 인터페이스를 사용하라

---

## 1. 개요
- **열거 타입(enum)** 은 대부분의 경우 **타입 안전 열거 패턴(type-safe enum pattern)** 보다 우수하다.
- 하지만 **열거 타입은 상속(확장)이 불가능**하다.
- 그럼에도 불구하고, 일부 상황에서는 **확장 가능한 열거 타입**이 유용하다.
- 대표적인 사례 → **연산 코드(Operation Code, opcode)**

---

## 2. 연산 코드(opcode) 예시
- **연산 코드**: 각 원소가 특정 연산을 의미하며, 기존 연산 외에 **새로운 연산을 추가**할 수 있어야 한다.
- 예시: 사칙연산을 기본으로 제공하고, 필요하면 새로운 수학 연산(제곱, 제곱근 등)을 확장.

---

## 3. 확장 가능한 열거 타입 설계 방법
열거 타입을 직접 상속할 수는 없지만, **인터페이스를 정의하고** 각 열거 타입이 이를 **구현**하도록 만들면 동일한 효과를 낼 수 있다.

### 3.1 인터페이스 정의
```java
public interface Operation {
    double apply(double x, double y);
}
```

### 3.2 기본 열거 타입 구현
```java
public enum BasicOperation implements Operation {
PLUS("+") {
public double apply(double x, double y) { return x + y; }
},
MINUS("-") {
public double apply(double x, double y) { return x - y; }
},
TIMES("*") {
public double apply(double x, double y) { return x * y; }
},
DIVIDE("/") {
public double apply(double x, double y) { return x / y; }
};

    private final String symbol;

    BasicOperation(String symbol) { this.symbol = symbol; }

    @Override
    public String toString() { return symbol; }
}
```
### 3.3 확장 열거 타입 구현
```java
public enum ExtendedOperation implements Operation {
    POWER("^") {
        public double apply(double x, double y) { return Math.pow(x, y); }
    },
    REMAINDER("%") {
        public double apply(double x, double y) { return x % y; }
    };

    private final String symbol;

    ExtendedOperation(String symbol) { this.symbol = symbol; }

    @Override
    public String toString() { return symbol; }
}
```
---
## 4. 사용 예시
### 4.1 Class 객체 기반
```java
public static <T extends Enum<T> & Operation> void test(
        Class<T> opEnumType, double x, double y) {
    for (Operation op : opEnumType.getEnumConstants()) {
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
    }
}

public static void main(String[] args) {
    test(BasicOperation.class, 4, 2);
    test(ExtendedOperation.class, 4, 2);
}
```
### 4.2 한정적 와일드카드 기반 (여러 구현체 조합 가능)

---
# 5. 장점
확장성: 새로운 열거 타입을 추가해도 기존 코드를 수정할 필요 없음.

유연성: API 사용자가 자신만의 연산을 추가 가능.

타입 안정성: 컴파일 시점에 잘못된 연산 타입 사용 방지.
---
---
# 6. 한계
열거 타입처럼 상속은 불가능하고, 반드시 인터페이스 기반으로만 확장 가능.

디폴트 메서드(Item 20) 를 사용하면, 인터페이스에 기본 구현을 제공할 수 있지만 모든 구현체에 동일하게 적용됨.

설계가 과도하게 복잡해질 수 있음.
---
# 7. 실제 자바 라이브러리 사례
   java.nio.file.LinkOption
   → CopyOption과 OpenOption 인터페이스를 구현.
----
이를 통해 열거 타입이 여러 인터페이스를 구현할 수 있음을 활용.

# 8. 핵심 정리
   열거 타입 자체는 확장할 수 없지만,
   인터페이스 + 기본 열거 타입 구현을 사용하면
   확장 가능한 열거 타입과 유사한 효과를 낼 수 있다.

----

이렇게 하면 **원문 내용 중 오타와 불명확한 표현**을 보완했고, 예제 코드도 `BasicOperation`과 `ExtendedOperation`을 모두 구현해서 확장성을 보여줬습니다.

원하시면 제가 이 예제에 **JUnit 테스트 코드**도 추가해서 실행 예시를 완성할 수 있습니다. 그러면 바로 돌려볼 수 있는 형태가 됩니다.  
JUnit 예시도 추가해 드릴까요?







