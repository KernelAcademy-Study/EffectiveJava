# [Item 15] 클래스와 멤버의 접근 권한을 최소화하라

## 서론
잘 설계된 컴포넌트는 클래스 내부 데이터와 내부 구현 정보를 외부 컴포넌트로부터 완벽히 숨긴다. 이를 **정보 은닉(Information Hiding)** 또는 **캡슐화(Encapsulation)**라고 한다. 정보 은닉의 장점과 올바른 접근 제어자 사용법을 알아보자.

---

## 정보 은닉의 장점

### 1. 시스템 개발 속도 향상
- 여러 컴포넌트를 병렬로 개발할 수 있다
- 각 컴포넌트의 내부 구현을 모르고도 인터페이스만으로 개발 가능

### 2. 시스템 관리 비용 낮춤
- 각 컴포넌트를 더 빨리 파악하여 디버깅할 수 있다
- 다른 컴포넌트로 교체하는 부담이 적다

### 3. 성능 최적화에 도움
- 완성된 시스템을 프로파일링해 최적화할 컴포넌트를 정한 다음, 다른 컴포넌트에 영향을 주지 않고 해당 컴포넌트만 최적화할 수 있다

### 4. 소프트웨어 재사용성 높임
- 외부에 거의 의존하지 않고 독자적으로 동작할 수 있는 컴포넌트라면 그 컴포넌트와 함께 개발되지 않은 낯선 환경에서도 유용하게 쓰일 가능성이 크다

### 5. 큰 시스템 제작 난이도 낮춤
- 시스템 전체가 아직 완성되지 않은 상태에서도 개별 컴포넌트의 동작을 검증할 수 있다

---

## 접근 제어자와 접근 수준

### 자바의 접근 제어자 4단계

1. **private**: 멤버를 선언한 톱레벨 클래스에서만 접근 가능
2. **package-private** (default): 멤버가 소속된 패키지 안의 모든 클래스에서 접근 가능
3. **protected**: package-private의 접근 범위를 포함하며, 이 멤버를 선언한 클래스의 하위 클래스에서도 접근 가능
4. **public**: 모든 곳에서 접근 가능

### 기본 원칙: 모든 클래스와 멤버의 접근성을 가능한 한 좁혀야 한다

```java
// 잘못된 예시 - 불필요하게 public
public class BadExample {
    public List<String> items = new ArrayList<>();  // 위험!
    public int count = 0;  // 위험!
    
    public void processItems() {
        // 처리 로직
    }
    
    public void helperMethod() {  // 외부에서 사용하지 않는데 public
        // 내부 도우미 로직
    }
}
```

```java
// 올바른 예시 - 접근성을 최소화
public class GoodExample {
    private final List<String> items = new ArrayList<>();
    private int count = 0;
    
    public void processItems() {
        helperMethod();
        count++;
    }
    
    private void helperMethod() {  // 내부에서만 사용하므로 private
        // 내부 도우미 로직
    }
    
    // 필요한 경우에만 public 접근자 제공
    public int getItemCount() {
        return count;
    }
    
    public List<String> getItems() {
        return new ArrayList<>(items);  // 방어적 복사
    }
}
```

---

## 클래스의 접근 수준

### 1. 톱레벨 클래스와 인터페이스

```java
// public 클래스: API의 일부가 됨
public class PublicCalculator {
    public int add(int a, int b) {
        return a + b;
    }
}

// package-private 클래스: 해당 패키지 내부에서만 사용
class InternalCalculator {
    int multiply(int a, int b) {
        return a * b;
    }
}

// 사용 예시
public class CalculatorService {
    private final InternalCalculator internal = new InternalCalculator();
    
    public int calculate(int a, int b, String operation) {
        switch (operation) {
            case "add":
                return new PublicCalculator().add(a, b);
            case "multiply":
                return internal.multiply(a, b);  // 같은 패키지에서만 접근 가능
            default:
                throw new IllegalArgumentException("Unknown operation: " + operation);
        }
    }
}
```

### 2. 한 클래스에서만 사용하는 package-private 톱레벨 클래스나 인터페이스는 이를 사용하는 클래스 안에 private static으로 중첩시키자

```java
// 개선 전
class DatabaseConfig {  // package-private
    String url;
    String username;
    String password;
}

public class DatabaseConnection {
    private DatabaseConfig config;
    
    public DatabaseConnection(DatabaseConfig config) {
        this.config = config;
    }
}
```

