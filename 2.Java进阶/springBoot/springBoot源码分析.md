# 1.springBoot原理

springBoot的一些原理关键点

## 1.1.自动配置

springBoot问世的目的就是为了减少spring开发的繁琐配置，它的一个突出特点就是“自动配置，开箱即用”，不要开发者去额外创建一大堆的xml文件来整合spring与其它框架的配置，只需要一个总的配置文件即可搞定。那么，springBoot是如何完成自动配置的？

### 1.1.1.自动配置过程

springBoot的自动配置类，有一个特点就是类名都是以这种形式：**xxxAutoConfiguration**，经常可以看到类似的取名规则。所有关于自动配置的信息，全部放在下面这个jar包中：**spring-boot-autoconfigure-1.5.9.RELEASE**

#### 1.1.1.1.@Conditional注解

springBoot是如何知道什么情况需要自动配置Web环境，什么时候需要配置Redis环境...，这都是依靠@Condition以及它的衍生注解来实现的。弄懂springBoot自动配置原理第一步，就是弄懂@Condition注解。@Conditional注解是spring 4.x版本推出来的，官方解释是"只有当所有指定的条件都满足时，组件才可以注册"，主要是在创建bean时增加一系列限制条件，属于Spring的注解，而springBoot在该注解的基础上做了大量封装，因此springBoot才能根据当前环境按需注入所需的组件。**注解放在配置类上，对整个配置类起作用**；**放在方法上，只对该方法起作用。**部分注解信息如下：

2

| 注解                            | 生效条件                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| @ConditionalOnBean              | 配置了某个指定的类                                           |
| @ConditionalOnMissBean          | 没有配置特定的类                                             |
| @ConditionalOnClass             | Classpath里有指定的类                                        |
| @ConditionalOnMissClass         | Classpath里缺少指定的类                                      |
| @ConditionalOnExpression        | 给定的Spring Expression Language(SpEl)表达式计算结果为true   |
| @ConditionalOnJava              | Java的版本匹配特定值或者一个范围值                           |
| @ConditionalOnJndi              | 参数中给定的JNDI位置必须存在一个，如果没有给参数，则要有JDNI InitialContext |
| @ConditionalOnProperty          | 指定的配置属性要有一个明确的值                               |
| @ConditionalOnResource          | Classpath里有指定的资源                                      |
| @ConditionalOnWebApplication    | 这是一个web应用程序                                          |
| @ConditionalOnNotWebApplication | 这不是一个web应用程序                                        |

要想知道spirngBoot注册了哪些自动配置类，在application.yml配置debug=true即可，它会在Console中打印注册的组件和未注册的组件。	

#### 1.1.1.2.注入自动配置类

何为自动配置？讲白了就是springBoot启动时自动地向IOC容器中注入必备的组件(各种Bean)。打开启动类注解**@SpringBootApplication**的源码，看它是怎么实现自动配置的：

1. 打开@SpringBootApplication注解的内容，从里面可以发现springBoot的自动配置注解@EnableAutoConfiguration，接着查看它的内容

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
// 启用自动配置注解
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    
}
```

2. 打开@EnableAutoConfiguration注解的内容，@Import是给IOC容器导入组件用的，可以看到导入了AutoConfigurationImportSelector类

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
// 向IOC容器注册组件
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    
}
```

3. AutoConfigurationImportSelector类从字面上理解就是自动导入选择器选择的组件，它实现了ImportSelector接口：

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationMetadata autoConfigurationMetadata = 
        AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
    // 获取AutoConfigurationEntry，它就会去扫描各个自动配置类
    AutoConfigurationEntry autoConfigurationEntry = 
 		getAutoConfigurationEntry(autoConfigurationMetadata,annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

4. AutoConfigurationImportSelector的getAutoConfigurationEntry()用来加载各个配置类：

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 加载全部的自动配置类
    List<String> configurations = 
        getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = filter(configurations, autoConfigurationMetadata);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

5. 接着实际上就是调用AutoConfigurationImportSelector的getCandidateConfigurations()从spring.factories中扫描各个自动配置类，将其全类名加载出来：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 事实就是依靠SpringFactoriesLoader去加载各个jar下META-INF/spring.factories
    // 获取各个配置类的全类名，然后加载到IOC容器中
	 List<String> configurations = 	    	               
     	SpringFactoriesLoader.
     		loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),                                                                  getBeanClassLoader());
Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

