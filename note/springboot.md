# 1.maven starter包

AOP 包：spring-boot-starter-aop

Web 开发包,将载入 Spring MVC 所需要的包,且内嵌 tomcat：spring-boot-starter-web

载测试依赖包:spring-boot-starter-test

# 2.IOC

所以IOC容器都实现了BeanFactory接口。

```java
public interface BeanFactory {
//前缀
String FACTORY BEAN PREFIX=" &” ;
//多个 getBean 方法
Object getBean(String name) throws BeansException;
<T> T getBean(String name,Class<T> requiredType) throwsBeansException;
<T> T getBean(Class<T> requiredType) throws BeansException;
Object getBean(String name, Object . . args) throws BeansException ;
<T> T getBean( Class<T> requiredType , Object ... args) throws BeansException;
//是否包含 Bean
boolean containsBean (String name);
// Bean 是否单例
boolean isSingleton(String name) throws NoSuchBeanDefinitionException ;
// Bean 是否原型
boolean isPrototype(String name) throws NoSuchBeanDefinitionException ;
//是否类型匹配
boolean isTypeMatch(String name , ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
boolean isTypeMatch(String name , Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
//获取 Bean 的类型
Class<?> getType(String name) throws NoSuchBeanDefinitionException ;
//获取 Bean 的别名
String[] getAliases(String name);
}
```

ApplicationContext 接口通过继承HierarchicaBeanFactory接口,进而继承 BeanFactory 接口 ,还扩展了消息国际化接口( MessageSource )、环境可配置接口 ( EnvironmentCapable )、应用事件发布接口( ApplicationEventPublisher ) 和 资源模式解析接口( ResourcePatternResolver ).

@Component 是标明哪个类被扫描进入 Spring IoC 容器,而@ComponentScan是标明采用何种策略去扫描装配 Bean(只会扫描Config类所在的当前包和子包).

@ SpringBootApplication 也注入了@ComponentScan。

@Autowired 提供这样的规则, 首先它会根据类型找到对应的 Bean,如果对应类型的 Bean不是唯 一 的,那么它会根据其属性名称和 Bean 的名称进行匹配。如果匹配得上,就会使用该 Bean ；如果还无法匹配,就会抛出异常。@Quelifier与@Autow ired 组合在一起,通过类型和名称一起找到 Bean 。

```java
<T> T getBean(String name , Class<T> requiredType) throws BeansException；
```

在参数上前加@Autowired也能注入进来。

Bean 的生命周期：

1. Bean定义(默认Spring会继续实例化Bean和依赖注入,但lazyInit==true时只有取出来的时候才做初始化和依赖注入等操作)
   1. Spring 通过配置,如 @ComponentScan 定义的扫描路径去找到带有@Component 的类 ,这个过程就是一个资源定位的过程 。
   2. 开始解析,并且将定义的信息保存起来 （保存到BeanDefinition实例中）。
   3. 把 Bean 定义发布到 Spring IoC 容器中 。

2. Bean的初始化

   ![](../img/微信图片_20190819111122.png)

3. Bean的生存期

4. Bean的销毁 

(@ConfigurationProperties ( ” database ” ))@ConfigurationProperties 中 配置的字符 串 database ,将与 POJO 的属性名称组成属性的全限定名去配置文件里查找 ,这样就 能将对应的属性读入到 POJO当中 。

使用@PropertySource 去定义对应的属性文件.

spring.profiles.active 和 spring .profiles.default 都没有 配置 的 情 况下 , 被 @Profile 标注 的 Bean 将不会被 Spring 装配到 IOC 容器 中 。

把选项-Dspring.profil es.active 配置的值记为{profile },则它 会用 application- {profile} . properties 文件去代替原来默认的 application.properties 文件,如JAVA_OPTS=''-Dspring . profiles.active=dev "。

装配 XML 定义的Bean：

```java
@ImportResource (value = {'’ classpath: spring-other . xml"})
```

