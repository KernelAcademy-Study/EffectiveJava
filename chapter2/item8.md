# [Item 8] finalizer와 cleaner 사용을 피하라

## 서론
자바에서는 객체가 소멸될 때 자동으로 호출되는 메커니즘으로 `finalizer`(Java 9부터 deprecated)와 `cleaner`(Java 9에서 도입)를 제공한다. 하지만 이들은 예측 불가능하고, 위험하며, 대부분 불필요하다. 왜 사용하면 안 되는지, 그리고 대안은 무엇인지 알아보자.

---

## finalizer와 cleaner의 문제점

### 1. 실행 시점을 예측할 수 없다

```java
// 문제가 있는 예시
public class ProblematicResource {
    private FileOutputStream fos;

    public ProblematicResource(String filename) throws IOException {
        this.fos = new FileOutputStream(filename);
    }

    public void write(String data) throws IOException {
        fos.write(data.getBytes());
    }

    // finalizer - 언제 실행될지 모름!
    @Override
    protected void finalize() throws Throwable {
        try {
            if (fos != null) {
                fos.close();
                System.out.println("파일이 finalizer에서 닫혔습니다");
            }
        } finally {
            super.finalize();
        }
    }
}

// 사용 예시
public class FinalizerTest {
    public static void main(String[] args) throws IOException {
        for (int i = 0; i < 100; i++) {
            ProblematicResource resource = new ProblematicResource("temp" + i + ".txt");
            resource.write("데이터 " + i);
            // close()를 호출하지 않음 - finalizer에 의존
        }
        
        System.gc();  // GC 요청해도 finalizer 실행 보장 안됨
        System.out.println("프로그램 종료");
        // 파일들이 제대로 닫혔을까? 보장할 수 없다!
    }
}
```

**문제**: finalizer는 GC가 실행될 때까지 호출되지 않으며, GC 실행 시점은 예측할 수 없다.

### 2. 성능 문제

```java
public class PerformanceTest {
    public static void main(String[] args) {
        long start, end;

        // finalizer 없는 객체 생성/소멸
        start = System.nanoTime();
        for (int i = 0; i < 500_000; i++) {
            Object obj = new Object();
        }
        end = System.nanoTime();
        System.out.println("일반 객체: " + (end - start) / 1_000_000 + "ms");

        // finalizer 있는 객체 생성/소멸
        start = System.nanoTime();
        for (int i = 0; i < 500_000; i++) {
            FinalizableObject obj = new FinalizableObject();
        }
        System.gc();
        end = System.nanoTime();
        System.out.println("finalizer 객체: " + (end - start) / 1_000_000 + "ms");
    }

    static class FinalizableObject {
        @Override
        protected void finalize() throws Throwable {
            // 빈 finalizer라도 성능에 큰 영향
        }
    }
}
```

**결과**: finalizer가 있는 객체는 생성과 소멸이 50배 이상 느릴 수 있다.

### 3. 예외 처리 문제

```java
public class FinalizerExceptionTest {
    @Override
    protected void finalize() throws Throwable {
        // finalizer에서 예외 발생
        throw new RuntimeException("finalizer에서 예외 발생!");
        // 이 예외는 무시되고, 스택 트레이스도 출력되지 않음
    }

    public static void main(String[] args) {
        FinalizerExceptionTest obj = new FinalizerExceptionTest();
        obj = null;
        System.gc();
        
        try {
            Thread.sleep(1000);  // finalizer 실행 대기
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("프로그램 계속 실행됨");
        // finalizer의 예외는 무시되고 프로그램은 계속 실행
    }
}
```

---

## 올바른 대안: AutoCloseable과 try-with-resources

### 1. 기본적인 자원 관리

```java
public class ProperResource implements AutoCloseable {
    private FileOutputStream fos;
    private boolean closed = false;

    public ProperResource(String filename) throws IOException {
        this.fos = new FileOutputStream(filename);
    }

    public void write(String data) throws IOException {
        if (closed) {
            throw new IllegalStateException("이미 닫힌 자원입니다");
        }
        fos.write(data.getBytes());
    }

    @Override
    public void close() throws IOException {
        if (!closed) {
            fos.close();
            closed = true;
            System.out.println("자원이 명시적으로 닫혔습니다");
        }
    }

    // 안전망으로만 사용하는 finalizer
    @Override
    protected void finalize() throws Throwable {
        if (!closed) {
            System.err.println("경고: 자원이 명시적으로 닫히지 않았습니다!");
            close();
        }
    }
}

// 올바른 사용법
public class ProperUsage {
    public static void main(String[] args) {
        // try-with-resources로 자동 자원 관리
        try (ProperResource resource = new ProperResource("output.txt")) {
            resource.write("안전하게 관리되는 데이터");
            // 블록을 벗어나면 자동으로 close() 호출
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 2. 복잡한 자원 관리 - cleaner 활용

```java
import java.lang.ref.Cleaner;

