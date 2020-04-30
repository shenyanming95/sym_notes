# 1.静态代理

静态代理指的是代理和被代理对象在代码执行之前是确定的，它们都实现相同的接口或继承相同的抽象类，可分为：继承和聚合。常见的例如需要为方法的执行添加日志，原接口和原实现类方法如下：

```java
// 静态代理的原始接口
public interface MoveAble {
    void move();
}
```

```java
// 静态代理的原始实现类
public class Car implements MoveAble {
   @Override
   public void move() {
      long begin = System.currentTimeMillis();
      System.out.println("汽车开始行驶...");
      Thread.sleep(new Random().nextInt(1000));
      long end = System.currentTimeMillis();
      System.out.println("汽车结束行驶...用时："+(end-begin)+"毫秒");
   }
}
```

## 1.1.继承

如果使用继承的方式：继承Car类，然后重写（覆盖）move方法。添加新的功能，用super.move()去调用父类的方法，如下：

```java
public class ProxyByExtend extends Car {
   @Override
   public void move() {
      // 新功能
      System.out.println("开始记录日志...");
      // 父类原有的功能
      super.move();
      // 新功能
      System.out.println("结束记录日志..."); 
   }
}
```

## 1.2.聚合

如果使用聚合的方式：需要与Car类实现同一个接口moveAble,把Car对象注入进来，再实现move()方法，如下：

```java
public class ProxyByImpl implements MoveAble {
  //用于注入已有的功能
  private MoveAble m;
  public ProxyByImpl(MoveAble m){
     //通过构造函数注入
     this.m = m;
  }
  @Override
  public void move() {
      // 新功能
      System.out.println("开始记录日志..."); 
      // Car已有的功能
      m.move();
      // 新功能
      System.out.println("结束记录日志..."); 
   }
}
```

## 1.3.静态代理总结

①聚合：代理类聚合了被代理类，且都实现了movable接口，可以实现灵活多变；

  继承：继承不够灵活，随着功能需求增多，继承体系会非常臃肿；

  如果使用静态代理，首选**聚合**

②静态代理都要实现同一接口再增加新功能，若工程包含大量实现类，如果都需要添加功能，则易造成类爆炸。因此，最好采用动态代理方式

# 2.动态代理

代理类在程序运行时创建的代理方式称为**动态代理**。也就是说，这种情况下，代理类并不是在Java代码中定义的，而是在运行时根据我们在Java代码中的"指示"动态生成的。 Java中常用的动态代理方式有两种，其一是JDK自带的，利用反射实现，只能针对带有接口的类；其二是第三方框架CGLIB，它基于字节码增强技术ASM动态地生成被代理类的子类字节码，通过加载子类实现代理，既适用于接口又适用于普通非final的类(以及非final的方法)

## 2.1.JDK

JDK的动态代理有局限性，它要求代理类和被代理类要实现同一组接口，这样才可以对接口内的方法实现代理。也就是说JDK动态代理**只能对类所实现接口中定义的方法进行代理。**对于从Object中继承的方法，JDK代理会把hashCode()、equals()、toString()这三个非接口方法转发给InvocationHandler，其余的Object方法则不会转发。详见[JDK Proxy官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)。

### 2.1.1.核心API

#### 2.1.1.1.Proxy

java.lang.reflect.Proxy，是JDK动态代理机制的主类，提供一组静态方法为一组接口动态的生成代理类对象.

```java
/**
 * 方法用于为指定类加载器、一组接口及调用处理器生成动态代理类实例
 * 
 * @param loader 类加载器（即用哪个类加载器来加载这个代理类到 JVM 的方法区）.
 * @param interfaces 指定原始类的哪些接口要被代理, 此参数是一个Class数组.
 * @param h调用处理器，即实现了InvocationHandler接口的类实例(描述了代理类增强功能)
 * @return 代理类的实例对象
 */
public static Object newProxyInstance(ClassLoader loader, 
               Class<?>[] interfaces,InvocationHandler h)
```

