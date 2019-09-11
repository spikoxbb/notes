# ThreadLocal

ThreadLoca是threadlocalvariable(线程局部变量)。

ThreadLocal起作用的根源是Thread类：

```java
public class Thread implements Runnable {
   ......
    ThreadLocal.ThreadLocalMap threadLocals = null;
    /*
     * InheritableThreadLocal，自父线程集成而来的ThreadLocalMap，
     * 主要用于父子线程间ThreadLocal变量的传递
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    ......
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

  private T setInitialValue() {
      //initialValue() 是 ThreadLocal 的初始值，默认返回 null，子类可以重写改方法，用于设置 ThreadLocal 的初始值。
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }

public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

### ThreadLocalMap:

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```



ThreadLocalMap 内部维护了一个哈希表（数组）来存储数据，并且定义了加载因子：

```java
// 初始容量，必须是 2 的幂
private static final int INITIAL_CAPACITY = 16;

// 存储数据的哈希表
private Entry[] table;

// table 中已存储的条目数
private int size = 0;

// 表示一个阈值，当 table 中存储的对象达到该值时就会扩容
private int threshold;

// 设置 threshold 的值
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

Entry继承自WeakReference:

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

被WeakReference引用的对象是ThreadLocal自身.

调用 set(ThreadLocal key, Object value) 方法将数据保存到哈希表中：

```java
private void set(ThreadLocal key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 计算要存储的索引位置
    int i = key.threadLocalHashCode & (len-1);

    // 循环判断要存放的索引位置是否已经存在 Entry，若存在，进入循环体
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal k = e.get();

        // 若索引位置的 Entry 的 key 和要保存的 key 相等，则更新该 Entry 的值
        if (k == key) {
            e.value = value;
            return;
        }

        // 若索引位置的 Entry 的 key 为 null（key 已经被回收了），表示该位置的 Entry 已经无效，用要保存的键值替换该位置上的 Entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 要存放的索引位置没有 Entry，将当前键值作为一个 Entry 保存在该位置
    tab[i] = new Entry(key, value);
    // 增加 table 存储的条目数
    int sz = ++size;
    // 清除一些无效的条目并判断 table 中的条目数是否已经超出阈值
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash(); // 调整 table 的容量，并重新摆放 table 中的 Entry
}
```

首先使用 key（当前 ThreadLocal）的 threadLocalHashCode 来计算要存储的索引位置 i。threadLocalHashCode 的值由 ThreadLocal 类管理，每创建一个 ThreadLocal 对象都会自动生成一个相应的 threadLocalHashCode 值，其实现如下：

```java
// ThreadLocal 对象的 HashCode
private final int threadLocalHashCode = nextHashCode();

// 使用 AtomicInteger 保证多线程环境下的同步
private static AtomicInteger nextHashCode =
    new AtomicInteger();

// 每次创建 ThreadLocal 对象是 HashCode 的增量
private static final int HASH_INCREMENT = 0x61c88647;

// 计算 ThreadLocal 对象的 HashCode
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

在调整 table 容量时，也会先清除无效对象，然后再根据需要扩容。

```java
private void rehash() {
    // 先清除无效 Entry
    expungeStaleEntries();
    // 判断当前 table 中的条目数是否超出了阈值的 3/4
    if (size >= threshold - threshold / 4)
        resize();
}
```

## 获取 Entry 对象

取值是直接获取到 Entry 对象，使用 getEntry(ThreadLocal key) 方法：

```java
private Entry getEntry(ThreadLocal key) {
    // 使用指定的 key 的 HashCode 计算索引位置
    int i = key.threadLocalHashCode & (table.length - 1);
    // 获取当前位置的 Entry
    Entry e = table[i];
    // 如果 Entry 不为 null 且 Entry 的 key 和 指定的 key 相等，则返回该 Entry
    // 否则调用 getEntryAfterMiss(ThreadLocal key, int i, Entry e) 方法
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

因为可能存在哈希冲突,所以需要调用 getEntryAfterMiss(ThreadLocal key, int i, Entry e) 方法获取。

```java
private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    // 索引位置上的 Entry 不为 null 进入循环，为 null 则返回 null
    while (e != null) {
        ThreadLocal k = e.get();
        // 如果 Entry 的 key 和指定的 key 相等，则返回该 Entry
        if (k == key)
            return e;
        // 如果 Entry 的 key 为 null （key 已经被回收了），清除无效的 Entry
        // 否则获取下一个位置的 Entry，循环判断
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

```java
private void remove(ThreadLocal key) {
    Entry[] tab = table;
    int len = tab.length;
    // 使用指定的 key 的 HashCode 计算索引位置
    int i = key.threadLocalHashCode & (len-1);
    // 循环判断索引位置的 Entry 是否为 null
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 若 Entry 的 key 和指定的 key 相等，执行删除操作
        if (e.get() == key) {
            // 清除 Entry 的 key 的引用
            e.clear();
            // 清除无效的 Entry
            expungeStaleEntry(i);
            return;
        }
    }
}
```

在 ThreadLocalMap 的 set()，get() 和 remove() 方法中，都有清除无效 Entry 的操作，这样做是为了降低内存泄漏发生的可能。

Entry 中的 key 使用了弱引用的方式，这样做是为了降低内存泄漏发生的概率，但不能完全避免内存泄漏。

假设 Entry 的 key 没有使用弱引用的方式，而是使用了强引用：由于 ThreadLocalMap 的生命周期和当前线程一样长，那么当引用 ThreadLocal 的对象被回收后，由于 ThreadLocalMap 还持有 ThreadLocal 和对应 value 的强引用，ThreadLocal 和对应的 value 是不会被回收的，这就导致了内存泄漏。所以 Entry 以弱引用的方式避免了 ThreadLocal 没有被回收而导致的内存泄漏，但是此时 value 仍然是无法回收的，依然会导致内存泄漏。

ThreadLocalMap 已经考虑到这种情况，并且有一些防护措施：在调用 ThreadLocal 的 get()，set() 和 remove() 的时候都会清除当前线程 ThreadLocalMap 中所有 key 为 null 的 value。这样可以降低内存泄漏发生的概率。所以我们在使用 ThreadLocal 的时候，每次用完 ThreadLocal 都调用 remove() 方法，清除数据，防止内存泄漏。

Spring的事务实现也借助了ThreadLocal类。Spring会从数据库连接池中获得一个connection，然会把connection放进ThreadLocal中，也就和线程绑定了，事务需要提交或者回滚，只要从ThreadLocal中拿到connection进行操作。