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
+ 文件
+ 管道
+ 网络连接
+ 内存缓存
+ Java标准输入、输出、错误输出
	- System.in, System.out, System.error

一些IO类：
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

![io-system](io-system.png)

Reader是Java IO中所有Reader的基类。Reader与InputStream类似，不同点在于，Reader基于字符而非基于字节。所以Reader用于读取文本，而InputStream用于读取原始字节。Writer同理

> Java内部使用UTF8编码表示字符串。输入流中一个字节可能并不等同于一个UTF8字符。如果你从输入流中以字节为单位读取UTF8编码的文本，并且尝试将读取到的字节转换成字符，可能会得不到预期的结果

## 常用操作

```java
byte[] content = new byte[1024];
int length;
while ((length = inputStream.read(content)) != -1) {
	outputStream.write(content, 0, length);
}
outputStream.flush();
outputStream.close();
```

```java
Reader reader = new InputStreamReader(new FileInputStream("xxx.txt"), "UTF-8");
int data = reader.read();
while(data != -1){
    // 这里不会造成数据丢失，因为返回的int类型变量data只有低16位有数据，高16位没有数据
    char theChar = (char) data;
    data = reader.read();
}
reader.close();

Writer writer = new OutputStreamWriter(new FileOutputStream("xxx.txt"));
writer.write("Hello World");
writer.close();
```

## 绝对路径与相对路径






# NIO

标准的IO基于字节流和字符流进行操作的，而NIO是基于通道(Channel)和缓冲区(Buffer)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中

当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。Java NIO可以让你非阻塞的使用IO

Java NIO引入了选择器的概念，选择器用于监听多个通道的事件(比如：连接打开，数据到达)。因此单个的线程可以监听多个数据通道

## Channels与Buffers

![buffer1](buffer1.png)

数据可以从Channel读到Buffer中，也可以从Buffer写到Channel中，即通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。通道可以异步地读写

Channel的一些实现：  
1、FileChannel：从文件中读写数据  
2、DatagramChannel：能通过UDP读写网络中的数据  
3、SocketChannel：能通过TCP读写网络中的数据  
4、ServerSocketChannel：可以监听新进来的TCP连接，对每一个新进来的连接都会创建一个SocketChannel

```java
RandomAccessFile aFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel inChannel = aFile.getChannel();

// 0.分配48字节capacity的ByteBuffer
ByteBuffer buf = ByteBuffer.allocate(48);

// 1.写入数据到Buffer
int bytesRead = inChannel.read(buf);
while (bytesRead != -1) {
    System.out.println("Read " + bytesRead);
	// 2.调用flip()方法，从写模式切换到读模式
    buf.flip();
	// 3.从Buffer中读取数据
    while(buf.hasRemaining()){
        System.out.print((char) buf.get());
    }
	// 4.调用clear()方法或者compact()方法，让它可以再次被写入
    buf.clear();
    bytesRead = inChannel.read(buf);
}
aFile.close();
```

Buffer的一些实现：  
1、ByteBuffer  
2、CharBuffer  
3、DoubleBuffer  
4、FloatBuffer  
5、IntBuffer  
6、LongBuffer  
7、ShortBuffer  
8、MappedByteBuffer

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存

![buffers-modes](buffers-modes.png)

capacity和limit的含义取决于Buffer处在读模式还是写模式。不管Buffer处在什么模式，capacity的含义总是一样的

+ capacity
	- Buffer有一个固定的大小值，叫"capacity"，只能往里写capacity个类型数据
	- 一旦Buffer满了，需要将其清空(通过读数据或者清除数据)，才能继续写数据往里写数据
+ position
	- 写模式下，position表示当前的位置。初始的position值为0，当写入数据后，position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1
	- 当将Buffer从写模式切换到读模式，position会被重置为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置
+ limit
	- 写模式下，limit表示你最多能往Buffer里写多少数据，即等于capacity
	- 当将Buffer从写模式切换到读模式，limit表示你最多能读到多少数据，即limit会被设置成写模式下的position值

```java
写入数据到Buffer:
int bytesRead = inChannel.read(buf);	// 从Channel读
or
buf.put(127);	// put有很多版本

从Buffer读取数据：
int bytesWritten = inChannel.write(buf);  // 写入channel
or
byte aByte = buf.get();		// get有很多版本
```

`rewind()`方法可以将position设回0，这样可以重读Buffer中的所有数据，而limit保持不变

`clear()`方法可以将position设回0，limit被设置成capacity的值。Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据，未读的数据将被遗忘

