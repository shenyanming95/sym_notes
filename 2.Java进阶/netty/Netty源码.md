# 1.源码：服务启动

熟悉netty的同学都知道，netty的服务端启动代码样板几乎一样，几行简单的代码便可以创建一个基于NIO的非阻塞式网络服务端程序，但实际上netty底层做了大量的工作：

```java
// 定义两个线程组, 一个处理连接请求, 一个处理连接读写. 其实现类用NioEventLoopGroup
EventLoopGroup parentGroup = new NioEventLoopGroup();
EventLoopGroup childGroup = new NioEventLoopGroup();

// serverBootstrap是netty提供用来便捷创建一个服务端的工具类, 用来对服务端作自定义配置
ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap.group(parentGroup, childGroup)
    .channel(NioServerSocketChannel.class)
    .childHandler(new SimpleSocketServerInitializer());

// 绑定端口号, 服务端启动, 同时主线程挂起, 等待通道关闭的消息
ChannelFuture channelFuture = serverBootstrap.bind(9090).sync();
channelFuture..closeFuture().sync();
```

netty服务启动其实是在2个线程中执行的，一个是我们的主线程，一个是EventLoop线程，它们之间的执行顺序：

- **main thread**
  - 初始化EventLoopGroup
  - 创建elector
  - 创建sever socket channel
  - 初始化server socket channel
  - 为server socket channel 选择一个NioEventLoop
- **eventloop thread**
  - 将server socket channel 注册到 NioEventLoop上的selector上
  - 绑定地址启动
  - 注册接收连接事件（OP_ACCEPT）到selector上

## 1.1.【创建事件循环组】

在netty中，我们一般会创建2个事件循环组：

```java
// 定义两个线程组, 一个处理连接请求, 一个处理连接
EventLoopGroup parentGroup = new NioEventLoopGroup(1);
EventLoopGroup childGroup = new NioEventLoopGroup();
```

parentGroup也称bossGroup，作用于**Reactor模式**中的初始分发器，用来接收客户端的请求；childGroup也称workerGroup，主要是线程池用来处理parentGroup接收的客户端连接的读写请求。一般我们会使用基于nio的事件循环组 - io.netty.channel.nio.NioEventLoopGroup

### 1.1.1.NioEventLoopGroup()

调用NioEventLoopGroup的构造方法会一层一层调用它重载的构造方法，就以其无参构造方法开始：

1. 无参构造方法会调用下一个带有线程数的构造方法，并将线程数设置为0

```java
public NioEventLoopGroup() {
  this(0);
}
```

2. 此构造方法接收线程数(①中设置的0)，调用下一个构造方法，传入一个类型为java.util.concurrent.Executor的空对象

```java
public NioEventLoopGroup(int nThreads) {
  this(nThreads, (Executor) null);
}
```

3. 此构造方法接收线程数和Executor(默认0和null)，调用下一个构造方法，传入一个java.nio.channels.spi.SelectorProvider对象

```java
public NioEventLoopGroup(int nThreads, Executor executor) {
  this(nThreads, executor, SelectorProvider.provider());
}
```

4. 接收前三个重载构造方法的参数值，并调用下一个构造方法，传入默认的由netty定义的选择策略工厂实现类- DefaultSelectStrategyFactory

```java
public NioEventLoopGroup(int nThreads, Executor executor, 
                         final SelectorProvider selectorProvider) {
  this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
```

5. 接收前四个重载构造方法的参数值，然后调用父类的构造方法，额外传入一个 netty定义的类似JDK线程池拒绝策略-RejectedExecutionHandler

```java
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider 
           selectorProvider,final SelectStrategyFactory selectStrategyFactory) {
  super(nThreads, executor, selectorProvider, selectStrategyFactory, 
        RejectedExecutionHandlers.reject());
}
```

### 1.1.2.MultithreadEventLoopGroup()

通过[NioEventLoopGroup](#1.1.1.NioEventLoopGroup())的构造方法调用，最终会调用到它的抽象父类MultithreadEventLoopGroup身上。根据new NioEventLoopGroup()方法的分析，在只调用它的无参构造方法基础上，nThreads=0，Executor=null，可变参数args有3个对象，分别为：SelectorProvider、selectStrategyFactory和RejectedExecutionHandler。最后调用父类构造方法

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args){
  // 如果传入的nThreads为0, 则netty默认取DEFAULT_EVENT_LOOP_THREADS. 其值等于
  // 系统配置io.netty.eventLoopThreads 和 底层CPU核数*2 的最大值. 一般就等于你电脑
  // 的CPU核心数 * 2。
  super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

### 1.1.3.MultithreadEventExecutorGroup()

调用MultithreadEventLoopGroup构造方法，它会去调用它的抽象父类MultithreadEventExecutorGroup的构造方法，实例化逻辑为：

1. 这个构造方法会额外加上一个EventExcutorChooserFactory，它用来创建EventExecutorChooser对象，进而可以选择合适的EventExecutor去执行 I/O事件

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, 
                                        Object... args){
  // nThreads一般情况为CPU核心*2; executor为null; args一般为:
  // WindowsSelectorProvider(window环境下)、DefaultSelectStrategyFactory
  // 和RejectedExecutionHandlers. 然后它额外添加了一个事件执行选择器工厂实现
  // DefaultEventExecutorChooserFactory, 最后调用重载的构造方法。
  this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```

2. 将在这个构造方法里面，做全部的属性赋值

```java
/* MultithreadEventExecutorGroup的成员属性 */
private final EventExecutor[] children;
private final Set<EventExecutor> readonlyChildren;

// 用于表示已终止的EventExecutor数量
private final AtomicInteger terminatedChildren = new AtomicInteger();

// 用于终止 EventExecutor 的异步 Future
private final Promise<?> terminationFuture = 
  	new DefaultPromise(GlobalEventExecutor.INSTANCE);

private final EventExecutorChooserFactory.EventExecutorChooser chooser;

// 完整的构造方法
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
        EventExecutorChooserFactory chooserFactory, Object... args) {
  // 如果executor为空, 实例化io.netty.util.concurrent.ThreadPerTaskExecutor对象
  // 它会用到netty默认的线程工厂io.netty.util.concurrent.DefaultThreadFactory
  if (executor == null) {
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
  }
  // children即EventExecutor[]数组, 它的大小跟传入的线程数nThreads一样.
  // 然后为数组的每个对象创建一个新对象
  children = new EventExecutor[nThreads];
  for (int i = 0; i < nThreads; i ++) {
    // 标志此次创建的对象是否成功
    boolean success = false;
    try {
      // 调用newChild()方法创建EventExecutor对象, 点击跳转newChild()方法
      children[i] = newChild(executor, args);
      success = true;
    } catch (Exception e) {
      throw new IllegalStateException("failed to create a child event loop", e);
    } finally {
        // 只要有一个创建失败了, 执行下面的逻辑
        if (!success) {
            // 将前面创建好的EventExecutor优雅关闭掉
            for (int j = 0; j < i; j ++) {
                children[j].shutdownGracefully();
            }
            for (int j = 0; j < i; j ++) {
                EventExecutor e = children[j];
                try {
                    // EventExecutor.isTerminated()只有在它旗下所有任务都已经执行完
                    // 才会返回true. 所以如果EventExecutor有任务还未执行完, 就进入
                    // 下面的循环.
                    while (!e.isTerminated()) {
                        // 等待EventExecutor旗下的任务执行直到3个条件发生：
                        // 1.时间超时; 2.线程被中断; 3.任务全部执行完
                        e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                    }
                } catch (InterruptedException interrupted) {
                    // Let the caller handle the interruption.
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }
  }
    // 如果正常地创建了指定数目的EventExecutor, 则通过参数的EventExecutorChooserFactory
    // 创建一个EventExecutorChooserFactory.EventExecutorChooser. 这里默认让
    // DefaultEventExecutorChooserFactory来创建, 点击跳转newChooser()方法
    chooser = chooserFactory.newChooser(children);
    // 创建监听器, 用于 EventExecutor 终止时的监听
    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                // 当所有的EventExecutor都终止后, 通过成员变量terminationFuture
                // 通知所有的监听器
                terminationFuture.setSuccess(null);
            }
        }
    };
    // 将上面创建的监听器添加到每个EventExecutor上.
    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }
    // 将之前创建好的EventExecutor数组转换为一个不可变的Set集合, 赋值给readonlyChildren
    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

