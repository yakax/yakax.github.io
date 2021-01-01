# Mybatis3源码分析(1)生命周期与核心配置解读以及批量操作


### 单独用mybatis编程式进行DB操作
```
--------------------利用mapper
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession();
try {
    BlogMapper mapper = session.getMapper(BlogMapper.class);
    Blog blog = mapper.selectBlogById(1);
    System.out.println(blog);
} finally {
    session.close();
}
--------------------利用mybatis的接口
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession(); // ExecutorType.BATCH
try {
    Blog blog = (Blog) session.selectOne("com.yakax.mapper.BlogMapper.selectBlogById", 1);
    System.out.println(blog);
} finally {
    session.close();
}
```
<!--more-->
<font color=red>这案例非常重要，后面分析源码还是基于它。</font>


### 生命周期
我们先从官网介绍开始一步步开始看,我们从每个对象的作用的角度来理解一下，只有理解了它们是干什么的，才知道什么时候应该创建，什么时候应该销毁。
#### SqlSessionFactoryBuiler
首先是SqlSessionFactoryBuiler。它是用来构建SqlSessionFactory的，而SqlSessionFactory只需要一个，所以只要构建了这一个SqlSessionFactory，它的使命就完成了，也就没有存在的意义了。所以它的生命周期只存在于<font color=red>方法的局部</font>。而SqlSessionFactoryBuilder一般都是配置文件构建SqlSessionFactory。
#### SqlSessionFactory
SqlSessionFactory 是用来创建 SqlSession 的，每次应用程序访问数据库，都需要创建一个会话。因为我们一直有创建会话的需要，所以 SqlSessionFactory 应该存在于应用的<font color=red>整个生命周期中（作用域是应用作用域）</font>。创建 SqlSession 只需要一个实例来做这件事就行了，否则会产生很多的混乱，和浪费资源。所以我们要采用单例模式
#### SqlSession
SqlSession 是一个会话，因为它不是线程安全的，不能在线程间共享。所以我们在请求开始的时候创建一个 SqlSession 对象，在请求结束或者说方法执行完毕的时候要及时关闭它<font color=red>（一次请求或者操作中）</font>。
#### Mapper
Mapper（实际上是一个代理对象）是从 SqlSession 中获取的。它的作用是发送 SQL来操作数据库的数据。它应该在一个<font color=red> SqlSession 事务方法之内</font>。
#### 总结生命周期
| 对象 | 生命周期 |
| ------ | ------ |
| SqlSessionFactoryBuiler | 方法局部（method） |
| SqlSessionFactory（单例） | 应用级别（application） |
| SqlSession | 请求和操作（request/method） |
| Mapper |  方法（method） |

### 核心配置解读
> 虽然说现在基本上都是springboot开发，但是我们学习mybatis还是得从他的配置文件开始学习。

