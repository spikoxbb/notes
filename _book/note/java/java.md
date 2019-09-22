[TOC]

## 面向对象

1. 继承.
2. 封装:通常认为封装是把数据和操作数据的方法绑定起来,对数据的访问只能通过已定义的接口。
3. 多态性:多态性是指允许不同子类型的对象对同一消息作出不同的响应。就是用同样的对象引用调用同样的方法但是做了不同的事情。多态性分为编译时的多态性和运行时的多态性。方法重载(overload)实现的是编译时的多态性(也称为前绑定),而方法重写(override)实现的是运行时的多态性(也称为后绑定)。

## 基础

### 重写

1. 构造方法不能被重写,声明为 final 的方法不能被重写,声明为 static 的方法不能被重写,但是能够被再次声明。
2. 访问权限不能比父类中被重写的方法的访问权限更低。
3. 重写的方法能够抛出任何非强制异常(UncheckedException,也叫非运行时异常),无论被重写的方法是否抛出异常。但是,重写的方法不能抛出新的强制性异常,或者比被重写方法声明的更广泛的强制性异常.

### 字符串

1. StringBuilder 是 Java5 中引入的,它和 StringBuffer 的方法完全相同,区别在于它是在单线程环境下使用的,因为它的所有方法都没有被 synchronized 修饰,因此它的效率理论上也比 StringBuffer 要高。StringBuffer:是线程安全的(对调用方法加入同步锁),执行效率较慢,适用于多线程下操作字符串缓冲区大量数据。
2. Java 编译器将"+"编译成了 StringBuilder.

### 日期

```java
//取得年月日、小时分钟秒
Calendar cal = Calendar.getInstance();
System.out.println(cal.get(Calendar.YEAR));
System.out.println(cal.get(Calendar.MONTH)); // 0 - 11
System.out.println(cal.get(Calendar.DATE));
System.out.println(cal.get(Calendar.HOUR_OF_DAY));
System.out.println(cal.get(Calendar.MINUTE));
System.out.println(cal.get(Calendar.SECOND));
// Java 8
LocalDateTime dt = LocalDateTime.now();
System.out.println(dt.getYear());
System.out.println(dt.getMonthValue()); // 1 - 12
System.out.println(dt.getDayOfMonth());
System.out.println(dt.getHour());
System.out.println(dt.getMinute());
System.out.println(dt.getSecond());

SimpleDateFormat oldFormatter = new SimpleDateFormat("yyyy/MM/dd");
Date date1 = new Date();
System.out.println(oldFormatter.format(date1));
// Java 8
DateTimeFormatter newFormatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
LocalDate date2 = LocalDate.now();
System.out.println(date2.format(newFormatter));
```

## 注解

Annatation(注解)是一个接口,程序可以通过反射来获取指定程序中元素的 Annotation对象,然后通过该 Annotation 对象来获取注解中的元数据信息。

## 异常

Throwable 是 Java 语言中所有错误或异常的超类。下一层分为 Error 和 Exception

## 内部类

1. 静态内部类可以访问外部类所有的静态变量和方法,即使是 private 的也一样。
2. 其它类使用静态内部类需要使用“外部类.静态内部类”方式,如下所示:Out.Inner inner =new Out.Inner();

定义在类内部的非静态类,就是成员内部类。成员内部类不能定义静态方法和变量(final 修饰的除外)。这是因为成员内部类是非静态的,类初始化的时候先初始化静态成员,如果允许成员内部类定义静态变量,那么成员内部类的静态变量初始化顺序是有歧义的。

## 泛型

泛型提供了编译时类型安全检测机制，类型擦除的基本过程也比较简单,首先是找到用来替换类型参数的具体类。这个具体类一般是 Object。如果指定了类型参数的上界的话,则使用这个上界。把代码中的类型参数都替换成具体的类。

## 序列化

使用 Java 对象序列化,在保存对象时,会把其状态保存为一组字节,在未来,再将这些字节组装成对象。必须注意地是,对象序列化保存的是对象的”状态”,即它的成员变量。由此可知,对象序列化不会关注类中的静态变量。

通过 ObjectOutputStream 和 ObjectInputStream 对对象进行序列化及反序列化。writeObject 和 readObject 自定义序列化策略在类中增加 writeObject 和 readObject 方法可以实现自定义序列化策略。

1. 在变量声明前加上 Transient 关键字,可以阻止该变量　被序列化到文件中,在被反序列化后,transient 变量的值被设为初始值,如 int 型的是 0,对象型的是 null。

2. 服务器端给客户端发送序列化对象数据,对象中有一些数据是敏感的,比如密码字符串等,希望对该密码字段在序列化时,进行加密,而客户端如果拥有解密的密钥,只有在客户端进行反序列化时,才可以对密码进行读取,这样可以一定程度保证序列化对象的数据安全。

3. #### 若实现的是Externalizable接口，则没有任何东西可以自动序列化，需要在writeExternal方法中进行手工指定所要序列化的变量，这与是否被transient修饰无关。

## IO/NIO

字节流读取的时候,读到一个字节就返回一个字节; 字符流使用了字节流读到一个或多个字节(中文对应的字节数是两个,在 UTF-8 码表中是 3 个字节)时。先去查指定的编码表,将查到的字符返回。 字节流可以处理所有类型数据,如:图片,MP3,AVI 视频文件,而字符流只能处理字符数据。

1. 阻塞 IO 模型：在读写数据过程中会发生阻塞现象。当用户线程发出 IO 请求之后,内核会去查看数据是否就绪,如果没有就绪就会等待数据就绪,而用户线程就会处于阻塞状态,用户线程交出 CPU。当数据就绪之后,内核会将数据拷贝到用户线程,并返回结果给用户线程,用户线程才解除 block 状态。典型的阻塞 IO 模型的例子为:data = socket.read();如果数据没有就绪,就会一直阻塞在 read 方法。

2. 非阻塞 IO 模型：当用户线程发起一个 read 操作后,并不需要等待,而是马上就得到了一个结果。如果结果是一个error 时,它就知道数据还没有准备好,于是它可以再次发送 read 操作。一旦内核中的数据准备好了,并且又再次收到了用户线程的请求,那么它马上就将数据拷贝到了用户线程,然后返回。所以事实上,在非阻塞 IO 模型中,用户线程需要不断地询问内核数据是否就绪,也就说非阻塞 IO不会交出 CPU,而会一直占用 CPU。典型的非阻塞 IO 模型一般如下：

   ```java
   //这样会导致 CPU 占用率非常高
   while(true){
       data = socket.read();
       if(data!= error){
           处理数据
           break;
       }
   }
   ```

