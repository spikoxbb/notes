[TOC]

# AbstractQueuedSynchronizer概述

AbstractQueuedSynchronizer是java中非常重要的一个框架类，它实现了最核心的多线程同步的语义，我们只要继承AbstractQueuedSynchronizer就可以非常方便的实现我们自己的线程同步器，java中的锁Lock就是基于AbstractQueuedSynchronizer来实现的。

在类结构上，AbstractQueuedSynchronizer继承了AbstractOwnableSynchronizer，AbstractOwnableSynchronizer仅有的两个方法是提供当前独占模式的线程设置：

```java

    private transient Thread exclusiveOwnerThread;

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
```

exclusiveOwnerThread代表的是当前获得同步的线程，因为是独占模式，在exclusiveOwnerThread持有同步的过程中其他的线程的任何同步获取请求将不能得到满足。

AbstractQueuedSynchronizer不仅支持独占模式下的同步实现，还支持共享模式下的同步实现。

在java的锁的实现上就有共享锁和独占锁的区别，而这些实现都是基于AbstractQueuedSynchronizer对于共享同步和独占同步的支持。从上面展示的AbstractQueuedSynchronizer提供的方法中，我们可以发现AbstractQueuedSynchronizer的API大概分为三类：

- 类似acquire(int)的一类是最基本的一类，不可中断。
- 类似acquireInterruptibly(int)的一类可以被中断。
- 类似tryAcquireNanos(int, long)的一类不仅可以被中断，而且可以设置阻塞时间。

**上面的三种类型的API分为独占和共享两套，我们可以根据我们的需求来使用合适的API来做多线程同步。**

下面是一个继承AbstractQueuedSynchronizer来实现自己的同步器的一个示例：

```java
 *class Mutex implements Lock, java.io.Serializable {
 *
 *   // Our internal helper class
 *   private static class Sync extends AbstractQueuedSynchronizer {
 *     // Reports whether in locked state
 *     protected boolean isHeldExclusively() {
 *       return getState() == 1;
 *     }
 *
 *     // Acquires the lock if state is zero
 *     public boolean tryAcquire(int acquires) {
 *       assert acquires == 1; // Otherwise unused
 *       if (compareAndSetState(0, 1)) {
 *         setExclusiveOwnerThread(Thread.currentThread());
 *         return true;
 *       }
 *       return false;
 *     }
 *
 *     // Releases the lock by setting state to zero
 *     protected boolean tryRelease(int releases) {
 *       assert releases == 1; // Otherwise unused
 *       if (getState() == 0) throw new IllegalMonitorStateException();
 *       setExclusiveOwnerThread(null);
 *       setState(0);
 *       return true;
 *     }
 *
 *     // Provides a Condition
 *     Condition newCondition() { return new ConditionObject(); }
 *
 *     // Deserializes properly
 *     private void readObject(ObjectInputStream s)
 *         throws IOException, ClassNotFoundException {
 *       s.defaultReadObject();
 *       setState(0); // reset to unlocked state
 *     }
 *   }
 *
 *   // The sync object does all the hard work. We just forward to it.
 *   private final Sync sync = new Sync();
 *
 *   public void lock()                { sync.acquire(1); }
 *   public boolean tryLock()          { return sync.tryAcquire(1); }
 *   public void unlock()              { sync.release(1); }
 *   public Condition newCondition()   { return sync.newCondition(); }
 *   public boolean isLocked()         { return sync.isHeldExclusively(); }
 *   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
 *   public void lockInterruptibly() throws InterruptedException {
 *     sync.acquireInterruptibly(1);
 *   }
 *   public boolean tryLock(long timeout, TimeUnit unit)
 *       throws InterruptedException {
 *     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
 *   }
 * }}
```

# AbstractQueuedSynchronizer实现细节

