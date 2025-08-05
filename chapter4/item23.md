# [Item 23] 태그 달린 클래스보다는 클래스 계층구조를 활용하라

## 서론
두 가지 이상의 의미를 표현할 수 있으며, 그중 현재 표현하는 의미를 태그 값으로 알려주는 클래스를 본 적이 있을 것이다. 이런 **태그 달린 클래스(tagged class)**는 장황하고, 오류를 내기 쉽고, 비효율적이다. 자바와 같은 객체 지향 언어는 타입 하나로 다양한 의미의 객체를 표현하는 훨씬 나은 수단을 제공한다. 바로 **클래스 계층구조**다.

---

## 태그 달린 클래스의 문제점

### 문제가 있는 태그 달린 클래스

```java
// 태그 달린 클래스 - 클래스 계층구조보다 훨씬 나쁘다!
public class Figure {
    enum Shape { RECTANGLE, CIRCLE }
    
    // 태그 필드 - 현재 모양을 나타낸다
    final Shape shape;
    
    // 다음 필드들은 모양이 사각형(RECTANGLE)일 때만 쓰인다
    double length;
    double width;
    
    // 다음 필드는 모양이 원(CIRCLE)일 때만 쓰인다
    double radius;
    
    // 원용 생성자
    public Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }
    
    // 사각형용 생성자
    public Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }
    
    double area() {
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
    
    double perimeter() {
        switch(shape) {
            case RECTANGLE:
                return 2 * (length + width);
            case CIRCLE:
                return 2 * Math.PI * radius;
            default:
                throw new AssertionError(shape);
        }
    }
}

// 사용 예시
public class TaggedClassExample {
    public static void main(String[] args) {
        Figure circle = new Figure(5);
        Figure rectangle = new Figure(4, 6);
        
        System.out.println("원의 넓이: " + circle.area());
        System.out.println("사각형의 넓이: " + rectangle.area());
        
        // 문제점들:
        // 1. circle.length를 실수로 접근할 수 있음 (잘못된 필드)
        // 2. 새로운 도형을 추가하려면 모든 switch 문을 수정해야 함
        // 3. 인스턴스가 불필요한 필드들까지 메모리에 저장
    }
}
```

### 태그 달린 클래스의 단점들

1. **열거 타입 선언, 태그 필드, switch 문 등 쓸데없는 코드가 많다**
2. **여러 구현이 한 클래스에 혼재하여 가독성이 나쁘다**
3. **다른 의미를 위한 코드도 언제나 함께 하니 메모리도 많이 먹는다**
4. **필드들을 final로 선언하려면 해당 의미에 쓰이지 않는 필드들까지 생성자에서 초기화해야 한다**
5. **또 다른 의미를 추가하려면 코드를 수정해야 한다**
6. **인스턴스의 타입만으로는 현재 나타내는 의미를 알 길이 전혀 없다**

---

## 클래스 계층구조로 변환

### 올바른 해결책: 추상 클래스와 구체 클래스들

```java
// 태그 달린 클래스를 클래스 계층구조로 변환
abstract class Figure {
    abstract double area();
    abstract double perimeter();
    
    // 공통 기능이 있다면 여기에 구현
    public void printInfo() {
        System.out.printf("넓이: %.2f, 둘레: %.2f%n", area(), perimeter());
    }
}

// 원을 나타내는 구체 클래스
class Circle extends Figure {
    final double radius;
    
    Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    double area() {
        return Math.PI * radius * radius;
    }
    
    @Override
    double perimeter() {
        return 2 * Math.PI * radius;
    }
    
    // 원에만 특화된 기능
    public double getDiameter() {
        return 2 * radius;
    }
}

// 사각형을 나타내는 구체 클래스
class Rectangle extends Figure {
    final double length;
    final double width;
    
    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }
    
    @Override
    double area() {
        return length * width;
    }
    
    @Override
    double perimeter() {
        return 2 * (length + width);
    }
    
    // 사각형에만 특화된 기능
    public boolean isSquare() {
        return length == width;
    }
}

// 사용 예시
public class HierarchyExample {
    public static void main(String[] args) {
        Figure circle = new Circle(5);
        Figure rectangle = new Rectangle(4, 6);
        
        // 다형성 활용
        Figure[] figures = {circle, rectangle};
        for (Figure figure : figures) {
            figure.printInfo();
        }
        
        // 타입 안전성 확보
        if (circle instanceof Circle) {
            Circle c = (Circle) circle;
            System.out.println("원의 지름: " + c.getDiameter());
        }
        
        if (rectangle instanceof Rectangle) {
            Rectangle r = (Rectangle) rectangle;
            System.out.println("정사각형인가? " + r.isSquare());
        }
    }
}
```