3. 多路复用 IO 模型:Java NIO 实际上就是多路复用 IO。会有一个线程不断去轮询多个 socket 的状态,只有当 socket 真正有读写事件时,才真正调用实际的 IO 读写操作。Java NIO 中,是通过 selector.select()去查询每个通道是否有到达事件.在非阻塞 IO 中,不断地询问 socket 状态时通过用户线程去进行的,而在多路复用 IO 中,轮询每个 socket 状态是内核在进行的,这个效率要比用户线程要高的多。一旦事件响应体很大,那么就会导致后续的事件迟迟得不到处理,并且会影响新的事件轮询。

4. 信号驱动 IO 模型:当用户线程发起一个 IO 请求操作,会给对应的 socket 注册一个信号函数,然后用户线程会继续执行,当内核数据就绪时会发送一个信号给用户线程,用户线程接收到信号之后,便在信号函数中调用 IO 读写操作来进行实际的 IO 请求操作。

5. 异步 IO 模型:当用户线程发起 read 操作之后,立刻就可以开始去做其它的事。而另一方面,从内核的角度,当它受到一个 asynchronous read 之后,它会立刻返回,说明 read 请求已经成功发起了,因此不会对用户线程产生任何 block。然后,内
   核会等待数据准备完成,然后将数据拷贝到用户线程,当这一切都完成之后,内 核会给用户线程发送一个信号,告诉它 read 操作完成了。也就说用户线程完全不需要实际的整个 IO 操作是如何进行的,只需要先发起一个请求,当接收内核返回的成功信号时表示 IO 操作已经完成,可以直接去使用数据了。**在信号驱动模型中,当用户线程接收到信号表示数据已经就绪,然后需要用户线程调用 IO 函数进行实际的读写操作;而在异步 IO 模型中,收到信号表示 IO 操作已经完成,不需要再在用户线程中调用 IO 函数进行实际的读写操作。**

   异步 IO 是需要操作系统的底层支持,在 Java 7 中,提供了 Asynchronous IO。

   NIO 主要有三大核心部分:Channel(通道),Buffer(缓冲区), Selector。传统 IO 基于字节流和字符流进行操作,而 NIO 基于 Channel 和 Buffer(缓冲区)进行操作,数据总是从通道读取到缓冲区中,或者从缓冲区写入到通道中。Selector(选择区)用于监听多个通道的事件(比如:连接打开,数据到达)。因此,单个线程可以监听多个数据通道。

   1. Java IO 面向流意味着每次从流中读一个或多个字节,直至读取所有字节,它们没有被缓存在任何地方,且不能前后移动流中的数据。如果需要前后移动从流中读取的数据,需要先将它缓存到一个缓冲区。NIO 的缓冲导向方法不同。数据读取到一个它稍后处理的缓冲区,需要时可在缓冲区中前后移动。
   2. IO 的各种流是阻塞的。当一个线程调用 read() 或 write()时,该线程被阻塞,NIO 的非阻塞模式。

   NIO 中的 Channel 的主要实现有:
   1. FileChannel
   2. DatagramChannel
   3. SocketChannel
   4. ServerSocketChannel

   客户端发送数据时,必须先将数据存入 Buffer 中,然后将 Buffer 中的内容写入通道。服务端这边接收数据必须通过 Channel 将数据读入到 Buffer 中,然后再从 Buffer 中取出数据来处理。

   Selector 能够检测多个注册的通道上是否有事件发生,如果有事件发生,便获取事件然后针对每个事件进行相应的响应处理。

## 反射

运行中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性。

```java
Person person = (Person)clazz2.newInstance();//只能使用无参构造函数创建
```

```java
Constructor construtor3 = clazz.getConstructor(String.class,int.class);
Person person1 = (Person)construtor3.newInstance("xuhanfeng",23);//可以通过制定参数类型来获得特定的构造器
```

```java
Method method3 = clazz.getDeclaredMethod("setName", String.class);// 获得指定的方法
method3.invoke(person, "xuhuanfeng");// person 为前面获得的实例
```

```java
//要操作非public类型的域的时候，需要设置暂时关闭Java的安全检验:
Field field3 = clazz.getDeclaredField("name"); 
field3.setAccessible(true); //关闭安全校验 
field3.set(person, "xuhuanfeng"); 
```

## 动态代理

 通过 Proxy 类生成的代理类都继承了 Proxy 类，即 `DynamicProxyClass extends Proxy`。

```java
static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler handler)
{
    //1. 根据类加载器和接口创建代理类
    Class clazz = Proxy.getProxyClass(loader, interfaces); 
    //2. 获得代理类的带参数的构造函数
    Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class });
    //3. 创建代理对象，并制定调用处理器实例为参数传入
    Interface Proxy = (Interface)constructor.newInstance(new Object[] {handler});
}
```

- `InvocationHandler getInvocationHandler(Object proxy)`: 获得代理对象对应的调用处理器对象。
- `Class getProxyClass(ClassLoader loader, Class[] interfaces)`: 根据类加载器和实现的接口获得代理类。

Proxy 类中有一个映射表，<ClassLoader>=>{[ ]<Interfaces>==><ProxyClass>}，一个类加载器对象和一个接口数组确定了一个代理类。

InvocationHandler 接口:invoke(Object proxy, Method method, Object[] args),这个函数是在代理对象调用任何一个方法时都会调用的.

动态代理的缺点：因为 Java 的单继承特性（每个代理类都继承了 Proxy 类），只能针对接口创建代理类，不能针对类创建代理类。

## 集合

### List

ArrayList(数组)：内部是通过数组实现的（内部是用 Object[]实现的）,它允许对元素进行快速随机访问。当数组大小不满足时需要增加存储能力,就要将已经有数组的数据复制到新的存储空间中。当从 ArrayList 的中间位置插入或者删除元素时,需要对数组进行复制、移动、代价比较高。因此,它适合随机查找和遍历,不适合插入和删除。

[ArrayList源码分析](../code/ArrayList.md)

Vector(数组实现、线程同步)：支持线程的同步,即某一时刻只有一个线程能够写 Vector,避免多线程同时写而引起的不一致性,但实现同步需要很高的花费,因此,访问它比访问 ArrayList 慢。

LinkedList(链表)：使用了循环双向链表数据结构。,很适合数据的动态插入和删除,随机访问和遍历速度比较慢。另外,他还提供了 List 接口中没有定义的方法,专门用于操作表头和表尾元素,可以当作堆栈、队列和双向队列使用。链表内部都有一个 header 表项,它既表示链表的开始,也表示链表的结尾。表项 header 的后驱表项便是链表中第一个元素,表项 header 的前驱表项便是链表中最后一个元素。

[LinkedList源码分析](../code/LinkedList.md)

### Set

HashSet(Hash 表)：底层是由 HashMap 实现.HashSet 首先判断两个元素的哈希值,如果哈希值一样,接着会比较equals 方法 如果 equls 结果为 true ,HashSet 就视为同一个元素。

[HashSet源码分析](../code/HashSet.md)

TreeSet(二叉树)：使用二叉树的原理对新 add()的对象按照指定的顺序排序(升序、降序),每增加一个对象都会进行排序,将对象插入的二叉树指定的位置。（自定义的类必须实现 Comparable 接口,并且覆写相应的 compareTo()函数）