#### 1.1.3.1.newChild()

newChild()方法是用来创建EventExecutor对象，它是一个抽象方法，由子类实现。由于在启动netty服务端时使用的是NioEventLoopGroup，自然地这个方法就由NioEventLoopGroup来实现。逻辑为：

```java
// 源码：NioEventLoopGroup - 125行
protected EventLoop newChild(Executor executor, Object... args){
    // 实际上就是new一个NioEventLoop实例返回. args参数就是调用上面构造方法时传过来的参数,
    // 有3个值, 依次是：
    // 1.nio中的SelectorProvider - 根据操作系统不同采用不同的实现;
    // 2.netty的选择策略工厂 - DefaultSelectStrategyFactory实现;
    // 3.netty的拒绝执行处理器 - RejectedExecutionHandlers.REJECT实现.
    // 实际就是创建NioEventLoop对象, 调用它的构造方法, 跳转[创建事件执行器]源码分析
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
                            ((SelectStrategyFactory) args[1]).newSelectStrategy(), 
                            (RejectedExecutionHandler) args[2]);}
```

#### 1.1.3.2.newChooser()

默认的netty都会使用DefaultEventExecutorChooserFactory来创建一个EventExecutor选择器，逻辑如下：

```java
// 源码：DefaultEventExecutorChooserFactory - 34行
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        //若参数EventExecutor[]的大小是2的幂次方, 则使用PowerOfTwoEventExecutorChooser
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        //其它情况则创建GenericEventExecutorChooser
        return new GenericEventExecutorChooser(executors);
    }
}
```

GenericEventExecutorChooser和PowerOfTwoEventExecutorChooser的区别，主要是next()方法的区别，通过不同的选择逻辑，高效率地快速选择事件执行器：

```java
// 源码：GenericEventExecutorChooser
public EventExecutor next() {
  // 很简单, 就是通过将下标idx对EventExecutor[]数组求余, 然后将idx累加一
  return executors[Math.abs(idx.getAndIncrement() % executors.length)];
}
```

```java
// 源码：PowerOfTwoEventExecutorChooser
public EventExecutor next() {
  // 这个设计专门针对EventExecutor[]数组是2的幂次方的情况, 挺巧妙, 有点像hashMap对
  // capacity的设计一样, 具体逻辑为：
  // 1.首先明确一点, 减运算“-”比 与运算“&”更高级, 所以先执行减法. 2的幂次方有个特点
  //   就是其二进制最高位为1, 其它位为0, 若减一后, 则变为最高位为0, 其它位为1;
  // 2.然后将下标idx与 第一步得到的结果进行求余, 就可以保证依次顺序地获取每一个下标.
  //   这点很像hashMap计算capacity的算法, 力保每个桶都能被用到, 有效避免hash冲突.
  return executors[idx.getAndIncrement() & executors.length - 1];
}
```

## 1.2.【创建事件执行器】

在创建事件循环组的源码分析中可得，MultithreadEventExecutorGroup里面会调用newChild()方法依次创建EventExecutor实现。大部分情况我们是使用基于nio的NioEventLoopGroup事件循环组，所以它创建的事件执行器实现是：io.netty.channel.nio.NioEventLoop

### 1.2.1.NioEventLoop()

先分析下NioEventLoop的成员变量：

```java
public final class NioEventLoop extends SingleThreadEventLoop {
  // 待议..
  private static final int CLEANUP_INTERVAL = 256;
  
  // 表示是否禁用SelectionKey的优化, 默认值为false, 表示开启
  private static final boolean DISABLE_KEYSET_OPTIMIZATION =
    SystemPropertyUtil.getBoolean("io.netty.noKeySetOptimization", false);
  
  // 少于该值, 不开启空轮询重建新的 Selector 对象的功能
  private static final int MIN_PREMATURE_SELECTOR_RETURNS = 3;
  
  // NIO Selector 空轮询此值的次数后, 重建新的 Selector 对象, 默认值为512
  private static final int SELECTOR_AUTO_REBUILD_THRESHOLD;
  
  // 包装过的Selector, 被netty优化过了
  private Selector selector;
  
  // 原生的Selector对象
  private Selector unwrappedSelector;
  
  // 一个Set实现, 用来存放SelectionKey集合. netty自己实现，经过优化
  private SelectedSelectionKeySet selectedKeys;
  
  // JDK原生接口, 用来创建一个Selector对象
  private final SelectorProvider provider;
  
  // 唤醒标记, 因为原生的Selector.wakeup()唤醒方法开销比较大, 通过该标识, 减少调用
  private final AtomicBoolean wakenUp = new AtomicBoolean();
  
  // 选择策略, 有两个值：SelectStrategy.SELECT(阻塞式选择) 和 
  // SelectStrategy.CONTINUE(非阻塞式重试性选择)
  private final SelectStrategy selectStrategy;
  
  // 处理 Channel 的就绪的 IO 事件, 占处理任务的总时间的比例
  private volatile int ioRatio = 50;
  
  // 被取消的选择键SelectionKey的数量
  private int cancelledKeys;
  
  // 是否需要再次发起Selector的select()操作
  private boolean needsToSelectAgain;
}
```

NioEventLoop的构造方法如下：

```java
// 源码：NioEventLoop - 139行
NioEventLoop(NioEventLoopGroup parent, Executor executor, 
             SelectorProvider selectorProvider, SelectStrategy strategy, 
             RejectedExecutionHandler rejectedExecutionHandler) {
  // 调用抽象父类SingleThreadEventLoop的构造方法, 跳转父类构造方法
  super(parent, executor, false,DEFAULT_MAX_PENDING_TASKS,rejectedExecutionHandler);
  // 空指针判断..
  if (selectorProvider == null) {
    throw new NullPointerException("selectorProvider");
  }
  if (strategy == null) {
    throw new NullPointerException("selectStrategy");
  }
  // 将方法参数的selectorProvider赋值给成员变量provider, 此对象根据操作系统不同而不同.
  provider = selectorProvider;
  // 调用openSelector()获取SelectorTuple对象, 然后将SelectorTuple的selector对象赋给
  // 成员变量selector(经过netty优化过的nio选择器Selector); 将selectorTuple的
  // unwrappedSelector赋给成员变量unwrappedSelector(原生的nio选择器Selector).
  final SelectorTuple selectorTuple = openSelector();
  selector = selectorTuple.selector;
  unwrappedSelector = selectorTuple.unwrappedSelector;
  // 保存选择策略, 默认实现为：DefaultSelectStrategyFactory
  selectStrategy = strategy;
}
```

#### 1.2.1.1.openSelector()

通过openSelector()方法来创建SelectorTuple对象，SelectorTuple类很简单，只有两个变量，都是java.nio.channels.Selector类型，其中一个为unwrappedSelector，另一个为selector，依次保存nio原生选择器和netty优化后的选择器。创建逻辑为：

