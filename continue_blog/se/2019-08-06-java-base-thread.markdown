---
layout:     post
title:      "Java基础SE(三) 线程与并发"
date:       2019-05-05
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
--- 


[Java基础SE(二) 集合](https://zhouj000.github.io/2021/02/26/java-base-collections/)  


[Java基础: JVM(九) Java内存模型](https://zhouj000.github.io/2019/07/09/java-base-jmm/)  
[Java性能优化02-并行优化](https://zhouj000.github.io/2019/01/08/java-optimize-02/)  




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





java虚拟机的实现中每个对象都有一个对象头，用于保存对象的系统信息。对象头中有一个mark word的部分，它是实现锁的关键。在32位系统中，它为32位的数据；在64位系统中，它为64位的数据。它是一个多功能的数据区，可以存放对象的哈希值、对象年龄、锁的指针等信息。一个对象是否占用锁，占有哪个锁，就记录在这个mark word中

以32位系统为例，普通对象的对象头比如：  
`hash:25 ------------>| age:4 biased_lock:1 lock:2`  
它表示mark word中有25位比特表示对象的哈希值，4位比特表示对象的年龄，1位比特表示是否偏向锁，2位比特表示锁的信息

当对象处于普通的未锁定状态时，格式为:  
`[header      |0|01] unlocked`  
前29位表示对象的哈希值、年龄等信息。倒数第3位位为0，最后两位为01，表示未锁定，与偏向锁最后2位相同，因此虚拟机通过倒数第三位比特来区分是否为偏向锁

偏向锁是程序没有竞争时、取消之前已经获得锁的线程同步操作，即某一锁被线程获取后，进入偏向模式，CAS操作在对象头存储锁偏向的线程 ID，该线程再次请求这个锁时，无需进行相关同步操作(CAS操作来加锁和解锁)，只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。对于偏向锁的对象，其对象头记录获得锁的线程：  
`[javaThread* | epoch | age |1|01]`  
前23位表示持有偏向锁的线程，后续2位表示偏向锁的时间戳，4位比特表示对象年龄，年龄后1位为1表示偏向锁，最后2位位01表示可偏向/未锁定。当获得锁的线程再次尝试获取锁时，通过mark word的线程信息可以判断当前线程是否持有偏向锁。当然在锁竞争激烈的场景下反而得不到优化，可以使用`-XX:-UseBiasedLocking`禁用偏向锁

偏向锁失败后，java虚拟机让线程申请轻量级锁。轻量级锁在虚拟机内部使用一个称为BasicObjectLock的对象实现，这个对象内部由一个BasicLock对象和一个持有该锁的Java对象指针组成。BaseicObjectLock对象放置在Java栈的栈帧中，在BasicLock对象内部还维护着displaced_header字段，它用于备份对象头部的mark word。当对象处于轻量级锁定时其头部mark word为:  
`[ptr         |00] locked`  
末尾2位为00。此时整个mark word为指向BasicLock对象的指针，由于BasicObjectLock对象在线程栈中，因此该指针必然指向持有该锁的线程栈空间，即存放在获得锁的线程栈中的该对象真实对象头。当需要判断某一线程是否持有该对象锁时，也只需简单判断对象头的指针是否在当前线程的栈地址范围即可。同时BasicLock对象的displaced_header备份了原对象的mark word内容。BasicObjectLock对象的obj字段则指向该对象
```
markOop mark = obj -> mark();
lock -> set_displaced_header(mark);
if (mark == (markOop)Atomic::cmpxchg_ptr(lock, ojb()->mark_addr(), mark)) {
	TEVENT(slow_enter: release stacklock);
	return;
}
```
首先BasicLock通过set_displaced_header方法备份原对象的mark word。然后使用CAS操作尝试将BasicLock的地址复制到对象头的mark word，如果复制成功则枷锁成功，如果失败则认为加锁失败，那么轻量级锁有可能膨胀为重量级锁
![]()

当轻量级锁失败，虚拟机就会使用重量级锁，当对象处于重量级锁定时对象的mark word为：  
`[ptr         |10] monitor`  
末尾2位为10。整个mark word表示指向monitor对象的指针，在轻量级锁处理失败后，虚拟机会执行以下操作：  
```
lock -> set_displaced_header(markOopDesc::unused_mark());
ObjectSynchronizer::inflate(THREAD, obj())-> enter(THREAD);
```
首先废弃前面BasicLock备份的对象头信息，然后正式启用重量级锁。启用过程分为两步：首先通过inflate()方法进行锁膨胀，其目的是获得对象的ObjectMonitor，然后使用enter()方法尝试进入该锁。在enter()方法调用中，线程很可能会在操作系统层面被挂起，如果这样，线程间切换和调度的成本会比较高。因此锁膨胀后虚拟机会做最后的争取，希望线程可以快速进入临界区而避免被操作系统挂起，一种较为有效的方式就是使用自旋锁。自旋锁可以使线程在没有获得锁时不被挂起，而转去执行一个空循环，在若干空循环后线程如果可以获得锁则继续执行，如果依旧不能获取才会挂起。JDK1.7以后不用设置参数，默认启用，且自选次数由虚拟机自行调整


















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