LinkHashSet( HashSet+LinkedHashMap )：继 承于HashSet 、 又 基 于 LinkedHashMap 来 实 现 的 。底层使用 LinkedHashMap 来保存所有元素。底层构造一个 LinkedHashMap 来实现,在相关操作上与父类 HashSet 的操作相同,直接调用父类 HashSet 的方法即可。

### Map

HashMap(数组+链表)：HashMap 最多只允许一条记录的键为 null,允许多条记录的值为 null。

**java7:**

![](../img/1566458240(1).png)

大方向上,HashMap 里面是一个数组,然后数组中每个元素是一个单向链表。上图中,每个绿色的实体是嵌套类 Entry 的实例,Entry 包含四个属性:key, value, hash 值和用于单向链表的 next。

1. capacity:当前数组容量,始终保持 2^n,可以扩容,扩容后数组大小为当前的 2 倍。
2. loadFactor:负载因子,默认为 0.75。
3. threshold:扩容的阈值,等于 capacity * loadFactor

**java8:**

由 数组+链表+红黑树 组成。Java7 HashMap查找的时候,根据 hash 值我们能够快速定位到数组的具体下标,但是之后的话,需要顺着链表一个个比较下去才能找到,时间复杂度为 O(n)。在 Java8 中,当链表中的元素超过了 8 个以后,会将链表转换为红黑树,在这些位置进行查找的时候可以降低时间复杂度为 O(logN)。

![](../img/1566458572(1).png)

[HashMap源码分析](../code/HashMap.md)

----

ConcurrentHashMap:由一个个 Segment 组成,Segment 代表”部分“或”一段“的意思,所以很多地方都会将其描述为分段锁。注意,行文中,我很多地方用了“槽”来代表一个segment。ConcurrentHashMap 是一个 Segment 数组,Segment 通过继承
ReentrantLock 来进行加锁,所以每次需要加锁的操作锁住的是一个 segment,这样只要保证每个 Segment 是线程安全的,也就实现了全局的线程安全。

![](../img/1566458775(1).png)

并行度(默认 16):concurrencyLevel:并行级别、并发数、Segment 数,怎么翻译不重要,理解它。默认是 16,也就是有 16 个 Segments,所以理论上,这个时候,最多可以同时支持 16 个线程并发写,只要它们的操作分别分布在不同的 Segment 上。这个值可以在初始化的时候设置为其他值,但是一旦初始化以后,它是不可以扩容的。

Java8 对 ConcurrentHashMap 进行了比较大的改动,Java8 也引入了红黑树:

![](../img/1566458572(1).png)

[ConcurrentHashMap源码分析](../code/ConcurrentHashMap.md)

----

HashTable(线程安全):承自 Dictionary 类,任一时间只有一个线程能写 Hashtable.不支持 null 值和 null 键;

TreeMap(可排序): 实现 SortedMap 接口,根据键排序,默认是按键值的升序排序,也可以指定排序的比较器,key 必须实现 Comparable 接口或者在构造 TreeMap 传入自定义的Comparator,否则会在运行时抛出 java.lang.ClassCastException .

LinkHashMap(记录插入顺序):LinkedHashMap 是 HashMap 的 一 个 子 类 , 保 存 了 记 录 的 插 入 顺 序 , 在 用 Iterator 遍 历LinkedHashMap 时,先得到的记录肯定是先插入的,也可以在构造时带参数,按照访问次序排序。

[ConcurrentHashMap源码分析](../code/ConcurrentHashMaps.md)

## 线程

### 启动方式

1. 继承 Thread 类：Thread 类本质上是实现了 Runnable 接口的一个实Thread 类本质上是实现了 Runnable 接口的一个实例,代表一个线程的实例。启动线程的唯一方法就是通过 Thread 类的 start()实例方法。start()方法是一个 native 方法,它将启动一个新线程,并执行 run()方法。
```java
public class MyThread extends Thread {
    public void run() {
        System.out.println("MyThread.run()");
    }
}
MyThread myThread1 = new MyThread();
myThread1.start();
```

2. 实现 Runnable 接口：如果自己的类已经 extends 另一个类,就无法直接 extends Thread,此时,可以实现一个Runnable 接口。

   ```Java
   public class MyThread extends OtherClass implements Runnable {
       public void run() {
           System.out.println("MyThread.run()");
       }
   }
   //启动 MyThread,需要首先实例化一个 Thread,并传入自己的 MyThread 实例:
   MyThread myThread = new MyThread();
   Thread thread = new Thread(myThread);
   //事实上,当传入一个 Runnable target 参数给 Thread 后,Thread 的 run()方法就会调用
   target.run()
   public void run() {
       if (target != null) {
           target.run();
       }
   }
   ```

3. ExecutorService、Callable<Class>、Future 有返回值线程有返回值的任务必须实现 Callable 接口,可以获取一个 Future 的对象。

   ```java
   ExecutorService pool = Executors.newFixedThreadPool(taskSize);
   List<Future> list = new ArrayList<Future>();
   for (int i = 0; i < taskSize; i++) {
       Callable c = new MyCallable(i + " ");
       Future f = pool.submit(c);
       list.add(f);
   }
   pool.shutdown();
   for (Future f : list) {
   System.out.println("res:" + f.get().toString());
   }
   ```



### ThreadLocal

[ThreadLocal源码解析](code/ThreadLocal.md)

### 线程池

#### 使用

线程池的顶级接口是 Executor,但是严格意义上讲 Executor 并不是一个线程池,而只是一个执行线程的工具。真正的线程池接口是 ExecutorService。

有几种不同的方式来将任务委托给 ExecutorService 去执行:

1. execute(Runnable):略

2. submit(Runnable)

   ```java
   Future future = executorService.submit(new Runnable()...);
   future.get();//获得执行完 run 方法后的返回值,这里使用的 Runnable,所以这里没有返回值,返回的是 null。
   executorService.shutdown();
   ```

3. submit(Callable)：略

   ......

4. invokeAny(...)：invokeAny() 方法要求一系列的 Callable 或者其子接口的实例对象，并返回其中一个 Callable 对象的结果。无法保证返回的是哪个 Callable 的结果，只能表明其中一个已执行结束。如果其中一个任务执行结束(或者抛了一个异常),其他 Callable 将被取消。

   ```java
   Set<Callable<String>> callables = new HashSet<Callable<String>>();
   String result = executorService.invokeAny(callables);
   ```

5. invokeAll(...)：invokeAll() 返回一系列的 Future 对象。

   ```java
   List<Future<String>> futures = executorService.invokeAll(callables);
   ```

