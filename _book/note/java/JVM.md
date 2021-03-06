[TOC]

## 内存模型

![](../img/1566372915(1).png)

程序计数器(线程私有):是当前线程所执行的字节码的行号指示器,正在执行 java 方法的话,计数器记录的是虚拟机字节码指令的地址(当前指令的地址)。如果还是 Native 方法,则为空。

虚拟机栈(线程私有):用于存储局部变量表、操作数栈、动态链接、方法出口等信息。

本地方法区(线程私有):HotSpot VM 直接就把本地方法栈和虚拟机栈合二为一。

堆(Heap-线程共享)-运行时数据区:创建的对象和数组都保存在 Java 堆内存中,也是垃圾收集器进行垃圾收集的最重要的内存区域.

方法区/永久代(线程共享):存储被 JVM 加载的类信息、常量、静态变量、即时编译器编译后的代码等数据.运行时常量池(Runtime Constant Pool)是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述等信息外,还有一项信息是常量池用于存放编译期生成的各种字面量和符号引用,这部分内容将在类加载后存放到方法区的运行时常量池中。

Java 堆从 GC 的角度还可以细分为: 新生代( Eden 区 、 From Survivor 区 和 To Survivor 区 )和老年代。是用来存放新生的对象。一般占据堆的 1/3 空间。由于频繁创建对象,所以新生代会频繁触发MinorGC 进行垃圾回收。新生代又分为 Eden 区、ServivorFrom、ServivorTo 三个区。

Eden是Java 新对象的出生地(如果新创建的对象占用内存很大,则直接分配到老年代)。当 Eden 区内存不够的时候就会触发 MinorGC,对新生代区进行一次垃圾回收。

## GC

### 定位对象

- 引用计数器算法(废弃)
  引用计数器算法是给每个对象设置一个计数器,当有地方引用这个对象的时候,计数器+1,当引用失效的时候,
  计数器-1,当计数器为 0 的时候,JVM 就认为对象不再被使用,是“垃圾”了。
  引用计数器实现简单,效率高;但是不能解决循环引用问问题(A 对象引用 B 对象,B 对象又引用 A 对象,但是
  A,B 对象已不被任何其他对象引用),同时每次计数器的增加和减少都带来了很多额外的开销,所以在 JDK1.1 之后,
  这个算法已经不再使用了。

- 根搜索算法(使用)
  根搜索算法是通过一些“GC Roots”对象作为起点,从这些节点开始往下搜索,搜索通过的路径成为引用链
  (Reference Chain),当一个对象没有被 GC Roots 的引用链连接的时候,说明这个对象是不可用的。

  GC Roots 对象包括:

  1. 虚拟机栈(栈帧中的本地变量表)中的引用的对象。
  2.  方法区域中的类静态属性引用的对象。
  3.  方法区域中常量引用的对象。
  4.  本地方法栈中 JNI(Native 方法)的引用的对象。

### GC类型

MinorGC：把 Eden 和 ServivorFrom 区域中存活的对象复制到 ServicorTo 区域(如果有对象的年龄以及达到了老年的标准,则赋值到老年代区),同时把这些对象的年龄+1(如果 ServicorTo 不够位置了就放到老年区)。

老年代在进行 MajorGC 前一般都先进行了一次 MinorGC,使得有新生代的对象晋身入老年代,导致空间不够用时才触发。当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次 MajorGC 进行垃圾回收腾出空间。

MajorGC 采用标记清除算法，MajorGC 会产生内存碎片,为了减少内存损耗,我们一般需要进行合并或者标记出来方便下次直接分配。当老年代也满了装不下的时候,就会抛出 OOM(Out of Memory)异常。

GC 不会在主程序运行期对永久区域进行清理。在 Java8 中,永久代已经被移除,被一个称为“元数据区”(元空间)的区域所取代，元空间并不在虚拟机中,而是使用本地内存。类的元数 据放入 nativememory, 字符串池和类的静态变量放入 java 堆中。

可达性分析：通过一系列的“GC roots”对象作为起点搜索。如果在“GC roots”和一个对象之间没有可达路径,则称该对象是不可达的。不可达对象变为可回收对象至少要经过两次标记过程。两次标记后仍然是可回收对象,则将面临回收。

### GC算法

1. 标记清除算法( Mark-Sweep ):标记阶段标记出所有需要回收的对象,清除阶段回收被标记的对象所占用的空间。但是内存碎片化严重。
2. 复制算法(copying)：
3. 标记整理算法(Mark-Compact)：将存活对象移向内存的一端。然后清除端边界外的对象。

