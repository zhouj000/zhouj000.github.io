---
layout:     post
title:      "RocketMQ源码(一) "
date:       2021-01-05
author:     "ZhouJ000"
header-img: "img/in-post/2020/post-bg-2020-headbg.jpg"
catalog: true
tags:
    - mq
--- 



# 概念

![]()

+ **异步**：在一对多调用时由消息系统通知。如下单核心流程环节太多，性能较差
+ **解耦**：和第三方系统耦合在一起，性能存在抖动的风险。解决不同重要程度/能力级别系统之间依赖
+ **削峰填谷**：解决瞬时写压力大于应用服务能力导致消息丢失、系统奔溃等问题，例如秒杀活动
+ **失败重试**：业务调用失败风险
+ **延时消息**：比如关闭过期订单的时候，存在扫描大量订单数据的问题
+ **监听BinLog发送到MQ**：数据同步到其他NoSQL

RocketMQ优势：
+ 支持事务型消息：消息发送和DB操作保持两方的最终一致性
	- rabbitmq和kafka不支持
+ 支持结合RocketMQ的多个系统之间数据最终一致性
	- 前提：多方事务，二方事务
+ 支持18个级别的延迟消息
	- rabbitmq和kafka不支持
+ 支持指定次数和时间间隔的失败消息重发
	- kafka不支持，rabbitmq需要手动确认
+ 支持consumer端tag过滤，减少不必要的网络传输
	- rabbitmq和kafka不支持
+ 支持重复消费
	- rabbitmq不支持，kafka支持

![]()

+ Name Server：是一个**几乎无状态**节点，可集群部署，节点之间**无任何信息同步**。每10s**定期清理**超过2分钟未上报心跳的broker
+ Broker：部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master 与Slave的对应关系通过指定**相同的**BrokerName，**不同的**BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master和Slave都可以部署多个。每个Broker与Name Server集群中的**所有**节点建立**长连接**，每隔30s**注册**Topic信息到所有Name Server
+ Producer：与Name Server集群中的其中一个节点(随机选择)建立长连接。每30s从Name Server取Topic路由信息，并向提供Topic服务的Broker **Master**建立**长连接**，每30s向Master发送心跳。Producer**完全无状态**，可集群部署
	- Producer每隔30s(由ClientConfig的pollNameServerInterval)从Name server获取所有topic队列的最新情况，这意味着如果Broker不可用，Producer最多30s能够感知，在此期间内发往Broker的所有消息都会**失败**
	- Producer每隔30s(由ClientConfig中heartbeatBrokerInterval决定)向所有关联的broker发送心跳，Broker每隔10s中扫描所有存活的连接，如果Broker在2分钟内没有收到心跳数据，则**关闭与Producer的连接**
+ Consumer：与Name Server集群中的其中一个节点(随机选择)建立长连接，每30s从Name Server取Topic路由信息，并向提供 Topic服务的Broker **Master/Slave**建立**长连接**，每30s向Master/Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定
	- Consumer每隔30s从Name server获取topic的最新队列情况，这意味着Broker不可用时，Consumer最多最需要30s才能感知
	- Consumer每隔30s(由ClientConfig中heartbeatBrokerInterval决定)向所有关联的broker发送心跳，Broker每隔10s扫描所有存活的连接，若某个连接2分钟内没有发送心跳数据，则**关闭连接**；并向该Consumer Group的**所有Consumer发出通知**，Group内的Consumer**重新分配队列**，然后继续消费
	- 当Consumer得到master宕机通知后，转向slave消费，slave不能保证master的消息100%都同步过来了，因此会有少量的消息**丢失**。但是一旦master恢复，未同步过去的消息会被**最终消费**掉



	
	
	

# 特色

## 事务


















https://www.jianshu.com/p/2838890f3284
https://blog.csdn.net/iie_libi/article/details/54236502
https://blog.csdn.net/initiallht/article/details/104745720
http://www.zyiz.net/tech/detail-139670.html
https://cloud.tencent.com/developer/article/1652750?from=article.detail.1645912
https://cloud.tencent.com/developer/article/1645910?from=article.detail.1652750


最佳部署实践：双主双从，同步复制异步刷盘  
异步刷盘ASYNC_FLUSH模式，flushCommitLogTimed改为true，否则还是会实时刷盘  
CommitLog异步刷盘默认刷盘间隔：500ms  
ConsumeQueue默认刷盘间隔：1s















