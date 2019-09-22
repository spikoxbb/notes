# DelayQueue

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