---
title: Spring IoC初始化过程
date: 2018-07-12 13:13:47
categories:
 - Spring
tags:
 - Spring
---

### 1. BeanFactory和ApplicationContext

Spring IoC衍生出了以BeanFactory和ApplicationContext的两大分支，但我们还是要理清，后者是继承前者的一个接口。

看一下ApplicationContext接口的继承图：

<center>
![2018-09-01-001](/images/2018-09-01-001.jpg)
</center>

ApplicationContext除了继承BeanFactory接口，还包括

```
ResourceReader———实现资源配置文件加载
MessageSource———消息源接口，实现国际化支持
ApplicationEventPublisher———引入事件机制，包括启动事件、关闭事件等，让容器在上下文中提供了对应用事件的支持
```

直接的BeanFactory实现是IOC容器的基本形式，ApplicationContext是实现了IoC容器的高级表现形式，但两者最根本的实现是DefaultListableBeanFactory，DefaultListableBeanFactory也是Spring默认的IoC容器。在后面的源码分析中可以看到它是如何在ApplicationContext系列容器中使用的。

ApplicationContext有多种实现方式，主要是根据多种不同的资源加载方式来实现的：

```
FileSystemXmlApplicationContext:从文件系统中加载资源配置文件
ClassPathXmlApplicationContext：从类路径中加载
WebXmlApplicationContext：允许从相对于Web根目录的路径中装载配置文件
AnnotationConfigApplicationContext：基于注解的资源配置文件加载方式
```

接下来，以ClassPathXmlApplicationContext来浅要解析一下Spring IoC的初始化过程。

### 2. ClassPathXmlApplicationContext使用示例

来一个老少皆宜的学生课程成绩查询，代码不用解释，很easy。

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

class Student {
    private Integer id;
    private String name;
    //getter and setter...
}

class Course {
    private Integer id;
    private String name;
    //getter and setter...
}

class Score {
    private Integer Score;
    private Student student;
    private Course course;
    //getter and setter...
}

public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("injection/injection.xml");
        Score score = (Score) context.getBean("score");
        System.out.println(score.getStudent().getName() + " " + score.getCourse().getName() + " : " + score.getScore());
    }
}
```

在inject.xml(自定义的配置文件名)配置文件中配置如下：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="student" class="injection.Student">
        <property name="id" value="1"></property>
        <property name="name" value="小明"></property>
    </bean>

    <bean id="course" class="injection.Course">
        <property name="id" value="1"></property>
        <property name="name" value="数学"></property>
    </bean>

    <bean id="score" class="injection.Score">
        <property name="score" value="90"></property>
        <property name="student" ref="student"></property>
        <property name="course" ref="course"></property>
    </bean>
</beans>
```

输出：

```
小明 数学 : 90
```

在上面一个示例中，可以通过getBean()获得你想要的配置对象，具体可以是对象名字或对象类型，或者是名字和对象类型。bean：豆子，spring中，所有的对象都称之为bean，bean有很多的配置项内容，这些配置项使得spring的IoC配置方式变得非常丰富。

### 3. ClassPathXmlApplicationContext源码追溯浅析

在上示例中 new ClassPathXmlApplicationContext("injection/injection.xml")构造方法如下：

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
    }

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
        super(parent);
        this.setConfigLocations(configLocations);
        if (refresh) {
            this.refresh();
        }

    }
```

在该构造方法内部，最核心的就是refresh()方法总的来说，spring是由refresh方法来启动IoC容器初始化过程，主要包括三步：

```
1. Resource定位（Bean配置文件位置定位）
2. 将Resource定位好的资源载入到BeanDefinition
3. 将BeanDefiniton注册到容器中
```

refresh()方法内部结构：

```java
public void refresh() throws BeansException, IllegalStateException {
    Object var1 = this.startupShutdownMonitor;
    synchronized (this.startupShutdownMonitor) {  
           //1. 调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识  
           prepareRefresh();  
           //2. 内部主要调用refreshBeanFactory()方法创建了DefaultListableBeanFactory类型的BeanFactory实例并返回该实例对象
           ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
           //为BeanFactory配置容器特性，例如类加载器、事件处理器等  
           prepareBeanFactory(beanFactory);  
           try {  
               //为容器的某些子类指定特殊的BeanPost事件处理器  
               postProcessBeanFactory(beanFactory);  
               //调用所有注册的BeanFactoryPostProcessor的Bean  
               invokeBeanFactoryPostProcessors(beanFactory);  
               //为BeanFactory注册BeanPost事件处理器.  
               //BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件  
               registerBeanPostProcessors(beanFactory);  
               //初始化信息源，和国际化相关.  
               initMessageSource();  
               //初始化容器事件传播器.  
               initApplicationEventMulticaster();  
               //调用子类的某些特殊Bean初始化方法  
               onRefresh();  
               //为事件传播器注册事件监听器.  
               registerListeners();  
               //初始化所有单例模式的Bean.  
               finishBeanFactoryInitialization(beanFactory);  
               //初始化容器的生命周期事件处理器，并发布容器的生命周期事件  
               finishRefresh();  
           }  
           catch (BeansException ex) {  
               //销毁以创建的单态Bean  
               destroyBeans();  
               //取消refresh操作，重置容器的同步标识.  
               cancelRefresh(ex);  
               throw ex;  
           }  
       }  
}
```

在获取obtainFreshBeanFactory()里面的refreshBeanFactory()方法中完成的创建DefaultListableBeanFactory类型的BeanFactory实例，该方法定义在AbstractRefreshableApplicationContext类中，上面提到的IoC容器初始化过程的第1、2步就是在refreshBeanFactory方法中完成的：

```java
protected final void refreshBeanFactory() throws BeansException {
	//若的beanFactory已经存在，则先进行销毁
    if (this.hasBeanFactory()) {
        this.destroyBeans();
        this.closeBeanFactory();
    }

    try {
        //创建一个DefaultListableBeanFactory实例
        DefaultListableBeanFactory beanFactory = this.createBeanFactory();
        beanFactory.setSerializationId(this.getId());
        this.customizeBeanFactory(beanFactory);

        //定位resource和将对应的bean载入到beanDefinition中
        this.loadBeanDefinitions(beanFactory);
        Object var2 = this.beanFactoryMonitor;
        synchronized(this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException var5) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
    }
}
```

把重心放到loadBeanDefinitions()方法中，可以看到，该方法完成了配置资源的定位以及将bean载入到beanDefinitionMap中：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
	this.initBeanDefinitionReader(beanDefinitionReader);
	this.loadBeanDefinitions(beanDefinitionReader);
}
```