1. 对永生代的回收主要包括废弃常量和无用的类。
2. 对象的内存分配主要在新生代的 Eden Space 和 Survivor Space 的 From Space（两者空间不足时空间不足时就会发生一次 GC，To Space无法足够存储某个对象,则将这个对象存储到老生代。），少数情况会直接分配到老生代。
3. 当对象在 Survivor 区躲过一次 GC 后,其年龄就会+1。默认情况下年龄到达 15 的对象会被移到老生代中。

强引用：把一个对象赋给一个引用变量,这个引用变量就是一个强引用，它处于可达状态,永远都不会被回收。

软引用：当系统内存足够时它不会被回收,当系统内存空间不足时它会被回收。

```java
SoftReference<String> s=new SoftReference<String>(new String("soft"));
//可以和引用队列联合使用,如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。
```

弱引用：只要垃圾回收机制一运行,不管 JVM 的内存空间是否足够,总会回收该对象占用的内存。

```java
WeakReference<String> s=new WeakReference<String>(new String("weak"));
//可以和引用队列联合使用,如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。
```

虚引用：它不能单独使用,必须和引用队列联合使用。虚引用的主要作用是跟踪对象被垃圾回收的状态。

```java
LinkedList<PhantomReference<String>> pa=LinkedList<PhantomReference<String>>(new String("phantom"),new ReferenceQueue);
```

分区收集算法:将整个堆分为连续的不同小区间, 每个小区间独立使用, 独立回收. 这样可以控制一次回收多少个小区间 , 根据目标停顿时间, 每次合理地回收若干个小区间(而不是整个堆), 从而减少一次 GC 所产生的停顿。

### 垃圾收集器

Serial 垃圾收集器(单线程、复制算法):使用复制算法的单线程的收集器,新生代垃圾收集器.在进行垃圾收集的同时,必须暂停其他所有的工作线程.

ParNew 垃圾收集器(Serial+多线程):使用复制算法的多线程收集器,在垃圾收集过程中同样也要暂停所有其他的工作线程。默认开启和 CPU 数目相同的线程数,通过-XX:ParallelGCThreads 参数来限制垃圾收集器的线程数。

Parallel Scavenge 收集器(多线程复制算法、高效)：使用复制算法,多线程的新生代垃圾收集器。关注的是程序达到一个可控制的吞吐量(运行用户代码时间/(运行用户代码时间+垃圾收集时间)),必须暂停其他所有的工作线程.

Serial Old 收集器(单线程标记整理算法 )：Serial 垃圾收集器年老代版本,它同样是个单线程的收集器,使用标记-整理算法。

Parallel Old 收集器(多线程标记整理算法)：Parallel Scavenge 的年老代版本,使用多线程的标记-整理算法。,在 JDK1.6
才开始提供。在 JDK1.6 之前,新生代使用 ParallelScavenge 收集器只能搭配年老代的 Serial Old 收集器,只能保证新生代的吞吐量优先,无法保证整体的吞吐量,Parallel Old 正是为了在年老代同样提供吞吐量优先的垃圾收集器,如果系统对吞吐量要求比较高,可以优先考虑新生代 Parallel Scavenge和年老代 Parallel Old 收集器的搭配策略。同样也必须暂停其他所有的工作线程.

CMS 收集器(多线程标记清除算法)：主要目标是获取最短垃圾回收停顿时间,和其他年老代使用标记-整理算法不同,它使用多线程的标记-清除算法。整个过程分为以下 4 个阶段:

1. 初始标记：标记一下 GC Roots 能直接关联的对象,速度很快,仍然需要暂停所有的工作线程。
2. 并发标记：进行 GC Roots 跟踪的过程,和用户线程一起工作,不需要暂停工作线程。
3. 重新标记：修正在并发标记期间,因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录,仍然需要暂停所有的工作线程。
4. 并发清除：清除 GC Roots 不可达对象,和用户线程一起工作。由于耗时最长的并发标记和并发清除过程中,垃圾收集线程可以和用户现在一起并发工作,所以总体上来看CMS 收集器的内存回收和用户线程是一起并发地执行。

G1 收集器：标记-整理算法。可以非常精确控制停顿时间,在不牺牲吞吐量前提下,实现低停顿垃圾回收。它把堆内存划分为大小固定的几个独立区域,并且跟踪这些区域的垃圾收集进度,同时在后台维护一个优先级列表,每次根据所允许的收集时间,优先回收垃圾最多的区域。在有限时间获得最高的垃圾收集效率。

2. 可以非常精确控制停顿时间,在不牺牲吞吐量前提下,实现低停顿垃圾回收。

## 类加载机制

### 双亲委托机制

