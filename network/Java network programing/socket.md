最近打算把Java网络编程相关的知识深入一下（IO、NIO、Socket编程、Netty)

Java网络编程主要涉及到对Socket和ServerSocket的使用上

阅读之前最好有TCP和UDP协议的理论知识以及Java I/O流的基础知识

[Java I/O流](https://juejin.im/post/6876684050917621774)

# TCP协议之上构建网络程序

## TCP协议的特点

- TCP是面向连接的协议，通信之前需要先建立连接

- 提供可靠传输，通过TCP传输的数据无差错、不丢失、不重复、并且按序到达

- 面向字节流（***虽然应用程序和TCP的交互是一次一个数据块，但是TCP把应用程序交下来的数据仅仅看成是一连串的无结构的字节流***）
- 点对点全双工通信
- 拥塞控制 & 滑动窗口

我们使用Java构建基于TCP的网络程序时主要关心客户端Socket和服务端ServerSocket两个类

## 客户端SOCKET

使用客户端SOCKET的生命周期：连接远程服务器 --> 发送数据、接受数据... --> 关闭连接

### 连接远程服务器

#### 通过构造函数连接

构造函数里指定远程主机和端口, 构造函数正常返回即代表连接成功， 连接失败会抛IOException或者UnkonwnHostException

```java
public Socket(String host, int port)
public Socket(String host, int port, InetAddress localAddr,int localPort)
```

#### 手动连接

当使用无参构造函数时，通信前需要手动调用connect进行连接（同时可设置SOCKET选项）

```java
Socket so = new Socket();
SocketAddress address = new InetSocketAddress("www.baidu.com", 80);
so.connect(address);
```

### 发送数据、接受数据

Java的I/O建立于流之上，读数据用输入流，写数据用输出流

下段代码连接本地7001端口的服务端程序，读取一行数据并且将该行数据回写服务端。

```java
 try (Socket so = new Socket("127.0.0.1", 7001)) {
     BufferedReader reader = new BufferedReader(new InputStreamReader(so.getInputStream()));
     BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(so.getOutputStream()));
     //read message from server
     String recvMsg = reader.readLine();
     //write back to sever.
     writer.write(recvMsg);
     writer.newLine();
     writer.flush();
  } catch (IOException e) {
     //ignore
  }
```

### 大端模式

大端模式是指数据的高字节保存在内存的低地址中（默认或者说我们阅读习惯都是大端模式）

### 关闭连接

Socket对象使用之后必须关闭，以释放底层系统资源

#### finally 块中关闭连接

```java
Socket so = null;
try {
  so = new Socket("127.0.0.1", 7001);
  //
}catch (Exception e){
	//
}finally {
  if(so != null){
    try {
      so.close();
    } catch (IOException e) {
      //
    }
  }
}
```

#### Try with resource 语法自动关闭连接

在try块中定义的Socket对象(以及其他实现了AutoCloseable的对象)Java会自动关闭

```java
//在try中定义的Socket对象(或其他实现了AutoCloseable的对象)Java会自动关闭
try (Socket so = new Socket("127.0.0.1", 7001)) {
		//do something
} catch(Exception e){
		//
}
```

## 服务端ServerSocket

使用ServerSocket的生命周期：绑定本地端口(服务启动) --> 监听客户端连接 --> 接受客户端连接  --> 通过该客户端连接与客户端进行通信 --> 监听客户端连接 --> .....(loop) --> 关闭服务器

### 绑定本地端口

直接在构造函数中指定端口完成绑定或者手工绑定

```java
//构造函数中指定端口完成绑定
ServerSokect ss = new  ServerSocket(7001);

//手工调用bind函数完成绑定
ServerSokect ss = new  ServerSocket();
ss.bind(new InetSocketAddress(7001));
```

### 接受客户端连接

accept方法返回一个Socket对象，代表与客户端建立的一个连接

```java
 ServerSokect ss = new ServerSocket(7001);  
 while(true){
    //阻塞等待连接建立
 		Socket so = ss.accept();
    // do something.
 }
```

### 与客户端进行通信

通过连接建立后的Socket对象，打开输入流、输出流即可与客户端进行通信

### 关闭服务器

同客户端Socket关闭一个道理

### Demo

下段代码服务器在连接建立时发送一行数据到客户端， 然后再读取一行客户端返回的数据，并比较这两行数据是否一样。

**主线程只接受客户端连接，连接建立后与客户端的通信在一个线程池中完成 **

```java
public class BaseServer {

    private static final String MESSAGE = "hello, i am server";
    private static ExecutorService threads = Executors.newFixedThreadPool(6);

    public static void main(String[] args) {
       //try with resource 写法绑定本地端口
        try (ServerSocket socket = new ServerSocket(7001)) {
            while (true) {
              	//接受客户端连接
                Socket so = socket.accept();
              	//与客户端通信的工作放到线程池中异步执行
                threads.submit(() -> handle(so));
            }
        } catch (IOException e) {
            //
        }
    }

    public static void handle(Socket so) {
       //try with resource 写法打开输入输出流
        try (InputStream in = so.getInputStream(); OutputStream out = so.getOutputStream()) {
            BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(out, "utf-8"));
            BufferedReader reader = new BufferedReader(new InputStreamReader(in));

            //send data to client.
            writer.write(MESSAGE);
            writer.newLine();
            writer.flush();

            //recv data from client.
            String clientResp = reader.readLine();
            System.out.println(MESSAGE.equals(clientResp));
        } catch (Exception e) {
            //ignore
        }finally {
          //关闭socket
            if(so != null){
                try {
                    so.close();
                } catch (IOException e) {
                    //
                }
            }
        }
    }
}
```

## Socket选项

### TCP_NODELAY

默认tcp缓冲是打开的，小数据包在发送之前会组合成更大的数据包发送， 在发送另一个包之前，本地主机需要等待对前一个包的确认-- Nagle算法

但是这种缓冲模式有可能导致某些应用程序响应太慢（比如一个简单的打字程序）

tcp_nodelay 设置为true关闭tcp缓冲, 所有的包一就绪就会发送

```java
 public void setTcpNoDelay(boolean on) 
```

###  SO_LINGER(linger是缓慢消失、徘徊的意思)

so_linger选项指定socket关闭时如何处理尚未发送的数据报，默认是close()方法立即返回，但是系统仍会将数据的数据发送

Linger 设置为0时，socket关闭时会丢弃所有未发送的数据

如果so_linger 打开且linger为正数，close()会阻塞指定的秒数，等待发送数据和接受确认，直到指定的秒数过去。

```java
public void setSoLinger(boolean on, int linger)
```

### SO_TIMEOUT

默认情况，尝试从socket读取数据时，read()会阻塞尽可能长的时间来获得足够多的字节

so_timeout 用于设置这个阻塞的时间，当时间到期抛出一个InterruptedException异常。

```java
public synchronized void setSoTimeout(int timeout)//毫秒，默认为0一直阻塞
```

### SO_KEEPLIVE

so_keeplive打开后，客户端每隔一段时间就发送一个报文到服务端已确保与服务端的连接还正常（TCP层面提供的心跳机制）

```java
public void setKeepAlive(boolean on)
```

### SO_RCVBUF 和SO_SNDBUF

设置tcp接受和发送缓冲区大小(内核层面的缓冲区大小)

对于传输大的数据块时（HTTP、FTP)，可以从大缓冲区中受益；对于交互式会话的小数据量传输（Telnet和很多游戏），大缓冲区没啥帮助

