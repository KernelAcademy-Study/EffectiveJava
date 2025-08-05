# [Item 12] toString을 항상 재정의하라

## 서론
모든 구체 클래스에서는 `Object`의 `toString`을 재정의해야 한다. `toString`을 잘 구현한 클래스는 사용하기 편하고, 그 클래스를 사용한 시스템은 디버깅하기 쉽다. `toString`은 객체를 출력했을 때나 디버거가 객체를 표시할 때, 로깅할 때 자동으로 호출된다.

---

## 기본 toString의 문제점

### Object의 기본 toString

```java
public class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "지역코드");
        this.prefix = rangeCheck(prefix, 999, "프리픽스");
        this.lineNum = rangeCheck(lineNum, 9999, "가입자 번호");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }
    
    // toString을 재정의하지 않은 상태
}

// 사용 예시
public class ToStringTest {
    public static void main(String[] args) {
        PhoneNumber jenny = new PhoneNumber(707, 867, 5309);
        System.out.println("제니의 번호: " + jenny);
        // 출력: 제니의 번호: PhoneNumber@163b91
        // 이 정보로는 전화번호를 알 수 없다!
    }
}
```

**문제점**: `PhoneNumber@163b91` 같은 정보는 전혀 유용하지 않다. 클래스명@16진수해시코드만 보여준다.

---

## 올바른 toString 구현

### 1. 기본적인 toString 재정의

```java
public class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "지역코드");
        this.prefix = rangeCheck(prefix, 999, "프리픽스");  
        this.lineNum = rangeCheck(lineNum, 9999, "가입자 번호");
    }

    @Override
    public String toString() {
        return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }
}

// 사용 예시
public class ImprovedToStringTest {
    public static void main(String[] args) {
        PhoneNumber jenny = new PhoneNumber(707, 867, 5309);
        System.out.println("제니의 번호: " + jenny);
        // 출력: 제니의 번호: 707-867-5309
        // 훨씬 유용한 정보!

        // 컬렉션에서도 유용
        List<PhoneNumber> phoneBook = Arrays.asList(
            new PhoneNumber(707, 867, 5309),
            new PhoneNumber(555, 123, 4567)
        );
        System.out.println(phoneBook);
        // 출력: [707-867-5309, 555-123-4567]
    }
}
```

### 2. 복잡한 객체의 toString

```java
public class Person {
    private final String name;
    private final int age;
    private final PhoneNumber phoneNumber;
    private final Address address;

    public Person(String name, int age, PhoneNumber phoneNumber, Address address) {
        this.name = name;
        this.age = age;
        this.phoneNumber = phoneNumber;
        this.address = address;
    }

    @Override
    public String toString() {
        return String.format("Person{name='%s', age=%d, phone=%s, address=%s}",
                name, age, phoneNumber, address);
    }
}

public class Address {
    private final String street;
    private final String city;
    private final String zipCode;

    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }

    @Override
    public String toString() {
        return String.format("%s, %s %s", street, city, zipCode);
    }
}

// 사용 예시
public class ComplexToStringTest {
    public static void main(String[] args) {
        PhoneNumber phone = new PhoneNumber(555, 123, 4567);
        Address address = new Address("123 Main St", "Springfield", "12345");
        Person person = new Person("홍길동", 30, phone, address);
        
        System.out.println(person);
        // 출력: Person{name='홍길동', age=30, phone=555-123-4567, address=123 Main St, Springfield 12345}
    }
}
```

---

## toString 구현 시 고려사항

### 1. 포맷 문서화 여부

**포맷을 명시하는 경우:**
```java
/**
 * 이 전화번호의 문자열 표현을 반환한다.
 * 이 표현은 "XXX-YYY-ZZZZ" 형식으로,
 * XXX는 지역 코드, YYY는 프리픽스, ZZZZ는 가입자 번호다.
 * 각각의 대문자는 10진수 숫자 하나를 나타낸다.
 * 
 * 전화번호의 각 부분이 차지하는 자리수가 적다면,
 * 앞에서부터 0으로 채워나간다. 예를 들어 가입자 번호가 123이라면
 * 전화번호의 마지막 네 문자는 "0123"이 된다.
 */
@Override
public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

**포맷을 명시하지 않는 경우:**
```java
/**
 * 이 포션의 대략적인 설명을 반환한다.
 * 다음은 이 설명의 일반적인 형태이나,
 * 상세 형식은 정해지지 않았으며 향후 변경될 수 있다.
 * 
 * "[포션 #9: 타입=사랑, 냄새=테러핀유, 겉모습=먹물]"
 */