```java
// 源码: NioEventLoop - 170行
private SelectorTuple openSelector() {
  // 定义一个Selector, 用来保存java原生nio的Selector对象
  final Selector unwrappedSelector;
  try {
    // 直接调用java.nio.channels.spi.SelectorProvider获取一个选择器
    unwrappedSelector = provider.openSelector();
  } catch (IOException e) {
    throw new ChannelException("failed to open a new selector", e);
  }
  // DISABLE_KEYSET_OPTIMIZATION是NioEventLoop的成员变量, 表示是否要禁用Selector的
  // 优化, 一般为false. 若为true的话, 则SelectorTuple的两个选择器引用都是nio原生的
  // 选择器Selector
  if (DISABLE_KEYSET_OPTIMIZATION) {
    return new SelectorTuple(unwrappedSelector);
  }
  // SelectedSelectionKeySet是netty自定义的一个Set实现, 用来保存nio中的选择键
  // SelectionKey, 其内部用一个数组来存放.
  final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();
  // 这边netty做了java的安全管理处理, 代码我做了简化, 它实际效果如下. 其实就是获取
  // sun.nio.ch.SelectorImpl的Class类型
  Object maybeSelectorImplClass = Class.forName("sun.nio.ch.SelectorImpl", false,
                                                PlatformDependent.getSystemClassLoader());
  // 如果无法获取sun.nio.ch.SelectorImpl的Class类型, 那么就不会优化Selector, 直接
  // 用原生的Selector包装SelectorTuple, 再将其返回
  if (!(maybeSelectorImplClass instanceof Class) ||
      !((Class<?>) maybeSelectorImplClass).isAssignableFrom(
        unwrappedSelector.getClass())) {
    if (maybeSelectorImplClass instanceof Throwable) {
      Throwable t = (Throwable) maybeSelectorImplClass;
    }
    return new SelectorTuple(unwrappedSelector);
  }
  // 代码执行到这里, 即可以获取sun.nio.ch.SelectorImpl的Class类型, 将其强转
  final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
  // 通过反射获取Selector的selectedKeys和publicSelectedKeys的属性Field对象,
  // 将netty 定义的SelectedSelectionKeySet赋给上面创建出来的unwrappedSelector
  Object maybeException = AccessController.doPrivileged(
    new PrivilegedAction<Object>() {
      @Override
      public Object run() {
        try {
          // 获取"selectedKeys"和"publicSelectedKeys" 的 Field对象
          Field selectedKeysField = 
            selectorImplClass.getDeclaredField("selectedKeys");
          Field publicSelectedKeysField = 
            selectorImplClass.getDeclaredField("publicSelectedKeys");
          // 设置Field可访问, 若报错, 直接返回错误对象
          Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField);
          if (cause != null) return cause;
          cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField);
          if (cause != null) return cause;
          // 设置 SelectedSelectionKeySet 对象到 unwrappedSelector 的 Field 中
          selectedKeysField.set(unwrappedSelector, selectedKeySet);
          publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
          return null;
        } catch (NoSuchFieldException e) {
          return e;
        } catch (IllegalAccessException e) {
          return e;
        }
      }
    });
  // 如果通过Field设值失败, 则返回未被优化的Selector
  if (maybeException instanceof Exception) {
    selectedKeys = null;
    Exception e = (Exception) maybeException;
    return new SelectorTuple(unwrappedSelector);
  }
  // 代码走到这里, 则说明已经赋值成功, 可以优化. 则netty会创建一个
  // SelectedSelectionKeySetSelector作为优化后的Selector. 它与原生的Selector组合成
  // SelectorTuple返回. 其实, netty优化后的SelectedSelectionKeySetSelector, 就是在
  // 每次选择完以后, 将原先Selector中的selectedKeys集合给清空掉(回忆下nio原生的
  // selector每次调用select()获取选择键后, 都要手动地删掉这个选择键)
  selectedKeys = selectedKeySet;
  return new SelectorTuple(unwrappedSelector,
                           new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
}
```

### 1.2.2.SingleThreadEventLoop()

NioEventLoop继承自SingleThreadEventLoop，先看下它的成员变量：

```java
public abstract class SingleThreadEventLoop 
	extends SingleThreadEventExecutor implements EventLoop {
  // 任务队列的最大容量
  protected static final int DEFAULT_MAX_PENDING_TASKS = Math.max(16, 
       SystemPropertyUtil.getInt("io.netty.eventLoop.maxPendingTasks", 
         Integer.MAX_VALUE));
  // 任务队列
  private final Queue<Runnable> tailTasks;
}
```

SingleThreadEventLoop的构造方法如下：

```java
// 源码：SingleThreadEventLoop - 55
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                boolean addTaskWakesUp, int maxPendingTasks,
                                RejectedExecutionHandler rejectedExecutionHandler) {
  // 此构造方法是在创建NioEventLoop中被调用的, 这几个方法参数含义如下：
  // 1.parent即NioEventLoopGroup实例, 因为是在NioEventLoopGroup调用newChild()方法创
  //   建EventExecutor实例;
  // 2.executor即在MultithreadEventExecutorGroup中创建的ThreadPerTaskExecutor实例;
  // 3.addTaskWakeUp为false, 是在NioEventLoop构造方法中直接指定的;
  // 4.maxPendingTasks即Integer.MAX_VALUE, 是SingleThreadEventLoop成员变量定义的;
  // 5. rejectedExecutionHandler即RejectedExecutionHandlers的匿名实现类, 默认跑异常
  // 然后调用父类SingleThreadEventExecutor的构造方法
  super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler)
    // 调用newTaskQueue()方法, 创建LinkedBlockingQueue队列赋给成员变量tailTasks
    tailTasks = newTaskQueue(maxPendingTasks);
}
```

### 1.2.3.SingleThreadEventExecutor()

SingleThreadEventExecutor继承了AbstractScheduledEventExecutor，实现了OrderedEventExecutor接口，基于单线程的 EventExecutor 抽象类，即一个 EventExecutor 对应一个线程，其成员变量为：

````java
public abstract class SingleThreadEventExecutor 
  extends AbstractScheduledEventExecutor implements OrderedEventExecutor {
  
  static final int DEFAULT_MAX_PENDING_EXECUTOR_TASKS = Math.max(16,
       SystemPropertyUtil.getInt("io.netty.eventexecutor.maxPendingTasks",
                                 Integer.MAX_VALUE));                                                                                 
  private static final int ST_NOT_STARTED = 1; //未开始
  private static final int ST_STARTED = 2; //已开始
  private static final int ST_SHUTTING_DOWN = 3; //正在关闭中
  private static final int ST_SHUTDOWN = 4; //已关闭
  private static final int ST_TERMINATED = 5; //已终止

  // 字段的原子更新器, jdk并发包提供. 作用于本类的“state”字段
  private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor> 
    STATE_UPDATER = AtomicIntegerFieldUpdater.newUpdater
    (SingleThreadEventExecutor.class, "state");
  
  // 字段的原子更新器, 作用于本类的“threadProperties”字段
  private static final AtomicReferenceFieldUpdater<SingleThreadEventExecutor, 
  ThreadProperties> PROPERTIES_UPDATER = AtomicReferenceFieldUpdater.newUpdater
    (SingleThreadEventExecutor.class, ThreadProperties.class, "threadProperties");
  
  // 任务队列, 通过Executor.execute()方法提交的任务就会添加到此
  private final Queue<Runnable> taskQueue;
  
  // EventExecutor内部自己维护的线程Thread, 靠它去执行任务队列taskQueue中的任务, 单线程
  private volatile Thread thread;
  
  // io.netty.util.concurrent.ThreadProperties, netty封装的对线程属性的配置
  private volatile ThreadProperties threadProperties;
  
  // java.util.concurrent.Executor, 通过它创建 thread 线程
  private final Executor executor;
  
  // 标志线程是否被中断了
  private volatile boolean interrupted;
  
  private final Semaphore threadLock = new Semaphore(0);
  private final Set<Runnable> shutdownHooks = new LinkedHashSet<Runnable>();
  
  // 添加任务到 taskQueue 队列时, 是否唤醒 thread 线程
  private final boolean addTaskWakesUp;
  
  // 最大等待执行的任务数量, 即taskQueue的大小
  private final int maxPendingTasks;
  
  // 任务拒绝策略
  private final RejectedExecutionHandler rejectedExecutionHandler;
  
  // 最后执行时间
  private long lastExecutionTime;
  
  // 线程状态, SingleThreadEventExecutor 在实现上，thread 的初始化采用延迟启动的方式，只
  // 有在第一个任务时，executor 才会执行并创建该线程. 线程状态即该类开头那几个
  private volatile int state = ST_NOT_STARTED;
  
  // 优雅关闭的时间配置
  private volatile long gracefulShutdownQuietPeriod;
  private volatile long gracefulShutdownTimeout;
  private long gracefulShutdownStartTime;
  private final Promise<?> terminationFuture = 
    new DefaultPromise<Void>(GlobalEventExecutor.INSTANCE);
}
````

