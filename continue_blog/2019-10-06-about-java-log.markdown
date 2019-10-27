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



在程序运行起来后，当需要了解程序的行为逻辑来对比预期时，使用日志打印是很好的一种方法：  
1、首先要记录日志，那么需要一个类表达日志的概念，且至少有两个属性：时间戳、消息本身。记录日志就像记录一个事件，那么就叫做LoggingEvent  
2、其次日志可以输出到不同的地方，比如控制台、文件、邮件等等，这个抽象一下可以叫做Appender(/Handler)，意为可以不断追加日志  
3、然后日志内容可以格式化，那么就定义一个Formatter(/Layout)接口去格式化消息，Appender引用Formatter  
4、最后定义一个Logger作为一个句柄与Appender关联，并且可以拥有多个Appender，使得可以按包名或类名进行区分  
5、剩下定义一下日志分级，即可以在Logger和Appender上定义一个Priority，使得对应级别的日志进行输出  
一个简单的日志框架就设计好了，其中Logger、Appender、Formatter可以任意扩展而不互相影响，可以任意组合(正交的概念)。这时候如果想使用其他日志工具，怎么切换比较方便呢？那就要提供一个抽象层，我们可以用抽象层的API来写日志，底层具体使用哪种日志工具并不关心，这个抽象就是Simple Logging Facade for Java，简称SLF4J
![log](log.png)

对于Log4j、JDK Logging、tinylog等工具需要一个适配层，将SLF4J的API转换为具体工具的调用接口实现，而Logback是直接实现了SLF4J的，因此不需要适配层，效率最高。因此SL4FJ(门面) + Logback的组合现在被广泛使用，大有超越Apache Common Logging(门面) + Log4j之势


# 日志框架

日志系统几乎是所有库都会用到的一个功能，每个库由于早期的技术选型和开发者喜好等原因，可能使用了不同的日志框架。我们在引入不同Maven库时，不知不觉间接引入了多种不同的日志框架。日志系统往往会尽可能早的进行初始化，并且由于日志桥接器和日志门面的存在，会尝试做一些绑定和劫持工作，一旦引入多个日志框架，轻则会导致程序中有好几套日志系统同时工作，日志输出混乱，重则会导致项目日志系统初始化死锁，项目无法启动。因此使用日志系统首先就要确定需要使用的框架

常用的日志门面框架有slf4j和commons-logging，门面系统能很大程度缓解日志系统的混乱，而在很多日志库的开发者也意识到日志门面系统的重要性，不在库中直接使用具体的日志实现框架。slf4j作为现代的日志门面系统，已经成为事实的标准，并且为其他日志系统做了十足的兼容工作，其实很多库还都自己造一个类似slf4j的日志门面系统，只是绑定实现的优先级不一样

如果我们直接暴力的排除其他日志框架，可能导致第三方库在调用日志接口时抛出ClassNotFound异常，这里就需要用到日志系统桥接器。比如log4j-over-slf4j，即log4j -> slf4j的桥接器，这个库定义了与log4j一致的接口(包名、类名、方法签名均一致)，但是接口的实现却是对slf4j日志接口的包装，即间接调用了slf4j日志接口，实现了对日志的转发。其中jul-to-slf4j是个例外，毕竟JDK自带的logging包排除不掉，该桥接器利用了JDK-logging的Handler机制，在root logger上install一个handler，将所有日志都劫持到slf4j上。要使jul-to-slf4j生效，除了spring boot日志模块初始化包含了该逻辑，使用其他框架时需要在入口处static执行逻辑，尽早初始化：
```
SLF4JBridgeHandler.removeHandlersForRootLogger();
SLF4JBridgeHandler.install();
```

