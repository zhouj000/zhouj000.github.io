---
layout:     post
title:      "Java基础SE(一) 数据类型与关键字"
date:       2019-05-05
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
--- 


[Java基础SE(二) 集合](https://zhouj000.github.io/2021/02/26/java-base-collections/)  



# 类型

## String

首先看String类的定义
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
```
可以了解到，String类是**final**的，即不可被继承。其实现了序列化接口、排序接口和字符序列

> CharSequence与String都能用于定义字符串，但CharSequence的值是可读可写序列，而String的值是**只读序列**

然后看存储结构
```
private final char value[];
private int hash;
```
因此做一些length、isEmpty、charAt等操作时，只需要对数组进行操作就行，非常方便。而像equals、compareTo、indexOf、lastIndexOf等都是对数组的每一个字符进行一一比较的。由于String是不可变的(final数组)，所以像substring、concat等操作，是直接创建一个**新的**String对象返回，同时这样使得字符串池得以实现，节省了很多堆空间，并且没有安全性问题(final)

String的hash使用31素数作为系数：`h = 31 * h + val[i]`

> JDK 6的substring方法虽然创建了一个新的String对象，但是其value属性依然指向堆内存中原来的那个数组，区别只是它们的count和offset不同了，因此可能会有性能问题。而在JDK 7以后，substring方法真正的在堆内存创建了另外一个数组，`this.value = Arrays.copyOfRange(value, offset, offset+count);`

replaceFirst可以使用正则匹配或字符串去替换第一个符合的内容。replace和replaceAll也都是使用Pattern.matcher的方式，但是replaceAll和replaceFirst一样可以使用正则也可以使用普通字符串，而replace是仅将字符串理解为字面意义上的字符串，**不能使用正则语法匹配**，所以这2个方法的区别在这里，而不是看起来字面意义上的替换单个和替换全部，它们都是替换全部的，这个需要注意，换个方式也就是如果不使用正则表达式，replace和replaceAll的结果是完全一致的

在Java语言中，**操作符重载是不被允许的**，虽然提高了灵活性，但会提高复杂性和可读性，而且与Java设计思想相悖。但是对于String对象而言，它可以将2个String用"+"相加，看起来像是对"+"的重载，事实上从class文件可以看出，这是Java提供的语法糖，它在拼接的时候会使用`StringBuilder.append()`方法而不是"+"，因此如果在循环中使用会造成创建多个StringBuilder对象的资源浪费

String中有许多方法会创建新的数组，其中就会使用Arrays.copyOf方法和System.arraycopy方法，而Arrays.copyOf方法底层也是用了System.arraycopy方法。需要注意的是，对于二维数组，新copy出的数组被复制的是引用，所以原数组和新数组都指向相同的一些数组，因此修改会影响到原数组。对于数组的复制，由于System.arraycopy使用native方法，因此贴近底层，使用了内存复制，省去了大量的数组寻址访问等时间，效率很高

扩展：  
[System.arraycopy()方法解析](https://www.jianshu.com/p/c6537472d623)  

String被设计为不可变的原因：  
1、字符串常量池的需要  
2、允许String对象缓存HashCode  
3、安全性  
4、线程安全

StringBuffer与StringBuilder共同父类AbstractStringBuilder，前者是线程安全的，后者是非线程安全的
```java
public AbstractStringBuilder append(String str) {
	if (str == null)
		return appendNull();
	int len = str.length();
	// 内部使用了Arrays.copyOf
	ensureCapacityInternal(count + len);
	str.getChars(0, len, value, count);
	count += len;
	return this;
}
```


## Integer与Long

Integer同样也是final class，并且值也是final的
```
public final class Integer extends Number implements Comparable<Integer> {
	@Native public static final int   MIN_VALUE = 0x80000000;
	@Native public static final int   MAX_VALUE = 0x7fffffff;
	public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
	private final int value;
	@Native public static final int SIZE = 32;
	public static final int BYTES = SIZE / Byte.SIZE;  // 32/8 = 4
```

从JDK 5以后加入的autoboxing功能，自动拆箱和装箱依靠JDK的编译器在编译期的预处理工作，可以实现基本数据类型int与引用数据对象Integer的封装转换。自动拆箱很典型的用法就是在进行运算的时候：因为对象不能直接进行运算，需要转化为基本数据类型后才能进行加减乘除

Java对autoboxing使用了**享元模式**(flyweight)，对常用的简单数字进行了重复利用，**缓存了-128到127之间的数字**。最小值-128是固定的，而缓存大小与缓存最大值可以通过JVM参数设置，默认为127
```java
private static class IntegerCache {
	static final int low = -128;
	static final int high;
	static final Integer cache[];

	static {
		// high value may be configured by property
		int h = 127;
		String integerCacheHighPropValue =
			sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
		if (integerCacheHighPropValue != null) {
			try {
				int i = parseInt(integerCacheHighPropValue);
				i = Math.max(i, 127);
				// Maximum array size is Integer.MAX_VALUE
				h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
			} catch( NumberFormatException nfe) {
				// If the property cannot be parsed into an int, ignore it.
			}
		}
		high = h;

		cache = new Integer[(high - low) + 1];
		int j = low;
		for(int k = 0; k < cache.length; k++)
			cache[k] = new Integer(j++);

		// range [-128, 127] must be interned (JLS7 5.1.7)
		assert IntegerCache.high >= 127;
	}

	private IntegerCache() {}
}


public static Integer valueOf(int i) {
	if (i >= IntegerCache.low && i <= IntegerCache.high)
		return IntegerCache.cache[i + (-IntegerCache.low)];
	return new Integer(i);
}
```
因此在这个范围内，除非使用**new**创建Integer对象，否则通过autoboxing封装的Integer对象间使用`==`都会返回true，而使用new则创建了新的对象，其地址不同，返回false。自动装箱拆箱不仅在基本数据类型中有应用，在String类中也有应用(即用new和不用new使用字符串池)

Integer的toString方法有2种，一种可以实现将数字i转变成radix进制数，另一种单纯打印数字，它们都将数字转换为char数组，然后创建String返回，而且使用了大量优化(位移代替除法、使用初始化好的char数组取值)

Integer里有许多方法使用了位移操作和位操作，其中有几个特殊意义的十六进制数
```
十六进制		二进制
0x55555555		01010101010101010101010101010101
0x33333333		00110011001100110011001100110011
0x0f0f0f0f		00001111000011110000111100001111
0xff00			1111111100000000
0x3f			111111
```
Integer在很多方法中使用负数，很容易**避免数据操作的溢出**。正数的原码、反码和补码是一样的，而负数的反码是其在原码的基础上除了符号位不变，其他位取反；且负数的补码是其在反码的基础上某位加1

与Integer基本类似，Long内部也有一个LongCache用来缓存固定的-128到127的Long对象

## Float与Double

float和double是java的基本类型，用于浮点数表示，Float和Double就是它们的包装类。不过它们都有精度问题，浮点数采用**"尾数+阶码"**的编码方式，类似于科学计数法的**"有效数字+指数"**的表示方式，二进制无法精确表示大部分的十进制小数，所以浮点数间的运算不准确，应该避免浮点数的比较。因此Float和Double只能用来做科学计算和工程计算。商业运算中我们要使用**BigDecimal**

内部的与String相互转换的方法(parseXXX与toString)使用了FloatingDecimal


## 数据类型长度

| 基本类型 |            包装类         |             长度           |             取值范围             |
| :------: | :------------------------:| :------------------------: | :------------------------------: |
| byte     | java.lang.Byte(Number)    | 1字节/8位 有符号           |     -128(-2^7) ~ 127(2^7-1)      |
| boolean  | java.lang.Boolean         | 1位                        |           true / false           |
| short    | java.lang.Short(Number)   | 2字节/16位 有符号          |  -32768(-2^15) ~ 32767(2^15-1)   |
| char     | java.lang.Character       | 2字节/16位 Unicode字符     |      0(\u0000) ~ 65535(\uFFFF)   |
| int      | java.lang.Integer(Number) | 4字节/32位 有符号          |          -2^31 ~ 2^31-1          |
| long     | java.lang.Long(Number)    | 8字节/64位 有符号          |          -2^63 ~ 2^63-1          |
| float    | java.lang.Float(Number)   | 4字节/32位(1,8,23) 单精度  | (+/-) 2^-149 ~ (2-2^-23)*2^127   |
| double   | java.lang.Double(Number)  | 8字节/64位(1,11,52) 双精度 | (+/-) 2^-1074 ~ (2-2^-52)·2^1023 |

另外一个英文字符或符号占用1个字节；而一个中文字符或符号可能是2个、3个、4个字节，不同的编码格式占字节数是不同的，UTF-8编码下的一个中文所占字节也是不确定的(汉字少数是3字节(52156个)，多数是4字节(64029个))

扩展：  
[Java一个汉字占几个字节(详解与原理)](https://www.cnblogs.com/lslk89/p/6898526.html)  


## BigDecimal

在使用BigDecimal解决精度问题的时候，只有使用BigDecimal(String)构造器创建对象，或者BigDecimal.valueOf去创建对象才有意义，否则使用double入参还是会发生精度丢失的问题

BigDecimal由一个整数非标度值(`private final transient long intCompact`或`private final BigInteger intVal`)和一个小数点标度(`private final int scale`)和一个总标度(`private transient int precision`)组成，此外有一个`stringCache`存储对应的字符串，其创建主要就是将字符串拆为字符数组后进行循环判断，生成这几个变量值
```
scale = 7
precision = 8
stringCache = "3.1415926"
intCompact = 31415926

scale = 1
precision = 4
stringCache = "-100.0"
intCompact = -1000


// 定义了intVal，或者传的字符串长度大于等于18时才使用BigInteger表示数字
BigDecimal bg = new BigDecimal(new BigInteger("12345"), 2);
// The unscaled value of this BigDecimal
intVal = {BigInteger@506} "12345"
scale = 2
precision = 0
stringCache = "123.45"
intCompact = 12345
```

此外BigDecimal缓存了0~10的数和标度在0~15的0，在一些地方可以直接使用。BigDecimal的计算主要就是加减乘除，即add、subtract、multiply、divide


> 十进制转二进制方法：
> 整数部分：除以2，取出余数，商继续除以2，直到得到0为止，将取出的余数逆序
> 小数部分：乘以2，然后取出整数部分，将剩下的小数部分继续乘以2，然后再取整数部分，一直取到小数部分为零为止。如果永远不为零，则按要求保留足够位数的小数，最后一位做0舍1入。将取出的整数顺序排列


## Enum

枚举类型是Java5中新增特性的一部分，它是一种特殊的数据类型，并为基础基元类型的值提供备用名称，之所以特殊是因为它既是一种类(class)类型却又比类类型多了些特殊的约束，但是这些约束的存在也造就了枚举类型的简洁性、安全性以及便捷性

在使用关键字enum创建枚举类型并编译后，编译器会为我们生成一个相关的类，这个类继承了JavaAPI中的`java.lang.Enum`类，事实上也是一个**类类型**

它用于声明一组命名的常数，当一个变量有几种可能的取值时，可以将它定义为枚举类型，比如月份、季节等。可以在枚举中添加新方法，覆盖枚举方法，自带方法:`name()、ordinal()`

目前**最安全的**实现单例的方法是通过内部静态Enum的方式来实现，因为JVM会保证Enum不会被反射，且构造器方法只执行一次



# 关键字

## Serializable与transient

Serializable是序列化接口，实现了Serializable的接口拥有了序列化功能，可以设置一个serialVersionUID，用于标识这个类的序列化版本，如果序列化与反序列化的同一个类的版本不同，会抛出异常

// TODO


transient是服务于序列化的，被transient修饰的成员变量将不被序列化



## instanceof

如果object是class的一个实例，则instanceof运算符返回true。如果object不是指定类的一个实例，或者object是null，则返回 false。

但是instanceof在Java的编译状态和运行状态是有区别的：  
1、在编译状态中，class可以是object对象的父类，自身类，子类。在这三种情况下Java编译时不会报错  
2、在运行转态中，class可以是object对象的父类，自身类，不能是子类。在前两种情况下result的结果为true，最后一种为false。但是class为子类时编译不会报错，只是运行结果为false



	 
    volatile  synchronized  final  static   const	 
	 
	 
	 
	 
	 


# 其他

泛型与泛型擦除

位运算

序列化

注解

正则表达式

编码方式

时间处理


invokedynamic



