#### 2.1.1.2.InvocationHandler

java.lang.reflect.InvocationHandler，调用处理器接口，只有一个Invoke方法，代理类的增强逻辑，就是写在Invoke()方法中，当我们调用原始类对象的方法时，这个“调用”会转送到invoke()方法中。这样一来，我们对原始类所有方法的调用都会变为对invoke的调用，便可以在invoke方法中添加统一的处理逻辑(也可以根据method参数的不同区分不同的方法调用)

```java
/**
 * @param proxy  代理类对象实例, 表示哪个代理对象调用了method方法
 * @param method 原始类将要被调用的方法
 * @param args   调用方法需要的参数
 * @return 执行该方法后的返回值
*/
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```

#### 2.1.1.3.ClassLoader

java.lang.ClassLoader：类装载器类，将类的字节码装载到 Java 虚拟机（JVM）中并为其定义类对象，然后该类才能被使用。Proxy类与普通类的唯一区别就是其字节码是由 JVM 在运行时动态生成的而非预存在于任何一个.class 文件中

### 2.1.2.使用方式

要为哪个类做代理，就需要实例该类对象，然后把它注入到调用处理器实现类，通过Proxy的newProxyInstance方法，获取到Car类的代理对象proxy,执行proxy.move()就会转到调用处理器的invoke()方法去执行，而invoke方法通过反射机制去调用实际类的方法

#### 2.1.2.1.原始类 

```java
// JDK动态代理的接口
public interface MoveAble {
   //无参无返回值
   void move();
   //带参有返回值
   String speed(String param); 
}
```

```java
// JDK动态代理的接口实现类
public class Car implements MoveAble {
  @Override
  public void move() {
     long begin = System.currentTimeMillis();
     System.out.println("汽车开始行驶...");
     Thread.sleep(new Random().nextInt(1000));
     long end = System.currentTimeMillis();
     System.out.println("汽车结束行驶...用时："+(end-begin)+"毫秒");
  }

  @Override
  public String speed(String param) {
     return "汽车加速了："+param+" km/h";
  }
}
```

#### 2.1.2.2.实现增强逻辑

实现增强逻辑，就需要实现InvocationHandler接口，如：

```java
// JDK动态代理的代理类增强逻辑
public class LogHandler implements InvocationHandler {
   // JDK动态代理是利用反射实现, 熟悉反射的同学都知道, 反射回调方法需要一个原始类对象实例.
   // 所以这个成员变量target就是保存原始类对象实例的引用.
   private Object target;
   public LogHandler(Object target){
      super();
      this.target = target;
   }
   // 增强逻辑的实现
   @Override
   public Object invoke(Object proxy, Method method, Object[] args) {
       // 新增的功能
   		 System.out.println("开始记录日志..."); 
       // 反射回调原本的功能
       Object returnParam = method.invoke(target,args); 
       //新增的功能
       System.out.println("结束记录日志..."); 
       return returnParam; 
   }
}
```

#### 2.1.2.3.生成代理对象

```java
   //要为Car类找代理
   Car car = new Car();
   LogHandler h = new LogHandler(car);
   //创建代理对象，proxy就是Car类的代理
   Class c = car.getClass();
   MoveAble proxy = (MoveAble)Proxy.newProxyInstance(c.getClassLoader(),
            c.getInterfaces(), h);
   //测试无参无返回值
   proxy.move();
   //测试有参有返回值
   String response = proxy.speed("40");
   System.out.println(response);
```

## 2.2.CGLIB

CGLIB动态代理会生成一个初始类的子类，子类重写初始类的所有不是final的方法。在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。原有类的每个方法调用都会转为调用实现了MethodInterceptor接口的proxy的intercept()方法，但是对于final方法，CGLIB无法进行代理！使用CGLIB需要引入下面的pom：

