# [Item 2] 생성자에 매개변수가 많다면 빌더를 고려하라

## 점층적 생성자 패턴 (Telescoping Constructor Pattern)
정적 팩터리와 생성자에는 선택적 매개변수가 많을 떄 적절히 대응하기 어렵다는 제약이 존재한다. 이 경우 점층적 생성자 패턴이 자주 사용되었다.  
점층적 생성자 패턴은 필수 매개변수를 포함하여, 선택적 매개변수를 1개 ~ n개까지 받는 생성자를 n개까지 늘려가는 방식이다.

```java
public class Room {
    private final int roomSize;         // 방 넓이,     필수
    private final int bathroomSize;     // 화장실 넓이,  필수
    private final int bedSize;          // 침대 넓이,    선택
    private final int tvNum;            // Tv 대수,     선택
    private final int foods;            // 보유 음식,    선택

    public Room (int roomSize, int bathroomSize) {
        this(roomSize, bathroomSize, 0)
    }

    public Room (int roomSize, int bathroomSize, int bedSize) {
        this(roomSize, bathroomSize, bedSize, 0)
    }

    public Room (int roomSize, int bathroomSize, int bedSize, int tvNum) {
        this(roomSize, bathroomSize, bedSize, tvNum, 0)
    }

    public Room (int roomSize, int bathroomSize, int bedSize, int tvNum, int foods) {
        this.roomSize = roomSize;
        this.bathroomSize = bathroomSize;
        this.bedSize = bedSize;
        this.tvNum = tvNum;
        this.foods = foods;
    }
}
```

### 단점
1. 사용자가 설정하길 원치 않는 매개변수까지 포함하기 쉬운데, 그런 매개변수에도 값을 지정해줘야 한다. (ex. this(roomSize, bathroomSize, 0))
2. 매개변수의 개수가 많아지면 클라이언트 코드를 작성하거나 읽기 어렵다.

---

## 자바빈즈 패턴(JavaBeans Pattern)
매개변수가 없는 생성자로 객체를 만든 후, 세터 메서드들을 호출해 매개변수의 값을 설정하는 방식이다.

```java
public class Room {
    private int roomSize = -1;         // 필수, 기본값 없음
    private int bathroomSize = -1;     // 필수, 기본값 없음
    private int bedSize = 0;
    private int tvNum = 0;
    private int foods = 0;

    public Room() { }

    public void setRoomSize(int roomSize) { this.roomSize = roomSize; }
    public void setBathroomSize(int bathroomSize) { this.bathroomSize = bathroomSize; }
    public void setBedSize(int bedSize) { this.bedSize = bedSize; }
    public void setTvNum(int tvNum) { this.tvNum = tvNum; }
    public void setFoods(int foods) { this.foods = foods; }
}
```

### 장점
1. 코드가 길어지긴 했지만 인스턴스를 만들기 쉽다.
2. 읽기 쉬운 코드로 발전이 가능하다.

### 단점
1. 객체 하나를 만들기 위해서는 메서드를 여러 개 호출해야 한다.
2. 객체가 완전히 생성되기 전까지는 일관성이 무너진 상태에 놓이게 된다.
3. 이로 인해 클래스를 불변으로 만들 수 없다.

---

## 빌더 패턴 (Builder Pattern)
점층적 생성자 패턴의 안전성과 자바빈즈 패턴의 가독성을 겸비한 패턴이다.  
클라이언트는 필요한 객체를 직접 만드는 대신, 필수 매개변수만으로 생성자를 호출해 **빌더 객체**를 얻는다.  
그 후, 빌더 객체가 제공하는 **세터 메서드들로 원하는 선택 매개변수들을 설정**한다.  
마지막으로 매개변수가 없는 build 메서드를 호출해 필요한(불변인) 객체를 얻는다.

```java
public class Room {
    private final int roomSize;
    private final int bathroomSize;
    private final int bedSize;
    private final int tvNum;
    private final int foods;

    public static class Builder {
        // 필수 매개변수
        private final int roomSize;
        private final int bathroomSize;

        // 선택 매개변수 - 기본값으로 초기화
        private final int bedSize = 0;
        private final int tvNum = 0;
        private final int foods = 0;

        public Builder(int roomSize, int bathroomSize) {
            this.roomSize = roomSize;
            this.bathroomSize = bathroomSize;
        }

        public Builder bedSize(int val) { bedSize = val;    return this; }
        public Builder tvNum(int val) { tvNum = val;    return this; }
        public Builder foods(int val) { foods = val;    return this; }

        public Room build() {
            return new Room(this);
        }

        private Room(Builder builder) {
            roomSize = builder.roomSize;
            bathroomSize = builder.bathroomSize;
            bedSize = builder.bedSize;
            tvNum = builder.tvNum;
            foods = builder.foods;
        }
    }
```

