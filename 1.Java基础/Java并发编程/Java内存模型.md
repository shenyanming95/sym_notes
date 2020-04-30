# 1.Java内存模型

JMM，全名是Java Memory Model，即Java内存模型。这里的Java内存模型与JVM内存模型是不同的概念，它相当于下图所示的结构：

![](./images\java内存模型.png)

主内存中存储程序定义的变量，这个变量包括了实例变量、静态变量和构成数组对象的元素，但是不包括局部变量和方法参数（因为局部变量和方法参数是线程私有的）。主内存是多线程共享的，所有线程都可以访问主内存，每个线程不能直接在主内存中操作变量，都是从主内存中拷贝变量的副本到自己的工作内存中，对变量操作完后，再重新写入主内存中。线程只能访问自己的工作内存，不可以访问其它线程的工作内存。

## 1.1.内存交互

### 1.1.1.主内存与工作内存：交互过程

Java内存模型规定了工作内存与主内存之间交互的协议，定义了8种原子操作，

Java中所有共享变量都是以这种方式读取的：（以下8个操作每一个都是原子性）

①**lock**：锁定主内存的变量，将一个变量标识为某个线程独占的状态；

②**unclock**：释放被lock锁定的变量，释放后的变量才可以被其它线程锁定；

③**read**：将变量的值从主内存传输到线程工作内存中，以便后续的load操作；

④**load**：将read操作从主内存获取得的变量值载入工作内存的变量副本中；

⑤**use**：将工作内存中一个变量的值传递给线程的执行引擎；

⑥**assign**：将执行引擎处理返回的值重新赋值给工作内存中的变量副本；

⑦**store**：将工作内存中某个变量的值传输到主内存中，以便后续的write操作；

⑧**write**：将store操作从工作内存得到的变量的值写入主内存的共享变量中。

![](./images\JMM内存交互.png)

### 1.1.2.主内存与工作内存：交互规则

Java内存模型还规定在执行8种原子操作时必须满足以下规则：

- 不允许read和load之间、store和write之间只有单个操作出现，即不允许变 量从主内存读出但工作内存不接受；变量从工作内存发起写回，主内存不接受。

- 不允许线程丢弃它最近的assign操作，即**变量的值在工作内存改变了，必须要*将该变量改变后的值同步回主内存**中。

- 不允许线程没有发生过任何assign操作，就把变量的值从工作内存中同步回主内存中。

- 一个新变量只能在主内存中定义，不允许直接在工作内存使用未被初始化 (没有经过load和assign操作)的变量。

- **一个变量在同一时刻只允许一条线程对其lock锁定操作**，但lock操作可以被同一条线程重复执行多次；执行多少次load，就要执行多少次unlock，变量才会被解锁。

- 如果一个线程对一个变量执行lock操作，将会**清空该线程的工作内存中此变**量的值，在执行引擎使用此变量前，需要重新执行load和assign操作。

- 如果一个变量未被lock操作锁定，就不允许对其执行unlock操作；也不允许去unlock一个被其它线程锁定住的变量。
- 对变量执行unlock之前，必须将此变量同步回主内存(执行store和write操作)

## 1.2.JMM规定

### 1.2.1.happen-before规则

happen-before，即先行发生原则。何为先行发生？指的是JMM定义的两项操作之间的偏序关系。假设操作A和操作B，如果保证A先行发生于B，则A的操作对B是可观察的，包括修改共享数据，发送消息，调用方法..等等。反之，如果无法确保A和B哪个先行发生，则无法保障A和B之间的操作互相可观察。举个例子：

```java
// 线程A
int i = 1;

// 线程B
int j = i;

// 线程C
int i = 2;
```

假设线程A先行发生于线程B，则可以在线程B执行后，变量j值为1，能确保这个结果的原因有两个：

①根据先行发生原则，线程A的操作“i=1”可以被线程B观察到；

②线程C还未启动

但是在线程C启动，即使确保线程A先行发生于线程B，但是B与C之间没有确定的先行发生关系，线程C有可能先于B执行，也有可能后于B执行，那么线程C的操作可能被B观察到，也可能不会，这就导致B可能读取到过期数据的风险，j的值可能为1，也可能为2。

