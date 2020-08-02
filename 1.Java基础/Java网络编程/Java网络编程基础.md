# 1.I/O模型

网络IO编程模型是一种无关语言和操作系统的基础知识点，网络IO在client-server服务端模型中，是这样一种请求模式：

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/网络IO编程模型.png)

内核空间是底层操作系统的运行环境，而用户空间是我们服务端程序的运行环境。从上图可以看出，数据无论从网卡到用户空间还是从用户空间到网卡都需要经过内核，需要通过系统调用（例如recvfrom/sendto）向内核读写数据，内核再进一步操作网卡

## 1.1.基础概念

网络I/O 模型大体可以分为：同步阻塞、同步非阻塞、异步阻塞、异步非阻塞4种。同步与非同步相对应，阻塞与非阻塞相对应，这两组间表达不同概念：

同步和异步指**消息的通知机制**，阻塞和非阻塞指**线程等待消息通知的状态**

1. 同步：程序执行IO请求，需要一直询问内核IO操作是否完成；

2. 异步：程序执行IO请求，内核在IO操作完成后，主动告知程序已经完成；

3. 阻塞：程序执行IO请求，请求还未完成，程序无法返回，一直等待；

4. 非阻塞：程序在IO请求，请求还未完成，但是程序立即返回，无需等待。

## 1.2.Unix网络编程模型

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/Unix网络编程模型概况.png)

unix有五种网络IO编程模型，分别是:**阻塞IO、非阻塞IO、IO复用、信号驱动、异步IO**，它们的执行流程总结起来就是两个阶段：

1. 等待内核把数据准备好；

2. 将数据从内核空间拷贝到用户空间

前4种IO编程模型(**阻塞IO、非阻塞IO、IO复用、信号驱动**)主要区别于第一阶段，因为它们的第二阶段都是一样的，即：在数据从kernel拷贝到process的缓冲区期间，都阻塞于 recvfrom 调用。相反，异步IO 模型在这两个阶段都不会阻塞，从而不同于其他四种模型

 很多时候比较容易混淆non-blocking IO(非阻塞IO)和asynchronous IO(异步IO)，认为都一样。而实际上，non-blocking IO只要求第一阶段(数据准备工作)不阻塞即可，而asynchronous IO要求第一、二阶段都不阻塞。不过可惜的是，linux系统目前并不支持异步IO，多是以IO 多路复用模式为主；但在window下，它通过了IOCP实现了真正的异步I/O！！

### 1.2.1.同步阻塞I/O

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/同步阻塞IO模型.png)

应用程序进行 recvfrom 系统调用时将阻塞在此调用，直到该套接字上有数据并且复制到用户空间缓冲区，调用线程在整个处理过程中，一直处于阻塞状态。Java传统的IO模型(ServerSocket+Socket)就是基于这种模式，因此它需要一个主线程不断循环接收客户端请求，另起线程池异步处理客户端请求.

### 1.2.2.同步非阻塞I/O

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/同步非阻塞IO模型.png)

应用程序需要把Socket设置为非阻塞模式，告诉内核，当所请求的 I/O 操作无法完成时，不要将线程阻塞，而是返回一个错误（不同的接口立即返回时的错误值不太一样，如recv、send、accept errno通常被设置为EAGIN 或者EWOULDBLOCK，connect 则为EINPRO- GRESS）应用程序基于 I/O 操作函数将不断的轮询数据是否已经准备好，如果没有准备好，继续轮询，直到数据准备好为止，然后当前线程阻塞住，等到操作系统将数据复制到用户空间成功后才会返回。

### 1.2.3.I/O多路复用

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/IO多路复用模型.png)

应用进程阻塞于 **select/poll/epoll** 等**操作系统函数**等待某个连接变成可读（即客户端发起的Socket连接），再调用 recvfrom 从连接上读取数据。虽然此模式也会阻塞在 select/poll/epoll 上，但与[阻塞IO 模型](#1.2.1.同步阻塞IO)不同的是它阻塞在等待多个连接上有读（写）事件的发生，以较少的代价来同时监听处理多个IO明显提高了效率且增加了单线程/单进程中并行处理多连接的可能。Java的选择器**Selector**就是基于这种模型。

### 1.2.4.信号驱动I/O

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/信号驱动IO模型.png)

