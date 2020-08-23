# 1.Netty简介

netty可以由一句话概括：**异步<sup>①</sup>**的**事件驱动<sup>②</sup>**的**网络应用<sup>③</sup>**程序框架

![](./images/netty架构.png)

netty基于java-nio，用来快速开发高性能、高可靠性的网络服务器和客户端程序。使用 netty 可以确保快速和简单地开发出一个网络应用，例如实现了某种协议的客户，服务端应用。netty 相当简化和流线化了网络应用的编程开发过程，例如：TCP和UDP的socket 服务开发！

# 2.Reactor模式

netty是按照Reactor模式去设计整体架构，何为Reactor模式？它是一种基于事件驱动的设计模式，将一个或多个客户端请求分离（demultiplex）和调度（dispatch）给事件处理器(event handler)处理。换句话说，注册感兴趣的事件→扫描是否有感兴趣的事件发生→事件发生后作出相应处理

reactor模式就是来解决`thread per connection`问题的，它的设计理念是以事件作为驱动，例如：在一次TCP连接中，从开始连接到数据交互完成，中间可以划分为多个事件如：连接就绪、读数据就绪、写数据就绪、连接关闭...每次事件发生都会回调对应的处理器（处理器即thread）

这种模式将服务端线程的职能划分得更细致，以往服务端线程都是一条龙服务，从连接就绪到解析数据，读取数据最终写回数据，都在一个线程中完成。但reactor模式划分更细致，读事件有读处理器回调，写事件有写处理器回调。Java的nio包下的Selector本质上就是IO多路复用器，也是实现Reactor模式的基础

## 2.1.演变过程

以下图总结自大神Doug Lea的Reactor模式一文，膜拜大佬！[http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)

### 2.1.1.thread_per_connection

对于TCP长连接，服务端通常都要维护与客户端关联的Socket，这就意味着每个客户端，服务端都需要新起一个线程去接收它的请求，这种编程模式是经典的、古老的`Thread per connection`，一个请求一个线程

![](./images/thread_per_connection.png)

这种编程模式优点是简单，适用于小型项目，并发量较少；对于高并发量的系统，一个请求一个线程，频繁地创建和销毁线程，本身就会对系统造成巨大的性能开销。虽然，在java中有了线程池的概念，虽然可以有效地减少创建线程带来的性能损耗，但却治标不治本，没从根本上解决一个请求一个线程的处理方式，一旦某次请求耗时，线程池任务用光，服务端就无法接收客户端的请求。伪代码如下：

```java
class Server implements Runnable {
    public void run() {
        while (!Thread.interrupted()){
            try {
                // 初始化ServerSocket进行监听
                ServerSocket ss = new ServerSocket(8080);
                // 阻塞等待客户端发起连接
                Socket socket = ss.accept();
                // 接收到客户端连接后, 开启一个线程处理
                executorService.execute(new Handler(socket));
            }catch(Exception ex){
                // 处理各种异常
            }
        }
    }

    static class Handler implements Runnable {
        final Socket socket;
        Handler(Socket s) { 
            socket = s; 
        }
        public void run() {
            try {
                // 解码
                byte[] input = new byte[MAX_INPUT];
                socket.getInputStream().read(input);
                // 逻辑处理
                byte[] output = process(input);
                // 重新编码, 写回Socket
                socket.getOutputStream().write(output);
            } catch (IOException ex) {

            }
        }

        /**
         * 实际处理逻辑
         */
        private byte[] process(byte[] cmd) {}
    }
}
```

### 2.1.2.single_thread_reactor

单线程Reactor模型，相当于所有的I/O事件（连接、读、写）都在一个线程里处理，这个线程就是创建并启动Reactor的线程。

![](./images/Reactor单线程模型.png)

但是，这种模型有个缺点，相当于把所有活儿都给Reactor干了，一是处理效率慢，二是如果该线程挂了，那服务就崩溃了。Reactor伪代码：

```java
public class Reactor implements Runnable {
    // 基于java.nio
    final Selector selector;
    final ServerSocketChannel serverSocket;

    public Reactor(int port) throws IOException {
        // 创建一个Selector, 同时开启一个ServerlSocketChannel
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        // 绑定端口
        serverSocket.socket().bind(new InetSocketAddress(port));
        // 配置为非阻塞模式
        serverSocket.configureBlocking(false);
        // 服务端只对连接事件感兴趣
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        // 选择键绑定一个Accector组件
        sk.attach(new Acceptor(serverSocket));
    }

    public void run() { 
        try {
            // Reactor一直在死循环, 等待客户端的连接
            while(!Thread.interrupted()){
                // 阻塞在监听事件上, 等待事件返回
                selector.select();
                // 感兴趣的事件发生, Selector#select()会返回, 返回值类型为SectionKey集合
                Set<SectionKey> selected = selector.selectedKeys();
                Iterator it = selected.iterator();
                while(it.hasNext()){
                    SectionKey sectionKey = it.next;
                    // 分发事件
                    dispatch(sectionKey);
                    // 清除事件
                    it.remove();
                }
            }
        }catch(Exception e){
            // 处理异常
        }
    }
    
    void dispatch(SelectionKey k) {
        // 根据SectionKey类型的不同, 调用不同的逻辑处理。这边统一抽调出接口Runnable, 然后不同的
        // 处理逻辑实现Runnable接口。例如Acceptor组件的逻辑只负责接收连接; Handler组件的逻辑负责
        // 处理读写请求
        Runnable r = (Runnable)(k.attachment());
        if (r != null){
            r.run();
        }
    }
}
```

Acceptor组件伪代码，它只负责接收客户端的连接：

