---
title: MyBatis技术解密（八）：延时加载
date: 2020-02-19
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---

> 我是一个SQL“高手”，在日常CURD中喜欢嵌套查询、多表关联查询，嗯，前些时间有同事离职让我来传递香火，在后期拜读他的代码过程中，有使用不少的association、collection来处理的表关联查询，这怪我这之前使用MyBatis不多，更没得撒深入研究的经验，借着研究MyBatis源码这股劲，来瞅瞅使用这玩意和直接使用SQL语句来完成数据拼接有啥区别，孰强孰弱，那种场景用那种方式比较好？OK，经过两天时间分析和在线精品文章学习，再稍加整理，这篇文章便就诞生了。

#### 应用场景
其实学习一门技术或一个特性，如果能将学好理论再结合具体项目的应用场景来分析便是极好的，在学习延迟加载特性的时候，我在网上看了一天，搜索：MyBatis延时加载的应用场景，但其实得到的demo案例都只是为了说明延时加载特性，在实际项目中什么样的场景具体能够怎么用还是有点懵逼，下面两篇还可以，分享一下：      
1. [mybatis 详解（八）------ 懒加载](https://www.cnblogs.com/ysocean/p/7336945.html?utm_source=debugrun&utm_medium=referral#_label0)    
2. [Mybatis 延迟加载](https://www.cnblogs.com/jack1995/p/7260722.html)  
3. [【MyBatis学习11】MyBatis中的延迟加载](https://blog.csdn.net/eson_15/article/details/51668523)  
嗯，其实看了这些文章后，我的疑虑是这个延时加载一般是在我们查询到数据后，然后在对数据处理时再做关联数据的查询，比如第二篇里面的：    
```java    
public void testFindOrdersByLazyLoad() throws Exception{
	SqlSession session = sessionFactory.openSession();
	Mapper mapper = session.getMapper(Mapper.class);
	// 查询主数据-只会发送查询订单信息的SQL
	List orders = mapper.findOrdersByLazyLoad();
	for (Orders order : orders){
		// 查询关联数据-会发生查询用户信息的SQL
		order.getUser();
	}
}

```

但是正如我给那些文章后面的留言，当然可能时间关系，博主没有回复，疑问是：

- 如果查询订单并且关联查询用户信息。如果先查询订单信息即可满足要求，当我们需要查询用户信息时再查询用户信息。把对用户信息的按需去查询就是延迟加载。你好，我有个疑问，就是假如说我们将订单信息以api反馈给前端，这个时候如果使用了懒加载，那么用户信息必然是不在订单详情里面的，那么我在向如何又能够触发到懒加载功能去数据库查询用户信息呢？ 其实这个如果已经提供给前端了额api，我觉得应该是没办法再触发懒加载了，要查询里面的用户信息，就下发新的接口查询，即根据订单里面的用户Id查询用户的详情，对吧？所以其实我比较迷惑的是到以api提供能用到懒加载吗？你这个都是在后端同一个请求里面进行的操作，还有懒加载的真实应用场景?

没搞懂的话，就不能很好的装13了，后面我几经思考，摸索出一个项目可以使用延时加载的应用场景，至少我觉得是ok的。具体项目涉及到公司的版权业务，就不细说，但可以简说一下场景：

- 提供云资源的列表查询，比如虚拟机数据里面有镜像、硬盘、公网IP、安全组、所属用户等关联查询对象；
- 在进行概览页面展示的时候只需要统计虚拟的使用情况，即正常、异常、总量、使用量等，和后面的关联资源就没什么关系了；
- 在做资源报表分析展示的时候，我可能要统计虚拟机按组织、按人头、按正常、异常情况进行汇总统计，也不涉及关联的镜像、公网IP等

ok，至少我觉得在项目中这个场景可以使用MyBatis的延时加载功能了，是伐，查询虚拟机列表时要查询关联数据，可以在Controller返回api数据前触发一次关联数据调用，可以统一的封一个util方法处理；概览和报表统计时不需要关联数据。感觉有这技术有实际的用武之地，心理踏实一些，那就继续开搞。

#### 如何使用    
网上教程还是比较多，MyBatis的延时加载功能主要是针对association和collection关联数据的查询操作，想要使用，首先我们要知道：
Mybatis的延迟加载功能默认是关闭的，需要开启延迟加载的属性，这个就不用多写了，上配置说明即可：

![延迟加载配置项说明](/images/200220_mybatis_lazyloading_1.png)


#### 走进源码
先提前总结：懒加载的核心就是**动态代理**！！！ 双击，又是动代！！！

动态代理相关的也是设置到全局configuration对象中：
```java
public class Configuration {
  protected boolean aggressiveLazyLoading;
protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
  protected boolean lazyLoadingEnabled = false;
  protected ProxyFactory proxyFactory = new JavassistProxyFactory();

 // 默认是javassist，看情况还可以使用cglib代理
 public void setProxyFactory(ProxyFactory proxyFactory) {
    if (proxyFactory == null) {
      proxyFactory = new JavassistProxyFactory();
    }
    this.proxyFactory = proxyFactory;
  }
}
```
那我们知道了延迟加载在动态代理阶段进行，那么具体时机是在哪个地方呢？以SimpleStatementHandler方式查询为例：
```java
 public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    // 处理结果
    return resultSetHandler.handleResultSets(statement);
  }
```
在handleResultSets中，最终会调用createResultObject来创建结果对象：

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>();
    final List<Object> constructorArgs = new ArrayList<>();
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
      for (ResultMapping propertyMapping : propertyMappings) {
        // issue gcode #109 && issue #149
        // 如果lazy 延时加载，则为该属性创建代理
        if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
          resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
          break;
        }
      }
    }
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
  }
```
