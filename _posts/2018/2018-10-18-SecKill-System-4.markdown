---
layout:     post
title:      "秒杀系统设计(四) 一致性-库存"
date:       2018-10-18
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 高并发
    - 架构
--- 

<font id="last-updated">最后更新于：2018-10-19</font>

[秒杀系统设计(一) 概述和原则](https://zhouj000.github.io/2018/10/14/SecKill-System-1)  
[秒杀系统设计(二) 高性能01-动静分离与热点缓存](https://zhouj000.github.io/2018/10/15/SecKill-System-2)  
[秒杀系统设计(三) 高性能02-流量削峰与服务端优化](https://zhouj000.github.io/2018/10/16/SecKill-System-3)  
[秒杀系统设计(四) 一致性-库存](https://zhouj000.github.io/2018/10/18/SecKill-System-4)  
[秒杀系统设计(五) 高可用-兜底方案](https://zhouj000.github.io/2018/10/19/SecKill-System-5)  



要设计一套秒杀系统，如果库存是100件，那就只能卖100件，减库存时不能超卖

# 减库存方式

在正常的电商平台购物场景中，用户的实际购买过程一般分为两步：下单和付款

## 下单减库存

即当买家下单后，在商品的总库存中减去买家购买数量。下单减库存是最简单的减库存方式，也是控制最精确的一种，下单时直接通过数据库的事务机制控制商品库存，这样一定不会出现超卖的情况。但是有个问题，有些人下完单后可能并不会付款

## 付款减库存

买家下单后，并不立即减库存，而是等到有用户付款后才真正减库存，否则库存一直保留给其他买家。但因为付款时才减库存，如果并发比较高，有可能出现买家下单后付不了款的情况，因为可能商品已经被其他人买走了

## 预扣库存

这种方式相对复杂一些，买家下单后，库存为其保留一定的时间（如10分钟），超过这个时间，库存将会自动释放，释放后其他买家就可以继续购买。在买家付款前，系统会校验该订单的库存是否还有保留：如果没有保留，则再次尝试预扣；如果库存不足（也就是预扣失败）则不允许继续付款；如果预扣成功，则完成付款并实际地减去库存



# 减库存出现的问题

由于购物过程中存在两步或者多步的操作，因此在不同的操作步骤中减库存，就会存在一些可能被恶意买家利用的漏洞，例如发生恶意下单的情况

假如我们采用**“下单减库存”**的方式，即用户下单后就减去库存，正常情况下，买家下单后付款的概率会很高，所以不会有太大问题。但是有一种场景例外，就是当卖家参加某个活动时，此时活动的有效时间是商品的黄金售卖时间，如果有竞争对手通过恶意下单的方式将该卖家的商品全部下单，让这款商品的库存减为零，那么这款商品就不能正常售卖了。要知道，这些恶意下单的人是不会真正付款的，这正是**“下单减库存”**方式的不足之处

既然“下单减库存”可能导致恶意下单，从而影响卖家的商品销售，那么有没有办法解决呢？如果采用**“付款减库存”**的方式是不是就可以了？的确可以。但是**“付款减库存”**又会导致另外一个问题：库存超卖。假如有100件商品，就可能出现500人下单成功的情况，因为下单时不会减库存，所以也就可能出现下单成功数远远超过真正库存数的情况，这尤其会发生在做活动的热门商品上。这样一来，就会导致很多买家下单成功但是付不了款，买家的购物体验自然比较差

不管是“下单减库存”还是“付款减库存”，都会导致商品库存不能完全和实际售卖情况对应起来的情况，既然都存在缺点，那能不能把两者相结合，采用**“预扣库存”**这种方式呢？

这种方案确实可以在一定程度上缓解上面的问题。但是否就彻底解决了呢？其实没有！

**针对恶意下单这种情况**，虽然把有效的付款时间设置为10分钟，但是恶意买家完全可以在10分钟后再次下单，或者采用一次下单很多件的方式把库存减完。针对这种情况，解决办法还是要结合安全和反作弊的措施来制止。例如，给经常下单不付款的买家进行识别打标（可以在被打标的买家下单时不减库存）、给某些类目设置最大购买件数（例如，参加活动的商品一人最多只能买3件），以及对重复下单不付款的操作进行次数限制等

**针对“库存超卖”这种情况**，在10分钟时间内下单的数量仍然有可能超过库存数量，遇到这种情况我们只能区别对待：对普通的商品下单数量超过库存数量的情况，可以通过补货来解决；但是有些卖家完全不允许库存为负数的情况，那只能在买家付款时提示库存不足



# 秒杀中减库存

目前来看，业务系统中最常见的就是预扣库存方案，像在买机票、买电影票时，下单后一般都有个“有效付款时间”，超过这个时间订单自动释放，这都是典型的预扣库存方案。而具体到秒杀这个场景，应该采用哪种方案比较好呢？

由于参加秒杀的商品，一般都是“抢到就是赚到”，所以成功下单后却不付款的情况比较少，再加上卖家对秒杀商品的库存有严格限制，所以秒杀商品采用**“下单减库存”**更加合理。另外，理论上由于“下单减库存”比“预扣库存”以及涉及第三方支付的“付款减库存”在逻辑上更为简单，所以性能上更占优势

“下单减库存”在数据一致性上，主要就是保证大并发请求时库存数据不能为负数，也就是要保证数据库中的库存字段值不能为负数，一般我们有多种解决方案：
+ 在应用程序中通过事务来判断，即保证减后库存不能为负数，否则就回滚
+ 直接设置数据库的字段数据为无符号整数，这样减后库存字段值小于零时会直接执行SQL语句来报错
+ 使用CASE WHEN判断语句，例如SQL语句:`UPDATE item SET inventory = CASE WHEN inventory &gt;= xxx THEN inventory-xxx ELSE inventory END`



# 数据库问题

1、MySQL自身对于高并发的处理性能就会出现问题，一般来说，MySQL的处理性能会随着并发thread上升而上升，但是到了一定的并发度之后会出现明显的拐点，之后一路下降，最终甚至会比单thread的性能还要差  
2、当减库存和高并发碰到一起的时候，由于操作的库存数目在同一行，就会出现争抢InnoDB行锁的问题，导致出现互相等待甚至死锁，从而大大降低MySQL的处理性能，最终导致前端页面出现超时异常

解决思路:  
1、关闭死锁检测，提高并发处理性能  
2、修改源代码，将排队提到进入引擎层前，降低引擎层面的并发度  
3、组提交，降低server和引擎的交互次数，降低IO消耗

解决方案:  
1. **内存**
	+ 将存库从MySQL前移到Redis中，所有的写操作放到内存中，由于Redis中不存在锁故不会出现互相等待，并且由于Redis的写性能和读性能都远高于MySQL，这就解决了高并发下的性能问题。然后通过队列等异步手段，将变化的数据异步写入到DB中
		- 优点：解决性能问题
		- 缺点：没有解决超卖问题，同时由于异步写入DB，存在某一时刻DB和Redis中数据不一致的风险
	+ 
2. **排队**
	+ 引入队列，然后将所有写DB操作在单队列中排队，完全串行处理。当达到库存阀值的时候就不在消费队列，并关闭购买功能。这就解决了超卖问题
		- 优点：解决超卖问题，略微提升性能
		- 缺点：性能受限于队列处理机处理性能和DB的写入性能中最短的那个，另外多商品同时抢购的时候需要准备多条队列，也存在少买的情况

## 单机java队列

Java的并发包提供了三个常用的并发队列实现，分别是：ConcurrentLinkedQueue、LinkedBlockingQueue、ArrayBlockingQueue

1、ConcurrentLinkedQueue使用的是CAS原语无锁队列实现，是一个异步队列，入队的速度很快，出队进行了加锁，性能稍慢。由于秒杀系统入队需求要远大于出队需求，可以选择ConcurrentLinkedQueue来作为请求队列实现  
2、LinkedBlockingQueue也是阻塞的队列，入队和出队都用了加锁，当队空的时候线程会暂时阻塞  
3、ArrayBlockingQueue是初始容量固定的阻塞队列，比如可以用来作为数据库模块成功竞拍的队列。假设有10件商品，那么我们就设定一个10大小的数组队列



# 秒杀减库存优化

在交易环节中，“库存”是个关键数据，也是个热点数据，因为交易的各个环节中都可能涉及对库存的查询。[上一篇](https://zhouj000.github.io/2018/10/16/SecKill-System-3)介绍分层过滤时提到过，秒杀中并不需要对库存有精确的一致性读，把库存数据放到缓存（Cache）中，可以大大提升读性能

解决大并发读问题，可以采用LocalCache（即在秒杀系统的单机上缓存商品相关的数据）和对数据进行分层过滤的方式，但是像减库存这种大并发写无论如何还是避免不了，这也是秒杀场景下最为核心的一个技术难题

秒杀商品和普通商品的减库存还是有些差异的，例如商品数量比较少，交易时间段也比较短，因此这里有一个大胆的假设，即能否把秒杀商品减库存直接放到缓存系统中实现，也就是直接在缓存中减库存或者在一个带有持久化功能的缓存系统（如Redis）中完成呢？

**如果你的秒杀商品的减库存逻辑非常单一，比如没有复杂的SKU库存和总库存这种联动关系的话，是可以的。但是如果有比较复杂的减库存逻辑，或者需要使用事务，那么还是必须在数据库中完成减库存**

由于MySQL存储数据的特点，同一数据在数据库里肯定是一行存储（MySQL），因此会有大量线程来竞争InnoDB行锁，而并发度越高时等待线程会越多，TPS（Transaction Per Second，即每秒处理的消息数）会下降，响应时间（RT）会上升，数据库的吞吐量就会严重受影响

这就可能引发一个问题，就是单个热点商品会影响整个数据库的性能，导致0.01%的商品影响99.99%的商品的售卖，这是我们不愿意看到的情况。一个解决思路是遵循前面介绍的原则进行隔离，把热点商品放到单独的热点库中。但是这无疑会带来维护上的麻烦，比如要做热点数据的动态迁移以及单独的数据库等

而分离热点商品到单独的数据库还是没有解决并发锁的问题，我们应该怎么办呢？要解决并发锁的问题，有两种办法：
+ **应用层做排队**。按照商品维度设置队列顺序执行，这样能减少同一台机器对数据库同一行记录进行操作的并发度，同时也能控制单个商品占用数据库连接的数量，防止热点商品占用太多的数据库连接
+ **数据库层做排队**。应用层只能做到单机的排队，但是应用机器数本身很多，这种排队方式控制并发的能力仍然有限，所以如果能在数据库层做全局排队是最理想的。阿里的数据库团队开发了针对这种MySQL的InnoDB层上的补丁程序（patch），可以在数据库层上对单行记录做到并发排队

排队和锁竞争不都是要等待吗，有什么区别呢？

> 如果熟悉MySQL的话，会知道InnoDB内部的死锁检测，以及MySQL Server和InnoDB的切换会比较消耗性能，淘宝的MySQL核心团队还做了很多其他方面的优化，如COMMIT_ON_SUCCESS和ROLLBACK_ON_FAIL的补丁程序，配合在SQL里面加提示（hint），在事务里不需要等待应用层提交（COMMIT），而在数据执行完最后一条SQL后，直接根据TARGET_AFFECT_ROW的结果进行提交或回滚，可以减少网络等待时间（平均约0.7ms）。目前阿里MySQL团队已经将包含这些补丁程序的MySQL开源

另外数据更新问题除了前面介绍的热点隔离和排队处理之外，还有些场景（如对商品的lastmodifytime字段的）更新会非常频繁，在某些场景下这些多条SQL是可以合并的，一定时间内只要执行最后一条SQL就行了，以便减少对数据库的更新操作

当然减库存还有很多细节问题，例如预扣的库存超时后如何进行库存回补，再比如目前都是第三方支付，如何在付款时保证减库存和成功付款时的状态一致性，这些都是很大的挑战


参考:  
[如何设计一个秒杀系统](https://time.geekbang.org/column/intro/127)  
