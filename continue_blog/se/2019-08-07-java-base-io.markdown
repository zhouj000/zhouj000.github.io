---
layout:     post
title:      "Java基础SE(四) IO"
date:       2019-05-05
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
--- 


[Java基础SE(二) 集合](https://zhouj000.github.io/2021/02/26/java-base-collections/)  


# IO


Java的IO包主要关注的是从原始数据源的读取以及输出原始数据到目标媒介。典型的数据源和目标媒介：
+ 

文件
管道
网络连接
内存缓存
System.in, System.out, System.error(注：Java标准输入、输出、错误输出)





+ 字符流
	- Reader
		+ BufferedReader
		+ InputStreamReader
			- FileReader
		+ StringReader
		+ PipedReader
		+ FilterReader
			- PushbackReader
	- Writer
		+ BufferedWriter
		+ OutputStreamWriter
			- FileWriter
		+ StringWriter
		+ PipedWriter
		+ CharArrayWriter
		+ FilterWriter
+ 字节流
	- InputStream
		+ FileInputStream
		+ FilterInputStream
			- BufferedInputStream
			- DataInputStream
			- PushbackInputStream
		+ ObjectInputStream
		+ PipedInputStream
		+ StringBufferInputStream
		+ ByteArrayInputStream
	- OutputStream
		+ FileOutputStream
		+ FilterOutputStream
			- BufferedOutputStream
			- DataOutputStream
			- PrintStream
		+ ObjectOutputStream
		+ PipedOutputStream
		+ ByteArrayOutputStream


## File操作

https://ifeve.com/java-io/









## 绝对路径与相对路径