根据《深入理解Java虚拟机第2版》介绍，Java自带8种先行发生规则，如果两个操作之间不在这8个规则，并且无法从这8个规则推导出来。则这两个操作就不会有顺序保障，虚拟机可以对它们进行重排序。（换句话说：不满足先行发生原则，就可能产生多线程数据问题）

| **名称**         | **规则**                                                     |
| :--------------- | ------------------------------------------------------------ |
| 程序次序规则     | 在一个线程中，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。 |
| 管程锁定规则     | 一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调是同一个锁，而“后面”是指时间上的先后顺序 |
| volatile变量规则 | 对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序 |
| 线程启动规则     | Thread对象的start()方法先行发生于此线程的每一个动作          |
| 线程终止规则     | 线程中的所有操作都先行发生于对此线程的终止检测，可以通过Thread.join()方法结束、Thread.isAlive()的返回等手段检测到线程已经终止执行 |
| 线程中断规则     | 对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.  interrupted()方法检测到是否有中断发生 |
| 对象终结规则     | 一个对象的初始化完成（构造方法执行结束）先行发生于它的finalize()方法的开始 |
| 传递性           | 如果操作A先行发生于操作B，操作B先行发生于操作C，则得出操作A先行发生于操作C的结论 |

### 1.2.2.as-if-serial语义

认识as-if-serial语义之前，先认识一个名词：指令重排序，它是编译器和处理器为了提高并行度，而会对代码指令进行重新排序。

**as-if-serial**：不管怎么重排序，（单线程）程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义，为了遵守as-if-serial语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被编译器和处理器重排序。举个例子：

```java
double pi = 3.14; //操作A
double r = 1.0; //操作B
double area = pi*r*r; // 操作C
```

A-C，B-C之间存在数据依赖关系，因此C不能被重排序到A或B前面(C的计算要A和B的赋值，如果C排在A和B前面，程序结果会被改变)。但是A和B之间没有依赖关系，编译器和处理器可以重排序A和B 之间的执行顺序！

# 2.内存可见性-volatile

volatile是Java中的关键字，用来实现变量的可见性：即一个线程修改了被volatile修饰的变量的值，其它线程都可以看到。它是处于普通变量与加锁变量之间的维度，它不会像synchronized一样锁住变量

## 2.1.底层原理

当查看被volatile修饰的变量的汇编代码时，会发现它在前面加了个lock：

![](./images\volatile底层原理.png)

其实volatile底层就是通过汇编lock前缀指令来实现，IA-32架构软件开发者手册对lock指令的解释：

①会将当前CPU缓存行的数据**立即**写回主内存 

②写回主内存的操作会引起在其它CPU缓存了该内存地址的数据无效（MESI）

**volatile整体的原理是这样：**

多个线程在操作被volatile修饰的共享变量v时，都会将v值从主内存拷贝到各自独有的工作内存中，当其中一个线程在自己的工作内存中将v值修改了，它就立即将新v值写回主内存，同时，其它线程通过**总线嗅探机制**可以感知到数据的变化立即将自己缓存里的数据失效，重新执行read和load操作，将变量v从主内存中拷贝到工作内存里，这里依靠的是硬件级别的MESI缓存一致性原则

## 2.2.执行流程

这里借图灵学院-诸葛老师画的一张图来举例说明volatile的执行流程：

![](./images\volatile执行流程.png)

假设两个线程(CPU)并行操作共享变量initFlag，它们分别先执行了JMM规定的read和load两个原子操作将主内存数据拷贝到各自工作内存中，此时的initFlag=false，这时候线程2获取到时间片，它执行use和assign操作，将initFlag的值由false改为true，就在这个瞬间，MESI缓存一致性原则生效，线程2会立即执行store和write原子操作，将initFlag的新值写回到主内存里，同时借助总线嗅探机制的监听，会让线程1工作内存中的initFlag的内存地址失效掉(可以理解为置空了)，线程1这时候就会重新执行read和load个原子操作，重新将initFlag读回到自己的工作内存里面去，这时候执行引擎就会拿到最新值（执行引擎是一直与线程工作内存打交道的，发现工作内存的数据失效了，它也执行不了了）

 这里可能会有个疑问：线程2修改共享变量后，立即写回主内存，同时线程1由于内存失效，它也会去读主内存，如果线程1优先于线程2执行，线程1岂不是仍然读到旧值？其实不会，注意看上图，线程2在执行store和write两个原子操作之前，会对主内存的initFlag加一个缓存行锁(这个锁超级快，可以忽略不计)，在write原子操作没执行完之前，线程1如果想读取主内存的initFlag，只能先等待，这样就保证了线程1拿到的数据是最新值！