---

## 계층구조의 장점

### 1. 타입 안전성과 확장성

```java
// 새로운 도형 추가가 매우 쉬움
class Triangle extends Figure {
    final double a, b, c;  // 세 변의 길이
    
    Triangle(double a, double b, double c) {
        if (a + b <= c || b + c <= a || c + a <= b) {
            throw new IllegalArgumentException("삼각형의 조건에 맞지 않습니다");
        }
        this.a = a;
        this.b = b;
        this.c = c;
    }
    
    @Override
    double area() {
        // 헤론의 공식 사용
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
    
    @Override
    double perimeter() {
        return a + b + c;
    }
    
    // 삼각형만의 특별한 메서드
    public String getTriangleType() {
        if (a == b && b == c) return "정삼각형";
        if (a == b || b == c || c == a) return "이등변삼각형";
        return "일반삼각형";
    }
}

// 정사각형 (사각형의 특수한 경우)
class Square extends Rectangle {
    Square(double side) {
        super(side, side);
    }
    
    // 정사각형만의 편의 메서드
    public double getSide() {
        return length;  // length == width이므로 어느 것이든 상관없음
    }
    
    @Override
    public boolean isSquare() {
        return true;  // 항상 정사각형
    }
}

// 확장된 사용 예시
public class ExtendedHierarchyExample {
    public static void main(String[] args) {
        Figure[] figures = {
            new Circle(3),
            new Rectangle(4, 5),
            new Square(4),
            new Triangle(3, 4, 5)
        };
        
        System.out.println("=== 모든 도형 정보 ===");
        for (Figure figure : figures) {
            System.out.println(figure.getClass().getSimpleName() + ": ");
            figure.printInfo();
            System.out.println();
        }
        
        // 특화된 기능 사용
        System.out.println("=== 특화된 기능 ===");
        for (Figure figure : figures) {
            if (figure instanceof Triangle) {
                Triangle t = (Triangle) figure;
                System.out.println("삼각형 타입: " + t.getTriangleType());
            } else if (figure instanceof Square) {
                Square s = (Square) figure;
                System.out.println("정사각형 한 변의 길이: " + s.getSide());
            } else if (figure instanceof Circle) {
                Circle c = (Circle) figure;
                System.out.println("원의 지름: " + c.getDiameter());
            }
        }
    }
}
```

### 2. 공통 기능의 체계적 관리

```java
// 더 복잡한 계층구조 예시
abstract class Shape {
    protected String color = "투명";
    protected boolean filled = false;
    
    public Shape() {}
    
    public Shape(String color, boolean filled) {
        this.color = color;
        this.filled = filled;
    }
    
    // 모든 도형이 가져야 할 추상 메서드
    public abstract double area();
    public abstract double perimeter();
    public abstract void draw();  // 그리기 기능
    
    // 공통 기능들
    public String getColor() { return color; }
    public void setColor(String color) { this.color = color; }
    
    public boolean isFilled() { return filled; }
    public void setFilled(boolean filled) { this.filled = filled; }
    
    public void printShapeInfo() {
        System.out.printf("%s (색상: %s, 채움: %s, 넓이: %.2f, 둘레: %.2f)%n", 
            getClass().getSimpleName(), color, filled ? "예" : "아니오", 
            area(), perimeter());
    }
}

// 향상된 원 클래스
class EnhancedCircle extends Shape {
    private final double radius;
    
    public EnhancedCircle(double radius) {
        this.radius = radius;
    }
    
    public EnhancedCircle(double radius, String color, boolean filled) {
        super(color, filled);
        this.radius = radius;
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
    
    @Override
    public void draw() {
        System.out.println("반지름 " + radius + "인 원을 그립니다");
        if (filled) {
            System.out.println(color + " 색으로 채웁니다");
        }
    }
    
    public double getRadius() { return radius; }
}

// 향상된 사각형 클래스
class EnhancedRectangle extends Shape {
    protected final double width;
    protected final double height;
    
    public EnhancedRectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    public EnhancedRectangle(double width, double height, String color, boolean filled) {
        super(color, filled);
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double area() {
        return width * height;
    }
    
    @Override
    public double perimeter() {
        return 2 * (width + height);
    }
    
    @Override
    public void draw() {
        System.out.printf("%.1f x %.1f 사각형을 그립니다%n", width, height);
        if (filled) {
            System.out.println(color + " 색으로 채웁니다");
        }
    }
    
    public double getWidth() { return width; }
    public double getHeight() { return height; }
}
```

