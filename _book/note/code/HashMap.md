# HashMap

1. initialCapacity：初始容量。指的是 HashMap 集合初始化的时候自身的容量。可以在构造方法中指定；如果不指定的话，总容量默认值是 16 。需要注意的是初始容量必须是 2 的幂次方。
2. size：当前 HashMap 中已经存储着的键值对数量，即 `HashMap.size()` 。
3. loadFactor：加载因子。
4. threshold：扩容阀值。即 扩容阀值 = HashMap 总容量 * 加载因子。当前 HashMap 的容量大于或等于扩容阀值的时候就会去执行扩容。
5. table：Entry 数组。

```java
 // 默认的构造方法使用的都是默认的初始容量和加载因子
    // DEFAULT_INITIAL_CAPACITY = 16，DEFAULT_LOAD_FACTOR = 0.75
    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    // 可以指定初始容量，并且使用默认的加载因子
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap(int initialCapacity, float loadFactor) {
        // 对初始容量的值判断
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +                                               loadFactor);
        // 找到第一个大于等于initialCapacity的2的平方的数
        int capacity = 1; 
        while (capacity < initialCapacity)
            capacity <<= 1;
        // 设置加载因子
        this.loadFactor = loadFactor;
        threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        // 初始化table数组，这是HashMap真实的存储容器
        table = new Entry[capacity];
        useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        // 空方法
        init();
    }

public V put(K key, V value) {
        // 如果 table 数组为空时先创建数组，并且设置扩容阀值
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        // 如果 key 为空时，调用 putForNullKey 方法特殊处理
        if (key == null)
            return putForNullKey(value);
        // 计算 key 的哈希值
        int hash = hash(key);
        // 根据计算出来的哈希值和当前数组的长度计算在数组中的索引
        int i = indexFor(hash, table.length);
        // 先遍历该数组索引下的整条链表
        // 如果该 key 之前已经在 HashMap 中存储了的话，直接替换对应的 value 值即可
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        // 如果该 key 之前没有被存储过，那么就进入 addEntry 方法
        addEntry(hash, key, value, i);
        return null;
    }

//length为2的幂次方，即一定是偶数，偶数减1，即是奇数，这样保证了（length-1）在二进制中最低位是1，而&运算结果的最低位是1还是0完全取决于hash值二进制的最低位。如果length为奇数，则length-1则为偶数，则length-1二进制的最低位横为0，则&位运算的结果最低位横为0，即横为偶数。这样table数组就只可能在偶数下标的位置存储了数据，浪费了所有奇数下标的位置，这样也更容易产生hash冲突。
static int indexFor(int h, int length) {
        return h & (length-1);
    }

void addEntry(int hash, K key, V value, int bucketIndex) {
        // 当前容量大于或等于扩容阀值的时候，会执行扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            // 扩容为原来容量的两倍
            resize(2 * table.length);
            // 重新计算哈希值
            hash = (null != key) ? hash(key) : 0;
            // 重新得到在新数组中的索引
            bucketIndex = indexFor(hash, table.length);
        }
        // 创建节点
        createEntry(hash, key, value, bucketIndex);
    }


void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        // 当前 HashMap 的容量加 1
        size++;
    }

void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        // 创建新的 entry 数组
        Entry[] newTable = new Entry[newCapacity];
         // hash有关
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        // 这里进行异或运算，一般为true
        boolean rehash = oldAltHashing ^ useAltHashing;
        // 将旧 entry 数组中的数据复制到新 entry 数组中
        transfer(newTable,rehash);
        // 将新数组的引用赋给 table
        table = newTable;
        // 计算新的扩容阀值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        // 遍历老的table数组
        for (Entry<K,V> e : table) {
            // 遍历老table数组中存储每条单项链表
            while(null != e) {
                // 取出老table中每个Entry
                Entry<K,V> next = e.next;
                if (rehash) {
                    //重新计算hash
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                // 根据hash值，算出老table中的Entry应该在新table中存储的index
                int i = indexFor(e.hash, newCapacity);
                // 让老table转移的Entry的next指向新table中它应该存储的位置
                // 即插入到了新table中index处单链表的表头
                e.next = newTable[i];
                // 将老table取出的entry，放入到新table中
                newTable[i] = e;
                // 继续取老talbe的下一个Entry
                e = next;
            }
        }
    }

public V get(Object key) {
        // 如果 key 是空的，就调用 getForNullKey 方法特殊处理
        if (key == null)
            return getForNullKey();
        // 获取 key 相对应的 entry 
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }

  final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }
        // 计算 key 的哈希值
        int hash = (key == null) ? 0 : hash(key);
        // 得到数组的索引，然后遍历链表，查看是否有相同 key 的 Entry
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        // 没有的话，返回 null
        return null;
    }

public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

final Entry<K,V> removeEntryForKey(Object key) {
        // 算出hash
        int hash = (key == null) ? 0 : hash(key);
        // 得到在table中的index
        int i = indexFor(hash, table.length);
        // 当前结点的上一个结点，初始为table[index]上单向链表的头结点
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            // 得到下一个结点
            Entry<K,V> next = e.next;
            Object k;
            // 如果找到了删除的结点
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                // 如果是table上的单向链表的头结点，则直接让把该结点的next结点放到头结点
                if (prev == e)
                    table[i] = next;
                else
                    // 如果不是单向链表的头结点，则把上一个结点的next指向本结点的next
                    prev.next = next;  
                // 空实现
                e.recordRemoval(this);
                return e;
            }
            // 没有找到删除的结点，继续往下找
            prev = e;
            e = next;
        }

        return e;
    }
```

entryset相关方法：

```java

 public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        // 相当于返回了new EntrySet
        return es != null ? es : (entrySet = new EntrySet());
    }

//EntrySet是HashMap的内部类
 private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        // 重写了iterator方法
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
       // 不相关代码
       ...
    }

 Iterator<Map.Entry<K,V>> newEntryIterator()   {
     return new EntryIterator();
 }

private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // 下一个要返回的Entry
        int expectedModCount;   // For fast-fail
        int index;              // 当前table上下标
        Entry<K,V> current;     // 当前的Entry

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }
        // 不相关
        ......
    }
```

null key:

```java
    private V putForNullKey(V value) {
        // 遍历table[0]上的单向链表
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            // 如果有key为null的Entry，则替换该Entry中的value
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        // 如果没有key为null的Entry，则构造一个hash为0、key为null、value为真实值的Entry
        // 插入到table[0]上单向链表的头部
        addEntry(0, null, value, 0);
        return null;
    }

 private V getForNullKey() {
        //  遍历table[0]上的单向链表
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            // 如果key为null，则返回该Entry的value值
            if (e.key == null)
                return e.value;
        }
        return null;
    }
```