@Override
public String toString() {
    return String.format("[포션 #%d: 타입=%s, 냄새=%s, 겉모습=%s]",
            potionNumber, type, smell, appearance);
}
```

### 2. toString에서 사용한 정보에 대한 접근자 제공

```java
public class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    // 생성자는 동일

    @Override
    public String toString() {
        return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
    }

    // toString에서 사용한 필드들에 대한 접근자 제공
    public int getAreaCode() { return areaCode; }
    public int getPrefix() { return prefix; }
    public int getLineNumber() { return lineNum; }
}
```

**이유**: toString의 반환값에 포함된 정보를 얻어올 수 있는 API를 제공하지 않으면, 그 정보가 필요한 프로그래머는 toString의 반환값을 파싱할 수밖에 없다.

---

## 다양한 toString 구현 예시

### 1. 컬렉션 클래스

```java
public class PhoneBook {
    private final Map<String, PhoneNumber> contacts;

    public PhoneBook() {
        this.contacts = new HashMap<>();
    }

    public void addContact(String name, PhoneNumber number) {
        contacts.put(name, number);
    }

    @Override
    public String toString() {
        if (contacts.isEmpty()) {
            return "PhoneBook{}";
        }

        StringBuilder sb = new StringBuilder();
        sb.append("PhoneBook{\n");
        for (Map.Entry<String, PhoneNumber> entry : contacts.entrySet()) {
            sb.append("  ").append(entry.getKey())
              .append(": ").append(entry.getValue()).append("\n");
        }
        sb.append("}");
        return sb.toString();
    }
}

// 사용 예시
public class PhoneBookTest {
    public static void main(String[] args) {
        PhoneBook book = new PhoneBook();
        book.addContact("홍길동", new PhoneNumber(555, 123, 4567));
        book.addContact("김철수", new PhoneNumber(555, 987, 6543));
        
        System.out.println(book);
        // 출력:
        // PhoneBook{
        //   홍길동: 555-123-4567
        //   김철수: 555-987-6543
        // }
    }
}
```

### 2. 불변 객체의 builder 패턴과 함께

```java
public class Computer {
    private final String cpu;
    private final String gpu;
    private final int ram;
    private final int storage;

    private Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.gpu = builder.gpu;
        this.ram = builder.ram;
        this.storage = builder.storage;
    }

    @Override
    public String toString() {
        return String.format("Computer{cpu='%s', gpu='%s', ram=%dGB, storage=%dGB}",
                cpu, gpu, ram, storage);
    }

    public static class Builder {
        private String cpu;
        private String gpu = "내장 그래픽";
        private int ram = 8;
        private int storage = 256;

        public Builder cpu(String cpu) {
            this.cpu = cpu;
            return this;
        }

        public Builder gpu(String gpu) {
            this.gpu = gpu;
            return this;
        }

        public Builder ram(int ram) {
            this.ram = ram;
            return this;
        }

        public Builder storage(int storage) {
            this.storage = storage;
            return this;
        }

        public Computer build() {
            return new Computer(this);
        }
    }
}

// 사용 예시
public class ComputerTest {
    public static void main(String[] args) {
        Computer gaming = new Computer.Builder()
                .cpu("Intel i9-13900K")
                .gpu("NVIDIA RTX 4090")
                .ram(32)
                .storage(1000)
                .build();

        System.out.println("게이밍 PC: " + gaming);
        // 출력: 게이밍 PC: Computer{cpu='Intel i9-13900K', gpu='NVIDIA RTX 4090', ram=32GB, storage=1000GB}
    }
}
```

### 3. 상속 관계에서의 toString

```java
public abstract class Shape {
    protected final String color;

    protected Shape(String color) {
        this.color = color;
    }