```java
public class Acceptor() implements Runnable {
    private final ServerSocket serverSocket;
    public Acceptor(ServerSocket serverSocket){
        this.serverSocket = serverSocket;
    }
       
    /**
     * 这边会由Reactor组件的dispatch()方法来调用
     */
    public void run() {
        try {
            SocketChannel channel = serverSocket.accept();
            if (c != null){
                // 接收到连接后, 创建一个Handler实例, 处理读写请求
                new Handler(selector, channel);
            }
        }catch(IOException ex) { /*异常处理*/ }
    }
}
```

Handler组件伪代码，它负责读写逻辑。每来一个客户端连接，都会创建一个Handler处理

```java
final class Handler implements Runnable {

    static final int READING = 0, SENDING = 1;
    final SocketChannel socket;
    final SelectionKey sk;
    ByteBuffer input = ByteBuffer.allocate(MAXIN);
    ByteBuffer output = ByteBuffer.allocate(MAXOUT);
    int state = READING;

    public Handler(Selector selector, SocketChannel channel) throws IOException {
        // 保存与客户端连接的通道Channel, 并将其配置为非阻塞模式
        this.socket = channel; 
        this.socket.configureBlocking(false);
        // 往同一个Selector中注册读写事件
        sk = socket.register(selector, 0);
        // 会返回一个选择键, 将当前Handler关联进去, 一旦事件发生, 可以调用此Handler的方法
        sk.attach(this);
        // 刚创建的只对读事件感兴趣
        sk.interestOps(SelectionKey.OP_READ);
        // 唤醒Selector, 避免还在select()方法上阻塞
        selector.wakeup();
    }
    
    boolean inputIsComplete() { /*返回数据是否读取完成*/ }
    boolean outputIsComplete() { /*返回数据是否写出完成*/ }
    void process() { /*具体读逻辑*/ }

    /**
     * 此方法被Reatcor的dispatch()方法调用
     */
    public void run() {
        try {
            if (state == READING){
                 read();
            }else if(state == SENDING){
                send();
            }
        } catch (IOException ex) { /*异常处理*/ }
    }
    
    /**
     * 实际读逻辑
     */
    void read() throws IOException {
        // 将数据读取到缓冲区中
        socket.read(input);
        if (inputIsComplete()) {
            // 实际读事件处理逻辑
            process();
            // 更改状态为发送
            state = SENDING;
            // 读事件处理完成以后, 感兴趣事件变为写
            sk.interestOps(SelectionKey.OP_WRITE);
        }
    }
    
    /**
     * 实际写逻辑
     */
    void send() throws IOException {
        socket.write(output);
        if (outputIsComplete()) {
            // 数据写回完成后, 释放SectionKey
            sk.cancel();
        }
    }
}
```

### 2.1.3.multi_thread_reactor

多线程Reactor模型，就是将上一步单线程Reactor模型的Handler组件从Reactor中解放出来，不在运行Reactor的线程中调用，而是启用一个线程池，可以称该线程池为Workers，交由它来执行读写逻辑。

![](./images/Reactor多线程模型.png)

其实很容易可以看出单线程Reactor模型的缺点，那就是如果业务逻辑处理比较耗时，其实是会占用Reactor所在线程的效率，那就没办法及时响应其它客户端的请求。所以把业务逻辑放到线程池中执行，其实也就是将Handler异步执行，伪代码为：

```java
class Handler implements Runnable {
    // 开启一个线程池
    static PooledExecutor pool = new PooledExecutor(...);
    static final int PROCESSING = 3;
    
    /**
     * 此方法被Reatcor的dispatch()方法调用
     */
    public void run() {
        try {
            if (state == READING){
                 read();
            }else if(state == SENDING){
                send();
            }
        } catch (IOException ex) { /*异常处理*/ }
    }
    
    // 读请求
    synchronized void read() { 
        socket.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            // 业务逻辑放到线程池中执行
            pool.execute(new Processer());
        }
    }
    
    // 在线程池中处理业务逻辑
    class Processer implements Runnable {
        public void run() { processAndHandOff(); }
    }
    
    // 实际处理请求
    synchronized void processAndHandOff() {
        process();
        state = SENDING;
        sk.interest(SelectionKey.OP_WRITE);
    }
    
}
```

### 2.1.2.master_slave_multi_thread_reactor

主从多线程Reactor模式，将职责更细分化，前面两个Reactor，不仅要监听连接事件，还要监听读写事件。而主从Reactor模型将这个职责更细分化，一个Reactor负责处理连接请求，另一个Reactor处理读写请求。其实netty也采用了这种设计模型，如netty常用的bossGroup和workerGroup。

这种编程模型组件功能如下：

1. mainReactor只负责接收请求，它会将客户端的请求封装成acceotor，将它交予subReactor。然后它就继续等待接收客户端请求（不会处理请求）

2. subReactor负责处理客户端请求，当客户端有数据传入时，通过线程池处理这些数据。实际上，就是在Socket对应事件发生时回调对应处理器。

![](./images/Reactor主从模型.png)

## 2.2.编程模型

上面是Doug Lea在《Scalable IO in Java》一文中针对于java提出reactor编程模式。还有一篇更古老的博客《reactor-siemens》诠释了Reactor模式的架构设计，也是很经典的一张图：

<img src="./images/Reactor模式的架构设计.png" style="zoom:80%;" />

### 2.2.1.五大组件

上图5个带有背景颜色的方框就是一个组件，它们作用分别是：

1. **Handle-事件句柄**。由操作系统提供，表示一种资源，该资源用来表示一个个事件，如网络编程中的Socket描述符，Handle是事件的发源地；

2. **Event Handler–事件处理器**。自身由多个回调方法构成，这些回调方法会在事件Handle发生时被调用(被初始分发器调用)。特别注意：在java nio中没有事件处理器的概念，而Netty基于nio升级了架构，它完善了事件处理器，在IO事件发生时，提供了相应的回调方法；

