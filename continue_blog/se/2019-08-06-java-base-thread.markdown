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

![thread_status](thread_status.jpg)

线程状态可以分为：  
1、**新建**(new)  
2、**就绪状态**/可运行状态(Runnable)  
3、**运行状态**(Running)  
4、**阻塞状态**(Blocked)：等待阻塞(wait进入等待队列)、同步阻塞(同步锁被占用进入锁池)、其他阻塞(sleep/join/IO阻塞)  
5、**死亡状态**(Dead)

其中sleep方法会休眠一段时间(进入阻塞)，但**不释放**对象锁，到时间后进入就绪状态，并不保证能马上运行。`yield`方法类似让出资源(cpu时间片)，让优先级更高的线程执行，自己不会阻塞而是进入**就绪**状态，让系统的线程调度器重新调度器重新调度一次，因此也有可能之后还是自己执行。`wait`方法则会让线程进入阻塞状态，当前线程释放对象锁，进入等待队列，直到有线程`notify/notifyAll`唤醒。而当一个线程需要等另一个(或多个)线程先执行完再继续执行时可以使用join方法，当前线程阻塞，但不释放对象锁。使用interrupt方法可以中断线程，将"中断标记"设置为true，如果该线程正处于阻塞状态则将"中断标记"立即清除为"false"并抛出InterruptedException异常。`interrupted`方法判断当前线程是否处于中断状态，返回后会清除中断状态，而`isInterrupted`方法则只是判断中断状态并不会清除

守护线程，可以在创建线程后将其`setDaemon(true)`即可。比如JVM的垃圾回收。内存管理都是守护线程等，当所有线程退出后，守护线程也将结束



# 多线程

并发(concurrency)和并行(parallellism)都是完成多任务更加有效率的方式，但还是有一些区别的：  
1、并发：**交替**做不同事情的能力，**CPU时间片**执行，体现在不同的代码块交替执行，强调在一个时间段内同时执行  
2、并行：**同时**做不同事情的能力，**多核**执行，体现在不同的代码块同时执行

对于多线程执行，如果没有共享变量，那大家都相安无事，相当于各个单线程执行，线程安全。而如果多个线程间有共享变量，又由于Java内存模型的通信机制，就会发生数据不同步、死锁等问题

+ 上下文切换
	- 线程从运行状态切换到阻塞状态或者等待状态的时候需要将线程的运行状态保存，线程从阻塞状态或者等待状态切换到运行状态的时候需要加载线程上次运行的状态
	- 线程的运行状态从保存到再加载就是一次上下文切换，而上下文切换的开销是非常大的，而我们知道CPU给每个线程分配的时间片很短，通常是几十毫秒(ms)，那么线程的切换就会很频繁
+ 死锁
+ 资源限制的挑战
	- 计算机硬件资源或软件资源限制了多线程的运行速度

#### 伪共享

为了解决计算器中主内存与CPU之间运行速度差的问题，会在CPU与主内存中添加一级或多级高速缓冲存储器(cache)(L1/L2)，这个cache是集成在CPU内部的，所以也叫CPU Cache。在Cache内部是按行存储的，其中一行成为一个Cache行，其是Cache与主内存进行交换数据的单位，Cache行的大小一般为2的幂次数字节。当CPU访问某个变量时，会先去CPU Cache查看，如果没有再到主内存获取，并将该变量所在内存区域的一个Cache行大小的内存复制进Cache中，由于存放在Cache行的是内存块而不是变量，所以可能有多个变量存储到一个Cache行中。如果多个线程同时修改一个缓存行里的多个变量时，由于同时只能有一个线程操作缓存行，所以相比将每个变量放到一个缓存行，性能就会有所下降，这就是伪共享

伪共享的产生原因是多个变量放在一个缓存行，这在单线程下对代码执行是更有利的，执行更快，多线程下相反。JDK 8之前是通过字节填充的方式解决的，创建一个变量时使用填充字段填充其缓存行，避免同一个缓存行中有多个变量。在JDK 8提供了一个sun.misc.Contended注解来解决伪共享问题，需要注意的是这个注解默认只使用于Java核心类，比如rt包下的类，我们自己创建的类要使用则需要`-XX:-RestrictContended`自定义宽度


