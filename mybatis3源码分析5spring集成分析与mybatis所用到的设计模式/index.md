# Mybatis3源码分析(5)spring集成分析与mybatis所用到的设计模式


这里我们以传统的 Spring 为例，因为配置更直观，在 Spring 中使用配置类注解是一样的。在前面文章里面，我基于编程式的工程已经弄清楚了 MyBatis 的工作流程、核心模块和底层原理。编程式的工程，也就是 MyBatis 的原生 API 里面有三个核心对象：
**SqlSessionFactory、SqlSession、MapperProxy**
大部分时候我们不会在项目中单独使用 MyBatis 的工程，而是集成到 Spring 里面使用，但是却没有看到这三个对象在代码里面的出现。我们直接注入了一个 Mapper 接口，调用它的方法。
所以有几个关键的问题，我要弄清楚：
1.  SqlSessionFactory 是什么时候创建的？
2. SqlSession 去哪里了？为什么不用它来 getMapper？
3. 为什么@Autowired 注入一个接口，在使用的时候却变成了代理对象？在 IOC的容器里面我们注入的是什么？ 注入的时候发生了什么事情？
<!--more-->

#### 关键配置
引入mybatis-spring的整合jar包
然后在 Spring 的 applicationContext.xml 里面配置 SqlSessionFactoryBean，它是用来帮助我们创建会话的，其中还要指定全局配置文件和 mapper 映射器文件的路径
```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
   <property name="configLocation" value="classpath:mybatis-config.xml"></property>
   <property name="mapperLocations" value="classpath:mapper/*.xml"></property>
   <property name="dataSource" ref="dataSource"/>
</bean>
```
然后在 applicationContext.xml 配置需要扫描 Mapper 接口的路径。在 Mybatis 里面有几种方式，第一种是配置一个 MapperScannerConfigurer。
```
<bean id="mapperScanner" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="basePackage" value="com.gupaoedu.crud.dao"/>
</bean>
第二种是配置一个<scan>标签：
<mybatis-spring:scan base-package="com.gupaoedu.crud.dao"/>
第三种就是直接用@MapperScan 注解，比如我们在 Spring Boot 的启动类上加上一个注解：
@SpringBootApplication
@MapperScan("com.gupaoedu.crud.dao")
public class MybaitsApp {
    public static void main(String[] args) {
    SpringApplication.run(MybaitsApp.class, args);
    }
}
```

#### 创建会话工厂
先记住实现了这几个接口
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/1.png)
这个方法是spring IOC容器里面，对象设置属性完成后，可以调用的方法
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/2.png)
这个是从ioc容器里面拿到这个对象的时候就会调用这个getobject方法另外两个是获取信息的。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/3.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/4.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/5.png)
很熟悉的配置文件解析
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/6.png)
最后通过parse进入
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/7.png) 
现在已经是mybatis的操作了。在mybatis 的包里面了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/8.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/9.png)
**也就是说spring 在创建SqlSessionFactoryBean的时候会通过getobject方法解析对应配置文件创建config设置属性**
最后调用了build方法 
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/10.png)
这个和编程式mybatis单独使用调用的默认的DefaultSqlSessionFactory一样

