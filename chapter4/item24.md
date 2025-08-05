# [Item 24] 멤버 클래스는 되도록 static으로 만들라

## 서론
중첩 클래스(nested class)란 다른 클래스 안에 정의된 클래스를 말한다. 중첩 클래스는 자신을 감싼 바깥 클래스에서만 쓰여야 하며, 그 외의 쓰임새가 있다면 톱레벨 클래스로 만들어야 한다. 중첩 클래스의 종류는 **정적 멤버 클래스**, **비정적 멤버 클래스**, **익명 클래스**, **지역 클래스** 이렇게 네 가지다. 이 중 첫 번째를 제외한 나머지는 내부 클래스(inner class)에 해당한다.

---

## 중첩 클래스의 종류와 특징

### 1. 정적 멤버 클래스 (Static Member Class)

정적 멤버 클래스는 바깥 클래스의 private 멤버에도 접근할 수 있다는 점만 빼고는 일반 클래스와 똑같다.

```java
public class Calculator {
    
    // 정적 멤버 클래스
    public static class Operation {
        public enum Type { PLUS, MINUS, MULTIPLY, DIVIDE }
        
        private final Type type;
        
        public Operation(Type type) {
            this.type = type;
        }
        
        public double calculate(double a, double b) {
            switch (type) {
                case PLUS: return a + b;
                case MINUS: return a - b;
                case MULTIPLY: return a * b;
                case DIVIDE: 
                    if (b == 0) throw new ArithmeticException("0으로 나눌 수 없습니다");
                    return a / b;
                default: throw new AssertionError(type);
            }
        }
        
        @Override
        public String toString() {
            return type.name();
        }
    }
    
    // 바깥 클래스의 기능
    public double evaluate(double a, Operation operation, double b) {
        return operation.calculate(a, b);
    }
}

// 사용 예시
public class StaticMemberClassExample {
    public static void main(String[] args) {
        // 바깥 클래스의 인스턴스 없이도 생성 가능
        Calculator.Operation plus = new Calculator.Operation(Calculator.Operation.Type.PLUS);
        Calculator.Operation divide = new Calculator.Operation(Calculator.Operation.Type.DIVIDE);
        
        Calculator calc = new Calculator();
        
        System.out.println("5 + 3 = " + calc.evaluate(5, plus, 3));
        System.out.println("10 / 2 = " + calc.evaluate(10, divide, 2));
    }
}
```

### 2. 비정적 멤버 클래스 (Non-static Member Class)

비정적 멤버 클래스의 인스턴스는 바깥 클래스의 인스턴스와 암묵적으로 연결된다.

```java
// 문제가 있는 예시 - 비정적 멤버 클래스의 잘못된 사용
public class MySet<E> {
    private Object[] elements = new Object[16];
    private int size = 0;
    
    public boolean add(E element) {
        if (size >= elements.length) {
            elements = Arrays.copyOf(elements, elements.length * 2);
        }
        elements[size++] = element;
        return true;
    }
    
    // 문제: 불필요한 바깥 인스턴스 참조를 가짐
    public class MyIterator implements Iterator<E> {
        private int position = 0;
        
        @Override
        public boolean hasNext() {
            return position < size;  // 바깥 클래스의 size에 접근
        }
        
        @Override
        @SuppressWarnings("unchecked")
        public E next() {
            if (!hasNext()) throw new NoSuchElementException();
            return (E) elements[position++];  // 바깥 클래스의 elements에 접근
        }
    }
    
    public Iterator<E> iterator() {
        return new MyIterator();  // 바깥 인스턴스에 대한 숨은 참조 생성
    }
}
```

### 3. 정적 멤버 클래스로 수정한 올바른 예시

```java
public class ImprovedMySet<E> {
    private Object[] elements = new Object[16];
    private int size = 0;
    
    public boolean add(E element) {
        if (size >= elements.length) {
            elements = Arrays.copyOf(elements, elements.length * 2);
        }
        elements[size++] = element;
        return true;
    }
    
    // 개선: 정적 멤버 클래스로 변경
    private static class MyIterator<E> implements Iterator<E> {
        private final Object[] elements;
        private final int size;
        private int position = 0;
        
        MyIterator(Object[] elements, int size) {
            this.elements = elements;
            this.size = size;
        }
        
        @Override
        public boolean hasNext() {
            return position < size;
        }
        
        @Override
        @SuppressWarnings("unchecked")
        public E next() {
            if (!hasNext()) throw new NoSuchElementException();
            return (E) elements[position++];
        }
    }
    
    public Iterator<E> iterator() {
        return new MyIterator<>(elements, size);  // 필요한 데이터만 전달
    }
}
```