- BootStrapClassLoader : 启 动 类 加 载 器 , 该 ClassLoader 是 jvm 在 启 动 时 创 建 的 , 用于加载 $JAVA_HOME$/jre/lib 下面的类库(或者通过参数-Xbootclasspath 指定)。由于启动类加载器涉及到虚拟机本地实现细节,开发者无法直接获取到启动类加载器的引用,所以不能直接通过引用进行操作。
- ExtClassLoader:扩展类加载器,该 ClassLoader 是在 sun.misc.Launcher 里作为一个内部类 ExtClassLoader定义的(即 sun.misc.Launcher$ExtClassLoader),ExtClassLoader 会加载 $JAVA_HOME/jre/lib/ext 下的类库(或者通过参数-Djava.ext.dirs 指定)。
- AppClassLoader:应用程序类加载器,该 ClassLoader 同样是在 sun.misc.Launcher 里作为一个内部类AppClassLoader 定义的(即 sun.misc.Launcher$AppClassLoader),AppClassLoader 会加载 java 环境变量CLASSPATH所 指 定 的 路 径下的类库, 而CLASSPATH所 指 定 的 路 径 可 以 通 过System.getProperty("java.class.path")获取;当然,该变量也可以覆盖,可以使用参数-cp,例如:java -cp 路径 (可以指定要执行的 class 目录)。
- CustomClassLoader:自定义类加载器,该 ClassLoader 是指自定义的ClassLoader,比如tomcat的StandardClassLoader 属于这一类。

```java
//ClassLoader 的 loadClass(String name, boolean resolve)的源码:
protected synchronized Class<?> loadClass(String name, boolean resolve) throws 																				ClassNotFoundException {
    // 首先找缓存是否有 class
    Class c = findLoadedClass(name);
    if (c == null) {
        //判断有没有父类
        try {
            if (parent != null) {
                //有的话,用父类递归获取 class
               　c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {}
        if (c == null) {
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

虽 然 ClassLoader 加 载 类 是 使 用 loadClass 方 法 , 但 是 鼓 励 用 ClassLoader的子类重写findClass(String),而不是重写 loadClass,这样就不会覆盖了类加载默认的双亲委派机制。

### 类初始化

JVM 类加载机制分为五个部分:加载,验证,准备,解析,初始化。

1. 加载：在内存中生成一个 java.lang.Class 对象。不一定非得要从一个 Class 文件获取,这里既可以从 ZIP 包中读取(比如从 jar 包和 war 包中读取),也可以在运行时计算生成(动态代理),也可以由其它文件生成(比如将 JSP 文件转换成对应的 Class 类)。

2. 验证：确保 Class 文件的字节流中包含的信息是否符合当前虚拟机的要求。

3. 准备：为类变量分配内存并设置类变量的初始值阶段,即在方法区中分配这些变量所使用的内存空间。

   ```java
   //v 在准备阶段过后的初始值为 0 而不是 8080,将 v 赋值为 8080 的指令是程序被编译后,存放于类构造器<client>方法之中。
   public static int v = 8080;
   //编译阶段会为 v 生成 ConstantValue 属性,在准备阶段虚拟机会根据 ConstantValue 属性将 v赋值为 8080.
   public static final int v = 8080;
   ```

   

4. 解析:虚拟机将常量池中的符号引用替换为直接引用.符号引用就是 class 文件中CONSTANT_Class_info,CONSTANT_Field_info,CONSTANT_Method_info等类型的常量。

5. 初始化:执行类构造器<client>方法的过程,<client>方法是由编译器自动收集类中的类变量的赋值操作和静态语句块中的语句合并而成的。虚拟机会保证子<client>方法执行之前,父类的<client>方法已经执行完毕,如果一个类中没有对静态变量赋值也没有静态语句块,那么编译器可以不为这个类生成<client>()方法。

以下几种情况不会执行类初始化:

1. 通过子类引用父类的静态字段,只会触发父类的初始化,而不会触发子类的初始化。
2. 定义对象数组,不会触发该类的初始化。
3. 通过xx.Class不会触发类的初始化。
4. 通过 Class.forName 加载指定类时,如果指定参数 initialize 为 false 时,也不会触发类初始化。
5. 通过 ClassLoader 默认的 loadClass 方法,也不会触发初始化动作。

1. 启动类加载器(Bootstrap ClassLoader)：负责加载 JAVA_HOME\lib 目录中的,或通过-Xbootclasspath 参数指定路径中的,且被虚拟机认可(按文件名识别,如 rt.jar)的类。
2. 扩展类加载器(Extension ClassLoader)：负责加载 JAVA_HOME\lib\ext 目录中的,或通过 java.ext.dirs 系统变量指定路径中的类库。
3. 应用程序类加载器(Application ClassLoader):负责加载用户路径(classpath)上的类库。JVM 通过双亲委派模型进行类的加载,当然我们也可以通过继承 java.lang.ClassLoader实现自定义的类加载器。

