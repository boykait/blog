---
title: MyBatis技术解密（二）：初始化流程
date: 2020-02-05
categories:
  - MyBatis
tags:
  - MyBatis
  - MyBatis技术解密
---
> 继续“解密”，Mybatis整体分为两大部分：配置信息的初始化阶段和具体使用阶段，MyBatis初始化过程说白了就是主要解析mybatis-config.xml和各个mapper.xml（注解方式略微不同），生成内存配置信息供后面使用，那我们本篇就介绍一下mybatis的配置信息解析流程。

#### 从初始化入口开始
我们还需要明确一点，初始化最最主要的目的就是根据XML配置文件生成SqlSessionFactory工厂，然后使用的时候从工厂中产生SqlSession，当然Sql具体的执行也是委托给Executor的，这个后面再说。还是先看一下之前的使用demo：
```java
// 初始化配置阶段
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
// 使用阶段
SqlSession sqlSession = sqlSessionFactory.openSession();
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
List<User> users = mapper.listUsers();
```
以输入流方式读入mybatis-config.xml配置文件，然后交给SqlSessionFactoryBuilder构造器来进行具体sqlSessionFactory的构建过程，先跟踪build方法干些什么是，最后我们可以定位到如下方法：

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
   ...
  // 1. 根据配置文件创建XMLConfigBuilder
  XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
  // 2. 执行解析过程得到Configuration全局配置对象
  // 3. 根据该配置对象实例化DefaultSqlSessionFactory
  return build(parser.parse());
  ...
}

```
我们的关注点开业进一步定位到parse方法了，parse()调用parseConfiguration()方法就开始按流程解析`mybatis-config.xml`配置文件了：

```java
  //分步骤解析
  //1.properties
  propertiesElement(root.evalNode("properties"));
  //2.类型别名
  typeAliasesElement(root.evalNode("typeAliases"));
  //3.插件
  pluginElement(root.evalNode("plugins"));
  //4.对象工厂
  objectFactoryElement(root.evalNode("objectFactory"));
  //5.对象包装工厂
  objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
  //6.设置
  settingsElement(root.evalNode("settings"));
  // read it after objectFactory and objectWrapperFactory issue #631
  //7.数据库环境配置
  environmentsElement(root.evalNode("environments"));
  //8.databaseIdProvider 数据库提供商
  databaseIdProviderElement(root.evalNode("databaseIdProvider"));
  //9.类型处理器
  typeHandlerElement(root.evalNode("typeHandlers"));
  //10.映射器，重点
  mapperElement(root.evalNode("mappers"));
```

我们先不挨着分析，看准重心，着重分析mappers的解析过程，这个过程是重要且复杂的，ok，继续跟进mapperElement方法：
```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // package方式，xxxMaper.java必须和xxxMapper.xml同名且在同一路径下
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          // 加入到全局mapperRegistry对应的Map<Class<?>, MapperProxyFactory<?>> knownMappers中
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          // resource方式加载，从resource路径下寻找xml文件进行解析
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            // 执行解析
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            // url方式加载
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            // 执行解析
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            // class方式加载、解析 具体同package方式类似，只是变为解析单个文件
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```
通过分析上一个文件我们知道，这一阶段的主要目的就是将所有的mapper存入到knownMappers中，knownMappers中保的key就是各具体的Mapper接口名，为了方便使用，内部具体会保存两份，全限定接口名("com.boykait.user.mapper.UserMapper")和具体文件名("UserMapper")，这个时候xxxMaper就对应指向了具体的MapperProxyFactory，做这个映射干嘛？你要知道我们具体xxxMaper接口没有实现类，凭什么能够使用？so，它实际的工作就是通过动态代理去完成的，具体的后面再说，现在先知道Maper接口是通过代理执行对应的CURD操作的。    
好的，我们就以resource方式继续跟进，进入mapperParser.parse()：

```java    
//1. XMLMapperBuilder.parse()
public void parse() {
	if (!configuration.isResourceLoaded(resource)) {
      // 重点
	  configurationElement(parser.evalNode("/mapper"));
	  configuration.addLoadedResource(resource);
	  bindMapperForNamespace();
	}
	// 处理前面的半成品
	parsePendingResultMaps();
	parsePendingCacheRefs();
	parsePendingStatements();
}
//2. XMLMapperBuilder.configurationElement()
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      // 处理依赖缓存，全局configuration中 Map<本命名空间, 依赖命名空间> cacheRefMap
      cacheRefElement(context.evalNode("cache-ref"));
      // 处理缓存 全局configuration中 Map<本命名空间, Cache对象> caches 
      cacheElement(context.evalNode("cache"));
      // 作废
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 处理结果映射configuration的 Map<命名空间.resultMapId, ResultMap> resultMaps
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // 处理sql片段 XMLMapperBuilder.Map<命名空间.sqlId, XNode节点数的root节点> sqlFragments
      sqlElement(context.evalNodes("/mapper/sql"));
      // 重点 为CURD创建MappedStatement
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }

// 3. XMLMapperBuilder.buildStatementFromContext()
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        // 重点
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  } 
```

在具体的开发过程中，比如做查询，我们一般会使用include|if|foreach|where|tirm等条件来创建满足需求的查询语句，所以追踪上面的重点继续，我发誓，这是最后一次粘贴源码:

```java
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    String nodeName = context.getNode().getNodeName();
	// 具体是CURD哪种操作命令
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
	// 将sql片段进行替换
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);

    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }
    // 重点 判断是哪一种SqlSource DynamicSqlSource|ProviderSqlSource|RawSqlSource|StaticSqlSource
    // ${}对应DynamicSqlSource #{}对应RawSqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");
    // 创建Map<String, MappedStatement> mappedStatements，存入全局configuration中
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

#### 梳理总结
还是懒，这篇写到这点了吧，到这，简要总结一下：
1. 初始化的主线流程
2. 应该知道mybatis mapper文件加载方式的优先级别
3. xxxMaper接口方法的使用是需要经过代理的（后面具体详细研究）
4. SqlSource具体的判断和创建（后面研究）

