---
layout:     post
title:      "从dubbo-demo初探dubbo源码"
date:       2018-07-16 23:00
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - java
    - rpc
--- 

<font id="last-updated">最后更新于：2018-07-17</font>

# 准备工作

源码从[github](https://github.com/apache/incubator-dubbo.git)上clone

使用zookeeper作为注册中心，从[官网](https://zookeeper.apache.org/)下载

通过maven构建项目，从[官网](https://maven.apache.org/)下载

## dubbo-demo结构

+ dubbo-demo-api
	- I:DemoService
+ dubbo-demo-consumer
	- Consumer
	- dubbo-demo-consumer.xml
	- dubbo.properties
	- log4j.properties
+ dubbo-demo-provider
	- DemoServiceImpl
	- Provider
	- dubbo-demo-provider.xml
	- dubbo.properties
	- log4j.properties

# Provider

首先修改dubbo-demo-provider.xml配置文件，将dubbo:registry设置为`zookeeper://127.0.0.1:2181`  
启动Provider的main方法。

1. spring的ClassPathXmlApplicationContext读取xml配置。调用到refresh()方法。
2. 










# Consumer




















