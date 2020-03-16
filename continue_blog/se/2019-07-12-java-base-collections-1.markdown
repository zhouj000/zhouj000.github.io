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


# Collection

## List

List是Java中最常用的集合类(容器)，List本身是一个接口，其继承了Collection接口，并表示有序的队列。常用的List实现有ArrayList、LinkedList、Vector

| List       | null值 | 稳定性(order) | 有序性(sort) | 线程安全(safe) |
| :--------- | :----: | :-----------: | :----------: | :------------: |
| ArrayList  |  yes   |      yes      |      no      |       no       |
| LinkedList |  yes   |      yes      |      no      |       no       |
| Vector     |  yes   |      yes      |      no      |      yes       |

ArrayList底层是用**数组**实现的，可以认为ArrayList是一个可改变大小的数组。随着越来越多的元素被添加到ArrayList中，其规模是**动态增加**的。ArrayList还实现了RandomAccess接口，获得了快速随机访问存储元素的功能
![arraylist](arraylist.png)

LinkedList底层是通过**双向链表**实现的。所以LinkedList和ArrayList之前的区别主要就是数组和链表的区别。数组中**查询和赋值**比较快，因为可以直接通过数组**下标**访问指定位置；链表中删除和增加比较快(数组的动态扩容会比较慢，然而随着JDK发展，已经差不多了)，因为可以直接通过修改链表的**指针(引用)**进行元素的增删。LinkedList还实现了**Queue(Deque)**接口，是一个双向队列，所以他还提供了offer()、peek()、poll()等方法
![linkedlist](linkedlist.png)

Vector和ArrayList一样，都是通过**数组**实现的，但是Vector是**线程安全**的，其中的很多方法都通过同步(**synchronized**)处理来保证线程安全，因此相对而言性能会差一点。它们的扩容大小也不同，默认ArrayList是增长原来的50%，Vector则增长原来的100%

### 扩容

\ArrayList扩容:
```java
// add时候传入最小扩容长度为size + 1，空列表时为10
private void ensureExplicitCapacity(int minCapacity) {
	// 版本号++
	modCount++;

	// overflow-conscious code
	if (minCapacity - elementData.length > 0)
		grow(minCapacity);
}

// 内部用数组维护
transient Object[] elementData;

private void grow(int minCapacity) {
	// overflow-conscious code
	int oldCapacity = elementData.length;
	// 默认增加一半，即原值的1.5倍
	int newCapacity = oldCapacity + (oldCapacity >> 1);
	if (newCapacity - minCapacity < 0)
		newCapacity = minCapacity;
	if (newCapacity - MAX_ARRAY_SIZE > 0)
		newCapacity = hugeCapacity(minCapacity);
	// minCapacity is usually close to size, so this is a win: 
	// 拷贝到新列表，底层还是用的System.arraycopy本地方法
	elementData = Arrays.copyOf(elementData, newCapacity);
}

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private static int hugeCapacity(int minCapacity) {
	if (minCapacity < 0) // overflow
		throw new OutOfMemoryError();
	return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

LinkedList由于是双向链表结构，因此不需要扩容，只需要分别设置前节点，后节点即可
```java
private static class Node<E> {
	E item;
	Node<E> next;
	Node<E> prev;
}	
```

对于Vector，基本上是个同步的ArrayList，不过扩容因子是增加1倍，即原值的2倍
```java
int newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity);
```

### 其他

ArrayList删除：
```java
public E remove(int index) {
	rangeCheck(index);

	modCount++;
	E oldValue = elementData(index);
	// 删除位后一位
	int numMoved = size - index - 1;
	if (numMoved > 0)
		// 拷贝后面的到前面
		System.arraycopy(elementData, index+1, elementData, index,
						 numMoved);
	// 最后一位置为null，然后size-1
	elementData[--size] = null; // clear to let GC do its work

	return oldValue;
}
```


## Set

Set是类似于List，又相较有点特殊的集合。



hashset
LinkedHashSet
treeset


















## Queue


# Map