---

## 실무에서의 활용 예시

### 1. 결제 시스템에서의 적용

```java
// 잘못된 태그 달린 결제 클래스
class BadPayment {
    enum PaymentType { CREDIT_CARD, PAYPAL, BANK_TRANSFER }
    
    final PaymentType type;
    
    // 신용카드용 필드들
    String cardNumber;
    String expiryDate;
    String cvv;
    
    // 페이팔용 필드들  
    String paypalEmail;
    String paypalToken;
    
    // 계좌이체용 필드들
    String bankAccount;
    String routingNumber;
    
    // 생성자들과 메서드들...
    public boolean processPayment(double amount) {
        switch (type) {
            case CREDIT_CARD:
                return processCreditCard(amount);
            case PAYPAL:
                return processPaypal(amount);
            case BANK_TRANSFER:
                return processBankTransfer(amount);
            default:
                throw new AssertionError(type);
        }
    }
    
    // 각 결제 방식별 처리 메서드들...
}
```

```java
// 올바른 계층구조 기반 결제 시스템
abstract class Payment {
    protected final double amount;
    protected final String merchantId;
    
    protected Payment(double amount, String merchantId) {
        this.amount = amount;
        this.merchantId = merchantId;
    }
    
    public abstract boolean process();
    public abstract String getPaymentMethod();
    
    protected void logTransaction(boolean success) {
        System.out.printf("[%s] %s 결제 %s: %.2f원%n", 
            getPaymentMethod(), merchantId, 
            success ? "성공" : "실패", amount);
    }
}

class CreditCardPayment extends Payment {
    private final String cardNumber;
    private final String expiryDate;
    private final String cvv;
    
    public CreditCardPayment(double amount, String merchantId, 
                           String cardNumber, String expiryDate, String cvv) {
        super(amount, merchantId);
        this.cardNumber = cardNumber;
        this.expiryDate = expiryDate;
        this.cvv = cvv;
    }
    
    @Override
    public boolean process() {
        // 신용카드 결제 로직
        boolean isValid = validateCard();
        boolean success = isValid && chargeCard();
        logTransaction(success);
        return success;
    }
    
    @Override
    public String getPaymentMethod() {
        return "신용카드";
    }
    
    private boolean validateCard() {
        // 카드 유효성 검사
        return cardNumber.length() >= 16 && !expiryDate.isEmpty() && !cvv.isEmpty();
    }
    
    private boolean chargeCard() {
        // 실제 카드 결제 처리
        System.out.println("카드 번호 ****" + cardNumber.substring(cardNumber.length() - 4) + "로 결제 처리");
        return true;  // 성공했다고 가정
    }
}

class PayPalPayment extends Payment {
    private final String email;
    private final String token;
    
    public PayPalPayment(double amount, String merchantId, String email, String token) {
        super(amount, merchantId);
        this.email = email;
        this.token = token;
    }
    
    @Override
    public boolean process() {
        boolean isAuthenticated = authenticateWithPayPal();
        boolean success = isAuthenticated && processPayPalPayment();
        logTransaction(success);
        return success;
    }
    
    @Override
    public String getPaymentMethod() {
        return "페이팔";
    }
    
    private boolean authenticateWithPayPal() {
        System.out.println(email + " 계정으로 페이팔 인증 중...");
        return !token.isEmpty();
    }
    
    private boolean processPayPalPayment() {
        System.out.println("페이팔로 결제 처리 중...");
        return true;
    }
}

// 결제 처리기
class PaymentProcessor {
    public void processPayments(Payment... payments) {
        double totalAmount = 0;
        int successCount = 0;
        
        for (Payment payment : payments) {
            if (payment.process()) {
                totalAmount += payment.amount;
                successCount++;
            }
        }
        
        System.out.printf("%n총 %d건 중 %d건 성공, 총 결제액: %.2f원%n", 
            payments.length, successCount, totalAmount);
    }
}

// 사용 예시
public class PaymentSystemExample {
    public static void main(String[] args) {
        PaymentProcessor processor = new PaymentProcessor();
        
        Payment[] payments = {
            new CreditCardPayment(50000, "MERCHANT_001", "1234567890123456", "12/25", "123"),
            new PayPalPayment(75000, "MERCHANT_001", "user@example.com", "paypal_token_123"),
            new CreditCardPayment(30000, "MERCHANT_002", "9876543210987654", "06/24", "456")
        };
        
        processor.processPayments(payments);
    }
}
```

### 2. 로깅 시스템에서의 적용

