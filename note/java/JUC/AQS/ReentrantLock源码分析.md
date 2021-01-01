[TOC]

| Lock 接口                         | ReentrantLock 实现                             |
| --------------------------------- | ---------------------------------------------- |
| lock()                            | sync.lock()                                    |
| lockInterruptibly()               | sync.acquireInterruptibly(1)                   |
| tryLock()                         | sync.nonfairTryAcquire(1)                      |
| tryLock(long time, TimeUnit unit) | sync.tryAcquireNanos(1, unit.toNanos(timeout)) |
| unlock()                          | sync.release(1)                                |
| newCondition()                    | sync.newCondition()                            |

ReentrantLock对于Lock接口的实现都是直接“转交”给sync对象的。

# 核心属性

ReentrantLock只有一个sync属性，别看只有一个属性，这个属性提供了所有的实现，我们上面介绍ReentrantLock对Lock接口的实现的时候就说到，它对所有的Lock方法的实现都调用了sync的方法，这个sync就是ReentrantLock的属性，它继承了AQS。

```
private final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();
    //...
}
```

在Sync类中，定义了一个抽象方法lock，该方法应当由继承它的子类来实现，关于继承它的子类。

# 构造函数

ReentrantLock共有两个构造函数：

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认的构造函数使用了非公平锁，另外一个构造函数通过传入一个boolean类型的`fair`变量来决定使用公平锁还是非公平锁。其中，FairSync和NonfairSync的定义如下：

```java
static final class FairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}

static final class NonfairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}
```

这里为什么默认创建的是非公平锁呢？

因为非公平锁的效率高呀，当一个线程请求非公平锁时，如果在**发出请求的同时**该锁变成可用状态，那么这个线程会跳过队列中所有的等待线程而获得锁。
之所以使用这种方式是因为：

> 在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟。

# Lock接口方法实现

## lock()

### 公平锁实现

关于ReentrantLock对于lock方法的公平锁的实现逻辑AQS中已经讲过了。

### 非公平锁实现

```java
// NonfairSync中的lock方法
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

对比公平锁中的lock方法：

```java
// FairSync中的lock方法
final void lock() {
    acquire(1);
}
```

可见，相比公平锁，非公平锁在当前锁没有被占用时，可以直接尝试去获取锁，而不用排队，所以它在一开始就尝试使用CAS操作去抢锁，只有在该操作失败后，才会调用AQS的acquire方法。

由于acquire方法中除了tryAcquire由子类实现外，其余都由AQS实现，仅仅看一下非公平锁的tryAcquire方法实现：

```java
// NonfairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

它调用了Sync类的nonfairTryAcquire方法：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 只有这一处和公平锁的实现不同，其它的完全一样。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这两个方法几乎一模一样，唯一的区别就是**非公平锁在抢锁时不再需要调用`hasQueuedPredecessors`方法先去判断是否有线程排在自己前面，而是直接争锁，其它的完全和公平锁一致。**

## tryLock()

由于tryLock仅仅是用于检查锁在当前调用的时候是不是可获得的，所以即使现在使用的是非公平锁，在调用这个方法时，当前线程也会直接尝试去获取锁，哪怕这个时候队列中还有在等待中的线程。

所以这一方法对于公平锁和非公平锁的实现是一样的，它被定义在Sync类中，由FairSync和NonfairSync直接继承使用：

```java
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
```

## tryLock(long timeout, TimeUnit unit)

与立即返回的`tryLock()`不同，`tryLock(long timeout, TimeUnit unit)`带了超时时间，所以是阻塞式的，并且在获取锁的过程中可以响应中断异常：

```java
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
}
```

与`lockInterruptibly`方法一样，该方法首先检查当前线程是否已经被中断过了，如果已经被中断了，则立即抛出`InterruptedException`。

随后我们通过调用`tryAcquire`和`doAcquireNanos(arg, nanosTimeout)`方法来尝试获取锁，注意，这时公平锁和非公平锁对于`tryAcquire`方法就有不同的实现了：

- 公平锁首先会检查当前有没有别的线程在队列中排队，关于公平锁和非公平锁对`tryAcquire`的不同实现上文已经讲过了。

我们直接来看`doAcquireNanos`，这个方法其实和前面说的`doAcquireInterruptibly`方法很像，我们通过将相同的部分注释掉，直接看不同的部分：

```java
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    /*final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;*/
                return true; // doAcquireInterruptibly中为 return
            /*}*/
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
       /* }
    } finally {
        if (failed)
            cancelAcquire(node);
    }*/
}
```

可以看出，这两个方法的逻辑大差不差，只是`doAcquireNanos`多了对于截止时间的检查。

不过这里有两点需要注意，一个是`doAcquireInterruptibly`是没有返回值的，而`doAcquireNanos`是有返回值的。这是因为`doAcquireNanos`有可能因为获取到锁而返回，也有可能因为超时时间到了而返回，为了区分这两种情况，因为超时时间而返回时，我们将返回false，代表并没有获取到锁。

另外一点值得注意的是，上面有一个`nanosTimeout > spinForTimeoutThreshold`的条件，在它满足的时候才会将当前线程挂起指定的时间，这个spinForTimeoutThreshold是个啥呢：

```java
/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices
 * to improve responsiveness with very short timeouts.
 */
static final long spinForTimeoutThreshold = 1000L;
```

它就是个阈值，是为了提升性能用的。如果当前剩下的等待时间已经很短了，我们就直接使用自旋的形式等待，而不是将线程挂起，可见作者为了尽可能地优化AQS锁的性能费足了心思。

## unlock()

unlock操作用于释放当前线程所占用的锁，这一点对于公平锁和非公平锁的实现是一样的，所以该方法被定义在Sync类中，由FairSync和NonfairSync直接继承使用：

```
public void unlock() {
    sync.release(1);
}
```

