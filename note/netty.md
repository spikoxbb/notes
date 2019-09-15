[TOC]

# 堆外内存之DirectByteBuffer 

堆内内存是由JVM所管控的Java进程内存，我们平时在Java中创建的对象都处于堆内内存中，并且它们遵循JVM的内存管理机制，JVM会采用垃圾回收机制统一管理它们的内存。那么堆外内存就是存在于JVM管控之外的一块内存区域，因此它是不受JVM的管控。

**DirectByteBuffer是通过虚引用(Phantom Reference)来实现堆外内存的释放的。**

## 关于linux的内核态和用户态

- 内核态：控制计算机的硬件资源，并提供上层应用程序运行的环境。比如socket I/0操作或者文件的读写操作等
- 用户态：上层应用程序的活动空间，应用程序的执行必须依托于内核提供的资源。
- 系统调用：为了使上层应用能够访问到这些资源，内核为上层应用提供访问的接口。

**通过JNI调用的native方法可以从用户态切换到了内核态。并且通过该系统调用使用操作系统所提供的功能。**intel cpu提供Ring0-Ring3四种级别的运行模式，Ring0级别最高，Ring3最低。Linux使用了Ring3级别运行用户态，Ring0作为内核态。Ring3状态不能访问Ring0的地址空间，包括代码和数据。因此用户态是没有权限去操作内核态的资源的，它只能通过系统调用外完成用户态到内核态的切换，然后在完成相关操作后再有内核态切换回用户态。

## DirectByteBuffer ———— 直接缓冲

DirectByteBuffer是Java用于实现堆外内存的一个重要类，可以通过该类实现堆外内存的创建、使用和销毁。DirectByteBuffer中的unsafe.allocateMemory(size);是个一个native方法，这个方法分配的是堆外内存，通过C的malloc来进行分配的。在DirectByteBuffer的父类Buffer中有个address属性：

```java
    //address表示分配的堆外内存的地址。
　　//unsafe.allocateMemory(size);分配完堆外内存后就会返回分配的堆外内存基地址，并将这个地址赋值给了address属性。这样后面通过JNI对这个堆外内存操作时都是通过这个address来实现的了。
    long address;
```

**在内核态的场景下，操作系统是可以访问任何一个内存区域的，所以操作系统是可以访问到Java堆的这个内存区域的。**
**那为什么操作系统不直接访问Java堆内的内存区域了？**

**因为JNI方法访问的内存区域是一个已经确定了的内存区域地质，那么该内存地址指向的是Java堆内内存的话，那么如果在操作系统正在访问这个内存地址的时候，Java在这个时候进行了GC操作，而GC操作会涉及到数据的移动操作，所以JNI调用的内存是不能进行GC操作的。堆内内存与堆外内存之间数据拷贝的方式(并且在将堆内内存拷贝到堆外内存的过程JVM会保证不会进行GC操作)**

要完成一个从文件中读数据到堆内内存的操作，即FileChannelImpl.read(HeapByteBuffer)。这里实际上File I/O会将数据读到堆外内存中，然后堆外内存再把数据拷贝到堆内内存，这样我们就读到了文件中的内存。

```java
    static int read(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
        if (var1.isReadOnly()) {
            throw new IllegalArgumentException("Read-only buffer");
        } else if (var1 instanceof DirectBuffer) {
            return readIntoNativeBuffer(var0, var1, var2, var4);
        } else {
            // 分配临时的堆外内存
            ByteBuffer var5 = Util.getTemporaryDirectBuffer(var1.remaining());
            int var7;
            try {
                // File I/O 操作会将数据读入到堆外内存中
                int var6 = readIntoNativeBuffer(var0, var5, var2, var4);
                var5.flip();
                if (var6 > 0) {
                    // 将堆外内存的数据拷贝到堆内内存中
                    var1.put(var5);
                }
                var7 = var6;
            } finally {
                // 里面会调用DirectBuffer.cleaner().clean()来释放临时的堆外内存
                Util.offerFirstTemporaryDirectBuffer(var5);
            }
            return var7;
        }
    }
```