```java
// 개선 후
public class DatabaseConnection {
    private final Config config;
    
    public DatabaseConnection(String url, String username, String password) {
        this.config = new Config(url, username, password);
    }
    
    // DatabaseConnection에서만 사용되므로 private static 중첩 클래스로
    private static class Config {
        final String url;
        final String username;
        final String password;
        
        Config(String url, String username, String password) {
            this.url = url;
            this.username = username;
            this.password = password;
        }
    }
}
```

---

## 멤버(필드, 메서드, 중첩 클래스, 중첩 인터페이스)의 접근 수준

### 1. 공개 API가 아닌 모든 멤버는 private으로 만들자

```java
public class UserService {
    private final UserRepository repository;  // private 필드
    private final EmailValidator validator;   // private 필드
    
    public UserService(UserRepository repository, EmailValidator validator) {
        this.repository = repository;
        this.validator = validator;
    }
    
    public User createUser(String name, String email) {
        validateInput(name, email);  // private 메서드 호출
        return repository.save(new User(name, email));
    }
    
    private void validateInput(String name, String email) {  // private 메서드
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("이름은 필수입니다");
        }
        if (!validator.isValid(email)) {
            throw new IllegalArgumentException("유효하지 않은 이메일입니다");
        }
    }
}
```

### 2. 같은 패키지의 다른 클래스가 접근해야 하는 멤버에 한하여 package-private으로 풀어주자

```java
// UserService.java
public class UserService {
    private final UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
    
    // 같은 패키지의 UserController에서 접근할 수 있도록 package-private
    User findUserById(Long id) {
        return repository.findById(id);
    }
    
    public List<User> getAllUsers() {
        return repository.findAll();
    }
}

// UserController.java (같은 패키지)
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findUserById(id);  // package-private 메서드 접근
        return ResponseEntity.ok(user);
    }
}
```

### 3. protected는 신중하게 사용하자

```java
public abstract class Animal {
    private String name;
    
    protected Animal(String name) {  // 하위 클래스에서 호출 가능
        this.name = name;
    }
    
    // 하위 클래스에서 재정의할 수 있도록 protected
    protected void makeSound() {
        System.out.println(name + "이(가) 소리를 냅니다.");
    }
    
    public final void introduce() {  // 하위 클래스에서 변경하면 안 되므로 final
        System.out.println("저는 " + name + "입니다.");
        makeSound();
    }
}

public class Dog extends Animal {
    public Dog(String name) {
        super(name);  // protected 생성자 호출
    }
    
    @Override
    protected void makeSound() {  // protected 메서드 재정의
        System.out.println("멍멍!");
    }
}
```

---

## public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다

### 문제가 있는 예시

```java
// 문제가 있는 클래스
public class Point {
    public double x;  // 위험!
    public double y;  // 위험!
}

// 클라이언트 코드
public class PointTest {
    public static void main(String[] args) {
        Point p = new Point();
        p.x = 3.0;
        p.y = 4.0;
        
        // 외부에서 직접 필드를 변경할 수 있어서 위험
        p.x = Double.NaN;  // 유효하지 않은 값 설정 가능
        p.y = Double.NEGATIVE_INFINITY;
    }
}
```

### 올바른 개선안

```java
public class Point {
    private double x;
    private double y;
    
    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    // 유효성 검사를 포함한 setter
    public void setX(double x) {
        if (Double.isNaN(x) || Double.isInfinite(x)) {
            throw new IllegalArgumentException("유효하지 않은 x 좌표: " + x);
        }
        this.x = x;
    }
    
    public void setY(double y) {
        if (Double.isNaN(y) || Double.isInfinite(y)) {
            throw new IllegalArgumentException("유효하지 않은 y 좌표: " + y);
        }
        this.y = y;
    }
    
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
}
```

---

## 예외적인 경우들

### 1. public static final 필드로 상수 공개하기

```java
public class MathConstants {
    // 기본 타입이나 불변 참조 타입만 public static final로 공개
    public static final double PI = 3.141592653589793;
    public static final String DEFAULT_ENCODING = "UTF-8";
    public static final List<String> SUPPORTED_FORMATS = 
        Collections.unmodifiableList(Arrays.asList("JSON", "XML", "CSV"));
}
```

### 2. 배열은 항상 가변이므로 주의

```java
// 잘못된 예시 - 보안 허점!
public class SecurityHole {
    public static final String[] VALUES = {"A", "B", "C"};  // 위험!
}

// 클라이언트에서 배열 내용을 바꿀 수 있음
SecurityHole.VALUES[0] = "X";  // 원래 배열이 변경됨!
```

