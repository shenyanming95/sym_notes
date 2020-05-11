mybatis源码分析只针对核心流程和核心接口，类似的xml解析过程和动态SQL拼接组装过程忽略~！

# 1.SqlSession执行流程

单独使用mybatis的API执行一句SQL的过程：

1. SqlSessionFactoryBuilder.build()创建出SqlSessionFactory对象；

2. SqlSessionFactory.openSession()开启一个新的SqlSession对象；

3. 通过SqlSession对象就可以执行select、insert、update...等操作；

4. 最后通过sqlSession的commit()或rallback()就可以提交或回滚事务。

## 1.1.解析xml

通过SqlSessionFactoryBuilder.build()传入mybatis配置文件sqlMapConfig.xml的输入流，就可以解析出一个SqlSessionFactory。这边不会对xml解析过程分析，主要看mybatis将配置解析到哪个对象上，不用猜也知道是org.apache.ibatis.session.Configuration，它就是全局配置类。

```java
//源码：XMLConfigBuilder -- 102行
private void parseConfiguration(XNode root) {
  // 依次将sqlMapConfig.xml的配置解析出来
  try {
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfs(settings);
    loadCustomLogImpl(settings);
    // 解析别名, 跳转链接
    typeAliasesElement(root.evalNode("typeAliases"));
    // 解析插件, 跳转链接
    pluginElement(root.evalNode("plugins"));
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    // 解析类型处理器, 跳转链接
    typeHandlerElement(root.evalNode("typeHandlers"));
    // 解析mapper配置文件, 跳转连接
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException();
  }
}
```

### 1.1.1.解析别名

先看下mybatis的别名注册器，xml配置文件对别名的设置都会保存到这里：

```java
public class TypeAliasRegistry {
  // 所有的别名信息都会放置到这个Map上
  private final Map<String, Class<?>> typeAliases = new HashMap<>();
}
```

解析别名的过程：

```java
//源码：XMLConfigBuilder -- 159行
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果配置的是package, mybatis就会获取配置的类路径, 将它交予TypeAliasRegistry
      // 来注册. TypeAliasRegistry会获取指定包下的所有Class, 判断它是否有注解Alias,
      // 有的话取注解Alias的值, 没有则调用Class.getSimpleName()取类名
      if ("package".equals(child.getName())) {
        String typeAliasPackage = child.getStringAttribute("name");
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
      } else {
        // 如果是单个别名配置, 取alias属性和type属性, 其中alias就是用户配置的别名,
        // type就是类名. mybatis在这里解析, 如果alias不为空, 则取用户配置的别名;
        // 否则就按照类名来获取.
        String alias = child.getStringAttribute("alias");
        String type = child.getStringAttribute("type");
        try {
          Class<?> clazz = Resources.classForName(type);
          if (alias == null) {
            typeAliasRegistry.registerAlias(clazz);
          } else {
            typeAliasRegistry.registerAlias(alias, clazz);
          }
        } catch (ClassNotFoundException e) {
          throw new BuilderException();
        }
      }
    }
  }
}
```

### 1.1.2.解析插件

先看下mybatis提供的插件链，对\<plugin>标签解析的插件类都会保存到这：

```java
public class InterceptorChain {
  // 很简单, 就是把所有插件放置在集合中
  private final List<Interceptor> interceptors = new ArrayList<>();
}
```

解析插件的过程：

```java
//源码：XMLConfigBuilder -- 183行
private void pluginElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 解析<plugin>的interceptor属性, 获取到插件的全类名
      String interceptor = child.getStringAttribute("interceptor");
      // 获取<plugin>旗下的属性配置, 如果有的话
      Properties properties = child.getChildrenAsProperties();
      // 通过反射获取到插件的实例
      Interceptor interceptorInstance = (Interceptor) 
        resolveClass(interceptor).getDeclaredConstructor().newInstance();
      // 为插件实例设置属性
      interceptorInstance.setProperties(properties);
      // 将其添加到全局配置类Configuration的插件链InterceptorChain上
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
```