    @Override
    public String toString() {
        return String.format("%s{color='%s'}", getClass().getSimpleName(), color);
    }
}

public class Circle extends Shape {
    private final double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public String toString() {
        return String.format("Circle{color='%s', radius=%.2f}", color, radius);
    }
}

public class Rectangle extends Shape {
    private final double width, height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public String toString() {
        return String.format("Rectangle{color='%s', width=%.2f, height=%.2f}", 
                color, width, height);
    }
}

// 사용 예시
public class ShapeTest {
    public static void main(String[] args) {
        List<Shape> shapes = Arrays.asList(
            new Circle("빨강", 5.0),
            new Rectangle("파랑", 3.0, 4.0)
        );

        for (Shape shape : shapes) {
            System.out.println(shape);
        }
        // 출력:
        // Circle{color='빨강', radius=5.00}
        // Rectangle{color='파랑', width=3.00, height=4.00}
    }
}
```

---

## toString을 구현하지 않아도 되는 경우

### 1. 정적 유틸리티 클래스

```java
public final class MathUtils {
    private MathUtils() {
        throw new AssertionError("인스턴스화 불가");
    }

    public static int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }

    // toString 재정의 불필요 - 인스턴스가 만들어지지 않음
}
```

### 2. 이미 완벽한 toString을 제공하는 상위 클래스

```java
public abstract class AbstractStringBuilder {
    @Override
    public String toString() {
        // 완벽한 구현
        return new String(value, 0, count);
    }
}

public final class StringBuilder extends AbstractStringBuilder {
    // toString을 재정의할 필요 없음 - 상위 클래스의 구현이 완벽
}
```

### 3. 열거 타입 (대부분의 경우)

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6),
    EARTH  (5.976e+24, 6.37814e6);

    private final double mass;
    private final double radius;

    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    // 기본 toString()이 상수명을 반환하므로 대부분 적절함
    // Planet.EARTH.toString() -> "EARTH"
}
```

---

## 실무에서의 모범 사례

### 1. IDE 자동 생성 활용 후 수정

```java
public class Product {
    private final Long id;
    private final String name;
    private final BigDecimal price;
    private final Category category;

    // 생성자, getter 등...

    // IDE가 생성한 기본 toString
    @Override
    public String toString() {
        return "Product{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", price=" + price +
                ", category=" + category +
                '}';
    }

    // 비즈니스 요구에 맞게 개선한 toString
    @Override
    public String toString() {
        return String.format("%s (ID: %d, 가격: %s원, 카테고리: %s)",
                name, id, price, category.getName());
    }
}
```

### 2. 로깅과 디버깅을 위한 상세한 toString

```java
public class OrderItem {
    private final Product product;
    private final int quantity;
    private final BigDecimal unitPrice;
    private final BigDecimal discount;

    // 생성자, getter 등...

    @Override
    public String toString() {
        BigDecimal totalPrice = unitPrice.multiply(BigDecimal.valueOf(quantity))
                                        .subtract(discount);
        
        return String.format("OrderItem{product=%s, quantity=%d, unitPrice=%s, discount=%s, totalPrice=%s}",
                product.getName(), quantity, unitPrice, discount, totalPrice);
    }
}
```

---

## 정리

1. **모든 구체 클래스에서 toString을 재정의하라**
   - 디버깅과 로깅이 훨씬 쉬워진다

2. **간결하면서도 사람이 읽기 편한 형태의 유익한 정보를 반환하라**
   - 객체가 가진 주요 정보를 모두 반환하는 것이 좋다

3. **반환값의 포맷을 문서화할지 정하라**
   - 포맷을 명시하면 클라이언트 코드가 의존할 수 있음을 고려하라

4. **toString이 반환한 값에 포함된 정보를 얻어올 수 있는 API를 제공하라**
   - 문자열 파싱에 의존하지 않도록 접근자를 제공해야 한다

5. **AutoValue, Lombok 등의 도구를 활용하는 것도 좋은 방법이다**
   - 반복적인 작업을 자동화할 수 있다

좋은 toString 구현은 그 클래스를 사용하는 프로그래머의 삶을 훨씬 윤택하게 만들어준다!