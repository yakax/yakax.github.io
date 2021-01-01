# Mybatis3源码分析(2)体系结构与缓存


### 工作流程分析
- 首先在 MyBatis 启动的时候我们要去解析配置文件，包括全局配置文件和映射器配置文件，这里面包含了我们怎么控制 MyBatis 的行为，和我们要对数据库下达的指令，也就是我们的 SQL 信息。我们会把它们解析成一个 Configuration 对象。
- 接下来就是我们操作数据库的接口，它在应用程序和数据库中间，代表我们跟数据库之间的一次连接：这个就是 SqlSession 对象。
- 我们要获得一个会话，必须有一个会话工厂SqlSessionFactory。SqlSessionFactory 里面又必须包含我们的所有的配置信息，所以我们会通过一个Builder 来创建工厂类。
- 我们知道，MyBatis 是对 JDBC 的封装，也就是意味着底层一定会出现 JDBC 的一些核心对象，比如执行 SQL 的 Statement，结果集 ResultSet。在 Mybatis 里面，SqlSession 只是提供给应用的一个接口，还不是 SQL 的真正的执行对象。
- 我们上次课提到了，SqlSession 持有了一个 Executor 对象，用来封装对数据库的操作。在执行器 Executor 执行 query 或者 update 操作的时候我们创建一系列的对象，来处理参数、执行 SQL、处理结果集，这里我们把它简化成一个对象：StatementHandler，
在阅读源码的时候我们再去了解还有什么其他的对象。
> 流程图

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/1.png)

<!--more-->

### MyBatis 架构分层与模块划分
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/3.png)

