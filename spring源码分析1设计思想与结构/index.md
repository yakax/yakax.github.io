# Spring5源码分析(1)设计思想与结构



```
源码地址(带有中文注解)git@github.com:yakax/spring-framework-5.0.2.RELEASE--.git 
```
#### Spring 的设计初衷其实就是为了简化我们的开发
1. 基于 POJO 的轻量级和最小侵入性编程；
2. 通过依赖注入和面向接口松耦合；
3. 基于切面和惯性进行声明式编程；
4. 通过切面和模板减少样板式代码；

<font color=red>**而他主要是通过：面向 Bean(BOP)、依赖注入（DI）以及面向切面（AOP）这三种方式来达成的。**</font>
<!--more-->

#### 面向Bean
- Spring 是面向Bean的编程(Bean Oriented Programming, BOP),Bean 在 Spring 中才是真正的主角。
- IOC(控制反转)：控制反转最常用的实现方式就是通过DI(依赖注入)所以在 Spring 中控制反转也被直接称作依赖注入。
> 在早期的spring中是有DL(依赖查找)的，后来由于使用频率过低就被移除了。

#### 依赖注入(DI)的基本概念
Spring 设计的核心 org.springframework.beans 包（架构核心是 org.springframework.core
包），它的设计目标是与 JavaBean 组件一起使用。这个包通常不是由用户直接使用，而是由服务器将
其用作其他多数功能的底层中介。下一个最高级抽象是 BeanFactory 接口，它是工厂设计模式的实现，
允许通过名称创建和检索对象。BeanFactory 也可以管理对象之间的关系。
BeanFactory 最底层支持两个对象模型。
1. 单例：提供了具有特定名称的全局共享实例对象，可以在查询时对其进行检索。Singleton 是默
认的也是最常用的对象模型。
2. 原型：确保每次检索都会创建单独的实例对象。在每个用户都需要自己的对象时，采用原型模式。
Bean 工厂的概念是 Spring 作为 IOC 容器的基础。IOC 则将处理事情的责任从应用程序代码转移到
框架。

#### AOP 编程理念
面向切面编程，即 AOP，是一种编程思想，它允许程序员对横切关注点或横切典型的职责分界线的行为（例如日志和事务管理）进行模块化。AOP 的核心构造是方面（切面），它将那些影响多个类的行为封装到可重用的模块中,AOP 的功能完全集成到了 Spring 事务管理、日志和其他各种特性的上下文中。

#### Spring5 系统架构
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/spring/1.png" />

##### 核心容器
- spring-core 和 spring-beans 
>Spring 框架的核心模块，包含了控制反转（Inversion ofControl, IOC）和依赖注入（Dependency Injection, DI）。BeanFactory 接口是 Spring 框架中的核心接口，它是工厂模式的具体实现。BeanFactory 使用控制反转对应用程序的配置和依赖性规范与实际的应用程序代码进行了分离。但 BeanFactory 容器实例化后并不会自动实例化 Bean，只有当 Bean 被使用时 BeanFactory 容器才会对该 Bean 进行实例化与依赖关系的装配。
- spring-context
>核心模块之上，他扩展了 BeanFactory，为她添加了 Bean 生命周期控制、框架事件体系以及资源加载透明化等功能。此外该模块还提供了许多企业级支持，如邮件访问、远程访问、任务调度等，ApplicationContext 是该模块的核心接口，她是 BeanFactory 的超类，与
BeanFactory 不同，ApplicationContext 容器实例化后会自动对所有的单实例 Bean 进行实例化与依赖关系的装配，使之处于待用状态。
- spring-context-support 
> 对 Spring IOC 容器的扩展支持，以及 IOC 子容器。
- spring-context-indexer
> Spring 的类管理组件和 Classpath 扫描。
- spring-expression
> 统一表达式语言（EL）的扩展模块，可以查询、管理运行中的对象，同时也方便的可以调用对象方法、操作数组、集合等。它的语法类似于传统 EL，但提供了额外的功能，最出色的要数函数调用和简单字符串的模板函数。这种语言的特性是基于 Spring 产品的需求而设计，他可以非常方便地同 Spring IOC 进行交互。

