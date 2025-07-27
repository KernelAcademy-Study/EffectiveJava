#  📌 아이템 22: 인터페이스는 타입을 정의하는 용도로만 사용하라

---

## ✅ 들어가기

자바에서 **인터페이스(interface)** 는 **자신을 구현한 클래스의 인스턴스를 참조할 수 있는 타입** 역할을 한다.

즉, **"인터페이스를 구현한 클래스가 어떤 동작을 한다"**는 보장을 클라이언트에게 제공하며, **인터페이스는 오직 '이용'을 위한 계약서 역할**에 충실해야 한다.

---
## 예외
**상수 인터페이스** : static final 필드로만 가득 찬 인터페이스

**상수 인터페이스 안티패턴**  
-> 클래스 내부에서 사용하는 상수는 외부 인터페이스가 아니라 내부구현
-> 내부구현을 API로 노출함 
-> 의미가 없을 뿐더러, 클라이언트 코드가 상수들에 종속됨
```java

public class ChemistryLab implements PhysicalConstants {
    // AVOGADROS_NUMBER 같은 상수를 사용하는데, 이게 ChemistryLab의 API처럼 보임
}
```
🙅 이런 예시는 java.io.ObjectStreamConstants 같은 **JDK 내부에도 있지만**, **절대 따라하지 말아야 할 잘못된 사용 예**입니다.
- - -
## 그렇다면 상수는 어디에 두어야 하나?
1. 유틸리티 클래스에 정의 (아이템4)
```java
public class PhysicalConstants {
    private PhysicalConstants() {} // 인스턴스화 방지

    public static final double AVOGADROS_NUMBER   = 6.022_140_857e23;
    public static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    public static final double ELECTRON_MASS      = 9.109_383_56e-31;
}
```

2. 열거 타입 사용 (아이템 34)
```java

public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6);

    private final double mass;   // in kilograms
    private final double radius; // in meters
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() {
        final double G = 6.67300E-11;
        return G * mass / (radius * radius);
    }
}

```
- - -
# 핵심 정리
인터페이스는 타입을 정의하는 용도로만 사용해야 한다. 
상수 공개용 수단으로 사용하지 말자
- - -
  
