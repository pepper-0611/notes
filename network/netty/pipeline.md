1.ChannelInboundInvoker: fire开头的方法抽象了所有IO线程流向用户handler的事件

2.ChannelOutboundInvoker：用户线程或者代码触发的事件read, write, connect, disconnect, close.... 

3.事件流向：IO线程触发的事件从HeadChannelContext开始处理（从头到尾），用户触发的事件从tail-->head

4.HeadContext: 用户触发的事件最后一个ChannelHanler，调用unsafe()对象进行真正的IO操作

5.TailContext: IO线程触发事件的最后一站路

6.DefaultChannelPipeline数据结构 head 和 tail,  双向链表

7.pipeline调用方法触发io事件的执行过程(connect, read等), tail.connect(AbstractChannelHeandlerConext中的抽象方法) --> 获取下一个AbstractChannelHanlerContext  next --> next.hanler().invokeConnect() --..... 到HeadContext的connect方法(ChannelHanler) --> unsafe().connect() --> 执行真正的连接

8.业务线程调用channel的io方法事件会到pipeline中去执行

