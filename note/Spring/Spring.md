[TOC]

# **Spring IOC** **原理**

Spring 通过一个配置文件描述 Bean 及 Bean 之间的依赖关系，利用 Java 语言的反射功能实例化Bean 并建立 Bean 之间的依赖关系。 Spring 的 IoC 容器在完成这些底层工作的基础上，还提供了 Bean 实例缓存、生命周期管理、 Bean 实例代理、事件发布、资源装载等高级服务。

## **Spring** **容器高层视图**

Spring 启动时读取应用程序提供的 Bean 配置信息，并在 Spring 容器中生成一份相应的 Bean 配置注册表，然后根据这张注册表实例化 Bean，装配好 Bean 之间的依赖关系，为上层应用提供准备就绪的运行环境。其中 Bean 缓存池为 HashMap 实现。

## **IOC** **容器实现**

BeanFactory 是 Spring 框架的基础设施，面向 Spring 本身。ApplicationContext 面向使用Spring 框架的开发者，几乎所有的应用场合我们都直接使用 ApplicationContext 而非底层的 BeanFactory。

- *BeanDefinitionRegistry* 注册表

  Spring 配置文件中每一个节点元素在 Spring 容器里都通过一个 BeanDefinition 对象表示，它描述了 Bean 的配置信息。

  而 BeanDefinitionRegistry 接口提供了向容器手工注册BeanDefinition 对象的方法。

- *BeanFactory* *顶层接口*

  位于类结构树的顶端 ，它最主要的方法就是 getBean(String beanName)，该方法从容器中返回特定名称的 Bean，BeanFactory 的功能通过其他的接口得到不断扩展。

- *ListableBeanFactory*

  该接口定义了访问容器中 Bean 基本信息的若干方法，如查看 Bean 的个数、获取某一类型Bean 的配置名、查看容器中是否包括某一 Bean 等方法。

- *HierarchicalBeanFactory* *父子级联*

  - 父子级联 IoC 容器的接口，子容器可以通过接口方法访问父容器。
  - 通过HierarchicalBeanFactory 接口， Spring 的 IoC 容器可以建立父子层级关联的容器体系，子容器可以访问父容器中的 Bean，但父容器不能访问子容器的 Bean。
  - Spring 使用父子容器实现了很多功能，比如在 Spring MVC 中，展现层 Bean 位于一个子容器中，而业务层和持久层的 Bean 位于父容器中。
  - 这样，展现层 Bean 就可以引用业务层和持久层的 Bean，而业务层和持久层的 Bean 则看不到展现层的 Bean。

- *ConfigurableBeanFactory*

  - 是一个重要的接口，增强了 IoC 容器的可定制性，它定义了设置类装载器、属性编辑器、容器初始化后置处理器等方法；

- *AutowireCapableBeanFactory* *自动装配*

  定义了将容器中的 Bean 按某种规则（如按名字匹配、按类型匹配等）进行自动装配的方法。

- *SingletonBeanRegistry* *运行期间注册单例* *Bean*

  - 定义了允许在运行期间向容器注册单实例 Bean 的方法。
  - 对于单实例（ singleton）的 Bean 来说，BeanFactory 会缓存 Bean 实例，所以第二次使用 getBean() 获取 Bean 时将直接从IoC 容器的缓存中获取 Bean 实例。
  - Spring 在 DefaultSingletonBeanRegistry 类中提供了一个用于缓存单实例 Bean 的缓存器，它是一个用 HashMap 实现的缓存器，单实例的 Bean 以beanName 为键保存在这个 HashMap 中。

- *依赖日志框框*

  在初始化 BeanFactory 时，必须为其提供一种日志框架，比如使用 Log4J， 即在类路径下提供 Log4J 配置文件，这样启动 Spring 容器才不会报错。

### **ApplicationContext** 面向开发应用

ApplicationContext 由 BeanFactory 派 生 而 来 ， 提 供 了 更 多 面 向 实 际 应 用 的 功 能 。

ApplicationContext 继承了 HierarchicalBeanFactory 和 ListableBeanFactory 接口，在此基础上，还通过多个其他的接口扩展了 BeanFactory 的功能：

1. ClassPathXmlApplicationContext：默认从类路径加载配置文件。
2. FileSystemXmlApplicationContext：默认从文件系统中装载配置文件。
3. ApplicationEventPublisher：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。
4. MessageSource：为应用提供 i18n 国际化消息访问的功能。
5. ResourcePatternResolver ： 所 有 ApplicationContext 实现类都实现了类似于PathMatchingResourcePatternResolver 的功能，可以通过带前缀的 Ant 风格的资源文件路径装载 Spring 的配置文件。
6. LifeCycle：该接口是 Spring 2.0 加入的，该接口提供了 start()和 stop()两个方法，主要用于控制异步处理过程。
   - 在具体使用时，该接口同时被 ApplicationContext 实现及具体Bean 实现， ApplicationContext 会将 start/stop 的信息传递给容器中所有实现了该接口的 Bean，以达到管理和控制 JMX、任务调度等目的。
