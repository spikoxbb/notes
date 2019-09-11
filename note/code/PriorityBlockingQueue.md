# PriorityBlockingQueue

1. PriorityBlockingQueue 内 部 有 个 数 组 queue 用 来 存 放 队 列 元 素.
2. size 用 来 存 放 队 列 元 素 个 数 ,
3. allocationSpinLockOffset 是用来在扩容队列时候做 cas 的,目的是保证只有一个线程可以进行扩容。
4. 比较器 comparator 用来比较元素大小。
5. lock 独占锁对象用来控制同时只能有一个线程可以进行入队出队操作。
6. notEmpty条件变量用来实现 take 方法阻塞模式。没有 notFull 条件变量是因为 put 操作是非阻塞的,因为是无界队列。
7.  PriorityQueue q 用来搞序列化的。

构造函数,默认队列容量为 11,默认比较器为 null;

```java
private static final int DEFAULT_INITIAL_CAPACITY = 11;
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityBlockingQueue(int initialCapacity,Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
	this.notEmpty = lock.newCondition();
	this.comparator = comparator;
	this.queue = new Object[initialCapacity];
}
```

Offer 操作
在队列插入一个元素,由于是无界队列,所以一直为成功返回 true;

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
	final ReentrantLock lock = this.lock;
	lock.lock();
	int n, cap;
	Object[] array;
	//如果当前元素个数>=队列容量,则扩容(1)
	while ((n = size) >= (cap = (array = queue).length))
		tryGrow(array, cap);
	try {
			Comparator<? super E> cmp = comparator;
        //默认比较器为 null
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp)；
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}

private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); //must release and then re-acquire main lock
    Object[] newArray = null;
    //cas 成功则扩容(4)
    if (allocationSpinLock == 0 &&UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,0, 1)) {
        try {
            //oldGap<64 则扩容新增 oldcap+2,否者扩容 50%,并且最大为 MAX_ARRAY_SIZE
            int newCap = oldCap + ((oldCap < 64) ?(oldCap + 2) :(oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {
                // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }
    //第一个线程 cas 成功后,第二个线程会进入这个地方,然后第二个线程让出 cpu,尽量让第一个线程执行下面点获锁,但是这得不到肯定的保证。
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

因为扩容时候是需要花时间的,如果这些操作时候还占用锁那么其他线程在这个时候是不能进行出队操作的,也不能进行入队操作,这大大降低了并发性。

具体建堆算法:

```java
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    //队列元素个数>0 则判断插入位置,否者直接入队(7)
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}
```

Poll 操作，在队列头部获取并移除一个元素,如果队列为空,则返回 null

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    //队列为空,则返回 null
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)//cmp=null 则调用这个,把对尾元素位置插入到 0 位置,并且调整堆为最小堆(3)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
private static <T> void siftDownComparable(int k, T x, Object[] array,int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;
        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = array[child];
            int right = child + 1;
            if (right < n &&((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)(8)
                break;
            array[k] = c;
            k = child;
        }
        array[k] = key;(9)
    }
}
```

Put 操作，内部调用的 offer,由于是无界队列,所以不需要阻塞

```java
public void put(E e) {
    offer(e); // never need to block
}
```

Take 操作，获取队列头元素,如果队列为空则阻塞。

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        //如果队列为空,则阻塞,把当前线程放入 notEmpty 的条件队列
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
}
```

Size 操作,获取队列元个数,由于加了独占锁所以返回结果是精确的

```java
public int size() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return size;
    } finally {
        lock.unlock();
    }
}
```