```xml
<dependency>
	<groupId>cglib</groupId>
	<artifactId>cglib</artifactId>
  <version>3.3.0</version>
</dependency>

```

### 2.2.1.原始类

```java
// 使用cglib动态代理，可以不用接口
public class Person {
   public void eat(){
      System.out.println("person正在吃饭...");
   }
   public void say(){
      System.out.println("person正在说话...");
   }
}
```

### 2.2.2.单个拦截器

#### 2.2.2.1.MethodInterceptor

在调用原始类方法时，CGLIB会回调MethodInterceptor接口方法拦截，来实现自定义的代理逻辑，类似于JDK中的[InvocationHandler](#2.1.1.2.InvocationHandler)接口。

```java
public class PersonProxy implements MethodInterceptor {
   /*
	  * 参数说明：
    *  1、obj: CGLib动态生成的代理类实例
    *  2、method: 基本类的方法，即Person类定义的方法
    *  3、params: 方法的参数
    *  4、mProxy: cglib方法代理对象
    */
	@Override
	public Object intercept(Object obj, Method method, Object[] params, 
       MethodProxy mProxy) throws Throwable {
    	// 新功能
    	System.out.println("Person在做饭....");
    	// 原始功能
    	mProxy.invokeSuper(obj, params);
    	// 新功能
    	System.out.println("Person吃完饭....");
    	return null;
	}
}
```

#### 2.2.2.2.Enhancer

Enhancer是CGLib中的一个字节码增强器，它可以方便的对你想要处理的类进行扩展，CGLIB通过Enhancer生成代理类实例对象：

```java
//Enhancer是Cglib的一个字节码增强器，用于生成代理对象
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Person.class);//设置被代理类(即基本类)
enhancer.setCallback(new PersonProxy());//设置方法拦截类
//生成代理对象, 并调用代理方法
Person person = (Person)enhancer.create();
erson.say();
```

#### 2.2.3.拦截器数组

​	拦截器数组用的是CallbackFilter，回调过滤器，可以设置对不同的方法执行不同的回调逻辑。在JDK动态代理中并没有类似的直接功能，在jdk动态代理中，代理对象无论执行哪个方法，都只会到该类的Invoke方法去执行，如果要对不同方法处理逻辑，只能用if语句去判断；而使用了CallbackFilter，当代理对象执行say()方法，用MethodInterceptor1这个拦截器实现类对象；当代理对象执行eat()方法，用MethodInterceptor2这个拦截器实现类对象....所以，需要几个业务逻辑，就要创建几个拦截器实现类。

​	我们会把所有实现MethodInterceptor接口的类实例对象，放到一个Callback数组中，CallbackFilter中的accept方法，根据不同的method返回不同的值i，这个值对应callback数组中对象的下标，相当于调用callbacks[i]这个对象。然后，创建一个实现CallBackFilter接口的实现类，实现接口中的唯一一个accept方法，判断此时调用的是哪个方法，返回值代表上面定义的Callback数组的下标，比如说，当调用eat方法时，用PersonProxy拦截类处理；当调用say方法时，用PersonProxy2拦截类处理

#### 2.2.3.1.CallbackFilter

```java
public class PersonCallBackFilter implements CallbackFilter {
  //根据方法的不同, 返回Callback[]数组的不同下标, 选择不同的方法拦截器 
  //MethodInterceptor
   @Override
   public int accept(Method method) {
      if("eat".equals(method.getName())){
          return 0;
      }else if("say".equals(method.getName())) {
         return 1;
      }
      return 0;
 	 }
}
```

#### 2.2.3.2.Enhancer

```java
// 创建拦截器数组
Callback[] callbacks = new Callback[]{new PersonProxy(),new PersonProxy2()};
// 通过Enhancer创建代理对象
Enhancer enhancer = new Enhancer();
//设置被代理的基本类
enhancer.setSuperclass(Person.class); 
//设置回调过滤器
enhancer.setCallbackFilter(new PersonCallBackFilter());
//设置回调时作选择的Callback数组
enhancer.setCallbacks(callbacks);
// 创建代理对象, 执行不同的方法就有不同的拦截器逻辑 
Person person = (Person)enhancer.create();
person.eat();
```

### 2.2.4.延迟加载

#### 2.2.4.1.LazyLoader

#### 2.2.4.2.Dispatcher

# 3.源码分析

## 3.1.JDK

https://blog.csdn.net/yhl_jxy/article/details/80586785

## 3.2.CGLIB

在使用CGLIB的动态代理功能时，添加下面的代码，可以让CGLIB动态生成的字节码文件保存到指定目录上，供分析原理使用：

```java
// 将cglib动态生成的class文件输出到指定目录下
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "F:\\cglib_class");
```

### 3.2.1.核心API

#### 3.2.1.1.FastClass

net.sf.cglib.reflect.FastClass保存了对某个类Class对象的引用。此类就是为了快速地对某个类的方法进行索引和调用(因为CGLIB觉得反射很慢)，它其实就是对某个类的方法记录，通过提供了一系列重载方法getIndex()，用来获取指定方法的索引值，然后在invoke()方法根据索引值调用类的不同方法。CGLIB在动态生成类的时候，会为原始类和代理类各自生成一个继承FastClass的类，详见[CGLIB动态生成的类](#3.2.2.CGLIB动态生成的类)。

```java
bstract public class FastClass {
	// FastClass唯一的成员变量, 保存了原始类的Class对象引用
	private Class type;
 /**
   * 3个重载方法, 用来获取指定方法对应的索引值; 此值可以在invoke()中用来区分不同的方法,
   * 然后进行方法直接调用(CGLIB这样设计是因为Java反射会损耗一定的性能)
   */
   abstract public int getIndex(String name, Class[] parameterTypes);
   abstract public int getIndex(Class[] parameterTypes);
	 abstract public int getIndex(Signature sig);
  /**
    * 通过上面提供的方法获取索引值来调用原始类方法
    * @param index  方法的索引值
    * @param obj    原始类对象
    * @param args   调用方法需要的参数
    * @return 执行该方法后的返回值
    */
    abstract public Object invoke(int index, Object obj, Object[] args);
}
```

#### 3.2.1.2.CreateInfo

net.sf.cglib.proxy.MethodProxy.CreateInfo保存了原始类和代理类的Class引用，以及一些动态类创建的取名规则和生成策略。源码如下：

```java
private static class CreateInfo {
	// 原始类的Class类型
	Class c1;
  // CGLIB动态生成的代理类Class类型
  Class c2;
  // 默认为DefaultNamingPolicy, 为动态生成的代理类取名称
  NamingPolicy namingPolicy;
  // 默认为DefaultGeneratorStrategy, 用来生成代理类的字节码
  GeneratorStrategy strategy;
  // 默认为false
  boolean attemptLoad;
}
```

#### 3.2.1.3.FastClssInfo

net.sf.cglib.proxy.MethodProxy.FastClassInfo保存了原始类和代理类的FastClass对象，而且保存了将要调用方法在原始类和代理类的索引值：

```java
private static class FastClassInfo {
	// 保存了原始类Class类型的FastClass对象, 这个对象由CGLIB动态生成
  FastClass f1;
  // 保存了生成的代理类Class类型的FastClass对象, 这个对象由CGLIB动态生成
  FastClass f2;
  // i1表示被调用方法在原始类的索引位置, 值为1
  int i1;
  // i2表示被调用方法在代理类的索引位置, 值为21
  int i2;
}
```

#### 3.2.1.4.MethodProxy

我们都知道一个方法就有一个java.lang.reflect.Method与其对应，同理在CGLIB中，一个方法就有一个MethodProxy与其对应。源码为：

```java
public class MethodProxy {
	// 原方法的签名, 名称与原方法一样, 类似：getUser()V       -- 假设getUser()是原方法
  private Signature sig1;
  // 代理方法的签名, 名称会加上前缀CGLIB$, 类似：CGLIB$getUser$1()V
  private Signature sig2;
  // 这个类定义的一个内部类, 用来保存创建动态类的取名规则和生成策略. 点击查看详情
  private CreateInfo createInfo;
  private final Object initLock = new Object();
  // 这个类定义的一个内部类, 用来保存原始类和生成的代理类的FastClass对象. 点击查看详情
  private volatile FastClassInfo fastClassInfo;
}
```

### 3.2.2.CGLIB动态生成的类

CGLIB会为每个原始类(被代理类)生成3个动态代理类，都长成这个样子：

![](.//images/CGLIB生成的类信息.png)

以上面为例(其它类取名规则类似)，从上往下：

第一个：BaseClass$$EnhancerByCGLIB$$a494ce61$$FastClassByCGLIB$$295a820b.class，它是CGLIB为代理类(即第二个)创建的实现FastClass的动态生成类；

第二个：BaseClass$$EnhancerByCGLIB$$a494ce61.class，它是CGLIB为原始类创建的继承于它的代理类的动态生成类；

第三个：BaseClass$$FastClassByCGLIB$$83fe61e6.class，它是CGLIB为原始类创建的实现了FastClass的动态生成类。

#### 3.2.2.1.原始类的FastClass动态生成类

BaseClass$$FastClassByCGLIB$$83fe61e6.class这种取名模式的类文件，是CGLIB针对于原始类(被代理类)生成的继承FastClass的动态生产类，主要用来快速地回调原始类的方法。由于它是程序生成的字节码，所以看起来很不友好，这里省略一些它的源代码：

```java
public class BaseClass$$FastClassByCGLIB$$83fe61e6 extends FastClass {
	// 因为FastClass需要保存一个Class类型的引用, 所以这边var1就是原始类的Class对象
	// 即BaseClass.class
	public BaseClass$$FastClassByCGLIB$$83fe61e6(Class var1) {
    super(var1);
	}
	// invoke()方法就是回调原始类方法, 其中var1表示方法索引; var2表示原始类的实例对象;
	// var3表示方法所需参数集. 
	public Object invoke(int var1, Object var2, Object[] var3) {
    	// 将var2强转成原始类的实例对象BaseClass
    	BaseClass var10000 = (BaseClass)var2;
    	int var10001 = var1;
    	// 通过方法索引值调用不同的方法, 注意不是反射调用, 而是实例对象调用!!!
      try {
          switch(var10001) {
          case 0:
              var10000.doRun();
              return null;
          case 1:
              var10000.doSomething();
              return null;
          case 2:
              return var10000.toJson();
          // 因为原始类BaseClass我只定义了3个方法, 所以CGLIB这边用索引0、1、2表示
          // ===================================================== //
          // 剩下3个方法为Object超类的方法，依次是：equals()、toString()、hashCode()
          case 3:
              return new Boolean(var10000.equals(var3[0]));
          case 4:
              return var10000.toString();
          case 5:
              return new Integer(var10000.hashCode());
          }
      } catch (Throwable var4) {
          // 回调方法异常
          throw new InvocationTargetException(var4);
      }
      // 索引值未匹配到方法的异常
      throw new IllegalArgumentException("Cannot find matching method/constructor");
	}
}
```

**备注：**<u>CGLIB生成原始类的FastClass动态生成类还是挺有规律的，一般原始类定义几个方法，它就会从索引0开始依次标注这些方法；最后在自动为超类Object的equals()、toString()、hashCode()这三个方法做索引；所以每个原始类的方法索引值最大为：非final方法个数 + 3.也从另一个方面说明了CGLIB只会代理Object的equals()、toString()、hashCode()这3个方法，其余的Object类方法不会代理！</u>

#### 3.2.2.2.动态生成的代理类

BaseClass$$EnhancerByCGLIB$$a494ce61.class这种取名模式的类文件，就是CGLIB创建的继承了原始类的动态生成类，它里面保存了各个像MethodInterceptor和Callback[]的引用，以及对调用方法的原始逻辑和增强逻辑。它通过Enhancer.create()方法获取到，而实际上我们在实际开发使用的都是这个类(spring-aop保存在IOC容器的也是这个类)，部分源码：

```java
public class BaseClass$$EnhancerByCGLIB$$a494ce61 
	//可以看到代理类继承了原始类BaseClass
	extends BaseClass implements Factory { 
	/** 代理类的全部属性如下 */
  private boolean CGLIB$BOUND;
  public static Object CGLIB$FACTORY_DATA;
  private boolean CGLIB$CONSTRUCTED;
  private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
  // 这个属性就是保存拦截器数组
  private static final Callback[] CGLIB$STATIC_CALLBACKS;
  // 这个属性就是保存单个拦截器
  private MethodInterceptor CGLIB$CALLBACK_0;
  private static Object CGLIB$CALLBACK_FILTER;
  // 下面的这些参数是CGLIB为原始类方法 + Object的3个基础方法
  // 生成的 java.lang.reflect.Method 和 net.sf.cglib.proxy.MethodProxy 对象
  private static final Method CGLIB$doRun$0$Method;
  private static final MethodProxy CGLIB$doRun$0$Proxy;
  private static final Object[] CGLIB$emptyArgs;
  private static final Method CGLIB$doSomething$1$Method;
  private static final MethodProxy CGLIB$doSomething$1$Proxy;
  private static final Method CGLIB$equals$2$Method;
  private static final MethodProxy CGLIB$equals$2$Proxy;
  private static final Method CGLIB$toString$3$Method;
  private static final MethodProxy CGLIB$toString$3$Proxy;
  private static final Method CGLIB$hashCode$4$Method;
  private static final MethodProxy CGLIB$hashCode$4$Proxy;
  private static final Method CGLIB$clone$5$Method;
  private static final MethodProxy CGLIB$clone$5$Proxy;
   /**
     * 如果调用的方法名为：CGLIB$..$()这种格式的, 则相当于直接调用原始类的方法
     * 可以看到方法体内调用的是父类的方法
     */
   final void CGLIB$doSomething$1() {
       super.doSomething();
   }
   /**
     * 如果调用的是原始类方法的名称, 则会判断是否可以调用 MethodInterceptor 定义的方法
     */
   public final void doSomething() {
      if (!this.CGLIB$CONSTRUCTED) {
          super.doSomething();
      } else {
          MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
          if (var10000 == null) {
              CGLIB$BIND_CALLBACKS(this);
              var10000 = this.CGLIB$CALLBACK_0;
          }
          if (var10000 != null) {
             var10000.intercept(this, CGLIB$doSomething$1$Method, CGLIB$emptyArgs,
                   CGLIB$doSomething$1$Proxy);
          } else {
              super.doSomething();
          }
      }
   }
}

```

备注：<u>CGLIB创建动态代理类时，会把用户在Enhancer设置的属性保存进去，所以我们反编译的时候可以看到一系列的成员变量(上面源码所示)；除此之外，CGLIB会为原始类的每个非final方法生成2个方法，一个是原始方法的名称，一个是带CGLIB$前缀的名称。</u>

#### 3.2.2.3.代理类的FastClass动态生成类

BaseClass$$EnhancerByCGLIB$$a494ce61$$FastClassByCGLIB$$295a820b.class这种取名模式的类文件是CGLIB针对于[代理类](#3.2.2.2.动态生成的代理类)生成的继承FastClass的动态生产类，主要用来快速地回调代理类的方法。由于它是程序生成的字节码，所以看起来很不友好，这里省略一些它的源代码：

