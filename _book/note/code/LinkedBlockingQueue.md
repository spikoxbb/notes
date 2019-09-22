# LinkedBlockingQueue

1. 有两个 Node 分别用来存放首尾节点
2. 有个初始值为 0 的原子变量 count用来记录队列元素个数
3. 有两个 ReentrantLock 的独占锁分别用来控制元素入队和出队加锁,其中 takeLock用来控制同时只有一个线程可以从队列获取元素,其他线程必须等待,putLock 控制同时只能有一个线程可以获取锁去添加元素,其他线程必须等待。
4. notEmpty 和 notFull 用来实现入队和出队的同步

```java
/** 通过 take 取出进行加锁、取出 */
private final ReentrantLock takeLock = new ReentrantLock();
/** 等待中的队列等待取出 */
private final Condition notEmpty = takeLock.newCondition();
/*通过 put 放置进行加锁、放置*/
private final ReentrantLock putLock = new ReentrantLock();
/** 等待中的队列等待放置 */
private final Condition notFull = putLock.newCondition();
/* 记录集合中的个数(计数器) */
private final AtomicInteger count = new AtomicInteger(0);

//队列初始容量,Integer 最大值
public static final int MAX_VALUE = 0x7fffffff;
public LinkedBlockingQueue() {
	this(Integer.MAX_VALUE);
}
public LinkedBlockingQueue(int capacity) {
	if (capacity <= 0) throw new IllegalArgumentException();
	this.capacity = capacity;
	//初始化首尾节点
	last = head = new Node<E>(null);
}
```

带时间的 Offer 方法

```java
public boolean offer(E e, long timeout, TimeUnit unit)throws InterruptedException {
    //空元素抛空指针异常
	if (e == null) throw new NullPointerException();
	long nanos = unit.toNanos(timeout);
	int c = -1;
	final ReentrantLock putLock = this.putLock;
	final AtomicInteger count = this.count;
	//获取可被中断锁,只有一个线程克获取
	putLock.lockInterruptibly();
    try {
		//如果队列满则进入循环
		while (count.get() == capacity) {
            //nanos<=0 直接返回
            if (nanos <= 0)
                return false;
            //否者调用 await 进行等待,超时则返回<=0(1)
            nanos = notFull.awaitNanos(nanos);
        }
        //await 在超时时间内返回则添加元素(2)
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        //队列不满则激活其他等待入队线程(3)
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        //释放锁
        putLock.unlock();
    }
    //c==0 说明队列里面有一个元素,这时候唤醒出队线程(4)
    if (c == 0)
        signalNotEmpty();
    return true;
}
private void enqueue(Node<E> node) {
    last = last.next = node;
}
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

带时间的 poll 操作:获取并移除队首元素,在指定的时间内去轮询队列看有没有首元素有则返回,否者超时后返回 null。

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    //出队线程获取独占锁
    takeLock.lockInterruptibly();
    try {
        //循环直到队列不为空
        while (count.get() == 0) {
            //超时直接返回 null
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        //出队,计数器减一
        x = dequeue();
        c = count.getAndDecrement();
        //如果出队前队列不为空则发送信号,激活其他阻塞的出队线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        //释放锁
        takeLock.unlock();
    }
    //当前队列容量为最大值-1 则激活入队线程。
    if (c == capacity)
        signalNotFull();
    return x;
}
```

size 操作，当前队列元素个数,直接使用原子变量 count 获取。

```java
public int size() {
	return count.get();
}
```

peek 操作，获取但是不移除当前队列的头元素,没有则返回 null。

```java
public E peek() {
    //队列空,则返回 null
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```

remove 操作,删除队列里面的一个元素,有则删除返回 true,没有则返回 false,在删除操作时候由于要遍历队列所以加了双重
锁,也就是在删除过程中不允许入队也不允许出队操作。

```java
public boolean remove(Object o) {
    if (o == null) return false;
    //双重加锁
    fullyLock();
    try {
        //遍历队列找则删除返回 true
        for (Node<E> trail = head, p = trail.next;p != null;trail = p, p = p.next) {
            if (o.equals(p.item)) {
                unlink(p, trail);
                return true;
            }
        }
        //找不到返回 false
        return false;
    } finally {
        //解锁
        fullyUnlock();
    }
}
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}
void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
void unlink(Node<E> p, Node<E> trail) {
    p.item = null;
    trail.next = p.next;
    if (last == p)
        //如果当前队列满,删除后,也不忘记最快的唤醒等待的线程
        if (count.getAndDecrement() == capacity)
            notFull.signal();
}
```


