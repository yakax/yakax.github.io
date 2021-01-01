# Mybatis3源码分析(6)简单手写思路及面试题


手写基本流程
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis6/1.png)

<!--more-->

#### 流程
1. 定义接口 Mapper 和方法，用来调用数据库操作。Mapper 接口操作数据库需要通过代理类。
2. 定义配置类对象 Configuration。
3. 定义应用层的 API SqlSession。它有一个 getMapper()方法，我们会从配置类Configuration 里面使用 Proxy.newProxyInatance()拿到一个代理对象MapperProxy。
4. 有了代理对象 MapperProxy 之后，我们调用接口的任意方法，就是调用代理对象的 invoke()方法。
5. 代理对象 MapperProxy 的 invoke()方法调用了 SqlSession 的 selectOne()。
6. SqlSession 只是一个 API，还不是真正的 SQL 执行者，所以接下来会调用执行器 Executor 的 query()方法。
7. 执行器 Executor 的 query()方法里面就是对 JDBC 底层的 Statement 的封装，最终实现对数据库的操作，和结果的返回。
附上[手写Mybatis](https://github.com/yakax/handwriteMybatis.git) github地址
> ps:只是简单的理解加深印象，里面有很多不足。

#### 面试分析
1. resultType 和 resultMap 的区别？
```
resultType 是<select>标签的一个属性，适合简单对象(POJO、JDK 自带类型：Integer、String、Map 等)只能自动映射，适合单表简单查询。
resultMap 是一个可以被引用的标签，适合复杂对象，可指定映射关系，适合关联复合查询。
```
2. 标签collection 和 标签association 的区别？
```
association：一对一
collection：一对多、多对多
```
3. PrepareStatement 和 Statement 的区别？
```
两个都是接口,PrepareStatement 是继承自 Statement 的；
Statement 处理静态 SQL，PreparedStatement 主要用于执行带参数的语句；
PreparedStatement 的 addBatch()方法一次性发送多个查询给数据库；
PS 相似 SQL 只编译一次（对语句进行了缓存，相当于一个函数），减少编译次数；
PS 可以防止 SQL 注入；
MyBatis 默认值：PREPARED 也就是PreparedStatement
```
4. MyBatis 解决了什么问题？
```
1) 资源管理（底层对象封装和支持数据源）
2）结果集自动映射
3）SQL 与代码分离，集中管理
4）参数映射和动态 SQL
5）其他：缓存、插件等
```
5. MyBatis 编程式开发中的核心对象及其作用？
```
SqlSessionFactoryBuilder 创建工厂类
SqlSessionFactory 创建会话
SqlSession 提供操作接口
MapperProxy 代理 Mapper 接口后，用于找到 SQL 执行
```
6. Java 类型和数据库类型怎么实现相互映射？
```
通过 TypeHandler，例如 Java 类型中的 String 要保存成 varchar，就会自动调用相应的 Handler。如果没有系统自带的 TypeHandler，也可以自定义--比如之前文章中写的json转对象的操作。
```
7. SIMPLE/REUSE/BATCH 三种执行器的区别？
```
SimpleExecutor 使用后直接关闭 Statement：closeStatement(stmt);
ReuseExecutor 放在缓存中，可复用：PrepareStatement:getStatement();
BatchExecutor 支持复用且可以批量执行 update()，通过 ps.addBatch()实现handler.batch(stmt);
```
8. MyBatis 一级缓存与二级缓存的区别？
```
一级缓存：在同一个会话（SqlSession）中共享，默认开启，维护在 BaseExecutor中。
二级缓存：在同一个 namespace 共享，需要在 Mapper.xml 中开启，维护在CachingExecutor 中。
```
9. MyBaits 支持哪些数据源类型？
```
UNPOOLED：不带连接池的数据源。
POOLED ： 带 连 接 池 的 数 据 源 ， 在 PooledDataSource 中 维 护PooledConnection。
JNDI：使用容器的数据源，比如 Tomcat 配置了 C3P0。
自定义数据源：实现 DataSourceFactory 接口，返回一个 DataSource。当 MyBatis 集成到 Spring 中的时候，使用 Spring 的数据源
```
10. 关联查询的延迟加载是怎么实现的？
```
动态代理（JAVASSIST、CGLIB），在创建实体类对象时进行代理，在调用代理对象的相关方法时触发二次查询。
```
11. MyBatis 翻页的几种方式和区别？
```
逻辑翻页：通过 RowBounds 对象。
物理翻页：通过改写 SQL 语句，可用插件拦截 Executor 实现。
```
12. 解析全局配置文件的时候，做了什么？
```
创建 Configuration，设置 Configuration
解析 Mapper.xml，设置 MappedStatement
```
13. 没有实现类，MyBatis 的方法是怎么执行的？
```
MapperProxy 代理，代理类的 invoke()方法中调用了 SqlSession.selectOne()
```
14. 接口方法和映射器的 statement id 是怎么绑定起来的？（怎么根据接口方法拿到 SQL 语句的？）
```
MappedStatement 对象中存储了 statement 和 SQL 的映射关系
```
15. 四大对象是什么时候创建的？
```
Executor：openSession()
StatementHandler、ResultsetHandler、ParameterHandler：
执行 SQL 时，在 SimpleExecutor 的 doQuery()中创建
```
16. MyBatis 哪些地方用到了代理模式？
```
接口查找 SQL：MapperProxy
日志输出：ConnectionLogger、StatementLogger
连接池：PooledDataSource 管理的 PooledConnection
延迟加载：ProxyFactory（JAVASSIST、CGLIB）
插件：Plugin
Spring 集成：SqlSessionTemplate 的内部类 SqlSessionInterceptor
```
17. MyBatis 插件怎么编写和使用？原理是什么?
```
使用：继承 Interceptor 接口，加上注解，在 mybatis-config.xml 中配置
原理：动态代理，责任链模式，使用 Plugin 创建代理对象
在被拦截对象的方法调用的时候，先走到 Plugin 的 invoke()方法，再走到Interceptor 实现类的 intercept()方法，
最后通过 Invocation.proceed()方法调用被拦截对象的原方法
```
18. JDK 动态代理，代理能不能被代理？
```
能。层层代理先进后出
```
19. MyBatis 集成到 Spring 的原理是什么？
```
SqlSessionTemplate 中有内部类SqlSessionInterceptor对DefaultSqlSession进行代理；
MapperFactoryBean 继 承 了 SqlSessionDaoSupport 获 取SqlSessionTemplate；
接口注册到 IOC 容器中的 beanClass 是 MapperFactoryBean。
```
20. DefaulSqlSession 和 SqlSessionTemplate 的区别是什么？
```
一个线程安全一个线程不安全
1）为什么 SqlSessionTemplate 是线程安全的？
其内部类 SqlSessionInterceptor 的 invoke()方法中的 getSqlSession()方法：如果当前线程已经有存在的 
SqlSession 对象，会在 ThreadLocal 的容器中拿到SqlSessionHolder，获取 DefaultSqlSession。
如果没有，则会 new 一个 SqlSession，并且绑定到 SqlSessionHolder，放到ThreadLocal 中。SqlSessionTemplate 
中在同一个事务中使用同一个 SqlSession。调用 closeSqlSession()关闭会话时，如果存在事务，减少 holder 的引用计数。否则直接关闭 SqlSession。
2) 在编程式的开发中，有什么方法保证 SqlSession 的线程安全？
SqlSessionManager 同时实现了 SqlSessionFactory、SqlSession 接口，通过ThreadLocal 容器维护 SqlSession。
```