在loadDefinition阶段，将获取到的资源resource再封装成为EncodedResource(resource)实体，再通过InputStream流的方式将资源

我们来看一下初始化过程都做了啥：

### 3.1 Resource资源定位：

这个过程是定位资源配置文件的位置，由ResourceLoader通过统一的Resource接口实现，在创建Spring IoC容器时，通常是二进制、文件、网络等多种方式访问资源，Spring将这些文件统称为Resource，看一下常用的Resource资源：

```
FileSystemResource：以文件的绝对路径方式进行访问资源，效果类似于Java中的File;
ClassPathResourcee：以类路径的方式访问资源，效果类似于this.getClass().getResource("/").getPath();
ServletContextResource：web应用根目录的方式访问资源，效果类似于request.getServletContext().getRealPath("");
UrlResource：访问网络资源的实现类。例如file: http: ftp:等前缀的资源对象;
ByteArrayResource: 访问字节数组资源的实现类。
```

有了各种资源，不同的资源处理的方式不一样，ResourceLoader进行加载Resource需要进行不同的处理


XmlBeanDefinitionReader会根据输入的资源路径去对资源进行加载，

```java
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = this.getConfigResources();
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }

    }
```

首先会根据资源的路径进行判断资源文件的加载方式并生成对应的Resource实例，随后将Resource实例封装成EncodedResource实例以便于进行资源的编码处理，并以InputStream流形式读取Resource资源。


### 3.2 将资源载入到BeanDefinition

XmlBeanDefinitionReader内部定义的DefaultDocumentLoader类型的documentLoader会将Resource资源处理成Document实例对象，内部使用工厂模式得到文档构造器builder，builder将资源字节流进行转换

```java
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver, ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
	DocumentBuilderFactory factory = this.createDocumentBuilderFactory(validationMode, namespaceAware);
	......
	DocumentBuilder builder = this.createDocumentBuilder(factory, entityResolver, errorHandler);
	return builder.parse(inputSource);
}
```

在上一步得到doc，registerBeanDefinitions(doc, resource)这一步，也就是准备将Document中的Bean按照Spring bean语义进行解析并转化为BeanDefinition类型

### 3.3 将BeanDefinition注册到容器中，

BeanDefiniation通过载入和解析后，已经存在于IoC容器中了，但还不能使用，需要向IoC容器注册，上面解析的每一个bean会被封装成为一个BeanDefinitionHolder类型的对象，该对象里面包括bean的name、aliases[]以及对应的BeanDefinition结构，通过使用BeanDefinitionRegistry内部定义registerBeanDefinitions方法来进行注册操作，在DefaultListableBeanFactory中通过一个BeanDefinitionMap维护这些BeanDefiniation数据结构，并将aliases别名存到aliasMap，方便根据别名获取实例，具体如下：

```java
public static void registerBeanDefinition(BeanDefinitionHolder bdHolder, BeanDefinitionRegistry beanFactory) throws BeansException {

	// Register bean definition under primary name.
	String beanName = bdHolder.getBeanName();
　　　　　// 注册beanDefinition!!!
	beanFactory.registerBeanDefinition(beanName, bdHolder.getBeanDefinition());

	// 如果有别名的话也注册进去，Register aliases for bean name, if any.
	String[] aliases = bdHolder.getAliases();
	if (aliases != null) {
		for (int i = 0; i < aliases.length; i++) {
			beanFactory.registerAlias(beanName, aliases[i]);
		}
	}
}
```

在finishBeanFactoryInitialization()方法内部将所有在BeanDefinition中定义的结构进行对象实例化，并放在Map类型的singletonObjects对象中，该对象里面存的便是实例化对象的<name, 对象实体>映射集合。