应用进程创建 SIGIO 信号处理程序，此程序可处理连接上数据的读写和业务处理。并向操作系统安装此信号，**线程可以往下执行**。当内核数据准备好会向应用进程发送信号，触发信号处理程序的执行。再在信号处理程序中进行 recvfrom 和业务处理，这期间调用线程也会处于阻塞状态，直到数据处理成功！

### 1.2.5.异步非阻塞I/O

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/异步非阻塞IO模型.png)

应用程序通过 aio_read 告知内核发起IO请求，然后立即返回；然后内核自行准备数据，并把数据从内核空间拷贝到用户空间，最后内核**自动**通知应用程序。整个IO操作已经完成。[信号驱动 IO](#1.2.4.信号驱动IO) 是内核通知程序何时可以启动一个 IO 操作，而异步 IO 模型是由内核通知程序 IO 操作何时完成。前面四种IO模型都带有阻塞部分：有的阻塞在等待数据准备期间，有的阻塞在从内核空间拷贝数据到用户空间，而异步非阻塞IO是真正实现了两个阶段都是异步的场景

## 1.3.select/poll/epoll

虽然在5种网络I/O模型中，异步非阻塞I/O性能最优，但其实很少有 Linux 系统支持，反而是 Windows 系统提供了一个叫 IOCP 线程模型属于这一种。所以呢，大部分Linux系统其实都是属于`I/0多路复用`，select、poll、epoll都是I/O多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个文件描述符，一旦某个描述符就绪（读就绪或写就绪），能够通知程序进行相应的读写操作 。

### 1.3.1.cpu中断

一般而言，由硬件产生的信号需要 CPU 立马做出回应，不然数据可能就丢失了，所以它的优先级很高。CPU 理应中断掉正在执行的程序，去做出响应；当 CPU 完成对硬件的响应后，再重新执行用户程序。就以键盘为例，当用户按下键盘某个按键时，键盘会给 CPU 的`中断引脚`发出一个高电平，CPU能够捕获这个信号，然后执行键盘中断程序。

所以，在网络传输中，CPU怎么知道有数据到来，就是：网卡收到网线传过来的二进制数据后，它会把数据写入到内存中，然后网卡向 CPU 发出一个中断信号，操作系统便能得知有新数据到来，再通过网卡中断程序去处理数据

### 1.3.2.socket等待队列

现代操作为了实现多任务处理，会实现进程调度的功能，会把进程分为多种状态，其中只有`运行`状态的进程才会获取到CPU使用权。而操作系统会将多个进程放到它的工作队列中，操作系统会分时执行各个运行状态的进程，由于速度很快，看上去就像是同时执行多个任务。例如，下图的计算机中运行着 A、B 与 C 三个进程，其中进程 A 执行着上述基础网络程序，一开始，这 3 个进程都被操作系统的工作队列所引用，处于运行状态，会分时执行

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/操作系统工作队列.png)

当进程 A 执行到创建 socket 的代码时，操作系统会创建一个由文件系统管理的 socket 对象。这个 socket 对象包含了`发送缓冲区`、`接收缓冲区`与`等待队列`等成员。等待队列是个非常重要的结构，它指向所有需要等待该 socket 事件的进程。**当程序执行到 recv 时，操作系统会将进程 A 的引用添加到该 socket 的等待队列中**（如下图），同时将进程A置为`阻塞`状态。此时由于工作队列只剩下了进程 B 和 C，依据进程调度，CPU 会轮流执行这两个进程的程序，不会执行进程 A 的程序，就会不会往下执行代码，也不会占用 CPU 资源。

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/操作系统socket等待队列.png)

进程A阻塞在Socket的recv()期间，计算机收到了对端的数据流，数据会经由网卡传送到内存中，然后网卡通过中断信号通知 CPU 有数据到达，CPU 执行中断程序。此处的中断程序主要有两项功能，

- 将网络数据写入到对应 socket 的接收缓冲区里面
- 从socket等待列表中唤醒进程

每个socket都占用操作系统唯一的端口号，网络数据包肯定会指定IP地址和端口号，内核通过端口号找到对应的 socket，然后将该 socket 队列上的进程变成运行状态，继续执行代码。同时由于 socket 的接收缓冲区已经有了数据，recv 可以返回接收到的数据。

### 1.3.3.socket监听

操作系统内核会在一个socket有数据到来时，唤醒在其等待列表中的进程，那么问题来了，内核是怎么同时监控多个socket的数据？这其实就是select、poll、epoll要完成的事。

