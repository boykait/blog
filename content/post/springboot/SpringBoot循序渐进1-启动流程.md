---
title: SpringBoot循序渐进1-启动流程
categories:
 - SpringBoot
tags:
 - SpringBoot循序渐进
---

> SpringBoot是目前用于构建后端项目比较热门的框架技术，能够帮助开发人员快速整合第三方框架，内置了http服务器（默认tomcat），无需繁琐的配置文件，能够以java应用程序启动运行，本小文主要就SpringBoot的启动过程进行浅析。

#### 启动流程
&emsp;&emsp;看一个简洁版的启动类：

```java
@SpringBootApplication 
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

### 1. SpringBootApplication注解

### 2. 走进SpringApplication.run
&emsp;&emsp;SpringApplication.run里面做了很多事情，其实最主要的也就是创建上下文并调用Spring本身的IOC进行对象的生命周期的管理和控制，当前，SpringBoot Http服务器的配置和启动也是在这一阶段完成的。调用SpringApplication的静态方法run内部其实主要分为两大步骤：实例化SpringApplication对象以及执行run方法。

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		...
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
#### 2.1 webApplicationType
&emsp;&emsp;webApplicationType用于确定web应用类型，主要根据web应用定义了三种类型：

- `REACTIVE` 应用程序应作为响应式Web应用程序运行，并应启动嵌入式响应式Web服务器。    
- `SERVLET` 应用程序应作为基于servlet的Web应用程序运行，并应启动嵌入式servlet Web服务器。
-  `NONE` 应用程序不应作为Web应用程序运行，也不应启动嵌入式Web服务器。 

&emsp;&emsp;可以通过setWebApplicationType或者配置spring.main.web-application-type=servlet|none|reactive来设置web应用类型，后面再run阶段根据不同的web应用类型以生成对应的上下文（后面会进行说明）。
#### 2.2 设置初始化器
&emsp;&emsp;接下来介绍的初始化操作主要是从定义在spring-boot-autoconfigure的`META-INF/spring.factories`配置文件中进行匹配，匹配什么呢，主要是匹配以`ApplicationContextInitializer`结尾的应用上下文初始化器，通过指定getSpringFactoriesInstances的参数为ApplicationContextInitializer.class：    

```java
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
org.springframework.boot.context.ContextIdApplicationContextInitializer
org.springframework.boot.context.config.DelegatingApplicationContextInitializer
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```
&emsp;&emsp;指定getSpringFactoriesInstances的参数类型为ApplicationListener.class

### 3 SpringBoot.run()过程
&emsp;&emsp;在初始化`new SpringBootApplication`后	，第二个核心步骤就是执行run方法，run方法返回ConfigurableApplicationContext上下文实例对象，先整体梳理一下该过程的执行逻辑。

```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        // 创建SpringApplicationRunListeners对象实例，同样是从spring.factories配置文件中进行匹配
		SpringApplicationRunListeners listeners = getRunListeners(args);
        // 启动所有监听器
		listeners.starting();
		try {
			// 参数
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			// 准备容器环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
            // 设置需要忽略的bean
			configureIgnoreBeanInfo(environment);
            // 打印banner
			Banner printedBanner = printBanner(environment);
            // 创建应用容器
			context = createApplicationContext();
            // 实例化SpringBootExceptionReporter.class，用来支持报告关于启动的错误
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
            // 准备容器
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
            // 刷新容器
			refreshContext(context);
            // 刷新容器后的扩展接口
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

#### 3.1 创建SpringApplicationRunListeners
&emsp;&emsp；SpringApplicationRunListeners监听器列表目前只包含一个监听类型，即EventPublishingRunListner，主要用于监听run阶段各个生命周期的执行动作。

#### 3.2 环境准备
&emsp;&emsp; prepareEnvironment环境准备阶段主要是根据之前的WebApplicationType来实例化相应的环境
```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
         // StandardServletEnvironment || StandardReactiveWebEnvironment || StandardEnvironment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
        // 配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```
- getOrCreateEnvironment： 根据前的WebApplicationType来实例化相应的环境（SandardServletEnvironment || StandardReactiveWebEnvironment || StandardEnvironment），以实例化SandardServletEnvironment为例，SandardServletEnvironment继承关系如下：   
  ![继承关系](images/190908-springboot_start_1.png)    
  &emsp;&emsp;这个过程会调用AbstractEvironment中定义的customizePropertySources设置PropertySources: servletContextInitParams、servletConfigInitParams、systemEnvironment以及systemProperties
- configureEnvironment：配置PropertySources和Profiles
