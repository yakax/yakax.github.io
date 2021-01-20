# MySQL-一些问题




#### 优化器未使用你想用的索引

```mysql
select * from t force index(a) where a between 10000 and 20000;
// 利用参数force index
```
<!--more-->

#### 前缀索引

```
直接创建完整索引，这样可能比较占用空间；
创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引；
```

#### count区别

> **MyISAM索引结构的优势count快就不说了而Innodb由于MVCC与可重复读的原因速度会相对慢点**

count(*)、count(主键id)和count(1) 都表示返回满足条件的结果集的总行数；而count(字段），则表示返回满足条件的数据行里面，参数“字段”不为NULL的总个数

**对于count(主键id)来说**，InnoDB引擎会遍历整张表，把每一行的id值都取出来，返回给server层。server层拿到id后，**判断是不可能为空的，就按行累加**。

**对于count(1)来说**，InnoDB引擎遍历整张表，但不取值。server层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。

单看这两个用法的差别的话，你能对比出来，count(1)执行得要比count(主键id)快。因为从引擎返回id会涉及到解析数据行，以及拷贝字段值的操作。

**对于count(字段)来说**：

1. 如果这个“字段”是定义为not null的话，一行行地从记录里面读出这个字段，判断不能为null，按行累加；
   1. 如果这个“字段”定义允许为null，那么执行的时候，判断到有可能是null，还要把值取出来再判断一下，不是null才累加。

**但是count(\*)是例外**，并不会把全部字段取出来，而是专门做了优化，不取值。count(*)肯定不是null，按行累加

所以结论是：按照效率排序的话，`count(字段)<count(主键id)<count(1)≈count(*)，所以我建议你，尽量使用count(*)`



#### Order By 排序流程

##### 内存排序

当Extra这个字段出现`Using filesort`表示的就是需要排序。

```mysql
select city,name,age from t where city='杭州' order by name limit 1000  ;
通常情况下，这个语句执行流程如下所示 ：
1. 初始化sort_buffer，确定放入name、city、age这三个字段；
2. 从索引city找到第一个满足city='杭州’条件的主键id，也就是图中的ID_X；
3. 到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4直到city的值不满足查询条件为止。
6. 对sort_buffer中的数据按照字段name做快速排序；
7. 按照排序结果取前1000行返回给客户端。
```

按name排序”可能在内存中完成，当参数`sort_buffer_size`内存不够时，会使用临时文件辅助排序。

`max_length_for_sort_data` 是MySQL中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL就认为单行太大，要换一个算法。city、name、age 这三个字段的定义总长度是36

```
新的算法放入sort_buffer的字段，只有要排序的列（即name字段）和主键id。
但这时，排序的结果就因为少了city和age字段的值，不能直接返回了，整个执行流程就变成如下所示的样子：
1. 初始化sort_buffer，确定放入两个字段，即name和id；
2. 从索引city找到第一个满足city='杭州’条件的主键id；
3. 到主键id索引取出整行，取name、id这两个字段，存入sort_buffer中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4直到不满足city='杭州’条件为止。；
6. 对sort_buffer中的数据按照字段name进行排序；
7. 遍历排序结果，取前1000行，并按照id的值回到原表中取出city、name和age三个字段返回给客户端。
```

##### 联合索引排序

并不是所有的order by语句，都需要排序操作的。从上面分析的执行过程，我们可以看到，MySQL之所以需要生成临时表，并且在临时表上做排序操作，其原因都是无序的，如果我们建立联合索引，保证天然有序就好了

```
1. 从索引(city,name)找到第一个满足city='杭州’条件的主键id；
2. 到主键id索引取出整行，取name、city、age三个字段的值，作为结果集的一部分直接返回；
3. 从索引(city,name)取下一个记录主键id；
4. 重复步骤2、3，直到查到第1000条记录，或者是不满足city='杭州’条件时循环结束。
```



#### Group By 分组优化总结

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/group by.png!print" style="zoom: 33%;" />

```apl
Using index，表示这个语句使用了覆盖索引，选择了索引a，不需要回表
Using temporary，表示使用了临时表；
Using filesort，表示需要排序。
“order by null”和“不加order by”不等价
这个语句的执行流程是这样的：
1. 创建内存临时表，表里有两个字段m和c，主键是m；
2. 扫描表t1的索引a，依次取出叶子节点上的id值，计算id%10的结果，记为x
   1. 如果临时表中没有主键为x的行，就插入一个记录(x,1);
   2. 如果表中有主键为x的行，就将x这一行的c值加1；
3. 遍历完成后，再根据字段m做排序，得到结果集返回给客户端。  
```

1. 如果对group by语句的结果没有排序要求，要在语句后面加 **order by null；**

2. 尽量让group by过程用上表的索引，确认方法是explain结果里没有Using temporary 和 Using filesort；**能用索引肯定更好**

3. 如果group by需要统计的数据量不大，尽量只使用内存临时表；也可以通过适当调大`tmp_table_size`临时表大小参数，来避免用
    到磁盘临时表；(group by语句中需要放到临时表上的数据量特别大，就会“先放到内存临时表，插入一部分数据后，发现内存临时表不够用了再转成磁盘临时表”)

4. 如果数据量实在太大，使用SQL_BIG_RESULT这个提示直接使用磁盘临时表，少去内存转临时的过程，来告诉优化器直接使用排序算法得到group by的结果。 

   `select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;`



> **其实DISTINCT 跟group by 逻辑基本一样。也是采用临时表**
>
> 1. 创建一个临时表，临时表有一个字段a，并且在这个字段a上创建一个唯一索引；
> 2. 遍历表t，依次取数据插入临时表中：
> 如果发现唯一键冲突，就跳过；
> 否则插入成功；
> 3. 遍历完成后，将临时表作为结果集返回给客户端。

#### 隐式转换

```mysql
1.查询的时候注意字段隐式转换
select * from tradelog where tradeid=110717;
交易编号tradeid这个字段上，本来就有索引，但是explain的结果却显示，这条语句需要走全表扫描。
tradeid的字段类型是varchar(32)，而输入的参数却是整型，所以需要做类型转换。
2.连接查询时主要表与表之前的编码问题。
```



#### 大表扫描200G数据问题

**server层**采用的是边读边发的操作。

```
1. 获取一行，写到net_buffer中。这块内存的大小是由参数`net_buffer_length`定义的，默认是16k。
2. 重复获取行，直到net_buffer写满，调用网络接口发出去。
3. 如果发送成功，就清空net_buffer，然后继续取下一行，并写入net_buffer。
4. 如果发送函数返回EAGAIN或WSAEWOULDBLOCK，表示本地（socket send buffer）写满了,进入等待。直到网络栈重新可写，再继续发送。
```

1. 一个查询在发送过程中，占用的MySQL内部的内存最大就是net_buffer_length这么大，并不会达到200G；
2. socket send buffer 也不可能达到200G（默认定义/proc/sys/net/core/wmem_default），如果socket send buffer被写
满，就会暂停读数据的流程。

**InnoDB**引擎处理

> 由于冷热数据链的原因，大表扫描并不会影响young区域，只会占用old区域。



由于MySQL采用的是边算边发的逻辑，因此对于数据量很大的查询结果来说，不会在server端保存完整的结果集。所以，
**如果客户端读结果不及时，会堵住MySQL的查询过程，但是不会把内存打爆**。
而对于InnoDB引擎内部，由于有淘汰策略，大查询也不会导致内存暴涨。并且，由于InnoDB对LRU算法做了改进，冷数
据的全表扫描，**对Buffer Pool的影响也能做到可控**。
当然，全表扫描还是比较耗费IO资源的，所以业务高峰期还是不能直接在线上主库执行全表扫描。



#### 刷脏抖动问题

```
刷redolog日志到磁盘导致抖动问题
由于刷盘时在执行sql 会导致sql执行时间偶尔变长，
优化mysql 的iops 参数：innodb_io_capacity:它会告诉InnoDB你的磁盘能力。这个值我建议你设置成磁盘的IOPS。磁盘的IOPS可以通过fio这个工具来测试
如果是SSD同时设置innodb_flush_neighbors为0，让他每次别刷临近缓存页，减少刷缓存页的数量，这样就可以把刷缓存页的性能提升到最高
同时还需要关注innodb_max_dirty_pages_pct是脏页比例。
```



#### RAID阵列的定期放电导致抖动问题

写脚本自动在凌晨低峰时自动放电

#### 连接数不够

先看看 linux 本身的连接数 再看 mysql 的连接数