`compact()`方法将所有未读的数据拷贝到Buffer起始处，position设到最后一个未读元素正后面，limit属性依然为capacity。这样Buffer准备好写数据，且不会覆盖未读的数据

`mark()`方法可以标记Buffer中的一个特定position，之后可以通过调用`reset()`方法恢复到这个position

#### Scatter/Gather

分散（scatter）从Channel中读取是指在读操作时将读取的数据写入多个buffer中。聚集（gather）写入Channel是指在写操作时将多个buffer的数据写入同一个Channel。scatter/gather经常用于需要将传输的数据分开处理的场合，例如传输一个由消息头和消息体组成的消息，你可能会将消息体和消息头分散到不同的buffer中，这样你可以方便的处理消息头和消息体

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);
ByteBuffer[] bufferArray = { header, body };

// 按照buffer在数组中的顺序将从channel中读取的数据写入到buffer，当一个buffer被写满后，channel紧接着向另一个buffer中写
channel.read(bufferArray);

// write data into buffers
// 按照数组顺序写入，注意只有position和limit之间的数据才会被写入，因此与Scatter相反，Gathering能较好的处理动态消息
channel.write(bufferArray);
```

#### 通道间传输

如果两个通道中有一个是FileChannel，那你可以直接将数据从一个channel传输到另外一个channel

```java
RandomAccessFile fromFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel toChannel = toFile.getChannel();

long position = 0;
long count = fromChannel.size();

// 将数据从源通道传输到FileChannel中
toChannel.transferFrom(position, count, fromChannel);
or
// 将数据从FileChannel传输到其他的channel中
fromChannel.transferTo(position, count, toChannel);
```

### FileChannel

FileChannel是一个连接到文件的通道，可以通过文件通道读写文件。FileChannel无法设置为非阻塞模式，总是运行在阻塞模式下

```java
RandomAccessFile aFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel inChannel = aFile.getChannel();

// 读取
ByteBuffer buf = ByteBuffer.allocate(48);
// 有时可能需要在FileChannel的某个特定位置进行数据的读/写操作
// long pos = channel.position();
// channel.position(pos +123);
// 表示了有多少字节被读到了Buffer中，-1表示到了文件末尾
int bytesRead = inChannel.read(buf);

// 写入
buf.clear();
buf.put("xxxxxx".getBytes());
buf.flip();
// 无法保证write()方法一次能向FileChannel写入多少字节，需要循环调用
while(buf.hasRemaining()) {
	channel.write(buf);
}

// 最后关闭
channel.close();
```

`FileChannel.truncate(1024);`可以截取一个文件到指定长度，后面部分将被删除，此例为截取文件的前1024个字节

`FileChannel.force()`方法将通道里尚未写入磁盘的数据强制写到磁盘上。出于性能考虑，操作系统会将数据缓存在内存中，所以无法保证写入到FileChannel里的数据一定会即时写到磁盘上。`force()`方法有一个boolean类型的参数，指明是否同时将文件元数据(权限信息等)写到磁盘上

### SocketChannel

SocketChannel是一个连接到TCP网络套接字的通道。可以通过打开一个SocketChannel并连接到服务器；或一个新连接到达ServerSocketChannel时创建一个，这两种方法来创建SocketChannel

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("http://xxx.com", 80));

// 阻塞模式的 读取与写入 与FileChannel相同
socketChannel.close();
```

SocketChannel可以设为非阻塞(非阻塞模式与选择器搭配会工作的更好)，在异步模式下调用：
```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.configureBlocking(false);
socketChannel.connect(new InetSocketAddress("http://xxx.com", 80));

// 在尚未写出任何内容时可能就返回了，因此在循环中做事
while(! socketChannel.finishConnect() ){
    //wait, or do something else...
}
```

### DatagramChannel

DatagramChannel是一个能收发UDP包的通道。因为UDP是无连接的网络协议，所以不能像其它通道那样读取和写入。它发送和接收的是数据包

```java
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9999));

// 接受数据
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
// 将接收到的数据包内容复制到指定的Buffer. 如果Buffer容不下收到的数据，多出的数据将被丢弃
channel.receive(buf);

// 发送数据
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put("xxxx".getBytes());
buf.flip();
// 发送xxxx到xxx.com的UDP端口9999，UDP在数据传送方面没有任何保证
int bytesSent = channel.send(buf, new InetSocketAddress("xxx.com", 9999));
```