#### 1.1.1.3.执行自动配置类

以HttpEncodingAutoConfiguration这个自动配置类为例(因为它最简单)介绍自动配置类是如何工作的。

1. 打开HttpEncodingAutoConfiguration这个类，可以看到它有5个注解：后3个以@Conditional开头的注解是判断注解，意思是只有当前环境满足这些判断注解的要求，HttpEncodingAutoConfiguration这个类才会生效。

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(HttpProperties.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass(CharacterEncodingFilter.class)
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {}
```

除了3个三个判断注解，剩下2个配置注解，其中@EnableConfigurationProperties注解注入了HttpEncodingProperties实体类，该实体类的属性是与配置文件绑定在一起，这也就是为什么我们修改application.yml这个默认配置，springBoot就会按照我们的配置而改变，因为它的自动配置类都会有一个或多个实体类，像HttpEncodingProperties一样，它们就是配置文件的映射。

2. 自动配置类HttpEncodingAutoConfiguration获取到实体映射类后，使用@Bean注解，将实体类中的属性封装后注入到IOC容器，这样自动配置类的功能就可以使用了。一个自动配置类会存在多个被@Bean注解标注的方法，这些方法都是给IOC容器注入组件用的。

```java
@Bean
@ConditionalOnMissingBean
public CharacterEncodingFilter characterEncodingFilter() {
    CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
    filter.setEncoding(this.properties.getCharset().name());
    filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
    filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
    return filter;
}
```

### 2.1.2.自动配置精髓

#### 2.1.2.1.总结自动配置过程

①springBoot启动会加载大量的自动配置类

②判断需要的功能在springBoot中是否有默认写好的自动配置类

③查看该自动配置类中配置了哪些组件(只要我们要用的组件有，就无需配置)

④给IOC容器中自动配置组件时，会从相应的实体映射类中获取属性，而实体映射类通过@ConfigurationProperties注解与配置文件绑定。因此，可以在application.yml自定义用户自己的配置

#### 2.1.2.2.修改默认配置

有了上面的笔记，可以知道springBoot默认配置很多组件，但是，很经常地我们需要定制符合当前需求的组件，例如：在mvc中，需要添加拦截器；默认是以Tomcat启动，如果想换成jetty启动...等等。springBoot支持3种方式来修改它的默认配置：

1. springBoot在自动配置组件时，有一个统一的模式：先判断容器中是否存在用户自己配置好的组件，若有则注入用户自己配置的，若没有才用springBoot默认的配置；但是，还有一些组件，它可以是多个，例如springMVC中的视图解析器ViewResolver，springBoot把用户配置的和自己默认配置的结合一起。

2. springBoot中有非常多的*Configurer接口可以扩展配置。当用户自己需要扩展功能时，就可以通过实现这些*Configurer类(例如扩展springmvc功能，实现WebMvcConfigurer类)，重写里面的方法来实现自己的扩展。

3. springBoot中有非常多的*Customizer接口可以定制配置。当需要额外功能时，可以实现这些*Customizer接口并将它们注入到IOC容器中。例如：

   - EmbeddedServletContainerCustomizer接口，是用来定制嵌入式servlet容器，通过实现该接口，实现customize()修改servlet容器的属性；

   - ConfigurationCustomizer接口，是用来定制springBoot与mybatis整合时对mybatis的额外配置，实现customize()增加额外配置。

**PS：**

*Configurer，是通过实现它，重写里面的方法来改变springBoot原先的配置；

*Customizer，是通过将其注入到IOC容器中来扩展springBoot原先的配置。

所以它们的使用方式不同，*Configurer一般要使用@Configuration注解表示它是一个配置类；*Customizer一般用@Bean注解，将其注入到IOC容器。

## 1.2.事件回调

springBoot的事件回调的核心接口，有4个。它们可以分为两类，

①需要配置在META-INF/spring.facories文件下：

**ApplicationContextInitializer**和**SpringApplicationRunListener**

②需要手动注册到IOC容器中：

**ApplicationRunner**和**CommandLineRunner**

### 1.2.1.ApplicationContextInitializer

ApplicationContextInitializer，即上下文初始化器，它有3个作用：

1. 在springBoot上下文刷新即refresh()方法之前被调用；

2. 在web环境中用来设置一些属性变量；

3. 通过Order接口可以进行排序，Order越小的越快执行它只有一个initialize()方法，方法参数可以获取到IOC容器。

```java
public class MyApplicationContextInitializer implements
    ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 方法参数 ConfigurableApplicationContext 就是IOC容器
        System.err.println("ApplicationContextInitializer执行了...");
    }
}
```

由于springBoot是在META-INF/spring.factories文件下扫描它的，所以我们需要将它配置在spring.factories里

```xml
# ApplicationContextInitializer
org.springframework.context.ApplicationContextInitializer=\
com.sym.listener.MyApplicationContextInitializer
```

### 1.2.2.SpringApplicationRunListener

spring本身就有事件回调机制，即接口ApplicationListener，它的范围最广，会在spring容器启动、运行、终止等期间回调相应的事件。但在springBoot额外定义了SpringApplicationRunListener，这个接口的范围较小，它负责IOC容器从创建到初始化完成的整个过程，在几个重要环节它都会被回调。

```java
public class SymApplicationRunListener implements SpringApplicationRunListener {
    /**
     * 强制要求需要有这样一个构造方法
     */
    public SymApplicationRunListener(SpringApplication application, String[] args){
    }