6. shutdown 只是将空闲的线程 interrupt() 了,shutdown()之前提交的任务可以继续执行直到结束。
    shutdownNow 是 interrupt 所有线程

  

  #### pool

  newCachedThreadPool：创建一个可根据需要创建新线程的线程池,但是在以前构造的线程可用时将重用它们。对于执行
  很多短期异步任务的程序而言,这些线程池通常可提高程序性能。调用 execute 将重用以前构造的线程(如果线程可用)。如果现有线程没有可用的,则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。因此,长时间保持空闲的线程池不会使用任何资源。

newFixedThreadPool：创建一个可重用固定线程数的线程池,以**共享的无界队列**方式来运行这些线程。在任意点,在大多数 nThreads 线程会处于处理任务的活动状态。如果在所有线程处于活动状态时提交附加任务,则在有可用线程之前,附加任务将在队列中等待。如果在关闭前的执行期间由于失败而导致任何线程终止,那么一个新线程将代替它执行后续的任务(如果需要)。在某个线程被显式地关闭之前,池中的线程将一直存在。

newScheduledThreadPool：创建一个线程池,它可安排在给定延迟后运行命令或者定期地执行。

```java
ScheduledExecutorService scheduledThreadPool= Executors.newScheduledThreadPool(3);
scheduledThreadPool.schedule(newRunnable(){
    @Override
    public void run() {
        System.out.println("延迟三秒");
    }
}, 3, TimeUnit.SECONDS);
scheduledThreadPool.scheduleAtFixedRate(newRunnable(){
    @Override
    public void run() {
        System.out.println("延迟 1 秒后每三秒执行一次");
    }
},1,3,TimeUnit.SECONDS);
```

newSingleThreadExecutor：Executors.newSingleThreadExecutor()返回一个线程池(这个线程池只有一个线程),这个线程
池可以在线程死后(或发生异常时)重新启动一个线程来替代原来的线程继续执行下去。

#### 线程池原理

我们可以继承重写Thread 类,在其 start 方法中添加不断循环调用传递过来的 Runnable 对象。 这就是线程池的实现原理。循环方法中不断获取 Runnable 是用 Queue 实现的,在获取下一个 Runnable 之前可以是阻塞的。**一般的线程池主要分为以下 4 个组成部分:

1. 线程池管理器:用于创建并管理线程池
2. 工作线程:线程池中的线程
3. 任务接口:每个任务必须实现的接口,用于工作线程调度其运行
4. 任务队列:用于存放待处理的任务,提供一种缓冲机制

![](/home/spiko/notes/img/4e60f650d0af3bffa60be96bd8b2e8c.png)

```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize, long keepAliveTime,
TimeUnit unit, BlockingQueue<Runnable> workQueue) {
this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
Executors.defaultThreadFactory(), defaultHandler);
}
```

1. corePoolSize:指定了线程池中的线程数量。

2. maximumPoolSize:指定了线程池中的最大线程数量。

3. keepAliveTime:当前线程池数量超过 corePoolSize 时,多余的空闲线程的存活时间,即多次时间内会被销毁。

4. unit:keepAliveTime 的单位。

5. workQueue:任务队列,被提交但尚未被执行的任务。

6. threadFactory:线程工厂,用于创建线程,一般用默认的即可

7. handler:拒绝策略,当任务太多来不及处理,如何拒绝任务。线程池中的线程已经用完了,无法继续为新任务服务,同时,等待队列也已经排满了,再也塞不下新任务了。这时候我们就需要拒绝策略机制合理的处理这个问题:

   1. AbortPolicy : 直接抛出异常,阻止系统正常运行

   2. CallerRunsPolicy在任务被拒绝添加后，会调用当前线程池的所在的线程去执行被拒绝的任务。

   3. DiscardOldestPolicy : 丢弃最老的一个请求,也就是即将被执行的一个任务,并尝试再次提交当前任务。

   4. DiscardPolicy : 该策略默默地丢弃无法处理的任务,不予任何处理。

      以上内置拒绝策略均实现了 RejectedExecutionHandler 接口,若以上策略仍无法满足实际需要,完全可以自己扩展 RejectedExecutionHandler 接口.

#### 线程池工作过程:

1. 线程池刚创建时,里面没有一个线程。任务队列是作为参数传进来的。不过,就算队列里面有任务,线程池也不会马上执行它们。
2. 当调用 execute() 方法添加一个任务时,线程池会做如下判断:
   1. 如果正在运行的线程数量小于 corePoolSize,那么马上创建线程运行这个任务;
   2. 如果正在运行的线程数量大于或等于 corePoolSize,那么将这个任务放入队列;
   3. 如果这时候队列满了,而且正在运行的线程数量小于 maximumPoolSize,那么还是要创建非核心线程立刻运行这个任务;
   4. 如果队列满了,而且正在运行的线程数量大于或等于maximumPoolSize,那么线程池会抛出异常RejectExecutionException。
3. 当一个线程完成任务时,它会从队列中取下一个任务来执行。
4. 当一个线程无事可做,超过一定的时间(keepAliveTime)时,线程池会判断,如果当前运行的线程数大于 corePoolSize,那么这个线程就被停掉。所以线程池的所有任务完成后,它最终会收缩到 corePoolSize 的大小。

### 线程生命周期(状态)：

- 新建状态(NEW)：当程序使用 new 关键字创建了一个线程之后,该线程就处于新建状态,此时仅由 JVM 为其分配内存,并初始化其成员变量的值。
- 就绪状态(RUNNABLE):当线程对象调用了 start()方法之后,该线程处于就绪状态。Java 虚拟机会为其创建方法调用栈和程序计数器,等待调度运行。
- 运行状态(RUNNING):如果处于就绪状态的线程获得了 CPU,开始执行 run()方法的线程执行体,则该线程处于运行状态。
- 阻塞状态(BLOCKED):阻塞状态是指线程因为某种原因放弃了 cpu 使用权,也即让出了 cpu timeslice,暂时停止运行。阻塞的情况分三种:
  1. 等待阻塞( o.wait-> 等待对列):运行(running)的线程执行 o.wait()方法,JVM 会把该线程放入等待队列(waitting queue)中。
  2. 同步阻塞 (lock-> 锁池 )：运行(running)的线程在获取对象的同步锁时,若该同步锁被别的线程占用,则 JVM 会把该线程放入锁池(lock pool)中。
  3. 其他阻塞 (sleep/join)：运行(running)的线程执行 Thread.sleep(long ms)或 t.join()方法,或者发出了 I/O 请求时,
     JVM 会把该线程置为阻塞状态。当 sleep()状态超时、join()等待线程终止或者超时、或者 I/O处理完毕时,线程重新转入可运行(runnable)状态。
- 线程死亡(DEAD)。

### 终止线程 4 种方式：

- 正常运行结束：程序运行结束,线程自动结束。

- 使用退出标志退出线程：常常有些线程是伺服线程。它们需要长时间的运行,只有在外部某些条件满足的情况下,才能关闭这些线程。使用一个变量来控制循环：

  ```java
  public class ThreadSafe extends Thread {
      public volatile boolean exit = false;
      public void run() {
          while (!exit){
              .......
          }
      }
  }
  ```

