## 介绍一下 Java 中的集合

Java 中的集合分为 2 个大接口：Map 和 Collection。

Collection 下又有 3 个主要接口：List, Set 和 Queue。
Collection 存储单列对象，Map 存储键值对对象。

---

## ArrayList 和 Vector 的区别？

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309190559388.jpg)

他们底层都使用 `Object[]` 存储数据，不过 ArrayList 是线程不安全的，Vector 是线程安全的。

Vector 是 List 的古老实现类，不推荐使用。如果要使用线程安全的 ArrayList，可以使用 CopyOnWriteArrayList。

- 读写分离，适合读多写少的场景
- 读不加锁
- 写时，先将容器复制一份，在副本上修改数据，此时写操作是加锁的，然后将原容器的引用指向副本

---

## ArrayList 和 LinkedList 的区别？

ArrayList 底层使用 Object 数组存储数据，LinkedList 使用双向链表存储数据。

由于 ArrayList 的底层是数组，因此增删元素的时间复杂度受元素位置影响；LinkedList 增删元素的时间复杂度为 O(1)。

ArrayList 支持快速的随机元素访问，LinkedList 不支持。

他们都是线程不安全的。

---

## HashMap 的底层原理

- JDK7 中的 HashMap 底层通过数组+链表的方式实现
- JDK8 中的 HashMap 底层通过数组+链表/红黑树的方式实现

引入红黑树主要是为了提高查询效率，当哈希冲突严重时，链表过长，查询效率退化为 O(N)，树化之后查询效率会优化为 O(logN)。

[HashMap 面试指南](https://zhuanlan.zhihu.com/p/76735726)

---

## HashMap VS HashTable

线程安全性：
HashMap 线程不安全，HashTable 线程安全

是否允许空键和空值：

- HashMap 允许一个空键和若干空值
- HashTable 不允许任何空键和空值

关于初始化容量和扩容：

- HashMap 如果没有设定初始容量，则默认为 16，之后每次扩容为原来的 2 倍
- HashTable 如果没有设定初始容量，则默认为 11，之后每次扩容，容量变为原来的 2n + 1
- HashMap 如果设定初始容量，则会将其向上扩充为最近的 2 的 n 次幂大小
- HashTable 如果设定初始容量，则会使用初始容量

底层数据结构：

- 都使用数组+链表实现
- 不过 JDK1.8 之后，HashMap 优化为使用数组+链表/红黑树实现

---

## HashMap 为什么线程不安全？

HashMap 在并发编程环境下有什么问题?

- 多线程扩容，引起的死循环问题
- 多线程 put 的时候可能导致元素丢失
- put 非 null 元素后 get 出来的却是 null

> 在 jdk1.8 中还有这些问题么?

在 jdk1.8 中，死循环问题已经解决。其他两个问题还是存在。

---

## HashMap 如何实现线程安全？

- 使用 ConcurrentHashMap
- 使用 `Collections.synchronizedMap()` 将 HashMap 包装成线程安全的 Map

---

## TODO HashMap 源码相关问题

---

## TODO HashMap 调用 put() 流程

---

## 什么是 ConcurrentHashMap？

[JUC 集合: ConcurrentHashMap 详解](https://pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html)

ConcurrentHashMap 是线程安全版的 HashMap 实现，并且相比 HashTable 有更高的并发性能。

之所以不使用 HashTable 主要是因为 HashTable 简单粗暴地使用 synchronized 锁住整张表，锁的粒度太大，所有操作都需要竞争同一把锁，导致性能太低。

在 JDK1.7 的版本，ConcurrentHashMap 是使用分段锁的机制实现的。通过将 HashMap 的 Entry 划分成多个 Segment，然后分别对每个 Segment 上一把锁，使得一个线程访问一个段的数据时，访问其他段的线程不会被阻塞。

总结：在 JDK1.7 版本，ConcurrentHashMap 底层是一个 Segment 数组，每个 Segment 又是一个 HashEntry 数组。Segment 继承 ReentrantLock，扮演可重入锁的角色。

在 JDK1.8 的版本，ConcurrentHashMap 放弃了分段的实现，转而通过 Node 数组 + 链表/红黑树实现。其中并发控制通过 synchronized 和 CAS 实现。

---

## TODO ConcurrentHashMap 的 `get()`, `put()` 和扩容

[JUC 集合: ConcurrentHashMap 详解](https://pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html)

---

## 手写一个 HashMap

```java
public interface MyMap<K, V> {
    V get(K k);

    void put(K k, V v);

    int size();

    V remove(K k);

    boolean isEmpty();
}
```

```java
public class MyHashMap<K, V> implements MyMap<K, V> {
    class Entry<K, V> {
        K k;
        V v;
        Entry<K, V> next;

        public Entry(K k, V v) {
            this.k = k;
            this.v = v;
        }
    }

    final static int DEFAULT_CAPACITY = 16;
    final static float DEFAULT_LOAD_FACTOR = 0.75f;

    private int capcacity;
    private float loadFactor;
    private int size;

    Entry<K, V>[] table;

    public MyHashMap() {
        this(DEFAULT_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    public MyHashMap(int capcacity, float loadFactor) {
        this.capcacity = upperMinPowerOf2(capcacity);
        this.loadFactor = loadFactor;
        this.table = new Entry[capcacity];
    }

    private static int upperMinPowerOf2(int n) {
        int power = 1;
        while (power <= n) {
            power *= 2;
        }
        return power;
    }

    @Override
    public V get(K k) {
        int idx = k.hashCode() % table.length;
        Entry<K, V> cur = table[idx];
        while (cur != null) {
            if (cur.k == k) {
                return cur.v;
            }
            cur = cur.next;
        }
        return null;
    }

    @Override
    public void put(K k, V v) {
        int idx = k.hashCode() % table.length;
        Entry<K, V> cur = table[idx];
        if (cur == null) {
            table[idx] = new Entry<>(k, v);
            size++;
        } else {
            while (cur != null) {
                if (cur.k == k) {
                    cur.v = v;
                    return;
                }
                cur = cur.next;
            }
            Entry<K, V> to_add = new Entry<>(k, v);
            to_add.next = table[idx];
            table[idx] = to_add;
            size++;
        }
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public V remove(K k) {
        int idx = k.hashCode() % table.length;
        V res = null;
        Entry<K, V> cur = table[idx];
        Entry<K, V> pre = null;
        while (cur != null) {
            if (cur.k == k) {
                res = cur.v;
                size--;
                if (pre != null) {
                    pre.next = cur.next;
                } else {
                    table[idx] = cur.next;
                }
                return res;
            }
            pre = cur;
            cur = cur.next;
        }
        return null;
    }

    @Override
    public boolean isEmpty() {
        return size == 0;
    }
}
```
