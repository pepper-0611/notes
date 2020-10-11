最近打算把Java网络编程相关的知识深入一下（IO、NIO、Socket编程、Netty)

Java NIO主要需要理解缓冲区、通道、选择器三个核心概念，作为对Java I/O的补充， 以提升大批量数据传输的效率。

学习NIO之前最好能有基础的网络编程知识

[Java I/O流](https://juejin.im/post/6876684050917621774)

[Java 网络编程](https://juejin.im/post/6879283720067481613)

[Java NIO：缓冲区](https://juejin.im/post/6879687154789875720)



通道(Channel)作为NIO的三大核心概念之一(缓冲区、通道、选择器)，用于在字节缓冲区与位于通道另一侧的实体（文件或者套接字）之间有效的传输数据（**核心是传输数据**）

NIO编程的一般模式是：把数据填充到发送字节缓冲区 --> 通过通道发送到通道对端文件或者套接字

## 通道基础

使用Channel的目的是进行数据传输，使用前需要打开通道、使用后需要关闭通道

### 打开通道

我们知道I/O有两大类：File IO和 Stream I/O，其对应到通道也就有文件通道（FileChannel）和套接字通道(SocketChannel、ServerSocketChannel、DatagramChannel)两种

对于套接字通道，使用静态工厂方法打开

```java
SocketChannel sc = SocketChannel.open();
ServerSocketChannel sc = ServerSocketChannel.open();
DatagramChannel sc = DatagramChannel.open();
```

对于文件通道只能通过对一个RandomAccessFile、FileInputStream、FileOutputStream对象调用getChannel()方法获取

```java
FileInputStream in = new FileInputStream("/tmp/a.txt");
FileChannel fc = in.getChannel();
```

### 使用通道进行数据传输

下段代码首先将要写入的数据放到ByteBuffer中， 然后打开文件通道，把缓冲区中的数据放到文件通道。

```java
//准备数据并放入字节缓冲区
ByteBuffer bf = ByteBuffer.allocate(1024);
bf.put("i am cool".getBytes());
bf.flip();
//打开文件通道
FileOutputStream out = new FileOutputStream("/tmp/a.txt");
FileChannel fc = out.getChannel();
//数据传输
fc.write(bf);
//关闭通道
fc.close();
```

### 关闭通道

如同Socket、FileInputStream等对象使用完毕之后需要关闭一样， 通道使用之后也需要关闭。一个打开的通道代表与一个特定I/O服务的特定连接并封装该连接的状态，通道关闭时连接丢失，不再连接任何东西。

### 阻塞 & 非阻塞模式

通道有阻塞和非阻塞两种运行模式，非阻塞模式的通道永远不会休眠，请求的操作要么立即完成，要么返回一个结果表明未进行任何操作（具体看Socket通道处的描述）。只有面向流的通道可使用非阻塞模式

## 文件通道

文件通道用于对文件进行访问， 通过对一个RandomAccessFile、FileInputStream、FileOutputStream对象调用getChannel()方法获取。调用getChannel方法返回一个连接到相同文件的FileChannel对象，该FileChannel对象具有与file对象相同的访问权限。

### 文件访问

使用文件通道的目的还是对文件进行读写操作，通道的读写api如下：

```java
public abstract int read(ByteBuffer dst) throws IOException;
public abstract int write(ByteBuffer src) throws IOException;
```

下面是一段读取文件的Demo

```java
//打开文件channel
RandomAccessFile f = new RandomAccessFile("/tmp/a.txt", "r");
FileChannel fc = f.getChannel();
//从channel中读取数据，直到文件尾
ByteBuffer bb = ByteBuffer.allocate(1024);
while (fc.read(bb) != -1) {
;
}
//翻转（读之前需要先进行翻转）
bb.flip();
StringBuilder builder = new StringBuilder();
//把每一个字节转为字符（ascii编码）
while (bb.hasRemaining()) {
builder.append((char) bb.get());
}
System.out.println(builder.toString());

```

上面这个demo有个问题：我们只能读取字节， 然后由应用程序去解码，这个问题我们可以通过工具类Channels将通道包装成Reader和Writer来解决；当然我们也可以直接使用Java I/O流模式的Reader和Writer操作字符

### 文件通道位置与文件空洞

文件通道位置(position)就是普通文件的位置, position的值决定了文件中哪个位置的数据接下来将被读或者写

读取超出文件尾部位置的数据会返回-1（文件EOF）

往一个超出文件尾部的位置写入数据会造成文件空洞：比如一个文件现在有10个字节， 但是此时往position=20 处写入数据就会造成10～20之间的位置是没有数据的，这就是文件空洞

### force操作

force操作强制通道将全部修改立即应用到磁盘文件（防止系统宕机导致修改丢失）

```java
public abstract void force(boolean metaData) throws IOException;
```

## 内存文件映射

FileChannel提供了一个map()方法，该方法可以在一个打开的文件和特殊类型的ByteBuffer(MappedByteBuffer)之间建立一个虚拟内存映射。

因为map方法返回的MappedByteBuffer对象是直接缓冲区，所以通过MappedByteBuffer来操作文件非常高效（尤其是大量数据传输的情况）

### MappedByteBuffer的使用

通过MappedByteBuffer读取文件

```java
FileInputStream in = new FileInputStream("/tmp/a.txt");
FileChannel fc = in.getChannel();
MappedByteBuffer mbb = fc.map(MapMode.READ_ONLY, 0, fc.size());
StringBuilder builder = new StringBuilder();
while (mbb.hasRemaining()) {
  builder.append((char) mbb.get());
}
System.out.println(builder.toString());
```

### MappedByteBuffer的三种模式

- READ_ONLY 

- READ_WRITE

- PRIVATE 

  只读和读写模式都好理解，PRIVATE模式下写操作写的是一个临时缓冲区，不会真正去写文件。（**写时拷贝思想**）

## Socket通道

Socket 通道可以运行在非阻塞模式且是可选择的，这两点使得对于网络编程我们不再需要为每个Socket连接创建一个线程，而是使用一个线程即可管理成百上千的Socket连接。

所有的Socket通道在实例化的时候都会创建一个对象的Socket对象， Socket通道并不负责协议相关的操作， 协议相关的操作都委派给对等socket对象（如SocketChannel对象委派给Socket对象）

### 非阻塞模式

相较于传统Java Socket的阻塞模式，SocketChannel提供了非阻塞模式，以构建高性能的网络应用程序

非阻塞模式下，几乎所有的操作都是立刻返回的。比如下面的SocketChannel运行在非阻塞模式下，connect操作会立即返回，如果success为true代表连接已经建立成功了， 如果success为false, 代表连接还在建立中（tcp连接需要一些时间）。

```java
 //打开Socket通道
 SocketChannel ch = SocketChannel.open();
 //非阻塞模式
 ch.configureBlocking(false);
 //连接服务器 
 boolean success = ch.connect(InetSocketAddress.createUnresolved("127.0.0.1", 7001));
 //轮训连接状态， 如果连接还未建立就可以做一些别的工作
 while (!ch.finishConnect()){
    //dosomething else
 }
 //连接建立， 做正事
 //do something;
```



### ServerSocketChannel

ServerSocketChannel与ServerSocket类似，只是可以运行在非阻塞模式下

下为一个通过ServerSocketChannel构建服务器的简单例子，主要体现了非阻塞模式，核心思想与ServerSocket类似

```java
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);
ssc.bind(new InetSocketAddress(7001));
while (true){
  SocketChannel sc = ssc.accept();
  if(sc != null){
    handle(sc);
  }else {
    Thread.sleep(1000);
  }
}
```

### SocketChannel 与 DatagramChannel

SocketChannel 对应 Socket， 模拟TCP协议；DatagramChannel对应DatagramSocket, 模拟UDP协议

二者的使用与SeverSocketChannel大同小异，看API即可

## 工具类

文体通道那里我们提到过， 通过只能操作字节缓冲区， 编解码需要应用程序自己实现。如果我们想在通道上直接操作字符，我们就需要使用工具类Channels，工具类Channels提供了通道与流互相转换、通道转换为阅读器书写器的能力，具体API入下

```java
//通道 --> 输入输出流
public static OutputStream newOutputStream(final WritableByteChannel ch);
public static InputStream newInputStream(final AsynchronousByteChannel ch);
//输入输出流 --> 通道
public static ReadableByteChannel newChannel(final InputStream in);
public static WritableByteChannel newChannel(final OutputStream out);
//通道  --> 阅读器书写器
public static Reader newReader(ReadableByteChannel ch, String csName);
public static Writer newWriter(WritableByteChannel ch, String csName);
```

通过将通道转换为阅读器、书写器我们就可以直接在通道上操作字符。

```java
	RandomAccessFile f = new RandomAccessFile("/tmp/a.txt", "r");
  FileChannel fc = f.getChannel();
  //通道转换为阅读器，UTF-8编码
  Reader reader = Channels.newReader(fc, "UTF-8");
  int i = 0, s = 0;
  char[] buff = new char[1024];
  while ((i = reader.read(buff, s, 1024 - s)) != -1) {
    s += i;
  }
  for (i = 0; i < s; i++) {
    System.out.print(buff[i]);
  }
```

## 总结

通道主要分为文件通道和套接字通道。

对于文件操作：如果是大文件使用通道的文件内存映射特性（MappedByteBuffer）来有利于提升传输性能， 否则我更倾向传统的I/O流模式（字符API）；对于套接字操作， 使用通道可以运行在非阻塞模式并且是可选择的，利于构建高性能网络应用程序。