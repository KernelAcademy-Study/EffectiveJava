# [Item 7] 다 쓴 객체 참조를 해제하라

## 서론
자바는 가비지 컬렉터(GC)가 메모리를 자동으로 관리해주지만, 이것이 메모리 누수를 완전히 방지해주지는 않는다. 개발자가 직접 참조를 해제하지 않으면 **메모리 누수(Memory Leak)**가 발생할 수 있고, 이는 성능 저하와 `OutOfMemoryError`의 원인이 된다.

---

## 메모리 누수가 발생하는 주요 상황

### 1. 스택 구현체에서의 메모리 누수

스택을 직접 구현할 때 `pop()` 메서드에서 객체 참조를 제거하지 않으면 메모리 누수가 발생한다.

```java
// 문제가 있는 스택 구현
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    // 문제: pop된 객체의 참조가 여전히 남아있음
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];  // 참조 해제 안됨!
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

**문제점**: `pop()`으로 제거된 객체도 배열에서 여전히 참조되고 있어서 GC가 수거하지 못한다.

```java
// 올바른 스택 구현
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;  // 다 쓴 참조 해제!
    return result;
}
```

### 2. 캐시에서의 메모리 누수

캐시에 객체를 넣고 나서 그 사실을 잊은 채 그 객체를 다시 쓰지 않는 일이 자주 있다.

```java
public class SimpleCache {
    private final Map<String, ExpensiveObject> cache = new HashMap<>();

    public ExpensiveObject get(String key) {
        ExpensiveObject obj = cache.get(key);
        if (obj == null) {
            obj = createExpensiveObject(key);
            cache.put(key, obj);  // 계속 누적됨!
        }
        return obj;
    }

    private ExpensiveObject createExpensiveObject(String key) {
        // 비용이 큰 객체 생성
        return new ExpensiveObject(key);
    }
}
```

**해결책 1: WeakHashMap 사용**
```java
public class ImprovedCache {
    // 키가 더 이상 참조되지 않으면 자동으로 제거
    private final Map<String, ExpensiveObject> cache = new WeakHashMap<>();

    public ExpensiveObject get(String key) {
        return cache.computeIfAbsent(key, this::createExpensiveObject);
    }

    private ExpensiveObject createExpensiveObject(String key) {
        return new ExpensiveObject(key);
    }
}
```

**해결책 2: 시간 기반 만료 정책**
```java
public class TimedCache {
    private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();
    private final long maxAge = 60_000; // 1분

    public ExpensiveObject get(String key) {
        CacheEntry entry = cache.get(key);
        if (entry == null || isExpired(entry)) {
            ExpensiveObject obj = createExpensiveObject(key);
            cache.put(key, new CacheEntry(obj, System.currentTimeMillis()));
            return obj;
        }
        return entry.object;
    }

    private boolean isExpired(CacheEntry entry) {
        return System.currentTimeMillis() - entry.timestamp > maxAge;
    }

    // 정기적으로 만료된 항목 정리
    public void cleanup() {
        long now = System.currentTimeMillis();
        cache.entrySet().removeIf(entry -> 
            now - entry.getValue().timestamp > maxAge);
    }

    private static class CacheEntry {
        final ExpensiveObject object;
        final long timestamp;

        CacheEntry(ExpensiveObject object, long timestamp) {
            this.object = object;
            this.timestamp = timestamp;
        }
    }
}
```

### 3. 리스너와 콜백에서의 메모리 누수

클라이언트가 콜백을 등록만 하고 명확히 해지하지 않는다면, 콜백은 계속 쌓여간다.

```java
public class EventSource {
    private final List<EventListener> listeners = new ArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(listener);
    }

    // 문제: 리스너 해제 메서드가 없거나 사용하지 않음
    public void removeListener(EventListener listener) {
        listeners.remove(listener);
    }

    public void fireEvent(String event) {
        for (EventListener listener : listeners) {
            listener.onEvent(event);
        }
    }
}
```

**해결책: WeakReference 사용**
```java
public class ImprovedEventSource {
    private final List<WeakReference<EventListener>> listeners = new ArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(new WeakReference<>(listener));
    }

    public void fireEvent(String event) {
        Iterator<WeakReference<EventListener>> it = listeners.iterator();
        while (it.hasNext()) {
            WeakReference<EventListener> ref = it.next();
            EventListener listener = ref.get();
            if (listener == null) {
                it.remove();  // 약한 참조가 null이면 제거
            } else {
                listener.onEvent(event);
            }
        }
    }
}
```

---

## 언제 참조를 해제해야 할까?

### 1. 자기 메모리를 직접 관리하는 클래스
- 배열 기반 스택, 큐, 리스트 등
- 객체 풀(Object Pool)
- 캐시 구현체

### 2. 캐시 구현체
- 시간이 지나면 엔트리가 자동으로 사라지도록 설계
- `ScheduledThreadPoolExecutor` 등으로 주기적 정리

### 3. 리스너와 콜백
- 약한 참조(weak reference)로 저장
- 또는 명시적 해제 메서드 제공

---

## 실제 활용 예시

### LinkedList에서 노드 제거
```java
public class CustomLinkedList<T> {
    private Node<T> head;
    private int size = 0;

    public void remove(int index) {
        if (index < 0 || index >= size) {
            throw new IndexOutOfBoundsException();
        }

        if (index == 0) {
            Node<T> oldHead = head;
            head = head.next;
            
            // 참조 해제로 GC 도움
            oldHead.data = null;
            oldHead.next = null;
        } else {
            Node<T> prev = getNode(index - 1);
            Node<T> target = prev.next;
            prev.next = target.next;
            
            // 참조 해제
            target.data = null;
            target.next = null;
        }
        size--;
    }

    private static class Node<T> {
        T data;
        Node<T> next;

        Node(T data, Node<T> next) {
            this.data = data;
            this.next = next;
        }
    }
}
```

### 파일 처리 후 정리
```java
public class FileProcessor {
    private Map<String, BufferedReader> openFiles = new HashMap<>();

    public String readLine(String filename) throws IOException {
        BufferedReader reader = openFiles.get(filename);
        if (reader == null) {
            reader = Files.newBufferedReader(Paths.get(filename));
            openFiles.put(filename, reader);
        }
        return reader.readLine();
    }

    public void closeFile(String filename) throws IOException {
        BufferedReader reader = openFiles.remove(filename);
        if (reader != null) {
            reader.close();
            // Map에서 제거함으로써 참조 해제 완료
        }
    }

    public void closeAllFiles() throws IOException {
        for (BufferedReader reader : openFiles.values()) {
            reader.close();
        }
        openFiles.clear();  // 모든 참조 해제
    }
}
```

---

## 정리

1. **자동 메모리 관리에 의존하지 말고**, 명시적 참조 해제가 필요한 상황을 인식하라
2. **배열이나 컬렉션을 직접 관리**할 때는 사용 완료된 요소를 `null`로 설정하라
3. **캐시는 시간 기반 만료**나 `WeakHashMap` 등을 활용하여 자동 정리되도록 하라
4. **리스너/콜백은 약한 참조**를 사용하거나 명시적 해제 메서드를 제공하라
5. **메모리 프로파일러**를 활용해 실제 메모리 누수를 모니터링하라

초보자일 때 "자바는 GC가 알아서 해준다"고 생각하기 쉽지만, 참조가 살아있으면 GC도 어쩔 수 없다는 점을 기억하자!