## 2.3.使用场景

volatile只能保证有序性和可见性，但无法保证原子性，所以它的使用场景：

①**确保只有一个线程操作volatile修饰的变量，其它线程都是读取而已**；

②**被volatile修饰的变量的值没有依赖于其它未控制线程安全的变量的值**。

```java
//直接在定义变量的时候加上volatile关键字
private volatile static int value = 0;
```

### 2.3.1.不保证原子性

volatile为什么不能保证原子性？举个例子，在两个线程中，对共享变量i各自进行1000次自增操作（i++不是原子操作，它实际上是三个步骤），理论上i最后值应该为2000，但实际上是小于等于2000的。根据MESI缓存一致性原则，线程1修改了变量i的值，立即使线程2缓存的变量i失效，若此时是这样：在线程2执行完use操作，执行引擎已经循环一次，准备执行assign将值写回工作内存的变量副本，但恰巧线程1先执行完assign，它会将变量i同步会主内存，导致线程2的变量副本内存地址失效了，它就重新到主内存读取，然后又交给执行引擎执行，此时的执行引擎要进行第二次循环（第一次循环由于内存失效就不起效果了）。这里就可以看出问题了，两个线程循环了3次，但实际i只自增了两次，数据就不正确了

### 2.3.2.指令重排序

Volatile虽然不能保证原子性，但是它可以防止指令重排序。何为指令重排序？

![](./images\指令重排序.jpg)

就是源代码编译成可执行机器码，会经过一系列重排序，例如：1的java编译器重排序，2和3的处理器重排序。原本代码定义的执行顺序是1-2-3,经过指令重排序后，执行顺序可能变为1-3-2，这可能导致多线程环境下可能出现问题，例如下面代码：