3. **Concrete Event Handler–事件处理器实现**。就是Event Handler的具体实现，在每个回调方法实现具体逻辑，这个组件就是我们变成的处理器实现；

4. **Synchronous Event Demultiplexer-同步事件分离器**。它本身是一个底层操作系统调用，用来等待事件的发生。调用方在调用它的时候会被阻塞，一直阻塞到同步事件分离器上有事件发生为止。对于Linux而言，同步事件分离器就是IO多路复用机制，比如select、poll、epoll等；在java nio中同步事件分离器就是选择器Selector，对应阻塞方法即select()方法；

5. **Initiation Dispatcher–初始分发器**。实际上就是[上图](#2.1.2.reactor)Doug Lea诠释的 Reactor角色，它提供了两大功能：其一是事件处理器的注册与销毁；其二是事件的调度。Initiation Dispatcher通过同步事件分离器等待事件的发生， 一旦事件发生(单个或多个)它会分离出每一个事件，选择注册在它身上的事件处理器，最后调用相关回调方法来处理这些事件。

### 2.2.2.执行流程

下图左侧文字代表每个步骤：

![](./images/reactor执行流程.png)

1. **initialize**。主程序(main program)初始化分发器Initiation Dispatcher；

2. **register handler**。向Initiation Dispatcher注册具体的事件处理器，同时标识出该事件处理器希望分发器在某个事件发生时向其通知该事件，这个事件会与Handle关联；

3. **extract handle**。Initiation Dispatcher会要求每个事件处理器向其传递内部的Handle，该handle向操作系统标识了事件处理器；

4. **run event loop**。当所有事件处理器注册完毕，主程序调用handle_events()方法启动Initiation Dispatcher的事件循环。此时，Initiation Dispatcher会将每个注册的事件管理器的handle合并起来，通过同步事件分离器等待事件发生；

5. **dispatch handler**。当事件对应的handle变为ready状态(即可用状态)，同步事件分离器就通知分发器，分发器就会通过这个handle选择恰当的事件处理器回调方法。

## 2.3.Netty对Reactor的支持

netty用它巧妙的设计，很简单地可以实现不同Reactor的效果：

- 单线程Reatcor

  ```java
  EventLoopGroup eventGroup = new NioEventLoopGroup(1);
  ServerBootstrap serverBootstrap = new ServerBootstrap();
  serverBootstrap.group(eventGroup);
  ```

- 多线程Reatcor

  ```java
  EventLoopGroup eventGroup = new NioEventLoopGroup();
  ServerBootstrap serverBootstrap = new ServerBootstrap();
  serverBootstrap.group(eventGroup);
  ```

- 主从Reactor

  ```java
  EventLoopGroup parentGroup = new NioEventLoopGroup(1);
  EventLoopGroup childGroup = new NioEventLoopGroup();
  ServerBootstrap serverBootstrap = new ServerBootstrap();
  serverBootstrap.group(parentGroup, childGroup);
  ```

# 3.Netty缓冲区

## 3.1.三种缓冲区

netty提供了三种不同的缓冲区，都可以由io.netty.buffer.Unpooled获取：

**①heap buf**

- 实例：Unpooled.buffer()；

- 描述：最常用类型，将数据存储到JVM堆，实际数据存放到字节数组上；

- 优点：可以快速地创建和释放，并且提供直接访问内部字节数组的方法；

- 缺点：每次读写数据都需要先将数据复制到直接缓冲区中再进行网络传输；

**②direct buf**

- 实例：Unpooled.directBuffer

- 描述：在JVM堆外分配内存空间，由操作系统在本地内存进行数据分配

- 优点：Socket传输时，由于数据直接位于操作系统本地内存，性能很好

- 缺点：内存空间的分配与释放要比heap buf更加复杂且速度要慢.(由于direct buf分配要慢，netty采用**内存池**方式解决！)

- 建议：对于业务处理用heap buf；对于I/O通信线程用direct buf；

**③composite buf**

- 实例：Unpooled.compositeBuffer()

- 描述：复合缓冲区，可以容纳其它ByteBuf(堆或非堆)，对它们统一管理

## 3.2.ByteBuf

netty针对缓冲区自己定义了一个io.netty.buffer.ByteBuf类，它内部维护了两个指针：readIndex和writeIndex。它们会把一个Bytebuf分为3个区域：(0,readIndex]**已读区域**(可丢弃)；(readIndex,writeIndex]**可读区域**(数据未读)；(wirteIndex,capacity]**可写区域**(数据空位置)。

![](./images/netty-ByteBuf.png)

# 4.Netty事件循环

<img src="./images/netty事件循环组类关系.png" style="zoom:80%;" />

上图是netty整个事件循环组的类继承图，主要从EventExecutorGroup开始往下看，再上层的父接口是JDK提供的并发包的内容，基础是线程池中可以执行周期任务的线程池服务。所以从这可以知道Netty可以实现周期任务，比如心跳检测。

## 4.1.EventLoop & EventLoopGroup

EventLoop，事件循环，用来处理Channel的所有I/O事件；

EventLoopGroup，事件循环组，用来管理多个EventLoop。

![](./images/EventLoopGroup和EventLoop的关系.png)

EventLoop、EventLoopGroup、Thread、Channel之间的相互关系：

1. 一个EventLoopGroup可以包含一个或多个EventLoop，即1:n

2. 一个EventLoop只能与一个Thread绑定，即1:1

3. 一个Channel只能注册到一个EventLoop上，即1:1

4. 一个EventLoop可以分配给多个Channel，即1:n。 

每当一个新连接到达，netty就会创建一个Channel，然后从EventLoopGroup中分配出一个EventLoop与该Channel绑定，然后在这个Channel整个生命周期中都是由这个EventLoop来处理它的IO事件。而且，所有EventLoop处理的I/O事件都交由它内部唯一一个专用的Thread来处理，从而保证线程安全！

**重要结论：**

不要将长时间执行的耗时任务放入到EventLoop的执行队列中，因为它将会一直阻塞该线程所对应的所有Channel上的其它执行任务，若需要进行阻塞调用或耗时的操作，则建议使用一个专门的EventExecutor(业务线程池)。

#  5.Netty处理器

## 5.1.ChannelHandler

ChannelHandler，通道处理器，主要用来处理各种事件。ChannelHandler 有两个核心子类 ChannelInboundHandler 和 ChannelOutboundHandler，其中 ChannelInboundHandler 用于接收、处理入站( Inbound )的数据和事件，而 ChannelOutboundHandler 则相反，用于接收、处理出站( Outbound )的数据和事件。除此之外，netty还提供了ChannelDuplexHandler，它可以同时用于接收、处理入站和出站的数据。

```java
public interface ChannelHandler {
  /**
   * 当此ChannelHandler被添加到ChannelPipeline中就会触发此方法.
   * 一般用于ChannelHandler的初始化逻辑.
   */
  void handlerAdded(ChannelHandlerContext ctx) throws Exception;

  /**
   * 当此ChannelHandler从ChannelPipeline移除就会触发此方法.
   * 一般用于ChannelHandler的销毁逻辑
   */
  void handlerRemoved(ChannelHandlerContext ctx) throws Exception;

  /**
   * 此方法已被取消, 转而去ChannelInboundHandler
   */
  @Deprecated
  void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;

  /**
   * 通常一个ChannlHandler只能注册在一个ChannelPipeline上, 但是如果加了这个注解,
   * 它就可以注册到多个ChannelPipeline上.
   */
  @Inherited
  @Documented
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @interface Sharable {
    // no value
  }
}
```

### 5.1.1.入站处理器

入站处理器即ChannelInboundHandler，它提供了在Channel状态变化时提供钩子函数，便于我们做出相应的处理逻辑。除此之外，netty还提供了一个适配器ChannelInboundHandlerAdapter，它实现了ChannelInboundHandler接口，里面方法实现很简单，就是通过上下文ChannelHandlerContext.firexxxx()方法将事件传播到下一个处理器。

```java
public interface ChannelInboundHandler extends ChannelHandler {
  /**
   * 当Channel被注册到EventLoop上, 会触发此方法
   */
  void channelRegistered(ChannelHandlerContext ctx) throws Exception;

  /**
   * 当Channel从EventLoop上被移除时, 就会触发此方法
   */
  void channelUnregistered(ChannelHandlerContext ctx) throws Exception;

  /**
   * 当Channel被激活时, 就会触发此方法
   */
  void channelActive(ChannelHandlerContext ctx) throws Exception;

  /**
   * 已注册的Channel处于非Active状态并且已到达它的生命周期, 触发此方法
   */
  void channelInactive(ChannelHandlerContext ctx) throws Exception;

  /**
   * 当Channel的对端有数据可读时, 就会触发此方法
   */
  void channelRead(ChannelHandlerContext ctx, Object msg);

  /**
   * 当上面的channelRead()方法读取到最后一条消息时, 就会触发此方法
   */
  void channelReadComplete(ChannelHandlerContext ctx) throws Exception;

  /**
   * 当用户事件触发时就会调用此方法
   */
  void userEventTriggered(ChannelHandlerContext ctx, Object evt);

  /**
   * 当Channel的可写状态被改变时, 就会触发此方法
   */
  void channelWritabilityChanged(ChannelHandlerContext ctx);

  /**
   * 在这个ChannelHandler处理事件发生异常时, 就会触发此方法
   */
  @Override
  @SuppressWarnings("deprecation")
  void exceptionCaught(ChannelHandlerContext ctx, Throwable cause);
}
```

### 5.1.2.出站处理器

ChannelOutboundHandler会处理I/O出站（将数据主动发送给对端）的操作。除此之外，netty还提供了一个适配器ChannelOutboundHandlerAdapter，它实现了ChannelOutboundHandler接口，里面方法实现很简单，直接通过上下文ChannelHandlerContext调用同名方法。

```java
public interface ChannelOutboundHandler extends ChannelHandler {
  /**
   * 当一个Channel绑定操作完成, 就调用此方法
   * @param ctx 表示为这个上下文对象进行绑定端口操作
   * @param localAddress  绑定到这个地址上
   * @param promise 一旦绑定操作完成, 使用这个ChannelPromise去通知所有监听器
   */
  void bind(ChannelHandlerContext ctx, SocketAddress localAddress, 
            ChannelPromise promise) throws Exception;

  /**
   * 当一个Channel连接操作完成, 就调用此方法
   * @param ctx t表示为这个上下文对象进行通道连接操作
   * @param remoteAddress 待连接的远程地址
   * @param localAddress 本地地址, 使用它取连接远程地址
   * @param promise 连接操作完成, 通过它去通知所有的监听器
   */
  void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,
               SocketAddress localAddress, ChannelPromise promise) throws Exception;

  /**
   * 当通道断开连接时, 就会调用此方法
   * @param ctx t表示为这个上下文对象进行断开连接操作
   * @param promise s断开连接操作完成, 通过这个ChannelPromise通知旗下的监听器
   */
  void disconnect(ChannelHandlerContext ctx, ChannelPromise promise);

  /**
   * 当通道关闭时, 就会调用此方法
   * @param ctx 表示为这个上下文对象进行通道关闭操作
   * @param promise t当操作完成, 通过这个ChannelPromise通知旗下的监听器
   */
  void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;

  /**
   * 从当前注册的EventLoop执行注销操作后调用
   */
  void deregister(ChannelHandlerContext ctx, ChannelPromise promise);

  /**
   * 拦截{@link ChannelHandlerContext#read()}的数据读取操作
   */
  void read(ChannelHandlerContext ctx) throws Exception;

  /**
   * 通过ChannelPipeline写入消息, 当写操作完成时就会调用此方法. 这些数据会在调用
   * flush()后从底层Channel中发送给对端.
   * @param ctx 为这个上下文对象执行写操作
   * @param msg 待写入的消息
   * @param promise t当写入操作完成, 通过这个promise通知旗下的所有监听器
   */
  void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise);

  /**
   * 刷新操作将尝试刷新所有以前写入的挂起消息, 操作完成后就会触发这个方法
   * @param ctx 为这个上下文执行刷新操作
   */
  void flush(ChannelHandlerContext ctx) throws Exception;
}
```

## 5.2.处理器上下文

### 5.2.1.ChannelHandlerContext

ChannelHandlerContext是连接ChannelHandler与ChannelPipeline的桥梁，它可以让一个ChannelHandler与它所在的ChannelPipeline中的其它ChannelHanler进行交互。

```java
public interface ChannelHandlerContext extends AttributeMap, 
ChannelInboundInvoker, ChannelOutboundInvoker {
  /**
   * 返回绑定在这个处理器上下文的通道Channel
   */
  Channel channel();

  /**
   * 返回用来执行任意任务的事件处理器EventExecutor
   */
  EventExecutor executor();

  /**
   * 当前上下文的唯一名称
   */
  String name();

  /**
   * 返回绑定在当前上下文的处理器ChannelHandler
   */
  ChannelHandler handler();

  /**
   * 与此上下文关联的ChannelHandler从ChannelPipeline移除时, 此方法就返回返回true.
   */
  boolean isRemoved();

  /**
   * 以下方法从ChannelInboundInvoker接口继承并修改了其返回值, 用来触发入站处理器定义
   * 的各个事件.
   */
  ChannelHandlerContext fireChannelRegistered();
  ChannelHandlerContext fireChannelUnregistered();
  ChannelHandlerContext fireChannelActive();
  ChannelHandlerContext fireChannelInactive();
  ChannelHandlerContext fireExceptionCaught(Throwable cause);
  ChannelHandlerContext fireUserEventTriggered(Object evt);
  ChannelHandlerContext fireChannelRead(Object msg);
  ChannelHandlerContext fireChannelReadComplete();
  ChannelHandlerContext fireChannelWritabilityChanged();

  // 下面两个方法从ChannelOutboundInvoker接口继承并修改其返回值
  ChannelHandlerContext read();
  ChannelHandlerContext flush();


  /**
   * 返回此上下文关联的管道ChannelPipeline
   */
  ChannelPipeline pipeline();

  /**
   * 返回此上下文关联的缓冲区分配器ByteBufAllocator, 用来申请ByteBuf
   */
  ByteBufAllocator alloc();

  // 下面两个方法继承自AttributeMap
  <T> Attribute<T> attr(AttributeKey<T> key);
  <T> boolean hasAttr(AttributeKey<T> key);
}

```

### 5.2.2.AbstractChannelHandlerContext

AbstractChannelHandlerContext是ChannelHandlerContext的抽象实现，基本上管道ChannelPipeline使用的上下文对象都是这个类，它是一个双向链表结构，其类信息如下：

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap
  implements ChannelHandlerContext, ResourceLeakHint {
  // 前驱节点和后继节点
  volatile AbstractChannelHandlerContext next;
  volatile AbstractChannelHandlerContext prev;
  // 上文下状态变量handlerState的原子更新器
  private static final AtomicIntegerFieldUpdater<AbstractChannelHandlerContext>
    HANDLER_STATE_UPDATER =AtomicIntegerFieldUpdater.newUpdater(
    AbstractChannelHandlerContext.class, "handlerState");
  
  /* 下面4个变量, 表示上下文状态handlerState的取值 */
  // 表示ChannelHandler.handlerAdded()即将被调用
  private static final int ADD_PENDING = 1;
  // 表示ChannelHandler.handlerAdded()已经被调用
  private static final int ADD_COMPLETE = 2;
  // 表示ChannelHandler.handlerRemoved()已经被调用
  private static final int REMOVE_COMPLETE = 3;
  // 表示ChannelHandler的handlerAdded()和handlerRemoved()都未被调用
  private static final int INIT = 0;
  
  // 若此上下文关联的ChannelHandler是入站处理器则inbound为true, 反之outbound为true
  private final boolean inbound;
  private final boolean outbound;
  
  // 关联的ChannelPipeline对象
  private final DefaultChannelPipeline pipeline;
  
  // 此上下文的唯一名称
  private final String name;
  
  // 此上下文使用的EventExecutor是否实现了OrderedEventExecutor
  private final boolean ordered;
  
  // 每个上下文会关联一个事件执行器EventExecutor
  final EventExecutor executor;
  
  // 成功的 ChannelFuture对象
  private ChannelFuture succeededFuture;
  
  // 执行 Channel ReadComplete 事件的任务
  private Runnable invokeChannelReadCompleteTask;
  
  // 执行 Channel Read 事件的任务
  private Runnable invokeReadTask;
  
  // 执行 Channel WritableStateChanged 事件的任务
  private Runnable invokeChannelWritableStateChangedTask;
  
  // 执行 flush 事件的任务
  private Runnable invokeFlushTask;
  
  // 表示此上下文的状态, 为上面定义的四种
  private volatile int handlerState = INIT; 
}
```

要注意一点，ChannelHandler在上下文对象ChannelHandlerContext中是使用AttributeKey来存储状态，因此如果同一个ChannleHandler多次添加到ChannelPipeline却指定不同的名称，即使都是同一个ChannelHanler，也会被调用四次，因为它们在上下文中的状态不一样，如：

```java
MyHandler fh = new MyHandler();

ChannelPipeline p1 = Channels.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);

