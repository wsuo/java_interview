# Java集合框架详解

## 1. 集合框架概述

### 1.1 集合框架层次结构

```
Collection
├── List (有序、可重复)
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector
├── Set (无序、不可重复)
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet
└── Queue (队列)
    ├── PriorityQueue
    └── Deque
        └── ArrayDeque

Map (键值对)
├── HashMap
├── LinkedHashMap
├── TreeMap
├── ConcurrentHashMap
└── Hashtable
```

### 1.2 集合选择指南

| 需求 | 推荐集合 | 原因 |
|------|----------|------|
| 频繁随机访问 | ArrayList | O(1)随机访问 |
| 频繁插入删除 | LinkedList | O(1)插入删除 |
| 去重且无序 | HashSet | O(1)查找 |
| 去重且有序 | TreeSet | 自动排序 |
| 键值对存储 | HashMap | O(1)查找 |
| 有序键值对 | TreeMap | 自动排序 |
| 线程安全 | ConcurrentHashMap | 高并发性能 |

## 2. List接口详解

### 2.1 ArrayList

**实现原理：**
- 基于动态数组实现
- 默认初始容量为10
- 扩容机制：容量扩展为原来的1.5倍
- 线程不安全

**源码分析：**
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    private static final int DEFAULT_CAPACITY = 10;
    transient Object[] elementData; // 存储数据的数组
    private int size; // 元素个数
    
    // 扩容方法
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5倍扩容
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
}
```

**性能特点：**
- 查找：O(1)
- 插入：O(n)（需要移动元素）
- 删除：O(n)（需要移动元素）

**使用场景：**
```java
// 适用场景：大量随机访问
List<String> list = new ArrayList<>();
for (int i = 0; i < 1000000; i++) {
    list.add("item" + i);
}

// 频繁随机访问
String item = list.get(500000); // O(1)

// 不适用场景：频繁插入删除
list.add(0, "newItem"); // O(n)，需要移动所有元素
```

### 2.2 LinkedList

**实现原理：**
- 基于双向链表实现
- 实现了List和Deque接口
- 每个节点包含数据、前驱和后继指针
- 线程不安全

**源码分析：**
```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    
    transient int size = 0;
    transient Node<E> first; // 头节点
    transient Node<E> last;  // 尾节点
    
    // 节点结构
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    
    // 添加元素
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
}
```

**性能特点：**
- 查找：O(n)
- 插入：O(1)（在已知位置）
- 删除：O(1)（在已知位置）

**使用场景：**
```java
// 适用场景：频繁插入删除
LinkedList<String> list = new LinkedList<>();
list.addFirst("first"); // O(1)
list.addLast("last");   // O(1)

// 用作栈
list.push("top");
String top = list.pop();

// 用作队列
list.offer("element");
String element = list.poll();
```

### 2.3 Vector

**特点：**
- 线程安全（synchronized）
- 扩容机制：默认2倍扩容
- 性能较差，现在很少使用

```java
// 线程安全，但性能差
Vector<String> vector = new Vector<>();
vector.add("item"); // synchronized方法

// 推荐使用
List<String> list = Collections.synchronizedList(new ArrayList<>());
// 或者使用CopyOnWriteArrayList
```

## 3. Set接口详解

### 3.1 HashSet

**实现原理：**
- 基于HashMap实现
- 元素作为HashMap的key存储
- 值为固定的PRESENT对象

**源码分析：**
```java
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable {
    
    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();
    
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
    
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
}
```

**使用场景：**
```java
// 去重
Set<String> uniqueItems = new HashSet<>();
uniqueItems.add("item1");
uniqueItems.add("item1"); // 不会重复添加

// 快速查找
boolean exists = uniqueItems.contains("item1"); // O(1)

// 集合运算
Set<String> set1 = new HashSet<>(Arrays.asList("a", "b", "c"));
Set<String> set2 = new HashSet<>(Arrays.asList("b", "c", "d"));

// 交集
set1.retainAll(set2); // {b, c}

// 并集
set1.addAll(set2); // {a, b, c, d}

// 差集
set1.removeAll(set2); // {a}
```

### 3.2 LinkedHashSet

**特点：**
- 维护插入顺序
- 基于LinkedHashMap实现
- 性能略低于HashSet

```java
// 保持插入顺序
Set<String> linkedSet = new LinkedHashSet<>();
linkedSet.add("first");
linkedSet.add("second");
linkedSet.add("third");

