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

# 安全发布

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

## 不可变对象

创建后不再更改；均为final域；创建时this未逸出。

String不可变，在String类中有个私有实例字段hash表示该串的哈希值，在第一次调用hashCode方法时，字符串的哈希值被计算并且赋值给hash字段，之后再调用hashCode方法便可以直接取hash字段返回。final只是在编译期，jvm是没有final的，因此可用反射更改。

### 未被正确发布对象，其他线程看到的域为空引用或旧值

```java
public class Holder {
  private int n; 
  public Holder(int n){
      this.n = n;
  }
  public void assertSanity() { 
    if (n != n) {
        throw new AssertionError("This statement is false"); 
    }
  }
}

// Unsafe publication
public Holder holder;
public void initialize() {
     holder = new Holder(42);
}
```

其他线程可能看到的Holder域是一个空引用或者之前的旧值。

在构造函数内对一个final域的写入,与随后把这个被构造对象的引用赋值给一个引用变量,这两个操作之间不能重排序.

**`final`在对象的构造函数中设置对象的字段；并且不要在对象的构造函数完成之前在另一个线程可以看到它的地方编写对正在构造的对象的引用。如果执行此操作，则当另一个线程看到该对象时，该线程将始终看到该对象`final`字段的正确构造版本。它还将看到那些`final` 字段所引用的任何对象或数组的版本至少与这些`final`字段一样最新。**

1. JMM 禁止编译器把 final 域的**写重排序到构造函数之外。**
2. **编译器会在 final 域的写之后，构造函数 return 之前，插入一个 StoreStore 屏障。**这个屏障禁止处理器把 final 域的写重排序到构造函数之外。
3. 写普通域的操作可能被编译器重排序到了构造函数之外
4. "初次读对象引用"与"初次读该对象包含的 final 域"，JMM 禁止处理器重排序这两个操作， **编译器会在读 final 域操作的前面插入一个 LoadLoad 屏障。**初次读对象引用与初次读该对象包含的 final 域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作（**读引用|loadload|读final域**）。大多数处理器也会遵守间接依赖，大多数处理器也不会重排序这两个操作。但有少数处理器允许对存在间接依赖关系的操作做重排序（比如 alpha 处理器），这个规则就是专门用来针对这种处理器（ **alpha 处理器因此可能出现先读普通域再读引用导致值不对**）。

### 安全发布

对象的引用，状态需同时可见。

- 在静态初始化函数中初始化一个对象引用
- 对象引用存入volatile域或AtomicReferance中
- 对象引用存入某个正确的构造的对象的final域中
- 对象引用存入一个由锁保护的域中（包括线程安全容器，如copyOnWriteArrayList）

**安全发布，对象可见，状态也可见，若状态不改变，则访问安全。**

##　实例封闭

1. 包装器工厂方法，如Collections.synchronizedList(装饰器模式)，包装器对象拥有底层对象的唯一引用。
2. java监视器模式，封装可变对象，由内置锁保护。
3. 如Collections.unmodifiableMap(...)【赋值final域，防止容器对象被修改】

车辆追踪，map与point均未发布：

```java
//一致却不实时，数据极大时降低性能
return map --> deepCopy,Collections.unmodifiableMap(new HashMap(......));
//unmodifiableMap不能防止修改容器中的对象，因此复制point对象本身 copy point then put it in map1 then return unmodifiableMap(map1)
return point -->new point(map.get(id).x/y......); 

//若类中各组件安全，可能不需要额外的线程安全层，如point中 --> final x, y
return point --> return map.get(id)
//不可修改却实时
return map --> deepCopy,Collections.unmodifiableMap(map);
```

## 客户端加锁

```java
List list = new ArrayList();
//list依旧不安全
synchronized void f (E e){ 　----------> synchronized(list){......}
	boolean b=!list.contains(e)
    if(b) {list.add(e);}
    ......
}
```

## 迭代器的及时失败的检查是在未同步的情况下进行的

**迭代期间对容器加锁？**

如果容器的规模很大， 或者在每个元素上执行操作的时间很长， 那么这些线程将长时间等待，持有锁的时间越长，锁上的竞争就可能越激烈， 这将会极大地降低吞吐率。

**克隆容器迭代副本（克隆时加锁）？**

由于副本被封闭在线程内， 因此其他线程不会在选代期间对其进行修改。在克隆容器时存在显著的性能开销，比如容器过大。

### 容器的隐藏迭代

1. 容器的toString()、hashCode()和equals()方法（容器作为另一个容器的元素或者键或值可能会调用；containsAll()、removeAll()、retainAll()等方法）　
2. System.out.print(set) [调用了set.toString()方法，toString()会迭代容器]，StringBuildre.append(Object)会调用toString()
3. 把容器作为参数的**容器的**构造函数（会拷贝此容器中的所有元素）。

## 并发容器

java5.0增加了concurrentHashMap,CopyOnWriteArrayList,并增加了新容器类型Queue和BlockingQueue.

java6引入了concurrentSkipListMap和concurrentSkipListSet.(同步的SortedMap和SortedSet)