其被[SingleThreadEventLoop()](#1.2.2.SingleThreadEventLoop())调用的构造方法为：

```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, int maxPendingTasks,
                                    RejectedExecutionHandler rejectedHandler) {
  // 一层一层调用到顶级父类AbstractEventExecutor, 赋给EventExecutorGroup属性..
  super(parent);
  
  // 它的构造方法就很简单, 都是属性赋值而已
  this.addTaskWakesUp = addTaskWakesUp;
  this.maxPendingTasks = Math.max(16, maxPendingTasks);
  
  // 这个executor比较重要, 是java.util.concurrent.Executor类型, 可以执行一个任务
  this.executor = ObjectUtil.checkNotNull(executor, "executor");
  
  // newTaskQueue()会创建LinkedBlockingQueue实例
  taskQueue = newTaskQueue(this.maxPendingTasks);
  rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, 
                                                     "rejectedHandler");
}
```

## 1.3.【准备绑定端口】

当创建好EventLoopGroup后，借助于ServerBootstrap工具类，通过它的bind()方法可以绑定通道到一个端口，最终通过调用doBind()方法完成服务端所有创建、初始化和注册工作，将服务端启动。大体流程为：[创建通道](#1.4.1.创建通道)→[初始化通道](#1.4.2.初始化通道_2)→[注册通道](#1.4.3.注册通道)→[绑定端口](#1.4.4.doBind0())

### 1.3.1.bind()

调用ServerBootstrap#bind()指定一个端口开启绑定操作

```java
// 源码：AbstractBootstrap
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}
```

它会帮我们创建创建InetSocketAddress对象

```java
// 源码：AbstractBootstrap
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}
```

### 1.3.2.doBind()

上面两步执行后，就会交由doBind()来执行绑定，这个内部都是异步的，所以执行完就会返回到主线程main

```java
// 源码：AbstractBootstrap – 282行
private ChannelFuture doBind(final SocketAddress localAddress) {
    // initAndRegister()做了3件事：创建、初始化和注册通道.
    final ChannelFuture regFuture = initAndRegister();

    // 获取上一步创建好的通道实例, NioServerSocketChannel
    final Channel channel = regFuture.channel();

    // 若异步创建和初始化通道发生异常了, 这个cause()方法就会返回异常, 这边就直接返回Future
    if (regFuture.cause() != null) {
        return regFuture;
    }

    // 若Future.isDone()返回true, 说明通道创建和初始化已经成功完成
    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        // 调用doBind0()绑定, 带上上面创建好的DefaultChannelPromise
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // 因为netty很多都是异步的, 有可能通道三部曲(创建,初始化,注册)还未完成. 所以就需要
        // 往Future上添加监听器, 在执行完毕后回调.
        final PendingRegistrationPromise promise = 
            		new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // future.cause()不为空, 说明出现异常, 将promise置为失败.
                    promise.setFailure(cause);
                } else {
                    // 如果注册一切顺利, 设置AbstractBootstrap的registered变量为true;
                    // 同样也是调用doBind0()执行绑定.
                    promise.registered();
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

## 1.4.【准备通道】

上面doBind()方法首先调用的就是iniAndRegister()方法，它主要完成了3件事：创建通道，初始化通道和注册通道

```java
// 源码：AbstractBootstrap - 318行
final ChannelFuture initAndRegister() {
    // io.netty.channel.Channel对象
    Channel channel = null;
    try {
        // 通过ChannelFactory创建一个通道, 默认使用反射的方式ReflectiveChannelFactory.
        // ReflectiveChannelFactory需要指定一个Class对象, 它通过反射调用它的无参构造方法
        // 创建一个Channel实例, 大部分情况下这个Class对象为：NioServerSocketChannel.
        // 这里就会调用NioServerSocketChannel的无参构造方法创建它的实例, 即创建通道
        channel = channelFactory.newChannel();
        // 上一步完成, 可以获取到通道实例, 紧接着调用init()方法初始化通道
        init(channel);
    } catch (Throwable t) {
        // 初始化通道失败, 若创建好了将其关闭, 然后返回一个失败的DefaultChannelPromise
        if (channel != null) {
            channel.unsafe().closeForcibly();
        }
        return new DefaultChannelPromise(channel, 
                     GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    // 若channel初始化成功, 执行注册通道操作
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        // 注册失败, 关闭通道.
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

### 1.4.1.创建通道

大部分情况下，我们都会使用NioServerSocketChannel作为交互的通道，所以在使用通道工厂ChannelFactory创建一个通道时，默认都会调用该通道的无参构造方法(netty默认使用ReflectiveChannelFactory，反射的方式)

#### 1.4.1.1.NioServerSocketChannel()

1. 此构造方法就是调用newSocket()方法通过provider.openServerSocketChannel()去创建java.nio.channels.ServerSocketChannel通道实例：

````java
public NioServerSocketChannel() {
    // DEFAULT_SELECTOR_PROVIDER是NioServerSocketChannel的静态成员变量, 它是一个
    // java.nio.channels.spi.SelectorProvider类型,通过它获取通道实例后,调用重载构造方法
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
````

2. 此构造方法接收上一个构造方法创建好的ServerSocketChannel实例，调用 父类的构造方法，并且创建一个NioServerSocketChannelConfig配置对象：

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    // 调用父类AbstractNioMessageChannel的构造方法
    super(null, channel, SelectionKey.OP_ACCEPT);
    // javaChannel()返回的是上面newSocket()方法创建的nio的ServerSocketChannel;
    // javaChannel().socket()就是获取java.net.ServerSocket对象, 由此创建通道配置类
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

#### 1.4.1.2.AbstractNioMessageChannel()

```java
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, 
                                    int readInterestOp) {
    // 三个方法参数含义依次是：
    // 1.parent为null
    // 2.ch为ServerSocketChannel实例
    // 3.readInterestOp为SelectionKey.OP_ACCEPT, 值为16, 表示对接收连接事情感兴趣
    // 然后调用父类AbstractNioChannel的构造方法,
    super(parent, ch, readInterestOp);
}
```

#### 1.4.1.3.AbstractNioChannel()

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp)
{
    // 调用父类AbstractChannel的构造方法, 因为是创建nio的Channel, 所以parent为null
    super(parent);
    // ch即nio的通道, readInterestOp即I/O事情感兴趣的事件, 其值即OP_ACCEPT
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        // 将nio的通道配置成非阻塞式
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            // 配置出现异常, 关掉通道
            ch.close();
        } catch (IOException e2) {
            // 这边会处理日志...省略掉了
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

#### 1.4.1.4.AbstractChannel()

```java
protected AbstractChannel(Channel parent) {
    // Channel是io.netty.channel.Channel类型, 当创建服务端通道时此值为null
    this.parent = parent;
    // 创建通道的全局Id
    id = newId();
    // 创建Unsafe类, 源码：newUnsafe()
    unsafe = newUnsafe();
    // 创建DefaultChannelPipeline实例, 源码在下面
    pipeline = newChannelPipeline();
}
```

##### 1.4.1.4.1.创建NioMessageUnsafe

newUnsafe()在AbstractChannel中是抽象方法，交由子类实现：

```java
// 源码：AbstractNioMessageChannel - 46行
protected AbstractNioUnsafe newUnsafe() {
    // 它是创建一个NioMessageUnsafe实例. Unsafe类的操作不允许被用户代码使用。这些函数是真
    // 正用于数据传输操作，必须被IO线程调用. 实际上, Channel 真正的具体操作, 通过调用对应
    // 的 Unsafe 实现. 这边创建AbstractNioMessageChannel的内部类NioMessageUnsafe
    return new NioMessageUnsafe();
}
```

##### 1.4.1.4.2.创建DefaultChannelPipeline

newChannelPipeline()就直接返回一个DefaultChannelPipeline实例

```java
// 源码：AbstractChannel - 117行
protected DefaultChannelPipeline newChannelPipeline() {
    // 直接创建DefaultChannelPipeline实例, 这实际反映了：一个Channel创建完毕, 与之相对应
    // 的ChannelPipeline就会创建好. 管道实际就是一个双向链表
    return new DefaultChannelPipeline(this);
}
```

简单分析下DefaultChannelPipeline成员变量：

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    // DefaultChannelPipeline维护着一个AbstractChannelHandlerContext链表, 下面这两个
    // 变量就是链表头节点和尾节点的名称(根据类名生成)
    private static final String HEAD_NAME = generateName0(HeadContext.class);
    private static final String TAIL_NAME = generateName0(TailContext.class);

    // 缓存AbstractChannelHandlerContext链表的每个节点的名称, 保证唯一性
    private static final FastThreadLocal<Map<Class<?>, String>> nameCaches ...;

    //成员变量estimatorHandle(MessageSizeEstimator.Handle类型)的原子更新器
    private static final AtomicReferenceFieldUpdater ESTIMATOR = 
        AtomicReferenceFieldUpdater.newUpdater(DefaultChannelPipeline.class, 
                       MessageSizeEstimator.Handle.class, "estimatorHandle");

    // AbstractChannelHandlerContext链表的头尾节点, 分别是：
    // HeadContext(入栈、出栈类型处理器)和TailContext(入栈类型处理器)
    // 由这两个实例去关联真正的上下文对象节点
    final AbstractChannelHandlerContext head;
    final AbstractChannelHandlerContext tail;

    // 每个管道都与一个通道关联
    private final Channel channel;

    // 成功的 Promise 对象
    private final ChannelFuture succeededFuture;

    // 用于一些方法执行, 需要传入 Promise 类型的方法参数, 但是不需要进行通知, 就传入该值
    private final VoidChannelPromise voidPromise;

    // 待定
    private final boolean touch = ResourceLeakDetector.isEnabled();

    // 一般情况下, ChannelHandler方法的调用都是靠Channel注册EventLoop的线程去执行, 
    // 但也可以手动指定一个EventLoopGroup专门执行此ChannelHandler. 这个变量就是专门保存
    // 这些子处理器
    private Map<EventExecutorGroup, EventExecutor> childExecutors;

    // 待定
    private volatile MessageSizeEstimator.Handle estimatorHandle;

    // 是否首次注册
    private boolean firstRegistration = true;

    // 因为 ChannelHandler 添加到 pipeline 中, 会回调ChannelHandler.handlerAdded(), 并且
    // 该事件需要在 Channel 所属的 EventLoop 中执行. 但是 若Channel 并未注册在 EventLoop 
    // 上, 就需要暂时将“触发 ChannelHandler 的添加完成( added )事件”的逻辑，作为一个
    // PendingHandlerCallback缓存起来, 在 Channel 注册到 EventLoop 上时，再回调执行.
    private PendingHandlerCallback pendingHandlerCallbackHead;

    // 指明与此管道关联的通道Channel是否已经注册了
    private boolean registered;
}
```

管道事件的触发：

```java
// ChannelPipeline实际是通过AbstractChannelHandlerContext来维护请求响应链,所以事件的触发,最终
// 都是交由AbstractChannelHandlerContext来执行, 就以 ChannelActive 这个事件为例:

// 源码：DefaultChannelPipeline
public final ChannelPipeline fireChannelActive() {
    // 实际调用AbstractChannelHandlerContext的静态方法invokeChannelActive()
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}

// 源码：AbstractChannelHandlerContext
static void invokeChannelActive(final AbstractChannelHandlerContext next) {
    // 首先判断下是不是在 EventExecutor 本线程内,然后最终都是执行next.invokeChannelActive()
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelActive();
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelActive();
            }
        });
    }
}

// 源码：AbstractChannelHandlerContext
private void invokeChannelActive() {
    if (invokeHandler()) {
        try {
            // 要注意,每次管道中传递事件,第一个接收到的永远都是:
            // DefaultChannelPipeline$HeadContext对象,因此下面的handler()方法的对象
            // 实际上就是DefaultChannelPipeline$HeadContext它本身
            ((ChannelInboundHandler) handler()).channelActive(this);
        } catch (Throwable t) {
            invokeExceptionCaught(t);
        }
    } else {
        // 如果当前执行的状态不对, 就直接出发下一个AbstractChannelHandlerContext执行
        fireChannelActive();
    }
}

// 源码：DefaultChannelPipeline$HeadContext
public void channelActive(ChannelHandlerContext ctx) {
    // 就跟上面方法的else语句块一样,触发调用链: 它会通过维护的上下文链表,每次找出一个Context进行
    // 回调, 具体逻辑可以看后面的I/O事件处理
    ctx.fireChannelActive();
    readIfIsAutoRead();
}
```

### 1.4.2.初始化通道

当通过ChannelFactory[创建](#1.4.1.创建通道)完通道后，紧接着就调用init()方法开始初始化通道，此方法交由AbstractBootstrap的子类去实现，这边查看ServerBootstrap的源码实现

#### 1.4.2.1.init()

```java
// 源码：ServerBootstrap - 141行
void init(Channel channel) throws Exception {
    // options0()方法返回保存在AbstractBootstrap的options变量, 专门用来给通道设置选项.
    // 在之前创建通道时, 调用NioServerSocketChannel()构造方法会为通道创建一个配置类.实际
    // 是在DefaultServerSocketChannelConfig 和 DefaultChannelConfig 分别为
    // java.net.ServerSocket 和 io.netty.channel.Channel 设置属性...
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        // 这边的源码新版本可能被换掉了,但是这边有一个点可以说明,就是加锁的时候减少锁的粒度...
        setChannelOptions(channel, options, logger);
    }

    // attrs0()方法返回保存在AbstractBootstrap的attrs变量, 专门给通道设置自定义的属性.
    // 它与ChannelOption的区别是, option是给Channel(和其底层ServerSocket)做配置的,
    // 而AttributeKey主要可以设置用户自定义的一些属性值, 可以在全局使用.
    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    // 获取与当前Channel绑定的管道对象ChannelPipeline
    ChannelPipeline p = channel.pipeline();

    // 记录当前的属性
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().
            toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().
            toArray(newAttrArray(childAttrs.size()));
    }

    // 为管道添加一个通道处理器, 是匿名的ChannelInitializer实现, 它会在管道第一次调用时
    // 起作用, 而后会自己从管道的处理器链中自我移除. 这些代码需要在服务端启动后第一次接收
    // 客户端请求才会执行.
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            // 获取使用ServerBootstrap.handler()为bossGroup服务的处理器, 如果它不为空
            // 将其添加到管道中. ChannelPipeline的addLast()方法还是值得研究一波的.
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            // 通过Channel获取与其对应的事件循环组(一个Channel只有一个EventLoop), 向其
            // 提交一个任务：主要为ChannelPipeline添加一个ChannelHandler具体实现类 - 
            // ServerBootstrapAcceptor
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    // 往管道中加入ServerBootstrapAcceptor
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, 
                        currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

