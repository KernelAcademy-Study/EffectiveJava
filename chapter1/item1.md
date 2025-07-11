# [Item 1] 생성자 대신 정적 팩토리 메서드를 고려하라

## 서론
자바 클래스의 인스턴스를 만들 때 보통은 **public 생성자**를 사용한다. 하지만 정적 팩터리 메서드(static factory method)를 활용하면 더 유연하고 표현력 있는 API를 설계할 수 있다.  
생성자 대신 정적 팩터리 메서드를 선택했을 때 얻을 수 있는 주요 이점 다섯 가지와 주의할 점 두 가지를 살펴보려고 한다. 

---

## 장점 1: 메서드 이름으로 의도를 드러낼 수 있다
일반 생성자는 클래스 이름과 매개변수만으로 역할을 유추해야 하지만, 팩터리 메서드는 **의미 있는 이름**을 붙여 반환 객체의 성격을 명확히 전달할 수 있다.

```java
public class User {
    private final String name;
    private final String email;
    private User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    // 정적 팩터리 메서드: 이메일 기반 생성
    public static User fromEmail(String email) {
        String namePart = email.substring(0, email.indexOf('@'));
        return new User(namePart, email);
    }

    // 정적 팩터리 메서드: 사용자명 기반 생성
    public static User fromName(String name) {
        String email = name.toLowerCase() + "@example.com";
        return new User(name, email);
    }
}

// 사용 예시
User u1 = User.fromEmail("alice@example.com");  
User u2 = User.fromName("Bob");  
```
---

## 장점 2: 매번 새 인스턴스를 만들지 않아도 된다
불변 객체의 경우 미리 생성해 두거나 필요할 때 캐시에서 가져오는 식으로 객체 생성을 제어할 수 있다. 이를 통해 불필요한 쓰레기 수집을 줄이고 성능을 높일 수 있다.

```java
public final class Color {
private final int red, green, blue;
private static final Map<String, Color> CACHE = new HashMap<>();

    private Color(int r, int g, int b) {
        this.red = r; this.green = g; this.blue = b;
    }

    // 캐시를 사용해 동일 색상 중복 생성 방지
    public static Color of(int r, int g, int b) {
        String key = r + "," + g + "," + b;
        return CACHE.computeIfAbsent(key, k -> new Color(r, g, b));
    }
}

// 사용 예시
Color c1 = Color.of(255, 0, 0);
Color c2 = Color.of(255, 0, 0);
// c1 == c2 가 true
```
---

## 장점 3: 구현 클래스를 숨기면서 반환 타입 유연성 확보
팩터리 메서드는 반환 타입으로 인터페이스나 추상 클래스를 지정하고, 내부에서 다양한 구현 클래스를 선택해 넘길 수 있다. 사용자에게는 깔끔한 API만 노출된다.

```java
public interface Payment {
    void pay(int amount);
}

class CreditCardPayment implements Payment {
    public void pay(int amount) { /* 카드 결제 로직 */ }
}

class PayPalPayment implements Payment {
    public void pay(int amount) { /* 페이팔 결제 로직 */ }
}

public class PaymentFactory {
    public static Payment create(String method) {
        if ("CARD".equalsIgnoreCase(method)) {
            return new CreditCardPayment();
        } else {
            return new PayPalPayment();
        }
    }
}

// 사용 예시
Payment p = PaymentFactory.create("CARD");
p.pay(10000);
```
---

## 장점 4: 조건에 따라 서로 다른 하위 클래스 인스턴스 반환

매개변수나 환경 설정에 따라 알맞은 구현체를 고를 수 있다. 예를 들어, 컬렉션 크기에 따라 서로 다른 내부 구조를 가진 클래스 인스턴스를 제공하는 식이다.

```java
public abstract class IntSet {
    public abstract boolean contains(int x);

    public static IntSet of(int... elements) {
        if (elements.length < 10) {
            return new SortedArraySet(elements);
        } else {
            return new BitVectorSet(elements);
        }
    }
}

class SortedArraySet extends IntSet { /* 작은 집합용 구현 */ }
class BitVectorSet   extends IntSet { /* 큰 집합용 구현 */ }

// 사용 예시
IntSet small = IntSet.of(1, 2, 3);
IntSet large = IntSet.of( /* 100개 이상 */ );
```

---
## 장점 5: 아직 존재하지 않은 클래스도 반환 대상으로 설계 가능

코드를 작성하는 시점에 클래스가 없더라도, 나중에 서비스 제공자 프레임워크처럼 플러그인 형태로 구현체를 추가할 수 있다. JDBC, ServiceLoader 등이 이 원리를 활용한다.
```java
public interface Storage {
    void save(String data);
}

public class StorageFactory {
    public static Storage getStorage() {
        // 시스템 속성이나 설정 파일로부터 구현체 클래스 이름을 읽어와 동적으로 로딩
        String impl = System.getProperty("storage.impl"); 
        try {
            Class<?> cls = Class.forName(impl);
            return (Storage) cls.getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new IllegalStateException("스토리지 구현체 로드 실패", e);
        }
    }
}

// 사용 예시 (JVM 옵션: -Dstorage.impl=com.example.S3Storage)
Storage s = StorageFactory.getStorage();
s.save("중요한 데이터");
```
---

### 단점 1: 상속이 제한된다

하위 클래스를 만들려면 public 또는 protected 생성자가 필요하다. 팩터리 메서드만 제공하면 상속을 막아야 할 때는 오히려 장점이지만, 라이브러리 확장이 필요할 땐 제약이 될 수 있다.

### 단점 2: API 탐색성이 떨어질 수 있다

생성자가 명시적으로 보이지 않으므로 사용자가 팩터리 메서드를 찾기 어려울 수 있다. 이 경우 문서화와 메서드 이름 네이밍 컨벤션(of, from, getInstance 등)을 잘 지켜야 한다.
