# 1.初始化

## 1.1.成员变量

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
  private static final long serialVersionUID = 8683452581122892189L;
  
  // 默认的初始化大小, 即创建的数组大小
  private static final int DEFAULT_CAPACITY = 10;
  
  // 共享的空数组
  private static final Object[] EMPTY_ELEMENTDATA = {};
  
  private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
  
  // 底层存储数据的数组
  transient Object[] elementData;
  
  // 已经存储元素的大小
  private int size;
}
```

## 1.2.构造方法

```java
public ArrayList(int initialCapacity) {
  if (initialCapacity > 0) {
    // 如果设置了容量, 就按照容量创建底层数组
    this.elementData = new Object[initialCapacity];
  } else if (initialCapacity == 0) {
    // 如果等于，就复用共享数组，避免了每个集合创建一个不必要的空数组
    this.elementData = EMPTY_ELEMENTDATA;
  } else {
    // 其它情况抛异常
    throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
  }
}
```