```java
// 해결책 1: private 배열 + public 불변 리스트
public class Solution1 {
    private static final String[] PRIVATE_VALUES = {"A", "B", "C"};
    public static final List<String> VALUES = 
        Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
}

// 해결책 2: private 배열 + 복사본을 반환하는 public 메서드
public class Solution2 {
    private static final String[] PRIVATE_VALUES = {"A", "B", "C"};
    
    public static final String[] values() {
        return PRIVATE_VALUES.clone();
    }
}
```

---

## 모듈 시스템의 접근 수준 (Java 9+)

### module-info.java를 통한 패키지 수준 접근 제어

```java
// module-info.java
module com.example.myapp {
    exports com.example.myapp.api;          // 외부에 공개할 패키지
    exports com.example.myapp.util to 
        com.example.clientapp;              // 특정 모듈에만 공개
    
    requires java.base;
    requires java.logging;
}
```

```java
// com.example.myapp.api 패키지 (공개됨)
package com.example.myapp.api;

public class PublicService {  // 다른 모듈에서 접근 가능
    public void doSomething() {
        InternalHelper.help();  // 같은 모듈 내에서는 접근 가능
    }
}
```

```java
// com.example.myapp.internal 패키지 (비공개)
package com.example.myapp.internal;

public class InternalHelper {  // public이지만 모듈 외부에서 접근 불가
    public static void help() {
        System.out.println("내부 도우미 메서드");
    }
}
```

---

## 실무에서의 모범 사례

### 1. Builder 패턴에서의 접근 제어

```java
public class Computer {
    private final String cpu;
    private final String gpu;
    private final int ram;
    private final int storage;
    
    // private 생성자
    private Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.gpu = builder.gpu;
        this.ram = builder.ram;
        this.storage = builder.storage;
    }
    
    // public 접근자들
    public String getCpu() { return cpu; }
    public String getGpu() { return gpu; }
    public int getRam() { return ram; }
    public int getStorage() { return storage; }
    
    // public static 중첩 클래스
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
```

### 2. 팩토리 메서드에서의 접근 제어

```java
public abstract class DatabaseConnection {
    // protected 생성자 - 하위 클래스에서만 생성 가능
    protected DatabaseConnection() {}
    
    // public 팩토리 메서드
    public static DatabaseConnection create(String type, String url) {
        switch (type.toLowerCase()) {
            case "mysql":
                return new MySqlConnection(url);  // package-private 클래스
            case "postgresql":
                return new PostgreSqlConnection(url);  // package-private 클래스
            default:
                throw new IllegalArgumentException("지원하지 않는 데이터베이스 타입: " + type);
        }
    }
    
    public abstract void connect();
    public abstract void disconnect();
}

// package-private 구현 클래스들
class MySqlConnection extends DatabaseConnection {
    private final String url;
    
    MySqlConnection(String url) {  // package-private 생성자
        this.url = url;
    }
    
    @Override
    public void connect() {
        System.out.println("MySQL에 연결: " + url);
    }
    
    @Override
    public void disconnect() {
        System.out.println("MySQL 연결 해제");
    }
}

class PostgreSqlConnection extends DatabaseConnection {
    private final String url;
    
    PostgreSqlConnection(String url) {  // package-private 생성자
        this.url = url;
    }
    
    @Override
    public void connect() {
        System.out.println("PostgreSQL에 연결: " + url);
    }
    
    @Override
    public void disconnect() {
        System.out.println("PostgreSQL 연결 해제");
    }
}
```

---

## 정리

1. **프로그램 요소의 접근성은 가능한 한 최소한으로 하라**
   - 꼭 필요한 것만 골라 최소한의 public API를 설계하자

2. **클래스의 접근성 관리**
   - 톱레벨 클래스와 인터페이스는 package-private으로 만들 수 있다면 그렇게 하자
   - 한 클래스에서만 사용하는 톱레벨 클래스는 private static 중첩 클래스로 만들자

3. **멤버의 접근성 관리**
   - 공개 API가 아닌 모든 멤버는 private으로 만들자
   - 같은 패키지에서 접근해야 할 멤버만 package-private으로 풀어주자
   - protected는 신중하게 사용하자

4. **public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다**
   - public 가변 필드를 갖는 클래스는 일반적으로 스레드 안전하지 않다
   - 상수라면 public static final 필드로 공개해도 좋다

5. **배열 필드는 항상 private으로 만들거나 불변으로 만들어라**
   - public static final 배열 필드나 이런 필드를 반환하는 접근자 메서드를 제공해서는 안 된다

접근 제어는 정보 은닉의 핵심이며, 잘 설계된 모듈은 구현 세부사항을 숨기고 API를 통해서만 다른 모듈과 소통한다!