- Interrupt 方法结束线程：使用 interrupt()方法来中断线程有两种情况:

  1. 线程处于阻塞状态:当调用线程的 interrupt()方法时,会抛出 InterruptException 异常，捕获该异常,然后 break 跳出循环状态,从而结束这个线程的执行。
  2. 线程未处于阻塞状态:使用 isInterrupted()判断线程的中断标志来退出循环。

- stop 方法终止线程(线程不安全)：thread.stop()调用之后,创建子线程的线程就会抛出 ThreadDeatherror 的错误,并且会释放子线程所持有的所有锁，那么被保护数据就有可能呈现不一致性。

对于 sleep()方法,我们首先要知道该方法是属于 Thread 类中的。而 wait()方法,则是属于Object 类中的。在调用 sleep()方法的过程中,线程不会释放对象锁。而当调用 wait()方法的时候,线程会放弃对象锁,进入等待此对象的等待锁定池,只有针对此对象调用 notify()方法后本线程才进入对象锁定池准备获取对象锁进入运行状态。

JAVA 后台线程：在 Daemon 线程中产生的新线程也是 Daemon 的。线程则是 JVM 级别的,以 Tomcat 为例,如果你在 Web 应用中启动一个线程,这个线程的生命周期并不会和 Web 应用程序保持同步。也就是说,即使你停止了 Web 应用,这个线程依旧是活跃的。

#### synchronized 

#### 原理

[synchronized及锁优化](lock.md)

#### Synchronized 核心组件:

1. Wait Set:哪些调用 wait 方法被阻塞的线程被放置在这里;
2. Contention List:竞争队列,所有请求锁的线程首先被放在这个竞争队列中;
3. Entry List Contention List 中那些有资格成为候选资源的线程被移动到 Entry List 中;
4. OnDeck:任意时刻,最多只有一个线程正在竞争锁资源,该线程被成为 OnDeck;
5. Owner:当前已经获取到所资源的线程被称为 Owner;
6. !Owner:当前释放锁的线程。

![](../img/1566468812(1).png)

1. JVM 每次从队列的尾部取出一个数据用于锁竞争候选者(OnDeck),但是并发情况下,ContentionList 会被大量的并发线程进行 CAS 访问,为了降低对尾部元素的竞争,JVM 会将一部分线程移动到 EntryList 中作为候选竞争线程。
2. Owner 线程会在 unlock 时,将 ContentionList 中的部分线程迁移到 EntryList 中,并指定EntryList 中的某个线程为OnDeck 线程(一般是最先进去的那个线程)。
3. Owner 线程并不直接把锁传递给 OnDeck 线程,而是把锁竞争的权利交给 OnDeck,OnDeck 需要重新竞争锁。这样虽然牺牲了一些公平性,但是能极大的提升系统的吞吐量,在JVM 中,也把这种选择行为称之为“竞争切换”。
4. OnDeck 线程获取到锁资源后会变为 Owner 线程,而没有得到锁资源的仍然停留在 EntryList中。如果 Owner 线程被 wait 方法阻塞,则转移到 WaitSet 队列中,直到某个时刻通过 notify或者 notifyAll 唤醒,会重新进去 EntryList 中。
5. 处于 ContentionList、EntryList、WaitSet 中的线程都处于阻塞状态,该阻塞是由操作系统来完成的(Linux 内核下采用 pthread_mutex_lock 内核函数实现的)。
6. Synchronized 是非公平锁。 Synchronized 在线程进入 ContentionList 时,等待的线程会先尝试自旋获取锁,如果获取不到就进入 ContentionList,这明显对于已经进入队列的线程是不公平的,还有一个不公平的事情就是自旋获取锁的线程还可能直接抢占 OnDeck 线程的锁资源。

#### ReentantLock

继承接口 Lock 并实现了接口中定义的方法,他是一种可重入锁,除了能完成 synchronized 所能完成的所有工作外,还提供了诸如可响应中断锁、可轮询锁请求、定时锁等避免多线程死锁的方法。

1. tryLock()只是"试图"获取锁, 如果锁不可用, 不会导致当前线程被禁用,当前线程仍然继续往下执行代码. 而 lock()方法则是一定要获取到锁, 如果锁不可用, 就一直等待, 在未获得锁之前,当前线程并不继续向下执行.
2. getHoldCount() :查询当前线程保持此锁的次数,也就是执行此线程执行 lock 方法的次数。
3. getQueueLength():返回正等待获取此锁的线程估计数,比如启动 10 个线程,1 个线程获得锁,此时返回的是 9

非公平锁:JVM 按随机、就近原则分配锁的机制则称为不公平锁,ReentrantLock 在构造函数中提供了是否公平锁的初始化方式,默认为非公平锁。非公平锁实际执行的效率要远远超出公平锁,除非程序有特殊需要,否则最常用非公平锁的分配机制。

公平锁:指的是锁的分配机制是公平的,通常先对锁提出获取请求的线程会先被分配到锁,ReentrantLock 在构造函数中提供了是否公平锁的初始化方式来定义公平锁。

lock 和 lockInterruptibly,如果两个线程分别执行这两个方法,但此时中断这两个线程,lock 不会抛出异常,而 lockInterruptibly 会抛出异常。

```java
// 创建一个计数阈值为 5 的信号量对象, 只能 5 个线程同时访问
Semaphore semp = new Semaphore(5);
try {
    // 申请许可
    semp.acquire();
    try {
        // 业务逻辑
    } catch (Exception e) {
        
    } finally {
        // 释放许可
        semp.release();
    }
} catch (InterruptedException e) {
    
}
```

可以通过 AtomicReference<V>将一个对象的所有操作转化成原子操作。

1. 非公平锁性能比公平锁高 5~10 倍,因为公平锁需要在多核的情况下维护一个队列
2. Java 中的 synchronized 是非公平锁,ReentrantLock 默认的 lock()方法采用的是非公平锁。

### 中断及切换

线程中断(interrupt):中断一个线程,其本意是给这个线程一个通知信号,会影响这个线程内部的一个中断标识位。这个线程本身并不会因此而改变状态(如阻塞,终止等)。

1. 调用 interrupt()方法并不会中断一个正在运行的线程。也就是说处于 Running 状态的线程并不会因为被中断而被终止,仅仅改变了内部维护的中断标识位而已。
2. 调用 sleep()而使线程处于 TIMED-WATING 状态,这时调用 interrupt()方法,会抛出InterruptedException,从而使线程提前结束 TIMED-WATING 状态。
3. 许多声明抛出 InterruptedException 的方法(如 Thread.sleep(long mills 方法)),抛出异常前,都会清除中断标识位,所以抛出异常后,调用 isInterrupted()方法将会返回 false。
4. 中断状态是线程固有的一个标识位,可以通过此标识位安全的终止线程。比如,你想终止一个线程 thread 的时候,可以调用 thread.interrupt()方法,在线程的 run 方法内部可以根据 thread.isInterrupted()的值来优雅的终止线程。