## CAS

CAS即compare and swap，是JDK提供的非阻塞原子性操作，它通过硬件保证了比较 —— 更新操作的原子性。Unsafe类中有3个`compareAndSwap*`方法。JDK的rt.jar包的Unsafe类提供了硬件级别的原子性操作，其中的方法都是native方法，使用JNI的方式访问本地C++实现库，提供的这些功能包括裸内存的申请/释放/访问，低层硬件的atomic/volatile支持，创建未初始化对象等。扩展了Java语言表达能力，便于在高层Java代码里实现原本在更底层C中实现的核心库功能。许多广泛使用的高性能开发库都是基于Unsafe类开发，比如 Netty、Hadoop、Kafka 等

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

CAS操作是乐观锁，每次不加锁而是假设没有冲突去完成某项操作，如果因为冲突失败就重试，直到成功为止。而synchronized就是悲观锁，线程一旦得到锁，其他需要锁的线程会挂起。尽管JDK 1.6为Synchronized做了优化，增加了从偏向锁到轻量级锁再到重量级锁的过度，但是在最终转变为重量级锁之后，性能仍然较低

CAS机制当中使用了3个基本操作数：内存地址V，旧的预期值A，要修改的新值B。更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。无论哪种情况，都会在CAS指令之前返回该位置的值(一些特殊情况下将仅返回CAS是否成功)

CAS的问题：  
1、在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很大的压力  
2、只保证一个共享变量的原子性操作，不能保证代码块/多个共享变量的原子性  
3、ABA问题(解决思路：使用版本号)


## 原子操作类

原子操作，有针对CPU指令级别的，和针对高级语言级别的。原子操作实质并不是不可分割，本质在于多个资源之间有一致性的要求，操作的中间态对外不可见

concurrent包中提供了一些原子类，在其atomic包下，例如AtomicInteger、AtomicLong、AtomicIntegerArray、AtomicReference、LongAdder等

```java
// AtomicInteger
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
private volatile int value;

static {
	try {
		valueOffset = unsafe.objectFieldOffset
			(AtomicInteger.class.getDeclaredField("value"));
	} catch (Exception ex) { throw new Error(ex); }
}

public final int getAndSet(int newValue) {
	return unsafe.getAndSetInt(this, valueOffset, newValue);
}

public final boolean compareAndSet(int expect, int update) {
	return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
由此可见，内部就是使用了volatile + CAS保证的。根据对象和偏移量获取当前值，与希望的值比较，相等的话进行更新操作

```java
// AtomicIntegerArray
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final int base = unsafe.arrayBaseOffset(int[].class);
private static final int shift;
private final int[] array;

static {
	int scale = unsafe.arrayIndexScale(int[].class);
	if ((scale & (scale - 1)) != 0)
		throw new Error("data type scale not a power of two");
	shift = 31 - Integer.numberOfLeadingZeros(scale);
}

private long checkedByteOffset(int i) {
	if (i < 0 || i >= array.length)
		throw new IndexOutOfBoundsException("index " + i);
	return byteOffset(i);
}

private static long byteOffset(int i) {
	return ((long) i << shift) + base;
}

public final int getAndSet(int i, int newValue) {
	return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
}
```
数组的略有不同，因为要计算出每个元素正确的偏移量

JDK 8新增的原子操作类LongAdder等。对于以前非阻塞的原子性操作类(AtomicLong等)在高并发下大量线程竞争更新同一个原子变量，不断循环尝试CAS，浪费CPU性能。新增的原子性递增或者递减类LongAdder用来克服在高并发下使用AtomicLong的缺点

LongAdder类与AtomicLong类的区别在于高并发时前者将对单一变量的CAS操作分散为对数组cells中多个元素的CAS操作，取值时进行求和；而在并发较低时仅对base变量进行CAS操作，与AtomicLong类原理相同

```java
transient volatile Cell[] cells;
transient volatile long base;

