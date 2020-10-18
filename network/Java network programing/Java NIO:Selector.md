最近打算把Java网络编程相关的知识深入一下（IO、NIO、Socket编程、Netty)

Java NIO主要需要理解缓冲区、通道、选择器三个核心概念，作为对Java I/O的补充， 以提升大批量数据传输的效率。

学习NIO之前最好能有基础的网络编程知识

[Java I/O流](https://juejin.im/post/6876684050917621774)

[Java 网络编程](https://juejin.im/post/6879283720067481613)

[Java NIO：缓冲区](https://juejin.im/post/6879687154789875720)

[Java NIO：通道](https://juejin.im/post/6882258005136506893)



传统监控多个Socket的Java解决方案是为每一个Socket创建一个线程并使线程阻塞在read调用处， 直到数据可读。这种方式在系统并发不高时可以正常运行，如果是并发很高的系统就需要创建很多的线程（每个连接需要一个线程）。过多的线程会导致频繁的上下文切换、且线程是系统资源，可创建最大线程数是有限制的且远小于可以建立的网络连接数。

 NIO的选择器就是为了解决这个问题， 选择器提供了同时询问多个通道是否准备好执行I/O的能力，比如SocketChannel对象是否还有更多的字节待读取， ServerSocketChannel是否有已经到达的客户端连接。

通过使用选择器，我们可以在一个线程里监听多个通道的就绪状态！

## 核心概念

选择器 ：管理可选择通道集合&更新可选择通道的就绪状态

可选择通道：所有继承了SelectableChannel的通道， Socket都是可选择的，而文件通道不是，只有可选择通道可以注册到选择器上。

选择键：可选择通道注册到选择器后返回选择键， 所以选择键其实是通道与选择器注册关系的一个封装

三者之间的关系：***可选择通道***注册到***选择器***上，返回***选择键*** 

## 选择器使用

使用选择器的步骤一般是：

1. 构造选择器

2. 可选择通道注册到选择器

3. 选择器选择（选择出就绪通道）

4. 对就绪通道进行读写操作

5. 重复 2 ～ 4

   下面从这几步进行讲解

### 构造选择器

使用静态工厂方法构造（底层使用SelectorProvider创建Selector实例， SelectorProvider支持java spi扩展）

```java
Selector selector = Selector.open();	
```

### 可选择通道注册到选择器

只有运行在**非阻塞模式***下的通道可以注册到选择器上

注册的方法定义在SelectableChannel类中， 注册时需带上可选择的操作（四种可选择操作，定义在SelectionKey中），也可以带上附件

注册成功返回选择键，具体API如下：

```java
//参数 选择器 + 可选择操作
public final SelectionKey register(Selector sel, int ops)
//带附件的版本
public abstract SelectionKey register(Selector sel, int ops, Object att)
```

### 选择器选择

select方法选择出就绪的通道，把该就绪通道关联的SelectionKey放到选择器的selectedKeys集合中（通道就绪指底层Socket已经就绪，执行连接或者读写操作时不会阻塞）

### 对就绪通道进行读写操作 

对就绪通道的读写操作见下面Demo

### Demo

 写了一个demo把上面几步整合起来，代码主要两部分：可选择通道注册&选择器选择 和 对就绪通道进行读写操作

#### 可选择通道注册&选择器选择

```java

	//构造选择器
	Selector selector = Selector.open();
	//初始化ServerSocketChannel绑定本地端口并设置为非阻塞模式
  ServerSocketChannel ch = ServerSocketChannel.open();
  ch.bind(new InetSocketAddress("127.0.0.1", 7001));
  ch.configureBlocking(false);
	//通道注册到选择器，并且关心ACCEPT操作（因为是Server)
  ch.register(selector, SelectionKey.OP_ACCEPT);
	//一直循环， 进行通道就绪状态的选择	
  while (true) {
    int n = selector.select();
    if (n == 0) {
      continue;
    }
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    //遍历所有就绪的通道
    while (it.hasNext()) {
      SelectionKey key = it.next();
      //如果就绪操作是ACCEPT, 接受客户端连接并注册到选择器。
      if (key.isAcceptable()) {
        try {
          SocketChannel cch = ch.accept();
          cch.configureBlocking(false);
          //客户端通道注册到选择选择器，并关心READ操作
          cch.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
        } catch (IOException e) {
          e.printStackTrace();
        }
      } else {
        //如果是其他就绪操作， 提交到线程池处理
        pool.submit(() -> handle(key));
      }
      //移除该选择键
      it.remove();
    }
  }
```



#### 对就绪通道进行读写操作

```java
public static void handle(SelectionKey key) {
    //如果通道是读就绪的
    if (key.isReadable()) {
        key.interestOps(key.interestOps() & (~SelectionKey.OP_READ));	
        SocketChannel ch = (SocketChannel) key.channel();
        ByteBuffer bf = (ByteBuffer) key.attachment();
        try {
            int n = ch.read(bf);
          	//读取数据（ascii)，并简单输出
            if (n > 0) {
                bf.flip();
                StringBuilder builder = new StringBuilder();
                while (bf.hasRemaining()) {
                    builder.append((char) bf.get());
                }
                System.out.print(builder.toString());
                bf.clear();
                key.interestOps(key.interestOps() | SelectionKey.OP_READ);
                key.selector().wakeup();
            } else if (n < -1) { //关闭连接
                ch.close();
            }
        } catch (IOException e) {
            //
        }
    }
}
```



## 选择器深入

### 可选择的操作

一共有四种可选择的操作（OP_READ、OP_WRITE、OP_ACCEPT、OP_ACCEPT），下为Socket通道对这四种可选择操作的支持情况

|                    | OP_READ | OP_WRITE | OP_ACCEPT | OP_CONNECT |
| ------------------ | ------- | -------- | --------- | ---------- |
| SocketChannel      | 支持    | 支持     | 不支持    | 支持       |
| SeverScoketChannel | 支持    | 支持     | 支持      | 不支持     |
| DatagramChannel    | 支持    | 支持     | 不支持    | 不支持     |

概括来说：客户端Socket通道不关心ACCEPT操作， 服务端Socket通道不关心CONNECT操作，数据报Socket通道只关心READ和WRITE操作

### 选择键(SelectionKey)

可选择的通道注册到选择器，然后返回一个选择键对象，即选择键代表通道和选择器的一个关联关系。主要需要了解他的几个属性

- interestOps：感兴趣的可选择操作集合，可选择通道注册到选取器的时候初始化，可以修改

- readyOps: 就绪的可选择操作集合， 选择器选择的时候会对该集合更新， 客户端不能修改。

- attachment：选择键可以关联一个对象，叫做附件（比如关联一个缓冲区对象）

### 选择器选择过程

在了解具体选择过程之前，我们先了解选择器中三个键集合的含义

- 已注册的键的集合，通过keys方法返回。通道注册的时候会把对应的选择键加入该集合

- 已选择的键的集合，选择器选择的时候会把就绪的通道对应的选择键放到该集合中
- 已取消的键的集合，选择键的cancel方法被调用后该选择键会加入该集合

具体选择过程：

1. 如果已取消的键的集合非空， 将每个已取消的键从其他两个集合中移除，并将相关的通道注销，最后将已取消键的集合清空。
2. 询问已注册键的集合中通道的就绪状态（系统调用），更新已选择键的集合以及键的readyOps集合。（如果一个键在该次选择操作之前就已在已选择键的集合中，则更新该键的readyOps集合；否则把键加入到已选择的集合中，并重置该键的readyOps集合）

### 最佳实践

> 我们通常使用一个选择器管理所有的可选择通道，并将就绪通道的服务委托给其他线程，只需要一个线程监控通道的就绪状态并使用一个协调好的工作线程池来读写数据

在这种方式下，进行相关操作前需要先将操作位从interestOps集合中移除，避免下次selector选择时再将选择键放到已选择集合中（造成一次就绪多次处理的问题），相关操作结束后再把操作加入到interestOps集合中并且唤醒selector进行下一次选择

下面是读操作简单伪码

```java
//读就绪
if (key.isReadable()) {
  	//把READ从interestOps中移除
		key.interestOps(key.interestOps() & (~SelectionKey.OP_READ));
		//......数据读取处理
  	//把READ加入到interestOps中
 		key.interestOps(key.interestOps() | SelectionKey.OP_READ);
  	//唤醒selector，重新选择（因为键的interestOps变化了）
 		key.selector().wakeup();
}
```



## 总结

1. ***可选择通道***注册到***选择器***上，返回***选择键*** 
2. 选择器核心思想：使用一个（或者少数个）选择器管理所有的可选择通道，每个选择器使用一个线程监控就绪状态，使用一个工作线程池处理就绪通道的数据读写