#### configuration
configuration 是整个配置文件的根标签，实际上也对应着 MyBatis 里面最重要的配置类 Configuration。它贯穿 MyBatis 执行流程的每一个环节。我们打开这个类看一下，这里面有很多的属性，跟其他的子标签也能对应上。<font color=red>注意：MyBatis 全局配置文件顺序是固定的，否则启动的时候会报错。</font>
我们可以通过build进去找到XMLConfigBuilder在里面他会new一个Configuration；这个configuration就是我们配置类对应的类，并且我们可以在里面看到许多设置的默认值。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/17.png)
##### 属性（properties）
为了避免直接把参数写死在 xml 配置文件中，我们可以把这些参数单独放在properties 文件中，用 properties 标签引入进来，然后在 xml 配置文件中用${}引用就可以了。
```
<properties resource="db.properties"></properties>
这样也可以把数据库的信息分离出来；把配置文件放在外部，避免修改后重新编译打包，只需要重启应用；
```
##### 设置（settings）
这是 MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为。 描述了设置中各项的意图、默认值等。<font color=red>核心配置</font>。具体可以看官网[链接](http://www.mybatis.org/mybatis-3/zh/configuration.html)。主要的东西就是懒加载(通过代理用于关联查询)，缓存，查询执行器，日志。

##### 类型别名（typeAliases）
TypeAlias 是类型的别名，跟 Linux 系统里面的 alias 一样，主要用来简化全路径类名的拼写。比如我们的参数类型和返回值类型都可能会用到我们的 Bean，如果每个地方都配置全路径的话，那么内容就比较多，还可能会写错。我们可以为自己的 Bean 创建别名，既可以指定单个类，也可以指定一个 package，自动转换。配置了别名以后，只需要写别名就可以了，比如 com.yakax.Blog都只要写 blog 就可以了。我们也可以指定某个包下面，这样包下面的全部默认都简写了。MyBatis 里面有系统预先定义好的类型别名，也是在TypeAliasRegistry 中。
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/18.png)
##### 类型处理器（typeHandlers）【重点】
由于 Java 类型和数据库的 JDBC 类型不是一一对应的（比如 String 与 varchar），所以我们把 Java 对象转换为数据库的值，和把数据库的值转换成 Java 对象，需要经过一定的转换，这两个方向的转换就要用到 TypeHandler。这就是为什么数据库类型是varchar而mybatis 晓得对应string 类型一样。因为 MyBatis 已经内置了很多 TypeHandler（在mybatis源码 type 包下），它们全部全部注册在 TypeHandlerRegistry 中，他们都继承了抽象类 BaseTypeHandler，泛型就是要处理的 Java 数据类型。
###### 案例
在开发过程中，我们常常把一些易变动的数据，或者易变动的对象存储为json字符串到数据库中(MySQL从5.7开始就有json格式和json查询语法)，而以前的操作就是,我们查询出来利用string类型接收json字符串，然后在自己利用json解析转成对象封装到VO给前端；而typehandlers可以帮我们解决这一点。
数据库字段与数据(一个id，一个对象，一个list，[对象里面也存在list])
```
数据库
create table blogdo (id int not null,blog json null,blog_list json null);
数据
id=1;
blog={"bid": 1, "name": "yakax", "naList": [{"na": "123"}, {"na": "234"}], "authorId": 120};
blog_list=[{"bid": 1, "name": "yakax", "naList": [{"na": "123"}], "authorId": 120}, {"bid": 1, "name": "yakax", "naList": [{"na": "123"}], "authorId": 120}];
```

DO与json对象
```
这个是数据库查询转换后的对象
@Data
@TableName("blogdo")
public class BlogDO {
    private Integer id;
    private Blog blog;
    private List<Blog> blogList;
}
JSON 对象
@Data
public class Blog {
     private Integer bid;
    private String name;
    private Integer authorId;
    private List<Json> naList;
}
@Data
public class Json {
    private String na;
}
```

<font color=red>建立处理对象的TypeHandler</font>

```
@MappedTypes({Blog.class, List.class, Json.class})//要处理对应得Java类型
@MappedJdbcTypes(value = JdbcType.BLOB)//要对什么数据库类型进行操作
public class MyTypeHandlerone<E> extends BaseTypeHandler<E> {

    /**
     * 处理的类型(注册类型与传类型进来)
     */
    private Class<E> type;

    /**
     * 注册类型
     * @param type
     */
    public MyTypeHandlerone(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }
    
    /**
     * 设置非空参数
     */
    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, E t, JdbcType jdbcType) throws SQLException {
        // set进入数据库
        preparedStatement.setString(i, JSON.toJSONString(t));
    }

    /**
     * 获取空结果集（根据列名），一般都是调用这个
     */
    @Override
    public E getNullableResult(ResultSet resultSet, String s) throws SQLException {
        String sqlJson = resultSet.getString(s);
        if (null != sqlJson) {
            return JSONObject.parseObject(sqlJson, type);
        }
        return null;
    }

    /**
     * 获取空结果集（根据下标值）
     */
    @Override
    public E getNullableResult(ResultSet resultSet, int i) throws SQLException {
        String sqlJson = resultSet.getString(i);
        if (null != sqlJson) {
            return JSONObject.parseObject(sqlJson, type);
        }
        return null;
    }

    /**
     * 存储过程用的
     */
    @Override
    public E getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        String sqlJson = callableStatement.getString(i);
        if (null != sqlJson) {
            return JSONObject.parseObject(sqlJson, type);
        }
        return null;
    }
}
```
<font color=red>然后在配置中心增加type-handlers-package=包地址</font>