---

## 비정적 멤버 클래스의 적절한 사용 사례

### 어댑터(Adapter) 패턴에서의 활용

```java
public class MyMap<K, V> {
    private Object[] keys = new Object[16];
    private Object[] values = new Object[16];
    private int size = 0;
    
    public V put(K key, V value) {
        // 간단한 구현 (실제로는 더 복잡)
        keys[size] = key;
        values[size] = value;
        size++;
        return null;
    }
    
    @SuppressWarnings("unchecked")
    public V get(K key) {
        for (int i = 0; i < size; i++) {
            if (Objects.equals(keys[i], key)) {
                return (V) values[i];
            }
        }
        return null;
    }
    
    // 비정적 멤버 클래스의 적절한 사용: 어댑터 뷰
    public class KeySet extends AbstractSet<K> {
        @Override
        public Iterator<K> iterator() {
            return new KeyIterator();
        }
        
        @Override
        public int size() {
            return MyMap.this.size;  // 바깥 인스턴스의 size 참조
        }
        
        @Override
        public boolean contains(Object o) {
            return MyMap.this.get((K) o) != null;  // 바깥 인스턴스의 get 메서드 사용
        }
        
        @Override
        public boolean remove(Object o) {
            return MyMap.this.remove(o) != null;  // 바깥 인스턴스의 remove 메서드 사용
        }
        
        private class KeyIterator implements Iterator<K> {
            private int position = 0;
            
            @Override
            public boolean hasNext() {
                return position < MyMap.this.size;
            }
            
            @Override
            @SuppressWarnings("unchecked")
            public K next() {
                if (!hasNext()) throw new NoSuchElementException();
                return (K) MyMap.this.keys[position++];
            }
        }
    }
    
    // 비정적 멤버 클래스의 적절한 사용: 어댑터 뷰  
    public class ValueCollection extends AbstractCollection<V> {
        @Override
        public Iterator<V> iterator() {
            return new ValueIterator();
        }
        
        @Override
        public int size() {
            return MyMap.this.size;
        }
        
        private class ValueIterator implements Iterator<V> {
            private int position = 0;
            
            @Override
            public boolean hasNext() {
                return position < MyMap.this.size;
            }
            
            @Override
            @SuppressWarnings("unchecked")
            public V next() {
                if (!hasNext()) throw new NoSuchElementException();
                return (V) MyMap.this.values[position++];
            }
        }
    }
    
    public Set<K> keySet() {
        return new KeySet();  // Map의 키들을 Set 뷰로 제공
    }
    
    public Collection<V> values() {
        return new ValueCollection();  // Map의 값들을 Collection 뷰로 제공
    }
    
    public V remove(K key) {
        for (int i = 0; i < size; i++) {
            if (Objects.equals(keys[i], key)) {
                @SuppressWarnings("unchecked")
                V oldValue = (V) values[i];
                // 제거 로직 (간단히 구현)
                System.arraycopy(keys, i + 1, keys, i, size - i - 1);
                System.arraycopy(values, i + 1, values, i, size - i - 1);
                size--;
                return oldValue;
            }
        }
        return null;
    }
}

// 사용 예시
public class AdapterExample {
    public static void main(String[] args) {
        MyMap<String, Integer> map = new MyMap<>();
        map.put("apple", 100);
        map.put("banana", 200);
        map.put("cherry", 300);
        
        // 키 뷰 사용
        Set<String> keys = map.keySet();
        System.out.println("키들: " + keys);
        
        // 값 뷰 사용
        Collection<Integer> values = map.values();
        System.out.println("값들: " + values);
        
        // 뷰를 통한 제거
        keys.remove("banana");
        System.out.println("제거 후 맵 크기: " + keys.size());
    }
}
```

---

## 익명 클래스 (Anonymous Class)

### 익명 클래스의 특징과 사용법

```java
public class AnonymousClassExample {
    
    // 함수형 인터페이스
    interface Calculator {
        int calculate(int a, int b);
    }
    
    public static void main(String[] args) {
        // 익명 클래스 사용
        Calculator adder = new Calculator() {
            @Override
            public int calculate(int a, int b) {
                System.out.println("덧셈 연산을 수행합니다");
                return a + b;
            }
        };
        
        Calculator multiplier = new Calculator() {
            @Override
            public int calculate(int a, int b) {
                System.out.println("곱셈 연산을 수행합니다");
                return a * b;
            }
        };
        
        System.out.println("5 + 3 = " + adder.calculate(5, 3));
        System.out.println("4 * 6 = " + multiplier.calculate(4, 6));
        
        // 람다로 더 간결하게 (Java 8+)
        Calculator subtractor = (a, b) -> {
            System.out.println("뺄셈 연산을 수행합니다");
            return a - b;
        };
        
        System.out.println("10 - 4 = " + subtractor.calculate(10, 4));
    }
}
```

