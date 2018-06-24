---
layout:     post
title:      "java项目 CPU占用100%问题"
date:       2018-06-24
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: false
tags:
    - java
    - 排查
    - 命令
--- 

<font color="#FFB6C1" size="1" face="黑体">最后更新与：2018-06-25</font>

>一个应用占用CPU很高，除了确实是计算密集型应用之外，通常原因都是出现了死循环。

# 1.确定Java应用进程

### jps

>一个显示当前所有java进程pid的命令

参数-m： 输出主函数main class传入的参数
````
jps -m

25650 Jps -m
```

参数-l： 输出应用程序main class的完整package名或者应用程序的jar文件完整路径名
```
jps -l

24582 sun.tools.jps.Jps
```

参数-v： 输出传递给JVM的参数
```
jps -v

24482 Jps -Denv.class.path=.:/usr/java/jdk1.7.0_79/lib/dt.jar:/usr/java/jdk1.7.0_79/lib/tool.jar -Dapplication.home=/usr/java/jdk1.7.0_79 -Xms8m
```


## Linux下查看Java应用中线程CPU占比

### top操作查看%CPU

> top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，常用于服务端性能分析

```
查看指定进程下的线程cpu占用比例，分析是具体哪个线程占用率过高：
top -p <PID> -H
```

top命令的结果分为两个部分：
+ 统计信息： 前五行是系统整体的统计信息
+ 进程信息： 统计信息下方类似表格区域显示的是各个进程的详细信息，默认5秒刷新一次

常用命令参数:
+ -u<用户名>： 指定用户名
+ -p<进程号>： 指定进程
+ -n<次数>： 循环显示的次数
+ -H： 显示进程的所有线程
+ -d： 屏幕刷新间隔时间

常用命令交互：
+ 基础操作
    - l(字母)： 切换显示平均负载和启动时间信息
	- m： 切换显示内存信息
	- t： 切换显示进程和CPU状态信息
	- c： 切换显示命令名称和完整命令行
	- 1(数字)： 显示CPU详细信息，每核显示一行
	- d / s： 修改刷新频率，单位为秒
	- < ENTER > / < SPACE >： 刷新显示
	- f： top进入另一个视图，在这里可以编排基本视图中的显示字段
	- h： 可显示帮助界面
+ 进程列表排序 
	- M： 根据驻留内存大小进行排序
	- P： 根据CPU使用百分比大小进行排序
	- T： 根据时间/累计时间进行排序
	- shift + > / shift + <： 向右或左改变排序列

常用字段：
+ 待


### ps命令查找进程与线程

```
ps -ef 是用标准的格式显示进程的(System V风格)
ps aux 是用BSD的格式来显示(BSD 风格)，aux会截断command列

ps aux | grep java

ps H -eo user,pid,ppid,tid,time,%cpu,cmd --sort=%cpu
ps -mp <pid> -o THREAD,tid,time
```

常用命令：
+ -A： 列出所有的行程 
+ a： 显示现行终端机下的所有程序，包括其他用户的进程
+ u： 以用户为主的格式来显示程序状况
+ x： 显示所有程序，不以终端机来区分，可列出较完整信息
+ e： 列出程序时，显示每个程序所使用的环境变量
+ -e： 此参数的效果和指定"A"参数相同
+ -m / m： 显示所有的执行绪
+ 输出格式：
	+ l / -l： 较长、较详细的将该PID 的的信息列出
	+ j / -j： 工作的格式 (jobs format)
	+ -f:  全格式输出，生成一个完整列表
	+ -o： 用户自定义格式

常用字段：
+ 待


### 补充
```
监控java线程数：
ps -eLf | grep java | wc -l

监控网络客户连接数：
netstat -n | grep tcp | grep <侦听端口> | wc -l

输出进程内存的状况，可以用来分析线程堆栈：
pmap <PID>

查看包含内存每个项目的报告，通过-S M或-S k可以指定查看的单位，默认为kb。结合watch命令就可以看到动态变化的报告;
vmstat -s -S M  

查看cpu波动情况，尤其是多核机器上：
mpstat -P ALL 10 

IO实时监控查看所有设备使用率、读写字节数等信息
iostat -P ALL  

IO实时监控只查看均值的：
iostat -c
```


## windows下查看后台进程信息

### pslist命令

```
pslist查看java进程详情：
pslist | findstr java

查看线程信息：
pslist -x <pid>
```


下载pstools压缩包，解压后复制到C:\Windows\System32目录下  

参数：
+ /?： 查看用法
+ -d:  显示线程详情
+ -m:  显示内存详情
+ -x:  显示线程和内存详情


### tasklist

> 该工具显示在本地或远程机器上当前运行的进程列表

```
tasklist | findstr java

