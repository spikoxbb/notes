[TOC]

# LinkedHashMap

LinkedHashMap 是 HashMap 的一个子类，保存了记录的插入顺序，在用 Iterator 遍历LinkedHashMap 时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。

```java
public LinkedHashMap() {
        // 调用HashMap的构造方法，其实就是初始化Entry[] table
        super();
        // 这里是指是否基于访问排序，默认为false
        accessOrder = false;
    }

//LinkedHashMap存储数据是有序的，而且分为两种：插入顺序和访问顺序。
// 第三个参数用于指定accessOrder值
Map<String, String> linkedHashMap = new LinkedHashMap<>(16, 0.75f, true);

@Override
    void init() {
        // 创建了一个hash=-1，key、value、next都为null的Entry
        header = new Entry<>(-1, null, null, null);
        // 让创建的Entry的before和after都指向自身，注意after不是之前提到的next
        // 其实就是创建了一个只有头部节点的双向链表
        header.before = header.after = header;
    }

//LinkedHashMap有自己的静态内部类Entry，它继承了HashMap.Entry
private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
}
```

LinkedHashMap没有重写put方法，所以还是调用HashMap得到put方法，如下：

```csharp
    public V put(K key, V value) {
        // 对key为null的处理
        if (key == null)
            return putForNullKey(value);
        // 计算hash
        int hash = hash(key);
        // 得到在table中的index
        int i = indexFor(hash, table.length);
        // 遍历table[index]，是否key已经存在，存在则替换，并返回旧值
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
        // 如果key之前在table中不存在，则调用addEntry，LinkedHashMap重写了该方法
        addEntry(hash, key, value, i);
        return null;
    }
```

LinkedHashMap的addEntry方法：

```csharp
    void addEntry(int hash, K key, V value, int bucketIndex) {
        // 调用父类的addEntry，增加一个Entry到HashMap中
        super.addEntry(hash, key, value, bucketIndex);

        // removeEldestEntry方法默认返回false，不用考虑
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

```csharp
void addEntry(int hash, K key, V value, int bucketIndex) {
        // 扩容相关
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        // LinkedHashMap进行了重写
        createEntry(hash, key, value, bucketIndex);
    }
```

```csharp
 void createEntry(int hash, K key, V value, int bucketIndex) {
       HashMap.Entry<K,V> old = table[bucketIndex];
       // e就是新创建了Entry，会加入到table[bucketIndex]的表头
       Entry<K,V> e = new Entry<>(hash, key, value, old);
       table[bucketIndex] = e;
       // 把新创建的Entry，加入到双向链表中
       e.addBefore(header);
       size++;
   }
```

LinkedHashMap.Entry的addBefore方法：

```cpp
 private void addBefore(Entry<K,V> existingEntry) {
     after  = existingEntry;
     before = existingEntry.before;
     before.after = this;
     after.before = this;
 }
```

从这里就可以看出，当put元素时，不但要把它加入到HashMap中去，还要加入到双向链表中，所以可以看出LinkedHashMap就是HashMap+双向链表.

![](../../img/4843132-32fb46d33b0ed3c0.jpg)

LinkedHashMap重写了transfer方法，数据的迁移，它的实现如下：

```java
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
        // 扩容后的容量是之前的2倍
        int newCapacity = newTable.length;
        // 遍历双向链表，把所有双向链表中的Entry，重新就算hash，并加入到新的table中
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }
```

从遍历的效率来说，遍历双向链表的效率要高于遍历table，因为遍历双向链表是N次（N为元素个数）；而遍历table是N+table的空余个数（N为元素个数）。

当key如果已经存在时，则进行更新Entry的value。就是HashMap的put方法中的如下代码：

```csharp
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 重排序
                e.recordAccess(this);
                return oldValue;
            }
        }
```

主要看e.recordAccess(this)，这个方法跟访问顺序有关，而HashMap是无序的，所以在HashMap.Entry的recordAccess方法是空实现，但是LinkedHashMap是有序的,LinkedHashMap.Entry对recordAccess方法进行了重写:

```java
 void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            // 如果LinkedHashMap的accessOrder为true，则进行重排序
            // 比如前面提到LruCache中使用到的LinkedHashMap的accessOrder属性就为true
            if (lm.accessOrder) {
                lm.modCount++;
                // 把更新的Entry从双向链表中移除
                remove();
                // 再把更新的Entry加入到双向链表的表尾
                addBefore(lm.header);
            }
        }
```

LinkedHashMap有对get方法进行了重写，如下：

```csharp
    public V get(Object key) {
        // 调用genEntry得到Entry
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        // 如果LinkedHashMap是访问顺序的，则get时，也需要重新排序
        e.recordAccess(this);
        return e.value;
    }
```

LinkedHashMap没有重写entrySet方法,LinkedHashMap中EntryIterator的定义：

```java
    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { 
          return nextEntry();
        }
    }
```

该类是继承LinkedHashIterator，并重写了next方法；而HashMap中是继承HashIterator。LinkedHashIterator的定义：

```java
  private abstract class LinkedHashIterator<T> implements Iterator<T> {
        // 默认下一个返回的Entry为双向链表表头的下一个元素
        Entry<K,V> nextEntry    = header.after;
        Entry<K,V> lastReturned = null;

        public boolean hasNext() {
            return nextEntry != header;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

            Entry<K,V> e = lastReturned = nextEntry;
            nextEntry = e.after;
            return e;
        }
        // 不相关代码
        ......
    }
```

调用next方法去取出Entry，LinkedHashMap中的EntryIterator重写了该方法，如下：

```csharp
 public Map.Entry<K,V> next() { 
    return nextEntry(); 
}
```

LinkedHashMap没有提供remove方法，所以调用的是HashMap的remove方法，实现如下：

```csharp
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }

    final Entry<K,V> removeEntryForKey(Object key) {
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                // LinkedHashMap.Entry重写了该方法
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
```

HashMap.Entry中recordRemoval方法是空实现，但是LinkedHashMap.Entry对其进行了重写，如下：

```csharp
        void recordRemoval(HashMap<K,V> m) {
            remove();
        }

        private void remove() {
            before.after = after;
            after.before = before;
        }
```

易知，这是要把双向链表中的Entry删除，也就是要断开当前要被删除的Entry被其他对象通过after和before的方式引用。