### 장점
1. 클래스는 불변이며, 모든 매개변수의 기본값들을 한곳에 모아둘 수 있다.
2. 세터 메서드들은 빌더 자신을 반환하기 때문에 연쇄 호출이 가능하다.
```java
Room myRoom = new Room.Builder(7, 2).bedSize(1).tvNum(1).foods(12).build();
```
3. 클라이언트 코드는 읽고 쓰기 쉽다. => 파이썬과 스칼라에 명명된 선택적 매개변수를 흉내낸 것

---

## 빌더 패턴과 계층적으로 설계된 클래스
추상 클래스는 추상 빌더를, 구체 클래스는 구체 빌더를 갖게 한다.  
아래에는 추상 클래스 - RoomStyler, 구체 클래스 - OneRoom, Officetel로 정의하였다.  
RoomStyler에는 가구를 추가하는 addFurniture() 함수가 있다.  
OneRoom는 월세를 임대인과 합의하여 지정하는 rental 매개변수를 필수로 받는다.  
OfficeTel는 복층인지 확인하는 isDouble 매개변수를 필수로 받는다.  

```java

// RoomStyler 클래스
public class RoomStyler {
    public enum Furniture { TV, BED, DESK, HANGER }
    final Set<Furniture> furnitures;

    abstract static class Builder<T extends Builder<T>> {
        public T addFurniture(Furniture furniture) {
            furnitures.add(furniture);
            return self();
        }

        abstract RoomStyler build();

        // 하위 클래스는 재정의하여 자기 자신을 반환하도록 한다. -> 연쇄 호출을 위해서
        protected abstract T self();
    }

    // 어떤 builder든 상관이 없고, furnitures만 클론하면 되기 때문에
    // 복잡하게 Builder<T>를 쓰지 않고 와일드카드 사용
    RoomStyler(Builder<?> builder) {
        furnitures = builder.furnitures.clone();
    }
}

// OneRoom 클래스
public class OneRoom extends RoomStyler {
    public enum Rental { FIFTY, HUNDRED, THOUSAND }
    private final Rental rental

    // 자기 자신을 타입 인자로 전달한 재귀적 제네릭 타입
    // 상위 추상 빌더 클래스에서 정의한 메서드들이 타입 안정성을 유지하면서 메서드 체이닝이 가능
    public static class Builder extends RoomStyler.Builder<Builder> {
        private final Rental rental;

        public Builder (Rental rental) {
            this.rental = rental;
        }

        @Override
        public OneRoom build() { return new OneRoom(this); }

        @Override
        protected Builder self() { return this; }
    }

    private OneRoom(Builder builder) {
        super(builder);
        rental = builder.rental;
    }
}

public class OfficeTel extends RoomStyler {
    private final boolean isDouble;

    public static class Builder extends RoomStyler.Builder<Builder> {
        private boolean isDouble = false;

        public Builder isDouble(boolean isDouble) { 
            this.isDouble = isDouble; return this;
        }

        @Override
        public OfficeTel build() { return new OfficeTel(this); }

        @Override
        protected Builder self() { return this; }
    }

    private OfficeTel(Builder builder) {
        super(builder);
        isDouble = builder.isDouble;
    }
}
```

### 코드의 흐름
1. 각 하위 클래스의 빌더가 정의한 build 메서드는 해당하는 구체 하위 클래스를 반환하도록 선언한다.
2. OneRoom.Builder는 OneRoom을, OfficeTel.Builder는 OfficeTel을 반환한다.
3. 이처럼 하위 클래스의 메서드가 상위 클래스의 메서드가 정의한 타입이 아닌, 그 하위 타입을 반환하는 것을 공변환 타이핑이라 한다.
```java
OneRoom oneRoom = new OneRoom.Builder(FIFTY).addFurniture(TV).addFurniture(BED).build();
OfficeTel officeTel = new OfficeTel.Builder().isDouble(true).addFurniture(HANGER).build();
```

---

## 빌더 패턴의 장단점

### 장점
1. 빌더 하나로 여러 객체를 순회하면서 만들 수 있다.
2. 빌더에 넘기는 매개변수에 따라 다른 객체를 만들 수도 있다.
3. 객체마다 부여되는 일련번호와 같은 특정 필드는 빌더가 알아서 채우도록 할 수 있다.  
=> **상당히 유연하다.**

### 단점
1. 객체를 만들려면 그에 앞서 빌더부터 만들어야 하기 때문에, 성능에 민감한 상황일 경우 문제가 될 수 있다.
2. 점층적 생성자 패턴보다는 코드가 장황해서 매개변수가 4개 이상은 되어야 값어치를 한다.

---

## 핵심 정리 
**생성자나 정적 팩터리가 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하는 것이 낫다.**