ChannelPipeline p2 = Channels.pipeline();
p2.addLast("f3", fh);
p2.addLast("f4", fh);
```

## 5.3.编码器

**编解码器：**无论向网络中写入的数据是什么类型（int、char、String..），数据在网络中传输，都是以字节流的形式呈现的；将数据由原先形式转换为字节流的操作称为编码（encode），将数据由字节转换为它原本的格式或其它格式的操作称为解码（decode），编解码统一称为codec！netty提供专门处理编解码的[ChannelHanlder](#5.1.ChannelHandler)，这些处理器就称为编解码器。

**编码器：**本质上是出站处理器，因此编码一定是[ChannelOutboundHandler](#5.1.2.出站处理器)

**解码器：**本质上以入站处理器，因此解码一定是[ChannelInboundHandler](#5.1.1.入站处理器).

（一般情况下，编码器以xxxEncoder命名，解码器以xxxDecoder命令）

1.编解码器接收的消息类型必须与待处理的参数类型一致，否则它不执行；

2.解码器进行数据解码时，一定要判断ByteBuf的字节数是否足够！！

### 5.3.1.ByteToMessageDecoder

netty最顶层的解码器抽象类：ByteToMessageDecoder

### 5.3.2.MessageToByteEncoder

netty最顶层的编码器抽象类：MessageToByteEncoder

## 5.4.常用组件

### 5.4.1.SimpleChannelInboundHandler

....

### 5.4.2.ChannelInitializer

....

# 6.Netty通道

## 6.1.Channel

netty相较于nio的通道自定义了一个组件io.netty.channel.Channel。它是netty网络操作抽象类，除了包括基本的 I/O 操作，如 bind、connect、read、write 之外，还包括了netty框架相关的一些功能，如获取该 Channel 的 EventLoop、获取通道配置ChannelConfig等等。

netty不仅提供nio支持，也提供了bio支持(但没提供aio支持)，所有已Nio开头的Channel如NioServerSocketChannel都是基于nio实现的。所有已Oio开头的Channel如OioServerSocketChannel都是原始的bio实现，意味着它是阻塞式的。

**重要结论：**

channel的实现一定是线程安全，基于此可以储存channel的引用，通过该引用向远端发送数据，即便当时有多个线程同时发送也不会出现并发问题；而且，消息一定会按顺序发送出去。

## 6.2.ChannelPipeline

ChannelPipeline即管道。当一个Channel创建好后，与之相对应的管道就会自动创建完毕，默认实现类：DefaultChannelPipeline。管道内维护着[ChannelHandlerContext](#5.2.处理器上下文)的链表集，再由它去维护ChannelHandler的关系。因此，ChannelPipeline本质是ChannelHandler集合，内部细分为ChannelInboundHandler和ChannelOutboundHandler，用来处理与之关联的Channel的数据流入和流出，形如：

![](./images/netty-ChannelPipeline流程.png)

与ChannelPipeline(管道)关联的一方是用户程序，一方是netty底层IO线程，当客户端传入数据，就会从Socket.read()读入，经过管道一个一个的ChannelInboundHandler处理，到达最后一个handler处理；若是数据写出则从最后一个ChannelOutboundHandler开始，到达netty底层的IO线程，最后socket.write()写出。处理顺序例如：

```java
ChannelPipeline p = ch.pipeline();
p.addLast("1", new InboundHandlerA());
p.addLast("2", new InboundHandlerB());
p.addLast("3", new OutboundHandlerA());
p.addLast("4", new OutboundHandlerB());
p.addLast("5", new InboundOutboundHandlerX());
```

则数据流入的顺序为：1→2→5。数据流出的顺序为：5→4→3，其实就像spring mvc的拦截器链一样，进来先处理的，出去就会晚处理。ChannelPipeline提供了许多以fire开头的方法，用来将事件交于下一个handler：

<img src="./images/netty-ChannelPipeline-api接口.png" style="zoom:80%;" />

## 6.3.ChannelFuture

netty的io.netty.util.concurrent.Future接口继承自JDK的java.util.concurrent.Future。因为netty是采用异步处理的逻辑，所以它的方法调用都会立即返回，类型就是这个Future。而且，netty的Future接口扩展JDK的Future接口，增加了监听器io.netty.util.concurrent.GenericFutureListener，它会在异步执行完后，以回调的方式主动通知监听器（标准的观察者模式），需要注意的是监听器中的operationComplete()方法是由I/O线程调用的，切记不要在此执行耗时操作！

ChannelFuture接口继承了Future接口。ChannelFuture 接口通过该接口的 #addListener(...) 方法，注册一个 ChannelFutureListener，当操作执行成功或者失败时，监听就会自动触发返回结果。

**tips:**

 <u>在java，区别方法是通过：方法名+参数，没有通过返回值来区分方法的。所以可以看到Future和ChannelFuture的#addListener()方法返回值不一样，但是ChannelFuture却覆盖Future#addListener()方法</u>

## 6.4.ChannelPromise

netty的io.netty.util.concurrent.Promise接口继承了io.netty.util.concurrent.Future接口，它是一种特殊的Future，在它的基础上提供了**写入**的功能。

ChannelPromise接口继承了ChannelFuture接口和Promise接口，同样提供了#addListener()方法，注册ChannelFutureListener，当通道操作执行成功或失败时，自动触发返回结果；并且ChannelPromise可以手动设定通道操作的成功与失败！

# 7.Netty配置类

## 7.1.AbstractBootstrap

netty提供了ServerBootstrap和Bootstrap用来便捷地配置netty服务，这两个类都实现了AbstractBootstrap抽象类。它们实际上都是配置类，用来启动一个netty服务，而且针对Reactor模式，AbstractBootstrap保存得都是bossgroup的配置，而其子类(如ServerBootstrap)保存的都是childgroup的配置。

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, 
C extends Channel> implements Cloneable {
  // 配置事件循环组
  volatile EventLoopGroup group;
  
  // 配置Channel生成工厂
  private volatile ChannelFactory<? extends C> channelFactory;
  
  // 配置本地地址
  private volatile SocketAddress localAddress;
  
  // 配置ChannelOption. ChannelOption是作为一个key来为Channel配置时设值用的
  private final Map<ChannelOption<?>, Object> options = 
    	new LinkedHashMap<ChannelOption<?>, Object>();
  
  // 配置属性Attribute, 以AttributeKey作为key
  private final Map<AttributeKey<?>, Object> attrs = 
    	new LinkedHashMap<AttributeKey<?>,Object>();
  
  // 配置用于bossGroup的通道处理器ChannelHandler
  private volatile ChannelHandler handler;
}
```