由于UDP是无连接的，连接到特定地址并不会像TCP通道那样创建一个真正的连接。而是锁住DatagramChannel，让其只能从特定地址收发数据
```java
channel.connect(new InetSocketAddress("xxx.com", 80));

// 和用传统的通道一样。只是在数据传送方面没有任何保证
int bytesRead = channel.read(buf);
int bytesWritten = channel.write(but);
```


## Selectors

![overview-selectors](overview-selectors.png)

向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪。一旦这个方法返回，线程就可以处理这些事件

> 现代的操作系统和CPU在多任务方面表现的越来越好，所以多线程的开销随着时间的推移，变得越来越小了

```java
Selector selector = Selector.open();

// 必须处于非阻塞模式，*FileChannel不能切换到非阻塞模式
channel.configureBlocking(false);
// 对多个事件感兴趣：int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
SelectionKey key = channel.register(selector, Selectionkey.OP_READ);
```

可以监听四种不同类型的事件：
+ Connect：SelectionKey.OP_CONNECT
+ Accept：SelectionKey.OP_ACCEPT
+ Read：SelectionKey.OP_READ
+ Write：SelectionKey.OP_WRITE

返回的SelectionKey对象包含了一些属性：
+ interest集合
+ ready集合
+ Channel
+ Selector
+ 附加的对象(可选)
	- `selectionKey.attach(theObject);`
	- `SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);`

注册完成后，就可以选择select返回感兴趣的事件了：
+ `int select()`
	- 阻塞到至少有一个通道在你注册的事件上就绪了，返回的int值表示有多少通道已经就绪
+ `int select(long timeout)`
	- 多设置了超时时间
+ `int selectNow()`
	- 不会阻塞立即返回，没有通道可选将直接返回零

当确定有通道就绪后，就可以访问"已选择键集"中的就绪通道了，最后遍历处理：
```java
while(true) {
	int readyChannels = selector.select();
	if(readyChannels == 0) continue;

	Set selectedKeys = selector.selectedKeys();
	Iterator keyIterator = selectedKeys.iterator();
	while(keyIterator.hasNext()) {
		SelectionKey key = keyIterator.next();
		if(key.isAcceptable()) {
			// a connection was accepted by a ServerSocketChannel.
		} else if (key.isConnectable()) {
			// a connection was established with a remote server.
		} else if (key.isReadable()) {
			// a channel is ready for reading
		} else if (key.isWritable()) {
			// a channel is ready for writing
		}
		// Selector不会自己从已选择键集中移除SelectionKey实例，必须在处理完后自己移除
		keyIterator.remove();
	}
}
```

`SelectionKey.channel()`返回的通道需要转型成你要处理的类型，比如ServerSocketChannel或SocketChannel等

`Selector.wakeup()`方法可以让阻塞在`select()`方法上的线程会立马返回，如果当前没有阻塞，则下一个调用`select()`方法的线程会立即醒来

`close()`方法会关闭该Selector，且使注册到该Selector上的所有SelectionKey实例无效。不过通道本身并不会关闭

## 非阻塞服务

一个非阻塞式服务器需要时不时检查输入的消息来判断是否有任何新的完整的消息发送过来。服务器可能会在一个或多个完整消息发来之前就检查了多次。同样，一个非阻塞式服务器需要时不时检查是否有任何数据需要写入。服务器需要检查是否有任何相应的连接准备好将该数据写入它们。只在第一次排队消息时检查是不够的，因为消息可能被部分写入

所有这些非阻塞服务器最终都需要定期执行的三个"管道"，在循环中重复执行：
+ 读取管道(The read pipeline)，用于检查是否有新数据从开放连接进来的
+ 处理管道(The process pipeline)，用于所有任何完整消息
+ 写入管道(The write pipeline)，用于检查是否可以将任何传出的消息写入任何打开的连接

![non-blocking-server-9](non-blocking-server-9.png)
![non-blocking-server-10](non-blocking-server-10.png)

## pipe

管道是2个线程之间的单向数据连接，Pipe有一个source通道和一个sink通道，从source通道读取，被写到sink通道

![pipe](pipe.bmp)

```java
Pipe pipe = Pipe.open();

// // 向管道写数据
Pipe.SinkChannel sinkChannel = pipe.sink();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put("hello world".getBytes());
buf.flip();
while(buf.hasRemaining()) {
    sinkChannel.write(buf);
}

// 从管道读取数据
Pipe.SourceChannel sourceChannel = pipe.source();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = sourceChannel.read(buf);
```









