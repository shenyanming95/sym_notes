dubbo版本：2.7.6.

# 4.dubbo扩展机制

dubbo spi的核心类为：org.apache.dubbo.common.extension.ExtensionLoader，获取扩展类实体也是通过它，一般我们可以通过如下两行代码获取扩展类：

```java
ExtensionLoader.getExtensionLoader(LoadBalance.class).getDefaultExtension();
ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension("my-loadBalance");
```

一个是获取默认的扩展类，一个是获取指定名称的扩展类

## 4.1.getExtensionLoader()

getExtensionLoader()是为了获取一个ExtensionLoader实例，不同Class类型的扩展类，dubbo都会为其创建一个ExtensionLoader实例

```java
// 源码：ExtensionLoader - 152行
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
  // 空指针判断
  if (type == null) {
    throw new IllegalArgumentException("Extension type == null");
  }
  // 接口判断，必须要求是接口才允许使用SPI机制
  if (!type.isInterface()) {
    throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
  }
  // 判断这个接口是否有使用了@SPI注解
  if (!withExtensionAnnotation(type)) {
    throw new IllegalArgumentException("Extension type (" + type +
                                       ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
  }
  // EXTENSION_LOADERS 是ExtensionLoader的静态成员变量，用来保存创建过的扩展类加载器，它是一个
  // ConcurrentHashMap，相当于缓存。首先就先从它来获取扩展类加载器，当然刚开始都为null
  ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
  if (loader == null) {
    // 如果不存在的话，直接new一个，然后由ConcurrentHashMap保证线程安全，将其放入到缓存里
    EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
    loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
  }
  return loader;
}
```

## 4.2.new ExtensionLoader()

调用ExtensionLoader的构造方法，创建一个ExtensionLoader实例，type就是我们定义的扩展类接口

```java
private ExtensionLoader(Class<?> type) {
  // 保存Class类型
  this.type = type;
  // 若type不是ExtensionFactory类型, dubbo会先创建一个ExtensionFactory类型的扩展类加载器；若type本身就是ExtensionFactory
  // 类型, 那么就没必要创建, 直接赋值null即可。因为我们定义的接口，肯定不是ExtensionFactory类型，所以每次都会先去获取
  // ExtensionFactory的扩展类加载器，然后再将它放到ExtensionLoader的缓存EXTENSION_LOADERS中，最后调用
  // getAdaptiveExtension()获取实例，其实就是再走一遍ExtensionLoader.getExtensionLoader()逻辑，设计得很巧妙！！！
  // 最终我们定义的扩展类接口都会使用AdaptiveExtensionFactory作为扩展类实例生成工厂，赋值给成员变量objectFactory，
  // 其中objectFactory是ExtensionLoader的一个成员变量（非静态），它用来创建扩展类的实例。接口定义如下面所示
  objectFactory = (type == ExtensionFactory.class ? null : 
                   ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

ExtensionFactory接口是用来创建扩展类的实例

```java
@SPI
public interface ExtensionFactory {
    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);
}
```

### 4.2.1.getAdaptiveExtension()

getAdaptiveExtension()方法是dubbo为了获取ExtensionFactory实例调用的方法

```java
// 源码：ExtensionLoader - 570行
public T getAdaptiveExtension() {
  // cachedAdaptiveInstance是ExtensionLoader的成员变量，它是一个Holder对象，就是简单地存放创建出来的扩展类实例，以保证
  // 创建出来的扩展类实例是一个单例
  Object instance = cachedAdaptiveInstance.get();
  if (instance == null) {
    // 判断创建过程中是否存在错误，若存在则直接抛出异常
    if (createAdaptiveInstanceError != null) {
      throw new IllegalStateException("Failed to create adaptive instance: " +
                                      createAdaptiveInstanceError.toString(),
                                      createAdaptiveInstanceError);
    }
		// 加锁创建
    synchronized (cachedAdaptiveInstance) {
      instance = cachedAdaptiveInstance.get();
      if (instance == null) {
        try {
          // 调用createAdaptiveExtension()方法创建实例，dubbo默认创建：AdaptiveExtensionFactory。
          instance = createAdaptiveExtension();
          cachedAdaptiveInstance.set(instance);
        } catch (Throwable t) {
          createAdaptiveInstanceError = t;
          throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
        }
      }
    }
  }

  return (T) instance;
}
```

### 4.2.2.createAdaptiveExtension()

```java
// 源码：ExtensionLoader - 996行
private T createAdaptiveExtension() {
  try {
    	return injectExtension((T) getAdaptiveExtensionClass().newInstance());
  } catch (Exception e) {
    	throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " 
                                    + e.getMessage(), e);
  }
}
```

## 4.3.getExtension()

上面方法走完以后，ExtensionLoader的扩展类加载器缓存就会有两个实例：

org.apache.dubbo.common.extension.ExtensionFactory 和 我们自定义的扩展类接口，接着就会调用getExtension()获取扩展类接口实例：

```java
// 源码：ExtensionLoader - 411行
public T getExtension(String name) {
      // 扩展类的名称不能为空
      if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
      }
      if ("true".equals(name)) {
        return getDefaultExtension();
      }
      // 获取or创建一个Holder
      final Holder<Object> holder = getOrCreateHolder(name);
      // 获取Holder中的扩展类实例
      Object instance = holder.get();
      if (instance == null) {
        synchronized (holder) {
          instance = holder.get();
          if (instance == null) {
            // 当Holder扩展类实例中的元素为null时，调用createExtension()方法创建
            instance = createExtension(name);
            holder.set(instance);
          }
        }
      }
      return (T) instance;
 }