- 测试
```
  BlogDO blogdo = blogMapper.selectById(1);
  log.info(blogdo.toString());
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/23.png)
> 从上图中可以看出，typehandlers已经帮我们转化好数据了，注意看数据格式，<font color=red>如果是对象，那么对象下面注册了的东西他都会转化，如果是list，那么他存储对象是用JSONobject，存值是hashMap，存储list是用JSONArray</font>；

- 我们看看他是在什么时候怎么注册进入的
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/2.png)
- 程序启动时调用构造方法-然后进入TypeHandlerRegistry进行注册，
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/20.png)
- 我们可以看到这个类里面有很多预置的类型对应
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/21.png)
- 这里我们可以看到我们的JAVA类型已经注册进来了
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/19.png)
- 这里也看到他有专门的一个map来存储我们自定义的handler
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2019-06-04/22.png)
<font color=blue>这就是mybatis自定义typehandlers的查询流程-循环根据MappedTypes({Blog.class, List.class, Json.class})调用构造方法-然后注册到map中-然后根据@MappedJdbcTypes(value = JdbcType.BLOB)执行那些数据要返回指定的对象，然后通过JSON转换；新增操作原理相同调用的是自定义TypeHandler的setNonNullParameter方法转换成JSON字符串添加到数据库中;</font>

##### objectFactory（对象工厂）
MyBatis 每次创建结果对象的新实例时，它都会使用一个对象工厂（ObjectFactory）实例来完成。 默认的对象工厂需要做的仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。意思就是我们查询出来的数据映射给对象的时候我们可以从中修改了在返回出来。这个操作我用的比较少，反正理解为mybatis创建实例的工厂就行了。什么时候创建：返回结果集的时候创建。
> 摘自官网————ObjectFactory 接口很简单，它包含两个创建用的方法，一个是处理默认构造方法的，另外一个是处理带参数的构造方法的。 最后，setProperties 方法可以被用来配置 ObjectFactory，在初始化你的 ObjectFactory 实例后， objectFactory 元素体中定义的属性会被传递给 setProperties 方法。

##### 插件（plugins）
这是mybatis一个很强大的机制，他可以拦截mybatis的4大对象：我会在后面单独分析
> MyBatis 允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

{% raw %}  
<table>
  <tr>
    <th width=20%>类/接口</th>
    <th width=80%>方法</th>
  </tr>
  <tr>
    <td>Executor</td>
    <td>update, query, flushStatements, commit, rollback, getTransaction, close, isClosed</td>
  </tr>
  <tr>
     <td>ParameterHandler</td>
    <td>getParameterObject, setParameters</td>
  </tr>
    <tr>
     <td>ResultSetHandler</td>
    <td>handleResultSets, handleOutputParameters</td>
  </tr>
  <tr>
     <td>StatementHandler</td>
    <td>prepare, parameterize, batch, update, query</td>
  </tr>
</table>
{% endraw %}

##### 环境配置（environments）
<font color=red>一个 environment 标签就是一个数据源，代表一个数据库。这里面有两个关键的标签，一个是事务管理器，一个是数据源。</font>
这也就是Mybatis多数据源多重数据库配置的标签。

###### transactionManager
如果配置的是 JDBC，则会使用 Connection 对象的 commit()、rollback()、close()管理事务。
如果配置成 MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因为我们跑的是本地程序，如果配置成 MANAGE 不会有任何事务。
如果是Spring + MyBatis,则没有必要配置,因为我们会直接在上下文里面配置数据源，覆盖 MyBatis 的配置；用到spring 的事务来管理。

###### dataSource
我们项目一般是集成spring的所以这个到时候数据源和事务都会交给spring来管理。

##### 数据库厂商标识（databaseIdProvider）
为了支持多数据库

##### 映射器（mappers）
这是让mybatis启动的时候去扫描我们配置的mapper.xml文件。
```
cache – 给定命名空间的缓存配置（是否开启二级缓存）。
cache-ref – 其他命名空间缓存配置的引用。这两个标签我准备缓存的时候会详细写。
resultMap - 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。
sql – 可被其他语句引用的可重用语句块
insert – 映射插入语句
update – 映射更新语句
delete – 映射删除语句
select – 映射查询语句
```
resultMap与resultType
> 其实mybatis查询出来的数据都是通过map的方式赋值的，而resultType就是对应的类型；resultMap是我们自定义的类型，一般用于对象不确定或者对应字段需要自定义的时候。

##### 总结

{% raw %}  
<table>
  <tr>
    <th width=20%>配置名称</th>
    <th width=20%>配置含义</th>
    <th width=60%>配置简介</th>
  </tr>
  <tr>
    <td>configuration</td>
    <td>包裹所有配置标签</td>
    <td>整个配置文件的顶级标签</td>
  </tr>
  <tr>
    <td>properties</td>
    <td>属性</td>
    <td>该标签可以引入外部配置的属性，也可以自己配置。该配置标签所在的同一个配置文件的其他配置均可以引用此配置中的属性</td>
  </tr>
     <tr>
    <td>setting</td>
    <td>全局配置参数</td>
    <td>用来配置一些改变运行时行为的信息，例如是否使用缓存机制，是否使用延迟加载，是否使用错误处理机制等。</td>
  </tr>
   <tr>
    <td>typeAliases</td>
    <td>类型别名</td>
    <td>用来设置一些别名来代替 Java 的长类型声明（如 java.lang.int变为 int），减少配置编码的冗余</td>
  </tr>
     <tr>
    <td>typeHandlers</td>
    <td>类型处理器</td>
    <td>将数据库获取的值以合适的方式转换为 Java 类型，或者将 Java类型的参数转换为数据库对应的类型</td>
  </tr>
       <tr>
    <td>objectFactory</td>
    <td>对象工厂</td>
    <td>实例化目标类的工厂类配置</td>
  </tr>
<tr>
    <td>plugins</td>
    <td>插件</td>
    <td>可以通过插件修改 MyBatis 的核心行为，例如对语句执行的某一点进行拦截调用</td>
  </tr>
  <tr>
    <td>environments</td>
    <td>环境集合属性对象</td>
    <td>数据库环境信息的集合。在一个配置文件中，可以有多种数据库环境集合，这样可以使 MyBatis 将 SQL 同时映射至多个数据库</td>
  </tr>
  <tr>
    <td>environment</td>
    <td>环境子属性对象</td>
    <td>数据库环境配置的详细配置</td>
  </tr>
    <tr>
    <td>transactionManager</td>
    <td>事务管理</td>
    <td>指定 MyBatis 的事务管理器</td>
  </tr>
      <tr>
    <td>dataSource</td>
    <td>数据源</td>
    <td>使用其中的 type 指定数据源的连接类型，在标签对中可以使用property 属性指定数据库连接池的其他信息</td>
  </tr>
<tr>
    <td>mappers</td>
    <td>映射器</td>
    <td>配置 SQL 映射文件的位置，告知 MyBatis 去哪里加载 SQL 映射文件</td>
  </tr>
</table>
{% endraw %}

#### 动态SQL

```
## if 判断
## choose (when, otherwise)
SELECT * FROM tbl_emp e
<where>
    <choose>
        <when test="empId !=null">
            e.emp_id = #{emp_id, jdbcType=INTEGER}
        </when>
        <otherwise>
         AND e.email = #{email, jdbcType=VARCHAR}
        </otherwise>
    </choose>
