### 1. 为什么要使用SpringBoot
- 相对于传统项目的优势，快速开发框架，快速整合第三方框架-Maven依赖关系、依赖继承、完全采用注解化、简化xml配置，内嵌http服务器（tomcat jetty）、最终以java应用程序执行，SpringBoot和微服务没有什么关系

### 2. SpringBoot和SpringCloud关系
- SpringCloud是一套完整的微服务框架 对标doubbo, 注册中心、客户端调用工具、服务治理（负载均衡 断路器 分布式配置中心、网关、服务链路
- SpringBoot和SpringCloud关系pringBoot是微服务框架吗？
  - 否，微服务通信技术 http+json SpringBoot Web组件默认继承了Spring MVC，SpringCloud依赖SpringBoot实现微服务、使用SpringMVC编写微服务接口， SpringBoot+SpringCloud实现微服务开发，
  - 核心区别：
### 3. SpringBoot和SpringMVC的关系
- SpringBoot的Web组件集成SpringMVC框架，但是SpringBoot启动SpringMVC的时候没有传统的配置文件？（SpringMVC3.0版本支持注解方式启动SpringMVC，相当于使用Java语言启动SpringMVC）

### 4. 防盗链技术

### 5. 伪静态html结合前端技术（当前前端框架下的前端渲染问题）
- Freemaker

### 6. 整合全局捕获异常

### 7. 考虑不同环境配置

### 8. SpringBoot事务
- 分类：声明，编程
- 事务原理： Aop通过环绕通知进行拦截，注意不要使用try，因为要将异常抛出给外层，事务的一些操作不会立马进行提交，直到整个方法执行结束

### 9. 多数据源
- 项目中能够有多少个不同的jdbc数据源连接？
- 多数据源如何划分：分包名||注解方式

### SpringBoot性能优化
- JVM参数调优
- Tomcat参数调优
- 扫包优化（开发扫包，启动）