##### AOP 和设备支持
- spring-aop
> Spring 的另一个核心模块，是 AOP 主要的实现模块。作为继 OOP 后，对程序员影响最大的编程思想之一，AOP 极大地开拓了人们对于编程的思路。在 Spring 中，他是以 JVM 的动态代理技术为基础，然后设计出了一系列的 AOP 横切实现，比如前置通知、返回通知、异常通知等，同时，Pointcut 接口来匹配切入点，可以使用现有的切入点来设计横切面，也可以扩展相关方法根据需求进行切入。
- spring-aspects
> 集成自 AspectJ 框架，主要是为 Spring AOP 提供多种 AOP 实现方法
- spring-instrument
> 是基于 JAVA SE 中的"java.lang.instrument"进行设计的，应该算是 AOP的一个支援模块，主要作用是在 JVM 启用时，生成一个代理类，程序员通过代理类在运行时修改类的字节，从而改变一个类的功能，实现 AOP 的功能。在分类里，我把他分在了 AOP 模块下，在 Spring 官方文档里对这个地方也有点含糊不清，这里是纯个人看法。

##### 数据访问与集成
- spring-jdbc
> Spring 提供的 JDBC 抽象框架的主要实现模块，用于简化 Spring JDBC 操作 。主要是提供 JDBC 模板方式、关系数据库对象化方式、SimpleJdbc 方式、事务管理来简化 JDBC 编程，主要实现类是 JdbcTemplate、SimpleJdbcTemplate 以及 NamedParameterJdbcTemplate。
- spring-tx
> Spring JDBC 事务控制实现模块。使用 Spring 框架，它对事务做了很好的封装，通过它的 AOP 配置，可以灵活的配置在任何一层；但是在很多的需求和应用，直接使用 JDBC 事务控制还是有其优势的。其实，事务是以业务逻辑为基础的；一个完整的业务应该对应业务层里的一个方法；如果业务操作失败，则整个事务回滚；所以，事务控制是绝对应该放在业务层的；但是，持久层的设计则应该遵循一个很重要的原则：保证操作的原子性，即持久层里的每个方法都应该是不可以分割的。所以，在使用 Spring JDBC 事务控制时，应该注意其特殊性。
- spring-orm
> ORM 框架支持模块，主要集成 Hibernate, Java Persistence API (JPA) 和Java Data Objects (JDO) 用于资源管理、数据访问对象(DAO)的实现和事务策略。
- spring-oxm
> 主要提供一个抽象层以支撑 OXM（OXM 是 Object-to-XML-Mapping 的缩写，它是一个 O/M-mapper，将 java 对象映射成 XML 数据，或者将 XML 数据映射成 java 对象），例如：JAXB, Castor, XMLBeans, JiBX 和 XStream 等。
- spring-jms
>（Java Messaging Service）能够发送和接收信息，自 Spring Framework 4.1 以后，他还提供了对 spring-messaging 模块的支撑。

##### Web组件
-spring-web 
> Spring 提供了最基础 Web 支持，主要建立于核心容器之上，通过 Servlet 或者 Listeners 来初始化 IOC 容器，也包含一些与 Web 相关的支持。
- spring-webmvc 模块
> 众所周知是一个的 Web-Servlet 模块，实现了 Spring MVC（model-view-Controller）的 Web 应用。spring-websocket 模块主要是与 Web 前端的全双工通讯的协议。
- spring-web 
> 是一个新的非堵塞函数式 Reactive Web 框架，可以用来建立异步的，非阻塞，事件驱动的服务，并且扩展性非常好

##### 通信报文
- spring-messaging 模块
> 从 Spring4 开始新加入的一个模块，主要职责是为 Spring 框架集成一些基础的报文传送应用。
##### 集成测试
- spring-test
> 主要为测试提供支持的，毕竟在不需要发布（程序）到你的应用服务器或者连接到其他企业设施的情况下能够执行一些集成测试或者其他测试对于任何企业都是非常重要的。
##### 集成兼容
- spring-framework-bom
> Bill of Materials.解决 Spring 的不同模块依赖版本不同问题

##### 各模块关系图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/spring/2.png"  />

#### Spring 版本命名规则
{% raw %}  
<table>
  <tr>
    <th width=10%>描述方式</th>
    <th width=10%>说明</th>
    <th>含义</th>
  </tr>
  <tr>
    <td>Snapshot</td>
    <td>快照版</td>
    <td>尚不不稳定、尚处于开发中的版本</td>
  </tr>
  <tr>
     <td>Release</td>
    <td>稳定版</td>
    <td>功能相对稳定，可以对外发行，但有时间限制</td>
  </tr>
    <tr>
     <td>GA</td>
    <td>正式版</td>
    <td>代表广泛可用的稳定版(General Availability)</td>
  </tr>
  <tr>
     <td>M</td>
    <td>里程碑版</td>
    <td> (M 是 Milestone 的意思）具有一些全新的功能或是具有里程碑意义的版本。</td>
  </tr>
  <tr>
     <td>RC</td>
    <td>终测版</td>
    <td> Release Candidate（最终测试），即将作为正式版发布。</td>
  </tr>
</table>
{% endraw %}

