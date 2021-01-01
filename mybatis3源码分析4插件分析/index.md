# Mybatis3源码分析(4)插件分析


### 源码总结回顾
{% raw %} 
<table>
  <tr>
    <th >对象</th>
    <th >相关对象</th>
    <th >作用</th>
  </tr>
  <tr>
    <td>Configuration</td>
    <td>MapperRegistry 
    TypeAliasRegistry
    TypeHandlerRegistry</td>
        <td>包含了 MyBatis 的所有的配置信息</td>
  </tr>
  <tr>
      <td>SqlSession</td>
    <td>SqlSessionFactory
DefaultSqlSession</td>
        <td>对操作数据库的增删改查的 API 进行了封装，提供给应用层使用</td>
  </tr>
    <tr>
    <td>Executor</td>
    <td>BaseExecutor
SimpleExecutor
BatchExecutor
ReuseExecutor</td>
        <td>MyBatis 执行器，是 MyBatis 调度的核心，负责 SQL 语句的生成和查
询缓存的维护</td>
  </tr>
  <tr>
    <td>StatementHandler</td>
    <td>BaseStatementHandler
SimpleStatementHandler
PreparedStatementHandler
CallableStatementHandler</td>
        <td>封装了 JDBC Statement 操作，负责对 JDBC statement 的操作，如设
置参数、将 Statement 结果集转换成 List 集合</td>
  </tr>
   <tr>
    <td>ParameterHandler</td>
    <td>DefaultParameterHandler</td>
        <td>把用户传递的参数转换成 JDBC Statement 所需要的参数</td>
  </tr>
   <tr>
    <td>ResultSetHandler</td>
    <td>DefaultResultSetHandler</td>
        <td>把 JDBC 返回的 ResultSet 结果集对象转换成 List 类型的集合</td>
  </tr>
   <tr>
    <td>MapperProxy</td>
    <td>MapperProxyFactory</td>
        <td>代理对象，用于代理 Mapper 接口方法</td>
  </tr>
   <tr>
    <td>MappedStatement</td>
    <td>SqlSource
BoundSql</td>
        <td>MappedStatement 维护了一条select|update|delete|insert节点
的封装，包括了 SQL 信息、入参信息、出参信息</td>
  </tr>
</table>
{% endraw %}

### 插件拦截的四大对象
{% raw %} 
<table>
  <tr>
    <th >对象</th>
    <th >描述</th>
    <th >可拦截的方法</th>
     <th >方法作用</th>
  </tr>
  <tr>
    <td  rowspan="8">Executor</td>
    <td  rowspan="8">上层的对象，SQL 执行全过程，包括组装参数，组装结果集返回和执行 SQL 过程</td>
    <td>update</td>
    <td>执行 update、insert、delete 操作</td>
  </tr>
<tr>
    <td>query</td>
    <td>执行 query 操作</td>
</tr>
   <tr>
    <td>flushStatements</td>
    <td>在 commit 的时候自动调用，SimpleExecutor
ReuseExecutor、BatchExecutor 处理不同</td>
</tr>
<tr>
    <td>commit</td>
    <td>提交事务</td>
</tr>
<tr>
    <td>rollback</td>
    <td>事务回滚</td>
</tr>
<tr>
    <td>getTransaction</td>
    <td>获取事务</td>
</tr>
<tr>
    <td>close</td>
    <td>结束（关闭）事务</td>
</tr>
<tr>
    <td>isClosed</td>
    <td>判断事务是否关闭</td>
</tr>
<tr>
    <td  rowspan="5">StatementHandler</td>
    <td  rowspan="5">上层的对象，SQL 执行全过程，包括组装参数，组装结果集返回和执行 SQL 过程</td>
    <td>prepare</td>
    <td>（BaseSatementHandler）SQL 预编译</td>
</tr>
<tr>
    <td>parameterize</td>
    <td>设置参数</td>
</tr>
<tr>
    <td>batch</td>
    <td>批处理</td>
</tr>
<tr>
    <td>update</td>
    <td>增删改操作</td>
</tr>
<tr>
    <td>query</td>
    <td>查询操作</td>
</tr>
<tr>
<td rowspan="2">ParameterHandler</td>
<td rowspan="2">SQL 参数组装的过程</td>
<td >getParameterObject</td>
<td >获取参数</td>
</tr>
<tr>
<td >setParameters</td>
<td >设置参数</td>
</tr>
<tr>
<td rowspan="2">ResultSetHandler</td>
<td rowspan="2">结果的组装</td>
<td >handleResultSets</td>
<td >处理结果集</td>
</tr>
<tr>
<td >handleOutputParameters</td>
<td >处理存储过程出参</td>
</tr>
 
