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
