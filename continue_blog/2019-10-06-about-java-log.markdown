---
layout:     post
title:      "Java日志"
date:       2019-05-06
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - log
--- 



在程序运行起来后，当需要了解程序的行为逻辑和进行调试查看时，使用日志打印是很好的一种方法

1、首先要记录日志，那么需要一个类表达日志的概念，且至少有两个属性：时间戳、消息本身，就把它叫做LoggingEvent，记录日志就像记录一个事件  
2、其次日志可以输出到不同的地方，比如控制台、文件、邮件等等，这个抽象一下可以叫做Appender  
3、然后日志内容可以格式化，那么就定义一个Formatter接口去格式化消息  
4、最后定义一个Logger作为一个句柄与Appender关联，并且可以拥有多个Appender，使得可以按包名或类名进行区分  
5、剩下定义一下日志分级，即可以在Logger和Appender上定义一个Priority，使得对应级别的日志进行输出  
一个简单的日志框架就设计好了，其中Logger、Appender、Formatter可以任意扩展而不互相影响，可以任意组合。这时候如果想使用其他日志工具，怎么切换比较方便呢？那就要提供一个抽象层，我们可以用抽象层的API来写日志，底层具体使用哪种日志工具并不关心，这个抽象就是Simple Logging Facade for Java，简称SLF4J

对于Log4j、JDK Logging、tinylog等工具需要一个适配层，将SLF4J的API转换为具体工具的调用接口实现，而Logback是直接实现了SLF4J的，因此不需要适配层，效率最高。因此SL4FJ + Logback的组合现在被广泛使用，大有超越Apache Common Logging + Log4j之势















# logback

# slf4j

# log4j




