@Value 中的${.. . .. . }代表占位符,它会读取上下文的属性值装配到属性中，#{..... }代表启用 Spring EL表达式 ,T( ..... )代表的是引入类。Java 默认加载 的包外需要写出全限定名才能引用类。

```
Value (”# { T (System) . currentTimeMillis () }”)
private Long initTime = null;
//赋值字符串
@Value ( ” 们 ’ 使用 Spring EL 赋值字符串 ’ } ” )
private String str = null;
//科学计数法赋值
@Value( " #(9 . 3E3 } ” )
private double d ;
//赋值浮点数
@Value ( ” # ( 3 .14 } ” )
private float pi ;
//还可以获取其他 Spring Bean 的 属性来给当 前的 Bean 属性赋值
@Value ( ” #( beanName . str } ” )
private String otherBeanProp=null ;
```

# 3. AOP

```java
public class Myinterceptor implements Interceptor {
    @Override
    public boolean before() {......}
    @Override
    public boolean useAround() {......}
    @Override
    public boolean after() {......}
    @Override
    public Object around(Invocation invocaiton) {
            ......
            Object obj = invocation.proceed();
            .......
            return obj;
    }
    @Override
    public boolean afterReturning() {......}
     @Override
    public boolean afterThrowing() {......}
}

HelloService helloService =new HelloServiceimpl();
HelloService proxy = (HelloService) ProxyBean.getProxyBean(helloServ1ce , new Myinterceptor()) ;

public class ProxyBean implements InvocationHandler {
    private Object target = null ;
    private Interceptor interceptor = null;
    public static Object getProxyBean(Object target , Interceptor interceptor){
        ProxyBean proxyBean =new ProxyBean() ;
        proxyBean.target = target ;
        proxyBean.interceptor =interceptor ;
        Object proxy = Proxy.newProxyinstance (target.getClass().getClassLoader(),target.getClass().getInterfaces(),proxyBean);
        return proxy ;
    }
    @Override
    public Object invoke(Object proxy , Method method , Object [] args){
        boolean exceptionFlag = false ;
        Invocation invocation = new Invocation(target , method , args) ;
        Object retObj = null ;
        try {
            if(this.interceptor. before () ) {
                retObj = this .interceptor.around(invocation);
            } else {
                retObj = method.invoke(target, args );
            } catch (Exception ex ) {
                exceptionFlag = true ;
                this . interceptor.after( );
                if (exceptionFlag ) {
                    this.interceptor . afterThrowing( ) ;
                } else {
                    this.interce pto r.afterReturning();
                    return retObj ;
                }
            }
        }
    }
}
```

Spring 是以@Aspect 作为切面声明的。@Pointcut 来定义切点。

```java
execution(*com.springboot.chapter4.aspect.service.impl.UserServceimpl .printUser ( . ) )
```

- execution 表示在执行的时候 ,拦截 里面的正 则匹配的方法.
  *表示任意返回类型的方法.
  com.spr ingboot.chapter4 .aspect.service. impl. U serServ icelmpl 指 定目标对象的全限定名称.
   printUser 指定目标对象的方法 .
   (.)表示任意参数进行匹配。

@DeclareParents , 它的作用是引入新的类来增强服务:

1. value :指向你要增强功能的目标对象 

2. defaultlmpl : 引入增强 功能 的 类

   ```java
   @Aspect
   public class MyAspect {
   @DeclareParents(
   value= "com.springboot.chapter4.aspect.service.impl.UserServiceImpl +”,
   defaultimpl=UserValidatorImpl.class)
   public UserValidator userValidator;
     ---------------------------------------------  
   UserValidator userValidator=(UserValidator)userService ;
   //验证用户是否为空
   if (userValidator.validate(user) ) {
   userService.printUser(user) ;
   }
   return user ;
   ```

   传递参数给通知:

   ```java
   @Before ( ” pointCut () && args(user)” )
   public roid beforeParam (JoinPoint point ,User user){
       Object[] args = point.getArgs ( ) ;
   }
   //将连接点(目标对象方法)名称为 user的参数传递进来对于非环绕通知而言, SpringAOP会自动把JoinPoint传递到通知中:对于环绕通知而言,可以使用 ProceedingJoinPoint(进行目标对象的回调)
   ```

   使用多个切面拦截时，切面的执行顺序是混乱的，可使用@Order或接口Ordered：

   ```java
   @Aspect
   @Order (1)
   public class MyAspectl {
       ......
   }
   
   @Aspect
   public class MyAspectl implements Ordered {
       @Override
       public int getOrder (){
           return 1 ;
       }
   }
   ```

   # 数据库

