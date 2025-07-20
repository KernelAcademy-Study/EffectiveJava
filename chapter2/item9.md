# [Item 9] try-finally보다는 try-with-resources를 사용하라

## 서론
    자바 라이브러리에는 close 메서드를 호출해서 직접 닫아야 하는 자원이 있다. 자바에서는 주로 파일 입출력, 네트워크 연결, 데이터베이스 연결 등에서 사용된다. 
    이는 자바뿐만 아니라 파이썬에서도 close()를 써야하는 경우가 있다.
    try-finally와 try-with-resources의 차이는 뭘까? close()는 언제, 어디서, 어떻게, 왜 사용하는지 궁금해졌다.

## 본론 
### 자바에서 리소스 해제의 원칙과 try-with-resources
#### 1. 왜 자원을 닫아야 할까? : close() 사용!
-> 자바에서는 파일, 네트워크 소켓, 데이터베이스 커넥션 같은 자원을 사용한 뒤에는 반드시 닫아야 한다. 
이 자원들은 운영체제 레벨에서 한정된 개수만 사용할 수 있기 때문에, 닫지 않으면 심각한 문제가 발생! 

#### 어떤 문제가 발생하나?

a. 메모리 누수(Memory Leak)
- 자바 객체는 GC(Garbage Collector)가 관리하지만, 네이티브 자원은 GC가 수거 못함!

b. 파일/네트워크 충돌 
- 파일이 잠겨서 다른 프로세스에서 접근 불가능 할 수 있음! 

c. 시스템 리소스 고갈
- 열려 있는 파일 디스크립터, 소켓 핸들 등이 일정 개수를 초과하면 시스템이 동작하지 않을 수 있음!

#### 2. 전통적인 방식 : try-finally
-> 예외가 발생해도 자원이 반드시 닫히도록 하기 위해 try-finally을 사용함. 

```java
FileInputStream fis = null;
try {
    fis = new FileInputStream("sample.txt");
// 파일을 읽는 로직
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fis != null) {
        try {
            fis.close(); // 리소스 해제
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
#### 문제점 
- 자원이 여러개 중첩된 경우에는 try-finally가 복잡해짐. 
- close() 호출 도중에 예외가 발생하면, 기존 예외가 덮여짐.
- 코드가 장황해지므로 실수가 나올 확률이 높아짐!

### 3. 선호하는 방식 : try-with-resources
Java 7부터 도입된 try-with-resources 문은 AutoCloseable 인터페이스을 구현하는 자원에 대해 자동적으로 close()를 써줌.
해당 자원이 AutoCloseable 또는 Closeable 인터페이스를 구현해야 하고, try() 괄호안에서 선언된 자원은 try 블록이 끝날때 자동으로 닫힘. 


```java
public static void readFile(String path) {
    try (FileInputStream fis = new FileInputStream(path)) {
        int data;
        while ((data = fis.read()) != -1) {
            System.out.print((char) data);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

#### 장점 
1) 코드 간결성
- 중첩 구조가 사라져 깔끔함.
2) 예외 처리
- close() 중 발생한 예외는 Suppressed로 Exception으로 보존됨
3) 실수 가능성을 줄임
- 실수로 close()를 빼먹을 일이 없음

## 요약 정리 
구분|try-finally|try-with-resources
|:-:|:-:|:-:|
코드 길이|장황하고 복잡함|간결하고 읽기 쉬움
예외 처리|여러 예외 중 일부 손실됨|Suppressed로 추적 가능
실수 가능성|close 누락 가능성 있음|자동으로 안전하게 닫힘
다중 자원 처리|중첩 구조 필요|try() 안에 나열 가능



