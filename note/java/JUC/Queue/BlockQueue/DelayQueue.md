[TOC]

# DelayQueue

是一个支持延时获取元素的无界阻塞队列。队列使用 PriorityQueue 来实现。队列中的元素必须实现 Delayed 接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将 DelayQueue 运用在以下应用场景：

1. 缓存系统的设计：可以用 DelayQueue 保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从 DelayQueue 中获取元素时，表示缓存有效期到了。
2. 定时任务调度：使用 DelayQueue 保存当天将会执行的任务和执行时间，一旦从DelayQueue 中获取到任务就开始执行，从比如 TimerQueue 就是使用 DelayQueue 实现的。

Delayed 扩展了 Comparable 接口,比较的基准为延时的时间值,Delayed 接口的实现类 getDelay 的返回值应为固定值(final)。DelayQueue 内部是使用 PriorityQueue 实现的。
**DelayQueue = BlockingQueue +PriorityQueue + Delayed**

```java
public class DelayQueue<E extends Delayed> implements BlockingQueue<E> {
    private final PriorityQueue<E> q = new PriorityQueue<E>();
}
```

DelayQueue 内部的实现使用了一个优先队列。当调用 DelayQueue 的 offer 方法时,把 Delayed 对象加入到优
先队列 q 中。如下:

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        q.offer(e);
        if (first == null || e.compareTo(first) < 0)
            available.signalAll();
        return true;
    } finally {
        lock.unlock();
    }
}
```

DelayQueue 的 take 方法,把优先队列 q 的 first 拿出来(peek),如果没有达到延时阀值,则进行 await
处理。如下:

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) {
                available.await();
            } else {
                long delay = first.getDelay(TimeUnit.NANOSECONDS);
                if (delay > 0) {
                    long tl = available.awaitNanos(delay);
                } else {
                    E x = q.poll();
                    assert x != null;
                    if (q.size() != 0)
                        available.signalAll(); // wake up other takers
                    return x;
                }
            }
        }
    } finally {
        lock.unlock();
    }
}
```