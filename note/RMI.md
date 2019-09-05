### RMI

方法调用从客户对象经占位程序（Stub)、远程引用层(Remote Reference Layer)和传输层（Transport Layer）向下，传递给主机，然后再次经传 输层，向上穿过远程调用层和骨干网（Skeleton），到达服务器对象。 占位程序扮演着远程服务器对象的代理的角色，使该对象可被客户激活。 远程引用层处理语义、管理单一或多重对象的通信，决定调用是应发往一个服务器还是多个。传输层管理实际的连接，并且追踪可以接受方法调用的远程对象。服务器端的骨干网完成对服务器对象实际的方法调用，并获取返回值。返回值向下经远程引用层、服务器端的传输层传递回客户端，再向上经传输层和远程调用层返回。最后，占位程序获得返回值。 

```java
//在Java中，只要一个类extends了java.rmi.Remote接口，即可成为存在于服务器端的远程对象，* 
public interface IHello extends Remote {
　　public String sayHello(String name) throws java.rmi.RemoteException;
}
/**
远程对象必须实现java.rmi.server.UniCastRemoteObject类，这样才能保证客户端访问获得远程对象时，该远程对象将会把自身的一个拷贝以Socket的形式传输给客户端，此时客户端所获得的这个拷贝称为“存根”，而服务器端本身已存在的远程对象则称之为“骨架”。其实此时的存根是客户端的一个代理，用于与服务器端的通信，而骨架也可认为是服务器端的一个代理，用于接收客户端的请求之后调用远程方法来响应客户端的请求。java.rmi.server.UnicastRemoteObject构造函数中将生成stub和skeleton
**/
public class HelloImpl extends UnicastRemoteObject implements IHello {
    // 这个实现必须有一个显式的构造函数，并且要抛出一个RemoteException异常  
    protected HelloImpl() throws RemoteException {
        super();
    }
    private static final long serialVersionUID = 4077329331699640331L;
public String sayHello(String name) throws RemoteException {
    return "Hello " + name;
}
    
//生成stub和skeleton,并返回stub代理引用,本地创建并启动RMI Service，被创建的Registry服务将在指定的端口上侦听到来的请求    
IHello hello = new HelloImpl(); 
LocateRegistry.createRegistry(1099);//服务端创建了一个RegistryImpl对象
java.rmi.Naming.rebind("rmi://localhost:1099/hello", hello);
    
IHello hello = (IHello) Naming.lookup("rmi://localhost:1099/hello");
System.out.println(hello.sayHello("zhangxianxin"));
```