其中的ServerBootstrapAcceptor是ServerBootstrap的内部类，它也是一个ChannelInboundHandler，所以它也会有channelRead0()方法，源码为：

```java
// 源码：ServerBootstrap$ServerBootstrapAcceptor - 243行
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 会接收ServerSocketChannel获取到的SocketChannel，也就是参数msg
    final Channel child = (Channel) msg;
    // 加入我们在ServerBootstrap设置的childHandler
    child.pipeline().addLast(childHandler);
    // 设置一些属性和参数值
    setChannelOptions(child, childOptions, logger);
    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
    try {
        // 注意：然后会将当前获取到的SocketChannel注册到childGroup中，这个childGroup
        // 就是我们在ServerBootstrap初始化的时候创建的：
        // EventLoopGroup workerGroup = new NioEventLoopGroup();
        // 即证明了Netty确实是使用了主从Reactor模式，把Channel的监听交给BossGroup就是Main 
        // Reactor，把Channel的读写交给WorkerGroup就是Sub Reactor
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

#### 1.4.2.2.addList()

ChannelPipeline可以添加多个ChannelHandler，形成一个处理器链。实际上每个添加进来的ChannelHandler，Channelpipeline都会为其生成一个ChannelHandlerContext，而它内部实际维护的是ChannelHandlerContext链表。addList()有多个重载方法，最全的有3个参数，以默认实现类DefaultChannelPipeline为例，源码为：

```java
/**
  * @param group 使用指定事件循环组来执行handler的回调方法(原先由底层I/O线程调用)
  * @param name  手动指定handler名称, 若不指定则netty根据类名生成
  * @param handler 待添加的处理器handler
  */