// Contended解决伪共享
@sun.misc.Contended static final class Cell {
	volatile long value;
	Cell(long x) { value = x; }
	final boolean cas(long cmp, long val) {
		return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
	}

	// Unsafe mechanics
	private static final sun.misc.Unsafe UNSAFE;
	private static final long valueOffset;
	static {
		try {
			UNSAFE = sun.misc.Unsafe.getUnsafe();
			Class<?> ak = Cell.class;
			valueOffset = UNSAFE.objectFieldOffset
				(ak.getDeclaredField("value"));
		} catch (Exception e) {
			throw new Error(e);
		}
	}
}
```

修改值：
```java
public void add(long x) {
	Cell[] as; long b, v; int m; Cell a;
	// 并发量不大时，直接CAS，与AtomicLong一样
	if ((as = cells) != null || !casBase(b = base, b + x)) {
		boolean uncontended = true;
		// 当竞争激烈到一定程度无法对base进行累加操作时，会对cells数组中某个元素进行更新
		if (as == null || (m = as.length - 1) < 0 ||
			(a = as[getProbe() & m]) == null ||
			!(uncontended = a.cas(v = a.value, v + x)))
			// 用一个死循环对cells数组中的元素进行操作：
			// 当要更新的位置的元素为空时插入新的cell元素，否则在该位置进行CAS的累加操作
			// 如果CAS操作失败并且数组大小没有超过核数就扩容cells数组当要更新的位置的元素为空时插入新的cell元素
			// 否则在该位置进行CAS的累加操作，如果CAS操作失败并且数组大小没有超过核数就扩容cells数组
			longAccumulate(x, null, uncontended);
	}
}

