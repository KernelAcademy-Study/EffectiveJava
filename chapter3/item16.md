# [Item 16] public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

## 서론
때로는 인스턴스 필드들을 모아놓는 일 외에는 아무 목적도 없는 클래스를 작성하게 된다. 이런 퇴보한 클래스는 public이어서는 안 된다. 대신 필드를 모두 private으로 바꾸고 public 접근자(getter)를 추가해야 한다.

---

## 문제가 있는 퇴보한 클래스

### 나쁜 예시: public 필드를 가진 클래스

```java
// 이런 퇴보한 클래스는 public이어서는 안 된다!
class Point {
    public double x;
    public double y;
}
```

**문제점:**
1. 데이터 필드에 직접 접근할 수 있어 캡슐화의 이점을 제공하지 못한다
2. API를 수정하지 않고는 내부 표현을 바꿀 수 없다
3. 불변식을 보장할 수 없다
4. 외부에서 필드에 접근할 때 부수 작업을 수행할 수 없다

### 실제 문제 상황

```java
public class PointTest {
    public static void main(String[] args) {
        Point p = new Point();
        p.x = 3.0;
        p.y = 4.0;
        
        // 문제 1: 유효하지 않은 값을 설정할 수 있음
        p.x = Double.NaN;
        p.y = Double.NEGATIVE_INFINITY;
        
        // 문제 2: 내부 표현 변경 시 클라이언트 코드도 모두 수정해야 함
        System.out.println("x: " + p.x + ", y: " + p.y);
        
        // 문제 3: 부수 작업(로깅, 유효성 검사 등)을 수행할 수 없음
        p.x = 10.0;  // 이 변경사항을 추적하거나 검증할 방법이 없음
    }
}
```

---

## 올바른 해결책: 접근자와 변경자 메서드

### 개선된 Point 클래스

```java
public class Point {
    private double x;
    private double y;
    
    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    // 접근자 메서드 (getter)
    public double getX() { 
        return x; 
    }
    
    public double getY() { 
        return y; 
    }
    
    // 변경자 메서드 (setter) - 유효성 검사 포함
    public void setX(double x) {
        if (Double.isNaN(x) || Double.isInfinite(x)) {
            throw new IllegalArgumentException("유효하지 않은 x 좌표: " + x);
        }
        System.out.println("x 좌표가 " + this.x + "에서 " + x + "로 변경됨");  // 로깅
        this.x = x;
    }
    
    public void setY(double y) {
        if (Double.isNaN(y) || Double.isInfinite(y)) {
            throw new IllegalArgumentException("유효하지 않은 y 좌표: " + y);
        }
        System.out.println("y 좌표가 " + this.y + "에서 " + y + "로 변경됨");  // 로깅
        this.y = y;
    }
    
    // 추가적인 기능
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
    
    public Point translate(double dx, double dy) {
        return new Point(x + dx, y + dy);
    }
    
    @Override
    public String toString() {
        return String.format("Point(%.2f, %.2f)", x, y);
    }
}
```

### 개선된 사용 예시

```java
public class ImprovedPointTest {
    public static void main(String[] args) {
        Point p = new Point(3.0, 4.0);
        
        // 장점 1: 유효하지 않은 값 설정 시 예외 발생
        try {
            p.setX(Double.NaN);
        } catch (IllegalArgumentException e) {
            System.out.println("오류 감지: " + e.getMessage());
        }
        
        // 장점 2: 부수 작업 수행 (로깅)
        p.setX(10.0);  // "x 좌표가 3.00에서 10.00로 변경됨" 출력
        
        // 장점 3: 추가 기능 활용
        System.out.println("현재 위치: " + p);
        System.out.println("원점으로부터의 거리: " + p.distanceFromOrigin());
        
        Point moved = p.translate(1.0, 1.0);
        System.out.println("이동된 위치: " + moved);
    }
}
```

---

## 내부 표현의 유연성

### 내부 표현 변경의 예시

```java
// 버전 1: 직교 좌표계
public class Point {
    private double x;
    private double y;
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

```java
// 버전 2: 극좌표계로 내부 표현 변경 (API는 동일하게 유지)
public class Point {
    private double r;     // 반지름
    private double theta; // 각도 (라디안)
    
    public Point(double x, double y) {
        this.r = Math.sqrt(x * x + y * y);
        this.theta = Math.atan2(y, x);
    }
    
