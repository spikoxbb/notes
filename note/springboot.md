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

   ![](/home/spiko/notes/img/微信图片_20190819111122.png)

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

@DeclareP缸ents , 它的作用是引入新的类来增强服务:

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
   public class MyAAspectl {
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

   