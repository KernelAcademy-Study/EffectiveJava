# [Item 20] 추상 클래스보다는 인터페이스를 우선하라

## 서론
자바가 제공하는 다중 구현 메커니즘은 인터페이스와 추상 클래스, 이렇게 두 가지다. 자바 8부터 인터페이스도 디폴트 메서드(default method)를 제공할 수 있게 되어, 두 메커니즘 모두 인스턴스 메서드를 구현 형태로 제공할 수 있다. 가장 큰 차이는 추상 클래스가 정의한 타입을 구현하는 클래스는 반드시 추상 클래스의 하위 클래스가 되어야 한다는 점이다. 반면 인터페이스가 선언한 메서드를 모두 정의하고 그 일반 규약을 잘 지킨 클래스라면 다른 어떤 클래스를 상속했든 같은 타입으로 취급된다.

---

## 인터페이스의 장점

### 1. 기존 클래스에도 손쉽게 새로운 인터페이스를 구현해넣을 수 있다

```java
// 기존에 존재하던 String 클래스
public final class String implements Serializable, Comparable<String>, CharSequence {
    // 기존 구현...
}

// 새로운 인터페이스 정의
public interface Printable {
    void print();
}

// 기존 클래스를 확장하여 새로운 인터페이스 구현
public class PrintableString extends String implements Printable {
    public PrintableString(String str) {
        super(str);  // String은 final이므로 실제로는 불가능하지만 예시용
    }
    
    @Override
    public void print() {
        System.out.println(this);
    }
}

// 실제 가능한 예시: 기존 클래스에 인터페이스 추가
public class Person implements Comparable<Person> {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Comparable 인터페이스를 나중에 추가로 구현
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
    
    // toString, equals, hashCode 등...
}
```

### 2. 인터페이스는 믹스인(mixin) 정의에 안성맞춤이다

믹스인이란 클래스가 구현할 수 있는 타입으로, 믹스인을 구현한 클래스에 원래의 '주된 타입' 외에도 특정한 선택적 행위를 제공한다고 선언하는 효과를 준다.

```java
// 믹스인 인터페이스들
public interface Serializable {
    // 직렬화 가능함을 나타내는 마커 인터페이스
}

public interface Cloneable {
    // 복제 가능함을 나타내는 마커 인터페이스
}

public interface Comparable<T> {
    int compareTo(T other);  // 비교 가능함을 나타냄
}

// 주된 타입: 학생
public class Student implements Serializable, Cloneable, Comparable<Student> {
    private String name;
    private int studentId;
    private double gpa;
    
    public Student(String name, int studentId, double gpa) {
        this.name = name;
        this.studentId = studentId;
        this.gpa = gpa;
    }
    
    // 주된 기능
    public void study() {
        System.out.println(name + "이(가) 공부하고 있습니다.");
    }
    
    // 믹스인 기능 1: 비교 가능
    @Override
    public int compareTo(Student other) {
        return Double.compare(this.gpa, other.gpa);
    }
    
    // 믹스인 기능 2: 복제 가능
    @Override
    public Student clone() {
        try {
            return (Student) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();  // 일어날 수 없는 일
        }
    }
    
    // 믹스인 기능 3: 직렬화 가능 (자동 처리)
    
    @Override
    public String toString() {
        return String.format("Student{name='%s', id=%d, gpa=%.2f}", name, studentId, gpa);
    }
}

// 사용 예시
public class MixinExample {
    public static void main(String[] args) {
        Student student1 = new Student("김철수", 20210001, 3.8);
        Student student2 = new Student("이영희", 20210002, 3.9);
        
        // 주된 기능
        student1.study();
        
        // 믹스인 기능들 활용
        System.out.println("성적 비교: " + student1.compareTo(student2));  // Comparable
        Student cloned = student1.clone();  // Cloneable
        System.out.println("복제된 학생: " + cloned);
        
        // 직렬화도 가능 (Serializable)
        // ObjectOutputStream 등으로 직렬화 가능
    }
}
```

### 3. 인터페이스로는 계층구조가 없는 타입 프레임워크를 만들 수 있다

