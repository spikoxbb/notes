[TOC]

# ArrayBlockingQueue

用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。

默认情况下不保证访问者公平的访问队列，所谓公平访问队列是指阻塞的所有生产者线程或消费者线程，当队列可用时，可以按照阻塞的先后顺序访问队列，即先阻塞的生产者线程，可以先往队列里插入元素，先阻塞的消费者线程，可以先从队列里获取元素。通常情况下为了保证公平性会降低吞吐量。

1. ArrayBlockingQueue 内部有个数组 items 用来存放队列元素
2. putindex 下标标示入队元素下标
3. takeIndex 是出队下标
4. count 统计队列元素个数
5. 独占锁 lock 用来对出入队操作加锁,这导致同时只有一个线程可以访问入队出队
6. notEmpty,notFull 条件变量用来进行出入队的同步。

构造函数必须传入队列大小参数,所以为有界队列,默认是 Lock 为非公平锁。

```java
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
	lock = new ReentrantLock(fair);
	notEmpty = lock.newCondition();
	notFull =lock.newCondition();
}
```

offer 方法,在队尾插入元素,如果队列满则返回 false,否者入队返回 true。加过锁后获取的共享变量都是从主内存获取的,
而不是在 CPU 缓存或者寄存器里面的值,释放锁后修改的共享变量值会刷新会主内存中。

```java
public boolean offer(E e) {
    //e 为 null,则抛出 NullPointerException 异常
    checkNotNull(e);
	//获取独占锁
	final ReentrantLock lock = this.lock;
	lock.lock();
    try {
        //如果队列满则返回 false
		if (count == items.length)
			return false;
		else {
			//否者插入元素
			insert(e);
			return true;
        }
    }finally{
        //释放锁
        lock.unlock();
    }
}

private void insert(E x) {
    //元素入队
    items[putIndex] = x;
	//计算下一个元素应该存放的下标
	putIndex = inc(putIndex);
	++count;
	notEmpty.signal();
}

final int inc(int i) {
	return (++i == items.length) ? 0 : i;
}
```

Put 操作,在队列尾部添加元素,如果队列满则等待队列有空位置插入后返回。
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //获取可被中断锁,await()方法会在中断标志设置后抛出 InterruptedException 异常后退出,所以还不如在加锁时候先看中断标志是不是被设置了,如果设置了直接抛出 InterruptedException 异常,就不用再去获取锁了。
    lock.lockInterruptibly();
    try {
        //如果队列满,则把当前线程放入 notFull 管理的条件队列
        while (count == items.length)
            notFull.await();
        //插入元素
        insert(e);
    } finally {
		lock.unlock();
	}
}
```

Poll 操作,从队头获取并移除元素,队列为空,则返回 null。

```java
public E poll() {
    final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		//当前队列为空则返回 null,否者
		return (count == 0) ? null : extract();
	} finally {
		lock.unlock();
	}
}

private E extract() {
    final Object[] items = this.items;
	//获取元素值
	E x = this.<E>cast(items[takeIndex]);
	//数组中值值为 null;
	items[takeIndex] = null;
	//队头指针计算,队列元素个数减一
	takeIndex = inc(takeIndex);
	--count;
	//发送信号激活 notFull 条件队列里面的线程
	notFull.signal();
	return x;
}
```

Take 操作,从队头获取元素,如果队列为空则阻塞直到队列有元素。

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
	lock.lockInterruptibly();
	try {
		//队列为空,则等待,直到队列有元素
		while (count == 0)
            notEmpty.await();
		return extract();
	} finally {
		lock.unlock();
	}
}
```

Peek 操作,返回队列头元素但不移除该元素,队列为空,返回 null。

```java
public E peek() {
    final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		//队列为空返回 null,否者返回头元素
		return (count == 0) ? null : itemAt(takeIndex);
	} finally {
		lock.unlock();
	}
}
final E itemAt(int i) {
	return this.<E>cast(items[i]);
}
```

Size 操作,获取队列元素个数,非常精确因为计算 size 时候加了独占锁,其他线程不能入队或者出队或者删除元素。

```java
public int size() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		return count;
	} finally {
		lock.unlock();
	}
}
```