</where>
类似于JAVA中的case when default
## trim (where, set) 
去处一些多余的符号的，比如拼接insert语句时最后一个逗号；
<trim prefix="(" suffix=")" suffixOverrides=",">
    <if test="deptId != null">
        dept_id,
    </if>
</trim>
foreach 循环
```

### 批量操作
开发中批量插入肯定不能使用java 的循环来插入:增删改查批量同理主要是sql的组装。
<font color=red>数据库默认接收数据包大小是4M(max_allowed_packet)所以如果批量过大，会报错</font>
####  foreach
```
insert into 表名(列名) values (插入的字段值),（插入的字段值），......
```

#### 使用Batch Executor查询
```
使用Batch Executor 需要开启setting里面的defaultExecutorType 里面有三个参数
也可以在会话 SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);
SIMPLE 默认 普通的sql 对应jdbc的statement
REUSE 执行器会重用预处理语句对应jdbc PreparedStatement ------也就是说我们执行的sql 他会先缓存起来，要用的时候通过缓存拿
BATCH 重用语句并执行批量更新
```
##### statement与PreparedStatement 的区别
1. PreparedStatement对象不仅包含了SQL语句，而且大多数情况下这个语句已经被预编译过，因而当其执行时，只需应用程序运行SQL语句，而不必先编译。加快了访问数据库的速度。这种转换也给你带来很大的便利，不必重复SQL语句的句法，而只需更改其中变量的值，便可重新执行SQL语句。
2. PreparedStatement接口代表预编译的语句，它主要的优势在于可以减少SQL的编译错误并增加SQL的安全性（减少SQL注射攻击的可能性）因为Statement 接口不接受参数， PreparedStatement 接口运行时接受输入的参数<font color=red>这也就是$与#的区别</font>
3. statement每次执行sql是需要编译的。