tasklist /? 查看用法
```



# 2.查看线程信息

## 将tid转换为16进制

Linux下:
`printf "%x\n" <tid>`

Windows下：
`计算器进行转换`

## jstack查看某进程的当前线程栈运行情况(thread dump)

> jstack是java虚拟机自带的一种堆栈跟踪工具。jstack用于打印出给定的java进程ID或core file或远程调试服务的Java堆栈信息，如果是在64位机器上，需要指定选项"-J-d64"，Windows的jstack使用方式只支持以下的这种方式：jstack [-l] pid

```
jstack <pid> | grep <tid> -A 30

jstack [-l] <pid> > ./jstack.log
```

主要分为两个功能： 
- 针对活着的进程做本地的或远程的线程dump
- 针对core文件做线程dump

jstack用于生成java虚拟机当前时刻的**线程快照**。线程快照是**当前java虚拟机内每一条线程正在执行的方法堆栈的集合**，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如**线程间死锁、死循环、请求外部资源导致的长时间等待等**。 线程出现停顿的时候通过jstack来查看各个线程的**调用堆栈**，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。

```
jstack [ option ] pid
jstack [ option ] executable core
jstack [ option ] [server-id@]remote-hostname-or-IP
```

基本参数：
+ -l： 长列表. 打印关于锁的附加信息，会使得JVM停顿得长久得多。一般情况不需要使用
+ -m： 打印java和native c/c++框架的所有栈信息，可以打印JVM的堆栈，显示上Native的栈帧，一般应用排查不需要使用

#### 线程状态

>**NEW**： 未启动的。不会出现在Dump中。  
**RUNNABLE**： 在虚拟机内执行的。运行中状态，可能里面还能看到locked字样，表明它获得了某把锁。  
**BLOCKED**： 受阻塞并等待监视器锁。被某个锁(synchronizers)給block住了。  
**WATING**： 无限期等待另一个线程执行特定操作。等待某个condition或monitor发生，一般停留在park(), wait(), sleep(),join() 等语句里。  
**TIMED_WATING**： 有时限的等待另一个线程的特定操作。和WAITING的区别是wait() 等语句加上了时间限制 wait(timeout)。  
**TERMINATED**： 已退出的。

#### Monitor

Monitor是Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者Class的锁。每一个对象都有，也仅有一个monitor。下 面这个图，描述了线程和Monitor之间关系，以及线程的状态转换图:
![java Monitor](/img/in-post/2018/java-monitor.jpg)

>**进入区(Entrt Set)**： 表示线程通过synchronized要求获取对象的锁。如果对象未被锁住,则迚入拥有者;否则则在进入区等待。一旦对象锁被其他线程释放,立即参与竞争。  
**拥有者(The Owner)**： 表示某一线程成功竞争到对象锁。  
**等待区(Wait Set)**： 表示线程通过对象的wait方法,释放对象的锁,并在等待区等待被唤醒。

#### 调用修饰

表示线程在方法调用时,额外的重要的操作。线程Dump分析的重要信息。修饰上方的方法调用。
>**locked &lt;地址> 目标**： 通过synchronized关键字,成功获取到了对象的锁,成为监视器的拥有者,在临界区内操作。对象锁是可以线程重入的。  
**waiting to lock &lt;地址> 目标**： 通过synchronized关键字,没有获取到了对象的锁,线程在监视器的进入区等待。在调用栈顶出现,线程状态为**Blocked**。  
**waiting on &lt;地址> 目标**： 通过synchronized关键字,成功获取到了对象的锁后,调用了wait方法,进入对象的等待区等待。在调用栈顶出现,线程状态为**WAITING**或**TIMED_WATING**。  
**parking to wait for &lt;地址> 目标**： park是基本的线程阻塞原语,不通过监视器在对象上阻塞。随concurrent包会出现的新的机制,不synchronized体系不同。

#### 线程动作

线程状态产生的原因：
>**runnable**： 状态一般为RUNNABLE。  
**in Object.wait()**： 等待区等待,状态为WAITING或TIMED_WAITING。  
**waiting for monitor entry**： 进入区等待,状态为BLOCKED。  
**waiting on condition**： 等待区等待、被park。  
**sleeping**： 休眠的线程,调用了Thread.sleep()。

Wait on condition 该状态出现在线程等待某个条件的发生。具体是什么原因，可以结合stacktrace来分析。 最常见的情况就是线程处于sleep状态，等待被唤醒；还有等待网络IO。

#### 线程dump的分析工具

+ [IBM Thread and Monitor Dump Analyze for Java](https://www.ibm.com/developerworks/community/groups/service/html/communitystart?communityUuid=2245aa39-fa5c-4475-b891-14c205f7333c)  一个小巧的Jar包，能方便的按状态，线程名称，线程停留的函数排序，快速浏览。
+ [http://spotify.github.io/threaddump-analyzer](http://spotify.github.io/threaddump-analyzer)  Spotify提供的Web版在线分析工具，可以将锁或条件相关联的线程聚合到一起。


# 3.模拟实战分析

待







