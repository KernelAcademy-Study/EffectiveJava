# [Item 4] 인스턴스화를 막으려거든 private 생성자를 사용하라

## 목적
이 아이템은 "이 클래스는 절대 인스턴스화되면 안 된다"는 요구사항을 만족시키기 위한 기법을 설명한다. 보통 **정적 메서드만 제공하는 유틸리티 클래스**나 **상수만 모아둔 클래스**가 해당된다.

## 문제 상황
예를 들어 `java.lang.Math`, `java.util.Collections`, `java.util.Arrays` 같은 클래스는 모든 메서드가 static이며 인스턴스를 만들 이유가 전혀 없다.
또한 추상 클래스로 만들어도 하위 클래스에서 인스턴스를 만들 수 있으므로, 인스턴스화 방지에는 적합하지 않다.
클래스에 생성자가 하나도 없으면, 컴파일러가 자동으로 public 기본 생성자를 추가해버린다. 이로 인해 외부에서 객체를 만들 수 있게 된다.
하지만 생성자를 명시하지 않으면 다음처럼 객체를 생성할 수 있다:

```java
Math m = new Math(); // ❌ 의미 없는 객체 생성
```

## 해결법: private 생성자
클래스 내부에 **private 생성자**를 명시해 외부에서 new로 인스턴스를 만들 수 없도록 막는다.

```java
public class Utility {
    private Utility() {
        throw new AssertionError();
    }

    public static int add(int a, int b) {
        return a + b;
    }
}

// 사용 예시
int result = Utility.add(3, 5); // ✅
// new Utility(); // ❌ 컴파일 에러
```

## 추가 장점
- 생성자를 `private`으로 선언하면 **상속도 막을 수 있음**
- 실수로라도 객체를 만들 수 없게 하여 **API 설계 의도를 강하게 표현**

## 실무 팁
- 보통 `final`까지 같이 붙여 **상속과 인스턴스화를 모두 차단**하는 게 일반적
- 생성자 내부에서 예외를 던지면, 리플렉션을 사용한 인스턴스화 시도도 차단 가능

```java
public final class Constants {
    public static final String APP_NAME = "MyApp";
    public static final int MAX_RETRY = 3;

    private Constants() {
        throw new AssertionError();
    }
}
```

```java
public final class PhysicsConstants {
    public static final double PLANCK = 6.626e-34;
    public static final double LIGHT_SPEED = 2.998e8;

    private PhysicsConstants() {
        throw new AssertionError();
    }
}
```

## 정리
객체를 만들 이유가 전혀 없는 클래스(유틸리티/상수 모음 등)는 반드시 **private 생성자**를 선언해 인스턴스화를 명확히 금지하라.