    // 외부 API는 동일하게 유지
    public double getX() { 
        return r * Math.cos(theta); 
    }
    
    public double getY() { 
        return r * Math.sin(theta); 
    }
    
    public void setX(double x) {
        double y = getY();
        this.r = Math.sqrt(x * x + y * y);
        this.theta = Math.atan2(y, x);
    }
    
    public void setY(double y) {
        double x = getX();
        this.r = Math.sqrt(x * x + y * y);
        this.theta = Math.atan2(y, x);
    }
    
    // 극좌표계의 장점을 활용한 새로운 메서드
    public double getRadius() { return r; }
    public double getAngle() { return theta; }
}
```

---

## 불변 클래스에서의 활용

### 불변 Point 클래스

```java
public final class ImmutablePoint {
    private final double x;
    private final double y;
    
    public ImmutablePoint(double x, double y) {
        if (Double.isNaN(x) || Double.isInfinite(x)) {
            throw new IllegalArgumentException("유효하지 않은 x 좌표: " + x);
        }
        if (Double.isNaN(y) || Double.isInfinite(y)) {
            throw new IllegalArgumentException("유효하지 않은 y 좌표: " + y);
        }
        this.x = x;
        this.y = y;
    }
    
    // 접근자만 제공 (변경자는 없음)
    public double getX() { return x; }
    public double getY() { return y; }
    
    // 새로운 인스턴스를 반환하는 메서드들
    public ImmutablePoint translate(double dx, double dy) {
        return new ImmutablePoint(x + dx, y + dy);
    }
    
    public ImmutablePoint scale(double factor) {
        return new ImmutablePoint(x * factor, y * factor);
    }
    
    public double distanceFrom(ImmutablePoint other) {
        double dx = this.x - other.x;
        double dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ImmutablePoint)) return false;
        ImmutablePoint point = (ImmutablePoint) o;
        return Double.compare(point.x, x) == 0 && 
               Double.compare(point.y, y) == 0;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }
    
    @Override
    public String toString() {
        return String.format("ImmutablePoint(%.2f, %.2f)", x, y);
    }
}
```

---

## package-private 클래스나 private 중첩 클래스의 예외

### package-private 클래스에서는 필드 노출이 때로는 나을 수 있다

```java
// 같은 패키지 내에서만 사용되는 클래스
class PackagePrivatePoint {
    public double x;  // 패키지 내부에서만 접근 가능
    public double y;  // 패키지 내부에서만 접근 가능
    
    PackagePrivatePoint(double x, double y) {
        this.x = x;
        this.y = y;
    }
}

// 이 클래스를 사용하는 public 클래스
public class Shape {
    private final List<PackagePrivatePoint> vertices;
    
    public Shape(PackagePrivatePoint... points) {
        this.vertices = Arrays.asList(points.clone());
    }
    
    public double calculateArea() {
        // 직접 필드에 접근하여 성능상 이점
        double area = 0.0;
        for (int i = 0; i < vertices.size(); i++) {
            PackagePrivatePoint current = vertices.get(i);
            PackagePrivatePoint next = vertices.get((i + 1) % vertices.size());
            area += current.x * next.y - next.x * current.y;
        }
        return Math.abs(area) / 2.0;
    }
}
```

### private 중첩 클래스에서의 예외

```java
public class ComplexCalculator {
    
    // private 중첩 클래스에서는 필드 노출이 문제없음
    private static class ComplexNumber {
        public double real;    // private 중첩 클래스이므로 public 필드 허용
        public double imaginary;
        
        ComplexNumber(double real, double imaginary) {
            this.real = real;
            this.imaginary = imaginary;
        }
    }
    
    public String add(double r1, double i1, double r2, double i2) {
        ComplexNumber c1 = new ComplexNumber(r1, i1);
        ComplexNumber c2 = new ComplexNumber(r2, i2);
        
        // 직접 필드 접근으로 간결한 코드
        ComplexNumber result = new ComplexNumber(c1.real + c2.real, c1.imaginary + c2.imaginary);
        
        return String.format("%.2f + %.2fi", result.real, result.imaginary);
    }
}
```

---

## 실무에서의 활용 예시

### 1. 데이터 전송 객체 (DTO)

```java
// 잘못된 DTO
public class BadUserDto {
    public String name;     // 위험!
    public String email;    // 위험!
    public int age;         // 위험!
}