#### 1.3.3.1.select机制

先看下在linux系统`/usr/include/sys/select.h`文件中对`select`方法的定义：

```c
## 定义结构体 fd_set
typedef struct
{ 
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
} fd_set;
## select函数
extern int select (int __nfds, 
                   fd_set *__restrict __readfds,
                   fd_set *__restrict __writefds,
                   fd_set *__restrict __exceptfds,
                   struct timeval *__restrict __timeout);
```

- **int __nfds**是`fd_set`中最大的描述符+1，当调用select时，内核态会判断fd_set中描述符是否就绪，__nfds告诉内核最多判断到哪一个描述符；

- **__readfds、__writefds、__exceptfds**都是结构体`fd_set`，fd_set可以看作是一个描述符的集合。 select函数中存在三个fd_set集合，分别代表三种事件，`readfds`表示读描述符集合，`writefds`表示读描述符集合，`exceptfds`表示异常描述符集合。当对应的fd_set = NULL时，表示不监听该类描述符；
- **timeval __timeout**用来指定select的工作方式，即当文件描述符尚未就绪时，select是永远等下去，还是等待一定的时间，或者是直接返回；
- **返回值int**表示： 就绪描述符的数量，如果为-1表示产生错误 。

select 的实现思路很直接，假设进程A调用了select()同时监视sock1、sock2 和 sock3 三个 socket，那么在调用 select()函数 之后，操作系统把进程 A 分别加入这三个 socket 的等待队列中，同时将进程A阻塞。当任何一个 socket 收到数据后，中断程序将唤起进程A。

当进程 A 被唤醒后，select()函数会将全量`fd_set`从用户空间拷贝到内核空间，并注册回调函数， 在内核态空间来判断每个请求是否准备好数据 。如果有一个或者多个描述符就绪，那么select将就绪的文件描述符放置到指定位置，然后select返回。返回后，由程序遍历查看哪个请求有数据。

**select缺陷：**

- 每次调用select，都需要把fd集合从用户态拷贝到内核态，fd越多开销则越大；
- 每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大。因此，规定 select 的最大监视数量，默认只能监视 1024 个 socket。

#### 1.3.3.2.poll机制

先看下linux系统中`/usr/include/sys/poll.h`文件中对`poll`方法的定义:

```c
## pollfd结构体
struct pollfd
{
    int fd;                     /* File descriptor to poll.  */
    short int events;           /* Types of events poller cares about.  */
    short int revents;          /* Types of events that actually occurred.  */
};

## poll函数
extern int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```

- **_fds**参数时Poll机制中定义的结构体`pollfd`，用来指定一个需要监听的描述符。结构体中fd为需要监听的文件描述符，events为需要监听的事件类型，而revents为经过poll调用之后返回的事件类型，在调用poll的时候，一般会传入一个pollfd的结构体数组，数组的元素个数表示监控的描述符个数。
- **__nfds**和**__timeout**参数都和select机制中的同名参数含义类似

poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用`pollfd`结构代替select的`fd_set`结构，其他的本质上都差不多。所以**Poll机制突破了Select机制中的文件描述符数量最大为1024的限制**。但是poll也会有select的缺陷，一模一样的

#### 1.3.3.3.epoll机制

先看下linux系统中`/usr/include/sys/epoll.h`对epoll机制定义的方法：

```c
## 创建一个epoll实例并返回，该实例可以用于监控__size个文件描述符
extern int epoll_create (int __size) __THROW;

## 向epoll中注册事件，该函数如果调用成功返回0，否则返回-1。其中各个参数含义为：
## __epfd为epoll_create返回的epoll实例
## __op表示要进行的操作
## __fd为要进行监控的文件描述符
## __event要监控的事件
extern int epoll_ctl (int __epfd, int __op, int __fd,
                      struct epoll_event *__event) __THROW;

## 类似与select机制中的select函数、poll机制中的poll函数，等待内核返回监听描述符的事件产生。该函数## 返回已经就绪的事件的数量，如果为-1表示出错。其中各个参数含义：
## __epfd为epoll_create返回的epoll实例
## __events数组为 epoll_wait要返回的已经产生的事件集合
## __maxevents为希望返回的最大的事件数量（通常为__events的大小）
## __timeout和select、poll机制中的同名参数含义相同
extern int epoll_wait (int __epfd, struct epoll_event *__events,
                       int __maxevents, int __timeout);
```

