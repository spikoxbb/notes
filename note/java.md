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