```java
// 가수와 작곡가 인터페이스
public interface Singer {
    AudioClip sing(Song s);
}

public interface Songwriter {
    Song compose(int chartPosition);
}

// 싱어송라이터: 두 인터페이스를 모두 구현
public interface SingerSongwriter extends Singer, Songwriter {
    AudioClip strum();
    void actSensitive();
}

// 실제 구현
public class PopStar implements SingerSongwriter {
    @Override
    public AudioClip sing(Song s) {
        System.out.println("노래를 부릅니다: " + s.getTitle());
        return new AudioClip(s);
    }
    
    @Override
    public Song compose(int chartPosition) {
        System.out.println("차트 " + chartPosition + "위 곡을 작곡합니다");
        return new Song("Hit Song #" + chartPosition);
    }
    
    @Override
    public AudioClip strum() {
        System.out.println("기타를 튕깁니다");
        return new AudioClip("guitar_strum.mp3");
    }
    
    @Override
    public void actSensitive() {
        System.out.println("감성적으로 연기합니다...");
    }
}

// 추상 클래스로 했다면?
public abstract class AbstractSinger {
    public abstract AudioClip sing(Song s);
}

public abstract class AbstractSongwriter {
    public abstract Song compose(int chartPosition);
}

// 다중 상속이 불가능하므로 조합 폭발 문제 발생
public abstract class AbstractSingerSongwriter extends AbstractSinger {
    // AbstractSongwriter의 기능을 어떻게 가져올까?
    // 위임(delegation)을 사용해야 함 - 복잡해짐
    private AbstractSongwriter songwriter = new ConcreteAbstractSongwriter();
    
    public Song compose(int chartPosition) {
        return songwriter.compose(chartPosition);
    }
    
    public abstract AudioClip strum();
    public abstract void actSensitive();
}
```

---

## 디폴트 메서드의 활용

### 기본 구현 제공으로 프로그래머의 일감 덜어주기

```java
public interface List<E> extends Collection<E> {
    // 추상 메서드들
    int size();
    boolean isEmpty();
    boolean add(E e);
    boolean remove(Object o);
    E get(int index);
    
    // 디폴트 메서드: 기본 구현 제공
    default void sort(Comparator<? super E> c) {
        Collections.sort(this, c);
    }
    
    default void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }
    
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.ORDERED);
    }
}

// 사용 예시
public class SimpleArrayList<E> implements List<E> {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_CAPACITY = 10;
    
    public SimpleArrayList() {
        elements = new Object[DEFAULT_CAPACITY];
    }
    
    // 필수 메서드들만 구현하면 됨
    @Override
    public int size() { return size; }
    
    @Override
    public boolean isEmpty() { return size == 0; }
    
    @Override
    public boolean add(E e) {
        ensureCapacity();
        elements[size++] = e;
        return true;
    }
    
    @Override
    public boolean remove(Object o) {
        for (int i = 0; i < size; i++) {
            if (Objects.equals(o, elements[i])) {
                System.arraycopy(elements, i+1, elements, i, size-i-1);
                elements[--size] = null;
                return true;
            }
        }
        return false;
    }
    
    @Override
    @SuppressWarnings("unchecked")
    public E get(int index) {
        if (index >= size) throw new IndexOutOfBoundsException();
        return (E) elements[index];
    }
    
    private void ensureCapacity() {
        if (size >= elements.length) {
            elements = Arrays.copyOf(elements, elements.length * 2);
        }
    }
    
    // sort(), replaceAll(), spliterator() 등은 
    // 인터페이스의 디폴트 메서드를 자동으로 상속받음!
}

// 사용
public class DefaultMethodExample {
    public static void main(String[] args) {
        List<String> list = new SimpleArrayList<>();
        list.add("banana");
        list.add("apple");
        list.add("cherry");
        
        // 디폴트 메서드 사용
        list.sort(String::compareTo);  // 기본 구현 활용
        System.out.println(list);  // [apple, banana, cherry]
        
        list.replaceAll(String::toUpperCase);  // 기본 구현 활용
        System.out.println(list);  // [APPLE, BANANA, CHERRY]
    }
}
```

---

## 인터페이스와 추상 골격 구현(skeletal implementation) 클래스

### 템플릿 메서드 패턴

인터페이스의 장점과 추상 클래스의 장점을 모두 취하는 방법이다.

