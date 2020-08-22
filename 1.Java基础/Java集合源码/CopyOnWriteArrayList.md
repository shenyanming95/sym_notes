java.util.concurrent.CopyOnWriteArrayList是在JDK1.5的时候发布的，它可以理解是ArrayList的扩展，或者说是它的线程安全解决方案。CopyOnWriteArrayList，简称COW，采用了一种`读写分离`思想，将List的`读操作`和`写操作`划分开来，以达到并发读写的功能。

不过，CopyOnWriteArrayList适合读多写少的并发场景，如果业务写的逻辑多，使用它反而效率不高，毕竟它需要拷贝数据

# 1.ArrayList非线程安全

首先需要知道为啥ArrayList是非线程安全的？

## 1.1.遍历时操作

ArrayList在执行遍历操作时，如果同时添加或者删除元素，就会抛出`ConcurrentModificationException`

```java
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9));
for (Integer i : list) {
  list.add(10);
}
```

这是因为ArrayList继承自AbtractList，在AbtractList中定义了一个属性`modCount`：

```java
protected transient int modCount = 0;
```

它会在调用诸如add()、delete()等方法时被递增1，如下面所示：

```java
private void ensureExplicitCapacity(int minCapacity) {
  modCount++;

  // overflow-conscious code
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```

然后当在遍历ArrayList的时候，它的迭代器实现如下：

```java
private class Itr implements Iterator<E> {
  
  int cursor;       // index of next element to return
  int lastRet = -1; // index of last element returned; -1 if no such
  int expectedModCount = modCount;
  
  public E next() {
    	// 其中第一步, 就是判断 expectedModCount 和 modCount 是否相等, 如果此时调用了add(), 它会将
    	// modCount++，导致 expectedModCount 与 modCount 就不相等, 从而抛出异常
      checkForComodification();
      int i = cursor;
      if (i >= size)
        throw new NoSuchElementException();
      Object[] elementData = ArrayList.this.elementData;
      if (i >= elementData.length)
        throw new ConcurrentModificationException();
      cursor = i + 1;
      return (E) elementData[lastRet = i];
  }
}
```

## 1.2.并发操作

ArrayList添加元素时，如果遇到容量不足时，会采取扩容的方式增加容量，而它实现扩容的方式是将数据拷贝一份新的，代码如下：

```java
public boolean add(E e) {
  // 判断是否需要扩容
  ensureCapacityInternal(size + 1);
  elementData[size++] = e;
  return true;
}


// 比较大小以判断是否需要扩容
private static int calculateCapacity(Object[] elementData, int minCapacity) {
  if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    return Math.max(DEFAULT_CAPACITY, minCapacity);
  }
  return minCapacity;
}


private void ensureExplicitCapacity(int minCapacity) {
  modCount++;
  // 如果需要扩容, 通过Arrays.copyOf()扩容数组
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```

如果此时ArrayList的容量已达到临界值，比如capacity=100，size=99，线程t1和t2，一起执行add()操作，恰巧判断结果都是不需要扩容。t1先将元素添加成功了，并且更新了size（size=100）。此时t2开始执行，当它执行到`elementData[size++] = e;`这行代码时，由于对象数组只有100个，即下标最大为99，t2此时塞入了一个下标为100的元素，就会抛出`java.lang.ArrayIndexOutOfBoundsException`

还有`elementData[size++] = e;`并不是原子操作，它为两步：`elementData[size] = e;`和`size++`，当t1设置完新值后还没来得及将size++，t2也执行了这行代码，那么t1设置的元素就会被覆盖，最终导致元素变少了

# 2.CopyOnWriteArrayList源码

CopyOnWriteArrayList的成员变量如下，其实相比于ArrayList多出了一个ReentrantLock，这是为了在写入的时候加锁使用

```java
public class CopyOnWriteArrayList<E>                                       
  implements List<E>, RandomAccess, Cloneable, java.io.Serializable {    
  private static final long serialVersionUID = 8673264195747942595L;     

  // 全局锁, 用于在修改元素操作加锁                            
  final transient ReentrantLock lock = new ReentrantLock();              

  // 注意这边是对对象数组Object[], 添加了关键字 volatile，它虽然可以保证内存可见性，但这是对这个引用而言，
  // 数组内的元素怎么变化，是没有办法保证其内容的可见性
  private transient volatile Object[] array;   
}
```