### 익명 클래스의 실제 활용 예시

```java
public class EventHandlerExample {
    
    interface EventListener {
        void onEvent(String event);
    }
    
    static class EventManager {
        private final List<EventListener> listeners = new ArrayList<>();
        
        public void addListener(EventListener listener) {
            listeners.add(listener);
        }
        
        public void fireEvent(String event) {
            for (EventListener listener : listeners) {
                listener.onEvent(event);
            }
        }
    }
    
    public static void main(String[] args) {
        EventManager manager = new EventManager();
        
        // 익명 클래스로 다양한 이벤트 리스너 생성
        manager.addListener(new EventListener() {
            @Override
            public void onEvent(String event) {
                System.out.println("로그 기록: " + event);
            }
        });
        
        manager.addListener(new EventListener() {
            @Override
            public void onEvent(String event) {
                if (event.contains("ERROR")) {
                    System.err.println("🚨 에러 알림: " + event);
                }
            }
        });
        
        manager.addListener(new EventListener() {
            private int eventCount = 0;
            
            @Override
            public void onEvent(String event) {
                eventCount++;
                System.out.println("이벤트 카운터: " + eventCount + "번째 이벤트 - " + event);
            }
        });
        
        // 이벤트 발생
        manager.fireEvent("사용자 로그인");
        manager.fireEvent("파일 업로드");
        manager.fireEvent("ERROR: 데이터베이스 연결 실패");
    }
}
```

---

## 지역 클래스 (Local Class)

```java
public class LocalClassExample {
    
    public List<String> processItems(List<String> items, String prefix) {
        // 지역 클래스 정의
        class ItemProcessor {
            private final String processingPrefix;
            
            ItemProcessor(String prefix) {
                this.processingPrefix = prefix;
            }
            
            String process(String item) {
                // 지역 변수와 매개변수에 접근 가능 (effectively final이어야 함)
                return processingPrefix + ": " + item.toUpperCase();
            }
            
            boolean isValid(String item) {
                return item != null && !item.trim().isEmpty();
            }
        }
        
        ItemProcessor processor = new ItemProcessor(prefix);
        List<String> result = new ArrayList<>();
        
        for (String item : items) {
            if (processor.isValid(item)) {
                result.add(processor.process(item));
            }
        }
        
        return result;
    }
    
    public static void main(String[] args) {
        LocalClassExample example = new LocalClassExample();
        
        List<String> items = Arrays.asList("apple", "banana", "", "cherry", null, "date");
        List<String> processed = example.processItems(items, "FRUIT");
        
        processed.forEach(System.out::println);
        // 출력:
        // FRUIT: APPLE
        // FRUIT: BANANA  
        // FRUIT: CHERRY
        // FRUIT: DATE
    }
}
```

---

## 메모리 누수 위험성

### 비정적 멤버 클래스의 메모리 누수 문제

```java
public class MemoryLeakExample {
    private final List<String> data = new ArrayList<>();
    
    {
        // 대량의 데이터 생성
        for (int i = 0; i < 100000; i++) {
            data.add("Large data item " + i);
        }
    }
    
    // 문제가 있는 비정적 멤버 클래스
    public class ProblematicIterator implements Iterator<String> {
        private int index = 0;
        
        @Override
        public boolean hasNext() {
            return index < data.size();
        }
        
        @Override
        public String next() {
            return data.get(index++);
        }
    }
    
    // 올바른 정적 멤버 클래스
    private static class GoodIterator implements Iterator<String> {
        private final List<String> data;
        private int index = 0;
        
        GoodIterator(List<String> data) {
            this.data = data;
        }
        
        @Override
        public boolean hasNext() {
            return index < data.size();
        }
        
        @Override
        public String next() {
            return data.get(index++);
        }
    }
    
    public Iterator<String> getBadIterator() {
        return new ProblematicIterator();  // 바깥 인스턴스 참조 유지
    }
    
    public Iterator<String> getGoodIterator() {
        return new GoodIterator(data);  // 필요한 데이터만 전달
    }
    
    public static void demonstrateMemoryLeak() {
        List<Iterator<String>> iterators = new ArrayList<>();
        
        // 메모리 누수 발생
        for (int i = 0; i < 1000; i++) {
            MemoryLeakExample example = new MemoryLeakExample();
            iterators.add(example.getBadIterator());  // 바깥 인스턴스가 GC되지 않음
            // example이 스코프를 벗어나도 Iterator가 참조를 유지하므로 GC되지 않음
        }
        
        System.out.println("1000개의 이터레이터가 생성되었습니다");
        System.out.println("각 이터레이터는 대용량 데이터를 가진 바깥 인스턴스를 참조합니다");
        
        // 강제 GC 실행
        System.gc();
        
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        System.out.println("GC 후에도 메모리가 해제되지 않을 가능성이 높습니다");
    }
    
    public static void main(String[] args) {
        demonstrateMemoryLeak();
    }
}
```