AbstractQueuedSynchronizer使用一个volatile类型的int来作为同步变量，任何想要获得锁的线程都需要来竞争该变量，获得锁的线程可以继续业务流程的执行，**而没有获得锁的线程会被放到一个FIFO的队列中去，等待再次竞争同步变量来获得锁。**AbstractQueuedSynchronizer为每个没有获得锁的线程封装成一个Node再放到队列中去。

## Node

下面先来分析一下Node这个数据结构：

每个节点都有其对应的状态，初始状态为0。

```java
//等待超时或被中断，取消获取锁
static final int CANCELLED =  1;
//说明该节点的后续被挂起了，当释放锁或取消时，需要唤醒后继节点 
static final int SIGNAL    = -1;
//表示节点处于Condition队列中 
static final int CONDITION = -2;
//用于共享式锁，表示下一次尝试获取共享锁时，需要无条件传播下去
static final int PROPAGATE = -3;
```

## 独占锁

### 不响应中断的独占锁获取

### acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

tryAcuquire() 方法为获取锁的抽象方法，返回 true 表示获取锁成功，需要实现类根据获取锁的方式自己定义。

1. 如果 tryAcquire() 获取锁失败，则通过 addWaiter() 加入到同步队列中，再通过 acquireQueued() 不断尝试获取锁。
2. 由于不响应中断，如果检测到中断，acquireQueued() 会返回 true，进入方法体selfInterrupt。
3.  **selfInterrupt（）由于检测时使用了 Thread.interrupted()，中断标志被重置，需要恢复中断标志**

#### **addWaiter**

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试通过一次 CAS 将节点加入到队尾
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 走到这里说明要么有竞争 CAS 失败，要么同步器队列还没初始化即 pred == null
    enq(node);
    return node;
}

```

1. 尝试通过一次 CAS 将节点加入到队尾。
2. 竞争 CAS 失败或者同步器队列还没初始化进入 enq(node)。

##### **enq**

```java
private Node enq(final Node node) {
    // 无限循环 CAS 直到将节点加入到队尾中
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

```

无限循环 CAS 直到将节点加入到队尾中。

#### **acquireQueued**

```java
/**
* 因获取锁失败而加入同步队列中的线程在这里不断尝试获取锁
* 返回中断状态交由上层函数处理
*/
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 由于这是一个先进先出的队列，只有当自己的前驱是头结点（头结点表示已经获取到锁）时，才能轮到自己来争夺锁
            if (p == head && tryAcquire(arg)) {
				        // tryAcquire() 返回 true 说明获取锁成功
                // 将 node节点设置为 head，此外 setHead() 是不需要 CAS 的，因为不会有竞争
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取失败后，查看是否需要被挂起，如果需要挂起，检查是否有中断信息
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

1. 由于这是一个先进先出的队列，只有当自己的前驱是头结点（头结点表示已经获取到锁）时，才能轮到自己来争夺锁.
2. 获取锁成功将 node节点设置为 head，此外 setHead() 是不需要 CAS 的，因为不会有竞争。
3. 获取失败后，查看是否需要被挂起，如果需要挂起，检查是否有中断信息。

#### **shouldParkAfterFailedAcquire**

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
	// 复习一下，SIGNAL 说明该节点的后续被挂起了，当释放锁或取消时，需要唤醒后继节点
	// 如果前驱节点已经是 SIGNAL 状态了 说明当前线程可以安心被挂起了，等待前驱来唤醒自己
    if (ws == Node.SIGNAL)
        return true;
    // ws > 0 说明前驱节点被取消了(CANCELLED == 1)，需要跳过被取消的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 将前驱节点通过CAS改为 SIGNAL 状态，但最后还是会返回 false 
		// 如果在下一次循环中如果还是没拿到锁，则会进入该方法第一个判断，返回true，
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // 挂起线程
    LockSupport.park(this);
    return Thread.interrupted();
}
```

1. 前驱节点已经是 SIGNAL 状态了 说明当前线程可以安心被挂起了，等待前驱来唤醒自己。
2. ws > 0 说明前驱节点被取消了(CANCELLED == 1)，需要跳过被取消的节点。
3. 将前驱节点通过CAS改为 SIGNAL 状态，但最后还是会返回 false 
   - 如果在下一次循环中如果还是没拿到锁，则会进入该方法第一个判断，返回true。

## 响应中断的独占锁获取

```java

public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

/**
* 和不响应中断的获取方法唯一不同的是,在检测到中断后是抛出中断异常而不是返回true,其他没有区别
*/
private void doAcquireInterruptibly(int arg)
    //...
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                throw new InterruptedException();
    // ...
}