## 7.2.ServerBootstrap

ServerBootstrap通常用来配置和启动netty服务端的工具类，它内部定义的属性用来给workerGroup配置。若调用它定义的childxx()方法就是给workerGroup配置的，其它则是调用super()方法给bossGroup配置：

```java
private final Map<ChannelOption<?>, Object> childOptions = 	new LinkedHashMap<ChannelOption<?>, Object>();
private final Map<AttributeKey<?>, Object> childAttrs = 
  new LinkedHashMap<AttributeKey<?>, Object>();
private final ServerBootstrapConfig config = new ServerBootstrapConfig(this);
private volatile EventLoopGroup childGroup;
private volatile ChannelHandler childHandler;
```

# 8.Netty内存池

netty设计了一套内存池（参考jemalloc的原理），用来对内存进行管理，netty内存管理实质就是先分配一块大内存，然后在内存的分配和回收过程中，使用一些数据结构记录内存使用状态，如果有新的分配请求，根据这些状态信息寻找最合适的位置并更新内存结构。释放内存时候：同步修改数据结构。

## 8.1.内存管理粒度

netty内存管理的粒度分为：

- **Chunk**：netty向操作系统申请内存是以Chunk为单位申请的，内存分配也是基于Chunk。Chunk是Page为单元的集合，默认16M；
- **Page**: 内存管理的最小单元，默认8K；
- **SubPage**: Page内的内存分配单元。