public class ResourceWithCleaner implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    
    private final State state;
    private final Cleaner.Cleanable cleanable;

    public ResourceWithCleaner(String filename) throws IOException {
        this.state = new State(filename);
        this.cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }

    // 정리 작업을 담당하는 static 중첩 클래스
    private static class State implements Runnable {
        private FileOutputStream fos;
        private boolean closed = false;

        State(String filename) throws IOException {
            this.fos = new FileOutputStream(filename);
        }

        @Override
        public void run() {
            if (!closed && fos != null) {
                try {
                    fos.close();
                    System.out.println("cleaner에 의해 자원이 정리되었습니다");
                } catch (IOException e) {
                    System.err.println("cleaner에서 자원 정리 실패: " + e.getMessage());
                }
                closed = true;
            }
        }
    }
}
```

**핵심**: `State` 클래스는 `ResourceWithCleaner` 인스턴스를 참조하지 않는다. 순환 참조가 생기면 GC가 객체를 수거하지 못한다.

---

## 언제 finalizer/cleaner를 사용할 수 있는가?

### 1. 안전망(Safety Net) 역할

```java
public class FileManager implements AutoCloseable {
    private FileInputStream fis;
    private boolean closed = false;

    public FileManager(String filename) throws IOException {
        this.fis = new FileInputStream(filename);
    }

    public int read() throws IOException {
        if (closed) {
            throw new IllegalStateException("파일이 이미 닫혔습니다");
        }
        return fis.read();
    }

    @Override
    public void close() throws IOException {
        if (!closed) {
            fis.close();
            closed = true;
        }
    }

    // 안전망으로만 사용 - 클라이언트가 close()를 깜빡했을 때
    @Override
    protected void finalize() throws Throwable {
        if (!closed) {
            System.err.println("경고: FileManager가 명시적으로 닫히지 않았습니다!");
            // 로그를 남기고 자원을 정리
            close();
        }
    }
}
```

### 2. 네이티브 피어(Native Peer) 정리

```java
public class NativeResourceManager {
    private final long nativeHandle;  // C/C++ 객체의 핸들

    public NativeResourceManager() {
        this.nativeHandle = createNativeResource();  // JNI 호출
    }

    // 네이티브 메서드들
    private native long createNativeResource();
    private native void destroyNativeResource(long handle);
    private native void processData(long handle, byte[] data);

    public void process(byte[] data) {
        if (nativeHandle == 0) {
            throw new IllegalStateException("네이티브 자원이 이미 해제되었습니다");
        }
        processData(nativeHandle, data);
    }

    // finalizer로 네이티브 자원 정리
    @Override
    protected void finalize() throws Throwable {
        if (nativeHandle != 0) {
            destroyNativeResource(nativeHandle);
        }
    }
}
```

---

## 실무에서의 모범 사례

### 1. 데이터베이스 연결 관리

```java
public class DatabaseConnection implements AutoCloseable {
    private Connection connection;
    private boolean closed = false;

    public DatabaseConnection(String url, String user, String password) 
            throws SQLException {
        this.connection = DriverManager.getConnection(url, user, password);
    }

    public ResultSet executeQuery(String sql) throws SQLException {
        if (closed) {
            throw new IllegalStateException("연결이 이미 닫혔습니다");
        }
        return connection.createStatement().executeQuery(sql);
    }

    @Override
    public void close() throws SQLException {
        if (!closed) {
            connection.close();
            closed = true;
        }
    }

    // 안전망
    @Override
    protected void finalize() throws Throwable {
        if (!closed) {
            System.err.println("경고: DB 연결이 명시적으로 닫히지 않았습니다!");
            close();
        }
    }
}

// 사용 예시
public void processData() {
    try (DatabaseConnection db = new DatabaseConnection("jdbc:h2:mem:test", "sa", "")) {
        ResultSet rs = db.executeQuery("SELECT * FROM users");
        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }
        // 자동으로 연결 닫힘
    } catch (SQLException e) {
        e.printStackTrace();
    }
}
```

### 2. 스레드 풀 관리

```java
public class TaskExecutor implements AutoCloseable {
    private final ExecutorService executor;
    private boolean shutdown = false;

    public TaskExecutor(int threadCount) {
        this.executor = Executors.newFixedThreadPool(threadCount);
    }

    public Future<?> submit(Runnable task) {
        if (shutdown) {
            throw new IllegalStateException("TaskExecutor가 이미 종료되었습니다");
        }
        return executor.submit(task);
    }

    @Override
    public void close() {
        if (!shutdown) {
            executor.shutdown();
            try {
                if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
                Thread.currentThread().interrupt();
            }
            shutdown = true;
        }
    }

    // 안전망
    @Override
    protected void finalize() throws Throwable {
        if (!shutdown) {
            System.err.println("경고: TaskExecutor가 명시적으로 종료되지 않았습니다!");
            close();
        }
    }
}
```

---

## 정리

1. **finalizer와 cleaner는 가능한 한 사용하지 마라**
   - 실행 시점이 불확실하고 성능에 악영향을 미친다

2. **AutoCloseable을 구현하고 try-with-resources를 사용하라**
   - 명확하고 예측 가능한 자원 관리

3. **finalizer/cleaner는 안전망 용도로만 사용하라**
   - 클라이언트가 close()를 깜빡했을 때의 마지막 보루

4. **네이티브 자원을 다룰 때는 cleaner를 고려하라**
   - JVM이 관리하지 않는 자원의 정리용

5. **상태를 확인하는 필드를 두어 중복 정리를 방지하라**
   - `closed` 플래그로 이미 정리된 자원에 대한 재정리 방지

자원 관리는 명시적이고 예측 가능하게 하는 것이 최선이다!