7. ConfigurableApplicationContext 扩展于 ApplicationContext，它新增加了两个主要的方法： refresh()和 close()，让 ApplicationContext 具有启动、刷新和关闭应用上下文的能力。
   - 在应用上下文关闭的情况下调用 refresh()即可启动应用上下文，在已经启动的状态下，调用 refresh()则清除缓存并重新装载配置信息，而调用 close()则可关闭应用上下文。

# Spring Bean 作用域

## singleton 单例模式

singleton：单例模式，Spring IoC 容器中只会存在一个共享的 Bean 实例，无论有多少个Bean 引用它，始终指向同一对象。该模式在多线程下是不安全的。Singleton 作用域是Spring 中的缺省作用域，也可以显示的将 Bean 定义为 singleton 模式，配置为：

<bean id="userDao" class="com.ioc.UserDaoImpl" scope="singleton"/>

## prototype 原型模式每次使用时创建

prototype:原型模式，每次通过 Spring 容器获取 prototype 定义的 bean 时，容器都将创建一个新的 Bean 实例，每个 Bean 实例都有自己的属性和状态，而 singleton 全局只有一个对象。

根据经验，对有状态的bean使用prototype作用域，而对无状态的bean使singleton作用域。 

## Request：一次 **request** 一个实例

request：在一次 Http 请求中，容器会返回该 Bean 的同一实例。

而对不同的 Http 请求则会产生新的 Bean，而且该 bean 仅在当前 Http Request 内有效,当前 Http 请求结束，该 bean实例也将会被销毁。

<bean id="loginAction" class="com.cnblogs.Login" scope="request"/>

## session

session：在一次 Http Session 中，容器会返回该 Bean 的同一实例。而对不同的 Session 请求则会创建新的实例，该 bean 实例仅在当前 Session 内有效。

同 Http 请求相同，每一次session 请求创建新的实例，而不同的实例之间不共享属性，且实例仅在自己的 session 请求内有效，请求结束，则实例将被销毁。

<bean id="userPreference" class="com.ioc.UserPreference" scope="session"/>

## global Session

global Session：在一个全局的 Http Session 中，容器会返回该 Bean 的同一个实例，仅在使用 portlet context 时有效。

# **Spring Bean** 生命周期

1. **实例化**

   实例化一个 Bean，也就是我们常说的 new。

2. **IOC** **依赖注入**

   按照 Spring 上下文对实例化的 Bean 进行配置，也就是 IOC 注入。

3. **setBeanName** **实现**

   如果这个 Bean 已经实现了 BeanNameAware 接口，会调用它实现的 setBeanName(String)方法，此处传递的就是 Spring 配置文件中 Bean 的 id 值。

4. **BeanFactoryAware** **实现**

   如果这个 Bean 已经实现了 BeanFactoryAware 接口，会调用它实现的 setBeanFactory，setBeanFactory(BeanFactory)传递的是 Spring 工厂自身（可以用这个方式来获取其它 Bean，只需在 Spring 配置文件中配置一个普通的 Bean 就可以）。

5. **ApplicationContextAware** **实现**

   如果这个 Bean 已经实现了 ApplicationContextAware 接口，会调用setApplicationContext(ApplicationContext)方法，传入 Spring 上下文（同样这个方式也可以实现步骤 4 的内容，但比 4 更好，因为 ApplicationContext 是 BeanFactory 的子接口，有更多的实现方法）

6. **postProcessBeforeInitialization** 接口实现-**初始化预处理**

   - 如果这个 Bean 关联了 BeanPostProcessor 接口，将会调用postProcessBeforeInitialization(Object obj, String s)方法，BeanPostProcessor 经常被用作是 Bean 内容的更改，并且由于这个**是在 Bean 初始化结束时调用那个的方法，也可以被应用于内存或缓存技术。**

7. **init-method**

   如果 Bean 在 Spring 配置文件中配置了 init-method 属性会自动调用其配置的初始化方法。

8. **postProcessAfterInitialization**

   - 如果这个 Bean 关联了 BeanPostProcessor 接口，将会调用postProcessAfterInitialization(Object obj, String s)方法。
   - 以上工作完成以后就可以应用这个 Bean 了，那这个 Bean 是一个 Singleton 的，所以一般情况下我们调用同一个 id 的 Bean 会是在内容地址相同的实例，当然在 Spring 配置文件中也可以配置非 Singleton。

