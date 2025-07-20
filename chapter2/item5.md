# [Item 5] 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

## 서론  
많은 클래스는 하나 이상의 자원(resource)에 의존한다.  
예를 들어, **맞춤법 검사기(SpellChecker)** 는 **사전(Lexicon)** 에 의존한다.  
이 자원을 직접 클래스 내부에서 만들거나, 싱글턴/정적 유틸리티 클래스에 하드코딩하면 **유연성과 테스트 용이성**이 떨어진다.  
이 문제를 해결하려면 **의존 객체 주입(Dependency Injection, DI)** 을 적용해야 한다.

---

## 문제점 1: 정적 유틸리티 클래스 사용 → 유연성 부족, 테스트 어려움

```
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    private SpellChecker() {} // 인스턴스 생성 방지

    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}
```

- 정적 필드에 의존 → 사전(Lexicon)을 교체하거나 설정 변경 불가능
- 여러 언어, 테스트용 사전 등을 지원하기 어려움
- 멀티스레드 환경에서 제어도 불가능

---

## 문제점 2: 싱글턴 사용 → 동일한 한계

```
public class SpellChecker {
    private final Lexicon dictionary = ...;

    public static final SpellChecker INSTANCE = new SpellChecker(...);

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

- INSTANCE는 단 하나의 사전만 사용 → 다국어, 테스트, 특수 목적 대응 어려움
- 내부 자원(dictionary)을 바꿀 방법이 없음

---

## 해결책: 의존 객체 주입 (Dependency Injection)

### 자원을 외부에서 주입하도록 설계하면 유연성과 재사용성이 대폭 향상된다.

```
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

### 장점
- 다양한 종류의 사전(Lexicon)을 지원 가능
- 테스트 코드 작성이 쉬움 (Mock 사전 등 사용 가능)
- 불변 보장 (`final` 필드) → 멀티스레드 환경에서도 안전
- 재사용성 증가 → 여러 객체에서 같은 자원 공유 가능

---

## 팩터리를 이용한 응용: 자원 팩터리도 주입 가능

```
Mosaic create(Supplier<? extends Tile> tileFactory) {
    // tileFactory.get()으로 Tile 인스턴스를 여러 개 생성
}
```

- 자원을 직접 넘기기보다 **필요할 때마다 생성하는 팩터리**를 주입할 수 있음
- 자바 8의 `Supplier<T>`는 간단한 팩터리로 이상적
- `< ? extends T >` 를 사용해 타입 유연성도 확보 가능

---

## 주의사항: 대규모 프로젝트에서는 코드가 어지러울 수 있다

- 수백~수천 개의 의존 객체가 생기면 DI 코드가 복잡해질 수 있음
- → **의존성 주입 프레임워크 사용** 고려 (예: Spring, Dagger, Guice 등)
- 이들 프레임워크는 객체 생성을 자동화하고 주입을 관리해준다

---

## 핵심 정리

- 클래스가 외부 자원에 의존한다면, 자원을 클래스 내부에서 생성하거나 하드코딩하지 말자.
- 자원을 생성자, 정적 팩터리, 빌더 등을 통해 외부에서 **주입(inject)** 하자.
- 이 방식을 **의존 객체 주입(Dependency Injection)** 이라고 하며,  
  클래스의 **유연성, 재사용성, 테스트 용이성**을 크게 개선해준다.
- DI는 단순한 클래스부터 대규모 프레임워크 기반 시스템까지 폭넓게 활용된다.
```