    @Override
    public void starting() {
        // 最开始回调的方法, SpringApplication.run()刚执行时就会被调用
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        // ConfigurableEnvironment创建后, ApplicationContext创建之前, 回调此方法
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        // ApplicationContext创建完, refresh()方法执行前, 回调此方法
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        // ApplicationContext完成加载, 在contextPrepared()回调后, refresh()方法执行前,
        // 回调此方法
    }

    @Override
    public void started(ConfigurableApplicationContext context) {
        // ApplicationContext刷新并启动后, 在CommandLineRunners和ApplicationRunner
        // 被调用前, 回调此方法
    }

    @Override
    public void running(ConfigurableApplicationContext context) {
        // springBoot所有操作都完成后, 整个run()方法结束前, 回调此方法, 
        // 不出异常情况下它最晚被回调
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        // 容器启动失败回调此方法
    }
}
```

由于springBoot是在META-INF/spring.factories文件下扫描它的，所以我们需要将它配置在spring.factories里

```tex
# SpringApplicationRunListener
org.springframework.boot.SpringApplicationRunListener=\
com.sym.listener.MySpringApplicationRunListener
```

### 1.2.3.ApplicationRunner

ApplicationRunner会在springBoot上下文创建并刷新完以后被调用，需要将它注册到容器中，还可以通过@Order指定其调用顺序，同一顺序值下会先于**CommandLineRunner**执行(因为源码中它优先加载到List集合中)。它的run()方法会封装了系统运行参数(即启动main()函数携带过来的参数)，对它们进行了解析封装为ApplicationArguments

```java
@Component
@Order(1)
public class SymApplicationRunner implements ApplicationRunner{
    @Override
    public void run(ApplicationArguments args) {
        // 方法会传入包装后的命令行参数ApplicationArguments
        Set<String> optionNames = args.getOptionNames();
        for(String optionArg : optionNames){
            List<String> optionValues = args.getOptionValues(optionArg);
            System.err.println(optionArg + "->" + optionValues);
        }
    }
```

### 1.2.4.CommandLineRunner

CommandLineRunner和**ApplicationRunner**一样，都是在上下文创建并刷新完回调，它的run()方法是直接携带main()方法运行时的参数

```java
@Component
@Order(1)
public class SymCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        if (args.length > 0) {
            System.err.println("SymCommandLineRunner：" + Arrays.toString(args));
        }
    }
}
```

# 2.springBoot启动源码

启动一个springBoot工程，需要执行SpringApplication.run()方法，它实际上干了两件事：其一创建上下文，其二启动容器。

## 2.1.创建容器

SpringApplication.run()首先实现的就是创建SpringApplication上下文，它可以指定多个Class类型，表示这些类是springBoot的启动类，会以它所在的包目录下扫描各个类文件。

```java
// 源码：SpringApplication - 267
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 默认的RsourceLoader为null
    this.resourceLoader = resourceLoader;
    // 至少需要指定一个配置类, springBoot会把它们保存到一个LinkedHashSet中
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 根据当前系统引进的jar, 判断当前上下文是否需要一个内嵌的web服务器, 返回值是
    // WebApplicationType, 它是一个枚举, 有3个类型：NONE、SERVLET、REACTIVE, 表示
    // 的含义就是：非web环境、servelt环境、响应式Webflux环境
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 获取上下文初始器ApplicationContextInitializer, 跳转相应源码
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    // 获取上下文监听器ApplicationListener, 跳转相应源码
    setListeners((Collection) getSpringFactoriesInstances(
        ApplicationListener.class));
    // 之前保存的配置类, 判断哪个类中带有main()方法, 则它就是启动类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 2.1.1.getSpringFactoriesInstances()