```
跟Spring 一样，MyBatis 按照功能职责的不同，所有的 package可以分成不同的工作层次。
我们可以把 MyBatis 的工作流程类比成餐厅的服务流程。
第一个是跟客户打交道的服务员，它是用来接收程序的工作指令的，我们把它叫做接口层。
第二个是后台的厨师，他们根据客户的点菜单，把原材料加工成成品，然后传到窗口。这一层是真正去操作数据的，我们把它叫做核心层。
最后就是餐厅也需要有人做后勤（比如清洁、采购、财务），来支持厨师的工作和整个餐厅的运营。我们把它叫做基础层。
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/4.png)

#### 接口层
首先接口层是我们打交道最多的。核心对象是 SqlSession，它是上层应用和 MyBatis打交道的桥梁，SqlSession 上定义了非常多的对数据库的操作方法。接口层在接收到调用请求的时候，会调用核心处理层的相应模块来完成具体的数据库操作。

#### 核心处理层
接下来是核心处理层。既然叫核心处理层，也就是跟数据库操作相关的动作都是在这一层完成的。核心处理层主要做了这几件事：
1. 把接口中传入的参数解析并且映射成 JDBC 类型；
2. 解析 xml 文件中的 SQL 语句，包括插入参数，和动态 SQL 的生成；
3. 执行 SQL 语句；
4. 处理结果集，并映射成 Java 对象
> <font color=red>插件也属于核心层，这是由它的工作方式和拦截的对象决定的。</font>

#### 基础支持层
最后一个就是基础支持层。基础支持层主要是一些抽取出来的通用的功能（实现复用），用来支持核心处理层的功能。比如数据源、缓存、日志、xml 解析、反射、IO、事务等等这些功能。

### MyBatis 缓存详解
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/5.png)

从上图中我们可以看出Cache的默认实现只有PerpetualCache；不过mybatis利用了<font color=red>装饰器模式</font>来扩展的缓存接口：上面包decorators下面的东西就是装饰器。我们也可以自定义缓存类，比如用redis来实现Cache这个类也是可以的，但是都是必须实现Cache这个接口。

#### 所有的缓存实现类总体上可分为三类：基本缓存、淘汰算法缓存、装饰器缓存。
参数的话我一般是使用注解@CacheNamespace到mapper上面也可以配置在Cache标签里面

{% raw %}
<table>
  <tr>
    <th width=20%>缓存实现类</th>
    <th width=20%>描述</th>
    <th width=40%>作用</th>
    <th width=20%>装饰条件</th>
  </tr>
 <tr>
    <td>基本缓存</td>
    <td>缓存基本实现类</td>
    <td>默认是 PerpetualCache,也可以自定义比如RedisCache、EhCache 等，具备基本功能的缓存类</td>
    <td>无</td>
  </tr>
  <tr>
    <td>LruCache</td>
    <td>LRU 策略的缓存</td>
    <td>当缓存到达上限时候，删除最近最少使用的缓存（Least Recently Use）</td>
    <td>eviction="LRU"（默认）</td>
  </tr>
  <tr>
    <td>FifoCache</td>
    <td>FIFO 策略的缓存</td>
    <td>当缓存到达上限时候，删除最先入队的缓存</td>
    <td>eviction="FIFO"</td>
  </tr>  
  <tr>
    <td>SoftCache、WeakCache</td>
    <td>带清理策略的缓存</td>
    <td>通过 JVM 的软引用和弱引用来实现缓存，当 JVM内存不足时，会自动清理掉这些缓存，基于SoftReference 和 WeakReference</td>
    <td>eviction="SOFT" eviction="WEAK"</td>
  </tr>   
   <tr>
    <td>LoggingCache</td>
    <td>带日志功能的缓存</td>
    <td>比如：输出缓存命中率</td>
    <td>基本</td>
  </tr> 
   <tr>
    <td>SynchronizedCache</td>
    <td>同步缓存</td>
    <td>基于 synchronized 关键字实现，解决并发问题</td>
    <td>基本</td>
  </tr>   
   <tr>
    <td>BlockingCache</td>
    <td>阻塞缓存</td>
    <td>通过在 get/put 方式中加锁，保证只有一个线程操作缓存，基于 Java 重入锁实现</td>
    <td>blocking=true</td>
  </tr>
   <tr>
    <td>SerializedCache</td>
    <td>支持序列化的缓存</td>
    <td>将对象序列化以后存到缓存中，取出时反序列化 </td>
    <td>readOnly=false（默认）</td>
  </tr>  
   <tr>
    <td>ScheduledCache</td>
    <td>定时调度的缓存</td>
    <td>在进行 get/put/remove/getSize 等操作前，判断缓存时间是否超过了设置的最长缓存时间（默认是一小时），如果是则清空缓存--即每隔一段时间清空一次缓存</td>
    <td>flushInterval 不为空</td>
  </tr>  
  <tr>
    <td>TransactionalCache</td>
    <td>事务缓存</td>
    <td>在二级缓存中使用，可一次存入多个缓存，移除多个缓存</td>
    <td>在TransactionalCacheManager 中用 Map维护对应关系</td>
  </tr>    
</table>
{% endraw %}

可以看出缓存是用map存储的
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/6.png)

#### 一级缓存
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/7.png)

>一级缓存也叫本地缓存，MyBatis的一级缓存是在会话（SqlSession）层面进行缓存的。MyBatis 的一级缓存是默认开启的，不需要任何的配置。
>禁用缓存可以使用@Options注解也可以使用在mapper.xml文件里面的flushCache

1. 我们知道，缓存最终会放到PerpetualCache中。
2. sqlSession默认实现是defaultSession，然而defaultSession里面有个配置文件和执行器，那么执行器又是BaseExecutor维护，缓存就应该在BaseExecutor里面
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/8.png)

##### 一级缓存验证
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/9.png)
> 执行更新操作后缓存会失效

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/10.png)
> 其他会话更新了数据，导致读取到脏数据（一级缓存不能跨会话共享）

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/11.png)


#### 二级缓存
<font color=red>二级缓存是Namespace级别的，也就是mapper级别的查询会缓存，增删改会刷新缓存，所以开启是针对mapper配置就行了。我一般用@CacheNamespace或者@CacheNamespaceRef(代表引用别的命名空间的Cache配置，两个命名空间的操作使用的是同一个Cache，这个可以解决连表查询导致缓存失效，但是缓存粒度就变大了,刷新缓存会影响多个mapper)</font>

二级缓存是用来解决一级缓存不能跨会话共享的问题的，可以被多个 SqlSession 共享（只要是同一个接口里面的相同方法，都可以共享），生命周期和应用同步。

<font color=red>二级缓存是工作在一级缓存之前的</font>。实际上 MyBatis 用了一个装饰器的类来维护，就是 CachingExecutor。如果启用了二级缓存，MyBatis 在创建 Executor 对象的时候会对 Executor 进行装饰。CachingExecutor 对于查询请求，会判断二级缓存是否有缓存结果，如果有就直接返回，如果没有委派交给真正的查询器 Executor 实现类，比如 SimpleExecutor 来执行查询，再走到一级缓存的流程。最后会把结果缓存起来，并且返回给用户。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/12.png)

##### 二级缓存验证
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/13.png)
可以看到已经开启了二级缓存了，共享了sqlsession了。从上图中，我们可以看出，缓存是通过用户提交后才缓存起来的。
MyBatis二级缓存的工作流程和前文提到的一级缓存类似，只是在一级缓存处理前，用CachingExecutor装饰了BaseExecutor的子类，实现了二级缓存的查询和写入功能。

通过执行器进入CachingExecutor打的debug(<font color=red>可以看出一层层装饰最后才到基本实现类</font>)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/14.png)

缓存是通过TransactionalCacheManager存储的
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/15.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/16.png)
存储TransactionalCache这个对象 这里就已经说明了，可以执行多个操作然后在提交缓存。支持多个缓存的添加与删除。

#### 总结
##### 一级缓存
1. MyBatis一级缓存的生命周期和SqlSession一致。
2. MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。
3. MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement，这样也就失去了缓存的意义。

##### 二级缓存
1. MyBatis的二级缓存相对于一级缓存来说，实现了SqlSession之间缓存数据的共享，同时粒度更加的细，能够到namespace级别，通过Cache接口实现类不同的组合，对Cache的可控性也更强。
2. MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
3. 在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本而mybatis官网也提供了很多第三方缓存的方案,但是也是比较鸡肋的。
4. 直接使用Redis、Memcached等分布式缓存可能成本更低，安全性也更高。实际上，一般开发中，要用真正的缓存来提高效率，目前还是单独的第三方缓存方案比较可行。











