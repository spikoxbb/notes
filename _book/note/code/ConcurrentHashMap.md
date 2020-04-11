# ConcurrentHashMap

[TOC]



## jdk7

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
    // 默认segment个数
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    // 默认的更新map的并发级别，即默认最多允许16个线程进行并发写，  
    // 这个级别与segment的个数一致
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    /**
     * The segments, each of which is a specialized hash table.
     */
    final Segment<K,V>[] segments;
    // 其他省略
}
```



```csharp
static final class Segment<K, V> extends ReentrantLock implements Serializable {
    private static final long serialVersionUID = 2249069246763182397L;

    // 和 HashMap 中的 HashEntry 作用一样，真正存放数据的桶
    // HashEntry 的组成和 HashMap 非常类似，唯一的区别就是其中的核心数据如 value ，以及链表都是 volatile 修饰的，保证了获取时的可见性。
    transient volatile HashEntry<K, V>[] table;
    // segment中hashtable的元素数量计数器，用于size方法中，分段计算汇总
    transient int count;
   // 执行更新操作时，获取segment锁的重试次数，多核CPU重试64次，单核CPU重试1次
    static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
    ......
}
```

构造方法如下：

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;// segments数组的大小
        // 根据并发级别算出segments的大小以及2的指数
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;// segment内部HashEntry的默认容量，为2
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]，即创建segments数组，
        // 并初始化segments[0]元素的HashEntry数组
        Segment<K,V> s0 = new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        // UNSAFE为sun.misc.Unsafe对象，使用CAS操作，
        // 将segments[0]的元素替换为已经初始化的s0，保证原子性。
        // Unsafe类采用C++语言实现，底层实现CPU的CAS指令操作，保证原子性。
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
```

```java
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        // 取前N位与2^N按位与
        int j = (hash >>> segmentShift) & segmentMask;
        // (j << SSHIFT) + SBASE)这段代码也是为了定位段下标。
        // 如果通过Unsafe类的CAS读取段下标元素，元素没有初始化，
        // 则调用ensureSegment进行初始化
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            //检查下标segment是否已经初始化，如果没有初始化，则调用ensureSegment进行初始化，内部用了CAS操作进行替换，达到初始化效果。初始化的过程进行了双重检查，UNSAFE.getObjectVolatile，通过这个方法执行了两次，以检查segment是否已经初始化，以及用UNSAFE.compareAndSwapObject进行CAS替换，CAS的替换有失败的可能，因此源码中还加了自旋重试的操作，保证最终CAS操作的成功。
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
}

```

segment类的put方法：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 调用父类ReentrantLock的tryLock()方法尝试锁住segment，
    // 获取锁失败，则调用scanAndLockForPut方法自旋重试获取锁
    //在scanAndLockForPut方法中，会通过重复执行tryLock()方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行tryLock()方法的次数超过上限时，则执行lock()方法挂起线程；这样设计目的是为了让线程切换和自旋消耗的CPU的时间达到平衡，不至于白白浪费CPU，也不会过于平凡切换线程导致更多的CPU浪费。
    HashEntry<K,V> node = tryLock()?null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                //更新segment的count计数器，用于size方法中计算map元素个数时不用对每个segment内部HashEntry遍历重新计算，提高性能。
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

get方法不加锁，并用Unsafe.getObjectVolatile方法读取元素，这个方法保证读取对象永远是最新的:

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // 超过重复计算的次数，采用全部加锁后计算
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            // 汇总每个segment的count计数器
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 判断连续两次计算结果是否相等
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        // 释放所有segment上的锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```

由于ConcurrentHashMap支持并发读写，所以在准确计算元素时存在一定的难度，思路如下：

> 1、重复汇总计算segments元素个数，即将每个segment的计数器count累加汇总。
> 2、如果连续两次计算的sum值相同，则结束计算，认为并发过程中计算的值正确，并返回。
> 3、如果重复三次，计算的结果都不相同，则强制锁住全部segment后，重新计算值。

虽然采用重复计算和最后加锁的方式再次计算，但size方法仍不能保证结果的准确性，例如，两次计算结果相等，在返回之前，又有新的线程插入新值，则此时的结果就是不准确的。所以，在并发读写环境下，size方法进行精确计算的意义并不大，只能作为一个大概的计算结果，因此也决定了在使用size方法时，要评估清楚自己的需求是什么，是精确计算，还是粗略计算。

## jdk8

JDK 1.8中ConcurrentHashMap抛弃了分段锁技术的实现，直接采用CAS + synchronized保证并发更新的安全性，底层采用数组+链表+红黑树的存储结构。

Java8中主要做了如下优化:
1.将Segment抛弃掉了，直接采用Node（继承自Map.Entry）作为table元素。
2.修改时，不再采用ReentrantLock加锁，直接用内置synchronized加锁，java8的内置锁比之前版本优化了很多，相较ReentrantLock，性能不并差。
3.size方法优化，增加了CounterCell内部类，用于并行计算每个bucket的元素数量。

## 弱一致性

 同一个Segment实例中的put操作是加了锁的，而对应的get却没有。  put操作可以分为两种情况，一是key已经存在，修改对应的value；二是key不存在，将一个新的Entry加入底层数据结构。 

### put-get

- key已经存在的情况：if (e != null)部分，前面已经说过HashEntry的value是个volatile变量，当线程1给value赋值后，会立马对执行get的线程2可见，而不用等到put方法结束。

- key不存在的情况：![](E:\notes\img\1582011468(1).png) tab变量是一个普通的变量，虽然给它赋值的是volatile的table。另外，虽然引用类型（数组类型）的变量table是volatile的，但table中的元素不是volatile的，因此⑧只是一个普通的写操作 。

  ![](E:\notes\img\1582011560(1).png)

 也就是说，如果某个Segment实例中的put将一个Entry加入到了table中，在未执行count赋值（ volatile写 ）操作之前有另一个线程执行了同一个Segment实例中的get，来获取这个刚加入的Entry中的value，那么是有可能取不到的！。

### clear

```java
public void clear() {
    for (int i = 0; i < segments.length; ++i)
        segments[i].clear();
}
```

因为没有全局的锁，在清除完一个segments之后，正在清理下一个segments的时候，已经清理segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。

###  **迭代器** 

ConcurrentHashMap中的迭代器主要包括entrySet、keySet、values方法。它们大同小异，这里选择entrySet解释。当我们调用entrySet返回值的iterator方法时，返回的是EntryIterator，在EntryIterator上调用next方法时，最终实际调用到了HashIterator.advance()方法，看下这个方法：

final void advance() {
    if (nextEntry != null && (nextEntry = nextEntry.next) != null)
        return;

```java
while (nextTableIndex >= 0) {
    if ( (nextEntry = currentTable[nextTableIndex--]) != null)
        return;
}

while (nextSegmentIndex >= 0) {
    Segment<K,V> seg = segments[nextSegmentIndex--];
    if (seg.count != 0) {
        currentTable = seg.table;
        for (int j = currentTable.length - 1; j >= 0; --j) {
            if ( (nextEntry = currentTable[j]) != null) {
                nextTableIndex = j - 1;
                return;
            }
        }
    }
}
```

这个方法在遍历底层数组。在遍历过程中，如果已经遍历的数组上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中。