```

## 带超时的响应中断的独占锁获取

```java
/**
* 方法入口
*/
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
}

/**
 * 基本上和之前的差不多，如果超时了就直接返回 false，挂起线程时也使用了带计时的 parkNanos
 */
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 如果超时了 返回false
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            // 注意这里 nanosTimeout > spinForTimeoutThreshold（默认1000纳秒）时才挂起，小于这个阈值时直接自旋，不再挂起
            if (shouldParkAfterFailedAcquire(p, node) && nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**nanosTimeout > spinForTimeoutThreshold（默认1000纳秒）时才挂起，小于这个阈值时直接自旋，不再挂起。**

## 独占锁释放

```java
/**
 * 和加锁一样，这里的 tryRelease() 也是抽象方法，需要子类自己实现
 * 实际工作就是唤醒后继节点而已，出队的操作也是在获取锁的时候由后继结点完成的
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        // 如果 h.waitStatus == 0 ，说明不是 SIGNAL 状态，没有需要唤醒的节点，直接返回
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
        
    // 如果后继节点已经取消了，那么重新调整后继直到没有取消的为止
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果有未取消的后继，唤醒他
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

1. 如果 h.waitStatus == 0 ，说明不是 SIGNAL 状态，没有需要唤醒的节点，直接返回，否则唤醒后继节点。
2. 如果后继节点已经取消了，从尾节点开始往前寻找直到没有取消的为止。

## 共享锁

### 不响应中断的共享锁获取

 在实现上，共享锁和独占锁在实现上的核心区别在于：队列中的线程节点尝试获取锁资源，如果成功则唤醒后面还在等待的共享节点并把该唤醒事件传递下去，即会依次唤醒在该节点后面的所有共享节点。

- 因为独占锁只能被一个线程持有，如果它还没有被释放，就没有必要去唤醒它的后继节点。
- 在共享锁模式下，当一个节点获取到了共享锁，我们在获取成功后就可以唤醒后继节点了，而不需要等到该节点释放锁的时候，这是因为共享锁可以被多个线程同时持有，一个锁获取到了，则后继的节点都可以直接来获取。因此，**在共享锁模式下，在获取锁和释放锁结束时，都会唤醒后继节点。** 这一点也是`doReleaseShared()`方法与`unparkSuccessor(h)`方法无法直接对应的根本原因所在。

```java
/**
* 方法入口，tryAcquireShared为抽象方法：
*/
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

```

* 返回小于0表示获取失败
* 等于0表示当前线程获取到锁，但后续线程获取不到，即不需要传播后续节点
* 大于0表示后续线程也能获取到，需要传播后续节点

### doAcquireShared

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                // >=0表示获取锁成功
                if (r >= 0) {
                    // 这里和独占模型不同，除了设置头结点后还需要向后传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

1. r >= 0表示获取锁成功,这里和独占模型不同，除了设置头结点后还需要向后传播。

#### setHeadAndPropagate

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    
    // 如果propagate > 0 或者 h.waitStatus < 0（PROPAGATE） 需要唤醒后继节点
    if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 如果后继结点是独占结点，就不唤醒了
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // 队列里至少有2个节点，否则没有传播必要
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                // 和共享锁不同的是，这个方法可以在setHeadAndPropagate和releaseShared两个方法中被调用
                // 存在一个线程正获取完锁向后传播，另一个线程释放锁的情况，所以需要CAS控制
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // ws == 0 表明是队列的最后一个节点，那么CAS为PROPAGATE，表明下一次tryShared时，需要传播
            // 如果失败说明有新后继节点将其改为了SIGNAL后挂起了，那么继续循环传播
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 如果head改变了，说明有新的排队的线程获取到了锁，再次检查
        if (h == head)                   // loop if head changed
            break;
    }
}
```

### 共享锁释放

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // 释放成功后，往后传播
        doReleaseShared();
        return true;
    }
    return false;
}

