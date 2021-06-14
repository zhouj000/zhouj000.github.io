---
layout:     post
title:      "Java基础SE(三) 线程与并发-补充"
date:       2021-05-24
author:     "ZhouJ000"
header-img: "img/in-post/2021/post-bg-2021-headbg.jpg"
catalog: true
tags:
    - java
--- 


[Java基础SE(一) 数据类型与关键字](https://zhouj000.github.io/2021/04/11/java-base-base/)  
[Java基础SE(二) 集合](https://zhouj000.github.io/2021/02/26/java-base-collections/)  
[Java基础SE(三) 线程与并发](https://zhouj000.github.io/2021/05/09/java-base-thread/)  
[Java基础SE(三) 线程与并发-补充](https://zhouj000.github.io/2021/05/24/java-base-thread2/)  
[Java基础SE(四) IO](https://zhouj000.github.io/2021/04/21/java-base-io/)  

[Java基础: JVM(九) Java内存模型](https://zhouj000.github.io/2019/07/09/java-base-jmm/)  
[Java性能优化02-并行优化](https://zhouj000.github.io/2019/01/08/java-optimize-02/)  



# 线程组

对相同性质的线程进行分组
```java
Runnable runnable // = ...;

ThreadGroup userGroup = new ThreadGroup("user");
Thread userTask1 = new Thread(userGroup, runnable, "user-task1");
Thread userTask2 = new Thread(userGroup, runnable, "user-task2");
userTask1.start();
userTask2.start();

System.out.println("线程组活跃线程数：" + userGroup.activeCount());
userGroup.list();
```

线程组还能统一设置组内所有线程的最高优先级，线程单独设置的优先级不会高于线程组设置的最大优先级`userGroup.setMaxPriority(Thread.MIN_PRIORITY);`

停止线程组内线程不推荐使用stop，使用`userGroup.interrupt();`中断线程组内的所有线程



