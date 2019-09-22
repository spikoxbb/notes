[TOC]

# RMI

## 原理

客户端如何能就像调用本地方法一样来调用远程机器上的方法，因此rmi的开发人员就引入了stub和skeleton模型.

所有与网络相关的代码都放在了stub和skeleton中，这样客户端和服务端就不需要处理网络相关的代码了。

stub同样也实现了和服务端同样 java.rmi.Remote接口，这样当client想调用server上方法的时候，就可以调用stub上的相同的方法，但是stub里面只有和网络相关的处理逻辑，并没有对应的业务处理逻辑，比如说server上有一个add方法，stub中同样也有一个add方法，但是stub上的这个add方法并不包含添加的逻辑实现，他仅仅包含如何连接到远程的skeleton、调用方法的详细信息、参数、返回值等等。

所以，目前看来大概是这个样子的：
Client<-->stub<-->[NETWORK]<-->skeleton<-->Server

在jdk1.2之后，skeleton就被合并到server中了，所以看起来是这个样子:
Client<--->stub<--->[NETWORK]<--->Server_with_skeleton

```java
/* IHello.java */
package mytest;
/* 
 *　在Java中，只要一个类extends了java.rmi.Remote接口，即可成为存在于服务器端的远程对象， 
 * 供客户端访问并提供一定的服务。JavaDoc描述：Remote 接口用于标识其方法可以从非本地虚拟机上 
 * 调用的接口。任何远程对象都必须直接或间接实现此接口。只有在“远程接口” 
 * （扩展 java.rmi.Remote 的接口）中指定的这些方法才可被远程调用。 
 */
import java.rmi.Remote;
 
public interface IHello extends Remote {
　　　　/* extends了Remote接口的类或者其他接口中的方法若是声明抛出了RemoteException异常，
 　　　　* 则表明该方法可被客户端远程访问调用。
 　　　　*/
	public String sayHello(String name) throws java.rmi.RemoteException;
}

/*
 * 远程对象必须实现java.rmi.server.UniCastRemoteObject类，这样才能保证客户端访问获得远程对象时，
 * 该远程对象将会把自身的一个拷贝以Socket的形式传输给客户端，此时客户端所获得的这个拷贝称为“存根”，
 * 而服务器端本身已存在的远程对象则称之为“骨架”。其实此时的存根是客户端的一个代理，用于与服务器端的通信，
 * 而骨架也可认为是服务器端的一个代理，用于接收客户端的请求之后调用远程方法来响应客户端的请求。
 */ 
 
/* java.rmi.server.UnicastRemoteObject构造函数中将生成stub和skeleton */
public class HelloImpl extends UnicastRemoteObject implements IHello {
    // 这个实现必须有一个显式的构造函数，并且要抛出一个RemoteException异常  
    protected HelloImpl() throws RemoteException {
        super();
    }
    
    private static final long serialVersionUID = 4077329331699640331L;
    public String sayHello(String name) throws RemoteException {
        return "Hello " + name + " ^_^ ";
    }
}


public class HelloServer {
    public static void main(String[] args) {
        try {
            IHello hello = new HelloImpl(); /* 生成stub和skeleton,并返回stub代理引用 */
            /* 本地创建并启动RMI Service，被创建的Registry服务将在指定的端口上侦听到来的请求 
             * 实际上，RMI Service本身也是一个RMI应用，我们也可以从远端获取Registry:
             *     public interface Registry extends Remote;
             *     public static Registry getRegistry(String host, int port) throws RemoteException;
             */
            LocateRegistry.createRegistry(1099);
            /* 将stub代理绑定到Registry服务的URL上 */
            java.rmi.Naming.rebind("rmi://localhost:1099/hello", hello);
            System.out.print("Ready");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}


/* 客户端向服务端请求远程对象服务 */
public class Hello_RMI_Client {
    public static void main(String[] args) {
        try {
            /* 从RMI Registry中请求stub
             * 如果RMI Service就在本地机器上，URL就是：rmi://localhost:1099/hello
             * 否则，URL就是：rmi://RMIService_IP:1099/hello
             */
            IHello hello = (IHello) Naming.lookup("rmi://localhost:1099/hello");
            /* 通过stub调用远程接口实现 */
            System.out.println(hello.sayHello("zhangxianxin"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

​        

RemoteObject抽象类实现了Remote接口和序列化Serializable接口，它和它的子类提供RMI服务器函数。当客户端通过RMI注册表找到一个远程接口的时候，所得到的其实是远程接口的一个动态代理对象。当客户端调用其中的方法的时候，方法的参数对象会在序列化之后，传输到服务器端。服务器端接收到之后，进行反序列化得到参数对象。并使用这些参数对象，在服务器端调用实际的方法。调用的返回值Java对象经过序列化之后，再发送回客户端。客户端再经过反序列化之后得到Java对象，返回给调用者。这中间的序列化过程对于使用者来说是透明的，由动态代理对象自动完成。