// 遍历时保持插入顺序
for (String item : linkedSet) {
    System.out.println(item); // first, second, third
}
```

### 3.3 TreeSet

**实现原理：**
- 基于红黑树实现
- 元素自动排序
- 实现NavigableSet接口

**使用场景：**
```java
// 自动排序
TreeSet<Integer> treeSet = new TreeSet<>();
treeSet.add(5);
treeSet.add(2);
treeSet.add(8);
treeSet.add(1);

System.out.println(treeSet); // [1, 2, 5, 8]

// 范围查询
SortedSet<Integer> subSet = treeSet.subSet(2, 8); // [2, 5]
Integer lower = treeSet.lower(5); // 2
Integer higher = treeSet.higher(5); // 8

// 自定义排序
TreeSet<Person> personSet = new TreeSet<>((p1, p2) -> 
    Integer.compare(p1.getAge(), p2.getAge()));
```

## 4. Map接口详解

### 4.1 HashMap

**实现原理：**
- 基于哈希表实现
- JDK 1.8后：数组 + 链表 + 红黑树
- 链表长度超过8时转换为红黑树

**源码分析：**
```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {
    
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8; // 链表转红黑树阈值
    
    transient Node<K,V>[] table; // 哈希表
    transient int size; // 元素个数
    int threshold; // 扩容阈值
    final float loadFactor; // 负载因子
    
    // 节点结构
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
    
    // 哈希函数
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
    // 获取索引
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            
            if (first.hash == hash && // 检查第一个节点
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            
            if ((e = first.next) != null) {
                if (first instanceof TreeNode) // 红黑树查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                    
                do { // 链表查找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
}
```

**扩容机制：**
```java
// 扩容条件：size >= threshold (capacity * loadFactor)
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 阈值也翻倍
    }
    // ... 重新计算索引并移动元素
}
```

**性能优化：**
```java
// 1. 合理设置初始容量和负载因子
Map<String, String> map = new HashMap<>(64, 0.75f);

// 2. 重写hashCode和equals方法
public class Person {
    private String name;
    private int age;
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person person = (Person) obj;
        return age == person.age && Objects.equals(name, person.name);
    }
}

// 3. 使用不可变对象作为key
Map<String, String> map = new HashMap<>();
map.put("immutableKey", "value"); // String是不可变的
```

### 4.2 LinkedHashMap

**特点：**
- 维护插入顺序或访问顺序
- 基于HashMap + 双向链表实现
- 可用于实现LRU缓存

**LRU缓存实现：**
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries;
    
    public LRUCache(int maxEntries) {
        super(16, 0.75f, true); // accessOrder=true
        this.maxEntries = maxEntries;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries;
    }
}

// 使用
LRUCache<String, String> cache = new LRUCache<>(3);
cache.put("1", "one");
cache.put("2", "two");
cache.put("3", "three");
cache.get("1"); // 访问，移到末尾
cache.put("4", "four"); // 会删除最久未使用的元素
```

### 4.3 TreeMap

**实现原理：**
- 基于红黑树实现
- 键自动排序
- 实现NavigableMap接口

**使用场景：**
```java
// 自动排序
TreeMap<Integer, String> treeMap = new TreeMap<>();
treeMap.put(3, "three");
treeMap.put(1, "one");
treeMap.put(2, "two");

System.out.println(treeMap); // {1=one, 2=two, 3=three}

// 范围查询
SortedMap<Integer, String> subMap = treeMap.subMap(1, 3); // {1=one, 2=two}
Map.Entry<Integer, String> firstEntry = treeMap.firstEntry(); // 1=one
Map.Entry<Integer, String> lastEntry = treeMap.lastEntry(); // 3=three

// 邻近查询
Integer lowerKey = treeMap.lowerKey(2); // 1
Integer higherKey = treeMap.higherKey(2); // 3
```

### 4.4 ConcurrentHashMap

**实现原理：**
- JDK 1.7：分段锁（Segment）
- JDK 1.8：CAS + synchronized
- 线程安全，高并发性能

**JDK 1.8实现：**
```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentMap<K,V>, Serializable {
    
    // 使用volatile保证可见性
    transient volatile Node<K,V>[] table;
    private transient volatile int sizeCtl;
    
    // CAS操作
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 空桶，使用CAS插入
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                    break;
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) { // 锁住头节点
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            // 链表操作
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeNode)
                            // 红黑树操作
                            oldVal = ((TreeNode<K,V>)f).putTreeVal(hash, key, value);
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
}
```