### 1.1.3.解析类型处理器

mybatis提供了类型注册器TypeHandlerRegistry，它可以注册类型处理器：实际上就是将JdbcType和TypeHandler关联起来，等在解析参数时，对不同的JdbcType选择对应的TypeHandler来解析参数

```java
public final class TypeHandlerRegistry {
  private final Map<JdbcType, TypeHandler<?>>  jdbcTypeHandlerMap = 
    new EnumMap<>(JdbcType.class);
  private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = 
    new ConcurrentHashMap<>();
  private final TypeHandler<Object> unknownTypeHandler = new UnknownTypeHandler(this);
  private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>();
  private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = 
    Collections.emptyMap();
  private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;
}
```

解析类型处理器的过程：

```java
//源码：XMLConfigBuilder -- 333行
private void typeHandlerElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果用户配置了<package>标签, 则解析它指定的包路径, 获取到该包旗下所有的
      // Class类型, 如果带有MappedTypes注解, 就按照注解配置来注册, 否则取它的原生实例
      if ("package".equals(child.getName())) {
        String typeHandlerPackage = child.getStringAttribute("name");
        typeHandlerRegistry.register(typeHandlerPackage);
      } else {
        // 如果用户一一指定了, 则分别取javaType、jdbcType 和 handler, 获取到它们的
        // Class对象, 然后注册到TypeHandlerRegistry中
        String javaTypeName = child.getStringAttribute("javaType");
        String jdbcTypeName = child.getStringAttribute("jdbcType");
        String handlerTypeName = child.getStringAttribute("handler");
        Class<?> javaTypeClass = resolveClass(javaTypeName);
        JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
        Class<?> typeHandlerClass = resolveClass(handlerTypeName);
        if (javaTypeClass != null) {
          if (jdbcType == null) {
            typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
          } else {
            typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
          }
        } else {
          typeHandlerRegistry.register(typeHandlerClass);
        }
      }
    }
  }
}
```

### 1.1.4.解析mapper

首先看下mybatis提供的mapper注册器，它将Class与MapperProxyFactory关联起来，源码：

```java
public class MapperRegistry {
  private final Configuration config;
  // Mapper接口就是保存在这里
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
}
```

解析Mapper的过程：

```java
//源码：XMLConfigBuilder -- 360行
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 如果是<package>子标签, 会加载该包下的所有Class(只获取接口), 然后靠它创建出
      // MapperProxyFactory对象, 一起保存到MapperRegistry的knownMappers属性中
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        configuration.addMappers(mapperPackage);
      } else {
        // 如果是<mapper>子标签, 取出它的三个属性.
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        if (resource != null && url == null && mapperClass == null) {
          // 如果配置的是resource属性, 即配置了*mapper.xml, 则根据它指定的路径生成一个
          // XMLMapperBuilder去解析
          ErrorContext.instance().resource(resource);
          InputStream inputStream = Resources.getResourceAsStream(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream,
                                                               configuration, resource, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url != null && mapperClass == null) {
          // 如果配置的url属性, 生成XMLMapperBuilder去解析. 这种方式比较少
          ErrorContext.instance().resource(url);
          InputStream inputStream = Resources.getUrlAsStream(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, 
                                                               configuration, url, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url == null && mapperClass != null) {
          // 如果配置的是class属性, 直接将其添加到MapperRegistry中
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          throw new BuilderException();
        }
```

## 1.2.获取SqlSession

通过对sqlMapConfig.xml的解析，mybatis会把xml的配置映射成一个org.apache.ibatis.session.Configuration对象，然后new一个org.apache.ibatis.session.defaults.DefaultSqlSessionFactory。源码：

```java
//源码：SqlSessionFactoryBuilder -- 91行
public SqlSessionFactory build(Configuration config) {
  // 参数config就是解析xml获取到的对象, 通过它来实例化DefaultSqlSessionFactory,
  // 这步执行完, 我们就可以获取到一个SqlSessionFactory实例
  return new DefaultSqlSessionFactory(config);
}
```