```

在独占锁模式下，由于头节点就是持有独占锁的节点，在它释放独占锁后，如果发现自己的waitStatus不为0，则它将负责唤醒它的后继节点。

在共享锁模式下，头节点就是持有共享锁的节点，在它释放共享锁后，它也应该唤醒它的后继节点，但是值得注意的是，我们在之前的`setHeadAndPropagate`方法中可能已经调用过该方法了，也就是说**它可能会被同一个头节点调用两次**，也有可能在我们从`releaseShared`方法中调用它时，当前的头节点已经易主了。

**doReleaseShared方法有几处调用？**

该方法有两处调用，一处在`acquireShared`方法的末尾，当线程成功获取到共享锁后，在一定条件下调用该方法；一处在`releaseShared`方法中，当线程释放共享锁的时候调用。

**调用该方法的线程是谁？**

在独占锁中，只有获取了锁的线程才能调用release释放锁，因此调用unparkSuccessor(h)唤醒后继节点的必然是持有锁的线程，该线程可看做是当前的头节点(虽然在setHead方法中已经将头节点的thread属性设为了null，但是这个头节点曾经代表的就是这个线程)

在共享锁中，持有共享锁的线程可以有多个，这些线程都可以调用`releaseShared`方法释放锁；而这些线程想要获得共享锁，则它们必然**曾经成为过头节点，或者就是现在的头节点**。

因此，如果是在`releaseShared`方法中调用的`doReleaseShared`，可能此时调用方法的线程已经不是头节点所代表的线程了，头节点可能已经被易主好几次了。

**调用该方法的目的是什么？**

无论是在`acquireShared`中调用，还是在`releaseShared`方法中调用，该方法的目的都是在当前共享锁是可获取的状态时，**唤醒head节点的下一个节点**。这一点看上去和独占锁似乎一样，但是它们的一个重要的差别是——在共享锁中，当头节点发生变化时，是会回到循环中再**立即**唤醒head节点的下一个节点的。也就是说，在当前节点完成唤醒后继节点的任务之后将要退出时，如果发现被唤醒后继节点已经成为了新的头节点，则会立即触发**唤醒head节点的下一个节点**的操作，如此周而复始。

**退出该方法的条件是什么**

该方法是一个自旋操作(`for(;;)`)，退出该方法的唯一办法是走最后的break语句：

```java
if (h == head)   // loop if head changed
    break;