桥接器解决问题，也可能带来问题。有些第三方库作者在使用日志时也会遇到同样的问题，他通过日志系统桥接器解决了，但是如果恰巧和你桥接的目标不同，比如log4j -> slf4j，slf4j -> log4j两个桥接器同时存在，就会造成互相委托，无限循环，堆栈溢出；又比如slf4j -> logback，slf4j -> log4j两个桥接器同时存在，那么由于slf4j的优先顺序是logback在前，这只是报警；而如果一些框架自定义了日志门面，且对绑定日志实现定义的优先级顺序与slf4j不一致，这可能会使程序产生2套日志系统工作。因此为了达成统一日志框架的目的，假设选定slf4j与logback需要：  
1、首先引入slf4j与logback日志包，slf4j -> logback桥接器  
2、排除common-logging、log4j、log4j2等日志包  
3、引入jdk-logging -> slf4j、common-logging -> slf4j、log4j -> slf4j、log4j2 -> slf4j桥接器  
4、排除slf4j -> jdk-logging、slf4j -> common-logging、slf4j -> log4j、slf4j -> log4j2桥接器  
PS.其中log4j2桥接器由log4j2提供，其他桥接器由slf4j提供。logback天生绑定slf4j，因此不需要桥接器  
PS.Slf4j介于Maven没有Gradle的全局依赖排除机制，由maven的路径最短优先原则和优先声明原则，提供version99仓库排除其他的日志框架(虚包)
![]()


## log4j

log4j由3个重要组件构成，分别是日志级别(从高到低ERROR、WARN、 INFO、DEBUG)、日志输出目的地(控制台、文件..)、日志的输出格式。

log4j可以使用配置文件，也可以使用JavaConfig，但是一般会使用配置文件.properties：
```
// 1、配置根Logger
// 分别是日志记录优先级(推荐使用ERROR、WARN、INFO、DEBUG这4种) 和 日志输出目的地
log4j.rootLogger = [level ... ],[appenderName ... ]


// 2、定义日志输出目的地Appender
/** log4j提供的Appender有：
  * org.apache.log4j.ConsoleAppender：控制台
  * org.apache.log4j.FileAppender：文件
  * org.apache.log4j.DailyRollingFileAppender：每天产生一个日志文件
  * org.apache.log4j.RollingFileAppender：滚动日志文件，文件大小到达指定大小时产生一个新的文件
  * org.apache.log4j.WriterAppender：将日志信息以流格式发送到任意指定的地方
  */
log4j.appender.[yourAppenderName] = [fully.qualified.name.of.appender.class]
log4j.appender.[yourAppenderName].[option0] = [value0]
...
log4j.appender.[yourAppenderName].[optionN] = [valueN]


// 3、配置日志格式(布局)
/** log4j提供的layout有：
  * org.apache.log4j.HTMLLayout：以HTML表格形式布局
  * org.apache.log4j.PatternLayout：可以灵活地指定布局模式
  * org.apache.log4j.SimpleLayout：包含日志信息的级别和信息字符串
  * org.apache.log4j.TTCCLayout：包含日志产生的时间、线程、类别等等信息
  *
  * 打印格式：
  * %m：输出代码中指定的消息
  * %p：输出优先级，即DEBUG，INFO，WARN，ERROR，FATAL
  * %d：输出日志时间点的日期或时间，比如：%d{yyy MMM dd HH:mm:ss,SSS}
  * %r：输出自应用启动到输出该log信息耗费的毫秒数
  * %l：输出日志事件的发生位置，包括类目名、发生的线程，以及在代码中的行数
  * %c：输出所属的类目，通常就是所在类的全名
  * %t：输出产生该日志事件的线程名
  * %n：输出一个回车换行符，Windows平台为"rn"，Unix平台为"n"
  */
log4j.appender.[yourAppenderName].layout = [fully.qualified.name.of.layout.class]
log4j.appender.[yourAppenderName].layout.ConversionPattern = [your pattern]
```
在设置好log4j配置后，就要在代码中对它进行使用了：
```
// 0、初始化，读取配置，有3种方式：缺省的(Console)、读取Property配置、读取XML配置
BasicConfigurator.configure();
PropertyConfigurator.configure([configFilename]);
DOMConfigurator.configure([filename]);

// 1、获取日志记录器
public static Logger logger = Logger.getLogger([yourClass.class])

// 2、插入日志信息
Logger.debug([msg]);
Logger.info([msg]);
Logger.error([msg]);
```


















# slf4j

# logback

# commons logging


# log4j2










参考：  
[slf4j、jcl、jul、log4j1、log4j2、logback大总结](https://my.oschina.net/pingpangkuangmo/blog/410224)  