缓冲区最大大小 = 带宽 * 时延 （如果带宽为2Mb/s, 时延为500ms, 则缓冲区最大大小为128KB左右)

如果应用程序不能充分利用带宽，可以适当增加缓冲区大小，如果存在丢包和拥塞现象，则要减小缓冲区大小

# UDP协议之上构建网络程序

## UDP协议的特点

- 无连接。发送数据之前不需要建立连接，省去了建立连接的开销

- 尽力最大努力交付。数据报可能丢失、乱序到达

- 面向报文（**UDP对应用层交下来的报文，既不合并，也不拆分，而是保留这些报文的边界**）

- UDP没有拥塞控制

- UDP支持一对一、一对多、多对一和多对多的交互通信

- UDP的首部开销小，只有8个字节，比TCP的20个字节的首部还要短。

  

  构建UDP协议的网络程序时， 我们关系DatagramSocket和DatagramPacket两个类

## 数据报

UDP是面向报文传输的，对应用层交下来的报文不合并也不拆分（TCP就存在拆包和粘包的问题）

数据报关心两个事：存储报文的底层字节数组 和 通信对端地址（对端主机和端口）

```java
//发送数据报指定发送的数据和对端地址
DatagramPacket sendPacket = new DatagramPacket(new byte[0], 0, InetAddress.getByName("127.0.0.1"), 7002);

//接受数据报只需要指定底层字节数组以及其大小
DatagramPacket recvPacket = new DatagramPacket(new byte[1024], 1024);
```



## UDP客户端

因为UDP是无连接的，所以构造DatagramSocket的时候只需要指定本地端口， 不需要指定远程主机和端口

远程主机的主机和端口是指定在数据报中的，所以UDP可以实现一对一、一对多、多对多传输

```java
 try (DatagramSocket so = new DatagramSocket(0)) {
   //数据报中指定对端地址（服务端地址）
   DatagramPacket sendPacket = new DatagramPacket(new byte[0], 0,
                                                  InetAddress.getByName("127.0.0.1"), 7002);
   //发送数据报
   so.send(sendPacket);

   //阻塞接受数据报
   DatagramPacket recvPacket = new DatagramPacket(new byte[1024], 1024);
   so.receive(recvPacket);
   //打印对端返回的数据
   System.out.println(new String(recvPacket.getData(), 0, recvPacket.getLength()));
 } catch (Exception e) {
   e.printStackTrace();
 }
```



## UDP服务端

UDP服务端同客户端一样使用的是DatagramSocket， 区别在于绑带的本地端口需要显示申明

下面的UDP服务端程序接受客户端的报文，从报文中获取请求主机和端口，然后返回固定的数据内容 "received"

```java
byte[] data = "received".getBytes();
try (DatagramSocket so = new DatagramSocket(7002)) {
  while (true) {
    try {
      DatagramPacket recvPacket = new DatagramPacket(new byte[1024], 1024);
      so.receive(recvPacket);
      DatagramPacket sendPacket = new DatagramPacket(data, data.length,
                                                     recvPacket.getAddress(), recvPacket.getPort());
      so.send(sendPacket);
    } catch (Exception e) {
      //
    }
  }

} catch (SocketException e) {
  //
}
```

## 连接

UDP是无连接的， 但是DatagramSocket提供了连接功能对通信对端进行限制（并不是真的连接）

连接之后只能向指定的主机和端口发送数据报， 否则会抛出异常。

连接之后只能接收到指定主机和端口发送的数据报， 其他数据报会被直接抛弃。

```java
 public void connect(InetAddress address, int port)
 public void disconnect() 
```



# 总结

Java 中TCP编程依赖于 Socket和ServerSocket，UDP编程依赖于DatagramSocket和DatagramPacket