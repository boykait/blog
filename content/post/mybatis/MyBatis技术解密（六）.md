---
title: MyBatis技术解密（六）：MyBatis二级缓存
date: 2020-02-13
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

> 昨天学习了MyBatis的SqlSession一级缓存，今天继续探究一些MyBatis的二级缓存，虽然说二级缓存的使用存在很大的局限性，但是在特点的场景下还是相当不错的，ok，go！

#### 应用场景
缓存的作用主要是能够提高查询效率，但如果是经常要进行添加、修改、删除等操作，需要同步更新缓存信息，这个时候性能和数据缓存一致性问题都是一个问题。MyBatis的二级缓存主要用在访问量大，但实时性要求不高的场景，比如在武汉疫情盛行的现阶段，我基本每天会使用丁香医生关注最新的疫情动态，这个小型应用有不错的访问量，数据应该是每天定时统计更新几次，就非常适合使用MyBatis的二级缓存：

![丁香医生疫情图](/images/200213_mybatis_level2_cache_1.png)

#### 如何使用二级缓存
MyBatis的二级缓存默认是不开启的，那么我们就一步步的来开启并使用二级缓存。
首先，在mybatis-config.xml配置启动二级缓存：   
```
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>
```
POJO对象需要序列化，否则报：    
```java
Exception in thread "main" org.apache.ibatis.exceptions.PersistenceException: 
### Error committing transaction.  Cause: org.apache.ibatis.cache.CacheException: Error serializing object.  Cause: java.io.NotSerializableException: com.boykait.user.model.User
### Cause: org.apache.ibatis.cache.CacheException: Error serializing object.  Cause: java.io.NotSerializableException: com.boykait.user.model.User
```
在UserMapper文件中添加cache配置并使用：    

```java

<cache
    eviction="FIFO"
    flushInterval="60000"
    size="512"
    readOnly="false"
/>
< select id="listUsers" resultType="com.boykait.user.model.User" flushCache="false" useCache="true">
  SELECT
    id,user_name AS userName, password,gender,age,telephone,email,address
  FROM tbl_user;
</select>

```

具体请求调用，可见不同的sqlSession请求可以实现缓存共享：
```java
public static void main(String[] args) throws Exception{
    InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    // 创建sqlSession1
    SqlSession sqlSession1 = sqlSessionFactory.openSession();
    UserMapper userMapper1 = sqlSession1.getMapper(UserMapper.class);
    List<User> users1 = userMapper1.listUsers();
    System.out.println("第一次，条数：" + users1.size());
    sqlSession1.close();
    // 创建sqlSession2
    SqlSession sqlSession2 = sqlSessionFactory.openSession();
    UserMapper userMapper2 = sqlSession2.getMapper(UserMapper.class);
    List<User> users2 = userMapper2.listUsers();
    System.out.println("第二次，条数：" + users2.size());
    sqlSession2.close();
}
// 结果
DEBUG - Cache Hit Ratio [com.boykait.user.mapper.UserMapper]: 0.0
Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
DEBUG - Opening JDBC Connection
DEBUG - Created connection 1157058691.
DEBUG - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@44f75083]
DEBUG - ==>  Preparing: SELECT id,user_name AS userName, password,gender,age,telephone,email,address FROM tbl_user; 
DEBUG - ==> Parameters: 
DEBUG - <==      Total: 2
第一次，条数：2
DEBUG - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@44f75083]
DEBUG - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@44f75083]
DEBUG - Returned connection 1157058691 to pool.
DEBUG - Cache Hit Ratio [com.boykait.user.mapper.UserMapper]: 0.5
第二次，条数：2

```
二级缓存的配置非常丰富，在不同的Mapper中还可以通过cache-ref进行缓存引用，具体大家可以进一步去官网上查看，接下来我们就来了解下二级缓存是怎么玩的。

#### 二级缓存原理
##### 初始化阶段

首先，在初始化阶段，之前我并没有详细展开mybatis-config.xml所有配置项的各个配置步骤，在settingsElement(settings)阶段，将配置文件中所有的setting项设置到全局configuration对象中：
```java
private void settingsElement(Properties props) {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setDefaultResultSetType(resolveResultSetType(props.getProperty("defaultResultSetType")));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
    configuration.setLogPrefix(props.getProperty("logPrefix"));
    configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
  }
```
其次，在mapperElement步骤内部会调用configurationElement，会依次解析UserMapper.xml文件中的cache-ref和cache配置：
```java
// XMLMapperBuilder
private void configurationElement(XNode context) {
  ...
  String namespace = context.getStringAttribute("namespace");
  builderAssistant.setCurrentNamespace(namespace);
  // 重点，解析引用，存入全局configuration对象的cacheRefMap
  cacheRefElement(context.evalNode("cache-ref"));
  // 解析cache配置信息，存入全局configuration对象的caches中，
  // 主要记录了当前Mapper的配置信息
  cacheElement(context.evalNode("cache"));
  parameterMapElement(context.evalNodes("/mapper/parameterMap"));
  resultMapElements(context.evalNodes("/mapper/resultMap"));
  sqlElement(context.evalNodes("/mapper/sql"));
  buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
  ... 
}

// 相应数据结构Configuration类中定义
{
// cache引用Map定义： 存储<引用namespace, 被引用referencedNamespace>
protected final Map<String, String> cacheRefMap = new HashMap<>();
// cache Map定义
protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
}
```