而写操作则反之，我们会将堆内内存的数据线写到对堆外内存中，然后操作系统会将堆外内存的数据写入到文件中。

直接使用堆外内存，如DirectByteBuffer：这种方式是直接在堆外分配一个内存(即，native memory)来存储数据，程序通过JNI直接将数据读/写到堆外内存中。因为数据直接写入到了堆外内存中，所以这种方式就不会再在JVM管控的堆内再分配内存来存储数据了，也就不存在堆内内存和堆外内存数据拷贝的操作了。这样在进行I/O操作时，只需要将这个堆外内存地址传给JNI的I/O的函数就好了。

# Buffer

- capacity：一个buffer的capacity指的就是它所包含的元素的个数。buffer的capacity永远不会是负数，且永远不会变化。
- limit：一个buffer的limit指的是不应该被读或写的第一个元素的索引( position <= limit )。一个buffer的limit永远不会是负数的，并且永远不会超过它的capacity。
- position：一个buffer的position指的是下一个将要被读或写的元素的索引。一个buffer的position永远不会是负数的，并且永远不会超过它的limit( 这里也说明，position最多等于limit，当position==limit时，这个时候是不能够在从buffer中读取到数据了 )。

## Java NIO 内存分配

- Heap buffer ：堆栈的内存分配。堆栈就是Java内存模型当中内存的区域，位于堆上，堆是我们生成对象的区域。
- Direct buffer ：堆外内存分配。这个内存本身不是由JVM进行控制的，它是由操作系统进行统一的处理的。

## 方法

- flip()：flip方法将Buffer从写模式切换到读模式。

  ```java
   public final Buffer flip() {
          limit = position;
          position = 0;
          mark = -1;
          return this;
      }
  ```

- rewind()：rewind()方法将position设回0，limit保持不变，所以可以重读Buffer中的所有数据。可见在调用rewind()之前Buffer已经是处于读模式了

- clear()：让Buffer重新准备好重头开始再次被写入。该方法会将position、limit重置。如果此时还没有读取的数据，则就无法读取到了。虽然clear()不会清楚数据，但是position、limit标志位被重置了，所以无法找到哪些未读取数据的位置了。

- compact()：compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据。

- Slice Buffer与原有buffer共享相同的底层数据：ByteBuffer.slice(start, end) —————— [start, end)，即包含start，不包含end，slice返回的ByteBuffer底层数据和源ByteBuffer是共享的，所以无论对那个buffer进行修改，都会影响到另一buffer。

- buffer.asReadOnlyBuffer()：只读buffer适用于方法传递时，你只希望你的调用端去读取你所提供的buffer。即，将一个只读buffer当做参数传递给某个方法。

- ByteBuffer.wrap(byte[] array)：该方法生成的ByteBuffer底层就是你传进来的这个array数组，并没有进行数组拷贝，所以是和你传进来的array共享内容的。这也导致如果你修改了传进来的array数组的内容，是会反映到ByteBuffer的。

- Scattering与Gathering：
  Scattering：允许read的时候传递一个buffer[]数组。将一个Channel中的数据给读到了多个buffer当中，它是按照顺序依次读入buffer当中的，而且总是当当前buffer已经写满了才会写下一个buffer。
  Gathering：允许write的时候传递一个buffer[]数组。将多个buffer的数据写到一个Channel中。它会将第一个buffer中可读的数据都写入channel后，再将下一个buffer中的数据写入到channel中，以此依次将buffer中可读取的数据写到channel中。

  *Scattering与Gathering适用于网络操作中的自定义协议。比如，一个请求中带有两个请求头以及一个body，第一个请求头的数据长度固定是10个byte，第二个请求头的数据长度固定是5个byte，而body的长度是不确定的。那么我们就可以用3个buffer组成的数组来接这样的请求。bytebuffer[]数组中，第一个bytebuffer元素的容量为10，用于接受第一个请求头的信息；第二个bytebuffer元素的容量为5，用于接受第二个请求头的信息；第三个定义一个大容量的bytebuffer用于接受body的信息。这样就天然的实现了一种数据的分门别类。*

## Selector

与Selector一起使用时，Channel必须处于非阻塞模式下。这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式。而套接字通道都可以。