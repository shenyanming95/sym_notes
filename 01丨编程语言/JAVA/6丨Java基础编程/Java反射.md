# 1.Type体系

在JDK1.5以后，Java增加了泛型，从而引入了类型(Type)的概念；而反射在JDK1.0就有了，所以Class先于Type出现，但是在JDK1.5以后，Class实现了Type接口。了解反射之前，还是先理清楚Java的Type体系！！！

## 1.1.Type

Type接口，全类名为：java.lang.reflect.Type，是整个Java体系所有类型的共同父接口，所有类型指：

1. 原始类型(raw_type)

2. 基本类型(primitive_type)

3. 参数化类型(parameterized_type)

4. 数组类型(array_type)

5. 类型变量(type_variables)

每一种类型都有一个继承自Type的接口，分别是：

1. 原始类型+基本类型：对应Class，包括数组、接口、注解、枚举；
2. 参数化类型：对应ParameterizedType，即集合中的泛型，如：List\<T>；
3. 数组类型：对应GenericArrayType，即带有泛型的数组，如List\<T>[]、T[]；
4. 类型变量：对应TypeVariable，如参数化类型中的E、K等类型变量

**PS**：Type还有一个子接口WildcardType，它表示通配符表达式(或泛型表达式)，如<? super T>,[? extends T]... 但它并不是Java类型中的一种！

## 1.2.ParameterizedType

ParameterizedType，参数化类型，即常说的**带有泛型的变量**都属于参数化类型。例如下面的两组成员变量：

```java
/**
 * 属于 ParameterizedType
 */
Map<String, Student> map;
Map.Entry<String, String> entry;
Set<String> set;
Class<?> clz;
List<String> list;


/**
 * 不属于ParameterizedType (虽然都是集合, 但是下面集合不带泛型, 所以不属于参数化类型)
 */
String string;
Integer i;
Set aSet;
List aList;
```

当一个类的成员变量属于参数化类型时，就可以通过Field获取到ParameterizedType实例，进而获取它泛型的具体Class类型：

```java
// 获取上面定义的变量: List<String> list;
Field field = aClass.getDeclaredField("list");

// 获取变量的类类型(Class实例), 返回：interface java.util.List
Class<?> fieldClass = field.getType();

// 获取变量的参数化类型(ParameterizedType实例), 返回：java.util.List<java.lang.String>
Type fieldType = field.getGenericType();
```

ParameterizedType只有三个方法，分别为：

| **方法名**                      | **作用**                                                     | **例子**                                                     |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Type[] getActualTypeArguments() | 获取此ParameterizedType的泛型的实际类型数组.(变量的泛型的实际类型)；如果泛型是嵌套的，需要一直调用此方法. | List\<String>  list1;  <br />List<List\<String>>  list2;  <br />两个ParameterizedType类型的变量：list1返回Class实例； <br /> list2返回ParameterizedType实例. |
| Type getRawType();              | 获取此ParameterizedType  的具体类型(变量自己的实际类型)      | List\<String>  list1;  List<List\<String>>  list2;  <br />两个ParameterizedType类型的变量返回的都是Class实例，结果都是：interface  java.util.List |
| Type getOwnerType();            | 获取当前ParameterizedType 所在的类的 Type;如果类是顶级类型返回null.实际上这个方法大部分情况下返回null | Map<String,String>返回null;  Map.entry<String,String>返回  Map.class对象. |

## 1.3.GenericArrayType

GenericArrayType，数组化类型，即泛型数组，简单地说就是组成数组的元素中有泛型，例如下面两组变量：

```java
/**
 * 属于 GenericArrayType
 */
T[] tArray;
List<String>[] listArray;
Map<Object, Object>[] mapArray;


/**
 * 不属于 GenericArrayType
 */
List<String> list;
String[] stringArray;
LabelEntity[] ints;
```

当一个类的成员变量属于数组化类型时，就可以通过Field获取到GenericArrayType实例：