### 1.2.1.openSession()

经过之前的分析，mybatis默认会使用DefaultSqlSessionFactory作为SqlSessionFactory接口的实现，通过它的openSession()就可以获取一个SqlSession，源码：

```java
//源码：DefaultSqlSessionFactory -- 46行
public SqlSession openSession() {
  // 实际调用openSessionFromDataSource()方法创建, 这边的ExecutorType为
  // ExecutorType.SIMPLE；
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), 
                                   null, false);
}
```

### 1.2.2.openSessionFromDataSource()

```java
//源码：DefaultSqlSessionFactory -- 90行
private SqlSession openSessionFromDataSource(ExecutorType execType, 
                                             TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    // 获取在sqlMapConfig.xml配置的Environment信息
    final Environment environment = configuration.getEnvironment();
    // 通过Environment.getTransactionFactory()获取事务工厂, 默认为
    // JdbcTransactionFactory；最后通过事务工厂new一个事务, 默认为JdbcTransaction
    final TransactionFactory transactionFactory = 
      getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, 
                                           autoCommit);
    // 通过全局配置对象Configuration创建出Executor实例, 转看newExecutor()方法
    final Executor executor = configuration.newExecutor(tx, execType);
    // 最后new一个DefaultSqlSession实例
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    // 如果出现异常, 将事务关闭
    closeTransaction(tx); 
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

### 1.2.3.nextExecutor()

```java
//源码：Configuration -- 601行
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  // ExecutorType为空时至少有个默认值, 即ExecutorType.SIMPLE
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    // 如果批量操作, 则创建BatchExecutor对象, this指的是Configuration
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    // 如果是重用类型, 则创建ReuseExecutor对象, this指的是Configuration
    executor = new ReuseExecutor(this, transaction);
  } else {
    // 其它情况都只创建SimpleExecutor对象, this指的是Configuration
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    // cacheEnabled是Configuration的成员变量, 表示mybatis一级缓存, 默认开启, 即true
    // 这边采用了装饰者模式, 用CachingExecutor去装饰前一步创建的SimpleExecutor
    executor = new CachingExecutor(executor);
  }
  // 用插件Interceptor去装饰执行器后返回, 很多插件例如PageHelp就会在这里返回执行器的代理
  // 对象, 进而改变调用流程
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

## 1.3.调用SqlSession

当获取到SqlSession，就可以通过它的API请求数据库，SqlSession有两个查询接口：selectOne()和selectList()，但其实selectOne()内部就是在调用selectList()，将结果值取1而已。

### 1.3.1.selectList()

```java
public <E> List<E> selectList(String statement) {
  // 原生mybatis使用, 这个statement就是mapper.xml的命名空间+sql标签的Id. 如果是注解
  // 则是类全名 + 方法名, 然后调用下面的重载方法
  return this.selectList(statement, null);
}
public <E> List<E> selectList(String statement, Object parameter) {
  // 这边就多了一个分页组件RowBounds的创建, 继续调用下面的重载方法
  return this.selectList(statement, parameter, RowBounds.DEFAULT);
}
//源码：DefaultSqlSession -- 144行
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    // 通过Configuration的成员变量mappedStatements(类型为StrictMap)获取对应的
    // MappedStatement对象(每一份SQL, mybatis都会定义一个MappedStatement)
    MappedStatement ms = configuration.getMappedStatement(statement);
    // wrapCollection()包裹类型是集合或数组的参数, mybatis会重新创建一个Map去包裹参数值;
    // 如果参数非集合非数组, 参数值就直接返回. 然后通过上一步创建的Executor来执行SQL.
    // 这边还会传入一个Executor.NO_RESULT_HANDLER, 默认为null
    return executor.query(ms, wrapCollection(parameter), rowBounds, 
                          Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException();
  } finally {
    ErrorContext.instance().reset();
  }
}
```

### 1.3.2.CachingExecutor#query()

如果我们没设置拦截器去修饰Executor，默认会使用CachingExecutor来执行，源码为：