getSpringFactoriesInstances()方法就是从classpath:/META-INF/下的spring.factories中找到各个组件的实现类(springBoot默认的+用户自定义的)，它的源码为：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}
```

上面方法实际上又调用了它的重载方法，默认传过来要获取的组件类型，即参数type，剩下的两个参数parameterTypes和args为空。源码为：

```java
// 源码：SpringApplication - 423
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] 
            parameterTypes, Object... args) {
    // 获取类加载器, 可以手动设置ResourceLoader, 如果没有设置, 就返回默认的类加载
    ClassLoader classLoader = getClassLoader();
    // 通过SpringFactoriesLoader去解析spring.factories, 返回符合type类型的全类名
    Set<String> names = new LinkedHashSet<>(
        SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 将上一步的全类名, 实例化成对象实体集合, 跳转相应源码
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, 
                                                       classLoader, args, names);
    // 将对象实体集合进行排序后返回
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

### 2.1.2.loadFactoryNames()

loadFactoryNames()就负责去spring.facties(有多个)找寻指定Class类型的组件全类名(所以我们使用spring.facties就需要用全类名表示)

```java
// 源码：SpringFactoriesLoader - 120
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable
                                            ClassLoader classLoader) {
    // 获取Class类型的全类名, 在springBoot启动过程中, 它会指定查询两次, 分为：
    // org.springframework.context.ApplicationContextInitializer
    // org.springframework.context.ApplicationListener
    String factoryTypeName = factoryType.getName();
    // 调用loadSpringFactories()方法解析, 它会返回一个Map<String, List<String>>,
    // 然后再调用Map.getOrDefault()通过要查询的全类名, 去找一下Map中是否有相应的key
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, 
                                                         Collections.emptyList());
}
```

从上面的分析也知道，真正查找的逻辑交给loadSpringFactories()来完成：

```java
// 源码：SpringFactoriesLoader - 125
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader)
{
    // cache是静态成员变量, 类型为：Map<ClassLoader, MultiValueMap<String, String>>.
    // springBoot是按照类加载器来划分的. 它会优先从缓存中查找
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }
    // 缓存不存在, 说明还未解析, 开始解析
    try {
        // getResources()和getSystemResources()的区别? 它们加载的路径都是：
        // META-INF/spring.factories
        Enumeration<URL> urls = (classLoader != null ? 
              classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
              ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        // 在一个系统中会引入多个jar, 每个jar都可以设置spring.factories, 所以
        // 类加载器加载后会有多个URL对象, 每个对象表示不同spring.factories在不同jar
        // 的路径
        while (urls.hasMoreElements()) {
            // 获取其中一个jar下的spring.factories, 将其封装成UrlResource
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            
            // 通过URL可以获取到InputStream, 再通过Properties.load()将其转为Properties
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            
            // 这边就是解析Properties, 把同一个接口类型的放在一起, 这里面用到spring自己
            // 实现的一个组件MultiValueMap, 它底层是一个LinkHashMap, 将它的value泛型定义
            // 成List, 所以它可以保存同一份key, 多个value取值, 具体看下它源码即可。
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                // 我们在spring.factories当有多个实现类时, 就可以用逗号隔开, springBoot
                // 会自动处理
                for (String factoryImplementationName : StringUtils.
                     commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        // 等所有spring.factories都解析完以后, 将其缓存起来, 不需要每次都去加载
        cache.put(classLoader, result);
        return result;
    }catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location" + 
             "[" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

### 2.1.3.createSpringFactoriesInstances()

经过loadFactoryNames()的解析，就可以将所有spring.factories指定的全类名拿到即参数names，然后利用反射实例化。默认情况下这个方法的参数parameterTypes和args是为空的。源码为：

```java
// 源码：SpringApplication - 433
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] 
        parameterTypes,ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            // 就是利用反射去实例化接口, 有参数的话携带参数...
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = 
                instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException();
        }
    }
    return instances;
}
```

## 2.2.启动容器

创建容器过程做的两件事就是：加载两个接口：ApplicationContextInitializer和ApplicationListener。真正的大头都是在后面的run()方法，参数args就是main()方法的args参数。源码为：

```java
// 源码：SpringApplication - 298
public ConfigurableApplicationContext run(String... args) {
    // StopWatch是spring提供用于记录程序执行时间的工具类
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    
    // 配置一些关于awt的设置, 可以省略
    configureHeadlessProperty();
    
    // 与创建容器阶段获取组件一样, 从spring.factories中加载 SpringApplicationRunListener
    // 组件, 然后把它们放入到 SpringApplicationRunListeners (它保存了多个监听器)
    SpringApplicationRunListeners listeners = getRunListeners(args);
    
    // 回调监听器 SpringApplicationRunListener.starting() 方法, 跳转查看源码
    listeners.starting();
    
    try {
        // 封装命令行参数
        ApplicationArguments applicationArguments = 
            new DefaultApplicationArguments(args);
        
        // 创建环境对象, 跳转查看源码
        ConfigurableEnvironment environment = prepareEnvironment(
            listeners, applicationArguments);
        
        // 向System.setProperty()配置“spring.beaninfo.ignore”属性, 属性值从
        // environment中获取, 默认值为 true.
        configureIgnoreBeanInfo(environment);
        
        // banner打印
        Banner printedBanner = printBanner(environment);
        
        // 创建上下文, 根据当前系统引进的jar包判断该创建哪种上下文, 跳转查看源码
        context = createApplicationContext();
        
        // 通过SpringFactoriesLoader加载 SpringBootExceptionReporter 组件, 做异常报告
        exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
        
        // 准备上下文环境, 下一步就是刷新上下文
        prepareContext(context, environment, listeners, 
                       applicationArguments, printedBanner);
        
        // 刷新上下文, 这步完成基本springBoot就可以使用了
        refreshContext(context);
        
        // afterRefresh()没有实现, 是springBoot给出的扩展方法, 在上下文刷新后处理相关逻辑
        afterRefresh(context, applicationArguments);
        
        // 结束程序记录并处理日志
        stopWatch.stop();
        
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).
                logStarted(getApplicationLog(), stopWatch);
        }
        
        // 回调SpringApplicationRunListener.started()方法, 逻辑与监听器回调一样
        listeners.started(context);
        
        // 回调 ApplicationRunner 和 CommandLineRunner 
        callRunners(context, applicationArguments);
        
    } catch (Throwable ex) {
        // 如果出现异常, 回调SpringApplicationRunListener.failed()方法, 并调用
        // 最开始定义的SpringBootExceptionReporter判断是否是已报告异常, 是的话再屌
        // SpringBootExceptionHandler注册此异常
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        // 回调SpringApplicationRunListener.running()方法, 逻辑与监听器回调一样
        listeners.running(context);
    } catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    // 到此springBoot启动完成, 容器就跑起来了
    return context;
}
```

### 2.3.1.监听器回调

SpringApplicationRunListener是springBoot额外定义的一个监听器，它的作用范围比spring原生的ApplicationListener更小，仅仅作用在boot容器启动过程中，它里面的每个方法源码逻辑都一样，只不过事件不同罢了；而且springBoot还巧妙地运用了另一种方式，保证在容器启动中，也能适配ApplicationListener，这边就以starting()方法为例子分析一下。

```java
// 源码：SpringApplicationRunListeners - 45
void starting() {
    // springBoot把多个SpringApplicationRunListener实现类以集合的形式保存到
    // SpringApplicationRunListeners(注意多了个s)中, 每次回调监听器方法的时候
    // 都是遍历这个集合依次回调, 所以自定义的 SpringApplicationRunListener 就会在这里
    // 被回调. 那么springBoot是如何调用原生spring的哪些监听器和事件呢?答案就是这个类：
    // org.springframework.boot.context.event.EventPublishingRunListener
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.starting();
    }
}
```

#### 2.3.1.1.EventPublishingRunListener

EventPublishingRunListener是SpringApplicationRunListener接口的实现，所以它是一个boot的监听器。它使用了“装饰器模式”内部定义一个事件多播器SimpleApplicationEventMulticaster，每次回调boot监听器的方法时，它内部都是交由这个多播器去完成的，其类结构如下：

```java
// EventPublishingRunListener还实现了Order接口, 它的值为0, 如果自己定义的监听器想先于
// 它执行, 也可以实现Order接口, 然后赋值比0小即可.
public class EventPublishingRunListener 
    implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        // 这边就是把当前上下文 ApplicationContext 中的监听器ApplicationListener
        // (在创建容器那一步就已经加载出来) 都添加到事件多播器中
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.initialMulticaster.addApplicationListener(listener);
        }
    }

    public void starting() {
        // 以starting()方法为例, 可以看到实际上EventPublishingRunListener在执行监听器
        // 回调方法时, 都是交由事件多播器initialMulticaster去完成, 而且而且重要的是
        // 它是这里才会创建一个相应的事件ApplicationStartingEvent, 通过事件多播器发布出去.
        // 其它回调方法也是这样处理的, 只不过发布的事件不一样罢了, 跳转查看源码
        this.initialMulticaster.multicastEvent(
            new ApplicationStartingEvent(this.application, this.args));
    }

}
```

EventPublishingRunListener监听方法和事件的对应关系：

| **监听方法**        | **对应事件**                        |
| ------------------- | ----------------------------------- |
| starting            | ApplicationStartingEvent            |
| environmentPrepared | ApplicationEnvironmentPreparedEvent |
| contextPrepared     | ApplicationContextInitializedEvent  |
| contextLoaded       | ApplicationPreparedEvent            |
| started             | ApplicationStartedEvent             |
| running             | ApplicationReadyEvent               |
| failed              | ApplicationFailedEvent              |

#### 2.3.1.2.multicastEvent()

EventPublishingRunListener监听器的回调方法底层都是交由SimpleApplicationEventMulticaster去发布的，调用的就是multicastEvent()，它会根据事件和事件类型，去选择匹配的监听器执行回调，源码为：

```java
// 源码：SimpleApplicationEventMulticaster – 131行
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
    // 获取事件类型, ResolvableType是spring针对于java.lang.reflect.Type的封装, 这里
    // 特指当前事件的类型
    ResolvableType type = (eventType != null ? eventType : 
                           resolveDefaultEventType(event));
    // 如果有定义线程池, 即异步发布事件, springBoot就会用线程池去执行
    Executor executor = getTaskExecutor();
    // getApplicationListeners(event, type) 方法会获取对当前事件感兴趣的监听器.
    // 底层是通过转换成GenericApplicationListener(如果无法直接转换就采用适配器模式)
    // 再调用supportsEventType()和supportsSourceType(), 若两个方法都返回true, 则
    // 表示此监听器对该事件感兴趣, 就会将它返回. springBoot这里做了缓存处理, 具体逻辑可以
    // 看下具体代码
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        // 有了监听器集合后, 就依次遍历, 执行每个监听器的onApplicationEvent()方法;
        // 有定义执行器就异步执行, 没有就同步调用. 整个监听器回调的逻辑到此就结束了.
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        } else {
            invokeListener(listener, event);
        }
    }
}
```

### 2.3.2.创建环境对象

springBoot会调用prepareEnvironment()方法创建环境对象ConfigurableEnvironment，同时发布事件。其源码为：

```java
// 源码：SpringApplication – 339行
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners 
       listeners,ApplicationArguments applicationArguments) {
    // 获取或创建 ConfigurableEnvironment, 跳转相应源码
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    
    // 配置上一步创建好的 ConfigurableEnvironment, 会加入main()方法携带的参数
    // 配置一些ConversionService和PropertySource..
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    
    // 回调监听器SpringApplicationRunListener.environmentPrepared()方法
    listeners.environmentPrepared(environment);
    
    // 将环境对象 ConfigurableEnvironment 绑定到 SpringApplication里面
    bindToSpringApplication(environment);
    
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

#### 2.3.2.1.getOrCreateEnvironment()

```java
// 源码：SpringApplication – 451行
private ConfigurableEnvironment getOrCreateEnvironment() {
    // 如果有手动指定环境对象, 那就直接返回即可
    if (this.environment != null) {
        return this.environment;
    }
    // 如果没有指定, springBoot会根据前一步创建容器分析出来的当前环境类型, 决定初始化哪一种
    // 环境对象
    switch (this.webApplicationType) {
        case SERVLET:
            // 传统MVC的Servlet环境, 创建 StandardServletEnvironment
            return new StandardServletEnvironment();
        case REACTIVE:
            // 响应式Web环境(WebFlux), 创建 StandardReactiveWebEnvironment
            return new StandardReactiveWebEnvironment();
        default:
            // 非Web环境, 就创建普通的 StandardEnvironment
            return new StandardEnvironment();
    }
}
```

### 2.3.3.创建上下文

springBoot创建上下文的方式跟创建环境的方式差不多，都是根据当前环境的jar依赖判断要创建哪一种上下文，获取它的Class对象，然后反射实例化。普通web环境用的是AnnotationConfigServletWebServerApplicationContext，以它作为上下文实现的例子

```java
// 源码：SpringApplication - 568
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         switch (this.webApplicationType) {
         case SERVLET:
            // 普通MVC模式, 创建AnnotationConfigServletWebServerApplicationContext
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
            // 响应式模式, 创建AnnotationConfigReactiveWebServerApplicationContext
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            // 默认创建, AnnotationConfigApplicationContext
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      } catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Unable create a default ApplicationContext, please specify an" +
			"ApplicationContextClass", ex);
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

### 2.3.4.准备上下文

在刷新上下文之前，springBoot会先准备上下文，调用的是prepareContext()方法，源码为：

```java
private void prepareContext(ConfigurableApplicationContext context, 
                            ConfigurableEnvironment environment,
                            SpringApplicationRunListeners listeners, 
                            ApplicationArguments applicationArguments, 
                            Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", 
                                  applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(
            new LazyInitializationBeanFactoryPostProcessor());
    }
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```

### 2.3.5.刷新上下文