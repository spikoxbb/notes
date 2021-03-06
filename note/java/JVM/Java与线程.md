[TOC]

# 线程的实现

## 内核线程实现

使用内核线程的一种高级接口---轻量级进程（Light Weight Process，LWP），轻量级进程就是我们通常意义上的线程。

- 需要用户态和内核台来回切换。
- 消耗一定的内核资源，因此一个系统支持轻量级进程的数量是有限的。

## **用户线程实现**

不需要系统内核支援，劣势也在于没有系统内核的支援，所有的线程操作都是需要用户程序自己处理。

阻塞处理等问题的解决十分困难，甚至不可能完成。所以使用用户线程会非常复杂。

## 混合实现

内核线程与用户线程混合使用。

可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级线程来完成，大大降低整个进程被完全阻塞的风险。用户线程与轻量级进程比例是N:M。

## Java线程的实现

JDK1.2之前，绿色线程——用户线程。

JDK1.2——基于操作系统原生线程模型来实现。Sun JDK,它的Windows版本和Linux版本都使用一对一的线程模型实现，一条Java线程就映射到一条轻量级进程之中。

## Java线程调度

线程调度是指系统为线程分配处理器使用权的过程，主要调度方式分两种，分别是协同式线程调度和抢占式线程调度。

- 协同式线程调度
  - 线程执行时间由线程本身来控制，线程把自己的工作执行完之后，要主动通知系统切换到另外一个线程上。
  - 最大好处是实现简单，且切换操作对线程自己是可知的。坏处是线程执行时间不可控制，如果一个线程有问题，可能一直阻塞在那里。
- 抢占式调度
  - 每个线程将由系统来分配执行时间，线程的切换不由线程本身来决定（Java中，Thread.yield()可以让出执行时间，但无法获取执行时间）。
  - 线程执行时间系统可控，也不会有一个线程导致整个进程阻塞。

Java线程调度就是抢占式调度。

希望系统能给某些线程多分配一些时间，给一些线程少分配一些时间，可以通过设置线程优先级来完成。

Java语言一共10个级别的线程优先级（Thread.MIN_PRIORITY至Thread.MAX_PRIORITY），在两线程同时处于ready状态时，优先级越高的线程越容易被系统选择执行。

**但优先级并不是很靠谱，因为Java线程是通过映射到系统的原生线程上来实现的，所以线程调度最终还是取决于操作系统。**

Windows存在优先级推进器：发现一个线程执行的很频繁可能会越过优先级为它分配执行时间。