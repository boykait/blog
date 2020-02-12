---
title: MyBatis技术解密（五）：MyBatis一级缓存
date: 2020-02-12
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

> MyBatis有一级缓存和二级缓存两种，这两种分别是在什么场景下用到呢？默认的开启状态、如何开启？什么时候缓存会失效？缓存是如何存储的等等，本篇的重点就是研究一级缓存相关的特点和相关源码设计。go！

#### 一级缓存
一级缓存是默认是处于开启状态的，是session级别的，意味着在同一SqlSession的相同的查询操作是不会再重复请求数据库的：

```java
public static void main(String[] args) throws Exception{
        InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        // 创建sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();
//        List<User> users = (List)sqlSession.selectList("xxxxx.listUsers", "1");
//        // 获取mapper代理对象
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
//        // 执行查询(CURD)操作
        List<User> users = mapper.listUsers();
        System.out.println(users.size());
        // 再次查询
        List<User> users1 = mapper.listUsers();
        System.out.println(users1.size());
        sqlSession.close();
    }
```
Debug查看数据库请求：    

```java
DEBUG - Opening JDBC Connection
DEBUG - Created connection 1263668904.
DEBUG - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@4b520ea8]
DEBUG - ==>  Preparing: SELECT id,user_name as userName, password,gender,age,telephone,email,address FROM tbl_user; 
DEBUG - ==> Parameters: 
DEBUG - <==      Total: 2
第一次：2
第二次：2
```
可以看出只向数据库发送了一次查询请求，这说明缓存起效了，那么如果在两次查询操作之间进行一次提交操作呢：sqlSession.commit()，Debug查看，是需要发送两次数据库查询请求的：

```java
DEBUG - Created connection 1263668904.
DEBUG - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@4b520ea8]
DEBUG - ==>  Preparing: SELECT id,user_name as userName, password,gender,age,telephone,email,address FROM tbl_user; 
DEBUG - ==> Parameters: 
DEBUG - <==      Total: 2
第一次：2
DEBUG - ==>  Preparing: SELECT id,user_name as userName, password,gender,age,telephone,email,address FROM tbl_user; 
DEBUG - ==> Parameters: 
DEBUG - <==      Total: 2
第二次：2
```
这说明commit操作会触发sqlSession缓存的失效，当然，除了commit操作，其它的删除、修改、创建操作都会导致sqlSession的失效：
```java
# 更新本条缓存的数据导致缓存失效
DEBUG - ==>  Preparing: SELECT id,user_name AS userName, password,gender,age,telephone,email,address FROM tbl_user WHERE id=?; 
DEBUG - ==> Parameters: 1(String)
DEBUG - <==      Total: 1
DEBUG - ==>  Preparing: UPDATE tbl_user SET user_name=?, password=?, gender=?, age=?, telephone=?, email=?, address=? Where id=? 
DEBUG - ==> Parameters: xiaoming1(String), 1234567(String), 男(String), null, 1234567890(String), xxx@126.com(String), 四川成都(String), 1(String)
DEBUG - <==    Updates: 1
DEBUG - ==>  Preparing: SELECT id,user_name AS userName, password,gender,age,telephone,email,address FROM tbl_user WHERE id=?; 
DEBUG - ==> Parameters: 1(String)
DEBUG - <==      Total: 1

# 更新其它数据条目同样会导致缓存失效
DEBUG - ==>  Preparing: SELECT id,user_name AS userName, password,gender,age,telephone,email,address FROM tbl_user WHERE id=?; 
DEBUG - ==> Parameters: 1(String)
DEBUG - <==      Total: 1
DEBUG - ==>  Preparing: UPDATE tbl_user SET user_name=?, password=?, gender=?, age=?, telephone=?, email=?, address=? Where id=? 
DEBUG - ==> Parameters: xiaoming1(String), 1234567(String), 男(String), null, 1234567890(String), xxx@126.com(String), 四川成都(String), 2(String)
DEBUG - <==    Updates: 1
DEBUG - ==>  Preparing: SELECT id,user_name AS userName, password,gender,age,telephone,email,address FROM tbl_user WHERE id=?; 
DEBUG - ==> Parameters: 1(String)
DEBUG - <==      Total: 1
```
OK，那我们进一步去了解一级缓存的实现机制

#### 缓存的设置时机
我们在进行maper查询的时候，最终会调CachingExecutor的query方法，这个方法在前面的执行阶段中有出现过，那我们就主要看看创建的缓存key是什么:
```ava
-1535480254:1232846930:com.boykait.user.mapper.UserMapper.getUserById:0:2147483647:SELECT
        id,user_name AS userName, password,gender,age,telephone,email,address
      FROM tbl_user WHERE id=?;:1:development
```
可以看出，缓存的key键其实就是我们这条查询语句呢

```java

//1. CachingExecutor
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	// 组合参数和Sql语句
	BoundSql boundSql = ms.getBoundSql(parameterObject);
	// 创建需要缓存的key，
	CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
	return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

// 2
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    // 使用二级缓存，先不关注
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
    // 执行查询，看是否存在一级缓存
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

// 3. 进入BaseExecutor中定义的query中
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ...
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 重点根据缓存key去localCache中拿去存放的值：localCache.getObject -> cache.get(key) ; 其中cache定义Map<Object, Object> cache = new HashMap<>()
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        // 存储过程相关处理，先不care
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 没有缓存或者缓存失效，就直接冲数据库中读取了
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }

  // 在queryFromDatabase方法时查询数据库，并把数据设置到缓存中

private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
通过query的调用链，应该可以清晰的知道一级缓存的是缓存在一个HashMap结构中的，对应的key就是封装的查询语句，当第一次通过数据库的查询操作会将数据缓存在该数据结构中，后面的相同的查询语句就会直接去读取里面的数据，当然是在缓存有效的情况下。

#### 缓存失效
上面说到，通过commit、rollback、update、insert或者delete相关操作会使得一级缓存失效，说白了这些操作是伴随了清空缓存相关行为的：

```java
// BaseExecutor类中
// insert update delete都调用该方法
public int update(MappedStatement ms, Object parameter) throws SQLException {
    clearLocalCache();
    return doUpdate(ms, parameter);
  }

  public void commit(boolean required) throws SQLException {
    clearLocalCache();
    flushStatements();
    if (required) {
      transaction.commit();
    }
  }

public void rollback(boolean required) throws SQLException {
    if (!closed) {
      try {
        clearLocalCache();
        flushStatements(true);
      } finally {
        if (required) {
          transaction.rollback();
        }
      }
    }
  }
```

#### 总结
ok，差不多，一级缓存的知识点相对比较简单，先分析到这里，此时，我们应该知道一级缓存的相关特点：
- 1. 一级缓存的作用域？
- 2. 缓存存储结构，什么时候会设置缓存，如何取到缓存值？
- 3. 什么情况下缓存会被清除掉？






