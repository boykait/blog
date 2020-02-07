---
title: MyBatis技术解密（三）：执行阶段
date: 2020-02-06
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

#### 概述
 初始化阶段的工作核心是根据配置生成SqlSessionFactory实例对象，当然配置信息已经解析并保存到全局configuration对象中，ok，现在我们就考虑具体使用阶段的主线原理。执行阶段按demo分为三大步骤：

```java
// 创建sqlSession
SqlSession sqlSession = sqlSessionFactory.openSession();
// 获取mapper代理对象
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
// 执行查询(CURD)操作
List<User> users = mapper.listUsers();
```

#### 1. 获取SqlSession
openSession有多个重载，具体使用见名知意，不解释，我们使用默认的DefaultSqlSessionFactory.openSession()，其实流程较为简单，跟着源码走览:

```java    
// SqlSessionFactory
SqlSession openSession();

SqlSession openSession(boolean autoCommit);

SqlSession openSession(Connection connection);

SqlSession openSession(TransactionIsolationLevel level);

SqlSession openSession(ExecutorType execType);

SqlSession openSession(ExecutorType execType, boolean autoCommit);

SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);

SqlSession openSession(ExecutorType execType, Connection connection);

// DefaultSqlSessionFactory.openSession()
@Override
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

// DefaultSqlSessionFactory.openSessionFromDataSource
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 事务实例 重要，后面单独研究
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 得到defaultSqlSession对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

#### 2. 获取mapper代理对象
前一篇中说道xxxMaper解析后会存放到全局configuration对象的mapperRegistry的knownMappers中，那么获取就简单了嘛，直接去拿就可以了：

```java
// DefaultSqlSession.getMapper
public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
}

// Configuration.getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}

// MapperRegistry.getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
	if (mapperProxyFactory == null) {
	  throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
	}
	try {
	  return mapperProxyFactory.newInstance(sqlSession);
	} catch (Exception e) {
	  throw new BindingException("Error getting mapper instance. Cause: " + e, e);
	}
}
```

#### 3. 执行CURD操作
ok，调用Mapper接口的代理对象能够可以进行具体的CURD操作，那么代理内部最终必然执行了数据库的请求操作，我们可以定位到MapperProxy中查看代理过程：

```java
// MapperProxy定义
public class MapperProxy<T> implements InvocationHandler, Serializable {
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  // 重点
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      // 如果是具体类，调用类方法
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (method.isDefault()) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 接口方式
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 重点，
    return mapperMethod.execute(sqlSession, args);
  }
}

```
mapperMethod.execute的核心工作就是判断我们当前具体是CURD中的那个操作，然后调用SqlSession具体的方法，该方法逻辑比较简单清晰，我们就以执行查询操作继续，其它类似：
```java    
// MapperMethod.execute
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    // 对增加|修改|删除执行完毕后统计影响行数
    // 对查询操作执行完毕后进行类型转换操作
    switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          // 进入
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }

// MapperMethod.executeForMany
private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    // 看看，内部调用了SqlSession的列表查询方法了啊
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
```

我们当然也可以直接调用SqlSession的selectList列表查询方法，和Mapper代理方式殊途同归，selectList()方法里：
```java    
// DefaultSqlSession.selectList
@Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 获取mappedStatement， statement为命名空间.mapperXmlId：com.boykait.user.mapper.UserMapper.listUsers
      MappedStatement ms = configuration.getMappedStatement(statement);
      // 执行查询
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

executor查询器会根据我们的配置使用	BaseExecutor|CachingExecutor，默认调用的是CachingExecutor（为什么呢）：

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
   // 绑定Sql语句和具体参数
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 缓存查询语句，方便同session操作共享
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)  {
  ...
  list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
  if (list != null) {
    // 同一个session中如果有相关查询缓存，则直接从缓存中获取
    handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
  } else {
    // 从数据库中获取
    list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }
  ...
  return list;
}

```

```java
@Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //配置statement(包括获取connection数据库连接、事务等)
      stmt = prepareStatement(handler, ms.getStatementLog());
      // 重点
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

// PreparedStatementHandler.query
@Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    // 啥也不说了，JDBC方式进行查询操作
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
  }
```

#### 总结
到这里，MyBatis初始化解析和执行两个阶段的主线流程基本就结束了，当然内部还有很多细节未逐一展开，后面有时间会将重要的部分做进一步的学习研读。整了个时序图，更清楚的知道其执行流程:

![](/images/191115_mybatis_action_3.png)