netty逻辑上将内存大小分为了tiny, small, normal, huge 几个单位。申请内存大于Chunk size 为huge，此时不在内存池中管理，由JVM负责处理；当Client申请的内存大于一个Page的大小（normal）, 在Chunk内进行分配; 对tiny&small大小的内存，在一个Page页内进行分配

<img src="./images/netty内存管理粒度.png" style="zoom:55%;" />

## 8.2.内存管理层级

netty内存池的层级结构，主要分为Arena、ChunkList、Chunk、Page、Subpage这5个层级，

- Arena代表1个内存区域，为了优化内存区域的并发访问，netty中内存池是由多个Arena组成的数组，分配时会每个线程按照轮询策略选择1个Arena进行内存分配；

- 1个Arena由两个**PoolSubpage**数组和多个**ChunkList**组成。

  - 两个PoolSubpage数组分别为**tinySubpagePools**和**smallSubpagePools**

  - 多个ChunkList按照双向链表排列，每个ChunkList里包含多个Chunk，每个Chunk里包含多个Page（默认2048个），每个Page（默认大小为8k字节）由多个Subpage组成

```java
abstract class PoolArena<T> implements PoolArenaMetric {
    
    private final PoolSubpage<T>[] tinySubpagePools;
    private final PoolSubpage<T>[] smallSubpagePools;

    // 存储内存利用率0-25%的chunk
    private final PoolChunkList<T> qInit;
    // 存储内存利用率1-50%的chunk
    private final PoolChunkList<T> q000;
    // 存储内存利用率25-75%的chunk
    private final PoolChunkList<T> q025;
    // 存储内存利用率50-100%的chunk
    private final PoolChunkList<T> q050;
    // 存储内存利用率75-100%的chunk
    private final PoolChunkList<T> q075;
    // 存储内存利用率100%的chunk
    private final PoolChunkList<T> q100;
}
```