</table>
{% endraw %}

<!--more-->
### 代理和拦截是怎么实现的?
我们从之前的课程知道插件是通过拦截时通过jdk动态代理的。

#### 四大对象什么时候被代理，也就是：代理对象是什么时候创建的？
我们还记得Executor上个文章里面讲到，实在openSession时创建的，StatementHandler是SimpleExecutor.doQuery()里面创建的，里面包含了处理参数的 ParameterHandler 和处理结果集的 ResultSetHandler 的创建，创建之后即调用InterceptorChain.pluginAll(),返回层层代理后的对象。

#### 多个插件的情况下，代理能不能被代理？代理顺序和调用顺序的关系？
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/1.png)

#### 谁来创建代理对象？
我们可以看从pagehelper这个插件进入看看-首先我们先从pageHelper的注册类进去
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/2.png)
可以看到实现了interceptor这个类----这样就形成了代理类
下面这个方法就是mybatis的操作创建插件的方法：这个之后会分析
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/3.png)



#### 被代理后，调用的是什么方法？怎么调用到原被代理对象的方法？
我们可以看到在intercept方法里面拿到执行器是调用的gettarget这个方法，利用代理类调用的，我们进入invocation看看
可以看到可以通过target方法拿到被代理对象
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/4.png)
因为代理类是 Plugin，所以最后调用的是 Plugin 的 invoke()方法。它先调用了定义的拦截器的 intercept()方法。可以通过 invocation.proceed()调用到被代理对象被拦截的方法。

#### 再来看看他是怎么对我们的天王代理的
 通过openSession 进去找到创建executor的时候 他调用了我们放在Configuration里面的<font color=red>interceptorChain</font>
 ![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/5.png)
 可以看到调用了所有插件-拿到所有实现了Interceptor的拦截器，然后这里就调用了实现Interceptor的类的plugin这个方法进行创建代理插件代理对象。这样被代理后我们之后的操作都会走到代理类的invoke方法里面。
 ![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/6.png)
 ![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/7.png)
 这样就会调用我们自定义的拦截器的intercept方法--之后里面就是我们自定已的逻辑操作了。

#### 实现Interceptor的三个方法含义
intercept():做我们想要做的事情 可以通过invocation 这个参数获取被拦截对象方法参数或者执行代理方法 --调用proceed可以走到被拦截对象的被拦截方法，这样就走完整个流程了。
plugin(): 创建代理对象，通过代理对象的invoke方法走到我们的插件
setProperties():这是我们在配置我们插件时可以提供很多属性，比如分页插件设置一些参数等等。

### pageHelper插件案例分析
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/8.png)
他是怎么来改写我们的sql的呢--我们首先来看看PageInterceptor里面的自定义逻辑
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/9.png)
根据方言进入不同的实现类
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/10.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/11.png)
然后我们在看看分页参数哪里来的
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/12.png)
可以看到这里startPage调用时传进来的，然后通过本地线程里面拿到的（每个线程里面拿到的都是属于自己独有的分页数据）
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/13.png)
这也就是写了startPage那句话，就会修改我们的sql 分页--<font color=red>这也就出现一个问题，在一个线程里我们的查询分页就有不稳定性！</font>

| 对象 | 作用 | 
| ------ | ------ | 
| PageInterceptor | 自定义拦截器 | 
| Page | 包装分页参数 |
| PageInfo | 包装结果 | 
| PageHelper | 工具类 |

### mybatis插件到底可以干些什么
1. 动态改变数据源。水平分表，通过注解的方式：可以通过invocation拿到方法的注解，在接口上添加注解，通过反射获取接口注解，根据注解上配置的参数进行分表，修改原 SQL，例如 id 取模，按月分表
2. 记录日志。
3. 数据权限---对不同用户查询出来的数据有些筛选
4. 脱敏也是可以的，数据加解密
5. 对数据查询的执行分析，比如执行时长的分析。

### 插件调用的流程图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/14.jpg)

### 上一个简单的sql执行时长分析插件
type: 表示拦截的类，这里是StatementHandler的实现类
method：表示拦截的方法，这里是拦截StatementHandler的query方法
args：表示方法参数(可以通过数组参数接收然后强转)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/15.png)
看看pagehelper的
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/17.png)
SpirngBoot注册插件
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mybatis3plugin/16.png)