**使用建议：**
```java
// 高并发场景
ConcurrentHashMap<String, String> concurrentMap = new ConcurrentHashMap<>();

// 原子操作
concurrentMap.putIfAbsent("key", "value");
concurrentMap.computeIfAbsent("key", k -> "newValue");
concurrentMap.merge("key", "value", (oldVal, newVal) -> oldVal + newVal);

// 批量操作
concurrentMap.forEach((k, v) -> System.out.println(k + "=" + v));
concurrentMap.reduce(1, (k, v) -> k + "=" + v, (s1, s2) -> s1 + ", " + s2);
```

## 5. 面试常见问题

### Q1: ArrayList和LinkedList的区别？

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 插入删除 | O(n) | O(1) |
| 内存占用 | 较小 | 较大（额外指针） |
| 缓存友好性 | 好 | 差 |

### Q2: HashMap的扩容机制？

1. **触发条件**：size >= threshold（capacity * loadFactor）
2. **扩容过程**：
   - 创建新数组，容量为原来的2倍
   - 重新计算所有元素的位置
   - 移动元素到新数组
3. **优化**：JDK 1.8中，链表节点要么保持原位置，要么移动到原位置+原容量的位置

### Q3: 如何解决HashMap的线程安全问题？

```java
// 方案1：使用Collections.synchronizedMap()
Map<String, String> syncMap = Collections.synchronizedMap(new HashMap<>());

// 方案2：使用ConcurrentHashMap
ConcurrentHashMap<String, String> concurrentMap = new ConcurrentHashMap<>();

// 方案3：使用外部锁
private final Object lock = new Object();
synchronized(lock) {
    map.put("key", "value");
}
```

### Q4: HashMap和Hashtable的区别？

| 特性 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 否 | 是 |
| null值 | 允许 | 不允许 |
| 继承关系 | AbstractMap | Dictionary |
| 初始容量 | 16 | 11 |
| 扩容机制 | 2倍 | 2倍+1 |

### Q5: 如何实现一个线程安全的HashMap？

```java
// 方案1：读写锁
public class ThreadSafeHashMap<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();
    
    public V get(K key) {
        readLock.lock();
        try {
            return map.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    public V put(K key, V value) {
        writeLock.lock();
        try {
            return map.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}

// 方案2：CopyOnWriteMap（读多写少场景）
public class CopyOnWriteHashMap<K, V> {
    private volatile Map<K, V> map = new HashMap<>();
    private final Object lock = new Object();
    
    public V get(K key) {
        return map.get(key);
    }
    
    public V put(K key, V value) {
        synchronized (lock) {
            Map<K, V> newMap = new HashMap<>(map);
            V oldValue = newMap.put(key, value);
            map = newMap;
            return oldValue;
        }
    }
}
```

## 6. 性能测试和选择建议

### 6.1 性能测试代码
```java
public class CollectionPerformanceTest {
    
    public static void main(String[] args) {
        int size = 100000;
        
        // 测试List性能
        testListPerformance(size);
        
        // 测试Set性能
        testSetPerformance(size);
        
        // 测试Map性能
        testMapPerformance(size);
    }
    
    private static void testListPerformance(int size) {
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        
        // 添加性能测试
        long start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
        }
        System.out.println("ArrayList add: " + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < size; i++) {
            linkedList.add(i);
        }
        System.out.println("LinkedList add: " + (System.currentTimeMillis() - start));
        
        // 随机访问性能测试
        Random random = new Random();
        start = System.currentTimeMillis();
        for (int i = 0; i < 1000; i++) {
            arrayList.get(random.nextInt(size));
        }
        System.out.println("ArrayList get: " + (System.currentTimeMillis() - start));
        
        start = System.currentTimeMillis();
        for (int i = 0; i < 1000; i++) {
            linkedList.get(random.nextInt(size));
        }
        System.out.println("LinkedList get: " + (System.currentTimeMillis() - start));
    }
}
```

### 6.2 选择建议

**List选择：**
- 大量随机访问：ArrayList
- 频繁插入删除：LinkedList
- 线程安全：Vector或Collections.synchronizedList()

**Set选择：**
- 高性能去重：HashSet
- 保持顺序：LinkedHashSet
- 自动排序：TreeSet

**Map选择：**
- 通用场景：HashMap
- 保持顺序：LinkedHashMap
- 自动排序：TreeMap
- 线程安全：ConcurrentHashMap