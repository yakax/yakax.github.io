# Mybatis3源码分析(3)流程走向


> 分析源码我们还是从编程式demo入手
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/0.png)

<!--more-->
### 我们通过建造者模式创建一个工厂类，配置文件的解析就是在这一步完成的，包括 mybatis-config.xml 和 Mapper 适配器文件。
1. 首先进入build
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/1.png)
2. 进入XMLconfigBuilder--这里就是配置文件创建的地方
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/2.png)
3. 可以看到这里有很多解析文件的类
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/3.png)
4. 解析节点--这里可以看出文件只解析了一次。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/4.png)
解析文件还是比较简单的，基本上就是读取文件的内容然后做存储设置，我们来看看时序图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/5.jpg)

### DefaultSqlSession--最主要的就是创建了一个执行器
1. 创建回话时拿到全局配置文件的配置，并且创建了事务工厂和创建执行器。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/6.png)
2. 对执行器做判断--两次判断的原因是因为怕有人把默认执行器设置为空--这里也判断了是否开启了二级缓存的--<font color=red>这里注意这个额插件调用方法先把他理解为拦截器进行了一次包装</font>
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/7.png)
3. 我在baseExecutor里面还看到了模板模式--把具体的增删改查的方法都交给子类去做了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/8.png)
先看看继承关系
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/9.png)
看看时序图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/10.jpg)

### getMapper
1. 为什么要引入mapper对象------为了解决这种硬编码的问题
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/11.png)
一步步走
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/12.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/14.png)
注意这个注册器-点进去看
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/13.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/15.png)
这里的代码就有点熟悉了吧。<font color=red>jdk的动态代理</font>那么作为代理类一定有个规范，我们进去看
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/16.png)
果然实现了invocationHandler
<font color=red>说明mapper对象是一个代理对象</font>
2. 为什么我们只需要mapper接口不需要去实现呢
目前来看，他的作用就是标识我们去找到对应的statementID找到sql语句他的功能也就完成了。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/17.jpg)

### MapperProxy
1. 既然是代理对象肯定会走到上图中的invoke方法执行到mapperMethod.exeute中
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/19.png)
为什么会判断defaultMethod方法的原因是jdk1.8接口都是可以有defaultMethod方法的，这里的这个cache是为了提升获取mapper的效率---这里调用了一个computeIfAbsent map的方法构建本地缓存
2. 判断类型准备执行的语句类型
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/18.png)
3. 走到selectOne
为什么用list接收：我觉得这是一种比较前瞻的写法吧，下面那个报错应该都很熟悉，以前把查出来 的list用对象接收就是这个错误
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/20.png)
4. 这里可以看到传进来的是一个statementid的位置字符串，具体怎么找到的就在MappedStatement这行代码解决的。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/21.png)
5. 走到query 的实现方法 --我们进入二级缓存这个实现方法
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/22.png)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/23.png)
这里的这个缓存key 就是判断我们缓存是否命中的。<font color=red>判断statementid和翻页信息和sql是否一样</font>
6. 继续走query-他会有缓存走缓存没缓存走查询
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/24.png)
这里是判断语句设置的一些参数做操作
7. 走到queryFromDatabase --这样就到了baseExecutor
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/25.png)
8. 接下来就是基本执行器的操作了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/26.png)
9. 走到默认的simple执行器
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/27.png)
在这里感觉已经离真相越来越近了..这里已经看到了jdbc 的statement了。
10. 根据这个newStatementHandler进入--这里就有我们熟悉的东西赋值了。
newParameterHandler 处理参数
newResultSetHandler 处理结果集
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/28.png)
11. 四大天王--能被插件拦截的四大天王
<font color=red>ParameterHandler，ResultSetHandler，StatementHandler，executor</font>
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/29.png)
12. 然后对参数进行预编译--然后执行查询--这里已经调用--然后处理结果集
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/30.png)
已经调用ps.execute了这已经是jdbc的操作了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Mybatis3/31.jpg)