当进程A调用 epoll_create 方法时，内核会创建一个 eventpoll 对象（也就是API中 epfd 所代表的对象）。eventpoll 对象也是文件系统中的一员，和 socket 一样，它也会有等待队列。而且它通过`mmap`内存映射的方式共享在用户态和内核态之间，所以可以减少了内核态和用户态fd文件的拷贝。

创建 epoll 对象后，可以用 epoll_ctl()方法 添加或删除所要监听的 socket。如果通过 epoll_ctl 添加 sock1、sock2 和 sock3 的监视，内核会将 eventpoll 添加到这三个 socket 的等待队列中，并设置中断程序（也可以成为回调函数）。当 socket 收到数据后，中断程序会操作 eventpoll 对象，而不会再直接操作进程。

socket 收到数据后，中断程序（回调函数）会给 eventpoll 的`就绪列表`rdlist 添加 socket 引用，当程序执行到 epoll_wait 时，如果 rdlist 已经引用了 socket，那么 epoll_wait 直接返回，如果 rdlist 为空，否则继续阻塞进程

**总而言之：**

- epoll 在 select 和 poll 的基础上引入了 eventpoll （即调用epoll_create()函数的返回值）作为中间层，eventpoll 对象相当于 socket 和进程之间的中介，socket 的数据接收并不直接影响进程，而是通过改变 eventpoll 的就绪链表rdlist 来改变进程状态。

**工作模式**：

Epoll内部还分为两种工作模式： **LT水平触发（level trigger）**和**ET边缘触发（edge trigger）**

- **LT模式：** 默认的工作模式，即当epoll_wait检测到某描述符事件就绪并通知应用程序时，应用程序**可以不立即处理**该事件；事件会被放回到就绪链表中，下次调用epoll_wait时，会再次通知此事件。
- **ET模式：** 当epoll_wait检测到某描述符事件就绪并通知应用程序时，应用程序**必须立即处理**该事件。如果不处理，下次调用epoll_wait时，不会再次响应并通知此事件。

epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个fd的阻塞I/O操作把多个处理其他文件描述符的任务饿死。ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。

# 2.计算机网络协议

## 2.1.TCP

### 2.1.1.TCP特性

TCP是一个面向连接的、端到端的、提供高可靠性服务的传输层协议。特性如下：

1. TCP提供一种面向连接的、可靠的字节流服务；

2. 一个TCP连接中，仅有两方进行全双工通信。广播和多播不能用于TCP；

3. TCP使用校验和，确认和重传机制来保证可靠传输；

4. TCP给数据分节进行排序，使用累积确认保证数据的顺序不变和非重复；

5. TCP使用滑动窗口机制实现流量控制，通过动态改变窗口大小进行拥塞控制

### 2.1.2.TCP报文

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/TCP报文.png)

其中不同的标志位，代表TCP报文的不同含义：

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/TCP报文之标志位.png" style="zoom:67%;" />

### 2.1.3.三次握手

建立一个TCP连接时，客户端和服务端总共需要进行三次交互，这个过程就称为“TCP三次握手”。三次握手的目的是连接服务器指定端口，建立TCP连接，并同步连接双方的序列号和确认号，交换TCP窗口大小信息。在socket编程中，客户端执行connect()时就会触发三次握手：

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/TCP三次握手.jpg)

**①第一次握手(SYN=1, seq=x)**

客户端发送TCP的SYN标志位置为1的包，指定客户端打算连接的服务器的端口，以及初始序号x，保存在TCP报文的序列号字段里。发送完毕后，客户端进入SYN_SEND状态。

**②第二次握手(SYN=1, ACK=1, seq=y, ACKnum=x+1)**

服务器发回确认包(ACK)应答，即SYN标志位和ACK标志位都为1。服务端选择自己的ISN序列号，放到序号域内，同时将确认序号设置为客户端的ISN加1，即x+1。发送完毕，服务端进入SYN_RCVD状态。

**③第三次握手(ACK=1, ACKnum=y+1)**

客户端再次发送确认包(ACK)，SYN标志位为0，ACK标志位为1，并且把服务端发送的序号字段+1，放在确认序号中发送给服务端，并且在数据段放入ISN+1。发送完毕后，客户端进入ESTABLISHED状态，当服务端接收到这个包后，也会进入ESTABLISHED状态，TCP握手结束，建立连接

### 2.1.4.四次握手