9. **Destroy** **过期自动清理阶段**

   当 Bean 不再需要时，会经过清理阶段，如果 Bean 实现了 DisposableBean 这个接口，会调用那个其实现的 destroy()方法；

10. **destroy-method** **自配置清理**

    最后，如果这个 Bean 的 Spring 配置中配置了 destroy-method 属性，会自动调用其配置的销毁方法。

# **Spring** **依赖注入四种方式**

- **构造器注入**

  ```java
  /*带参数，方便利用构造器进行注入*/ 
  public CatDaoImpl(String message){ 
    this. message = message; 
  } 
  <bean id="CatDaoImpl" class="com.CatDaoImpl"> 
  <constructor-arg value=" message "></constructor-arg> 
  </bean>
  ```

- **setter** **方法注入**

  ```java
  public class Id { 
    private int id; 
    public int getId() { return id; } 
    public void setId(int id) { this.id = id; } 
  } 
  <bean id="id" class="com.id "> <property name="id" value="123"></property> </bean>
  ```

- **静态工厂注入**

  静态工厂顾名思义，就是通过调用静态工厂的方法来获取自己需要的对象，为了让 spring 管理所有对象，我们不能直接通过"工程类.静态方法()"来获取对象，而是依然通过 spring 注入的形式获取。

- **实例工厂**

  实例工厂的意思是获取对象实例的方法不是静态的，所以你需要首先 new 工厂类，再调用普通的实例方法。

# **5** **种不同方式的自动装配**

Spring 装配包括手动装配和自动装配，手动装配是有基于 xml 装配、构造方法、setter 方法等自动装配有五种自动装配的方式，可以用来指导 Spring 容器用自动装配方式来进行依赖注入。

- no：默认的方式是不进行自动装配，通过显式设置 ref 属性来进行装配。
- byName：通过参数名 自动装配，Spring 容器在配置文件中发现 bean 的 autowire 属性被设置成 byname，之后容器试图匹配、装配和该 bean 的属性具有相同名字的 bean。
- byType：通过参数类型自动装配，Spring 容器在配置文件中发现 bean 的 autowire 属性被设置成 byType，之后容器试图匹配、装配和该 bean 的属性具有相同类型的 bean。如果有多个 bean 符合条件，则抛出错误。
- constructor：这个方式类似于 byType， 但是要提供给构造器参数，如果没有确定的带参数的构造器参数类型，将会抛出异常。
- autodetect：首先尝试使用 constructor 来自动装配，如果无法工作，则使用 byType 方式。

# **AOP** **两种代理方式**

Spring 提供了两种方式来生成代理对象: JDKProxy 和 Cglib，具体使用哪种方式生成由AopProxyFactory 根据 AdvisedSupport 对象的配置来决定。

默认的策略是如果目标类是接口，则使用 JDK 动态代理技术，否则使用 Cglib 来生成代理。

## **JDK** **动态接口代理**

JDK 动态代理主要涉及到 java.lang.reflect 包中的两个类：Proxy 和 InvocationHandler。InvocationHandler是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编制在一起。Proxy 利用 InvocationHandler 动态创建一个符合某一接口的实例，生成目标类的代理对象。

## **CGLib** **动态代理**

CGLib 全称为 Code Generation Library，是一个强大的高性能，高质量的代码生成类库，可以在运行期扩展 Java 类与实现 Java 接口，CGLib 封装了 asm，可以再运行期动态生成新的 class。

和 JDK 动态代理相比较：JDK 创建代理有一个限制，就是只能为接口创建代理实例，而对于没有通过接口定义业务方法的类，则可以通过 CGLib 创建动态代理。

# **Spring MVC** **原理**

Spring 的模型-视图-控制器（MVC）框架是围绕一个 DispatcherServlet 来设计的，这个 Servlet会把请求分发给各个处理器，并支持可配置的处理器映射、视图渲染、本地化、时区与主题渲染等，甚至还能支持文件上传。

1. **Http** **请求到** **DispatcherServlet**。

    客户端请求提交到 DispatcherServlet。

2. **HandlerMapping** **寻找处理器**

   由 DispatcherServlet 控制器查询一个或多个 HandlerMapping，找到处理请求的Controller。

3. **调用处理器** **Controller**

   - DispatcherServlet 将请求提交到 Controller。

4. **Controller调用业务逻辑处理后，返回 ModelAndView**。

   调用业务处理和返回结果：Controller 调用业务逻辑处理后，返回 ModelAndView。

5. **DispatcherServlet** **查询** **ModelAndView**

   处理视图映射并返回模型： DispatcherServlet 查询一个或多个 ViewResoler 视图解析器，找到 ModelAndView 指定的视图。

6. **ModelAndView** **反馈浏览器** **HTTP**

    Http 响应：视图负责将结果显示到客户端。