// 源码：DefaultChannelPipeline - 205行
public final ChannelPipeline addLast(EventExecutorGroup group, String name, 
                                     ChannelHandler handler) {
    // netty并不是维护ChannelHandler链表, 而是维护ChannelHandlerContext链表
    final AbstractChannelHandlerContext newCtx;

    // 获取当前管道ChannlePipeline的对象锁, 每个Channel都有唯一的管道, 不同Channel间互不影响.
    synchronized (this) {
        // 检查handler是否可以被添加, 查看源码：checkMultiplicity()
        checkMultiplicity(handler);

        // 通过filterName()方法：若name为空根据handler类名生成, 然后遍历当前管道底层链表
        // 每个AbstractChannelHandlerContext, 判断是否有名称重复. 最后通过newContext()
        // 创建DefaultChannelHandlerContext实例, 直接调用它的构造方法生成, 方法内会根据
        // ChannelInboundHandler还是ChannelOutboundHandler决定是入栈处理器还是出栈处理
        newCtx = newContext(group, filterName(name, handler), handler);

        // 将创建好的上下文加入到链表中, 查看源码：addLast0()
        addLast0(newCtx);

        // 若registered变量为false, 说明当前ChannelPipeline关联的通道Channel并没有注
        // 册到EventLoop上. 此时, 就会添加一个任务PendingHandlerCallback到当前管道上,
        // 一旦通道注册上了, 就会调用ChannelHandler.handlerAdded(...)
        if (!registered) {
            // CAS修改上下文的状态(handlerState变量)由INIT → ADD_PENDING, 表示即将添加
            newCtx.setAddPending();

            // DefaultChannelPipeline内部维护一个单向任务列表PendingHandlerCallback,
            // 用来回调ChannelHandler.handlerAdded()方法.
            callHandlerCallbackLater(newCtx, true);

            return this;
        }
        // 若通道Channel已经注册, 则从上下文中获取事件执行器
        EventExecutor executor = newCtx.executor();

        // 若当前线程不是EventExecutor内部维护的Thread对象, 则添加一个任务.
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    // 若当前线程就是EventExecutor内部维护的Thread对象, 直接调用callHandlerAdded0().
    // 回调ChannelHandler.handlerAdded()方法
    callHandlerAdded0(newCtx);
    return this;
}
```

##### 1.4.2.2.1.checkMultiplicity()

```java
// 源码：DefaultChannelPipeline - 592行
private static void checkMultiplicity(ChannelHandler handler) {
    // 只针对ChannelHandlerAdapter类型的处理器做校验
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        // 如果handler不是共享性(没有使用@Sharable注解), 而且它已经被添加了, 直接抛错
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException();
        }
        // 其它情况, 将此handler的added标志置为true, 表明它已经被添加过了.
        h.added = true;
    }
}
```

##### 1.4.2.2.2.addList0()

```java
// 源码：DefaultChannelPipeline - 239行
private void addLast0(AbstractChannelHandlerContext newCtx) {
    // DefaultChannelPipeline内有2个指针：head和tail, 类型为内部类：HeadContext和
    // TailContext, 它们并不直接指向实际有效节点, 而是双向关联实际节点. 
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

### 1.4.3.注册通道

在创建完并且初始化完通道后，就会继续调用EventGroup.register()方法注册通道，源代码是这样子：

```java
// 通过AbstractBootstrap.config().group()获取到EventLoopGroup对象, 即创建事件循环组
// 到的NioEventLoopGroup, 调用它的register()
ChannelFuture regFuture = config().group().register(channel);
```

#### 1.4.3.1.EventLoopGroup.register()

NioEventLoopGroup的register()是由其父类MultithreadEventLoopGroup来实现，源码为：

```java
// 源码：MultithreadEventLoopGroup - 85行
public ChannelFuture register(Channel channel) {
    // 调用next()方法, 实际通过EventExecutorChooserFactory.EventExecutorChooser选择一
    // 个EventExecutor, 将Channel注册到它身上
    return next().register(channel);
}
```

上面知道next()方法是返回一个事件执行器EventExecutor，通过它来注册通道Channel。也就是说EventLoopGroup每次都会选择一个EventExecutor，让Channel注册到它身上。由[前面](#1.2.创建事件执行器)分析知道，事件执行器实现类为NioEventLoop，它的register()方法由SingleThreadEventLoop实现，源码为：

```java
// 源码：SingleThreadEventLoop - 73行
public ChannelFuture register(Channel channel) {
    // 会将通道channel和this对象(EventExecutor类型)创建DefaultChannelPromise, 
    // 继续调用重载方法
    return register(new DefaultChannelPromise(channel, this));
}
```

上面的register()方法会调用下面的重载方法，最终，实际上还是通过通道上的Unsafe类来注册通道，源码为：

```java
// 源码：SingleThreadEventLoop - 78行
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // 创建通道时在调用顶级父类AbstractChannel构造方法, 会创建NioMessageUnsafe, 所以
    // 每个Channel都有自己的Unsafe实现. 最终就是通过Unsafe.regisrer()来注册Channel. 
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

#### 1.4.3.2.Unsafe.register()

NioMessageUnsafe.register()方法是由其父类AbstractUnsafe实现，AbstractUnsafe是一个内部类位于AbstractChannel内，源码为：

```java
// 源码：AbstractChannel.AbstractUnsafe - 459行
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // eventLoop判空
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    // 如果通道已经注册了, 就报错..
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException());
        return;
    }
    // 判断当前EventLoop是否兼容, 在这里即当前EventLoop是否为NioEventLoop类型
    if (!isCompatible(eventLoop)) {
        promise.setFailure(new IllegalStateException());
        return;
    }
    // AbstractChannel.this的意思就是获取AbstractChannel的具体实现子类, 即
    // NioServerSocketChannel, 将方法参数eventLoop赋值给它
    AbstractChannel.this.eventLoop = eventLoop;
    /*
   * 下面这种代码格式在netty中极为常见, 它通过inEventLoop()判断当前线程是不是
   * eventLoop底层引用的Thread对象, 若是直接调用; 否则提交任务到eventLoop中, 最终由
   * 底层引用的线程去执行. 最终调用register0()方法注册通道
   */
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            // 若任务提交失败
            closeForcibly(); //调用javaChannel()方法获取nio通道将其关闭
            closeFuture.setClosed(); //设值CloseFuture, 通知它旗下的监听器, 已关闭通道
            safeSetFailure(promise, t);//将方法参数promise置为failure, 通知旗下监听器
        }
    }
}
```

#### 1.4.3.3.register0()

```java
// 源码：AbstractChannel.AbstractUnsafe - 496行
private void register0(ChannelPromise promise) {
    try {
        // setUncancellable()是设置当前Future不能被取消, 当它返回false意味着该Future
        // 已经被取消了; ensureOpen()是保证通道Channel时开着的. 当这两个方法有一个返回
        // false, 则register0()直接结束执行.
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        // neverRegistered为ture说明当前通道从未注册过, 其它情况都为false. 这边先将它
        // 的值用另一个变量保存起来, 用于下面触发ChannelActive()时使用.
        boolean firstRegistration = neverRegistered;

        // 调用doRegister()真正注册通道
        doRegister();

        // 注册完毕, 将通道标识为已注册状态.
        neverRegistered = false;
        registered = true;

        // 之前在创建ChannlePipeline时, 曾分析过一个变量pendingHandlerCallbackHead,
        // 它是预防Channel还未注册到EventLoop中就回调Channelhandler的handlerAdded()
        // 方法, 所以DefaultChannelPipeline将其用一个任务链表缓存起来. 下面这行代码就是
        // 依次调用链表上的每个任务, 回调Channelhandler.handlerAdded()方法.
        pipeline.invokeHandlerAddedIfNeeded();

        // 尝试将ChannelPromise设置为success, 并通知旗下的所有监听器
        safeSetSuccess(promise);

        // 回调ChannelInboundHandler.channelRegistered()方法
        pipeline.fireChannelRegistered();

        // 如果通道已经激活且已经在等待客户端连接
        if (isActive()) {
            if (firstRegistration) {
                // 仅当从未注册过通道时才触发channelActive, 这是为了防止在通道注销并
                // 重新注册情况下, 导致触发多次触发channelActive事件
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // config().isAutoRead()当且仅当ChannelHandlerContext.read()会自动被
                // 调用而无需用户程序主动调时, 返回true(即当前通道可以自动读取)
                // 才会调用beginRead()方法
                beginRead();
            }
        }
    } catch (Throwable t) {
        // 出现异常, 直接关闭通道以避免FD泄漏
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

#### 1.4.3.4.doRegister()

AbstractChannel.doRegister()没有实现，子类要自定义注册逻辑，就会交给AbstractNioChannel来实现，源码：

```java
// 源码：AbstractNioChannel - 383行
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // javaChannel()是返回nio通道SelectableChannel实例;
            // eventLoop().unwrappedSelector()是返回nio选择器Selector;
            // 然后将通道注册到选择器上, 同时还把当前通道NioServerSocketChannel当成attachment
            // 一般nio注册, 都会指定SelectionKey.OP_ACCEPT类似值, 但netty这边指定为0. 原因：
            // SelectionKey#interestOps(int ops)方法可以方便地修改监听操作位, 所以, 这边会将
            // 注册返回的SelectionKey 赋给AbstractNioChannel的成员变量 selectionKey.真正
            // 设置感兴趣的事件,会留到执行触发ChannelActive事件时完成.
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(),
                                                  0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // 强制Selector立即选择， 因为“取消的”选择键可能仍被缓存并且未被删除, 
                // 因为尚未调用Select.select()操作
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

## 1.5.【绑定通道】

在准备通道（从创建→初始化→注册）执行完后，一个ServerSocketChannel就可以绑定到端口上，这就会调用AbstractBootstrap#doBind0()。该方法一定会在initAndRegister()方法之后执行，说明此时通道一定已经初始化完成并且也注册到一个EventLoop上。这时这个方法就是将通道绑定到端口上。doBind0()源码：

```java
// 源码：AbstractBootstrap - 355行
private static void doBind0(
    final ChannelFuture regFuture, final Channel channel,
    final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  
    // Give user handlers a chance to set up the pipeline in its channelRegistered()
    // implementation.

    // 获取通道事件循环组, 并向其提交一个任务
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            // 如果regFuture成功了, 直接调用Channel的bind()方法, 并且添加一个监听器
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(
                    ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                // regFuture失败则设置失败项, 然后返回
                promise.setFailure(regFuture.cause());
            }
        }
    });
    
    // 到此主线程就会返回了,后面事情就会交由EventLoop中的线程来执行....
}
```

### 1.5.1.ChannelOutboundInvoker.bind()

在[doBind0()](#1.3.doBind0())方法中，可以看到netty实际是通过ChannelOutboundInvoker#bind()绑定通道，方法交由AbstractChannel实现，源码为：

```java
// 源码：AbstractChannel - 253行
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    // 通过管道ChannelPipeline绑定
    return pipeline.bind(localAddress, promise);
}
```

查看默认管道实现DefaultChannelPipeline的bind()方法实现：

```java
// 源码：DefaultChannelPipeline - 988行
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    // 交由管道维护的AbstractChannelHandlerContext上下文链表的尾节点执行bind()方法
    return tail.bind(localAddress, promise);
}
```

由于DefaultChannelPipeline的tail是TailContext类型，它继承自AbstractChannelHandlerContext，其中bind()方法在抽象父类中已经实现：

```java
// 源码：AbstractChannelHandlerContext - 474行
public ChannelFuture bind(final SocketAddress localAddress, 
                          final ChannelPromise promise) {
    // 类型校验...
    if (localAddress == null) 
        throw new NullPointerException("localAddress");
    if (isNotValidPromise(promise, false)) 
        return promise;

    // findContextOutbound()从链表尾部一直向前找, 直至找到一个ChannelOutboundHandler
    // 类型的上下文, 一般是管道上下文链表的头结点DefaultChannelPipeline$HeadContext对象
    final AbstractChannelHandlerContext next = findContextOutbound();

    // 获取AbstractChannelHandlerContext对应的EventExecutor
    EventExecutor executor = next.executor();

    // 常见的线程模型, 若当前线程是EventExecutor底层Thread, 则直接调用; 否则添加一个任务,
    // 让底层Thread慢慢消费. 实际调用invokeBind()方法.
    if (executor.inEventLoop()) {
        next.invokeBind(localAddress, promise);
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeBind(localAddress, promise);
            }
        }, promise, null);
    }
    return promise;
}
```

### 1.5.2.invokeBind()

invokeBind()是AbstractChannelHandlerContext的私有方法，而且需要注意的是此时的AbstractChannelHandlerContext实例对象是管道的内部类DefaultChannelPipeline$HeadContext对象。源码为：

```java
// 源码：AbstractChannelHandlerContext - 498行
private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
    // invokeHandler()方法判断当前上下文AbstractChannelHandlerContext是否已经
    // 回调过ChannelHandler.handlerAdded()方法.
    if (invokeHandler()) {
        try {
            // 获取与ChannelHandlerContext绑定的ChannelHandler对象, 实际还是
            // DefaultChannelPipeline$HeadContext(看它的类定义即可), 调用它
            // 的bind()方法, 最终落到Unsafe.bind()方法上
            ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    } else {
        // 如果invokeHandler()返回false, 则回去继续调用bind(), 取另一个上下文来调用.
        bind(localAddress, promise);
    }
}
```

### 1.5.3.Unsafe.bind()

Unsafe#bind()方法才会真正将通道绑定起来，交由抽象类AbstractUnsafe来实现，源码为：

```java
// 源码：AbstractChannel$AbstractUnsafe - 498行
public final void bind(SocketAddress localAddress, ChannelPromise promise){
    // 变量条件判断
    assertEventLoop();
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    // See: https://github.com/netty/netty/issues/576. 下面代码省略, 就是做了日志警告
    // ...
    // 记录 Channel 是否激活
    boolean wasActive = isActive();
    try {
        // 调用doBind()方法
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    // 如果通道之前未激活, 现在激活了, 则回调ChannelActive()方法
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                // 执行事件触发调用链,当前事件为ChannelActive,在这里会重新设置
                // ServerSocketChannel的感兴趣事件.
                pipeline.fireChannelActive();
            }
        });
    }
    // 设置成功状态
    safeSetSuccess(promise);
}
```

#### 1.5.3.1.doBind()

```java
// 源码：NioServerSocketChannel - 126行
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        // 如果Java版本大于等于7, 则通过nio通道绑定
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        // 小于7的JDK版本, 通过nio通道的Socket来绑定
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