```java
//源码：CachingExecutor -- 80行
public <E> List<E> query(MappedStatement ms, Object parameterObject, 
                         RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  // 从MappedStatement中获取绑定的SQL语句, 为其附上参数值, 转到getBoundSql()
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  // 通过MappedStatement的id①、分页RowBounds的值②、待执行的sql③、执行sql的参数值④
  // 和当前环境Environment的id⑤. 将这些值保存到CacheKey对象内并且计算它们hash值总和.
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  // 调用当前类的重载方法, 转到query()
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

#### 1.3.2.1.getBoundSql()

```java
//源码：MappedStatement -- 296行
public BoundSql getBoundSql(Object parameterObject) {
  // 通过当前MappedStatement的SqlSource获取BoundSql对象, 内部就是new一个BoundSql
  // 对象实例, 将属性值赋值进去而已.这个BoundSql里面保存了我们写的SQL语句、参数映射
  // ParameterMapping和实际参数值parameterObject
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  // 如果没有参数映射(即SQL没有查询条件), 创建一个没有参数映射的BoundSql
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), 
                            parameterMap.getParameterMappings(), parameterObject);
  }
  // 这边检查当前的SQL标签, 是否有配置嵌套的ResultMap
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) hasNestedResultMaps |= rm.hasNestedResultMaps();
    }
  }
  return boundSql;
}
```

#### 1.3.2.2.query()

上一个[query()](#1.3.2.CachingExecutor#query)其实就是为了获取sql对象BoundSql和构建缓存键CacheKey，然后会具体调用到这个query()方法上，源码为：

```java
//源码：CachingExecutor -- 93行
public <E> List<E> query(MappedStatement ms, Object parameterObject, 
                         RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql){
  // 从MappedStatement中获取缓存Cache, 这个缓存就是Mybatis的二级缓存
  Cache cache = ms.getCache();
  if (cache != null) {
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      // 若二级缓存Cache存在当前缓存键CacheKey, 则直接取值;
      // 若没有则调用CachingExecutor的包装的执行器即BaseExecutor来查询..
      // 这个tcm就是TransactionalCacheManager, 里面管理着多个缓存Cache
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, 
                              key, boundSql);
        tcm.putObject(cache, key, list);
      }
      return list;
    }
  }
  // 如果mapper.xml本身就没有开启二级缓存, 则直接查询, 调用包装的Executor来查询
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

### 1.3.3.BaseExecutor#query()