// 对Striped64的BASE的值(base值所对应的内存偏移量)进行累加并返回是否成功，
final boolean casBase(long cmp, long val) {
	return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```
取值：
```java
public long longValue() {
	return sum();
}

// 返回的是base和cells数组中所有元素的和
public long sum() {
	Cell[] as = cells; Cell a;
	long sum = base;
	if (as != null) {
		for (int i = 0; i < as.length; ++i) {
			if ((a = as[i]) != null)
				sum += a.value;
		}
	}
	return sum;
}
```

## 锁

锁分类：
+ 按处理方式
	- 乐观锁：相对于悲观锁而言，采取了更加宽松的加锁机制
		+ CAS
		+ 版本号控制
	- 悲观锁：具有强烈的独占和排他特性
		+ 关系型数据库的行锁，表锁，读锁，写锁
		+ synchronized
+ 按抢占方式
	- 公平锁
		+ 队列FIFO
	- 非公平锁
		+ 默认ReentrantLock对象
+ 按持有
	- 独占锁：排他锁，写锁，不能与其他锁并存
	- 共享锁：读锁，可共享一把锁访问数据，但是不能修改

选择因素：
+ 响应效率
+ 冲突频率
+ 重试代价

### synchronized

+ 可见性：语义保证了共享变量的可见性
	- 线程加锁前：需要将工作内存清空，从而保证了工作区的变量副本都是从主存中获取的最新值
	- 线程解锁前；需要将工作内存的变量副本写回到主存中
+ 有序性：保证了有序性
	- 使用阻塞的同步机制，共享变量只能同时被一个线程操作，所以JMM不用像volatile那样考虑加内存屏障去保证synchronized多线程情况下的有序性，因为CPU在单线程情况下是保证了有序性的
+ 原子性：保证了原子性
	- 使用阻塞的同步机制，共享变量加锁了，在对共享变量进行读/写操作的时候是原子性的

**synchronized同步代码块**是通过加`monitorenter`和`monitorexit`指令实现的
	
> 每个对象都有个监视器锁(monitor)，当monitor被占用的时候就代表对象处于锁定状态，而monitorenter指令的作用就是获取 monitor的所有权(计数为1，锁重入+1)，monitorexit的作用是释放monitor的所有权(计数-1直到0)
	
**synchronized同步方法**，常量池中比普通的方法多了个`ACC_SYNCHRONIZED`标识，JVM就是根据这个标识来实现方法的同步。当调用方法的时候，调用指令会检查方法是否有ACC_SYNCHRONIZED标识，有的话线程需要先获取monitor，获取成功才能继续执行方法，方法执行完毕之后，线程再释放monitor，同一个monitor同一时刻只能被一个线程拥有

所以，synchronized同步代码块是需要JVM通过字节码显式的去获取和释放monitor实现同步；而synchronized同步方法只是检查ACC_SYNCHRONIZED标志是否被设置，不需要JVM去显式的实现。实际上这两个同步方式实际都是通过获取monitor和释放monitor来实现同步的，而monitor的实现依赖于底层操作系统的**mutex**互斥原语，而操作系统实现线程之间的切换的时候需要从用户态转到内核态，这个转成过程开销比较大
	

	
	


java虚拟机的实现中每个对象都有一个对象头，用于保存对象的系统信息。对象头中有一个mark word的部分，它是实现锁的关键。在32位系统中，它为32位的数据；在64位系统中，它为64位的数据。它是一个多功能的数据区，可以存放对象的哈希值、对象年龄、锁的指针等信息。一个对象是否占用锁，占有哪个锁，就记录在这个mark word中

以32位系统为例，普通对象的对象头比如：  
`hash:25 ------------>| age:4 biased_lock:1 lock:2`  
它表示mark word中有25位比特表示对象的哈希值，4位比特表示对象的年龄，1位比特表示**是否偏向锁**，2位比特表示**锁的信息**

当对象处于普通的**未锁定**状态时，格式为:  
`[header      |0|01] unlocked`  
前29位表示对象的哈希值、年龄等信息。倒数第3位位为0，最后两位为01，表示未锁定，与偏向锁最后2位相同，因此虚拟机通过**倒数第3位**比特来区分是否为偏向锁

**偏向锁**是程序没有竞争时、取消之前已经获得锁的线程同步操作，即某一锁被线程获取后，进入偏向模式，CAS操作在对象头存储锁偏向的**线程ID**，该线程再次请求这个锁时，无需进行相关同步操作(CAS操作来加锁和解锁)，只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。对于偏向锁的对象，其对象头记录获得锁的线程：  
`[javaThread* | epoch | age |1|01]`  
前23位表示持有偏向锁的线程，后续2位表示偏向锁的时间戳，4位比特表示对象年龄，年龄后1位为1表示偏向锁，最后2位位01表示可偏向/未锁定。当获得锁的线程再次尝试获取锁时，通过mark word的线程信息可以判断当前线程是否持有偏向锁。当然在锁竞争激烈的场景下反而得不到优化，可以使用`-XX:-UseBiasedLocking`禁用偏向锁

![basicObjectLock](basicObjectLock.png)

偏向锁失败后，java虚拟机让线程申请**轻量级锁**。轻量级锁在虚拟机内部使用一个称为**BasicObjectLock**的对象实现，这个对象内部由一个**BasicLock对象**和一个**持有该锁的Java对象指针**组成。BaseicObjectLock对象放置在Java栈的栈帧中，在BasicLock对象内部还维护着**displaced_header**字段，它用于备份对象头部的**mark word**。当对象处于轻量级锁定时其头部mark word为:  
`[ptr         |00] locked`  
末尾2位为00。此时整个mark word为指向BasicLock对象的**指针**，由于BasicObjectLock对象在线程栈中，因此该指针必然**指向持有该锁的线程栈空间**，即存放在获得锁的线程栈中的该对象真实对象头。当需要判断某一线程是否持有该对象锁时，也只需简单判断**对象头的指针**是否在当前线程的栈地址范围即可。同时BasicLock对象的displaced_header备份了原对象的mark word内容。BasicObjectLock对象的obj字段则指向该对象
```
markOop mark = obj -> mark();
lock -> set_displaced_header(mark);
if (mark == (markOop)Atomic::cmpxchg_ptr(lock, ojb()->mark_addr(), mark)) {
	TEVENT(slow_enter: release stacklock);
	return;
}
```
首先BasicLock通过set_displaced_header方法备份原对象的mark word。然后使用CAS操作尝试将BasicLock的地址复制到对象头的mark word，如果复制成功则枷锁成功，如果失败则认为加锁失败，那么轻量级锁有可能膨胀为重量级锁

![weight-lock](weight-lock.jpg)

+ Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。它会根据对象的状态复用自己的存储空间
+ Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例

当轻量级锁失败，虚拟机就会使用**重量级锁**，当对象处于重量级锁定时对象的mark word为：  
`[ptr         |10] monitor`  
末尾2位为10。整个mark word表示**指向monitor对象的指针**，在轻量级锁处理失败后，虚拟机会执行以下操作：  
```
lock -> set_displaced_header(markOopDesc::unused_mark());
ObjectSynchronizer::inflate(THREAD, obj())-> enter(THREAD);
```
首先**废弃**前面BasicLock备份的对象头信息，然后正式启用重量级锁。启用过程分为两步：首先通过`inflate()`方法进行**锁膨胀**，其目的是获得对象的**ObjectMonitor**，然后使用`enter()`方法尝试**进入该锁**。在`enter()`方法调用中，线程很可能会在操作系统层面被挂起，如果这样，线程间切换和调度的成本会比较高。因此锁膨胀后虚拟机会做最后的争取，希望线程可以快速进入**临界区**而避免被操作系统挂起，一种较为有效的方式就是使用**自旋锁**。自旋锁可以使线程在没有获得锁时不被挂起，而转去执行一个空循环，在若干空循环后线程如果可以获得锁则继续执行，如果依旧不能获取才会挂起。JDK1.7以后不用设置参数，默认启用，且自选次数由虚拟机自行调整


#### 锁分离

在某些情况下，可以对一组独立对象上的锁进行分解，这种情况称为锁分段。例如JDK 7的ConcurrencyHashMap是有一个包含16个锁的数组实现，每个锁保护所有散列桶的1/16，其中第N个散列桶由第(N mod 16)个锁来保护。假设所有关键字都均匀分布，那么相当于把锁的请求减少到原来的1/16

锁分段的劣势在于，与采用单个锁来实现独占访问相比，要获取多个锁来实现独占访问将更加困难并且开销更高，比如计算size、重hash










### AQS







深入分析synchronized原理(二)
http://www.liuhaihua.cn/archives/561521.html

深入分析Synchronized原理(阿里面试题)
https://www.cnblogs.com/aspirant/p/11470858.html


深入分析synchronized原理和锁膨胀过程(二)
https://www.lagou.com/lgeduarticle/63370.html

https://ddnd.cn/2019/03/22/java-synchronized-2/

分布式锁实现：数据库、redis、zookeeper
https://www.cnblogs.com/duanxz/p/3509423.html




## 线程池






Java 并发编程之 ReentrantLock 源码分析
http://www.liuhaihua.cn/archives/707946.html



## 其他


### threadlocal

threadlocal / inheritableThreadLocal  继承

Random  next随机种子 原子变量 /   threadlocalRandom 

ThreadLocal源码分析
https://blog.csdn.net/u011497638/article/details/93889173

使用ThreadLocal 不当可能会导致内存泄漏



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

 

## AQS

ArrayBlockingQueue
https://www.infoq.cn/article/jdk1.8-abstractqueuedsynchronizer
https://www.infoq.cn/article/java8-abstractqueuedsynchronizer/





## 使用

并发编程实战（一） logback 异步日志打印模型中ArrayBlockingQueue 的使用、Tomcat 的 NIOEndPoint 中 ConcurrentLinkedQueue 的使用
https://blog.csdn.net/weixin_41750142/article/details/110221334


SimpleDateFormat 是线程不安全的
使用Timer 时需要注意的事情

对需要复用但是会被下游修改的参数要进行深复制
创建线程和线程池时要指定与业务相关的名称
使用线程池的情况下当程序结束时记得调用shutdown 关闭线程池
线程池使用FutureTask 时需要注意的事情