// 올바른 DTO
public class UserDto {
    private String name;
    private String email;
    private int age;
    
    public UserDto() {}  // 기본 생성자 (직렬화용)
    
    public UserDto(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public int getAge() { return age; }
    public void setAge(int age) { 
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("유효하지 않은 나이: " + age);
        }
        this.age = age; 
    }
}
```

### 2. 설정 클래스

```java
public class DatabaseConfig {
    private String url;
    private String username;
    private String password;
    private int maxConnections = 10;
    private boolean autoCommit = true;
    
    // 체인 방식 setter 제공
    public DatabaseConfig url(String url) {
        this.url = Objects.requireNonNull(url, "URL은 null일 수 없습니다");
        return this;
    }
    
    public DatabaseConfig username(String username) {
        this.username = Objects.requireNonNull(username, "사용자명은 null일 수 없습니다");
        return this;
    }
    
    public DatabaseConfig password(String password) {
        this.password = Objects.requireNonNull(password, "비밀번호는 null일 수 없습니다");
        return this;
    }
    
    public DatabaseConfig maxConnections(int maxConnections) {
        if (maxConnections <= 0) {
            throw new IllegalArgumentException("최대 연결 수는 0보다 커야 합니다: " + maxConnections);
        }
        this.maxConnections = maxConnections;
        return this;
    }
    
    public DatabaseConfig autoCommit(boolean autoCommit) {
        this.autoCommit = autoCommit;
        return this;
    }
    
    // getter들
    public String getUrl() { return url; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public int getMaxConnections() { return maxConnections; }
    public boolean isAutoCommit() { return autoCommit; }
}

// 사용 예시
DatabaseConfig config = new DatabaseConfig()
    .url("jdbc:mysql://localhost:3306/mydb")
    .username("user")
    .password("password")
    .maxConnections(20)
    .autoCommit(false);
```

### 3. 측정값 클래스

```java
public class Temperature {
    private final double celsius;
    
    private Temperature(double celsius) {
        this.celsius = celsius;
    }
    
    // 팩토리 메서드들
    public static Temperature celsius(double celsius) {
        return new Temperature(celsius);
    }
    
    public static Temperature fahrenheit(double fahrenheit) {
        return new Temperature((fahrenheit - 32) * 5.0 / 9.0);
    }
    
    public static Temperature kelvin(double kelvin) {
        if (kelvin < 0) {
            throw new IllegalArgumentException("절대온도는 0 미만일 수 없습니다: " + kelvin);
        }
        return new Temperature(kelvin - 273.15);
    }
    
    // 변환 메서드들
    public double toCelsius() { return celsius; }
    public double toFahrenheit() { return celsius * 9.0 / 5.0 + 32; }
    public double toKelvin() { return celsius + 273.15; }
    
    @Override
    public String toString() {
        return String.format("%.2f°C", celsius);
    }
}

// 사용 예시
Temperature temp1 = Temperature.celsius(25.0);
Temperature temp2 = Temperature.fahrenheit(77.0);
Temperature temp3 = Temperature.kelvin(298.15);

System.out.println(temp1 + " = " + temp1.toFahrenheit() + "°F");
```

---

## 정리

1. **public 클래스는 절대 가변 필드를 직접 노출해서는 안 된다**
   - 캡슐화의 이점을 누릴 수 없게 된다

2. **불변 필드라면 노출해도 덜 위험하지만 완전히 안심할 수는 없다**
   - API를 변경하지 않고는 표현 방식을 바꿀 수 없고, 필드를 읽을 때 부수 작업을 수행할 수 없다

3. **package-private 클래스나 private 중첩 클래스에서는 종종 필드를 노출하는 편이 나을 수 있다**
   - 클라이언트 코드가 같은 패키지에 묶여있거나 바깥 클래스에 묶여있어서 변경의 영향 범위가 제한적이다

4. **접근자 메서드를 제공할 때는 다음을 고려하라**
   - 유효성 검사
   - 부수 작업 (로깅, 알림 등)
   - 내부 표현의 유연성
   - 불변성 보장

5. **성능이 중요한 내부 클래스에서는 필드 노출을 고려할 수 있다**
   - 하지만 이는 매우 신중하게 결정해야 한다

접근자 메서드는 단순히 필드를 반환하는 것 이상의 가치를 제공한다. 캡슐화를 통해 유연하고 안전한 API를 만들 수 있다!