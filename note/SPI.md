SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，可以很容易的通过 SPI 机制为程序提供拓展功能。 Dubbo 就是通过 SPI 机制加载所有的组件。不过，Dubbo 并未使用 Java 原生的 SPI 机制，而是对其进行了增强，使其能够更好的满足需求。

在jar包的META-INF/services目录下创建一个以“接口全限定名”为命名的文件，内容为实现类的全限定名；接口实现类所在的jar包放在主程序的classpath中；主程序通过java.util.ServiceLoder动态装载实现模块，它通过扫描META-INF/services目录下的配置文件找到实现类的全限定名，把类加载到JVM；SPI的实现类必须携带一个不带参数的构造方法；

```java
public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<IShout> shouts = ServiceLoader.load(IShout.class);
        for (IShout s : shouts) {
            s.shout();
        }
    }
}
```

```java
public final class ServiceLoader<S> implements Iterable<S>{
private static final String PREFIX = "META-INF/services/";

    // 代表被加载的类或者接口
    private final Class<S> service;

    // 用于定位，加载和实例化providers的类加载器
    private final ClassLoader loader;

    // 创建ServiceLoader时采用的访问控制上下文
    private final AccessControlContext acc;

    // 缓存providers，按实例化的顺序排列
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 懒查找迭代器
    private LazyIterator lookupIterator;
  
    ......
}
```

实现的流程如下：

- 1 应用程序调用ServiceLoader.load方法
  ServiceLoader.load方法内先创建一个新的ServiceLoader，并实例化该类中的成员变量，包括：

  - loader(ClassLoader类型，类加载器)
  - acc(AccessControlContext类型，访问控制器)
  - providers(LinkedHashMap<String,S>类型，用于缓存加载成功的类)
  - lookupIterator(实现迭代器功能)

- 2 应用程序通过迭代器接口获取对象实例
  ServiceLoader先判断成员变量providers对象中(LinkedHashMap<String,S>类型)是否有缓存实例对象，如果有缓存，直接返回。
  如果没有缓存，执行类的装载，实现如下：

  - (1) 读取META-INF/services/下的配置文件，获得所有能被实例化的类的名称，值得注意的是，ServiceLoader**可以跨越jar包获取META-INF下的配置文件**，具体加载配置的实现代码如下：

  ```jsx
          try {
              String fullName = PREFIX + service.getName();
              if (loader == null)
                  configs = ClassLoader.getSystemResources(fullName);
              else
                  configs = loader.getResources(fullName);
          } catch (IOException x) {
              fail(service, "Error locating configuration files", x);
          }
  ```

  - (2) 通过反射方法Class.forName()加载类对象，并用instance()方法将类实例化。
  - (3) 把实例化后的类缓存到providers对象中，(LinkedHashMap<String,S>类型）
    然后返回实例对象。