#### 1.5.3.2.fireChannelActive()

这边的 pipeline.fireChannelActive()理论上只是一个事件的触发，但是netty会在这个调用链中完成对通道感兴趣事件的修改，所以一起拿到这边来说：

```java
// 源码：DefaultChannelPipeline
public final ChannelPipeline fireChannelActive() {
    // pipeline上下文调用链的首位元素永远都是HeadContext对象,也就是参数中的head,即链表的首结点
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}
```

其中回调的逻辑就是AbstractChannelHandlerContext的实现细节了，这边咱不讨论，我们只需要知道这个事件会交由DefaultChannelPipeline$HeadContext#channelActive()方法执行：

```java
// 源码：DefaultChannelPipeline$HeadContext
public void channelActive(ChannelHandlerContext ctx) {
    // 调用下一个上下文实例用来传递事件
    ctx.fireChannelActive();
    // 调用下面方法, 重新设置感兴趣事件
    readIfIsAutoRead();
}
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        // 这个配置默认都为true, 所以就会调用channel.read()方法
        channel.read();
    }
}
```

调用channel.read()又会调用pipeline.read()，然后交由DefaultChannelPipeline$TailContext来执行，它又会从回调链路中找出一个可以执行read()的上下文，那最终又会回到DefaultChannelPipeline\$HeadContext中，回调它的read()方法，然后它会交给改Channel对应的UnSafe对象....饶了一大圈最终交给AbstractNioChannel：