###### 应用阶段
在cache的应用阶段，无非就主要根据我们是否配置相关的缓存策略来判断是读取缓存数据呢，还是读取数据库数据，还有就是在删除，修改、创建时会清除当前Mapper的cache数据。通过sqlSession1.getMapper(UserMapper.class)获取到mapper代理对象，然后跟着调用链会调用CachingExecutor中的查询方法：
```java
// CachingExecutor
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    // 配置了二级缓存
    if (cache != null) {
      // 判断是否需要刷新缓存，默认false，不做刷新，insert|update|delete一般需要在Mapper.xml中配置 flushCache为true
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // TransactionalCacheManager:tcm中获取缓存数据
        //  Map<Cache, TransactionalCache> transactionalCaches
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 查询并设置二级缓存
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

// 关于TransactionalCacheManager中所使用的TransactionalCache定义
public class TransactionalCache implements Cache {
  private final Cache delegate;
  private boolean clearOnCommit;
  private final Map<Object, Object> entriesToAddOnCommit;
  private final Set<Object> entriesMissedInCache;

  public TransactionalCache(Cache delegate) {
    this.delegate = delegate;
    this.clearOnCommit = false;
    this.entriesToAddOnCommit = new HashMap<>();
    this.entriesMissedInCache = new HashSet<>();
  }
  @Override
  public Object getObject(Object key) {
    // 最终获取数据的地方，从缓存器中获取数据
    Object object = delegate.getObject(key);
    if (object == null) {
      entriesMissedInCache.add(key);
    }
    // issue #146
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }
```
关于上面的delegate是什么样子的呢，我么看一看，在按照demo部分的cache配置，这个具体存储层次怎么这么深，OMG！

![2](/images/200213_mybatis_level2_cache_2.png)

这是什么原理呢，原来Cache的实现是有很多种：

![3](/images/200213_mybatis_level2_cache_3.png)
在构造cache过程中会使用一种叫做装饰器的设计模式来进行缓存数据的存储设计。在创建Cache对象的阶段，会根据我们的配置利用这种设计模式来处理：

```java
// MapperBuilderAssistant
public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
       // 默认使用最简单的PerpetualCache数据存储结构即：Map<Object, Object> cache
       // 默认使用LRU缓存溢出处理策略
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }

// CacheBuilder.build方法	
public Cache build() {
    setDefaultImplementations();
    Cache cache = newBaseCacheInstance(implementation, id);
    setCacheProperties(cache);
    
    if (PerpetualCache.class.equals(cache.getClass())) {
      for (Class<? extends Cache> decorator : decorators) {
        cache = newCacheDecoratorInstance(decorator, cache);
        setCacheProperties(cache);
      }
      // 依次根据配置参数进行修饰操作
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      cache = new LoggingCache(cache);
    }
    return cache;
  }

// CacheBuilder
private Cache setStandardDecorators(Cache cache) {
    try {
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      if (size != null && metaCache.hasSetter("size")) {
        metaCache.setValue("size", size);
      }
      if (clearInterval != null) {
        cache = new ScheduledCache(cache);
        ((ScheduledCache) cache).setClearInterval(clearInterval);
      }
      if (readWrite) {
        cache = new SerializedCache(cache);
      }
      cache = new LoggingCache(cache);
      cache = new SynchronizedCache(cache);
      if (blocking) {
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }

```

#### 脏数据
我们知道了，MyBatis的二级缓存的粒度是Mapper，即一个Mapper对应一个cache缓存，那么如果我们在做查询的操作了关联表，比如查询用户和关联的公司信息：

```
SELECT id, user_name, company_info form tbl_user, tbl_company where tbl_user.company_id=company.id
```
假如我们在user的Mapper中缓存了对应的查询数据，但是如果我们在company对应的Mapper中修改了company_info信息，那么在user的缓存中是不会被更新到的，所以这个时候就可能会出现脏数据了。所以在使用的使用要考虑避免这种情况。

#### 总结
OK，到这，关于MyBatis二级缓存的实现似乎差不读明白了。当然关于缓存优化，网上就有很多案例和demo了，主要是利用第三方缓存框架比如Redis，分离数据缓存服务和Web应用服务，降低应用服务的压力，大家可以继续深入去学习，配置还是比较简单。到这里，我们应用知道：
- 二级缓存是什么？
- 如何创建二级缓存？
- 如何清除二级缓存的？
- 关于脏数据以及如何避免？