线程上下文切换：CPU 给每个任务都服务一定的时间,然后把当前任务的状态保存下来,在加载下一任务的状态后,继续服务下一任务,任务的状态保存及再加载, 这段过程就叫做上下文切换。时间片轮转的方式使多个任务在同一颗 CPU 上执行变成了可能。

上下文：是指某一时间点 CPU 寄存器和程序计数器的内容。

寄存器：是 CPU 内部的数量较少但是速度很快的内存(与之对应的是 CPU 外部相对较慢的 RAM 主内存)。寄存器通过对常用值(通常是运算的中间值)的快速访问来提高计算机程序运行的速度。

程序计数器：是一个专用的寄存器,用于表明指令序列中 CPU 正在执行的位置,存的值为正在执行的指令的位置或者下一个将要被执行的指令的位置,具体依赖于特定的系统。

### 并发队列

常用的并发队列有阻塞队列和非阻塞队列,前者使用锁实现,后者则使用 CAS 非阻塞算法实现。

#### 阻塞队列

无法向一个 BlockingQueue 中插入 null。如果你试图插入 null, BlockingQueue 将会抛出一个NullPointerException.

1. 当队列中没有数据的情况下,消费者端的所有线程都会被自动阻塞(挂起),直到有数据放入队列。
2. 当队列中填满数据的情况下,生产者端的所有线程都会被自动阻塞(挂起),直到队列中有空的位置,线程被自动唤醒。

![](../img/1566498744(1).png)

抛异常:如果试图的操作无法立即执行,抛一个异常。
特定值:如果试图的操作无法立即执行,返回一个特定的值(常常是 true / false)。
阻塞:如果试图的操作无法立即执行,该方法调用将会发生阻塞,直到能够执行。
超时:如果试图的操作无法立即执行,该方法调用将会发生阻塞,直到能够执行,但等待时间不会超过给定
值。返回一个特定值以告知该操作是否成功(典型的是 true / false)。

java 中的阻塞队列:

1. **ArrayBlockingQueue** :由数组结构组成的有界阻塞队列。此队列按照先进先出(FIFO)的原则对元素进行排序。默认情况下不保证访问者公平的访问队列.创建一个公平的阻塞队列:

   ```java
   ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(1000,true);
   ```

   [ArrayBlockingQueue源码解析](code/ArrayBlockingQueue.md)

2. **LinkedBlockingQueue** :由链表结构组成的有界阻塞队列。按照先进先出(FIFO)的原则对元素进行排序。对于生产者
   端和消费者端分别采用了独立的锁来控制数据同步,这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据,以此来提高整个队列的并发性能。LinkedBlockingQueue 会默认一个类似无限大小的容量(Integer.MAX_VALUE)。

   [LinkedBlockingQueue源码解析](code/LinkedBlockingQueue.md)

   **tomcat 中任务队列 TaskQueue继承了 LinkedBlockingQueue 并且泛化类型固定了为 Runnalbe.重写了 offer,poll,take 方法。tomcat 中有个线程池 ThreadPoolExecutor,在 NIOEndPoint 中当 acceptor 线程接受到请求后,会把任务放入队列,然后 poller 线程从队列里面获取任务,然后就把任务放入线程池执行。这个 ThreadPoolExecutor 中的的一个参数就是 TaskQueue。**

3. **PriorityBlockingQueue** :支持优先级排序的无界阻塞队列。默认情况下元素采取自然顺序升序排列.可以自定义实现 compareTo()方法来指定元素进行排序规则,或者初始化 PriorityBlockingQueue 时,指定构造参数 Comparator 来对元素进行排序。不能保证同优先级元素的顺序。

   [PriorityBlockingQueue源码解析](code/PriorityBlockingQueue.md)

4. DelayQueue:支持延时获取元素的无界阻塞队列。队列使用 PriorityQueue 来实现。队列中的元素必须**实现 Delayed 接口,**在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将 DelayQueue 运用在以下应用场景:

   - 缓存系统的设计:可以用 DelayQueue 保存缓存元素的有效期,使用一个线程循环查询DelayQueue,一旦能从 DelayQueue 中获取元素时,表示缓存有效期到了。
   - 定 时 任 务 调 度 : 使 用 DelayQueue 保 存 当 天 将 会 执 行 的 任 务 和 执 行 时 间 , 一 旦 从DelayQueue 中获取到任务就开始执行,从比如 TimerQueue 就是使用 DelayQueue 实现的

   [DelayQueue源码解析](code/DelayQueue.md)

5. SynchronousQueue:不存储元素的阻塞队列。每一个 put 操作必须等待一个 take 操作,否则不能继续添加元素。非常适合于传递性场景,比如在一个线程中使用的数据,传递给另 外 一 个 线 程 使 用 ,

   [SynchronousQueue源码解析](code/SynchronousQueue.md)

6. LinkedTransferQueue:由链表结构组成的无界阻塞队列。相 对 于 其 他 阻 塞 队 列 ,LinkedTransferQueue 多了 tryTransfer 和 transfer 方法。

   1. transfer 方法:如果当前有消费者正在等待接收元素(消费者使用 take()方法或带时间限制的poll()方法时),transfer 方法可以把生产者传入的元素立刻 transfer(传输)给消费者。如果没有消费者在等待接收元素,transfer 方法会将元素存放在队列的 tail 节点,并等到该元素被消费者消费了才返回。
   2. tryTransfer 方法。则是用来试探下生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素,则返回 false。和 transfer 方法的区别是 tryTransfer 方法无论消费者是否接收,方法立即返回。而 transfer 方法是必须等到消费者消费了才返回。

7. LinkedBlockingDeque:由链表结构组成的双向阻塞队列.可以运用在“工作窃取”模式中(为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行)。

#### 非阻塞队列

非阻塞队列使用的是 CAS(compare and swap)来实现线程执行的非阻塞。

**ConcurrentLinkedQueue** :非阻塞无界链表队列.

[ConcurrentLinkedQueue源码解析](code/ConcurrentLinkedQueue.md)

### 其他

CountDownLatch(线程计数器 ):比如有一个任务 A,它要等待其他 2个任务执行完毕之后才能执行:

```java
final CountDownLatch latch = new CountDownLatch(2);
new Thread(){public void run() {
	System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
	Thread.sleep(3000);
	System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
	latch.countDown();
};}.start();
new Thread(){public void run() {
	System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
	Thread.sleep(3000);
	System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
	latch.countDown();
};}.start();
System.out.println("等待 2 个子线程执行完毕...");
latch.await();
System.out.println("2 个子线程已经执行完毕");
System.out.println("继续执行主线程");
}
```