TCP 的连接的拆除需要发送四个包，因此称为四次挥手(Four-way handshake)，也叫做改进的三次握手。客户端或服务器均可主动发起挥手动作，在 socket 编程中，任何一方执行 close() 操作即可产生挥手操作。

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/TCP四次握手.jpg" style="zoom:80%;" />

**①第一次挥手(FIN=1，seq=x)**

假设客户端想要关闭连接，客户端发送一个 FIN 标志位置为1的包，表示自己已经没有数据可以发送了，但是仍然可以接受数据。发送完毕后，客户端进入 FIN_WAIT_1 状态。

**②第二次挥手(ACK=1，ACKnum=x+1)**

服务器端确认客户端的 FIN 包，发送一个确认包，表明自己接受到了客户端关闭连接的请求，但还没有准备好关闭连接。发送完毕后，服务器端进入 CLOSE_WAIT 状态，客户端接收到这个确认包之后，进入 FIN_WAIT_2 状态，等待服务器端关闭连接。

**③第三次挥手(FIN=1，seq=y)**

服务器端准备好关闭连接时，向客户端发送结束连接请求，FIN 置为1。发送完毕后，服务器端进入 LAST_ACK 状态，等待来自客户端的最后一个ACK。

**④第四次挥手(ACK=1**，**ACKnum=y+1**)

客户端接收到来自服务器端的关闭请求，发送一个确认包，并进入 TIME_WAIT状态，等待可能出现的要求重传的 ACK 包。服务器端接收到这个确认包之后，关闭连接，进入 CLOSED 状态。客户端等待了某个固定时间（两个最大段生命周期，2MSL，2 Maximum Segment Lifetime）之后，没有收到服务器端的 ACK ，认为服务器端已经正常关闭连接，于是自己也关闭连接，进入 CLOSED 状态。

## 2.2.UDP

UDP 是一个简单的传输层协议。和 TCP 相比，UDP 有下面几个显著特性：

1. UDP 缺乏可靠性。UDP 本身不提供确认，序列号，超时重传等机制。UDP 数据报可能在网络中被复制，被重新排序。即 UDP 不保证数据报会到达其最终目的地，也不保证各个数据报的先后顺序，也不保证每个数据报只到达一次；

2. UDP 数据报是有长度的。每个 UDP 数据报都有长度，如果一个数据报正确地到达目的地，那么该数据报的长度将随数据一起传递给接收方。而 TCP 是一个字节流协议，没有任何（协议上的）记录边界；

3. UDP 是无连接的。UDP 客户和服务器之前不必存在长期的关系。UDP 发送数据报之前也不需要经过握手创建连接的过程；

4. UDP 支持多播和广播

## 2.3.HTTP

# 3.零拷贝模式

零拷贝(Zero-Copy)是指计算机在执行操作时，**CPU不需要先将数据从某处内存复制到一个特定区域**，从而节省CPU时钟周期和内存带宽 -- 维基百科。例如下面这两个示意图：

①普通数据拷贝：

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/普通数据拷贝.jpg" style="zoom:80%;" />

②数据零拷贝:

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/数据零拷贝.jpg" style="zoom:80%;" />

## 3.1.内核空间/用户空间

在现代CPU的所有指令中，有些指令具有一定的危险性，所以CPU 将指令分为特权指令和非特权指令。以linux为例，这就衍生出两种程序运行空间：

1. user space，也称：用户空间

2. kernel space，也称：内核空间

内核空间可以执行**任意命令**，调用系统的一切资源(调用硬件、文件读写等)；用户空间只能执行简单的运算，**不能直接调用系统资源**，必须通过系统接口（又称 **system call**），才能向内核发出指令。Linux可以分为三个部分，从下往上依次为：硬件 -> 内核空间 -> 用户空间

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/linux空间划分.png" style="zoom:67%;" />

## 3.2.上下文切换

​	操作系统既然划分了用户空间和内核空间，因此当进程运行在内核空间时就处于内核态，而进程运行在用户空间时则处于用户态。当进程在内核态与用户态之间切换时，就被称为上下文切换。

​	所谓的进程上下文，就是一个进程在执行的时候，CPU的所有寄存器中的值、进程的状态以及堆栈中的内容，当内核需要切换到另一个进程时，它需要保存当前进程的所有状态，即保存当前进程的进程上下文，以便再次执行该进程时，能够恢复切换时的状态，继续执行

## 3.3.何为零拷贝？

