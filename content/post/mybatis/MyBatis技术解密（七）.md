---
title: MyBatis技术解密（七）：MyBatis事务管理
date: 2020-02-17
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

> MyBatis在结合Spring或SpringBoot时会将事务功能交由后者来进行管理，但这并不妨碍我们对它的学习，不然看着mybatis-config.xml中的`<transactionManager type="JDBC"/>`配置项总会觉得这个是咋个玩起来了，蒙，ok，为了完善一下知识链，这一篇文章就来研究一下。

#### 

想要使用MyBatis事务功能的话我们在环境变量配置的时候一般需要指定事务管理器以及类型，即上面提到的`<transactionManager type="JDBC"/>`，事务的被抽象成接口，目前具体实有两种实现模式，以继承图方式看一下：

![](/images/200217_mybatis_transaction_management_1.png)

所以，我么上面提到的type的值有两种，除了JDBC类型，还有MANAGED类型，我们一般是使用前者，JDBC类型是默认使用JDBC提供的事务管理功能，MANAGED是将事务管理交由容器处理，在MyBatis中，接下来，我们开始以JDBC类型的的常规使用作为入口进行分析，demo走一个：

```java 

public class TransactionTest {
    public static void main(String[] args) throws Exception {
        InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        // 创建sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
            User user = userMapper.getUserById("1");
            System.out.println(user.toString());
            doUpdateTest(userMapper, "1");
            sqlSession.commit();
        } catch (Exception e) {
            sqlSession.rollback();
        } finally {
            sqlSession.close();
        }
    }

    private static Integer doUpdateTest(UserMapper mapper, String id) {
        User u = new User();
        u.setId(id);
        u.setUserName("小明");
        u.setPassword("123456789");
        u.setGender("男");
        u.setTelephone("1234567890");
        u.setEmail("xxx@126.com");
        u.setAddress("四川成都");
        return mapper.updateUser(u);
    }
}
```
我们要知道，在MyBatis中，事务功能是默认开启的，所以我们就需要知道事务是何时开启，何时使用？源码之下无秘密，在openSession阶段，为当前SqlSession注入了事务能力，TransactionIsolationLevel是对应数据库的四种隔离级别，我们可以手动指定：

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment); 
      // 创建对应的实例jdbcTransaction或者ManagedTransaction
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      // 返回我们熟悉的DefaultSqlSession实例对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

正如我们前面所说，指定为JDBC事务是使用JDBC自身事务功能来实现的，MyBatis只是封装了而已：

```java
public class JdbcTransaction implements Transaction {

  private static final Log log = LogFactory.getLog(JdbcTransaction.class);

  protected Connection connection;
  protected DataSource dataSource;
  protected TransactionIsolationLevel level;
  protected boolean autoCommit;

  public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommit = desiredAutoCommit;
  }

  public JdbcTransaction(Connection connection) {
    this.connection = connection;
  }

  @Override
  public Connection getConnection() throws SQLException {
    if (connection == null) {
      openConnection();
    }
    return connection;
  }

  @Override
  public void commit() throws SQLException {
    // 提交事务
    if (connection != null && !connection.getAutoCommit()) {
      connection.commit();
    }
  }

  @Override
  public void rollback() throws SQLException {
    // 回滚
    if (connection != null && !connection.getAutoCommit()) {
      connection.rollback();
    }
  }

  @Override
  public void close() throws SQLException {
     // 关闭
    if (connection != null) {
      resetAutoCommit();
      connection.close();
    }
  }

  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
      if (connection.getAutoCommit() != desiredAutoCommit) {
        connection.setAutoCommit(desiredAutoCommit);
      }
  }

  protected void resetAutoCommit() {
      if (!connection.getAutoCommit()) {   
        connection.setAutoCommit(true);
      } 
  }

  protected void openConnection() throws SQLException {
    connection = dataSource.getConnection();
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());
    }
    setDesiredAutoCommit(autoCommit);
  }
}

```
ManagedTransaction实现类：通过容器来进行事务管理，它对事务提交和回滚并不会做任何操作，源码如下：

```java
  @Override
  public void commit() throws SQLException {
    // Does nothing
  }

  @Override
  public void rollback() throws SQLException {
    // Does nothing
  }
````

我们都知道，MySQL数据库的默认隔离级别是可重复读，是通过next-keys技术解决幻读问题的，客户端查看一下：

```
mysql> select @@global.tx_isolation,@@tx_isolation;
+-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
```

接下来，数据库的事务隔离级别的设置可分为全局和当前会话两种，我们在客户端如果使用默认的隔离级别那么请求传递到数据库服务端对当前请求会话其实就是使用了global全局的隔离级别：

```java
public enum TransactionIsolationLevel {
  NONE(Connection.TRANSACTION_NONE),
  READ_COMMITTED(Connection.TRANSACTION_READ_COMMITTED),
  READ_UNCOMMITTED(Connection.TRANSACTION_READ_UNCOMMITTED),
  REPEATABLE_READ(Connection.TRANSACTION_REPEATABLE_READ),
  SERIALIZABLE(Connection.TRANSACTION_SERIALIZABLE);

  private final int level;

  TransactionIsolationLevel(int level) {
    this.level = level;
  }

  public int getLevel() {
    return level;
  }
}
```
#### 总结
事务这块好像没写什么东西，就其核心就是有两种管理方式：JDBC和MANAGED，前者是使用原生JDBC事务管理，后者是交由JBoss这类的容器管理，在结合Spring的时候，再继续补充，因为那个时候就完全就是使用Spring的统一事务管理器来进行事务的管理了。