```

即，只有在当前head没有易主时，才会退出，否则继续循环。

为了说明问题，这里我们假设目前sync queue队列中依次排列有

> dummy node -> A -> B -> C -> D

现在假设A已经拿到了共享锁，则它将成为新的dummy node，

> dummy node (A) -> B -> C -> D

此时，A线程会调用doReleaseShared，我们写做`doReleaseShared[A]`，在该方法中将唤醒后继的节点B，它很快获得了共享锁，成为了新的头节点：

> dummy node (B) -> C -> D

此时，B线程也会调用doReleaseShared，我们写做`doReleaseShared[B]`，在该方法中将唤醒后继的节点C，但是别忘了，在`doReleaseShared[B]`调用的时候，`doReleaseShared[A]`还没运行结束呢，当它运行到`if(h == head)`时，发现头节点现在已经变了，所以它将继续回到for循环中，与此同时，`doReleaseShared[B]`也没闲着，它在执行过程中也进入到了for循环中。。。

由此可见，我们这里形成了一个doReleaseShared的“**调用风暴**”，大量的线程在同时执行doReleaseShared，这极大地加速了唤醒后继节点的速度，提升了效率，同时该方法内部的CAS操作又保证了多个线程同时唤醒一个节点时，只有一个线程能操作成功。

那如果这里`doReleaseShared[A]`执行结束时，节点B还没有成为新的头节点时，`doReleaseShared[A]`方法不就退出了吗？是的，但即使这样也没有关系，因为它已经成功唤醒了线程B，即使`doReleaseShared[A]`退出了，当B线程成为新的头节点时，`doReleaseShared[B]`就开始执行了，它也会负责唤醒后继节点的，这样即使变成这种每个节点只唤醒自己后继节点的模式，从功能上讲，最终也可以实现唤醒所有等待共享锁的节点的目的，只是效率上没有之前的“调用风暴”快。

明确了上面几个问题后，我们再来详细分析这个方法，它最重要的部分就是下面这两个if语句：

```java
if (ws == Node.SIGNAL) {
    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
        continue;            // loop to recheck cases
    unparkSuccessor(h);
}
else if (ws == 0 &&
         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
    continue;                // loop on failed CAS
```

- 第一个if很好理解，如果当前ws值为Node.SIGNAL，则说明后继节点需要唤醒，这里采用CAS操作先将Node.SIGNAL状态改为0，这是因为前面讲过，可能有大量的doReleaseShared方法在同时执行，我们只需要其中一个执行`unparkSuccessor(h)`操作就行了，这里通过CAS操作保证了`unparkSuccessor(h)`只被执行一次。
- 比较难理解的是第二个else if，首先我们要弄清楚ws啥时候为0，一种是上面的`compareAndSetWaitStatus(h, Node.SIGNAL, 0)`会导致ws为0，但是很明显，如果是因为这个原因，则它是不会进入到else if语句块的。所以这里的ws为0是指**当前队列的最后一个节点成为了头节点**。为什么是最后一个节点呢，因为每次新的节点加进来，在挂起前一定会将自己的前驱节点的waitStatus修改成Node.SIGNAL的。

其次，`compareAndSetWaitStatus(h, 0, Node.PROPAGATE)`这个操作什么时候会失败？

既然这个操作失败，说明就在执行这个操作的瞬间，ws此时已经不为0了，说明有新的节点入队了，ws的值被改为了Node.SIGNAL，此时我们将调用`continue`，在下次循环中直接将这个刚刚新入队但准备挂起的线程唤醒。

其实，如果我们再结合外部的整体条件，就很容易理解这种情况所针对的场景，不要忘了，进入上面这段还有一个条件是

```java
if (h != null && h != tail)
```

这个条件意味着，队列中至少有两个节点。

结合上面的分析，我们可以看出，这个

```java
else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
```

描述了一个极其严苛且短暂的状态：

1. 首先，大前提是队列里至少有两个节点。
2. 其次，要执行到`else if`语句，说明我们跳过了前面的if条件，说明头节点是刚刚成为头节点的，它的waitStatus值还为0，尾节点是在这之后刚刚加进来的，它需要执行`shouldParkAfterFailedAcquire`，将它的前驱节点（即头节点）的waitStatus值修改为`Node.SIGNAL`，**但是目前这个修改操作还没有来的及执行**。这种情况使我们得以进入else if的前半部分`else if (ws == 0 &&`
3. 紧接着，要满足`!compareAndSetWaitStatus(h, 0, Node.PROPAGATE)`这一条件，说明此时头节点的`waitStatus`已经不是0了，这说明之前那个没有来得及执行的 **在`shouldParkAfterFailedAcquire`将前驱节点的的waitStatus值修改为`Node.SIGNAL`的操作**现在执行完了。

由此可见，`else if` 的 `&&` 连接了两个不一致的状态，分别对应了`shouldParkAfterFailedAcquire`的`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`执行成功前和执行成功后，因为`doReleaseShared`和
`shouldParkAfterFailedAcquire`是可以并发执行的，所以这一条件是有可能满足的，只是满足的条件非常严苛，可能只是一瞬间的事。