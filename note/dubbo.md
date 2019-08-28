# 暴露服务

- 服务发布过程中做了哪些事?
- dubbo都有哪些协议,他们之间有什么特点,缺省值是什么?
- 什么是本地暴露和远程暴露,他们的区别?

发布过程的一些动作:

- 暴露本地服务
- 暴露远程服务
- 启动netty
- 连接zookeeper
- 到zookeeper注册
- 监听zookeeper

服务提供者暴露服务：

serviceConfig拿到对外提供服务的实际类ref(如xxxImpl），然后通过ProxyFactory(javaassistProxyFactory或jdkProxyFactory)的getInvoker方法使用ref生成一个AbstractProxyInvoker实例，此时完成具体服务到invoker的转化。接下来就是invoker到exporter的转化。

dubbo支持多种协议,默认使用的是`dubbo`协议.

一个服务可能既是`Provider`,又是`Consumer`,因此就存在他自己调用自己服务的情况,如果再通过网络去访问,那自然是舍近求远,因此他是有`本地暴露`服务的这个设计.从这里我们就知道这个两者的区别

- 本地暴露是暴露在JVM中,不需要网络通信.
- 远程暴露是将ip,端口等信息暴露给远程客户端,调用时需要网络通信.