#### 获得会话
先看看DefaultSqlSessionFactory 我看到官方给的一个注释
```
Note that this class is not Thread-Safe
```
实际上我们用DefaultSqlSessionFactory的时候他就是单例的，在多个service之间共享，那一定就会存在<font color=red>线程安全问题</font>。
我们看看mybatis-spring包提供的实现类：进入这个类我就看见注释上就写着
```
Thread safe, Spring managed, {@code SqlSession} that works with Spring
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11.png)
为什么线程安全呢：我们随便找个方法对比
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/15.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/12.png)
这个sqlsessionProxy一看就有鬼:这个jdk动态代理又来了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/CXUIT%40ZAUVKNM%40%24Y~%604ZK4H.png)
然后我们进入被代理类SqlSessionInterceptor看看
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/5.png)
**我一步步进入发现是存放在一个threadLocal里面也就是我们获取到的每个DefaultSqlSession都是一个单独的-->也就是一个事务一个sqlsession所以是线程安全的**
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/6.png)
我继续看他是怎么管理的-->这里就是代理的DefaultSqlSession的-->并且创建了一个事务管理器
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/16.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/7.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/8.png)
我发现只是减少次数没有关闭，之前我们看到，是通过一个事务管理器创建的这个容器；也就是说在我们同一个事务里面可以使用相同的会话，然后每次都减少，直到只有一个的时候就关闭会话
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/4E%7BE%5DL%5BS8%243OUTTID46VRZR.png)
我们知道在 Spring 里面会用 SqlSessionTemplate 替换 DefaultSqlSession，那么接下来看一下怎么在DAO层拿到一个SqlSessionTemplate。MyBatis 里面，它提供了一个 SqlSessionDaoSupport，里面持有一个SqlSessionTemplate 对象，并且提供了一个 getSqlSession()方法，让我们获得一个SqlSessionTemplate。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/13.png)
也就是说我们让 DAO 层的实现类继承 SqlSessionDaoSupport，就可以获得SqlSessionTemplate，然后在里面封装 SqlSessionTemplate 的方法。<font color=red>也就是说很多封装好的增删改的插件基本上就是继承这个SqlSessionDaoSupport</font>早期的ibatis也就是这样来的。
{% raw %} 
<table>
  <tr>
    <th width=27%>对象</th>
    <th >生命周期</th>
  </tr>
  <tr>
    <td>SqlSessionTemplate</td>
    <td>Spring 中 SqlSession 的替代品，是线程安全的，通过代理的方式调用 DefaultSqlSession 的方法</td>
  </tr>
 <tr>
    <td>SqlSessionInterceptor（内部类）</td>
    <td>代理对象，用来代理 DefaultSqlSession，在 SqlSessionTemplate 中使用</td>
  </tr>
   <tr>
    <td>SqlSessionDaoSupport</td>
    <td>用于获取 SqlSessionTemplate，只要继承它即可</td>
  </tr>
   <tr>
    <td>MapperFactoryBean</td>
    <td>注册到 IOC 容器中替换接口类，继承了 SqlSessionDaoSupport 用来获取SqlSessionTemplate，因为注入接口的时候，就会调用它的 getObject()方法</td>
  </tr>
     <tr>
    <td>SqlSessionHolder</td>
    <td>控制 SqlSession 和事务</td>
  </tr>
</table>
{% endraw %}


#### 接口扫描注册
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/14.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/1.png)
在MapperScannerConfigurer一步步找到这个方法 在注册时并没有注册这个类本身的beanclass，而是注册了mapperFactoryBean 我们看看官方注释说明
```
the mapper interface is the original class of the bean
but, the actual class of the bean is MapperFactoryBean
```

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/2.png)
那我们去看看mapperFactorBean-->果然继承了SqlSessionDaoSupport
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/3.png)
**所以：通过spring IOC 扫描注测接口本身的时候替换成了beanclass替换成了MapperFactoryBean这样也就可以拿到SqlSessionTemplate**

| 接口 | 方法 | 作用 |
| ------ | ------ | ------ |
| FactoryBean | getObject() | 返回由 FactoryBean 创建的 Bean 实例 |
| InitializingBean | afterPropertiesSet() | bean 属性初始化完成后添加操作 |
| BeanDefinitionRegistryPostProcessor | postProcessBeanDefinitionRegistry() | 注入 BeanDefination 时添加操作 |

    
#### 注入使用
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3spring/11/4.png)
<font color=red>在spring 注入初始化时mapper作为有注解的属性，这个时候会调用MapperFactoryBean-->然后调用getObject方法（因为他继承了SqlSessionDaoSupport所以能拿到SqlSessionTemplate）->然后就可以拿到我们的mapper代理对象（因为实现了 SqlSession）然后就可以调用mapper的数据库方法了</font>
#### 设计模式总结
{% raw %} 
<table>
  <tr>
    <th width=27%>设计模式</th>
    <th >类</th>
  </tr>
  <tr>
    <td>工厂</td>
    <td>SqlSessionFactory、ObjectFactory、MapperProxyFactory</td>
  </tr>
 <tr>
    <td>建造者</td>
    <td> XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuidler</td>
  </tr>
   <tr>
    <td>单例模式</td>
    <td> SqlSessionFactory、Configuration、ErrorContext</td>
  </tr>
   <tr>
    <td >代理模式</td>
    <td>绑定：MapperProxy <br>
        延迟加载：ProxyFactory（CGLIB、JAVASSIT）<br>
        插件：Plugin<br>
        Spring 集成 MyBaits：SqlSessionTemplate 的内部类 SqlSessionInterceptor<br>
        MyBatis 自带连接池：PooledDataSource 管理的 PooledConnection<br>
        日志打印：ConnectionLogger、StatementLogger</td>
  </tr>
<tr>
    <td>适配器模式</td>
    <td>logging 模块，对于 Log4j、JDK logging 这些没有直接实现 slf4j 接口的日志组件，需要适配器</td>
  </tr>
<tr>
    <td>模板方法</td>
    <td>BaseExecutor 与子类 SimpleExecutor、BatchExecutor、ReuseExecutor</td>
  </tr>
 <tr>
    <td>装饰器模式</td>
    <td>LoggingCache、LruCache 等对 PerpectualCache 的装饰 CachingExecutor 对其他 Executor 的装饰</td>
  </tr>
 <tr>
    <td>责任链模式</td>
    <td>InterceptorChain</td>
  </tr>
   <tr>
    <td>策略模式</td>
    <td>RoutingStatementHandler</td>
  </tr>
</table>
{% endraw %}