---

## 실무에서의 선택 기준

### 중첩 클래스 사용 결정 흐름도

```java
public class NestedClassDecisionGuide {
    
    // 1. 메서드 밖에서도 사용해야 하거나, 메서드 안에 정의하기엔 너무 긴가?
    //    YES -> 멤버 클래스로 만든다
    //    NO -> 지역 클래스로 만든다
    
    // 2. 멤버 클래스의 인스턴스 각각이 바깥 인스턴스를 참조하는가?
    //    YES -> 비정적으로 만든다
    //    NO -> 정적으로 만든다
    
    // 3. 지역 클래스의 경우: 익명 클래스도 쓸 수 있는가?
    //    YES -> 익명 클래스로 만든다
    //    NO -> 지역 클래스로 만든다
    
    // 예시들:
    
    // 1. 정적 멤버 클래스 - 바깥 인스턴스 참조 불필요
    public static class Config {
        private final String host;
        private final int port;
        
        public Config(String host, int port) {
            this.host = host;
            this.port = port;
        }
        
        // getter들...
    }
    
    // 2. 비정적 멤버 클래스 - 바깥 인스턴스 참조 필요
    public class ConnectionPool {
        private final List<Connection> connections = new ArrayList<>();
        
        public Connection getConnection() {
            // 바깥 클래스의 Config를 사용해야 함
            return createConnection(/* 바깥 인스턴스의 데이터 사용 */);
        }
        
        private Connection createConnection() {
            // 연결 생성 로직
            return null;  // 예시용
        }
    }
    
    public void demonstrateLocalClass() {
        final String prefix = "LOG";
        
        // 3. 지역 클래스 - 복잡한 로직이 필요한 경우
        class Logger {
            void log(String message) {
                System.out.println(prefix + ": " + message);
            }
            
            void logWithTimestamp(String message) {
                System.out.println(prefix + " [" + System.currentTimeMillis() + "]: " + message);
            }
        }
        
        Logger logger = new Logger();
        logger.log("시스템 시작");
        logger.logWithTimestamp("사용자 로그인");
    }
    
    public void demonstrateAnonymousClass() {
        // 4. 익명 클래스 - 간단한 구현
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("백그라운드 작업 실행");
            }
        };
        
        // 또는 람다로 더 간결하게
        Runnable lambdaTask = () -> System.out.println("람다 백그라운드 작업 실행");
        
        new Thread(task).start();
        new Thread(lambdaTask).start();
    }
}
```

---

## 정리

1. **중첩 클래스에는 네 가지가 있으며, 각각의 쓰임이 다르다**

2. **메서드 밖에서도 사용해야 하거나 메서드 안에 정의하기엔 너무 길다면 멤버 클래스로 만든다**

3. **멤버 클래스의 인스턴스 각각이 바깥 인스턴스를 참조한다면 비정적으로, 그렇지 않으면 정적으로 만든다**
   - 비정적 멤버 클래스는 바깥 인스턴스에 대한 숨은 외부 참조를 갖는다
   - 이 참조를 저장하려면 시간과 공간이 소비된다
   - 더 심각한 문제는 가비지 컬렉션이 바깥 인스턴스를 수거하지 못하는 메모리 누수가 생길 수 있다는 점이다

4. **중첩 클래스가 한 메서드 안에서만 쓰이면서 그 인스턴스를 생성하는 지점이 단 한 곳이고 해당 타입으로 쓰기에 적합한 클래스나 인터페이스가 이미 있다면 익명 클래스로 만들고, 그렇지 않으면 지역 클래스로 만든다**

5. **정적 멤버 클래스는 흔히 바깥 클래스와 함께 쓰일 때만 유용한 public 도우미 클래스로 쓰인다**
   - 예: Calculator.Operation.PLUS 같은 형태

멤버 클래스에서 바깥 인스턴스에 접근할 일이 없다면 무조건 static을 붙여서 정적 멤버 클래스로 만들자!