```java
// 인터페이스
public interface MyList<E> {
    // 필수 구현 메서드들
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    boolean add(E e);
    boolean remove(Object o);
    
    // 선택적 구현 메서드들 (디폴트 메서드로 기본 구현 제공)
    default boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c) {
            if (add(e)) {
                modified = true;
            }
        }
        return modified;
    }
    
    default void clear() {
        Iterator<E> it = iterator();
        while (it.hasNext()) {
            it.next();
            it.remove();
        }
    }
}

// 추상 골격 구현 클래스
public abstract class AbstractMyList<E> implements MyList<E> {
    
    // 인터페이스의 디폴트 메서드들을 활용하거나 재구현
    
    // 공통 구현: 기본 메서드들의 조합으로 구현
    @Override
    public boolean isEmpty() {
        return size() == 0;
    }
    
    @Override
    public boolean contains(Object o) {
        Iterator<E> it = iterator();
        if (o == null) {
            while (it.hasNext()) {
                if (it.next() == null) return true;
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) return true;
            }
        }
        return false;
    }
    
    @Override
    public boolean remove(Object o) {
        Iterator<E> it = iterator();
        if (o == null) {
            while (it.hasNext()) {
                if (it.next() == null) {
                    it.remove();
                    return true;
                }
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) {
                    it.remove();
                    return true;
                }
            }
        }
        return false;
    }
    
    // 하위 클래스에서 구현해야 할 핵심 메서드들은 추상으로 남김
    // abstract int size();
    // abstract Iterator<E> iterator();
    // abstract boolean add(E e);
}

// 구체적인 구현 클래스
public class ArrayMyList<E> extends AbstractMyList<E> {
    private Object[] elements;
    private int size = 0;
    
    public ArrayMyList() {
        elements = new Object[10];
    }
    
    // 핵심 메서드들만 구현
    @Override
    public int size() {
        return size;
    }
    
    @Override
    public Iterator<E> iterator() {
        return new Iterator<E>() {
            private int index = 0;
            
            @Override
            public boolean hasNext() {
                return index < size;
            }
            
            @Override
            @SuppressWarnings("unchecked")
            public E next() {
                if (!hasNext()) throw new NoSuchElementException();
                return (E) elements[index++];
            }
            
            @Override
            public void remove() {
                if (index <= 0) throw new IllegalStateException();
                ArrayMyList.this.removeAt(--index);
            }
        };
    }
    
    @Override
    public boolean add(E e) {
        ensureCapacity();
        elements[size++] = e;
        return true;
    }
    
    private void removeAt(int index) {
        System.arraycopy(elements, index + 1, elements, index, size - index - 1);
        elements[--size] = null;
    }
    
    private void ensureCapacity() {
        if (size >= elements.length) {
            elements = Arrays.copyOf(elements, elements.length * 2);
        }
    }
}
```

### 골격 구현 활용의 실제 예시

```java
// Map.Entry 인터페이스를 구현하는 골격 구현
public abstract class AbstractMapEntry<K, V> implements Map.Entry<K, V> {
    
    // 변경 불가능한 엔트리를 위한 기본 구현
    @Override
    public V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    
    // Object의 일반 규약에 맞는 equals 구현
    @Override
    public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof Map.Entry)) return false;
        
        Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
        return Objects.equals(getKey(), e.getKey()) &&
               Objects.equals(getValue(), e.getValue());
    }
    
    // Object의 일반 규약에 맞는 hashCode 구현
    @Override
    public int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }
    
    @Override
    public String toString() {
        return getKey() + "=" + getValue();
    }
}

// 간단한 구현 클래스
public class SimpleEntry<K, V> extends AbstractMapEntry<K, V> {
    private final K key;
    private V value;
    
    public SimpleEntry(K key, V value) {
        this.key = key;
        this.value = value;
    }
    
    @Override
    public K getKey() {
        return key;
    }
    
    @Override
    public V getValue() {
        return value;
    }
    
    @Override
    public V setValue(V value) {
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }
}
```

---

## 시뮬레이트한 다중 상속 (simulated multiple inheritance)

골격 구현을 우회적으로 이용하는 방법이다.