## 2.1.add()

```java
public boolean add(E e) {    
  	// 加锁
    final ReentrantLock lock = this.lock;                                  
    lock.lock();                                                           
    try {
      // 获取旧的对象数组
      Object[] elements = getArray();                                    
      int len = elements.length;
      // 拷贝新的对象数组
      Object[] newElements = Arrays.copyOf(elements, len + 1);
      // 设置新元素
      newElements[len] = e;
      // 将对象数组的引用指向新拷贝的数组
      setArray(newElements);                                             
      return true;                                                       
    } finally {                                                            
      lock.unlock();                                                     
    }                                                                      
}                                                                          
```

add()的原理很简单，就是加锁，拷贝数组，最后将对象数组引用指向新数组。这样在遍历时操作元素就不会再抛出`ConcurrentModificationException`异常，因此每次读取的都是旧的数组。

## 2.2.remove()

```java
public boolean remove(Object o) {                                         
  Object[] snapshot = getArray();                                       
  int index = indexOf(o, snapshot, 0, snapshot.length);                 
  return (index < 0) ? false : remove(o, snapshot, index);              
}

// 真正执行删除逻辑的方法
private boolean remove(Object o, Object[] snapshot, int index) {                 
  final ReentrantLock lock = this.lock;                                        
  lock.lock();                                                                 
  try {                                                                        
    Object[] current = getArray();                                           
    int len = current.length;
    // 这边是为了找到元素所在下标index
    if (snapshot != current) findIndex: {                                    
      int prefix = Math.min(index, len);                                   
      for (int i = 0; i < prefix; i++) {                                   
        if (current[i] != snapshot[i] && eq(o, current[i])) {            
          index = i;                                                   
          break findIndex;                                             
        }                                                                
      }                                                                    
      if (index >= len)                                                    
        return false;                                                    
      if (current[index] == o)                                             
        break findIndex;                                                 
      index = indexOf(o, current, index, len);                             
      if (index < 0)                                                       
        return false;                                                    
    }
    // 新数组的元素比旧数组少一
    Object[] newElements = new Object[len - 1];
    // 先复制新数组
    System.arraycopy(current, 0, newElements, 0, index);
    // 然后新数组index后面的元素全部向前移动一位, 这一步很优雅, 避免了在for循环中一个一个移动
    System.arraycopy(current, index + 1, newElements, index, len - index - 1);                                       
    setArray(newElements);                                                   
    return true;                                                             
  } finally {                                                                  
    lock.unlock();                                                           
  }                                                                            
}                                                                                
```

remove()操作也很简单，其实跟add()大同小异，都是拷贝新数组，在新数组中执行删除操作，完事后再将对象数组的引用指向新数组

## 2.3.Iterator()

CopyOnWriteList的Iterator()方法返回的是COWIterator实现：

```java
static final class COWIterator<E> implements ListIterator<E> {
  private final Object[] snapshot;  
  private int cursor;

  public boolean hasNext() {                   
    return cursor < snapshot.length;         
  } 

  // 这边会有个隐患, 如果游标cursor判断代码已经跑过了, 符合cursor < snapshot.length, 如果此时另一个线
  // 程执行了删除操作, 把对象数组的容量变小了, 此时如果执行snapshot[cursor++], 就会抛出数组越界异常
  public E next() {                          
    if (!hasNext())                       
      throw new NoSuchElementException();
    return (E) snapshot[cursor++];         
  }
	
  // java8的CopyOnWriteList的迭代器是不支持删除操作的, 要小心
  public void remove() {                        
    throw new UnsupportedOperationException();
  }                                             
}     
```

# 3.不足之处

CopyOnWriteList凡是涉及修改元素的方法，诸如add()、remove()、set()、clear()、addAll()和removeAll()...都会申请锁，保证并发修改只有一个。但是在获取元素的方法诸如：get()，都是不需要加锁，而且读取的是旧的数组，所以CopyOnWriteList有两个不足的地方：

- 由于每次修改元素的时候，都需要拷贝一份旧数组的数据，如果对象数组存储的是大对象，很有可能会造成频繁的Full GC
- 由于它是类似读写分离操作，所以只能保证弱一致性，因为每次读取的数据都是旧数组，不能保证每次读取的数据都是新增加或新删除的，即不能保证数据的实时一致性。

所以，CopyOnWriteArrayList适合**读多写少**的情况