```java
public class Singleton {
private static Singleton singleton;
    private Singleton(){};
    public static Singleton getSingleton(){
        if( singleton == null ){
            synchronized (Singleton.class){
                if( singleton == null ){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

一个看似很完美的双检锁单例，但是忽略了一个问题，就是new指令其实不是一个原子性操作，它实际操作为：

① 虚拟机遇到new指令，到常量池定位到这个类的符号引用

② 检查符号引用代表的类是否被加载、解析、初始化过

③ 虚拟机为对象分配内存

④ 虚拟机将分配到的内存空间都初始化为零值

⑤ 虚拟机对对象进行必要的设置

⑥ 将对象的引用指向这个内存区域

**归纳为三步：**

1、 JVM为对象o分配一块内存n

2、 JVM在内存n上为对象o初始化

3、 将内存地址赋值给对象o的引用

按照指令重排序，第1步一定先执行(因为2,3步依赖于第1步)，但是2和3没有依赖关系，可能执行顺序会变为1-3-2，先赋值引用再进行初始化。在上面代码中，假设有2个线程在调用方法getSingleton()。如果没发生指令重排序，Thread1抢到锁进入方法为Singleton初始化，在第3步执行之前，singleton==null一直为true，此时thread2就会在synchronized那边等待锁资源；一旦发生了指令重排序,thread1执行完第1步后直接执行第3步，将内存地址赋值给引用，这是singleton==null就会返回false，thread2就直接越过if判断，将对象返回，但此时对象仍在初始化，就极有可能抛出空指针异常。所以对于这种情况，应该为Singleton对象加上volatile修饰，防止其指令重排序，其原理是内存屏障！！！

# 3.线程互斥-synchronized

Java中线程互斥，可以使用**synchronized**关键字，它是一种互斥锁，是由JVM控制的（区别JUC下的Lock，它是人为控制）。之前synchronized都被称为重量级锁，随着Java SE 1.6对其优化，引入了偏向锁和轻量级锁，使其性能与JUC下的Lock性能相当，只不过Lock比synchronized多了额外功能，在同等场景下，优先使用synchronized！！！

## 3.1.使用方式

明确一点，synchronized需要锁住一个对象，所以怎么使用synchronized，就是区分锁住什么对象？下面是4种synchronized的用法

### 3.1.1.修饰代码块

修饰代码块，被修饰的代码块称为同步语句块，其作用的范围是花括号{}包裹起来的代码，锁住的对象是调用这个代码块的对象。所以，多个线程在执行时就需要用这个对象作为线程共享变量（以便可以竞争同一把锁）。

```java
public class SynchronizedForCodeDiv implements Runnable {
    // 模拟：线程的共享变量
    private int ticket = 77;
    // run()代表线程要做的事
    @Override
    public void run() {
        /**
         * 如果去掉 synchronized，3个线程竞争cpu资源，执行结果时线程交叉打印 ticket 递减
         * 的值；如果加上 synchronized，3个线程只有一个可以拿到 MyThread 对象的锁，执行结
         * 果时一个线程从77打印到1
         */
        // 未被synchronized锁住的代码
        System.out.println(Thread.currentThread().getName() + "开始执行");
        synchronized ( this ){
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + ":" + ticket--);
            }
        }
        // 未被synchronized锁住的代码
        System.out.println(Thread.currentThread().getName() + "结束执行");
    }
    public static void main(String[] args) {
    // 实例化 SynchronizedForStaticMethod 对象，该对象就有一个同步锁，用此对象创建的线程相
    // 互之间才会互斥。如果创建多个 SynchronizedForStaticMethod 对象，这些对象就各自拥有自
    // 己的同步锁，由这些对象创建的线程就不会互斥。
    SynchronizedForStaticMethod myThread = new SynchronizedForStaticMethod();
    // 开启3个线程,谁抢到锁,谁就去执行
    new Thread(myThread, "线程1").start();
    new Thread(myThread, "线程2").start();
    new Thread(myThread, "线程3").start();
    }
}
```

上面代码synchronized锁住的this，就是指一个Mythread实例对象，当用同一个实例对象创建线程时，线程之间才会起到互斥的作用；因为synchronized只锁定对象，每个对象只有一个锁与之相关联，但是如果创建两个Mythread实例对象，会有两把锁分别锁定这两个对象，这两把锁之间是互不干扰的，不形成互斥，使用这两个对象创建的线程之间，就不会有互斥效果。

**上面代码的执行逻辑：**

①线程1、线程2、线程3一起启动，准备竞争CPU资源

②哪个线程抢到CPU资源它就获得锁，就执行它的run()方法逻辑

③剩下两个线程就挂起等待，坐等锁资源的释放

④当抢到锁的线程执行完synchronized包裹起来的代码后，就会释放锁，剩下线

 程就可以执行未被synchronized包裹起来的代码

**synchronized只对指定代码块加锁：**

一个类中如果既有synchronized修饰的代码块也有非synchronized代码块，当一个线程访问一个对象的synchronized代码块时，别的线程可以访问该对象的非synchronized代码块而不会受阻塞（但是只能访问同步代码块之前的代码）。当然synchronized加锁的对象肯定不限制为this， 当有一个明确的对象作为锁时，可以使用下面这种方式:

```java
public void getApp(Object obj){
    synchronized (obj){
    }
}

