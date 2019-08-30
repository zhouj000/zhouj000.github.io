---
layout:     post
title:      "Java基础SE(二) 集合"
date:       2019-05-05
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
--- 



# List

List是Java中最常用的结合类，List本身是一个接口，其继承了Collection接口，表示有序的队列。常用的List实现有ArrayList、LinkedList、Vector

| List       | null值 | 稳定性(order) | 有序性(sort) | 线程安全(safe) |
| :--------- | :----: | :-----------: | :----------: | :------------: |
| ArrayList  |  yes   |      yes      |      no      |       no       |
| LinkedList |  yes   |      yes      |      no      |       no       |
| Vector     |  yes   |      yes      |      no      |      yes       |

ArrayList底层是用数组实现的，可以认为ArrayList是一个可改变大小的数组。随着越来越多的元素被添加到ArrayList中，其规模是动态增加的

LinkedList底层是通过双向链表实现的。所以LinkedList和ArrayList之前的区别主要就是数组和链表的区别。数组中查询和赋值比较快，因为可以直接通过数组下标访问指定位置；链表中删除和增加比较快，因为可以直接通过修改链表的指针(引用)进行元素的增删。LinkedList还实现了Queue接口，所以他还提供了offer()、peek()、poll()等方法

Vector和ArrayList一样，都是通过数组实现的，但是Vector是线程安全的，其中的很多方法都通过同步(synchronized)处理来保证线程安全，因此相对而言性能会差一点。它们的扩容大小也不同，默认ArrayList是增长原来的50%，Vector则增长原来的100%















