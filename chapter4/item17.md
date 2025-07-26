# [Item 17] 변경 가능성을 최소화하라. 

## 불변 클래스(Immutable Class)
- 인스턴스 생성 후 내부 상태가 절대 바뀌지 않는 클래스
- 대표 예: String, 박싱된 기본 타입 클래스(Integer 등), BigInteger, BigDecimal

### 불변 클래스로 설계하는 5가지 규칙
1.	변경자(Setter) 메서드 제공 금지
<br> 내부 상태를 바꾸는 메서드를 만들지 않는다.
2.	클래스 확장 금지
<br>final 선언 또는 private 생성자+정적 팩토리로 외부 상속을 차단.
3.	필드를 private final로 선언
- final: 한 번 초기화된 후 절대 재할당 불가
- private: 외부에서 직접 접근·변경 불가
4.	가변 컴포넌트 노출 금지
- 생성자, 접근자(getter), 역직렬화(readObject) 시 방어적 복사 수행
- 참조를 그대로 반환하면 외부에서 내부 객체를 조작당할 수 있음
5.	불변 객체 재활용 권장
- 자주 쓰이는 값은 public static final 상수로 제공 (Complex.ZERO, Complex.ONE 등)
- 정적 팩토리(valueOf)로 캐싱 기능 추가 가능

### 불변 클래스의 장점·단점
#### 장점
- 설계·구현·사용이 단순하고 안전
- 스레드 안전 보장(동기화 필요 없음)
- 방어적 복사 불필요
- 실패 원자성 제공
#### 단점
- 상태가 다르면 새 인스턴스 생성 → 메모리·GC 부담
- 복잡한 연산 시 객체 생성 폭증
- 대응 방안: 다단계 연산을 API에 미리 제공하거나, 내부 가변 동반 클래스로 최적화
---

### 공부하며 깨달은 점 
> 안전한 공개 API를 위해서는 외부에 노출되는 모든 경로를 생각해서 막아야 한다. 
- 생성 순간이 곧 완결점 
<br>생성자 안에서 모든 불변식을 만족시켜야, 이후 상태 변화 가능성을 완전히 제거할 수 있다. 
- 정적 팩토리의 유연성
<br>생성자를 private으로 숨기고 valueOf 같은 팩토리를 쓰면, 훗날 캐싱,상속 대안 구현 등 확장이 쉽다. 
- 내부 최적화와 외부 단순화의 균형 
<br>겉으로는 불변이 보장되지만, 내부엔 성능 최적화를 위한 가변 동반클래스가 숨겨질 수 있다. 


### Complex 불변 클래스(정적 팩토리 방식)
```java
public class Complex {
    private final double re;
    private final double im;

    // 생성자는 외부에서 직접 호출 불가능
    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    // 정적 팩토리: 캐싱·확장성 확보
    public static Complex valueOf(double re, double im) {
        // 자주 쓰이는 값은 캐시해서 같은 인스턴스 재사용 가능
        if (re == 0 && im == 0) return ZERO;
        if (re == 1 && im == 0) return ONE;
        if (re == 0 && im == 1) return I;
        return new Complex(re, im);
    }

    // 중요한 상수 미리 정의
    public static final Complex ZERO = new Complex(0, 0);
    public static final Complex ONE  = new Complex(1, 0);
    public static final Complex I    = new Complex(0, 1);

    // 접근자: 내부 상태 노출 없이 값만 반환
    public double realPart()      { return re; }
    public double imaginaryPart() { return im; }

    // 사칙 연산: 기존 인스턴수는 그대로, 새 인스턴스 반환
    public Complex plus(Complex c) {
        return new Complex(this.re + c.re, this.im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(this.re - c.re, this.im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(
            this.re * c.re - this.im * c.im,
            this.re * c.im + this.im * c.re
        );
    }

    public Complex dividedBy(Complex c) {
        double denom = c.re * c.re + c.im * c.im;
        return new Complex(
            (this.re * c.re + this.im * c.im) / denom,
            (this.im * c.re - this.re * c.im) / denom
        );
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Complex)) return false;
        Complex c = (Complex) o;
        return Double.compare(c.re, re) == 0
            && Double.compare(c.im, im) == 0;
    }

    @Override
    public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }

    @Override
    public String toString() {
        return String.format("(%f + %fi)", re, im);
    }
}
```
- private 생성자 + 정적 팩토리로 상속 차단, 인스턴스 재활용 지원

### 방어적 복사 필요한 경우 
```java
public final class Person {
    private final String name;
    private final Date   birthDate;  // Date는 가변 클래스

    public Person(String name, Date birthDate) {
        this.name = name;
        // 방어적 복사: 외부 Date 참조를 그대로 저장하면 위험!
        this.birthDate = new Date(birthDate.getTime());
    }

    public String getName() {
        return name;
    }

    public Date getBirthDate() {
        // 내부 Date를 그대로 반환하면 외부에서 조작 가능하므로
        return new Date(birthDate.getTime());
    }
}

```
- Date처럼 가변 객체를 필드로 가질 경우, 반드시 복사하여 내부 상태 노출을 막아야 한다.