```java
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>2.0.0</version>
</dependency>
```

@Alias指定别名。(mapper.xml中使用)

枚举是可以通过 typeHandler 进行转换.抽象类 BaseTyp空Handler<T>实现了 TypeHandler<T>.

```java
//声明jdbcType
@MappedJdbcTypes (value=JdbcType.Integer )
//声明JavaType
@MappedTypes (value=SexEnum.class )
public class SexTypeHandler extends BaseTypeHandler<SexEnum>{
    ......
}
```

便用 MapperFactoryBean 装配 MyBatis 接口：

```java
@Autowired
SqlSessionFactory sqlSessionFactory = null;
@Bean
public MapperFactoryBean<MyBatisUserDao> initMyBatisUserDao () {
    MapperFactoryBean<MyBatisUserDao> bean =new MapperFactoryBean<>();
    bean.setMapperinterface(MyBatisUserDao . class) ;
    bean.setSqlSessionFactory(sqlSessionFactory);
    return bean;
}
```

# 事务

@Transactional可以标注在类或者方法上，标注在类上时,代表这个类所有公共( pub lic )非静态的方法都将启用事务功能。

Spring IoC 容器在加载时会配置信息解析出来,然后把这些信息存到事务定义器( TransactionDefinition 接口的实现类〉
里 , 并且记录哪些类或者方法需要启动事务功能,采取什么策 略去执行事务。

```java
@Target({E lernentType . METHOD , ElernentType . TYPE})
@Retention(RetentionPolicy . RUNTIME)
@Inherited
@Documented
public @interface Transactional {
//通过 bean name 指定事务管理器
@AliasFor (” transactionManager ” )
String value() default  "";
//同 value 属性
@AliasFor ( ” value ” )
String transactionManager () default "";
//指定传播行为
Propagation propagation() default Propagation.REQUIRED ;
//指定隔离级别
Isolation isolation () default Isolation.DEFAULT;
//指定超时时间(单位秒)
int timeout() default TransactionDefinition.TIMEOUT_DEFAULT ;
//是否只读事务
boolean readOnly() default false ;
//方法在发生指定异常时回  ,默认是所有异常都回滚
Class< ? extends Throwable > [] roll back For( ) default {);
//方法在发生指定异常名称时回滚 , 默认是所有异常都回滚
String[] rollbackForClassNarne() default {) ;
//方法在发生指定异常时不回滚 , 默认是所有异常都回滚
Class< ? extends Throwable> [] noRollbackFor () default {} ;
//方法在发生指定异常名称时不回滚,默认是所有异常都回滚
String[] noRollbackForClassNarne() default {} ;
}
```

事务管理器 的顶层接口为 PlatformTransactionManager,(MyBatis 用到的事务管理器是 DataSourceTransactionManager)

```java
public interface PlatforrnTransactionManager {
    //获取事务 , 它还会设置数据属性
    TransactionStatus getTransaction(TransactionDefinition definition)throws TransactionException ;
    //提交事务
    void commit(TransactionStatus status) throws TransactionException ;
    //回滚事务
    void rollback(TransactionStatus status) throws TransactionException; 
}
```

ACID:Atomic (原子性);Consistency (一致性);Isolation (隔离性);Durability (持久性)

第一类丢失更新:一个事务回滚另一个事务提交而引发的数据不一致的情况.

第二类丢失更新:多个事务都提交引发的丢失更新.

1. 未提交读：允许一个事务读取另外一个事务没有提交的数据

2. 读写提交：一个事务只能读取另外 一 个事务已经提交的数据 ,

3. 可重复读：克服读写提交中出现的不可重复读的现象，直至事务 1 提交,事务 2 才能读取库存的值。

   幻读不是针对 一条数据库记录而言,而是多条记录，可重复读是针对 数据库 的单 一 条记录

4. 串行化：所有的 SQL 都会按照顺序执行

Oracle 默认的隔离级别为读写提交, MySQL 则 是 可重复读。

当前方法调用子方法的时候,让每一 个子方法 不在当前事务中执行,而是创建一个新的事务去执行子方法,我们就说 当前方法调用子方法的传播行为为新建事务。

```java
public enum Propagation {
//需要事务,它是默认传播行为,如果当前存在事务,就沿用当前事务,否则新建一个事务运行子方
    REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
//支持事务,如果当前存在事务,就沿用当前事务 ,如果不存在 ,则继续采用无事务的方式运行子方法
    SUPPORTS(TransactIonDefinition.PROPAGATION_SUPPORTS) ,
//必须使用事务,如果当前没有事务,则会抛出异常,如果存在当前事务 , 就沿用当前事务
    MANDATORY ( TransactionDefinition.PROPAGATION_MANDATORY) ,
//无论当前事务是否存在,都会创建新事务运行方法 ,这样新事务就可以拥有新的锁和隔离级别等特性,与当前事务相互独立
    REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
//不支持事务,当前存在事务时,将挂起事务,运行方法
    NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED ),
//不支持事务,如果当前方法存在事务,则抛出异常,否则继续使用无事务机制运行
    NEVER(TransactionDefinition.PROPAGATION_NEVER),
//在当前方法调用子方法时,如果子方法发生异常,女只因滚子方法执行过的 SQL ,而不回滚当前方法的事务
    NESTED(TransactionDefinition.PROPAGATION_NESTED);
private final int value ;
Propagation (int value){
    this value = value ;
}
public int value(){
    return this.value ;
}
)
```

在自调用的过程中 , 是类自身的调用 ,而 不是代理对象去调 用, 那么就不会产生 AOP , 这样事务就失效了。

# Spring MVC

1. 客户端请求提交到 DispatcherServlet。
2. 由 DispatcherServlet 控制器查询一个或多个 HandlerMapping,找到处理请求的Controller。
3. DispatcherServlet 将请求提交到 Controller。
4. Controller 调用业务逻辑处理后,返回 ModelAndView。
5. DispatcherServlet 查询一个或多个 ViewResoler 视图解析器,找到 ModelAndView 指定的视图。
6. 视图负责将结果显示到客户端。

# mybatis

Mybatis 的一级缓存原理( sqlsession 级别 )：第一次发出一个查询 sql,sql 查询结果写入 sqlsession 的一级缓存中,缓存使用的数据结构是一个 map。

- key:MapperID+offset+limit+Sql+所有的入参
- value:用户信息。同一个 sqlsession 再次发出相同的 sql,就从缓存中取出数据。如果两次中间出现 commit 操作(修改、添加、删除),本 sqlsession 中的一级缓存区域全部清空,下次再去缓存中查询不到所以要从数据库查询,从数据库查询到再写入缓存。

二级缓存原理( mapper 基本 )：二级缓存的范围是 mapper 级别(mapper 同一个命名空间),mapper 以命名空间为单位创建缓存数据结构,结构是 map。mybatis 的二级缓存是通过 CacheExecutor 实现的。CacheExecutor其实是 Executor 的代理对象。所有的查询操作,在 CacheExecutor 中都会先匹配缓存中是否存在,不存在则查询数据库。
key:MapperID+offset+limit+Sql+所有的入参

具体使用需要配置:
1. Mybatis 全局配置中启用二级缓存配置
2. 在对应的 Mapper.xml 中配置 cache 节点
3. 在对应的 select 查询节点中添加 useCache=true