首先明确一点，零拷贝依赖于操作系统，操作系统有提供就有，没提供就没有！下面会以时序图的方式解释零拷贝的演变过程，其中有几个概念：

1. User space，指用户空间，在这里可以理解成JVM上运行的程序；

2. Kernel space，指内核空间，在这里可以理解成Linux内核程序；

3. Hardware，指外部存储介质，不仅仅表示硬件，也表示网络请求的另一端；

4. DMA，指直接内存访问机制。

如果考虑到DMA拷贝过程，则会多算出2次拷贝，拷贝次数为：4 → 2 → 1；

如果不考虑DMA拷贝过程，只关心内核空间与用户空间的拷贝，则实际上拷贝次数为：2 → 1 → 0 。

### 3.3.1.四次拷贝(或两次拷贝)

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/4次拷贝(或2次拷贝)时序图.png" style="zoom:65%;" />

1. VM向OS发起系统调用read()，**上下文切换**，程序由用户态切换内核态；

2. 从外部存储介质发起DMA请求，将数据拷贝到内核缓冲区(第一次拷贝)；

3. **上下文切换**，程序由内核态切回用户态，数据拷贝到用户缓冲区(第二次拷贝)；

4. JVM向OS发起系统调用write()，**上下文切换**，程序由用户态切换内核态；

5. 数据拷贝到内核中与目的地Socket关联的缓冲区(第三次拷贝)；

6. 数据由内核缓冲区通过DMA拷贝到网卡NIC buffer(第四次拷贝)；

7. 系统调用write()完成返回，**上下文切换**，程序由内核态切回用户态。

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/4次拷贝(或2次拷贝)流程图.jpg" style="zoom:80%;" />

### 3.3.2.三次拷贝(或一次拷贝)

通过上面的分析可以看出，第2、3次拷贝（也就是从内核空间到用户空间的来回复制）是没有意义的，数据可以直接从内核缓冲区直接送入Socket缓冲区；零拷贝机制就实现了这一点，但是它需要由操作系统直接支持，不同OS有不同的实现方法。大多数Unix-like系统都是提供了sendfile()的系统调用。

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/3次拷贝(或1次拷贝)时序图.png" style="zoom:80%;" />

1. JVM向OS发起系统调用sendfile()，**上下文切换**，用户态切换成内核态；

2. 从外部介质发起DMA请求，将数据拷贝到内核缓冲区(第一次拷贝)；

3. 由内核直接将数据拷贝到与目的地socket关联的缓冲区(第二次拷贝)；

4. 数据由内核缓冲区通过DMA拷贝到网卡NIC buffer(第三次拷贝)；

5. 系统调用sendfile()完成，**上下文切换**，由内核态切回用户态。

![](C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/3次拷贝(或1次拷贝)流程图.png)

### 3.3.3.两次拷贝(或零次拷贝)

在上面的分析中，有一个从“Read buffer”到“Socket buffer”的拷贝，这是因为在一般的Block DMA方式中，源物理地址和目标物理地址都得是连续的，所以一次只能传输物理上连续的一块数据，每传输一个块发起一次中断，直到传输完成，所以必须要在两个缓冲区之间拷贝数据。

但是现在linux提供了Scatter/Gather DMA方式，会预先维护一个物理上不连续的**块描述符的链表**，描述符中包含有数据的起始地址和长度。传输时只需要遍历链表，按序传输数据，全部完成后发起一次中断即可。也就是说，Scatter/Gather DMA根据socket缓冲区中描述符提供的位置和偏移量信息直接将内核空间缓冲区中的数据拷贝到协议引擎上，不需要再从内核缓冲区向Socket缓冲区拷贝数据 —— **这就是零拷贝的最终版**

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/2次拷贝(或0次拷贝)时序图.png" style="zoom:75%;" />

1. JVM向OS发起系统调用sendfile()，**上下文切换**，由用户态切换成内核态；

2. 从外部介质中发起DMA请求，将数据拷贝到内核缓冲区上(第一次拷贝)；

3. 通过Scatter/Gather DMA将数据拷贝到网卡NIC buffer(第二次拷贝)；

4. 系统调用sendfile()完成，**上下文切换**，由内核态切回用户态。

<img src="C:/Users/Administrator/Desktop/笔记-md/1.Java基础/Java网络编程/images/2次拷贝(或0次拷贝)流程图.png" style="zoom:80%;" />