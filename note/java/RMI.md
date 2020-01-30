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

## 过程

**如果有新的方法调用请求到来，Server产生一个单独的线程来处理新接收的请求.**

```java
//Server端调用UnicastRemoteObject的export方法输出远程对象，export方法会在一个线程里监听某个TCP端口上的方法调用请求：
public void exportObject(Target target) throws RemoteException {  
   ......
while (true) {  
    ServerSocket myServer = server;  
    if (myServer == null)  
      return;  
    Throwable acceptFailure = null;  
    final Socket socket;   
    try {  
    socket = myServer.accept();  
    InetAddress clientAddr = socket.getInetAddress();  
    String clientHost = (clientAddr != null  
                 ? clientAddr.getHostAddress()  
                 : "0.0.0.0");  
    Thread t = (Thread)  
        java.security.AccessController.doPrivileged (  
            new NewThreadAction(new ConnectionHandler(socket,  
                              clientHost),  
                    "TCP Connection(" + ++ threadNum +  
                    ")-" + clientHost,  
                    true, true));  
    t.start();        
    } catch (IOException e) {  
    acceptFailure = e;  
    } catch (RuntimeException e) {  
    acceptFailure = e;  
    } catch (Error e) {  
    acceptFailure = e;  
    }  
}  
......
```

  **dispatch方法处理接收的请求**

