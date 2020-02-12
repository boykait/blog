---
title: MyBatis技术解密（四）：细聊SqlSession
date: 2020-02-09
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

> SqlSession是开发者直接操作MyBatis的接口，无论是用SqlSessionFactory、SqlSessionManager抑或是在Spring和SpringBoot中的SqlSessionTemplate，其底层核心还是调用的JDBC，但这三种产生SqlSession的区别，我们还是有必要探索一下，好咧，走起！
先把这几者的关系做一个梳理


#### Demo展示

SqlSessionFactory版：
```java    
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
// 创建sqlSession
SqlSession sqlSession = sqlSessionFactory.openSession();
// 获取mapper代理对象
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
// 执行查询(CURD)操作
List<User> users = mapper.listUsers();
users.forEach(user -> System.out.println(user.toString()));
sqlSession.close();
```

SqlSessionManager版：
```java    
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
// 实例化sqlSessionManager
SqlSessionManager sqlSessionManager = SqlSessionManager.newInstance(inputStream);
// 开启管理SqlSession，创建一个SqlSession并存入到ThreadLocal中
sqlSessionManager.startManagedSession();
// 使用
UserMapper mapper1 = sqlSessionManager.getMapper(UserMapper.class);
List<User> users1 = mapper1.listUsers();
users1.forEach(user -> System.out.println(user.toString()));
```

#### SqlSessionManager
我们都知道使用DefaultSqlSessionFactory创建出来的DefaultSqlSession是线程非安全的，那么SqlSessionManager的主要作用就是解决SqlSession线程非安全的问题（虽然说现在的开发都主要是结合Spring Or SpringBoot，但SqlSessionManager还是可以稍作研究），SqlSessionManager本身是实现了SqlSession和SqlSessionFactory两个接口，通过SqlSessionManager.newInstance操作的配置初始化操作和DefaultSqlSessionFactory的过程是一致的，最主要就是：

```java
// SqlSessionManager定义
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final SqlSessionFactory sqlSessionFactory;
  // sqlSessionProxy代理
  private final SqlSession sqlSessionProxy;

  private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<>();

  private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    // 为所有的SqlSession相关操作添加代理操作
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
  }
  // newInstance调用
  public static SqlSessionManager newInstance(Reader reader) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, null, null));
  }
}
```

所以说SqlSessionManager的诀窍就主要是在对SqlSession相关的操作进行拦截代理，SqlSessionInterceptor作为SqlSessionManager定义的一个内部类，内部实现了具体的代理逻辑：
```java
@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 开启了SqlSession的管理功能即调用了startManagedSession()，当前SqlSession就存放到localSqlSession中
      final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
      if (sqlSession != null) {
        try {
          return method.invoke(sqlSession, args);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } else {
        // 创建一个新的DefaultSqlSession并执行相关的CURD操作，即每次都使用新的SqlSession对象
        try (SqlSession autoSqlSession = openSession()) {
          try {
            final Object result = method.invoke(autoSqlSession, args);
            autoSqlSession.commit();
            return result;
          } catch (Throwable t) {
            autoSqlSession.rollback();
            throw ExceptionUtil.unwrapThrowable(t);
          }
        }
      }
    }
```

#### SqlSessionTemplate
在公司，现在后端开发子项目都是基于SpringBoot来进行的，好处就不用再唠叨了，方便快速，这里在配置[DEMO]()，在引入mybatis-spring-boot-starter这个starter时会自动引入mybatis-spring-boot-autoconfigure这个依赖包，里面的MybatisAutoConfiguration类文件就去做了SqlSessionFactory，SqlSessionTemplate等的实例化配置并注册到Spring的IOC容器中，核心流程其实和使用单独使用MyBatis异曲同工，由SqlSessionFactoryBean产生SqlSessionFactory实例：

注册SqlSessionFactory实例到IOC:
![1](images/200209_mybatis_session_1.png)
![2](images/200209_mybatis_session_2.png)

SqlSessionTemplate对象:
![3](images/200209_mybatis_session_3.png)
```java
 public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {
    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // 返回SqlSession代理对象
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
```
好像和SqlSessionManager的创建过程的很像，那我们继续看SqlSessionInterceptor:

```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        // 如果不是由spring管理事务，那就直接提交
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        // 关闭sqlSession
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```
其实核心操作当然也是调用method.invoke(sqlSession, args)，但是相比SqlSessionManager多了一些东西呢，MyBatis在配合Spring或SpringBoot时会将事务管理功能委托给后者来完成，所以在代理阶段，Spring或SpringBoot会进行事务的开启和关闭操作，在getSession阶段，如果开启了事务功能，首先尝试从事务同步器中获取（当然第一次是不能获取到的），否则就通过DefaultSqlSessionFactory中实例化一个DefaultSqlSession来使用并根据事情的开启情况注册到当前事务中：
```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    ...
    // 从事务同步器中获取
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    // 从工厂中获取
    session = sessionFactory.openSession(executorType);
    // 根据是否绑定了事务操作来判断是否注册到事务管理器中
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```
那么重点来了，SqlSessionTemplate模式实现线程安全的秘诀是？现在来揭晓吧。

```java
// SqlSessionUtils
private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
    SqlSessionHolder holder;
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      Environment environment = sessionFactory.getConfiguration().getEnvironment();

      if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
        holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
        // bindResource是重点
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);
        TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
        holder.setSynchronizedWithTransaction(true);
        holder.requested();
      } else {
        ...
    } 
    ...
}

// TransactionSynchronizationManager
public static void bindResource(Object key, Object value) throws IllegalStateException {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Assert.notNull(value, "Value must not be null");
		Map<Object, Object> map = resources.get();
		// set ThreadLocal Map if none found
		if (map == null) {
			map = new HashMap<>();
			resources.set(map);
		}
        // 重点，设置resources中的map值
		Object oldValue = map.put(actualKey, value);
		// Transparently suppress a ResourceHolder that was marked as void...
		if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
			oldValue = null;
		}
		if (oldValue != null) {
			throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
					actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Bound value [" + value + "] for key [" + actualKey + "] to thread [" +
					Thread.currentThread().getName() + "]");
		}
	}
```
答案可以揭晓了，其实秘诀还是ThreadLocal，resources是定义为ThreadLocal<Map<Object, Object>>，里面就存储了当前请求的<sqlSessionFactory，sqlSessionHolder>，在getResource将sqlSessionFactory作为key去取获取数据，这也说明了一个线程里面只能有一个sqlSession实例对象。

#### 总结
- SqlSessionManager和SqlSessionTemplate都是通过动态代理功能实现的SqlSession功能增强；
- 单用MyBatis的时候可以使用DefaultSqlSessionFactory或SqlSessionManager去创建SqlSession，但后者相对更优，通过SqlSession本地化实现了线程安全，此外还完成了自动关闭；
- 结合Spring或SpringBoot后，使用的是SqlSessionTemplate，核心思想也是通过SqlSession本地化完成的线程安全，但在代理过程中增加了事务处理功能。
