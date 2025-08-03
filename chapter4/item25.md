# 아이템 25: 톱레벨 클래스는 한 파일에 하나만 담아라

## 3줄 요약
- 같은 이름의 톱레벨 클래스가 여러 파일에 다르게 정의되어 있으면 컴파일 순서에 따라 실행 결과가 달라져 예측 불가능한 버그가 생긴다.
- 해결: 톱레벨 클래스는 파일 하나당 하나만 두고, public 클래스면 파일명과 정확히 맞춰라.
- 관련된 보조 개념은 정적 멤버 클래스로 묶어 이름 충돌과 책임을 정리하자.

---

## 문제 있는 예시 (컴파일 순서에 따라 출력이 바뀜)

### Main.java
    public class Main {  
        public static void main(String[] args) {  
            System.out.println(Brand.Name + " " + Product.Model);  
        }  
    }

### BrandProduct1.java (첫 번째 정의)
    class Brand {  
        static final String Name = "Samsung";  
    }

    class Product {  
        static final String Model = "Galaxy";  
    }

### BrandProduct2.java (다른 정의)
    class Brand {  
        static final String Name = "Apple";  
    }

    class Product {  
        static final String Model = "iPhone";  
    }

### 동작 차이
- `javac Main.java BrandProduct1.java` → 출력: Samsung Galaxy
- `javac BrandProduct2.java Main.java` → 출력: Apple iPhone

**왜 문제인지**  
같은 이름(Brand, Product)의 톱레벨 클래스가 여러 파일에 서로 다른 내용으로 정의돼 있어서, 어떤 파일부터 컴파일하느냐에 따라 실제로 사용되는 정의가 달라진다. 빌드 환경이나 순서에 따라 결과가 달라지면 버그를 찾고 재현하는 게 어려워진다.

---

## 올바른 해결

### 1. 톱레벨 클래스는 파일 하나당 하나씩 분리 (권장)

#### Brand.java
    class Brand {  
        static final String Name = "Samsung";  
    }

#### Product.java
    class Product {  
        static final String Model = "Galaxy";  
    }

#### Main.java
    public class Main {  
        public static void main(String[] args) {  
            System.out.println(Brand.Name + " " + Product.Model);  
        }  
    }

- 어떤 순서로 컴파일해도 항상 "Samsung Galaxy"가 출력된다. 컴파일 순서나 환경에 영향을 받지 않는다.

### 2. 관련된 부차적 개념은 정적 멤버 클래스로 묶기 (대안)

#### Main.java
    public class Main {  
        public static void main(String[] args) {  
            System.out.println(Portfolio.Brand.Name + " " + Portfolio.Product.Model);  
        }  

        private static class Portfolio {  
            static class Brand {  
                static final String Name = "Apple";  
            }  

            static class Product {  
                static final String Model = "iPhone";  
            }  
        }  
    }

- Brand와 Product가 하나의 묶음 안에 들어가는 부수적인 개념이라면 이렇게 내부에 감싸서 구조를 명확히 하면 충돌을 피하면서도 관계를 표현할 수 있다.

---

## 해결법 설명

### 왜 이렇게 해야 하는지
자바에서는 동일한 이름의 톱레벨 클래스를 여러 파일에 정의할 수 있는데, 그 정의가 서로 다르면 어떤 파일을 먼저 컴파일하느냐에 따라 실제로 사용되는 클래스가 달라진다. 그러면 같은 소스 코드라도 빌드할 때마다 출력이 달라질 수 있고, 원인 파악이 어렵다.

### 무엇을 해야 하는지
- 클래스 하나당 파일 하나를 둔다. public 클래스면 반드시 그 이름과 일치하는 파일에만 넣는다.
- 관련성이 있는 작은 클래스들은 따로 분리하지 말고, 상위 구조 안에서 정적 멤버 클래스로 묶어서 함께 관리한다.
- 필요한 경우 패키지를 써서 같은 이름이라도 다른 네임스페이스로 분리한다.

### 실제 작업 흐름 체크리스트
1. 동일한 이름의 톱레벨 클래스가 여러 파일에 있는지 코드에서 찾아본다.
2. 중복 정의를 정리하거나, 역할이 다르면 패키지/이름을 분리한다.
3. 부차적인 관계면 내부 정적 클래스로 묶는다.
4. public 클래스면 파일명이 정확한지 확인한다.
5. 빌드해서 출력이 항상 일관적인지, 중복 정의로 인한 오류가 없는지 점검한다.
6. 코드 리뷰할 때 이 규칙이 지켜졌는지 체크리스트에 포함시킨다.

### 자주 실수하는 부분
- 이름이 비슷한 파일들을 그냥 두고 어떤 게 실제 쓰이는 건지 헷갈리는 상황
- 같은 클래스 이름을 패키지 없이 여러 곳에 두고 있어서 컴파일 순서에 따라 정의가 바뀌는 상황
- 모든 것을 내부 정적 클래스로 묶으려다 오히려 구조가 복잡해지는 경우 (진짜 관련 있는지 다시 생각)

---

## 정리
1. 톱레벨 클래스는 한 파일에 하나만 둬라.
2. 부수적인 관련 클래스는 정적으로 묶어서 이름 충돌을 줄이고 관계를 드러내라.
3. 패키지로 네임스페이스를 분리하고, public 클래스는 파일명과 맞춰라.
4. 빌드와 리뷰 과정에서 이 구조가 지켜졌는지 반드시 확인해라.