```java
// 源码：AbstractNioChannel
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }
    readPending = true;
    // 最终设置感兴趣事件为0或16,这里要回顾下NIO的事件：
    // 1. 读 :  SelectionKey.OP_READ   （值为1）
    // 2. 写 :  SelectionKey.OP_WRITE  （值为4）
    // 3. 连接 : SelectionKey.OP_CONNECT （值为8）
    // 4. 接收 : SelectionKey.OP_ACCEPT  （值为16）    
    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

## 1.6.【主线程等待】

在netty启动服务端模型代码中，还有这样两行代码：

```java
ChannelFuture channelFuture = serverBootstrap.bind(9090).sync();
channelFuture.channel().closeFuture().sync();
```

在调用bind()方法获取到一个ChannleFuture，通过它的sync()方法阻塞到通道的所有初始化操作完毕。在通过这个ChannelFuture获取通道channel，再获取通道内的closeFuture (它会在通道关闭时通知旗下监听器)，接着调用它的sync()方法，这样整个netty服务端就会一直运行直至服务器的通道NioServerSocketChannel关闭！！！

```java
// 源码：DefaultPromise - 332行
public Promise<V> sync() throws InterruptedException {
    // 调用await()等待Future执行完毕
    await();
    rethrowIfFailed();
    return this;
}
```

### 1.6.1.await()

await()方法其实就是利用Object提供的wait()和notifyAll()方法，在一个Future未完成时，阻塞当前线程; 在Future执行成功后，唤醒阻塞线程：

```java
// 源码：DefaultPromise - 217行
public Promise<V> await() throws InterruptedException {
    // 如果Future已经执行完, 即成员变量result不为空, 则方法直接返回
    if (isDone()) {
        return this;
    }
    // 上面if语句没返回说明了这个Future肯定是没有完成的
    // 判断线程是否以中断
    if (Thread.interrupted()) {
        throw new InterruptedException(toString());
    }
    // 判断死锁：若当前线程与这个Future关联的EventExecutor所在的线程是同一个,则肯定会死锁.
    checkDeadLock();
    // 对当前Future加锁
    synchronized (this) {
        // 使用wait()进行线程通信, 建议用while, 而不是单纯地使用if判断, 防止假唤醒.
        while (!isDone()) {
            // 等待这个Future的线程数加一
            incWaiters();
            try {
                // 在这里阻塞, 等待Future完成
                wait();
            } finally {
                // 等待这个Future的线程数减一
                decWaiters();
            }
        }
    }
    return this;
}
```

注：netty的Future还有一个这样的方法checkNotifyWaiters()会被setValue0()调用，而setValue0()方法会被setSuccess()或setFailure()调用，这其实就是netty的Future监听器原理.

```java
// 源码：DefaultPromise - 548行
private synchronized void checkNotifyWaiters() {
    if (waiters > 0) {
        // 唤醒所有阻塞在await()的线程
        notifyAll();
    }
}
```

# 2.源码：I/O交互

经过对[启动服务端](#1.源码：启动服务端)源代码的分析，确定了事件循环实现为：io.netty.channel.nio.NioEventLoop，它继承了抽象父类io.netty.util.concurrent.SingleThreadEventExecutor，而父类提供了实现了execute()方法。回忆一下是否存在这样的代码：Channel.eventLoop().execute()即利用通道channel的事件循环提交一个任务，它实际就是调用SingleThreadEventExecutor.execute()。这实际就是nio的Selector执行I/O多路复用和处理SelectionKey的起点。

```java
// 源码：SingleThreadEventExecutor - 736行
public void execute(Runnable task) {
  if (task == null) {
    throw new NullPointerException("task");
  }
  boolean inEventLoop = inEventLoop();
  if (inEventLoop) {
    addTask(task);
  } else {
    startThread();
    addTask(task);
    if (isShutdown() && removeTask(task)) {
      reject();
    }
  }

  if (!addTaskWakesUp && wakesUpForTask(task)) {
    wakeup(inEventLoop);
  }
}
```

## 2.1.startThread()

....

## 2.2.run()

....

# 3.源码：内存分配