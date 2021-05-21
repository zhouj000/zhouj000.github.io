---
layout:     post
title:      "重看RocketMQ源码(二)"
date:       2021-05-10
author:     "ZhouJ000"
header-img: "img/in-post/2021/post-bg-2021-headbg.jpg"
catalog: true
tags:
    - mq
--- 


# Producer

MessageQueue是RocketMq的一种**数据分片**+**物理存储**机制

![msgqueue1](msgqueue1.png)

一般在创建Topic的时候会指定MessageQueue的数量。如上图中，一个Topic中有4个MessageQueue，每个Brokers上有2个MessageQueue，生产者通过算法(默认是均匀分配//负载均衡)来把消息写入不同的MessageQueue中。MessageQueue的数据可以持久化在磁盘上。这样就把消息分散到了多个Broker上，大大提升Broker的抗并发能力

Producer通过NameSever获取指定Topic的Broker路由信息，并在本地保存一份缓存数据，比如一个Topic有哪些MessageQueue，MessageQueue在哪几台Broker上，Broker的ip、port等等。Producer发送消息只发到Master Broker上，Slave通过主从同步获取数据


































