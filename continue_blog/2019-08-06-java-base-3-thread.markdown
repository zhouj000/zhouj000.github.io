---
layout:     post
title:      "Java基础 线程与并发"
date:       2019-05-05
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
--- 



[Java性能优化02-并行优化](https://zhouj000.github.io/2019/01/08/java-optimize-02/)  

[Java基础: JVM(九) Java内存模型](https://zhouj000.github.io/2019/07/09/java-base-jmm/)  



# 线程

进程是操作系统调度和分配资源的基本单位，进程之间的通信需要通过专门的系统机制，比如消息、socket和管道来完成。而线程是比进程更小的执行单位，每个线程拥有自己的栈和寄存器等资源数据，多个线程之间共享进程的代码、数据和文件

线程的优点如下：  
1、创建一个线程比创建一个进程的代价要小  
2、线程的切换比进程间的切换代价小  
3、充分利用多处理器  
4、数据共享(数据共享使得线程之间的通信比进程间的通信更高效)  

在Java中创建线程有3种方式：  
1、继承Thread类  
2、实现Runnable接口  
3、实现Callable接口


## 线程状态

![]()
线程状态可以分为：  
1、新建(new)  
2、就绪状态/可运行状态(Runnable)  
3、运行状态(Running)  
4、阻塞状态(Blocked)：等待阻塞、同步阻塞、其他阻塞  
5、死亡状态(Dead)

其中sleep方法会休眠一段时间(进入阻塞)，到时间后进入就绪状态，并不保证能马上运行。yield方法类似让出资源，让优先级更高的线程执行，自己不会阻塞而是进入就绪状态，让系统的线程调度器重新调度器重新调度一次，因此也有可能之后还是自己执行。wait方法则会让线程进入阻塞状态，直到有线程notify/notifyAll唤醒。而当一个线程需要等另一个(或多个)线程先执行完再继续执行时可以使用join方法。使用interrupt方法可以中断线程，将"中断标记"设置为true，如果该线程正处于阻塞状态则将"中断标记"立即清除为“false”并抛出InterruptedException异常。interrupted方法判断当前线程是否处于中断状态，返回后会清除中断状态，而isInterrupted方法则只是判断中断状态并不会清除

守护线程，可以在创建线程后将其setDaemon(true)即可。比如JVM的垃圾回收。内存管理都是守护线程等，当所有线程退出后，守护线程也将结束



# 多线程

并发(concurrency)和并行(parallellism)都是完成多任务更加有效率的方式，但还是有一些区别的：  
1、并发：交替做不同事情的能力，CPU时间片执行，体现在不同的代码块交替执行，强调在一个时间段内同时执行  
2、并行：同时做不同事情的能力，多核执行，体现在不同的代码块同时执行

对于多线程执行，如果没有共享变量，那大家都相安无事，相当于各个单线程执行，线程安全。而如果多个线程间有共享变量，又由于Java内存模型的通信机制，就会发生数据不同步、死锁等问题，这2个问题在之前的博客中讲过了(synchronized、volatile、原子性操作、CAS、顺序获取资源、锁分离等)

#### 伪共享

为了解决计算器中主内存与CPU之间运行速度差的问题，会再CPU与主内存中添加一级或多级高速缓冲存储器(cache)(L1/L2)，这个cache是集成在CPU内部的，所以也叫CPU Cache。在Cache内部是按行存储的，其中一行成为一个Cache行，其是Cache与主内存进行交换数据的单位，Cache行的大小一般为2的幂次数字节。当CPU访问某个变量时，会先去CPU Cache查看，如果没有再到主内存获取，并将该变量所在内存区域的一个Cache行大小的内存复制进Cache中，由于存放在Cache行的是内存块而不是变量，所以可能有多个变量存储到一个Cache行中。如果多个线程同时修改一个缓存行里的多个变量时，由于同时只能有一个线程操作缓存行，所以相比将每个变量放到一个缓存行，性能就会有所下降，这就是伪共享

比如x，y同时存在一个缓存行，线程1修改了x，首先会修改CPU1的一级缓存x所在行，在一致性协议下，CPU2的变量x所在的缓存行失效，你们线程2在写入x时就回去二级缓存中查找，破坏了一级缓存，如果只有一级缓存则会导致频繁地访问主内存

伪共享的产生原因是多个变量放在一个缓存行，这在单线程下对代码执行是更有利的，执行更快，多线程下相反。JDK 8之前是通过字节填充的方式解决的，创建一个变量时使用填充字段填充其缓存行，避免同一个缓存行中有多个变量。在JDK 8提供了一个sun.misc.Contended注解来解决伪共享问题，需要注意的是这个注解默认只使用于Java核心类，比如rt包下的类，我们自己创建的类要使用则需要`-XX:-RestrictContended`自定义宽度


## CAS

CAS即compare and swap，是JDK提供的非阻塞原子性操作，它通过硬件保证了比较 —— 更新操作的原子性。Unsafe类中有3个compareAndSwap*方法。JDK的rt.jar包的Unsafe类提供了硬件级别的原子性操作，其中的方法都是native方法，使用JNI的方式访问本地C++实现库，提供的这些功能包括裸内存的申请/释放/访问，低层硬件的atomic/volatile支持，创建未初始化对象等。扩展了Java语言表达能力，便于在高层Java代码里实现原本在更底层C中实现的核心库功能。许多广泛使用的高性能开发库都是基于Unsafe类开发，比如 Netty、Hadoop、Kafka 等

虽然Unsafe很强大，可以直接操作内存，但是由于安全性考虑无法直接获取，但是可以通过反射拿到
```java
@CallerSensitive
public static Unsafe getUnsafe() {
	Class var0 = Reflection.getCallerClass();
	// 是不是Bootstrap类加载器加载的
	if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
		throw new SecurityException("Unsafe");
	} else {
		return theUnsafe;
	}
}
```

扩展：  
[Unsafe 相关整理](https://www.jianshu.com/p/2e5b92d0962e)  

## 原子操作类

。。
JDK 8 新增的原子操作类LongAdder










## 锁









按处理方式：乐观锁、悲观锁  
按抢占方式：公平锁、非公平锁  
按持有：独占锁、共享锁  



## 其他


### threadlocal

threadlocal / inheritableThreadLocal  继承

Random  next随机种子 原子变量 /   threadlocalRandom 

https://blog.csdn.net/u011497638/article/details/93889173





## 并发包中并发List 源码
 
初始化   添加元素  获取指定位置元素  修改指定元素  删除元素  弱一致性的迭代器

@@@ Java 并发包中并发队列

LinkedBlockingQueue   ArrayBlockingQueue    PriorityBlockingQueue     DelayQueue 






##  并发包中锁原理

LockSupport工具类  抽象同步队列AQS(锁的底层支持/条件变量的支持/自定义同步器)
独占锁ReentrantLock(类图结构/获取锁/释放锁)

## 读写锁ReentrantReadWriteLock 

类图结构   写锁的获取与释放   读锁的获取与释放

## JDK 8 中新增的StampedLock 锁









## 线程池ThreadPoolExecutor 

ScheduledThreadPoolExecutor 

## 线程同步器CountDownLatch    回环屏障CyclicBarrier   信号量Semaphore

 











11.1 ArrayBlockingQueue 的使用 284
11.1.1 异步日志打印模型概述 284
11.1.2 异步日志与具体实现 285

11.2 Tomcat 的NioEndPoint 中ConcurrentLinkedQueue 的使用 293
11.2.1 生产者——Acceptor 线程 294
11.2.2 消费者——Poller 线程 298

11.3 并发组件ConcurrentHashMap 使用注意事项 300

11.4 SimpleDateFormat 是线程不安全的 304
11.4.1 问题复现 304
11.4.2 问题分析 305

11.5 使用Timer 时需要注意的事情 309
11.5.1 问题的产生 309
11.5.2 Timer 实现原理分析 310

11.6 对需要复用但是会被下游修改的参数要进行深复制 314
11.6.1 问题的产生 314
11.6.2 问题分析 316

11.7 创建线程和线程池时要指定与业务相关的名称 319
11.7.1 创建线程需要有线程名 319
11.7.2 创建线程池时也需要指定线程池的名称 321

11.8 使用线程池的情况下当程序结束时记得调用shutdown 关闭线程池 325
11.8.1 问题复现 325
11.8.2 问题分析 327

11.9 线程池使用FutureTask 时需要注意的事情 329
11.9.1 问题复现 329
11.9.2 问题分析 332

11.10 使用ThreadLocal 不当可能会导致内存泄漏 336
11.10.1 为何会出现内存泄漏 336
11.10.2 在线程池中使用ThreadLocal 导致的内存泄漏 339
11.10.3 在Tomcat 的Servlet 中使用ThreadLocal 导致内存泄漏 341