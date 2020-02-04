---
title: mybatis技术内幕阅读杂烩笔记
date: 2020-02-03
categories:
  - mybatis
---


#### mybatis-config.xml的启动流程
1.读取并解析mybatis-config.xml文件,parseConfiguration，得到的是什么呢？DefaultSqlSessionFactory,重点是mappers的映射过程

```java
private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
2. 设置参数 properties\setting等参数
3. 映射mappers->：通过XMLMapperBuilder进行解析 注册到MapperRistory中，存入mapperedStatement中:

```java
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```
 - mapper的级别顺序 package\resource\url\class四种顺序方式
 - 依次解析cache-ref、cache、result、sql、select|delete|update|insert
 - cache通过CacheBuilder解析后存到StrictMap<Cache对象>中，Cache对应的Id就是namespace，cache-ref可以共享Cache对象
 - resultMap: ResultMap: {List<ResultMapping>},存在依赖，未解析到的存到incompleteResultMaps中，后面再返回来解析：
 - MapperBuilderAssistant.addResultMap存入到全局configuration的resultMaps中
 - sql片段： Map<String, XNode>(id, 节点) sqlFragments，其中id为namespace+"."+sqlId,sqlFragments存在于单个mapper中
 - 操作语句： XMLStatementBuilder解析存入到 Map<String, MappedStatement> mappedStatements ， 比较重要的sqlCommandType字段，标识操作类型， 先解析<include> 替换成sql片段节点， ${xxx}”占位符替换成真实的参数
 
```java
public class ResultMapResolver {
  private final MapperBuilderAssistant assistant;
  private final String id;
  private final Class<?> type;
  private final String extend;
  private final Discriminator discriminator;
  private final List<ResultMapping> resultMappings;
  private final Boolean autoMapping;
```
 - langDriver.createSqlSource中XMLLanguageDriver.createSqlSourceO 方法中会创建 XMLScriptBuilder 对 象并调用
XMLScriptBuilder.parseScriptNode（）方法创建 SqlSource 对象，会判断是否为动态sql
 - MappedStatement核心结构

```java
 public SqlSource parseScriptNode() {
    // 先判断当前的节点是不是有动态 SQL ，动态 SQL 会包括占位符或是动态 SQL 的相关节点
// SqlNode 集合包装成一个 MixedSqlNode ，后面会详细介绍 SqlNode 以及 MixedSqlNode 的 内容
    MixedSqlNode rootSqlNode = parseDynamicTags(context);
    SqlSource sqlSource;
、// 根据是否是动态 SQL ， 创建相应的 SqlSource 对象  判断动态sql的操作
    if (isDynamic) {
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }
```
```java
private Str工ng resource ; II 节点中的 id~性（包括命名空间前缀）
private SqlSource sqlSource ; II SqlSource 对象，对应一条 SQL 语句
private SqlCommandType sqlCommandType ; II SQL 的类型 ， INSERT 、 UPDATE 、 DELETE、 SELECT或 FLUSH
```
- OGNL ( Object Graphic Navigation Language，对象图导航语言）
- DynamicContext 主要用于记录解析动态 SQL 语句之后产生的 SQL 语句片段，可以认为它
是一个用于记录动态 SQL 语句解析结果的容器。 还有静态类型的
- SqlNode: 各个xml标签对应的节点定义， 我的理解，在该阶段将所有的xml解析存入到节点中，形成一颗SqlNode树
- 在经过 SqlNode.apply（）方法的解析之后， SQL 语句会被传递到 SqlSourceBuilder 中进行进
一步的解析。 SqlSourceBuilder 主要完成了两方面的操作， 一方面是解析 SQL 语句中的“＃｛｝”
占位符 中定义的属性，格式类似于＃｛ 仕c item_ 0, javaType= int, jdbcType=NUMERIC,
typeHandler=MyTypeHandler ｝， 另一方面是将 SQL 语句中的呀。”占位符替换成“？ ” 占位符。，再调用StaticSqlSource

- 延迟加载-动态代理

#### 接口层
- 如何使用sqlsession相关
- SqlSession 是 MyBatis 核心接口之一，也是 MyBatis 接口层的主要组成部分，对外提供
MyBatis 常用 APL MyBatis 提供了两个 Sq!Session 接口的 实现，如图 3-53 所示，这里使用了 工厂方法模式，其中开发人员最常用的是 DefaultSq!Session 实现 。
- SqlSession->DefaultSqlSession 是定义了一系列的操作方法CURD和事务的接口，真正其作用的是内部的Executor：

```java
public class DefaultSqlSession implements SqlSession {

private final Configuration configuration;
private final Executor executor;

private final boolean autoCommit;
private boolean dirty;
private List<Cursor<?>> cursorList;
```
- 在 DefaultSq!Session 中使用到了策略模式， DefaultSq!Session 扮演了 Context 的角色，而将所有数据库相关的操作全部封装到 Executor 接口实现中，并通过 executor 字段选择不同的
Executor 实现。  
- DefaultSqlSession SqlSessionManager  SqlSessionTemplate（Spring和SpringBoot）
- DefaultSqlSessionFactory主要产生DefaultSqlSession

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 获取 mybatis-config.xml 配置文件 中 配置的 Environment 对象
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
      boolean autoCommit;
      try {
        autoCommit = connection.getAutoCommit();
      } catch (SQLException e) {
        // Failover to true, as most poor drivers
        // or databases won't support transactions
        autoCommit = true;
      }
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      final Transaction tx = transactionFactory.newTransaction(connection);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
- SqlSessionManager 同时实现了 SqlSession 接口和 SqlSessionFactory 接口 ，也就同时提供了	SqlSessionFactory 创建 SqlSession 对象 以及 SqlSession 操纵数据库的功能。

```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final SqlSessionFactory sqlSessionFactory;
  // ThreadLocal 变量，记录一个与当前线希呈绑定的 SqlSession 对象
  private final SqlSession sqlSessionProxy;

  // localSqlSession 中记录的 SqlSession 对象的代理对象，在 SqlSessionManager初始化时，
  // 会使用 JDK 动 态代理的 方式为localSqlSession 建代理对象
  private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<>();
}
```
- Sq!SessionManager 与 DefaultSqlSessionFactory 的主要不同点是 Sq!SessionManager 提供了两种模 式： 第 一 种模式 与 DefaultSq!SessionFactory 的行为相同，同 一 线程每次通过
Sq!Ses sionManager 对象访问数据库时，都会创建新的 DefaultSession 对象完成数据库操作：第
二种模式是 Sq!SessionManager 通过 loca!Sq!Session 这个 ThreadLocal 变量，记录与当前线程绑定的 Sq!Session 对象，供当前线程循环使用，从而避免在同一线程多次创建 Sq!Session 对象带来的性能损失。

- 以查询过程为例

```java
// 1. DefaultSqlSession
public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

// 2. Executor.query

 @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 从SqlNode树种拼装SQL语句
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }

// 3. BaseExecutor.query为例
...
if (list != null) {
  handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
} else {
  list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
...

// 4. BaseExecutor.queryFromDatabase
 private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 重点  BatchExecutor SimpleExecutor ReuseExecutor ClosedExecutor根据配置策略选一个
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

// 5. SimpleExecutor为例, JDBC操作
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
// 
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }

// SimpleStatementHandler.query
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    // 执行JDBC查询操作
    statement.execute(sql);
    // 处理查询的结果数据，主要就是执行数据库字段到java类型的映射转换
    return resultSetHandler.handleResultSets(statement);
  }

```

- getMapper方式
-  CustomerMapper mapper = (CustomerMapper)sqlSession.getMapper(CustomerMapper.class);
-  返回的是代理类 h{sqlSession mapperInterface methodCache}
- 