```java
public void dispatch(Remote obj, RemoteCall call) throws IOException {  
// positive operation number in 1.1 stubs;  
// negative version number in 1.2 stubs and beyond...  
int num;  
long op;  
try {  
    // read remote call header  
    ObjectInput in;  
    try {  
    in = call.getInputStream();  
    num = in.readInt();  
    if (num >= 0) {  
        if (skel != null) {  
        oldDispatch(obj, call, num);  
        return;  
        } else {  
        throw new UnmarshalException(  
            "skeleton class not found but required " +  
            "for client version");  
        }  
    }  
    //读取方法的编号来获得方法名称 
    op = in.readLong();
    } catch (Exception readEx) {  
    throw new UnmarshalException("error unmarshalling call header",  
                     readEx);  
    }  
    /* 
     * Since only system classes (with null class loaders) will be on 
     * the execution stack during parameter unmarshalling for the 1.2 
     * stub protocol, tell the MarshalInputStream not to bother trying 
     * to resolve classes using its superclasses's default method of 
     * consulting the first non-null class loader on the stack. 
     */  
    MarshalInputStream marshalStream = (MarshalInputStream) in;  
    marshalStream.skipDefaultResolveClass();  
    Method method = (Method) hashToMethod_Map.get(new Long(op));  
    if (method == null) {  
    throw new UnmarshalException("invalid method hash");  
    }  
    // if calls are being logged, write out object id and operation  
    logCall(obj, method);  
    // unmarshal parameters  
    Class[] types = method.getParameterTypes();  
    Object[] params = new Object[types.length];  
    try {  
    unmarshalCustomCallData(in);  
    for (int i = 0; i < types.length; i++) {  
        params[i] = unmarshalValue(types[i], in);  
    }  
    } catch (java.io.IOException e) {  
    throw new UnmarshalException(  
        "error unmarshalling arguments", e);  
    } catch (ClassNotFoundException e) {  
    throw new UnmarshalException(  
        "error unmarshalling arguments", e);  
    } finally {  
    call.releaseInputStream();  
    }  
    // make upcall on remote object  
    Object result;  
    try {  
    result = method.invoke(obj, params);  
    } catch (InvocationTargetException e) {  
    throw e.getTargetException();  
    }  
    // marshal return value  
    try {  
    ObjectOutput out = call.getResultStream(true);  
    Class rtype = method.getReturnType();  
    if (rtype != void.class) {  
        //方法执行结果写入到输出流中
        marshalValue(rtype, result, out);  
    }  
    } catch (IOException ex) {  
    throw new MarshalException("error marshalling return", ex);  
    /* 
     * This throw is problematic because when it is caught below, 
     * we attempt to marshal it back to the client, but at this 
     * point, a "normal return" has already been indicated, 
     * so marshalling an exception will corrupt the stream. 
     * This was the case with skeletons as well; there is no 
     * immediately obvious solution without a protocol change. 
     */  
    }  
} catch (Throwable e) {  
    logCallException(e);  
      
    ObjectOutput out = call.getResultStream(false);  
    if (e instanceof Error) {  
    e = new ServerError(  
        "Error occurred in server thread", (Error) e);  
    } else if (e instanceof RemoteException) {  
    e = new ServerException(  
        "RemoteException occurred in server thread",  
        (Exception) e);  
    }  
    if (suppressStackTraces) {  
    clearStackTraces(e);  
    }  
    out.writeObject(e);  
} finally {  
    call.releaseInputStream(); // in case skeleton doesn't  
    call.releaseOutputStream();  
}  
}  

protected static void marshalValue(Class type, Object value, ObjectOutput out) throws IOException{ if (type.isPrimitive()) {  
    if (type == int.class) {  
    out.writeInt(((Integer) value).intValue());  
    } else if (type == boolean.class) {  
    out.writeBoolean(((Boolean) value).booleanValue());  
    } else if (type == byte.class) {  
    out.writeByte(((Byte) value).byteValue());  
    } else if (type == char.class) {  
    out.writeChar(((Character) value).charValue());  
    } else if (type == short.class) {  
    out.writeShort(((Short) value).shortValue());  
    } else if (type == long.class) {  
    out.writeLong(((Long) value).longValue());  
    } else if (type == float.class) {  
    out.writeFloat(((Float) value).floatValue());  
    } else if (type == double.class) {  
    out.writeDouble(((Double) value).doubleValue());  
    } else {  
    throw new Error("Unrecognized primitive type: " + type);  
    }  
} else {  
    out.writeObject(value);  
}  
  }  

protected static Object unmarshalValue(Class type, ObjectInput in) throws IOException, ClassNotFoundException  {  
if (type.isPrimitive()) {  
    if (type == int.class) {  
    return new Integer(in.readInt());  
    } else if (type == boolean.class) {  
    return new Boolean(in.readBoolean());  
    } else if (type == byte.class) {  
    return new Byte(in.readByte());  
    } else if (type == char.class) {  
    return new Character(in.readChar());  
    } else if (type == short.class) {  
    return new Short(in.readShort());  
    } else if (type == long.class) {  
    return new Long(in.readLong());  
    } else if (type == float.class) {  
    return new Float(in.readFloat());  
    } else if (type == double.class) {  
    return new Double(in.readDouble());  
    } else {  
    throw new Error("Unrecognized primitive type: " + type);  
    }  
} else {  
    return in.readObject();  
}  
 }  

```

 **Client端请求一个远程方法调用:**