```java
// 로그 레벨별 처리를 위한 계층구조
abstract class LogMessage {
    protected final String message;
    protected final long timestamp;
    protected final String source;
    
    protected LogMessage(String message, String source) {
        this.message = message;
        this.source = source;
        this.timestamp = System.currentTimeMillis();
    }
    
    public abstract void log();
    public abstract String getLevel();
    
    protected String formatTimestamp() {
        return java.time.Instant.ofEpochMilli(timestamp).toString();
    }
    
    protected String getBasicFormat() {
        return String.format("[%s] %s - %s: %s", 
            formatTimestamp(), getLevel(), source, message);
    }
}

class InfoMessage extends LogMessage {
    public InfoMessage(String message, String source) {
        super(message, source);
    }
    
    @Override
    public void log() {
        System.out.println(getBasicFormat());
    }
    
    @Override
    public String getLevel() {
        return "INFO";
    }
}

class WarningMessage extends LogMessage {
    private final String recommendation;
    
    public WarningMessage(String message, String source, String recommendation) {
        super(message, source);
        this.recommendation = recommendation;
    }
    
    @Override
    public void log() {
        System.err.println("⚠️ " + getBasicFormat());
        if (recommendation != null) {
            System.err.println("   권장사항: " + recommendation);
        }
    }
    
    @Override
    public String getLevel() {
        return "WARNING";
    }
}

class ErrorMessage extends LogMessage {
    private final Exception exception;
    
    public ErrorMessage(String message, String source, Exception exception) {
        super(message, source);
        this.exception = exception;
    }
    
    @Override
    public void log() {
        System.err.println("❌ " + getBasicFormat());
        if (exception != null) {
            System.err.println("   예외: " + exception.getClass().getSimpleName() 
                + " - " + exception.getMessage());
        }
        
        // 에러는 파일에도 저장
        logToFile();
    }
    
    @Override
    public String getLevel() {
        return "ERROR";
    }
    
    private void logToFile() {
        // 실제로는 파일에 저장하는 로직
        System.out.println("   (에러 로그가 파일에 저장되었습니다)");
    }
}

// 로거 클래스
class Logger {
    private final String source;
    
    public Logger(String source) {
        this.source = source;
    }
    
    public void info(String message) {
        new InfoMessage(message, source).log();
    }
    
    public void warning(String message, String recommendation) {
        new WarningMessage(message, source, recommendation).log();
    }
    
    public void error(String message, Exception exception) {
        new ErrorMessage(message, source, exception).log();
    }
}

// 사용 예시
public class LoggingSystemExample {
    public static void main(String[] args) {
        Logger logger = new Logger("PaymentService");
        
        logger.info("결제 서비스가 시작되었습니다");
        logger.warning("결제 처리 시간이 평소보다 오래 걸리고 있습니다", 
                      "네트워크 상태를 확인해보세요");
        logger.error("결제 처리 중 오류가 발생했습니다", 
                    new RuntimeException("Database connection failed"));
    }
}
```

---

## 정리

1. **태그 달린 클래스를 써야 하는 상황은 거의 없다**
   - 새로운 클래스를 작성하는데 태그 필드가 등장한다면 태그를 없애고 계층구조로 대체하는 방법을 생각해보자

2. **기존 클래스가 태그 필드를 사용하고 있다면 계층구조로 리팩터링하는 것을 고민해보자**
   - 리팩터링할 때는 기존 코드와의 호환성을 고려해야 한다

3. **클래스 계층구조의 장점들**
   - 간결하고 명확하다
   - 쓸데없는 코드가 모두 사라진다
   - 각 의미를 독립적인 클래스에 담아 관련 없던 데이터 필드를 모두 제거할 수 있다
   - 살아남은 필드들은 모두 final이다
   - 각 클래스의 생성자가 모든 필드를 남김없이 초기화하고 추상 메서드를 모두 구현했는지 컴파일러가 확인해준다
   - 루트 클래스의 코드를 건드리지 않고도 다른 프로그래머들이 독립적으로 계층구조를 확장하고 함께 사용할 수 있다
   - 타입이 의미별로 따로 존재하니 변수의 의미를 명시하거나 제한할 수 있고, 또 특정 의미만 매개변수로 받을 수 있다

4. **계층구조는 타입 사이의 자연스러운 계층 관계를 반영할 수 있어서 유연성은 물론 컴파일타임 타입 검사 능력을 높여준다는 장점도 있다**

태그 달린 클래스는 클래스 계층구조를 어설프게 흉내 낸 아류일 뿐이다!