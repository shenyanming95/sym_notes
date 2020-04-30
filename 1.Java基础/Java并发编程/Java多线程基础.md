并发编程三大特性：原子性、可见性、有序性

# 1.Java线程基础

## 1.1.线程状态

java语言把线程状态分为了6种，它们位于Thread类中的一个枚举类State。在任意一个时刻，线程只能处于其中的一个状态。在Java中，把操作系统中的运行和就绪两个状态合并称为运行状态

![](./images\java线程运行状态.jpg)

基于上面的6种，java线程状态的变迁如下：

![](./images\java线程变迁.jpg)

- **阻塞**：当一个线程试图获取一个内部的对象锁（非java.util.concurrent库中的锁），而该锁被其他线程持有，则该线程进入阻塞状态
- **等待**：当一个线程等待另一个线程通知调度器一个条件时，该线程进入等待状态，例如调用：Object.wait()、Thread.join()以及等待Lock或Condition

实际上不用区分两者，它们效果都一样：都是暂停线程的执行。只不过，等待状态是线程主动的，阻塞状态是线程被动的，更进一步说：等待状态是在同步代码(synchronized)之内，而阻塞状态是在同步代码(synchronized)之外！！！

## 1.2.线程优先级

Java线程优先级，表示线程的重要程度或紧急程度,优先级高的线程分配时间片的数量要多于优先级低的线程，意味着它较大概率获取到CPU的资源。线程的优先级分为1-10这10个等级，默认是5，值越大线程级别越高。Thread中提供了3个静态变量表示优先级：

```java
public final static int MIN_PRIORITY = 1;  //级别最小
public final static int NORM_PRIORITY = 5; //默认的
public final static int MAX_PRIORITY = 10; //级别最大
```

请注意，**线程的优先级与线程的执行顺序没有绝对联系**，并不是优先级高的线程一定比优先级低的线程先执行，具体还是要看能否抢到CPU资源；而且，**线程优先级不能作为程序正确性的依赖**。因为有些操作系统会忽略掉java线程对优先级的限定！！

## 1.3.启动线程

### 1.3.1.继承Thread

继承Thread类,重写run方法,调用Thread.start()方法便可以启动线程

```java
/**
 * 线程实现方式一：继承java.lang.Thread，重写run()方法
 */
public class MyThread extends Thread {
    /**
     * 线程要执行的逻辑，写在run()方法里
     */
@Override
public void run() {
        System.out.println("线程正在执行...");
    }
    public static void main(String[] args){
        /* 通过调用start()就可以开启线程 */0
        MyThread myThread = new MyThread()00;
        myThread.start();
    }
}

```

### 1.3.2.实现Runnable

实现Runnable接口,实现run方法,虽然是实现接口的方式,但是启动线程还是要先new Thread(),通过构造函数把接口对象传值,再调用start()方法启用线程。推荐使用实现Runnable接口的方式，因为用接口的方式，操作线程时更灵活。而且用接口执行线程，也可以实现共享变量...等

```java
/**
 * 线程实现方式二：实现java.lang.Runnable接口，实现run()方法
 */
public class MyThread implements Runnable {
    /**
     * 线程要执行的逻辑，写在run()方法里
     */
    @Override
    public void run() {
        System.out.println("线程正在执行...");
    }
    public static void main(String[] args){
        /* 将接口实现类传给Thread()，再调用start()方法允许线程 */
        MyThread myThread = new MyThread();
        new Thread(myThread).start();
    }
}
```

### 1.3.3.实现Callable

使用继承Thread类或实现Runnable接口，这两种方式执行线程都不能获取返回值(run()方法的返回值都定义为void)。JDK1.5以后，Java提供了Callable接口，可以获取线程执行后的返回值。要执callable接口，要么使用FutureTask，要么使用线程池:

```java
/**
 * 线程实现方式三：实现java.util.concurrent.Callable接口，实现call()方法
 */
public class MyThread implements Callable<String> {
    /**
     * 线程要执行的逻辑，写在call()方法里
     * @return 线程执行后的返回值
     */
    @Override
    public String call() throws Exception{
        return "线程执行后结果...";
    }
    public static void main(String[] args){
        // 实例化Callable接口 -> 来创建FutureTask对象 -> 创建Thread对象
        FutureTask<String> ft = new FutureTask<>(new MyThread());
        new Thread(ft).start();
        try {
            /* 通过FutureTask的get()方法获取返回值 */
            ft.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

## 1.4.终止线程

在Thread中，有几个过期方法：suspend()、resume()和stop()，它们分别代表暂停、恢复和终止线程。suspend()方法调用后，线程会占有着资源(比如锁)进入睡眠状态，易引发死锁问题；而stop()方法会直接杀死线程，很有可能资源还没释放，线程就已经停掉，导致程序运行处于不确定状态。因此，当前终止线程推荐使用**中断标志位**，由我们在程序中显示确定线程还要不要执行：

```java
Thread t = new Thread(()->{
while ( !Thread.currentThread().isInterrupted() ){
      // 只有当线程的中断标志位为false时，线程才会一直运行
      // 一旦此线程的被中断了，中断标志位就会返回true，这样在下次循环时
      // 线程就会优雅地退出，并且释放资源
    }
});
t.start();
```

## 1.5.多线程共享数据

“多线程共享数据”和“线程范围内数据共享”是不同的概念：

前一个：是多个线程，一起使用同一个数据，例如卖票系统的票数、缓冲区等。

后一个：线程内的数据共享于在线程内的各个模块，而线程间的数据互不干扰。

多线程共享数据，意味着多个线程需要操作同一个对象，这样对象内的数据对于线程而言，才是共享的。当每个线程的代码逻辑都是一样的，我们就可以创建一个Runnable接口，将对象放到接口中，以该接口创建的线程就实现共享数据；当每个线程的代码逻辑不一样，那么就需要把对象独立出来，然后多个线程一起操作这个对象，切记此时对象只能实例化一个。

### 1.5.1.执行逻辑相同

如果每个线程要执行的代码是一样的，只需要创建一个实现Runnable接口的类实例，将共享数据放到实现类中

```java
public class DataRunnable implements Runnable {
    // 共享数据
    private int data;
    // 良好的编程习惯，把线程逻辑写在外部方法中
    @Override
    public void run() {
        doSomething();
    }
    // 线程执行逻辑
    public void doSomething(){
      while( data>0 ){
        System.out.println(Thread.currentThread().getName()+",取走数据,剩余："+ --data);
      }
   }
}
```

只需要创建一个DataRunnable实例，使用该实例创建线程：

```java
public static void main(String[] args) {
    // 只初始化一个Runnable实现类
    Runnable runnable = new DataRunnable(10);
    // 开启3个线程操作
    for (int i = 1; i < 4; i++) {
        new Thread(runnable,"线程"+i).start();
    }
}
```

### 1.5.2.执行逻辑不同

#### 1.5.2.1.共享数据封装在对象中

将数据封装在对象中，只实例一个共享数据对象，其他线程一起来使用这个对象作逻辑处理，就可以实现数据共享，不同的线程在各自的run方法中实现不同的代码即可。

```java
public class SharingData {
    // 共享变量
    private int id;
    private String name;
}
```

多个线程执行时，操作同一个实例对象即可：

```java
public static void main(String[] args) {
    // 线程都操作 sharingData 实例对象
    SharingData sharingData = new SharingData(1,"测试");
   // 线程1
    new Thread(()->{
        sharingData.setName("线程1");
        System.out.println(sharingData);
    },"线程1").start();

    // 线程2
    new Thread(()->{
        sharingData.setName("线程2");
        System.out.println(sharingData);
    },"线程2").start();
}
```

#### 1.5.2.2.共享数据放在外部类中

这种方式跟"将共享数据放在一个对象中类似"，只不过，它是利用了内部类都能访问外部类变量的原理：将Runnable对象当做内部类，将共享数据放在外部类上，然后定义内部类，针对外部类的共享变量实现不同线程的不同处理

```java
public class ThreadSharingWithOuterClass {
    // 定义外部类成员变量，内部类可以直接访问外部类成员变量
    private int data = 10;
    // 良好的编程习惯，把线程执行逻辑定义到外部方法中
    public void print(String name) {
        while (data > 0) {
            System.out.println(name + "->" + --data);
        }
    }
    /**
     * 定义内部类线程1
     */
    class ThreadOne implements Runnable {
        @Override
        public void run() {
            print(Thread.currentThread().getName());
        }
    }
    /**
     * 定义内部类线程2
     */
    class ThreadTwo implements Runnable {
        @Override
        public void run() {
            print(Thread.currentThread().getName());
        }
    }
}
```

初始化内部类，需要先实例化外部类，通过外部类实例对象来初始化内部类

```java
public static void main(String[] args) {
    // 初始化内部类，需要先初始化外部类，然后通过外部类实例对象初始化内部类
    ThreadSharingWithOuterClass thread = new ThreadSharingWithOuterClass();
    new Thread(thread.new ThreadOne()).start();
    new Thread(thread.new ThreadTwo()).start();
}
```

## 1.6.常用方法

### 1.6.1.join()

将当前线程挂起，让出CPU资源给指定线程执行，直至指定线程执行完，再重新请求CPU资源执行。

```java
Thread t1 = new Thread(()->{
    for(int i=1;i<6;i++){
        System.out.println("线程1执行");
    }
});
Thread t2 = new Thread(()->{
    try {
        // t2线程会挂起，让t1线程先执行，直至t1线程执行完
        t1.join();
        for(int i=1;i<6;i++){
            System.out.println("线程2执行");
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
t1.start();
t2.start();
```

执行结果：

![](./images\join()执行结果.png)

### 1.6.2.yield()

yield()让当前线程从执行态（运行状态）变为可执行态（就绪状态），以允许具有相同优先级的其它线程获得运行的机会。但是需要注意的是，最终哪个线程能运行是靠cpu调度的，意味着当前线程有可能会再次被cpu调度而接着执行，所以yield()方法有可能没有任何效果。

```java
// 创建线程t1并启动
Thread t1 = new Thread(()->{
    for( int i=0;i<7;i++ ){
        // 当i=5的时候，t1让出cpu资源，然后和t2一起抢夺cpu调度
        // 谁抢到谁就可以执行
        if( i == 5 ){
            Thread.yield();
        }
        System.out.println("线程t1执行-"+i);
    }
});
t1.start();

// 创建线程t2并启动
Thread t2 = new Thread(()->{
    for( int i=0;i<7;i++ ){
        System.out.println("线程t2执行-"+i);
    }
});
t2.start();
```

如果yield()方法生效，即t2抢到了cpu资源，则会打印下面的结果：

![](./images\yield()执行结果.png)

t1在循环到i=5时，让出cpu资源，而t2先于t1抢到cpu资源，所以上图中可以看到t1打印到"4"的时候，t2插入打印了"0"。但是yield()并不是join()，它不会等待t2线程执行完，仍然会继续抢夺cpu资源，所以t1也会继续执行。

### 1.6.3.interrupt()

interrupt()可以标志线程的中断状态为true，它并不能中止线程的运行。使用它分为两种情况：

 ①对于阻塞的线程，调用interrupt()方法会立即抛出InterruptedException异常，并且清除此线程的中断状态

②对于正在运行的线程，调用interrupt()方法只是标志它为中断状态，并不会停止线程的运行

```java
// 对于阻塞的线程，调用interrupt()方法会抛出InterruptedException异常
Thread t1 = new Thread(() -> {
    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

// 对于正在运行的线程，只会标志线程的中断状态，而不会停止线程的运行
Thread t2 = new Thread(()->{
    for(;;){
       // 当t2线程被中断后，控制台并不会立即退出，仍在继续运行
       // 说明interrupt()并不会中止线程的运行
    }
});

// 启动线程
t1.start();
t2.start();

// 主线程分别中断t1和t2线程
try {
    Thread.sleep(1000);
    t1.interrupt();
    t2.interrupt();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

**注意：**

​    从Java的API中可以看到，许多声明抛出InterruptedException的方法（例如Thread.sleep(long millis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false！！

### 1.6.4.isInterrupted()

isInterrupted()返回指定线程的中断状态，注意此方法不会清除线程的中断状态而已，它只是作为一个判断而已：

```java
Thread t1 = new Thread(()->{
    for(;;){
        // 线程t1在此一直运行
    }
});
t1.start();
System.out.println("线程t1是否中断?"+t1.isInterrupted());
t1.interrupt();
System.out.println("线程t1是否中断?"+t1.isInterrupted());
System.out.println("线程t1是否中断?"+t1.isInterrupted());
```

### 1.6.5.interrupted()

Thread.interrupted()方法是获取当前线程的中断状态并且清除它的中断状态，意味着紧连两次调用它，第二次必定返回false

```java
Thread t1 = new Thread(()->{
    for(int i=0;i<10;i++){
        if( i == 5 ){
            // 因为t1线程未被中断，所以Thread.interrupted()返回false
            System.out.println("t1线程是否中断?"+Thread.interrupted());
            // 开始中断t1线程
            Thread.currentThread().interrupt();
            // 紧连两次调用Thread.interrupted()，第二次必定返回false，
            // 因为interrupted()方法会清除当前线程的中断状态
            System.out.println("t1线程是否中断?"+Thread.interrupted());
            System.out.println("t1线程是否中断?"+Thread.interrupted());
        } 
    }
});
t1.start();
```

### 1.6.6.setUncaughtExceptionHandler()

当线程发生未捕获异常时，又或者父线程想处理子线程的异常时，就可以给子线程设置一个回调处理器：java.lang.Thread.UncaughtExceptionHandler。当线程发生未经捕获的异常，JVM将调用Thread中的dispatchUncaughtException方法把异常传递给线程的未捕获异常处理器：

```java
Thread childThread = new Thread(()->{
    //子线程抛出未捕获异常
throw new NullPointerException("空指针exception");
}, "子线程");
//父线程可以设置一个回调处理器, 在子线程发生异常(未捕获), JVM就会自动回调它, 并传入两个参数
//发生异常的子线程和发生的异常信息..这样父线程就可以控制子线程的未捕获异常.
childThread.setUncaughtExceptionHandler((t, e)->{
    System.out.println(t.getName());
    System.out.println("抛出异常：" + e.getMessage());
});
childThread.start();
```

