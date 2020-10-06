最近打算把Java网络编程相关的知识深入一下（IO、NIO、Socket编程、Netty)

Java NIO主要需要理解缓冲区、通道、选择器三个核心概念，作为对Java I/O的补充， 以提升大批量数据传输的效率。

学习NIO之前最好能有基础的网络编程知识

[Java I/O流](https://juejin.im/post/6876684050917621774)

[Java 网络编程](https://juejin.im/post/6879283720067481613)

## 基础

除了需要对 Java 网络编程有一定了解，还需要对用户空间、内核空间、内存空间多重映射等知识有一定了解

### 用户空间与内核空间

为了提供操作系统的稳定性，操作系统将虚拟地址空间分为用户空间和内核空间

其中用户进程（我们自己的程序）只能操作用户空间

### I/O过程中的数据流向

假设我们需要从磁盘中某个文件读取数据。进程发起read()系统调用，进入内核态，内核随即向磁盘控制硬件发出命令， 要求其从磁盘读取数据，磁盘控制器将数据直接写入到内核内存缓冲区中（这一步DMA完成，不需要CPU参与），随后内核把数据从内核空间的临时缓冲区拷贝到用户缓冲区（需要CPU参与），进程切换回用户态继续执行。

总结起来的数据流向是： 磁盘 ---> 内核缓冲区 ---> 用户缓冲区

那么问题来了：内核缓冲区的数据拷贝到用户缓冲区的这一步显得有点多余，是否可以避免？ 

### 内存空间多重映射

我们知道对于虚拟地址空间，一个以上的虚拟地址可指向同一个物理内存地址

如果把用户空间的虚拟地址和内核空间的虚拟地址映射到同一个物理地址，那么这块物理地址代表的空间就对内核和用户进程都可见了！！ 便可省去数据在内核缓冲区和用户缓冲区来回复制的开销。（**这便是直接缓冲区的思想**）

## 缓冲区(Buffer)

Java NIO数据传输过程： 数据先放到发送缓冲区  -->  通过通道发送到接收端 --->  接受端通道接受数据并填充到接受缓冲区

所以缓冲区的作用其实是连接通道作为数据传输的目标或者来源（或者说缓冲区是通道的输入或者输出）

### 核心概念

#### 属性

要理解Buffer的工作机制，首先要了解几个属性的意义

- capacity（容量） 缓冲区的容量，创建缓冲区时指定

- position（位置）**下一个要被读取或者写入的元素的索引**

- limit（上界） **缓冲区中第一个不能被读或者写的位置**

- mark（标记）一个备忘位置

  其中 mark <= position <= limit <= capacity，对limit和position两个属性的理解非常重要

#### 存取

缓冲区的核心就在于存取操作，Buffer提供了相对位置存取和绝对位置存取两种方式

- 相对位置存取：在当前position位置写入或者读取数据， 然后增加position的值

- 绝对位置存取：在指定的位置写入或者读取数据，不改变position的值

```java
 //相对位置存取
 public abstract ByteBuffer put(byte b);
 public abstract byte get();

 //绝对位置存取
 public abstract ByteBuffer put(int index, byte b);
 public abstract byte get(int index);
```



#### 翻转(flip)

翻转是Buffer的核心概念，我们可以理解Buffer有两种模式：写模式和读模式

写模式下，我们分配一个缓冲区，然后直接填充数据(position的值从0开始递增)；读模式下， 我们也从头开始读取数据（position从0开始递增）

那么我们怎么从写模式切换到读模式呐？翻转！！ 翻转的时候我们用limit记录待读取数据的长度， 然后将position置为0 就可以开始读取数据了

下为翻转的源码

```java
public final Buffer flip() {
  	//记录待读取数据的长度
    limit = position;
  	//从头开始读取数据
    position = 0;
    mark = -1;
    return this;
}
```



#### demo

一个完整的例子

```java
//创建一个缓冲区 
ByteBuffer buffer = ByteBuffer.allocate(100);
//写数据
for (char c : "hello".toCharArray()) {
  buffer.put((byte) c);
}
//翻转
buffer.flip();//等价于 buffer.limit(buffer.position()).position(0);
//读数据
while (buffer.hasRemaining()) {
  char c = (char) buffer.get();
  System.out.println(c);
}
```



### 创建缓冲区

Buffer不能直接通过构造函数实例化，都是通过**静态工厂方法**来创建。下为ByteBuffer的静态工厂方法

```java
//创建内存缓冲区
public static ByteBuffer allocate(int capacity);
//创建直接缓冲区
public static ByteBuffer allocateDirect(int capacity) ;

public static ByteBuffer wrap(byte[] array, int offset, int length)
```



### 直接缓冲区(DirectByteBuffer)

对于一般的I/O过程，数据的流向总是：磁盘或者网络 --> 内核临时缓冲区 --> 用户空间缓冲区，其中内核空间临时缓冲区到用户空间缓冲区复制这一步显得有点多余！！

直接缓冲区解决了这个问题， 直接缓冲区对内核和用户空间都可见，这样就可以避免"内核临时缓冲区到用户空间缓冲区"复制的开销

虽然直接缓冲区是I/O的最佳选择，但是其比创建非直接缓冲区花费更大的成本，所以我们对直接缓冲区一般都会重复使用（每次使用都创建的话成本就太高了）。



## 总结

本文主要讲解NIO学习需要掌握的一些基础知识以及缓冲区的使用，重点是对直接缓冲区的理解。











