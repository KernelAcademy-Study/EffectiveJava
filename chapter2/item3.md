# [Item 3] 생성자 대신 정적 팩토리 메서드를 고려하라

## 서론
싱글턴(singleton)이란 인스턴스를 오직 하나만 생성할 수 있는 클래스를 말한다.
 
클래스를 싱글턴으로 만들면 이를 사용하는 클라이언트를 테스트하기가 어려워질 수 있다. 타입을 인터페이스로 정의한 다음 그 인터페이스를 구현해서 만든 싱글턴이 아니라면 싱글턴 인스턴스를 가짜(mock) 구현으로 대체할 수 없기 때문이다.

---

싱글턴을 만드는 방식은 보통 둘 중 하나다. 두 방식 모두 생성자는 private으로 감춰두고, 유일한 인스턴스에 접근할 수 있는 수단으로 public static 멤버를 하나 마련해둔다.

## 1. public static final 필드 방식의 싱글턴
```java
public class Elvis{
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }

    public void leaceTheBuilding() { ... }
}
```

 private 생성자는 public static final 필드인 Elvis.INSTANCE를 초기화 할 때 딱 한 번만 호출된다. public이나 protected 생성자가 없으므로 Elvis 클래스가 초기화 될 때 만들어진 인스턴스가 전체 시스템에서 하나뿐임이 보장된다.

 하지만 리플렉션 API로 private 생성자를 호출할 수 있는 취약점이 있다. 따라서 이러한 공격을 방어하려면 생성자를 수정하여 두 번째 객체가 생성되려 할 때 예외를 던져 방어할 수 있다.

__장점__
 - 해당 클래스가 싱글턴임이 API에 명백히 들어난다. public static 필드가 final이니 절대로 다른 객체를 참조할 수 없다.
 - 간결하다.

---

 ## 2. 정적 팩터리 방식의 싱글턴

 정적 팩터리 메서드를 public static 멤버로 제공하는 방법이다.

 ```java
 public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() { ... }
 }
 ```

 Elvis.getInstance는 항상 같은 객체의 참조를 반환하므로 제2의 Elvis 인스턴스란 결코 만들어지지 않는다. 하지만 이 역시 리플렉션을 통한 예외는 똑같이 적용해야 한다.

__장점__
 - API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다. 유일한 인스턴스를 반환하던 팩터리 메서드가 호출하는 스레드별로 다른 인스턴스를 넘겨주게 할 수 있다.
 - 원한다면 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있다.
 - 정적 팩터리의 메서드 참조를 공급자로 사용할 수 있다. Elvics::getInstance를 Supplier<Elvis> 와 같이 사용할 수 있다.


> 둘 중 하나의 방식으로 만든 싱글턴 클래스를 직렬화하려면 단순히 Serializable을 구현한다고 선언하는 것만으로는 부족하다. 모든 인스턴스 필드를 일시적이라고 선언하고 readResolve 메서드를 제공해야 한다. 이렇게 하지 않으면 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어진다.

<br>

```java
// 싱글턴임을 보장해주는 readResolve 메서드
private Object readResolve() {
    // '진짜' Elvis를 반환하고, 가짞 Elvis는 가비지 컬렉터에 맡긴다.
    return INSTANCE;
}
```

---

## 3. 열거 타입 방식의 싱글턴

```java
public enum ELvis {
    INSTANCE;

    public void leaveTheBuilding() { ... }
}
```

public 필드 방식과 비슷하지만, 더 간결하고, 추가 노력 없이 직렬화할 수 있고, 심지어 아죽 복잡한 직렬화 상황이나 리플렉션 공격에서도 제2의 인스턴스가 생기는 일을 완벽히 막아준다.

__단점__
- 만들려는 싱글턴이 Enum 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없다. (열거 타입이 다른 인터페이스를 구현하도록 선언할 수는 있다.)
