## [Item 33] 타입 안전 이종 컨테이너를 고려하라

### 서론

여러 타입의 값을 한 곳에 담아야 할 때 보통 Map<String, Object>를 쓰지만, 이 방식은 캐스팅 실수와 런타임 오류 위험이 크다. 
키에 타입 정보를 싣는 방식으로 바꾸면 컴파일 타임에 타입을 보장하면서도 서로 다른 타입을 한 컨테이너에 담을 수 있다. 
이를 타입 안전 이종 컨테이너(Type-Safe Heterogeneous Container) 패턴이라 한다. 
핵심은 Class<T> 같은 타입 토큰을 키로 쓰고, put/get에서 키의 타입으로 안전 캐스팅하는 것이다.

핵심 아이디어 (요약)

- 컨테이너가 아니라 키를 제네릭화한다: Map<Class<?>, Object>
- 저장/조회 시 Class<T>.cast(...)로 즉시 타입 검증
- 결과: 여러 타입을 한 컨테이너에서 타입 안전하게 다룸

---

### 장점

1) **여러 타입을 한 컨테이너에 안전하게 보관**

Map<Class<?>, Object> 구조 + type.cast(...)로 컴파일 타임 시그니처와 런타임 검증을 함께 확보.

2) **캐스팅 제거로 가독성·리팩토링 안정성 향상**

(Type) obj 캐스팅이 사라져 코드가 깔끔해지고, 타입 변경 시 컴파일러가 오류를 바로 잡아준다.

3) **키 충돌·오타 감소**

문자열 키 대신 클래스 키를 쓰면 “동일 이름 다른 의미” 문제와 오타로 인한 버그가 줄어든다.

4) **도메인 친화적 확장 가능**

Class<T> 대신 Column<T> 같은 커스텀 키를 정의해 DB Row, 파이프라인 메타데이터 등으로 확장 가능.

5) **어노테이션/리플렉션과 자연스러운 호환**

Class<T> 기반이라 getAnnotation(Class<T>) 같은 한정적 타입 토큰 패턴과 잘 맞는다.

---

### 주의점

1) **Raw type(로 타입) 금지**

(Class) Integer.class 등 로 타입으로 넣으면 타입 안전성이 깨진다. put에서 type.cast(value)로 즉시 실패하게 만들어 오류를 빨리 드러내자.

2) **실체화 불가 타입 한계**

List<String>.class는 존재하지 않으므로 List<String> 같은 구체 제네릭 타입을 직접 키로 쓰기 어렵다. 
정말 필요하면 스프링의 ParameterizedTypeReference<T> 같은 슈퍼 타입 토큰 기법을 별도 컨테이너로 설계한다.

코드 예시

#### A. 최소 구현 — 타입 토큰 기반 컨테이너
~~~java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class Favorites {
private final Map<Class<?>, Object> map = new HashMap<>();

    public <T> void put(Class<T> type, T value) {
        map.put(Objects.requireNonNull(type), type.cast(value)); // 타입 즉시 검증
    }

    public <T> T get(Class<T> type) {
        return type.cast(map.get(type));
    }

    public static void main(String[] args) {
        Favorites f = new Favorites();
        f.put(String.class, "Java");
        f.put(Integer.class, 123);

        String s = f.get(String.class);   // 캐스팅 불필요
        Integer n = f.get(Integer.class);

        System.out.println(s + " / " + n); // Java / 123
    }
}
~~~
- 포인트: Map<Class<?>, Object> + type.cast(...)로 안전 저장/조회.

#### B. 도메인 확장 — Row/Column 모델

~~~java
final class Column<T> {
    private final String name;
    private final Class<T> type;

    private Column(String name, Class<T> type) {
        this.name = name; this.type = type;
    }
    public static <T> Column<T> of(String name, Class<T> type) {
        return new Column<>(name, type);
    }
    public Class<T> type() { return type; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Column<?> other)) return false;
        return name.equals(other.name) && type.equals(other.type);
    }

    @Override
    public int hashCode() {
        return 31 * name.hashCode() + type.hashCode();
    }
}
~~~
- 포인트: Class<T> 대신 **도메인 키(Column<T>)**로 확장.
---
### 정리
- 포인트는 컨테이너가 아니라 ‘키’를 제네릭화하는 것.
- Map<Class<?>, Object> + type.cast(...)를 사용하면 이종 데이터도 컴파일 타임 타입 안전성을 유지한 채 한 곳에서 다룰 수 있다.
- 로 타입을 피하고, 제네릭 타입 키가 꼭 필요할 때만 슈퍼 타입 토큰 등 심화 기법을 선택적으로 쓴다.