```java
// 获取上面定义的变量: Map<Object, Object>[] mapArray;
Field field = aClass.getDeclaredField("mapArray ");

// 获取变量的类类型(Class实例), 返回：class [Ljava.util.Map;  
Class<?> fieldClass = field.getType();

// 获取字段的数组化类型(GenericArrayType实例), 
// 返回：java.util.Map<java.lang.Object, java.lang.Object>[]
Type fieldType = field.getGenericType();
```

GenericArrayType自身只有一个方法：

| **方法名**                     | **作用**                                                     | **例子**                                                     |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Type getGenericComponentType() | 获取数组实际组件元素的类型Type,一般为TypeVariable和  ParameterizedType | K[] kArray;  <br />List\<String>[]  listAarray; <br />上面这两种变量分别返回：  TypeVariable实例和  ParameterizedType实例 |

## 1.4.TypeVariable

TypeVariable,类型变量，可以表示任何类，例如[参数化类型](#1.2.ParameterizedType)中的E、K等类型变量,例如下面两组变量：

```java
@Data
class TypeVariableBean<K extends SubClassEntity, V> {
  /**
   * 属于TypeVariable
   */
  K key;   // K 的上边界指：SubClassEntity
  V value; // V 没有指定, 则它的上边界指：Object

  
  /**
   * 不属于TypeVariable
   */
  V[] values;
  String string;
  List<K> kList;
}
```

当一个类的成员变量属于类型变量时，就可以通过Field获取到typeVariable实例:

```java
// 获取上面定义的变量: K key;
Field field = aClass.getDeclaredField("key");

// 获取变量的类类型(Class实例), 返回：class com.sym.domain.SubClassEntity
Class<?> fieldClass = field.getType();

// 获取变量的类型变量(typeVariable实例), 返回：K
Type fieldType = field.getGenericType();
```

TypeVaribale自身共有4个方法(JDK1.8)，分别是：

| **方法名**              | **作用**                                         | **例子**                                                     |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| getBounds()             | 获取当前类型变量TypeVariable的上边界类型Type     | Map<K extends  InputStream,V>  类型变量K和V,调用getBounds()  K返回InputStream.class  V返回Object.class |
| getGenericDeclaration() | 获取当前类型变量TypeVariable所在类的类型Type     | class TypeBean\<K  extends Entity>  类型变量K调用此方法就会返回  类类型TypeBean.class |
| getName()               | 获取当前类型变量TypeVariable在源代码中定义的名称 | AList<Tt,Oo>,Tt和Oo都是类型变量，调用getName()方法时，分别返回“Tt”和“Oo” |
| getAnnotatedBounds();   | 1.8出来的，暂时不知道用处。                      |                                                              |

## 1.5.WildcardType

WildcardType，通配符(泛型)表达式，它并不是Java的一种具体类型，而是表示泛型表达式如：

Class<?>，<? super T>，<? extends T>

```java
class WildcardTypeBean {

  // 上边界为Object, 下边界为SubClassEntity(表达式只规定为SubClassEntity的父类, 
  // 而Object是所有对象的父类, 所以上限就到Object)
  List<? super SubClassEntity> list;

  // 上边界为SubClassEntity, 没有下边界(表达式规定只能为SubClassEntity的子类, 
  // 所以上限就只能为SubClassEntity, 而子类可以无限扩展, 所以无下限)
  TypeBeanEntity<? extends SubClassEntity> entity;
}
```

当一个变量的泛型被指定为表达式,才属于WildcardType,所以得先获取到它的参数化类型ParameterizedType，然后才能获取到它的实际泛型类型：

```java
// 获取上面定义的变量list
Field field = aClass.getDeclaredField("list");

// 先将其强转为 ParameterizedType 类型
ParameterizedType parameterizedType = (ParameterizedType) field.getGenericType();

// 再从 ParameterizedType 类型中获取泛型表达式类型
Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
WildcardType wildcardType = (WildcardType) actualTypeArguments[0];
```

WildcardType只有2个方法，分别是：

```java
// 得到上边界 Type 的数组
Type[] upperBounds = wildcardType.getUpperBounds();

// 得到下边界 Type 的数组
Type[] lowerBounds = wildcardType.getLowerBounds();
```

其中的上下边界，指的是泛型表达式能使用的最顶级实体类

# 2.Java反射

## 2.1.何为反射？

JAVA反射机制是在**运行状态**中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取信息以及动态调用对象的方法的功能称为java语言的反射机制。学习java反射机制需要牢记一点：**在面向对象的世界里，万事万物都是对象；类也是对象，是java.lang.Class类的对象**。

| 类名        | 用途                                             |
| ----------- | ------------------------------------------------ |
| Class       | 代表类的实体，在允许的Java应用程序中表示类和接口 |
| Field       | 代表类的成员变量（成员变量也称为类的属性）       |
| Method      | 代表类的方法                                     |
| Constructor | 代表类的构造方法                                 |

## 2.2.反射的用途

①在运行时判断任意一个对象所属的类；

②在运行时构造任意一个类的对象；

③在运行时判断任意一个类所具有的成员变量和方法；

④在运行时调用任意一个对象的方法。

## 2.3.获取Class实例

每一个类都是java.lang.Class类的实例对象，这个实例对象可以叫做类类型，有3种方式可以获取Class对象：

1. 通过Class类forName()方法，方法参数为类的全路径，例如：

```java
try {
  Class personClass = 	Class.forName("com.sym.reflect.entity.Person");
} catch (ClassNotFoundException e) {
  e.printStackTrace();
}
```

2. 通过类的隐藏属性

```java
Class personClass = Person.class;
```

3. 通过类的实例对象的getClass()方法

```java
Person person = new Person();
Class personClass = person.getClass();
```

# 3.反射的类型

## 3.1.方法的反射

万事万物都是对象，方法也是对象，是java.lang.reflect.Method类的对象。通过类的Class对象，可以获取到Method对象，每个Method对象对应类中的一个方法。

```java
Class c = Person.class;
// 获取类自己声明的和继承过来的所有public方法,返回Method数组
Method[] methods = c.getMethods();
// 获取类自己声明的和继承过来的指定方法名和参数类型的单个public方法
Method method = c.getMethod("add", int.class,double.class);
// 获取类自己声明的所有方法(所有的权限),返回Method数组
Method[] method = c.getDeclaredMethods();
// 获取类自己声明的指定方法名和参数类型的方法(所有权限)
Method method = c.getDeclaredMethod("add", int.class,double.class);
```

通过Method对象可以获取类里面方法的信息，例如：方法名、方法参数、方法返回值、方法抛出异常...等等。Method类中提供的方法：

| **方法名**                 | **返回值** | **作用**                                                     |
| :------------------------- | ---------- | ------------------------------------------------------------ |
| getDeclaringClass()        | Class      | 此  Method 对象表示的方法的类或接口的 Class 对象             |
| getExceptionTypes()        | Class[]    | 此 Method 对象表示的方法抛出的异常类的Class对象              |
| getGenericExceptionTypes() | Type[]     | 此  Method 对象表示的方法抛出的异常的Type类型                |
| getParameterTypes()        | Class[]    | 此 Method 对象所表示的方法的形参的Class对象                  |
| getGenericParameterTypes() | Type[]     | 此  Method 对象所表示的方法的形参的  Type类型                |
| getReturnType()            | Class[]    | 此 Method 对象所表示的方法的返回值的Class对象                |
| getGenericReturnType()     | Type       | 此Method对象所表示的方法的返回值的Type类型                   |
| getModifiers()             | int        | 以整数形式返回此 Method 对象所表示方法的 Java 语言修饰符。  default -128；public-129；  private-130；protected-132 |
| getName()                  | String     | 此Method对象表示的方法的名称                                 |
| invoke()                   | Object     | 对带有指定参数的指定对象调用由此 Method 对象表示的底层方法   |
| isBridge()                 | boolean    | 此方法是  bridge (桥接)方法，则返回 true；否则，返回 false   |
| isSynthetic()              | boolean    | 此方法为复合方法，则返回 true；否则，返回 false              |
| isVarArgs()                | boolean    | 此方法声明为带有可变数量的参数，则返回 true；否则，返回 false |
| toGenericString()          | String     | 返回描述此 Method 的字符串，包括类型参数                     |

## 3.2.成员变量的反射

万事万物都是对象，成员变量也是对象，是java.lang.reflect.Field类的对象。通过类的Class对象，可以获取到Field对象，每个Field对象对应类中的一个成员变量。

```java
// 获取类中访问修饰符为public的所有成员变量(包括继承的和自己声明)
Field[] fields = c.getFields();
// 获取类中继承的和自己声明的public类型的指定变量名的成员变量
Field field = c.getField("name");
// 获取类自己声明的所有成员变量(所有权限)
Field[] fields = c.getDeclaredFields();
// 获取类自己声明的指定变量名的成员变量(所有权限)
Field field = c.getDeclaredFields("name");
```

通过Field对象可以获取变量的信息，获取或者修改变量的值。 Field类提供的方法有：

| **方法名**                      | **返回值**   | **作用**                                                     |
| ------------------------------- | ------------ | ------------------------------------------------------------ |
| get(Object obj)                 | Object       | 返回指定对象上此 Field 表示的成员变量的值。get()有扩展方法，例如：  getChar(Object  obj)是将值转换成字节类型...依次类推还有boolean、int、  long、byte、double、float |
| getDeclaredAnnotations()        | Annotation[] | 返回此成员变量上的所有注解                                   |
| getDeclaringClass()             | Class        | 此 Field  对象表示的成员变量所在的类或接口的Class对象        |
| getType()                       | Class        | 此 Field 对象所表示成员变量的Class对象                       |
| getGenericType()                | Type         | 此 Field  对象所表示成员变量的声明类型                       |
| getModifiers()                  | int          | 以整数形式返回由此 Field 对象表示的字段的 Java 语言修饰符:  default-0；public-1；private-2；  protected-4 |
| getName()                       | String       | 返回此Field对象表示的字段的名称                              |
| isEnumConstant()                | boolean      | 如果此字段表示枚举类型的元素，则返回 true；否则返回 false    |
| isSynthetic()                   | boolean      | 如果此字段是复合字段，则返回 true；否则返回 false            |
| set(Object obj, Object   value) | void         | 将指定对象变量上此 Field 对象表示的成员变量设置为指定的新值。set()有扩展方法，例如：setChar(Object obj,byte b)是将指定对象的指定变量设置一个char类型的新值。依次类推还有boolean、int、long、byte..等 |
| toGenericString()               | String       | 返回一个描述此  Field（包括其一般类型）的字符串              |

## 3.3.构造方法的反射

万事万物皆对象，构造方法也是对象，是Constructor类的对象。通过Class对象可以获取到Constructor对象，每个Constructor对象对应类中的一个构造方法。

```java
Class c = LabelBean.class;
// 获取类自己声明为public的所有构造方法,返回Constructor数组
Constructor[] constructors = c.getConstructors();
// 获取类自己声明为public的指定构造方法,通过构造方法的形参来决定
Constructor constructor = c.getConstructor(int.class, char.class);
// 获取类自己声明的所有的构造方法(不论权限),返回Constructor数组
Constructor[] constructors = c.getDeclaredConstructors();
// 获取类自己声明的指定构造方法(不论权限),返回Constructor对象
Constructor constructor = c.getDeclaredConstructor(int.class, char.class);
```

Constructor类提供的方法为：

| **方法名**                      | **返回值**                        | **作用**                                                     |
| ------------------------------- | --------------------------------- | ------------------------------------------------------------ |
| getDeclaredAnnotations()        | Annotation[]                      | 获取位于此构造方法上的所有注解                               |
| getDeclaringClass()             | Class                             | 此 Constructor 对象表示的构造方法所在的类或接口的Class对象   |
| getExceptionTypes()             | Class[]                           | 此 Constructor 对象表示的构造方法抛出的所有异常类的Class对象 |
| getGenericExceptionTypes()      | Type[]                            | 此 Constructor 对象表示的构造方法抛出的所有异常类的Type类型  |
| getGenericParameterTypes()      | Type[]                            | 此 Constructor 对象所表示的构造方法的所有形参的Type类型      |
| getParameterTypes()             | Class[]                           | 此 Constructor 对象所表示的构造方法的所有形参的Class对象     |
| getTypeParameters()             | TypeVariable  <Constructor\<T>>[] | 按照声明顺序返回一组 TypeVariable 对象，这些对象表示通过此 GenericDeclaration 对象所表示的一般声明来声明的类型变量。 |
| isSynthetic()                   | boolean                           | 如果此构造方法是一个复合构造方法，则返回 true；否则返回 false。 |
| isVarArgs()                     | boolean                           | 如果声明此构造方法可以带可变数量的参数，则返回 true；否则返回 false |
| newInstance(Object... initargs) | T                                 | 使用此 Constructor 对象表示的构造方法来创建该构造方法的声明类的新实例，并用指定的初始化参数初始化该实例 |
| toGenericString()               | String                            | 返回描述此 Constructor 的字符串，其中包括类型参数            |

## 3.4.java修饰符

 Field、Method、Constructor类都有一个getModifiers()方法，用来解析获取Java修饰符，返回一个整数，该整数可以用Modifier类的toString()方法来解析，返回访问修饰符的字符串，方法源码为：

```java
public static String toString(int mod) {
    StringBuilder sb = new StringBuilder();
    int len;
    if ((mod & PUBLIC) != 0)        sb.append("public ");
    if ((mod & PROTECTED) != 0)     sb.append("protected ");
    if ((mod & PRIVATE) != 0)       sb.append("private ");
    /* Canonical order */
    if ((mod & ABSTRACT) != 0)      sb.append("abstract ");
    if ((mod & STATIC) != 0)        sb.append("static ");
    if ((mod & FINAL) != 0)         sb.append("final ");
    if ((mod & TRANSIENT) != 0)     sb.append("transient ");
    if ((mod & VOLATILE) != 0)      sb.append("volatile ");
    if ((mod & SYNCHRONIZED) != 0)  sb.append("synchronized ");
    if ((mod & NATIVE) != 0)        sb.append("native ");
    if ((mod & STRICT) != 0)        sb.append("strictfp ");
    if ((mod & INTERFACE) != 0)     sb.append("interface ");
    if ((len = sb.length()) > 0)    /* trim trailing space */
        return sb.toString().substring(0, len-1);
    return "";
}
```

# 4.反射的操作

## 4.1.获取类实例

通过newInstance()方法可以实例化一个类，但是newInstance（）是调用无参构造方法来创建实例,所以要求类中存在无参构造方法

```java
public void instanceTest(){
   Class c = Student.class;
   try {
      Student s1 = (Student)c.newInstance();
      Student s2 = (Student)c.newInstance();
      System.out.println(s1==s2);//结果为false，说明是两个对象
   } catch (InstantiationException e) {
      e.printStackTrace();
   } catch (IllegalAccessException e) {
      e.printStackTrace();
   }
}
```

## 4.2.成员变量反射操作

可以用Field类的对象来获取一个类中的成员变量，Field.get()方法可以获取成员变量的值；Field.set()方法可以设置成员变量的值。当然，处理private的成员变量，需要先setAccessible(true)

```java
Student s = new Student();
Class c = s.getClass();
//给定成员变量名称就可以获取单个的成员变量
Field f = c.getDeclaredField("name");
//若为private类型,要先执行下面这行代码
f.setAccessible(true);
Object o = f.get(s);
System.out.println("反射操作前,name="+o);
//通过set方法可以改变成员变量的值
f.set(s,"我在运行中被修改的");
System.out.println("反射操作后,name="+s.getName());
```

注意：

1. Field不仅有get()，还有getInt、getString、getDouble...可直接取对 应类型的成员变量的值

2. AccessibleObject.setAccessible(array, flag)可以为整个成员变量数组取消private的安全访问限制，同样，适用于Method[]、Constructor[]

  例如：

```java
    Field[] fs = c.getDeclaredFields();
    AccessibleObject.setAccessible(fs, true);
```

## 4.3.方法反射操作

唯一确定一个方法: 1、方法名；2、方法参数列表

方法的反射操作： Method对象有个invoke方法：method.invoke()

 利用反射机制调用方法的步骤： 

1. 获取类类型

2. 由类类型获取方法对象Method

3. 调用method.invoke()方法

```java
try {
  Class c = LabelBean.class;
  // 根据方法名和方法参数，获取Method对象，如果方法参数为不定参数，可以传递数组的Class对象
  Method method1 = c.getDeclaredMethod("count", int[].class);
  // 如果方法没有参数，就不用传参
  Method method2 = c.getDeclaredMethod("getLabelId");
  // 创建LabelBean实例对象来调用方法
  LabelBean labelBean = new LabelBean(1,"标签");
  // invoke()方法需要指定调用方法的对象和方法所需的参数
  int[] array = {1,2,3};
  Object result1 = method1.invoke(labelBean, array);
  Object result2 = method2.invoke(labelBean);
  // invoke()返回方法执行后的返回值，若方法无返回值，invoke()返回null
  System.out.println(result1);
  System.out.println(result2);
} catch (NoSuchMethodException e) {
  e.printStackTrace();
} catch (IllegalAccessException e) {
  e.printStackTrace();
} catch (InvocationTargetException e) {
  e.printStackTrace();
}
```

## 4.4.数组的反射操作

数组反射操作，明显区别于其它类型的反射。java.lang.reflect.Array类提供了动态创建和访问Java数组的方法，该类里面的方法都是static和native的。

```java
int[] array = {10,20,30,40};

// 获取数组的Class对象
Class c = array.getClass();
// 或者这样获取：int[].class;
Class c1 = int[].class;

System.out.println("是否数组："+c.isArray()); //执行结果：true
System.out.println("什么类型的数组："+c.getComponentType()); // 执行结果：int
System.out.println("是否为基本类型："+c.isPrimitive()); // 执行结果：false
System.out.println("数组长度："+ Array.getLength(array)); // 执行结果：4

// 获取数组指定下标的元素
Object ret = Array.get(array, 1);
System.out.println(ret); // 执行结果：20

// 实例化指定类型、指定长度的数组
Object obj = Array.newInstance(String.class, 7);
String[] strArray = (String[])obj;

// 为数组赋值
Array.set(strArray,0,"中");
Array.set(strArray,1,"化");
Array.set(strArray,2,"人");
Array.set(strArray,3,"民");
Array.set(strArray,4,"共");
Array.set(strArray,5,"和");
Array.set(strArray,6,"国");

Arrays.stream(strArray).forEach(System.out::print); // 执行结果：中化人民共和国
```

Array类提供的重要方法：

1. getLength()运行时获取数组的长度

2. get()运行时获取数组指定的元素

3. set()运行时修改数组中的指定元素

4. newInstance()实例化一个指定类型、指定长度的数组

## 4.5.反射与集合泛型

集合的泛型仅在编译时起作用，到运行时泛型就不存在了；而反射都是在运行时起作用的，因此可以用反射绕过集合的泛型

```java
// 创建泛型为string的List
List<String> stringList = new ArrayList<>();
// 此时如果要给 stringList 添加整型数据，会报错
// stringList.add(20);

// 通过反射来越过编译时的泛型校验
Class c = stringList.getClass();

try {
  // 获取list.add()方法对应的Method对象
  Method method  = c.getDeclaredMethod("add",Object.class);

  // 添加数据
  method.invoke( stringList,10 );
  method.invoke( stringList,20 );

  // 查看是否添加成功
  System.out.println("集合长度："+ stringList.size());
  // 注意：此时遍历集合，不能使用foreach，会抛出 java.lang.ClassCastException 异常
  // 只能通过for循环
  for( int i=0,len=stringList.size();i<len;i++ ){
    Object obj = stringList.get(i);
    System.out.println(obj.getClass()); // 执行结果：class java.lang.Integer
  }
} catch (NoSuchMethodException e) {
  e.printStackTrace();
} catch (IllegalAccessException e) {
  e.printStackTrace();
} catch (InvocationTargetException e) {
  e.printStackTrace();
}
```

java中的泛型是在编译期间有效的,在运行期间将会被删除,也就是所有泛型参数类型在编译后都会被清除掉。