## 8.3.内存分配流程

- PoolArena：内存分配中心
- PoolChunk：负责Chunk内的内存分配
- PoolSubpage：负责Page内的内存分配

![](./images/netty内存池的布局与管理.jpg)

# *.Netty实战

## *.1.ChannelHandler不要执行耗时操作

ChannelHandler的回调方法是由EventLoop唯一关联的线程Thread(即netty的I/O线程)调用的，因此若将长时间执行的耗时任务放到Handler回调方法中，它会阻塞该线程处理其它I/O事件，通常有两种实现方式：

1. ChannelHandler定义业务线程池，执行异步调用；

2. 借助于netty提供的想ChannelPipeline添加ChannelHandler时调用的addList()方法来传递EventExecutor。方法如下：

```java
/**
 * @param group 用来执行ChannelHandler中的方法
 * @param handlers  the handlers to insert last
 */
ChannelPipeline addLast(EventExecutorGroup group, ChannelHandler... handlers);
```

## *.2.两种写回数据的方式

在ChannelHander回调方法如channelRead0()，我们可以将数据写回到通道对端，可以直接由ChannelHandlerContext写回，也可以通过它获取到Channel写回，即：

```java
// 通过Channel写回数据
ctx.channel().writeAndFlush();
// 通过ChannelHandlerContext写回数据
ctx.writeAndFlush();
```

这两种方式是有区别的：

1. Channel：数据从ChannelPipeline的ChannelHandler链表末尾开始传递直至底层Socket;