CyclicBarrier(回环栅栏-等待至 barrier 状态再全部同时执行):让一组线程等待至某个状态(barrier状态)之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后,CyclicBarrier 可以被重用。调用 await()方法之后,线程就处于 barrier 了。

1. public int await():用来挂起当前线程,直至所有线程都到达 barrier 状态再同时执行后续任务;
2. public int await(long timeout, TimeUnit unit):让这些线程等待至一定的时间,如果还有线程没有到达 barrier 状态就直接让到达 barrier 的线程执行后续任务。

```java
public static void main(String[] args) {
	int N = 4;
	CyclicBarrier barrier = new CyclicBarrier(N);
	for(int i=0;i<N;i++)
		new Writer(barrier).start();
}
static class Writer extends Thread{
	private CyclicBarrier cyclicBarrier;
	public Writer(CyclicBarrier cyclicBarrier) {
	this.cyclicBarrier = cyclicBarrier;
	}
    @Override
	public void run() {
        try {
            Thread.sleep(5000);//以睡眠来模拟　线程需要预定写入数据操作
            System.out.println(" 线 程 "+Thread.currentThread().getName()+" 写 入 数 据 完毕,等待其他线程写入完毕");
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }catch(BrokenBarrierException e){
            e.printStackTrace();
        }
        System.out.println("所有线程写入完毕,继续处理其他任务,比如数据操作");
    }
}
```

Semaphore(信号量-控制同时访问的线程个数):S emaphore 翻译成字面意思为 信号量,Semaphore 可以控制同时访问的线程个数,通过acquire() 获取一个许可,如果没有就等待,而 release() 释放一个许可。

1. public void acquire(): 用来获取一个许可,若无许可能够获得,则会一直等待,直到获得许可。

2. public void acquire(int permits):获取 permits 个许可。

3. public void release() { } :释放许可。注意,在释放许可之前,必须先获获得许可。

4. public void release(int permits) { }:释放 permits 个许可。

   上面 4 个方法都会被阻塞,如果想立即得到执行结果,可以使用下面几个方法:

   1. public boolean tryAcquire():尝试获取一个许可,若获取成功,则立即返回 true,若获取失败,则立即返回 false。
   2. public boolean tryAcquire(long timeout, TimeUnit unit):尝试获取一个许可,若在指定的时间内获取成功,则立即返回 true,否则则立即返回 false。
   3. public boolean tryAcquire(int permits):尝试获取 permits 个许可,若获取成功,则立即返回 true,若获取失败,则立即返回 false。
   4. public boolean tryAcquire(int permits, long timeout, TimeUnit unit): 尝试获取 permits个许可,若在指定的时间内获取成功,则立即返回 true,否则则立即返回 false。
   5. 通过 availablePermits()方法得到可用的许可数目。

若一个工厂有 5 台机器,但是有 8 个工人,一台机器同时只能被一个工人使用,只有使用完了,其他工人才能继续使用：

