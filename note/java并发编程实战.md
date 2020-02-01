[TOC]
# 先序

### RMI调用创建多少个线程？

-  在JDK1.5及以前版本中，RMI每接收一个远程方法调用就生成一个单独的线程来处理这个请求，请求处理完成后，这个线程就会释放.
- 在JDK1.6之后，RMI使用线程池来处理新接收的远程方法调用请求-ThreadPoolExecutor。

### 重载synchronized方法是否会获取父类的monitor lock?

实例化子类不会实例化父类，会锁住是因为调用super的方法是使用的子类对象，所以才会说锁住的是同一个对象。

个人理解super只是个标记，用于告诉jvm需要调用超类的方法。super关键字在字节码里面对应的是invokespecial指令。
invokespecial指令用于调用实例初始化，父类初始化和私有方法。功能与super完美契合，可以说super就是java对invokespecial指令的实现。

### 未同步时，编译，运行时可能重排序

为什么重排序？

- 编译期重排序。编译器的本意是提升程序在CPU上的运行性能，**减少汇编指令的数量，降低程序执行需要的CPU周期，减少CPU读写主存的时间**。int a, b; a = b + 11; b = 0;＝＝》int a, b; b = 0; a = b + 11;编译器会优化掉它认为不需要的一些变量，同时也会将一些本应去内存中取得数据存入寄存器中，然后下次取得时候就可以直接从寄存器中获取（**这样也可能导致多线程中共享变量的可见性问题**）。
- 处理器重排序。比如就是不管代码里这个值使用顺序多靠后，都先用**mov**加载后再使用**add**对这个值进行运算。

### volatile不会将内存变量和其他内存操作一起重排序。

- 如双重检查锁构造单例，如果没有使用volatile修饰singleton，就可能会造成错误。这是因为使用new关键字初始化一个对象的过程并不是一个原子的操作，它分成三个步骤进行：a. 给 singleton 分配内存　 b. 调用 Singleton 的构造函数来初始化成员变量　 c. 将 singleton 对象指向分配的内存空间（执行完这步 singleton 就为非 null 了）

  如果虚拟机存在指令重排序优化，则步骤b和c的顺序是无法确定的。如果A线程率先进入同步代码块并先执行了c而没有执行b，此时因为singleton已经非null。这时候线程B判断singleton非null并将其返回使用，因为此时Singleton实际上还未初始化，自然就会出错。synchronized可以解决内存可见性，但是不能解决重排序问题。

- **有volatile修饰的变量，赋值后多执行了一个“load addl $0x0, (%esp)”操作，这个操作相当于一个内存屏障**（指令重排序时不能把后面的指令重排序到内存屏障之前的位置）。

  - 在指令前插入Load Barrier，可以让高速缓存中的数据失效，强制从新从主内存加载数据；

  - 在指令后插入Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。

java的内存屏障通常所谓的四种即LoadLoad,StoreStore,LoadStore,StoreLoad实际上也是上述两种的组合：

-  LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
-  StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能

**在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障； 在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障；**

## 逸出

指不应该发布的对象被发布，如返回私有数组。

### 内部类导致逸出

```java
public class A {
    public A(B b){
        f2(C c){       
            new D(){
                public void f3(E e){......}
            }
        }
    }
}
```

1. A发布D时隐含地发布了A实例本身，因为内部类持有外部类的引用。
2. 从构造函数中发布对象时如上，发布了尚未构造完成的对象（如上，this引用在构造过程中逸出）。

#### 为什么内部类持有外部类的引用

编译器自动为内部类添加一个成员变量(this$0), 这个成员变量的类型和外部类的类型相同， 编译器自动为内部类的构造方法添加一个参数， 参数的类型是外部类的类型， 在构造方法内部使用这个参数为内部类中添加的成员变量赋值；在调用内部类的构造函数初始化内部类对象时，会默认传入外部类的引用。

## 线程封闭

仅在单线程内访问数据，不需要同步。如connection隐含地被封闭于线程中（安全），connection返回之前，连接池不会将它分配于其他线程。

#### Ad-hoc线程封闭

指维护线程封闭性的职责完全由程序实现来承担。Ad-hoc线程封闭是非常脆弱的，因为没有任何一种语言特性，例如可见性修饰符或局部变量，能将对象封闭到目标线程上。

#### ThreadLocal封闭

ThreadLocal对每一个使用该变量的线程都存有一份副本。

JDBC初始化Connection对象，避免在每调用一个方法时传递一个connection对象。

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<connection>(){
    public Connection initialValue(){
		return DriverManager.getConnection(url);
    }
}
public Connection getConnection(){
    return connectionHolder.get();
}
```

