# MySQL性能调优(3)查询优化详解


#### 查询执行路径
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/15.png"/>
##### MySQL 客户端/服务端通信
- MySQL客户端与服务端的通信方式是“半双工”;
- 客户端一旦开始发送消息，另一端要接收完整个消息才能响应。客户端一旦开始接收数据没法停下来发送指令。
- 对于一个MySQL连接，或者说一个线程，时刻都有一个状态来标识这个连接正在做什么
- 查看命令 show full processlist / show processlist
我正在通过navicat向虚拟机里面的数据库导入数据
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/16.png" />
```
Sleep 线程正在等待客户端发送数据
Query 连接线程正在执行查询
Locked 线程正在等待表锁的释放
Sorting result 线程正在对结果进行排序
Sending data 向请求端返回数据
可通过kill {id}的方式进行连接的杀掉
```
[官网状态全集](https://dev.mysql.com/doc/refman/8.0/en/general-thread-states.html)

##### 查询缓存
- 前话：为什么MySQL默认关闭了缓存开启？？
> <font color=red>MySQL 8.0不支持查询缓存，用户升级后将被鼓励使用服务器端查询重写或ProxySQL作为中间缓存。</font>

<!--more-->

###### 下面是了解缓存应该知道的知识：
- 工作原理：
> 缓存SELECT操作的结果集和SQL语句；
> 新的SELECT语句，先去查询缓存，判断是否存在可用的记录集；

- 判断标准：
> 与缓存的SQL语句，是否完全一样(包括语句内的空格)，区分大小写 (简单认为存储了一个key-value结构，key为sql，value为sql查询结果集)

- 不会缓存的情况
    - 当查询语句中有一些不确定的数据时，则不会被缓存。如包含函数NOW()，CURRENT_DATE()等类似的函数，或者用户自定义的函数，存储函数，用户变量等都不会被缓存
    - 当查询的结果大于query_cache_limit(查询缓存返回行数)设置的值时，结果不会被缓存
    - 对于InnoDB引擎来说，当一个语句在事务中修改了某个表，那么在这个事务提交之前，所有与这个表相关的查询都无法被缓存。因此长时间执行事务，会大大降低缓存命中率
    - 查询的表是系统表
    - 查询语句不涉及到表
    - 当前表在之前做过其他增删改，这个表中其他数据做的缓存都会失效、
    > 比如当前表中一行数据你做了缓存，表中其他数据更改了，也会影响这行数据的缓存失效。

#### 查询优化处理
1. 解析sql
> 通过lex词法分析,yacc语法分析将sql语句解析成解析树[学习连接](https://www.ibm.com/developerworks/cn/linux/sdk/lex/)
2. 预处理阶段
> 根据MySQL的语法的规则进一步检查解析树的合法性，如：检查数据的表和列是否存在，解析名字和别名的设置。还会进行权限的验证
3. 查询优化器
> 优化器的主要作用就是找到最优的执行计划
##### 如何找到最优执行计划
- 使用等价变化规则
> 5 = 5 and a > 5 改写成 a > 5
> a < b and a = 5 改写成 b > 5 and a = 5
> 基于联合索引，调整条件位置等
- 优化count 、min、max等函数
> min函数只需找索引最左边(因为innodb本身索引有序)
> max函数只需找索引最右边(因为innodb本身索引有序)
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/17.png"  />
> myisam引擎count(*)
- 覆盖索引扫描
> 比如你查询返回的字段命中了索引，他就会直接返回数据：这里我给uname建立了索引
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/18.png"  />
- 子查询优化（我们做优化的话：最好就是把子查询改成连接查询）
> 子查询合并。即把多个子查询合并成一个子查询，作用是减少元组扫描的次数
>子查询反嵌套。也称作子查询上拉，即将子查询重写为等价的多表连接
> 聚集子查询消除。即聚集函数上推，将消除聚集函数的子查询与父查询做外连接

- 提前终止查询
用了limit关键字(他就索引limit长度的数据)或者使用不存在的条件;
- IN的优化
>先进性排序，再采用二分查找的方式;
>在条件多的情况下，or还是会一个个去扫描判断，而in他会先找到中间的在判断;再根据大小判断。这样就会减少多余的扫描。

<font color= blue> MySQL的查询优化器是基于成本计算的原则。他会尝试各种执行计划。数据抽样的方式进行试验（随机的读取一个4K的数据块进行分析）</font>

#### 查询执行引擎
##### 执行计划顺序
select查询的序列号，标识执行的顺序
```
1. id相同是相当于一组，执行顺序由上至下
2. id不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
3. id相同又不同即两种情况同时存在，id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行
```


##### 查询类型select_type
```
查询的类型，主要是用于区分普通查询、联合查询、子查询等
SIMPLE：简单的select查询，查询中不包含子查询或者union
PRIMARY：查询中包含子部分，最外层查询则被标记为primary
SUBQUERY/MATERIALIZED：SUBQUERY表示在select 或 where列表中包含了子查询
MATERIALIZED表示where 后面in条件的子查询
UNION：若第二个select出现在union之后，则被标记为union
UNION RESULT：从union表获取结果的select
```


##### 执行计划table
查询涉及到的表,直接显示表名或者表的别名
```
<unionM,N> 由ID为M,N 查询union产生的结果
<subqueyN> 由ID为N查询生产的结果
```

##### 执行计划type
<font color=red>所以优化得至少在range及以上</font>
```
访问类型，sql查询优化中一个很重要的指标，结果值从好到坏依次是：system > const > eq_ref > ref > range > index > ALL
system：表只有一行记录（等于系统表），const类型的特例，基本不会出现，可以忽略不计
const：表示通过索引一次就找到了，const用于比较primary key 或者 unique索引
eq_ref：唯一索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描
ref：非唯一性索引扫描，返回匹配某个单独值的所有行，本质是也是一种索引访问
range：只检索给定<font color=red>范围</font>的行，使用一个索引来选择行
index：Full Index Scan，索引全表扫描，把索引从头到尾扫一遍
ALL：Full Table Scan，遍历全表以找到匹配的行
```

##### 其他关键字
```
possible_keys:查询过程中有可能用到的索引
key:实际使用的索引，如果为NULL，则没有使用索引 
rows:根据表统计信息或者索引选用情况，大致估算出找到所需的记录所需要读取的行数
filtered:它指返回结果的行占需要读到的行(rows列的值)的百分比表示返回结果的行数占需读取行数的百分比,filtered的值越大越好
```


##### 额外信息Extra
1. Using filesort ：
>MySQL对数据使用一个外部的文件内容进行了排序，而不是按照表内的索引进行排序读取
2. Using temporary：
>使用临时表保存中间结果，也就是说MySQL在对查询结果排序时使用了临时表，常见于order by 或 group by但是innodb用id排序分组是不会出现临时表的。
3. Using index：
>表示相应的select操作中使用了覆盖索引（Covering Index），避免了访问表的数据行，效率高
4. Using where ：
> 使用了where过滤条件
5. select tables optimized away：
>基于索引优化MIN/MAX操作或者MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段在进行计算，查询执行计划生成的阶段即可完成优化

##### 分析一个组合查询
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/19.png" />
```
这条语句就是把两个子查询查出来再结合
这里有个特殊情况：id等于null那个是最后执行的，因为他需要的到其他查询结果结合。
//--------------------------------
1. 这里就看id最大的4：select_type是subquery说明是子查询；type为ref非唯一性索引扫描；用到了id和address索引。
2. id为3的开始执行，select_type为union说明第二个select出现在union之后，则被标记为union；
3. id等于2和id等于1就是之前的子查询与查询；并且用到了主键索引。
4. 然后最后再把1和3合并；这里可以看出组合查询使用到了临时表。
```
#### 执行引擎
调用插件式的存储引擎的原子API的功能进行执行计划的执行
#### 返回客户端
增量的返回结果：
> 开始生成第一条结果时,MySQL就开始往请求方逐步返回数据
> 好处： MySQL服务器无须保存过多的数据，浪费内存用户体验好，马上就拿到了数据

#### 慢查询日志
```
// 查看是否打开慢查询日志
show variables like 'slow_query_log'
// 打开慢查询日志
set global slow_query_log = on
//  保存日志位置
set global slow_query_log_file ='/var/lib/mysql/gupaoedu-slow.log'
 // 没有命中索引的都保存
set global log_queries_not_using_indexes = on
// 超过多少秒记录日志
set global long_query_time = 0.1 (秒)
//查看是否开启
show variables like 'slow_query%';
```

##### 慢日志文件查看
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/20.png"  />

```
Time ：日志记录的时间
User@Host：执行的用户及主机
Query_time：查询耗费时间 Lock_time 锁表时间 Rows_sent发送给请求方的记录条数 Rows_examined 语句扫描的记录条数
SET timestamp 语句执行的时间点
select .... 执行的具体语句
```
##### MySQL自带的分析工具
```
mysqldumpslow -t 10 -s at /var/lib/mysql/gupaoedu-slow.log
```
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/21.png" />