CacheExecutor对BaseExecutor做了一层包装（这是mybatis在构建缓存模块时用到的设计模式：装饰者模式），当缓存没有发现CacheKey指定的查询时就会调用包装的Exectuor来查询.之前分析过，每个SqlSession创建时都会创建新的Executor([详见这里](#_newExecutor()))，这意味着Executor内的成员变量是每个SqlSession独立拥有的，看下面的源码：

```java
//源码：BaseExecutor -- 141行
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, 
                         ResultHandler resultHandler, CacheKey key, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 如果 queryStack 为零, 并且要求清空本地缓存, 例如：<select flushCache="true">
  // 那么就把一级缓存清空.
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    // queryStack是BaseExector的一个成员变量, 未加任何修饰, 默认值为0
    // 提醒：每个SqlSession有各自的Executor, 不同SqlSession互不影响.
    queryStack++;
    // localCache是BaseExecutor的成员变量, 类型为：PerpetualCache. 它实际上就是mybatis
    // 的一级缓存, sqlSession级别. 如果resultHandler为空, 先从一级缓存中取值, 否则为null
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      // 一级缓存中取到值, 是处理存储过程的情况, 这里忽略掉.
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 一级缓存未取到值, 查询数据库, 转到queryFromDatabase()
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    // 查询栈减1
    queryStack--;
  }
  if (queryStack == 0) {
    // 遍历所有DeferredLoad, 执行延迟加载
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // 清空队列
    deferredLoads.clear();

    // 如果缓存级别为SQL语句级别, 则清空本地缓存.
    // 缓存级别只有两种：SESSION(会话级)、STATEMENT(SQL语句级别)
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      clearLocalCache();
    }
  }
  return list;
}
```

#### 1.3.3.1.queryFromDatabase()

当一级缓存没有查询结果时，Executor就会发起数据库查询，源码为：

```java
//源码：BaseExecutor -- 320行
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, 
                                      RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql){
  List<E> list;
  // 先在缓存中添加此缓存键CacheKey, 这一步是与延迟加载有关的
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    // 真正执行查询, 有没有发现这些框架都有一个do..()的方法, 表示真正执行的意思...
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    // 将上一步的占位对象EXECUTION_PLACEHOLDER清空掉.
    localCache.removeObject(key);
  }
  // 将查询结果保存到一级缓存中
  localCache.putObject(key, list);
  // 暂时忽略，存储过程相关
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

### 1.3.4.SimpleExecutor#doQuery()

BaseExecutor的doQuery()方法是protected修饰的，由子类去实现。前面分析获取SqlSession知道，在[创建Executor](#1.2.3.nextExecutor())时，默认都以SimpleExecutor实现。源码为：

```java
//源码：SimpleExecutor -- 57行
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds,
                           ResultHandler resultHandler, BoundSql boundSql) {
  // java.sql.Statement对象
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    // 通过mybatis全局配置对象Configuration, 创建StatementHandler.实际上由它来执行SQL
    // 每次查询都会new一个StatementHandler, 创建流程看这里
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, 
                                                                 parameter,rowBounds, resultHandler, boundSql);
    // 真正创建java.sql.Statement实例, 并为其赋值参数, 具体构造流程看这里.
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 到此一个完整的java.sql.Statement就已经创建好了, 可以调用StatementHandler.query()
    // 方法查询数据库了, 转到StatementHandler.query().
    return handler.query(stmt, resultHandler);
  } finally {
    // 查询完会关闭数据资源java.sql.Statement, 即执行：statement.close();
    // 通过JDBC知识, 关闭一个资源, 只会关掉以它做底层的上级资源. 所以statement关掉
    // 不会关闭数据库链接Connection, 但会关闭查询结果ResultSets
    closeStatement(stmt);
  }
}
```

#### 1.3.4.1.newStatementHandler()

```java
//源码：Configuration -- 591行
public StatementHandler newStatementHandler(Executor executor, MappedStatement 
                                            mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler 
                                            resultHandler, BoundSql boundSql) {
  // 调用RoutingStatementHandler的构造方法, 它会按照MappedStatement类型, 创建不同的
  // StatementHandler实例, 然后将其包裹起来.源码在下方..
  StatementHandler statementHandler = new RoutingStatementHandler(executor, 
                                                                  mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  // 用配置的插件修改这个StatementHandler
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```

RoutingStatementHandler用到的也是装饰者模式，它内部定义了一个private final StatementHandler delegate;用来引用实际具体的处理器：

```java
//源码：RoutingStatementHandler -- 39行
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter,
                               RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  // 根据StatementType, 创建不同的StatementHandler. StatementType总共有3种：
  // STATEMENT, PREPARED, CALLABLE, 对应JDBC的Statement的不同操作；mybatis提供
  // 的这些StatementHandler实现都继承自BaseStatementHandler(源码在下面)
  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds,
                                            resultHandler, boundSql);
      break;
    case PREPARED:
      // 因为这里的SQL用了“#{}”, 而mybatis会将这种表达式解析为“?”, 用预编译的
      // 方式执行sql, 所以mybatis会创建PreparedStatementHandler.其实不管在创建
      // 哪一个StatementHandler的具体实现, 都会调用抽象父类BaseStatementHandler的
      // 构造方法, 里面会创建2个重要的对象：ParameterHandler和ResultSetHandler
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds,
                                              resultHandler, boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds,
                                              resultHandler, boundSql);
      break;
    default:
      throw new ExecutorException();
  }
}
```

这边有一个很重要的知识点，作为StatementHandler接口的抽象实现BaseStatementHandler是mybatis所有handler的父类，它里面会定义一些组件，还有调用拦截器链生成两个重要的代理对象：

```java
//源码：BaseStatementHandler -- 53行
protected BaseStatementHandler(Executor executor, MappedStatement 
                               mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  // 一堆组件赋值
  this.configuration = mappedStatement.getConfiguration();
  this.executor = executor;
  this.mappedStatement = mappedStatement;
  this.rowBounds = rowBounds;
  this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
  this.objectFactory = configuration.getObjectFactory();
  if (boundSql == null) {
    // 若boungSql为空, 先创建
    generateKeys(parameterObject);
    boundSql = mappedStatement.getBoundSql(parameterObject);
  }
  this.boundSql = boundSql;
  // 创建ParameterHandler, 并且用拦截器去渲染它, 获取代理对象
  this.parameterHandler = configuration.newParameterHandler(mappedStatement, 
                                                            parameterObject, boundSql);
  // 创建ResultSetHandler, 并且用拦截器去渲染它, 获取代理对象
  this.resultSetHandler = configuration.newResultSetHandler(executor,
                                                            mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}
```

#### 1.3.4.2.prepareStatement()

真正创建java.sql.Statement实例, 并为其赋值参数:

```java
//源码：SimpleExecutor -- 84行
private Statement prepareStatement(StatementHandler handler, Log statementLog){
  Statement stmt;
  // 调用抽象父类BaseExecutor的Transaction对象的getConnection()获取数据库链接
  Connection connection = getConnection(statementLog);
  // 通过StatementHandler.prepare()方法创建Statement实例, 并为它设置查询超时时间。
  // 前面分析知道, 此时的StatementHandler为PreparedStatementHandler实现类, 它就会通过
  // 数据库链接connection的prepareStatement()方法创建出PreparedStatement实例, 这个就是
  // 原生jdbc的代码。而PreparedStatement实例由数据库驱动(如mysql-driver)提供..
  stmt = handler.prepare(connection, transaction.getTimeout());
  // 为创建好的Statement设置参数值, 例如 PrepareStatement 对象上的占位符.这里面会依次
  // 遍历BoundSql的ParameterMapping集合, 取到它们对应的参数值, 通过jdbc的
  // PreparedStatement的各种setxx()方法赋值, 注意下标从1开始...其中赋值操作有交由
  // mybatis的组件TypeHandler来完成, 每种不同的JdbcType都会有一个与其对应的TypeHandler
  // (这是因为jdbc的PreparedStatement太坑爹, 它的setxxx()方法需要区分不同类型的参数)
  // 所以mybatis才会对每一种参数创建一个TypeHandler..我猜是这样子
  handler.parameterize(stmt);
  return stmt;
}
```

### 1.3.5.StatementHandler#query()

到代码执行到这里就非常简单了，就是将一切准备就绪的Statement拿去执行然后处理它的返回结果ResultSet就行。源码：

```java
//源码：PreparedStatementHandler -- 62行
public <E> List<E> query(Statement statement, ResultHandler resultHandler){
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.handleResultSets(ps);
}
```

## 1.4.解析ResultSet

通过调用了SqlSession，里面进行了一大堆xxxHandler和数据库链接的创建，这些前奏最终都是为了创建JDBC的Statement实例，然后通过execute()方法查询数据库。接着交给ResultSetHandler组件，让它来解析ResultSet.默认实现类为DefaultResultSetHandler！

### 1.4.1.handleResultSets()

分析到这里... DefaultResultSetHandler - 181行，从这里开始分析

# 2.mapper代理

使用代理mapper查询一次SQL语句，执行流程为：

1. SqlSessionFactoryBuilder.build()创建出SqlSessionFactory对象；

2. SqlSessionFactory.openSession()开启一个新的SqlSession对象；

3. 定义mapper接口和相应的mapper.xml配置文件；

4. 通过SqlSessionFactory.getMapper()获取接口的代理类；

5. 调用接口中定义的方法执行SQL语句；

## 2.1.解析xml + 获取SqlSession

这两步跟之前的逻辑一样，先[解析](#1.1.解析xml)出SqlSessionFactory，再[创建](#1.2.获取SqlSession)一个SqlSession.

## 2.2.获取Mapper代理类

### 2.2.1.getMapper

通过SqlSession#getMapper(Class)方法可以获取一个mapper接口的代理类：

```java
//源码：DefaultSqlSession --290行
public <T> T getMapper(Class<T> type) {
  // this表示当前SqlSession实例, 即DefaultSqlSession
  return configuration.getMapper(type, this);
}
```

它实际是通过Configuration#getMapper(Class)方法获取：

```java
//源码：Configuration --778行
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

Configuration底层又是交给MapperRegistry去创建Mapper代理，先看MapperRegistry的成员变量，会有一个map来保存已处理的Mapper接口：

```java
public class MapperRegistry {
  private final Configuration config;
  // 这部分会在解析xml中就已知晓, 所以mybatis这里用的是hashMap而不是concurrentHashMap
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
}
```

最后分析MapperRegistry#getMapper()方法：

```java
//源码：MapperRegistry --44行
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 先从缓存中knownMappers获取是否有这个Class类型的Mapper代理工厂
  final MapperProxyFactory<T> mapperProxyFactory = 
    (MapperProxyFactory<T>) knownMappers.get(type);
  // 如果格式定义正确, 在xml解析中就可以知道MapperProxyFactory; 如果配置失败或者压根就
  // 没有这个Mapper接口, 则抛出异常BindingException
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    // 调用mapperProxyFactory的newInstance()方法, 传入SqlSession对象
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

### 2.2.2.newInstance()

MapperRegistry会为每个Mapper接口注册着相应的MapperProxyFactory(在解析xml就已经确定)，底层会获取该接口相对应的MapperProxyFactory对象去创建Mapper代理。先看Mapper代理工厂的成员变量：

```java
public class MapperProxyFactory<T> {
  // 相对应的Mapper接口的Class类型
  private final Class<T> mapperInterface;
  // Mapper接口内部的方法缓存, 提高反射效率吧. 它和后面的MapperProxy的方法缓存是同一对象
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();
}
```

通过它的newInstance()方法来创建Mapper代理：

```java
//源码：MapperProxyFactory -- 50行
public T newInstance(SqlSession sqlSession) {
  // 创建MapperProxy实例, 会传入SqlSession、Mapper接口Class对象和该接口的方法缓存;
  // MapperProxy实现了java.lang.reflect.InvocationHandler, 就是JDK反射的处理接口.
  // 所以也可以猜到对Mapper接口方法的调用, 最后会转到MapperProxy的invoke()方法上.
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, 
                                                       methodCache);
  // 调用重载的newInstance(MapperProxy)方法
  return newInstance(mapperProxy);
}
```

newInstance(SqlSession)会调用其重载方法newInstance(MapperProxy)：

```java
//源码：MapperProxyFactory -- 46行
protected T newInstance(MapperProxy<T> mapperProxy) {
  // 方法很简单, 就是JDK的动态代理API调用, 需要注意的是InvocationHandler就是上面提及的
  // MapperProxy对象, 到这步时Mapper接口代理已经创建完成.
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), 
                                    new Class[] { mapperInterface }, mapperProxy);
}
```

## 2.3.调用Mapper代理类

获取完Mapper代理类后，就可以直接调用接口中定义的方法，经过[上面](#2.2.2.newInstance())的分析也知道JDK动态代理会将方法的调用转发到MapperProxy上。

### 2.3.1.invoke()

MapperProxy实现了InvocationHandler接口，它的invoke()方法就是Mapper接口代理类的实际执行方法：

```java
//源码：MapperProxy -- 78行
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    // 因为JDK动态代理还会处理Object类的方法, 所以这边遇到这种情况, 直接回调原方法
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
      // JDK8的接口可以携带default关键字, 然后给出默认实现, 这边就是作出这样的判断. 一般
      // Mapper接口是不会有默认实现的..
    } else if (method.isDefault()) {
      if (privateLookupInMethod == null) {
        // 对JDK8的适配
        return invokeDefaultMethodJava8(proxy, method, args);
      } else {
        // 对JDK9的适配
        return invokeDefaultMethodJava9(proxy, method, args);
      }
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  // 如果是Mapper接口的普通方法调用, 代码就会执行到这里. 先获取MapperMethod, 源码在下面
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  // 调用MapperMethod的execute()方法, 传入方法参数
  return mapperMethod.execute(sqlSession, args);
}
```

每一个Method都会有一个MapperMethod与其对应，而且mybatis会将其缓存起来，方法如下：

```java
//源码：MapperProxy -- 96行
private MapperMethod cachedMapperMethod(Method method) {
  // methodCache是MapperProxy的成员变量, 它是通过MapperProxy的构造方法赋值的, 而
  // 构造方法是通过MapperProxyFactory调用的, 所以这个methodCache是MapperProxyFactory
  // 传入的, 用的就是它自身的成员变量, 类型为：ConcurrentHashMap. 这里就是对并发做控制了
  // 缓存没有就创建一个新的MapperMethod, 有就直接返回.
  return methodCache.computeIfAbsent(method,
                                     k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
}
```

MapperMethod是mybatis为Mapper接口中每一个方法创建的执行器，它的成员变量有：

```java
public class MapperMethod {
  // SqlCommand是MapperMethod定义的内部静态类
  private final SqlCommand command;

  // MethodSignature也是MapperMethod定义的内部静态类, 用来计算出方法的一些特性
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
}
```

### 2.3.2.execute()

真正执行SQL语句的还是细粒化到每个方法之上，实际就是通过MapperMethod来执行的，调用它的execute()方法，传入SqlSession对象和JDK动态代理携带过来的方法参数：

```java
//源码：MapperMethod -- 57行
public Object execute(SqlSession sqlSession, Object[] args) {
  // SQL执行结果
  Object result;
  // command是MapperMethod的成员变量, 它用来定位出SQL属于哪种类型- SqlCommandType.
  // 该枚举类共有六种：UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH
  switch (command.getType()) {
    case INSERT: {
      // 如果是新增, 先转换参数, 实际再通过SqlSession.insert()方法执行
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      // 如果是修改, 先转换参数, 实际再通过SqlSession.update()方法执行
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      // 如果是删除, 先转换参数, 实际再通过SqlSession.delete()方法执行
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
      // 上面代码可以看出来, 实际上也是Mapper代理开发也是调用SqlSession提供的API执行SQL.
      // 如果是查询语句, 会根据方法返回结果的不同做处理
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        // 如果方法返回值为void并且有指定的ResultHandler处理, 调用这个方法
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        // 要是方法返回的是集合类型, 调用executeForMany()方法
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        // 要是方法返回的是Map类型, 调用executeForMap()方法
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        // 要是方法返回的是游标Cursor类型, 调用executeForCursor()方法
        result = executeForCursor(sqlSession, args);
      } else {
        // 其它情况下, 例如返回的单体Bean, 那么久直接调用SqlSession.selectOne()方法
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        // JDK8的适配, 处理返回类型为：java.util.Optional
        if (method.returnsOptional()
            && (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      // 如果SQL是清空类型, 则清空所有Statement缓存
      result = sqlSession.flushStatements();
      break;
    default:
      // 未知类型抛异常
      throw new BindingException("Unknown execution method for: " + command.getName());
  }----到这里整个switch()结束
    // 方法定义返回值不为null, 但是执行结果后却为null, 就抛出异常
    if (result == null && method.getReturnType().isPrimitive() && 
        !method.returnsVoid()) {
      throw new BindingException("");
    }
  // 到这里, 整个SQL执行流程就完了, 实际上Mapper代理开发, mybatis也是调用SqlSession
  // 去执行SQL的。
  return result;
}
```

#### 2.3.2.1.executeWithResultHandler()

...

#### 2.3.2.2.executeForMany()

...

#### 2.3.2.3.executeForMap()

...

#### 2.3.2.4.executeForCursor()

...