```

如果没有明确的对象作为锁，只是想让一段代码同步时，可以创建一个特殊的对象来充当锁。实践证明：零长度的**byte数组对象**创建起来将比任何对象都实惠，查看它编译后的字节码：生成零长度的byte[]对象只需3条操作码，而Object lock = new Object()则需要7行操作码:

```java
private byte[] lock = new byte[0];
public void getApp(){
    synchronized (lock){
    }
}
```

### 3.1.2.修饰方法

synthronized修饰方法和修饰代码块的效果差不多，只不过代码块的作用范围是在花括号{}之内，而方法的作用范围则是在函数体内。在修饰方法时，synchronized放在方法访问权限和方法返回值之间，如：

```java
public synchronized void run(){
    
}
```

**需要注意的问题:**

①synchronized关键字不能继承。意思就是父类的方法使用了synchronized关键字，子类的方法重写父类方法，除非子类方法自己显式声明synchronized，否则是不会有互斥效果；当然，如果子类在重写的方法中调用了父类的方法，即使没有synchronized也会有互斥效果。

②synchronized不能修饰接口方法

③synchronized不能修饰构造方法，但可以用synchronized代码块包裹构造方法 内的代码

### 3.1.3.修饰静态方法

由于静态方法是属于类的，所有该类的实例对象都可以访问，所以当synchronized修饰静态方法时，是在类级别上加锁，即所有的实例对象都是同一把锁，该类所有实例对象都会有互斥效果。语法为：public synchronized static void run(){}，将synchronized关键字放在方法访问权限关键字和static关键字之间

```java
public class SynchronizedForStaticMethod implements Runnable {
    private static int ticket = 77;
    // run()代表线程要做的事
    @Override
    public void run() {
        /**
         * 当修饰static方法时,是在类级别上加锁,该类所有的实例对象都会存在互斥效果
         * 加锁的范围仅限于synchronized的方法之内,意味着方法之外的代码可以被其它
         * 未抢到锁的线程执行
         */
        System.out.println(Thread.currentThread().getName() + "开始执行");
        print();
        System.out.println(Thread.currentThread().getName() + "结束执行");
    }
    /**
     * synchronized修饰静态方法,关键字放在public和static之间,表示整个方法处于互斥状态
     */
    public synchronized static void print() {
        while (ticket > 0) {
            System.out.println(Thread.currentThread().getName() + ":" + ticket--);
        }
    }
    public static void main(String[] args) {
        // synchronized修饰类的静态方法,是在类级别上加锁(静态方法不属于任何一个实例对象)
        // 所以无论创建多少个实例对象,它们都是同一把锁,该类所有实例对象都会产生互斥效果
        SynchronizedForStaticMethod myThreadOne = new SynchronizedForStaticMethod();
        SynchronizedForStaticMethod myThreadTwo = new SynchronizedForStaticMethod();
        SynchronizedForStaticMethod myThreadThree = new SynchronizedForStaticMethod();
        // 开启3个线程,谁抢到锁,谁就去执行
        new Thread(myThreadOne, "线程1").start();
        new Thread(myThreadTwo, "线程2").start();
        new Thread(myThreadThree, "线程3").start();
    }
}
```

静态方法操作的是静态变量，静态变量同样是属于类的，不属于某个实例对象所有，当把静态变量修改后，每个实例对象都会获取到修改后的值。因此，当抢到锁的线程把print()方法执行完，ticker的值变为0，释放锁后其它线程抢夺该锁，但由于ticket的值已经变为0，所以就算抢到锁，也不会执行print()方法，因为循环条件不满足了。

### 3.1.4.修饰类

synchronized修饰类时，是对该类的类类型加锁，由于一个类只有一个类类型，所以该类的所有实例对象共用同一把锁，意味着该类的所有实例对象都会互斥。synchronized修饰类时，语法与一样，为：synchronized(T.class){}。效果与修饰静态方法一样，类级别加锁

```java
public class SynchronizedForClass implements Runnable{
    // 如果ticket改为静态类型,private static int ticket = 77 会有不一样的效果
    private int ticket = 77;
    // run()代表线程要做的事
    @Override
    public void run() {
        /**
         * synchronized修饰类,即对类的类类型加锁,由于一个类只有一个类类型,所以该类的所有实
         * 例对象都会互斥，加锁的范围仅限于synchronized包裹起来的代码块，未被包裹的代码可
         * 以被其它未抢到锁的线程执行
         */
        System.out.println(Thread.currentThread().getName() + "开始执行");
        print();
        System.out.println(Thread.currentThread().getName() + "结束执行");
    }
    /**
     * synchronized修饰类，就是对类的类类型加锁,该类所有对象都会互斥,效果与修改静态方法一样.
     * 但是,要注意方法逻辑操作的是局部变量?还是成员变量?还是静态变量,不同变量打印效果不一样
     */
   
public void print() {
        synchronized (SynchronizedForClass.class) {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + ":" + ticket--);
            }
        }
    }
    public static void main(String[] args) {
        // synchronized修饰类时,是对类的类类型加锁,由于一个类只有一个类类型,所以无论
        // 创建多少个实例对象,它们都是同一把锁，该类所有实例对象都会产生互斥效果。
        SynchronizedForClass myThreadOne = new SynchronizedForClass();
        SynchronizedForClass myThreadTwo = new SynchronizedForClass();
        SynchronizedForClass myThreadThree = new SynchronizedForClass();
        // 开启3个线程,谁抢到锁,谁就去执行
        new Thread(myThreadOne, "线程1").start();
        new Thread(myThreadTwo, "线程2").start();
        new Thread(myThreadThree, "线程3").start();
    }
}
```

synchronized修饰静态方法和修饰类的效果虽然一样，但是还是有所区别，就在于方法逻辑操作哪一种类型的变量。当方法逻辑操作的是静态变量，所以无论创建多少个实例对象，只要其中一个实例对象把静态变量的值改了，其它实例对象获取到的静态变量的值都是修改后的；而当方法逻辑操作的是成员变量，创建多少个实例对象，就有多少个成员变量，每个实例对象的成员变量都是互不干扰的。这就导致了虽然synchronized修饰静态方法和修饰类类型都是对类级别加锁，但是修饰静态方法时，先抢到锁的线程执行完方法逻辑后，静态变量值被改变了，其它线程就算抢到锁，也不会执行方法逻辑；而修饰类类型时，先抢到锁的线程执行完方法逻辑后，只是改变当前实例对象的成员变量，其它线程抢到锁后，获取的是它自己的成员变量值，与上一个线程修改的成员变量无关。

### 3.1.5.总结

①无论synchronized关键字加在方法上还是对象上，如果它作用的对象是非静态的，则它取得的锁作用于对象；如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是类，该类所有的对象同一把锁。

②每个对象只有一个锁（lock）与之相关联，谁拿到这个锁谁就可以运行这个锁控制的那段代码。 

③实现同步是要很大的系统开销作为代价的，甚至可能造成死锁，所以尽量避免无谓的同步控制

## 3.2.底层原理

《Java并发编程的艺术》一书指出：Synchonized的实现原理是基于进入和退出Monitor对象（Monitor又依赖于底层操作系统的mutex Lock来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现的（显式），而方法同步是使用方法修饰符ACC_SYNCHRONIZED（隐式）实现！！

### 3.2.1.monitor机制

在操作系统中，semaphore信号量和mutex互斥量是最重要的同步原语，在此基础上提出monitor机制，需要注意，操作系统本身不支持monitor机制，它是属于编程语言的范畴。因此在使用monitor时，需要先了解下编程语言是否支持，例如：C语言不支持monitor，java语言支持monitor。 下图是monitor机制的示意图，分为三部分：

entry set — 等待进入临界区的线程

the owner — 临界区

wait set  — 等待条件变量的线程

![](./images\monitor机制.png)

monitor机制最大的特点就是：同一时刻，只有一个线程可以进入临界区，其它线程都会被阻塞。entry set好理解，就是线程来到临界区，发现有线程已经进入临界区了，它就会在entry set阻塞等待；而wait set是这样：是线程已经进入临界区，但是条件不匹配(类似生产者-消费者模型)，它就会从临界区出来，进入wait set阻塞等待条件的满足，这个即Object类的wait()和notify()方法！！！等到进入临界区的线程出来后，entry set和wait set的线程就会共同竞争CPU，以获取进入临界区的资格！！！以上就是synchronized基于monitor机制实现的线程互斥效果，可以看出实现monitor机制需要3个元素：

①临界区

②monitor 对象及锁

③条件变量以及定义在 monitor 对象上的 wait，signal 操作

 这个临界区反映到Java层面，就是synchronized包裹的代码块或修饰的方法.而这个monitor对象映射到Java层面，就是synchronized锁住的对象前面的4种用法。Java虚拟机给每个对象都内置了一个监听器Monitor，当且一个monitor被持有后，它将处于锁定状态，Java会保存锁标识，将其保存到monitor对象的对象头上！！