```java
 public Object invoke(Remote obj, java.lang.reflect.Method method, Object[] params, long opnum)  
throws Exception {  
if (clientRefLog.isLoggable(Log.VERBOSE)) {  
    clientRefLog.log(Log.VERBOSE, "method: " + method);  
}  
if (clientCallLog.isLoggable(Log.VERBOSE)) {  
    logClientCall(obj, method);  
}  
  
Connection conn = ref.getChannel().newConnection();  
RemoteCall call = null;  
boolean reuse = true;  
/* If the call connection is "reused" early, remember not to 
 * reuse again. 
 */  
boolean alreadyFreed = false;  
try {  
    if (clientRefLog.isLoggable(Log.VERBOSE)) {  
    clientRefLog.log(Log.VERBOSE, "opnum = " + opnum);  
    }  
    // create call context  
    call = new StreamRemoteCall(conn, ref.getObjID(), -1, opnum);  
    // marshal parameters  
    try {  
    ObjectOutput out = call.getOutputStream();  
    marshalCustomCallData(out);  
    Class[] types = method.getParameterTypes();  
    for (int i = 0; i < types.length; i++) {   
        marshalValue(types[i], params[i], out);  
    }  
    } catch (IOException e) {  
    clientRefLog.log(Log.BRIEF,  
        "IOException marshalling arguments: ", e);  
    throw new MarshalException("error marshalling arguments", e);  
    }  
    // unmarshal return  
    call.executeCall();  
    try {  
    Class rtype = method.getReturnType();  
    if (rtype == void.class)  
        return null;  
    ObjectInput in = call.getInputStream();  
      
    /* StreamRemoteCall.done() does not actually make use 
     * of conn, therefore it is safe to reuse this 
     * connection before the dirty call is sent for 
     * registered refs.   
     */  
    Object returnValue = unmarshalValue(rtype, in);  
    /* we are freeing the connection now, do not free 
     * again or reuse. 
     */  
    alreadyFreed = true;  
    /* if we got to this point, reuse must have been true. */  
    clientRefLog.log(Log.BRIEF, "free connection (reuse = true)");  
    /* Free the call's connection early. */  
    ref.getChannel().free(conn, true);  
    return returnValue;  
      
    } catch (IOException e) {  
    clientRefLog.log(Log.BRIEF,  
             "IOException unmarshalling return: ", e);  
    throw new UnmarshalException("error unmarshalling return", e);  
    } catch (ClassNotFoundException e) {  
    clientRefLog.log(Log.BRIEF,  
        "ClassNotFoundException unmarshalling return: ", e);  
    throw new UnmarshalException("error unmarshalling return", e);  
    } finally {  
    try {  
        call.done();  
    } catch (IOException e) {  
        /* WARNING: If the conn has been reused early, 
         * then it is too late to recover from thrown 
         * IOExceptions caught here. This code is relying 
         * on StreamRemoteCall.done() not actually 
         * throwing IOExceptions.   
         */  
        reuse = false;  
    }  
    }  
} catch (RuntimeException e) {  
    /* 
     * Need to distinguish between client (generated by the 
     * invoke method itself) and server RuntimeExceptions. 
     * Client side RuntimeExceptions are likely to have 
     * corrupted the call connection and those from the server 
     * are not likely to have done so.  If the exception came 
     * from the server the call connection should be reused. 
     */  
    if ((call == null) ||   
    (((StreamRemoteCall) call).getServerException() != e))  
           {  
    reuse = false;  
    }  
    throw e;  
} catch (RemoteException e) {  
    /* 
     * Some failure during call; assume connection cannot 
     * be reused.  Must assume failure even if ServerException 
     * or ServerError occurs since these failures can happen 
     * during parameter deserialization which would leave 
     * the connection in a corrupted state. 
     */  
    reuse = false;  
    throw e;  
} catch (Error e) {  
    /* If errors occurred, the connection is most likely not 
            *  reusable.  
     */  
    reuse = false;  
    throw e;  
} finally {  
    /* alreadyFreed ensures that we do not log a reuse that 
     * may have already happened. 
     */  
    if (!alreadyFreed) {  
    if (clientRefLog.isLoggable(Log.BRIEF)) {  
        clientRefLog.log(Log.BRIEF, "free connection (reuse = " +  
                   reuse + ")");  
    }  
    ref.getChannel().free(conn, reuse);  
    }  
}  
   }  

```

## RMI线程模型

在JDK1.5及以前版本中，RMI每接收一个远程方法调用就生成一个单独的线程来处理这个请求，请求处理完成后，这个线程就会释放：

```java
Thread t = (Thread)  
            java.security.AccessController.doPrivileged (  
                new NewThreadAction(new ConnectionHandler(socket,  
                                  clientHost),  
                        "TCP Connection(" + ++ threadNum +  
                        ")-" + clientHost,  
                        true, true));
```

在JDK1.6之后，RMI使用线程池来处理新接收的远程方法调用请求-ThreadPoolExecutor。

在JDK1.6中，RMI提供了可配置的线程池参数属性：

- sun.rmi.transport.tcp.maxConnectionThread - 线程池中的最大线程数量
- sun.rmi.transport.tcp.threadKeepAliveTime - 线程池中空闲的线程存活时间