```java
int N = 8;
//工人数
Semaphore semaphore = new Semaphore(5); //机器数目
for(int i=0;i<N;i++)
	new Worker(i,semaphore).start();
}
static class Worker extends Thread{
    private int num;
    private Semaphore semaphore;
    public Worker(int num,Semaphore semaphore){
        this.num = num;
        this.semaphore = semaphore;
    }
    @Override
    public void run() {
        try {
            semaphore.acquire();
            System.out.println("工人"+this.num+"占用一个机器在生产...");
            Thread.sleep(2000);
            System.out.println("工人"+this.num+"释放出机器");
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**CountDownLatch 一般用于某个线程 A 等待若干个其他线程执行完任务之后执行;而 CyclicBarrier 一般用于一组线程互相等待至某个状态,然后这一组线程再同时执行;另外,CountDownLatch 是不能够重用的,而 CyclicBarrier 是可以重用的。Semaphore 其实和锁有点类似,它一般用于控制对某组资源的访问权限。**

当对非 volatile 变量进行读写的时候,每个线程先从内存拷贝变量到 CPU 缓存中。如果计算机有多个 CPU,每个线程可能在不同的 CPU 上被处理,这意味着每个线程可以拷贝到不同的 CPUcache 中。而声明变量是 volatile 的,JVM 保证了每次读变量都从内存中读,跳过 CPU cache这一步。

synchronized 和 ReentrantLock的共同点:都是可重入锁,同一线程可以多次获得同一个锁

不同点:synchronized 在发生异常时,会自动释放线程占有的锁,因此不会导致死锁现象发生;而 Lock 在发生异常时,如果没有主动通过 unLock()去释放锁,则很可能造成死锁现象,因此使用 Lock 时需要在 finally 块中释放锁。

Synchronized进过编译，会在同步块的前后分别形成monitorenter和monitorexit这个两个字节码指令。在执行monitorenter指令时，首先要尝试获取对象锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象锁，把锁的计算器加1，相应的，在执行monitorexit指令时会将锁计算器就减1，当计算器为0时，锁就被释放了。如果获取对象锁失败，那当前线程就要阻塞，直到对象锁被另一个线程释放为止。

ReenTrantLock的实现是一种自旋锁，通过循环调用CAS操作来实现加锁。它的性能比较好也是因为避免了使线程进入内核态的阻塞状态。想尽办法避免线程进入内核的阻塞状态是我们去分析和理解锁设计的关键钥匙。使用链表作为队列，使用volatile变量state，作为锁状态标识位。

lock():

 (1)通过原子的比较并设置操作，如果成功设置，说明锁是空闲的，当前线程获得锁，并把当前线程设置为锁拥有者；
 (2)否则，调用acquire方法；

```java
package java.util.concurrent.locks.ReentrantLock;
final void lock() {
            if (compareAndSetState(0, 1))//表示如果当前state=0，那么设置state=1，并返回true；否则返回false。由于未等待，所以线程不需加入到等待队列
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
}
 
 package java.util.concurrent.locks.AbstractOwnableSynchronizer  //AbstractOwnableSynchronizer是AQS的父类
 protected final void setExclusiveOwnerThread(Thread t) {
            exclusiveOwnerThread = t;
}
//如果当前锁已经被持有，那么接下来进行可重入检查，如果可重入，需要为锁状态加上请求数。如果不属于上面两种情况，那么说明锁是被其他线程持有，当前线程应该放入等待队列。在放入等待队列的过程中，首先要检查队列是否为空队列，如果为空队列，需要创建虚拟的头节点，然后把对当前线程封装的节点加入到队列尾部。由于设置尾部节点采用了CAS，为了保证尾节点能够设置成功，这里采用了无限循环的方式，直到设置成功为止。如果当前节点之前的节点的等待状态小于1，说明当前节点之前的线程处于等待状态(挂起)，那么当前节点的线程也应处于等待状态(挂起)。挂起的工作是由LockSupport类支持的，LockSupport通过JNI调用本地操作系统来完成挂起的任务(java中除了废弃的suspend等方法，没有其他的挂起操作).在当前等待的线程，被唤起后，检查中断状态，如果处于中断状态，那么需要中断当前线程。
```

ConcurrentHashMap 并发:它内部细分了若干个小的 HashMap,称之为段(Segment)。如果在 ConcurrentHashMap 中添加一个新的表项,并不是将整个 HashMap 加锁,而是首先根据 hashcode 得到该表项应该存放在哪个段中,然后对该段加锁,并完成 put 操作。ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。Segment 是一种可重入锁 ReentrantLock,,HashEntry 则用于存储键值对数据。每个 Segment 守护一个 HashEntry 数组里的元素,当对 HashEntry 数组的数据进行修改时,必须首先获得它对应的 Segment 锁。

### Java 中用到的线程调度:

1. 抢占式调度:每条线程执行的时间、线程的切换都由系统控制,系统控制指的是在系统某种运行机制下,可能每条线程都分同样的执行时间片,也可能是某些线程执行的时间片较长,甚至某些线程得不到执行的时间片。在这种机制下,一个线程的堵塞不会导致整个进程堵塞。
2. 某一线程执行完后主动通知系统切换到另一线程上执行.

java 使用的线程调使用抢占式调度,进程调度算法:

1. 先来先服务调度算法(FCFS):当在作业调度中采用该算法时,每次调度都是从后备作业队列中选择一个或多个最先进入该队列的作业,将它们调入内存,为它们分配资源、创建进程,然后放入就绪队列。在进程调度中采用 FCFS 算法时,则每次调度是从就绪队列中选择一个最先进入该队列的进程,为之分配处理机,该进程一直运行到完成或发生某事件而阻塞后才放弃处理机.
2. 短作业优先(SJF):从后备队列中选择一个或若干个估计运行时间最短的作业,将它们调入内存运行。而短进程优先(SPF)调度算法则是从就绪队列中选出一个估计运行时间最短的进程,将处理机分配给它.使它立即执行并一直执行到完成,或发生某事件而被阻塞放弃处理机时再重新调度。
3. 非抢占式优先权算法:在这种方式下,系统一旦把处理机分配给就绪队列中优先权最高的进程后,该进程便一直执行下去,直至完成;或因发生某事件使该进程放弃处理机时。这种调度算法主要用于批处理系统中;也可用于某些对实时性要求不严的实时系统中。
4. 抢占式优先权调度算法:把处理机分配给优先权最高的进程,使之执行。但在其执行期间,只要又出现了另一个其优先权更高的进程,进程调度程序就立即停止当前进程(原优先权最高的进程)的执行,重新将处理机分配给新到的优先权最高的进程。显然,这种抢占式的优先权调度算法能更好地满足紧迫作业的要求,故而常用于要求比较严格的实时系统中,以及对性能要求较高的批处理和分时系统中。
5. 高响应比优先调度算法:短作业优先算法是一种比较好的算法,其主要的不足之处是长作业的运行得不到保证。使作业的优先级随着等待时间的增加而以速率 a 提高,则长作业在等待一定的时间后,必然有机会分配到处理机。在
   利用该算法时,每要进行调度之前,都须先做响应比的计算,这会增加系统开销。
6. 时间片轮转法:系统将所有的就绪进程按先来先服务的原则排成一个队列,每次调度时,把 CPU 分配给队首进程,并令其执行一个时间片。时间片的大小从几 ms 到几百 ms。当执行的时间片用完时,由一个计时器发出时钟中断请求,调度程序便据此信号来停止该进程的执行,并将它送往就绪队列的末尾;然后,再把处理机分配给就绪队列中新的队首进程,同时也让它执行一个时间片。这样就可以保证就绪队列中的所有进程在一给定的时间内均能获得一时间片的处理机执行时间。
7. 多级反馈队列调度算法:
   - 应设置多个就绪队列,并为各个队列赋予不同的优先级。第一个队列的优先级最高,第二个队列次之,在优先权愈高的队列中,为每个进程所规定的执行时间片就愈小。
   - 当一个新进程进入内存后,首先将它放入第一队列的末尾,按 FCFS 原则排队等待调度。当轮到该进程执行时,如它能在该时间片内完成,便可准备撤离系统;如果它在一个时间片结束时尚未完成,调度程序便将该进程转入第二队列的末尾,再同样地按 FCFS 原则等待调度执行;如果它在第二队列中运行一个时间片后仍未完成,再依次将它放入第三队列,......,如此下去,当一个长作业(进程)从第一队列依次降到第 n 队列后,在第 n 队列便采取按时间片轮转的方式运行。
   - 仅当第一队列空闲时,调度程序才调度第二队列中的进程运行;如果处理机正在第 i 队列中为某进程服务时,又有新进程进入优先权较高的队列(第 1~(i-1)中的任何一个队列),则此时新进程将抢占正在运行进程的处理机,即由调度程序把正在运行的进程放回到第 i 队列的末尾,把处理机分配给新到的高优先权进程。

### CAS 

CAS(Compare And Swap/Set)比较并交换,CAS 算法的过程是这样:它包含 3 个参数CAS(V,E,N)。V 表示要更新的变量(内存值),E 表示预期值(旧的),N 表示新值。当且仅当 V 值等于 E 值时,才会将 V 的值设为 N,如果 V 值和 E 值不同,则说明已经有其他线程做了更新,则当前线程什么都不做.最后,CAS 返回当前 V 的真实值。

原子包 java.util.concurrent.atomic(锁自旋):

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private volatile int value;
	public final int get() {
		return value;
	}
    public final int getAndIncrement() {
for (;;) { 
    //CAS 自旋,一直尝试,直达成功
    int current = get();
    int next = current + 1;
    if (compareAndSet(current, next))
        return current;
}
 public final boolean compareAndSet(int expect, int update) {
     return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
 }
}
```

ABA 问题：比如说一个线程 one 从内存位置 V 中取出 A,这时候另一个线程 two 也从内存中取出 A,并且two 进行了一些操作变成了 B,然后 two 又将 V 位置的数据变成 A,这时候线程 one 进行 CAS 操作发现内存中仍然是 A,然后 one 操作成功。尽管线程 one 的 CAS 操作成功,但是不代表这个过程就是没有问题的。部分乐观锁的实现是通过版本号(version)的方式来解决 ABA 问题,乐观锁每次在执行数据的修改操作时,都会带上一个版本号,一旦版本号和数据的版本号一致就可以执行修改操作并对版本号执行+1 操作,否则就执行失败。因为每次操作的版本号都会随之增加,所以不会出现 ABA 问题,因为版本号只会增加不会减少。

### AQS( 抽象的队列同步器 )：

AbstractQueuedSynchronizer 类如其名,抽象的队列式的同步器,AQS 定义了一套多线程访问共享资源的同步器框架,许多同步类实现都依赖于它,如常用的ReentrantLock/Semaphore/CountDownLatch。

![](../img/1566537014.png)

[AQS原理](code/AQS.md)