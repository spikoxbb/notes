# ConcurrentHashMap

[TOC]

## jdk7

在一个实例中使用多个锁控制hashtable中不同的分段部分。在这里使用的是segment，每一个segment其实就是一个独立的小的hashtable，并且因为它继承了reentrantlock，所以每一个分段都已经拥有了锁。

结构性修改方法(比如put)会在确定分段后只锁住某一个segment，而不影响剩余段的操作。不过有一些方法存在跨段操作，比如size/containsValue等。这些方法就要**按顺序**锁住整个表来进行操作。



![截屏2020-12-02 上午12.54.45](../../../../img/截屏2020-12-02 上午12.54.45.png)

## 核心成员变量

```java
    /**
     * Segment 数组，存放数据时首先需要定位到具体的 Segment 中。
     */
    final Segment<K,V>[] segments;

    transient Set<K> keySet;
    transient Set<Map.Entry<K,V>> entrySet;
```

## Segment 是 ConcurrentHashMap 的一个内部类

```java
    static final class Segment<K,V> extends ReentrantLock implements Serializable {

        private static final long serialVersionUID = 2249069246763182397L;
        
        // 和 HashMap 中的 HashEntry 作用一样，真正存放数据的桶
        transient volatile HashEntry<K,V>[] table;

        transient int count;

        transient int modCount;

        transient int threshold;

        final float loadFactor;
        
    }
```

##  HashEntry

和 HashMap 非常类似，唯一的区别就是其中的核心数据如 value ，以及链表都是 volatile 修饰的，保证了获取时的可见性。

原理上来说：ConcurrentHashMap 采用了分段锁技术，其中 Segment 继承于 ReentrantLock。不会像 HashTable 那样不管是 put 还是 get 操作都需要做同步处理，理论上 ConcurrentHashMap 支持 CurrencyLevel (Segment 数组数量)的线程并发。每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

## put()

```java
public V put(K key, V value) {
  Segment<K,V> s;
  if (value == null)
    throw new NullPointerException();
  int hash = hash(key);
  int j = (hash >>> segmentShift) & segmentMask;
  if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null) 
    s = ensureSegment(j);
  return s.put(key, hash, value, false);
}
```

## size()

因为`ConcurrentHashMap`是可以并发插入数据的，所以在准确计算元素时存在一定的难度，一般的思路是统计每个`Segment`对象中的元素个数，然后进行累加，但是这种方式计算出来的结果并不一样的准确的，因为在计算后面几个`Segment`的元素个数时，已经计算过的`Segment`同时可能有数据的插入或则删除，在1.7的实现中，采用了如下方式：

```java
try {
    for (;;) {
        if (retries++ == RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                ensureSegment(j).lock(); // force creation
        }
        sum = 0L;
        size = 0;
        overflow = false;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                sum += seg.modCount;
                int c = seg.count;
                if (c < 0 || (size += c) < 0)
                    overflow = true;
            }
        }
        if (sum == last)
            break;
        last = sum;
    }
} finally {
    if (retries > RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
            segmentAt(segments, j).unlock();
    }
}
```

先采用不加锁的方式，连续计算元素的个数，最多计算3次：
 1、如果前后两次计算结果相同，则说明计算出来的元素个数是准确的；
 2、如果前后两次计算结果都不同，则给每个`Segment`进行加锁，再计算一次元素的个数；

## jdk8

没有继续沿用 1.7 segment 分段锁。而是采用了 CAS + synchronized 关键字实现。整个数组的结构上在取消了 segment 之后，又和 hashMap 有了一些相似度。

![截屏2020-12-02 上午12.55.13](../../../../img/截屏2020-12-02 上午12.55.13.png)

## initTable（）

只有在执行第一次`put`方法时才会调用`initTable()`初始化`Node`数组，实现如下：

```kotlin
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## put()

当执行`put`方法插入数据时，根据key的hash值，在`Node`数组中找到相应的位置，实现如下：

```csharp
如果相应位置的`Node`还未初始化，则通过CAS插入相应的数据；
else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
    if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
        break;                   // no lock when adding to empty bin
}
```



## 弱一致性

 同一个Segment实例中的put操作是加了锁的，而对应的get却没有。  put操作可以分为两种情况，一是key已经存在，修改对应的value；二是key不存在，将一个新的Entry加入底层数据结构。 

### put-get

- key已经存在的情况：if (e != null)部分，前面已经说过HashEntry的value是个volatile变量，当线程1给value赋值后，会立马对执行get的线程2可见，而不用等到put方法结束。

- key不存在的情况：![](E:\notes\img\1582011468(1).png) tab变量是一个普通的变量，虽然给它赋值的是volatile的table。另外，虽然引用类型（数组类型）的变量table是volatile的，但table中的元素不是volatile的，因此⑧只是一个普通的写操作 。

  ![](../../img/1582011560(1).png)

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