```

### 4.3.1.createExtension()

通过调用createExtension()方法来创建扩展类实例

```java
//源码：ExtensionLoader - 622行
private T createExtension(String name) {
  			// 调用getExtensionClasses()方法获取name指定的扩展类的实现类的Class类型
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
}
```

### 4.3.2.getExtensionClasses()

getExtensionClasses()就是加载配置文件，把它的Class类型保存起来，

```java
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                  	// 如果缓存没有，就调用loadExtensionClasses()加载
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
}
```

### 4.3.3.loadExtensionClasses()

通过加载策略，加载配置文件，获取扩展类接口的Class对象

```java
private Map<String, Class<?>> loadExtensionClasses() {
  // 检查并设置@Spi设置的默认名称
  cacheDefaultExtensionName();
  // 结果集保存
 	Map<String, Class<?>> extensionClasses = new HashMap<>();
	// dubbo有三种加载策略，分别是：
  // DubboInternalLoadingStrategy，加载路径：”META-INF/dubbo/internal/“
  // DubboLoadingStrategy，加载路径：”META-INF/dubbo/“
  // ServicesLoadingStrategy，加载路径：”META-INF/services/“
  for (LoadingStrategy strategy : strategies) {
    // 实际调用loadDirectory()发起I/O请求，读取配置文件
    loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(),
                  strategy.overridden(), strategy.excludedPackages());
    loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"),
                  strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
  }
  return extensionClasses;
}
```

### 4.3.4.loadDirectory()

真正加载配置文件，通过java的URL对象

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
     boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
  			// 根据文件目录和扩展类接口的类全名拼接文件路径
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls = null;
            ClassLoader classLoader = findClassLoader();

            // try to load from ExtensionLoader's ClassLoader first
            if (extensionLoaderClassLoaderFirst) {
                ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
                if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                    urls = extensionLoaderClassLoader.getResources(fileName);
                }
            }
						// 通过ClassLoader加载文件路径
            if (urls == null || !urls.hasMoreElements()) {
                if (classLoader != null) {
                    urls = classLoader.getResources(fileName);
                } else {
                    urls = ClassLoader.getSystemResources(fileName);
                }
            }
						// 可以获取到多个URL路径，遍历它们，然后加载到extensionClasses返回
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

### 4.3.5.loadResource()

loadResource()就是加载到配置文件，然后使用BufferedReader按行读取

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
       java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
        try {
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), 
                                                                                  StandardCharsets.UTF_8))) {
                String line;
                while ((line = reader.readLine()) != null) {
                  	// 去掉#符号
                    final int ci = line.indexOf('#');
                    if (ci >= 0) {
                        line = line.substring(0, ci);
                    }
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                          	// 截取”=“号，将名称和类全名分隔开
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                line = line.substring(i + 1).trim();
                            }
                            if (line.length() > 0 && !isExcluded(line, excludedPackages)) {
                              	// 通过Class.forName()获取扩展类的Class对象，然后调用loadClass()方法缓存起来
                                loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader),
                                          name, overridden);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException();
                            exceptions.put(line, e);
                        }
                    }
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }
```

### 4.3.6.loadClass()

缓存加载出来的Class和name

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, 
                       String name,boolean overridden) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + " is not subtype of interface.");
        }
  			// 缓存Adaptive注解
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz, overridden);
        } else if (isWrapperClass(clazz)) { // 判断是否为包装类
            cacheWrapperClass(clazz);
        } else {
          	// 一般我们的接口来到这里，缓存起来
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException();
                }
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    cacheName(clazz, n);
                    saveInExtensionClass(extensionClasses, clazz, n, overridden);
                }
            }
        }
    }
```