2. ChannelHandlerContext：数据从ChannelPipeline的ChannelHandler链表中的下一个ChannelHandler开始写回直至底层Socket.

**结论：**

对于与Channel的同名方法来说，ChannelHandlerContext的方法将会产生更短的事件流，所以可以在可能的情况下利用它提升应用性能；

## *.3.TCP粘包/拆包

拆包：一个完整的数据包可能会被TCP拆分为多个包进行发送；

粘包：多个小的数据包可能被封装成一个大的包进行发送，有人称为半包；

### *.3.1.产生原因

在计算机科学中，bit是表示信息的最小单位，叫做二进制位，一般用0和1表示；而byte叫做字节，由8个位（8bit）组成。他们之间的区别如下：

1. bit
   - 计算机中的最小存储单元  - 存储内容总是0或1
   - 所有二进制状态实体都能使用1bit表示  - 8bits组成1byte
   - 不能够单独寻址

2. byte

   - 1byte包含8bits  
- 可以存储所有ASCII所有字符（这是它包  含8bits的初衷）  
   - 十进制整数范围[-128,127]或[0,  255]  
- 最小的可寻址存储单元

因此在Java中给一个byte类型数据初始化时，可以用字符，也可以用整数，但是这个整数必须要在-128和127之间（因为byte是8位），它会将这个数转化为一个字符然后存放起来！作为面向字节(byte)流的网络传输协议-TCP，所谓流，就是没有界限的一串数据。且由于TCP是全双工通信协议，因此TCP连接的两端，都会设置发送缓冲区和接收缓冲区，用来临时存放双向通信的数据。其根本原因是因为 TCP 是基于字节流来传输数据的，数据包相互之间没有边界，导致接收方无法准确的分辨出每一个单独的数据包。

**产生粘包的原因：**

- 发送方写入的数据，小于套接字缓冲区的大小；网卡会将程序多次写入的数据一起发送到网络上
- 接收方不及时读取套接字缓冲区数据，在下一次读取中就会获取到多个数据包

**产生拆包的原因：**

- 发送方写入的数据，大于套接字缓冲区大小；网卡会对其进行拆分，发送到网络上
- 发送的数据大于协议的MTU（最大传输单元），就一定会拆包

注意：UDP就像邮寄的包裹，虽然一次运输多个，但每个包裹都有分界线，对端会一个一个签收，所以不会有粘包和拆包的问题

### *.3.2.图解流程

1. 一个正常的TCP报文传输是这样子，client将消息Msg1，Msg2依次发给server，而且server也是按照这样子的顺序接收到两个消息：

![](./images/TCP粘包与拆包流程1.png)

2. 一旦发生上面介绍的产生原因，比如数据报小于TCP缓冲区容量，就会发生粘包，TCP会将Msg1和Msg2组装成一个消息发给服务端，这样服务端就只会收到一个消息（但实际要拆分成2个消息）

![](./images/TCP粘包与拆包流程2.png)

3. 还有一种情况那就是数据报太大，超过了TCP缓冲区大小，则它会将其拆包，可能Msg1全部+Msg2部分当作一个消息发送给服务端：

![](./images/TCP粘包与拆包流程3.png)

注：这上面所说的消息（即Msg1、Msg2）都是一串一串的字节流

### *.3.3.解决方案

这边记录下[网络博客](http://www.ideawu.net/blog/archives/993.html)介绍的一个伪代码实例：

```java
char recv_buf[];
Buffer buffer;
// 网络循环：必须在一个循环中读取网络，因为网络数据是源源不断的。
while(1){
    // 从TCP流中读取不定长度的一段流数据，不能保证读到的数据是你期望的长度
    tcp.read(recv_buf);
    // 将这段流数据和之前收到的流数据拼接到一起
    buffer.append(recv_buf);
    // 解析循环：必须在一个循环中解析报文，避免所谓的粘包
    while(1){
        // 尝试解析报文
        msg = parse(buffer);
        if(!msg){
            // 报文还没有准备好，糟糕，我们遇到拆包了！跳出解析循环，继续读网络。
            break;
        }
        // 将解析过的报文对应的流数据清除
        buffer.remove(msg.length);
        // 业务处理
        process(msg);
    }
}
```

它实际上就是想表达，网络传输的数据是不断地发送过来，而已这数据都是字节流，里面掺杂了由于粘包/拆包导致的其它消息数据，所以才需要两个循环：网络循环和解析循环。其实，解决TCP粘包/拆包问题的根本手段：**就是找出消息的边界**，常用的解决方案有两种：

| 方法                       | 寻找消息边界                               | 优点                 | 缺点                                           |
| -------------------------- | ------------------------------------------ | -------------------- | ---------------------------------------------- |
| 分隔符                     | 分隔符之间                                 | 不浪费空间，比较简单 | 内容本身出现分隔符时需要转义，所以需要扫描内容 |
| 固定长度字段保存内容的长度 | 先解析固定长度的字段的值，然后读取后续内容 | 精确定义数据长度     | 长度理论上有限制，需提前预知可能的最大长度     |

在netty中，它提供了3个解码器，可以有效地解决TCP粘包/拆包问题：

| **组件**                   | **结果**                                                     |
| -------------------------- | ------------------------------------------------------------ |
| LineBasedFrameDecoder      | 基于换行符解决，即传输完一个消息以  \r\n作为结尾；           |
| DelimiterBasedFrameDecoder | 基于分隔符解决，指定一个分隔符作为消息结尾                   |
| FixedLengthFrameDecoder    | 指定长度解决。指定消息头和消息体，消息头的值就是消息体的数据大小 |
| 无组件                     | 自定义协议，即规定消息如何划分个体，例如dubbo，自定义数据传输协议，然后client和server按照这个协议将字节流转换成消息。 |
