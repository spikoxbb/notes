[TOC]

# Spring整体架构

## Core Container层

- Core模块

  主要包括Spring框架的基本核心**工具类**，Spring其他组件均要用到，是其他组件的基本核心。

- Beans模块

  包含访问配置文件，创建和管理Bean，进行IOC/DI操作相关的类。

- Context模块

  构建与Core和Beans模块基础之上，提供了一种类似JNDI注册器的对象访问方法。Context模块继承了Beans的特性，提供了扩展，添加了国际化，事件传播（发布和接收事件），资源加载。同时也支持EJB，JMX和基础的远程处理。**该模块的关键是ApplicationContext接口**。

  > JNDI——Java Naming and Directory Interface（JAVA命名和目录接口）。
  >
  > - NDI中的命名（Naming），就是将Java对象以某个名称的形式绑定（binding）到一个容器环境（Context）中，以后调用容器环境（Context）的查找（lookup）方法又可以查找出某个名称所绑定的Java对象。
  >
  > - JNDI中的目录（Directory）是指将一个对象的所有属性信息保存到一个容器环境中。JNDI API中提供的代表目录容器环境的类为DirContext，DirContext是Context的子类，显然它除了能完成目录相关的操作外，也能完成所有的命名（Naming）操作。
  >
  > 一个DirContext容器环境中也可以只绑定对象的属性信息，而不绑定任何对象自身。

- Expression Language模块

  提供了表达式语言，用于在运行时查询和操作对象。支持设置和获取属性的值等。

## Data Access/Intergation层

- JDBC模块

  提供了JDBC抽象层。该模块包含了对JDBC数据访问进行封装的所有类。

- ORM模块

  为流行的对象-关系映射API（JPA，Hibernate等）提供了一个交互层。

- OXM模块

  提供了对Object/XML映射实现的抽象层。

- JMS模块

  主要包括制造和消费信息的特征。

- Transaction模块

  支持编程和声明式的事务管理。

## Web层

- Web模块

  提供了基础的Web特性。如多文件上传，使用servlet Listens初始化IOC容器以及一个面向Web的应用上下文。

- Web-Servlet

  包含了MVC实现。

- Web-Struts

  提供了对Struts的实现，Spring3.0中已被弃用。

- Web-porlet

  包含了用于Portlet环境的MVC实现。

## AOP层

- AOP模块

  提供了符合AOP联盟标准的面向切面编程的实现，从而将逻辑代码分开降低耦合性。

- Aspects模块

  提供了对AspectJ的集成支持。

- Instrumentation模块

  类定义动态改变和操作。程序运行时，通过 -javaagent 参数指定一个特定的 jar 文件来启动  Instrumentation 的代理程序。可构建一个独立于应用程序的代理程序（Agent）,监测和协助运行在JVM 上的程序，可以实现一种虚拟机级别的AOP实现。执行其实现的premain方法，而且这一步是在JVM启动执行main方法之前执行的。

> **Weaving**：
>
> **Weaving：**
>
> AspectJ使用了三种不同类型的织入：
>
> 1. 编译时织入：AspectJ编译器同时加载我们切面的源代码和我们的应用程序，并生成一个织入后的类文件作为输出。
> 2. 编译后织入：这就是所熟悉的二进制织入。它被用来编织现有的类文件和JAR文件与我们的切面。
> 3. 加载时织入：这和之前的二进制编织完全一样，所不同的是织入会被延后，直到类加载器将类加载到JVM。
>
> AspectJ使用的是编译期和类加载时进行织入，Spring AOP利用的是运行时织入。
>
> 运行时织入，在使用目标对象的代理执行应用程序时，编译这些切面（使用JDK动态代理或者CGLIB代理）。
>
> -----
>
> **Joinpoints**
>
> Spring AOP需要目标类的子类这也也伴随着局限性，**不能跨越final类来应用横切关注点（或切面），因为它们不能被覆盖，从而导致运行时异常。同样地，也不能应用于静态和final的方法。由于不能覆写，Spring的切面不能应用于他们。**因此，Spring AOP由于这些限制，只支持执行方法的连接点。然而，AspectJ在运行前将横切关注点直接织入实际的代码中。 与Spring AOP不同，它不需要继承目标对象，因此也支持其他许多连接点。
>
> ------
>
> **Simplicity**：
>
> Spring AOP更简单，因为它没有引入任何额外的编译期或在编译期织入。它使用了运行期织入的方式，Spring AOP只作用于Spring管理的beans 。然而AspectJ需要引入AJC编译器，重新打包所有库（除非我们切换到编译后或加载时织入）。这种方式相对于前一种，更加复杂，因为它引入了我们需要与IDE或构建工具集成的AspectJ Java工具（包括编译器（ajc），调试器（ajdb），文档生成器（ajdoc），程序结构浏览器（ajbrowser））。
>
> ------
>
> **Performance**：
>
> 考虑到性能问题，编译时织入比运行时织入快很多。Spring AOP是基于代理的框架，因此应用运行时会有目标类的代理对象生成。另外，每个切面还有一些方法调用，这会对性能造成影响。AspectJ不同于Spring AOP，是在应用执行前织入切面到代码中，没有额外的运行时开销。AspectJ经过测试大概8到35倍快于Spring AOP。

## Test

Test模块支持使用JUnit和TestNG来测试。

