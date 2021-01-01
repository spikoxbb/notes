[TOC]

# SynchronousQueue

是一个不存储元素的阻塞队列。每一个 put 操作必须等待一个 take 操作，否则不能继续添加元素。

SynchronousQueue 可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合于传递性场景,比如在一个线程中使用的数据，传递给另 外 一 个 线 程 使 用 ， SynchronousQueue 的 吞 吐 量 高 于LinkedBlockingQueue 和ArrayBlockingQueue。



它的特别之处在于它内部没有容器,一个生产线程,当它生产产品(即put 的时候),如果当前没有人想要消费产品(即当前没有线程执行 take),此生产线程必须阻塞,等待一个消费线程调用 take 操作,take 操作将会唤醒该生产线程,同时消费线程会获取生产线程的产品(即数据传递),这样的一个过程称为一次配对过程(当然也可以先 take 后 put,原理是一样的)。

不 像 ArrayBlockingQueue 、 LinkedBlockingDeque 之 类 的 阻 塞 队 列 依 赖 AQS 实 现 并 发 操 作 ,
SynchronousQueue 直接使用 CAS 实现线程的安全访问。

**公平模式**下,底层实现使用的是 TransferQueue 这个内部队列,它有一个 head 和 tail 指针,用于指向当前正在等待匹配的线程节点。队尾匹配队头出队。

1. 线程 put1 执行 put(1)操作,由于当前没有配对的消费线程,所以 put1 线程入队列,自旋一小会后睡眠等待,这时队列状态如下:

   head-->空-->put1<--tail

2. 接着,线程 put2 执行了 put(2)操作,跟前面一样,put2 线程入队列,自旋一小会后睡眠等待,这时队列:

   head-->空-->put1-->put2<--tail

3. 这时候,来了一个线程 take1,执行了 take 操作,由于 tail 指向 put2 线程,put2 线程跟 take1 线程配对了(一 put 一 take),这时 take1 线程不需要入队,但是要唤醒的线程并不是 put2,而是 put1。

**非公平模式**底层的实现使用的是TransferStack,一个栈,实现中用 head 指针指向栈顶.

1. 线程 put1 执行 put(1)操作,由于当前没有配对的消费线程,所以 put1 线程入栈,自旋一小会后睡眠等
   待,这时栈状态如下:

   head --> put1

2. 接着,线程 put2 再次执行了 put(2)操作,跟前面一样,put2 线程入栈,自旋一小会后睡眠等待,

   ​               put2

   head --> put1

3. 这时候,来了一个线程 take1,执行了 take 操作,这时候发现栈顶为 put2 线程,匹配成功,但是实现会先把 take1 线程入栈,然后 take1 线程循环执行匹配 put2 线程逻辑,一旦发现没有并发冲突,就会把栈顶指针直接指向 put1 线程