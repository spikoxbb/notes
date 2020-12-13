[TOC]

# IO类关系图

![java_io_hierarchy](../../../img/java_io_hierarchy.jpg)

# NIO

NIO 主要有三大核心部分：**Channel(通道)**，**Buffer(缓冲区)**, **Selector**。

- 传统 IO 基于字节流和字符流进行操作，而 NIO 基于 Channel 和 Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。
- Selector(选择区)用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

## 缓冲区

- Java IO 面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。
- NIO 的缓冲导向方法不同。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性。但是，还需要检查是否该缓冲区中包含所有您需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不要覆盖缓冲区里尚未处理的数据。

## NIO 的非阻塞

- IO的各种流是阻塞的。
- NIO是非阻塞的，非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。 线程通常将非阻塞 IO 的空闲时间用于在其它通道上执行 IO 操作，所以一个单独的线程现在可以管理多个输入和输出通道。

## Channel

Channel 和 IO 中的 Stream是差不多一个等级的。只不过 Stream 是单向的，譬如：InputStream, OutputStream，而 Channel 是双向

的，既可以用来进行读操作，又可以用来进行写操作。NIO 中的 Channel 的主要实现有：

- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel

## Buffer

缓冲区，实际上是一个容器，是一个连续数组。Channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 Buffer。

Buffer 是一个顶层父类，它是一个抽象类，常用的 Buffer 的子类有：

- ByteBuffer
- IntBuffer
- CharBuffer
-  LongBuffer
- DoubleBuffer
- FloatBuffer
- ShortBuffer

## Selector

Selector 能够检测多个注册的通道上是否有事件发生，如果有事件发生，便获取事件然后针对每个事件进行相应的响应处理。