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