```java
// 인터페이스
public interface Musician {
    void play();
    void practice();
    
    default void perform() {
        practice();
        play();
        System.out.println("공연 완료!");
    }
}

// 골격 구현
public abstract class AbstractMusician implements Musician {
    private String name;
    
    protected AbstractMusician(String name) {
        this.name = name;
    }
    
    @Override
    public void practice() {
        System.out.println(name + "이(가) 연습합니다.");
    }
    
    protected String getName() {
        return name;
    }
    
    // play()는 하위 클래스에서 구현
}

// 다른 클래스를 상속받아야 하는 상황
public class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    public void eat() {
        System.out.println(name + "이(가) 식사합니다.");
    }
}

// 시뮬레이트한 다중 상속
public class MusicianPerson extends Person implements Musician {
    // 골격 구현을 감싼 내부 클래스
    private class MusicianHelper extends AbstractMusician {
        MusicianHelper() {
            super(MusicianPerson.this.getName());
        }
        
        @Override
        public void play() {
            System.out.println(getName() + "이(가) 연주합니다!");
        }
    }
    
    private final MusicianHelper musicianHelper = new MusicianHelper();
    
    public MusicianPerson(String name, int age) {
        super(name, age);
    }
    
    // Musician 인터페이스의 메서드들을 위임
    @Override
    public void play() {
        musicianHelper.play();
    }
    
    @Override
    public void practice() {
        musicianHelper.practice();
    }
    
    // Person의 기능도 유지
    public void introduceMyself() {
        System.out.println("안녕하세요, 저는 " + getName() + "이고 " + getAge() + "살입니다.");
        eat();
        perform();  // Musician의 기본 구현 사용
    }
}

// 사용 예시
public class SimulatedInheritanceExample {
    public static void main(String[] args) {
        MusicianPerson person = new MusicianPerson("김뮤직", 25);
        
        person.introduceMyself();
        // 출력:
        // 안녕하세요, 저는 김뮤직이고 25살입니다.
        // 김뮤직이(가) 식사합니다.
        // 김뮤직이(가) 연습합니다.
        // 김뮤직이(가) 연주합니다!
        // 공연 완료!
    }
}
```

---

## 단순 구현(simple implementation)

골격 구현의 작은 변종으로, 동작하는 가장 단순한 구현이다.

```java
public class ArrayMap<K, V> extends AbstractMap<K, V> {
    private final K[] keys;
    private final V[] values;
    private int size = 0;
    
    @SuppressWarnings("unchecked")
    public ArrayMap(int capacity) {
        keys = (K[]) new Object[capacity];
        values = (V[]) new Object[capacity];
    }
    
    @Override
    public Set<Entry<K, V>> entrySet() {
        return new AbstractSet<Entry<K, V>>() {
            @Override
            public Iterator<Entry<K, V>> iterator() {
                return new Iterator<Entry<K, V>>() {
                    private int index = 0;
                    
                    @Override
                    public boolean hasNext() {
                        return index < size;
                    }
                    
                    @Override
                    public Entry<K, V> next() {
                        if (!hasNext()) throw new NoSuchElementException();
                        return new SimpleImmutableEntry<>(keys[index], values[index++]);
                    }
                };
            }
            
            @Override
            public int size() {
                return ArrayMap.this.size;
            }
        };
    }
    
    @Override
    public V put(K key, V value) {
        // 기존 키 찾기
        for (int i = 0; i < size; i++) {
            if (Objects.equals(keys[i], key)) {
                V oldValue = values[i];
                values[i] = value;
                return oldValue;
            }
        }
        
        // 새 키-값 쌍 추가
        if (size >= keys.length) {
            throw new IllegalStateException("배열이 가득 참");
        }
        keys[size] = key;
        values[size] = value;
        size++;
        return null;
    }
}
```

---

## 정리

1. **일반적으로 다중 구현용 타입으로는 인터페이스가 가장 적합하다**
   - 기존 클래스에 쉽게 새 인터페이스를 구현해넣을 수 있다
   - 믹스인 정의에 안성맞춤이다
   - 계층구조가 없는 타입 프레임워크를 만들 수 있다

2. **복잡한 인터페이스라면 구현하는 수고를 덜어주는 골격 구현을 함께 제공하는 방법을 꼭 고려해보자**
   - 골격 구현은 '가능한 한' 인터페이스의 디폴트 메서드로 제공하여 그 인터페이스를 구현한 모든 곳에서 활용하도록 하는 것이 좋다
   - '가능한 한'이라고 한 이유는, 인터페이스에 걸려 있는 구현상의 제약 때문에 골격 구현을 추상 클래스로 제공하는 경우가 더 흔하기 때문이다

3. **단순 구현도 골격 구현의 작은 변종**이다
   - AbstractMap.SimpleEntry가 좋은 예다

4. **인터페이스의 메서드 중 구현 방법이 명백한 것이 있다면, 그 구현을 디폴트 메서드로 제공해 프로그래머들의 일감을 덜어주자**
   - 단, @implSpec 자바독 태그를 붙여 문서화해야 한다는 것을 잊지 말자

인터페이스는 타입을 정의하는 용도로만 사용해야 한다. 상수 공개용 수단으로 사용하지 말자!