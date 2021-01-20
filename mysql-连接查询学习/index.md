# MySQL-连接查询学习


## 连接查询的基本语法

```mysql
INNER JOIN  ## 内连接和外连接的根本区别就是在驱动表中的记录不符合ON子句中的连接条件时不会把该记录加入到最后的结果集
LEFT JOIN  ## 右边未匹配的null展示
RIGHT JOIN   ## 左边未匹配的null展示
```
<!--more-->
## 连接的原理

需要知道的知识

> 对于两表连接来说，驱动表只会被访问一遍，但被驱动表却要被访问到好多遍，来匹配记录。
>
> **如果我们自己用代码来做连接查询，用不到索引走数去查询被驱动表，性能肯定完全没有mysql那么好**
>
> **在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表我们应该尽量使用小表做驱动表，这样尽可能的减少IO访问次数**

### 嵌套循环连接(Nested-Loop Join)

- 步骤1：选取驱动表，使用与驱动表相关的过滤条件，选取代价最低的单表访问方法来执行对驱动表的单表查询
- 步骤2：对上一步骤中查询驱动表得到的结果集中每一条记录，都分别到被驱动表中查找匹配的记录
- 如果有3个表进行连接的话，那么步骤2中得到的结果集就像是新的驱动表，然后第三个表就成为了被驱动表

**所以我们设计的连接查询的时候关联条件，在被驱动表中是可以使用到索引的。但是还是得注意，只有在二级索引 + 回表的代价**
**比全表扫描的代价更低时才会使用索引。<font color=red>还有就是最终查询出来的字段最好不要是`*`  这样尽可能的使用覆盖索引查询更好</font>**



### 基于块的嵌套循环连接(Block Nested-Loop Join)

> 这个可以在执行计划Extra字段里面看到“Block Nested Loop”字样。

 **尽量减少访问被驱动表的次数**

MySQL提出了一个join buffer的概念，join buffer就是执行连接查询前申请的一块固定大小的内存，先把若干条驱动表结果集中的记录装在这个join buffer中，然后开始扫描被驱动表，每一条被驱动表的记录一次性和join buffer中的多条驱动表记录做匹配，因为匹配的过程都是在内存中完成的，所以这样可以显著减少被驱动表的I/O代价。最好的情况是join buffer足够大，能容纳驱动表结果集中的所有记录，这样只需要访问一次被驱动表就可以完成连接操作了。

这个join buffer的大小是可以通过启动参数或者系统变量`join_buffer_size`进行配置，默认大小为262144字节（也就是256KB），最小可以设置为128字节。当然，对于优化被驱动表的查询来说，**最好是为被驱动表加上效率高的索引**，**如果实在不能使用索引，并且自己的机器的内存也比较大可以尝试调大join_buffer_size的值来对连接查询进行优化。**

```mysql
show variables like 'join_buffer_size%'
```

<font color=red>另外需要注意的是，驱动表的记录**并不是所有列**都会被放到join buffer中，只有查询列表中的列和过滤条件中的列才会被放到join buffer中，所以再次提醒我们，最好不要把`*`作为查询列表，只需要把我们关心的列放到查询列表就好了，这样还可以在join buffer中放置更多的记录。**如果放不下驱动表的所有数据话，策略很简单，就是分段放**</font>



## 优化相关

#### Multi-Range Read优化 介绍

MRR优化在与在于，**定位数据后没有直接回表通过id随机查询，而是在中间用了一块`read_rnd_buffer`**，把大量定位到的数据进行排序后，去主键id索引树上**顺序查询记录**。但是这个是默认关闭的`set optimizer_switch="mrr_cost_based=off"`



#### (NLJ的优化BKA)Batched Key Access

> NLJ算法的优化 MySQL默认内置的优化策略，建议默认开启使用

```MYSQL
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
前两个参数的作用是要启用MRR。这么做的原因是，BKA算法的优化要依赖于MRR。
```

**NLJ算法执行的逻辑是：从驱动表t1，一行行地取出a的值，再到被驱动表t2去做join。也就是说，对于表t2来说，每次都是匹配一个值**。

当用上BKA算法后，**他会利用join_buffer 这个空间**来临时存储从被驱动表取出来的数据，也就减少了join的次数。



#### (BNL的优化)

> MySQL在大表的情况下使用Block Nested-Loop Join算法时，很依赖于`join_buffer_size`，这个值往往比较小。所以MySQL在大表连接查询性能会不好。

首先明确一点：由于InnoDB对Bufffer Pool的LRU算法做了优化，即：第一次从磁盘读入内存的数据页，会先放在old区域。如果1秒之后这个数据页不再被访问了，就不会被移动到LRU链表头部，这样对Buffer Pool的命中率影响就不大。

但是，如果一个使用BNL算法的join语句，多次扫描一个冷表，而且这个语句执行时间超过1秒，就会在再次扫描冷表的时候，把冷表的数据页移到LRU链表头部。如果这个冷表很大，就会出现另外一种情况：业务正常访问的数据页，没有机会进入young区

**大表join操作虽然对IO有影响，但是在语句执行结束后，对IO的影响也就结束了。但是，对Buffer Pool的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。**

1. 可能会多次扫描被驱动表，占用磁盘IO资源；
2. 判断join条件需要执行M*N次对比（M、N分别是两张表的行数），如果是大表就会占用非常多的CPU资源；
3. 可能会导致Buffer Pool的热数据被淘汰，影响内存命中率。

我们执行语句之前，需要通过理论分析和查看explain结果的方式，确认是否要使用BNL算法。

**给被驱动表的join字段加上索引，把BNL算法转成BKA算法**

如果被驱动表数据量少，觉得建立索引浪费，那也只能在代码中利用hash 的方法来匹对数据了。



## 相关问题

```
首先inner join 会根据数据量来确定驱动表。
而left join 或者 right join 指定驱动表，最后MySQL也不一定是这样执行的。
```

```mysql
select * from a left join b on(a.f1=b.f1) and (a.f2=b.f2); /*Q1*/
select * from a left join b on(a.f1=b.f1) where (a.f2=b.f2);/*Q2*/
```

> 在MySQL里，NULL跟任何值执行等值判断和不等值判断的结果，都是NULL。这里包括， select NULL = NULL 的结果，也是返回NULL

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/join/leftjoin.png!print " />

语句Q2里面where a.f2=b.f2就表示，查询结果里面不会包含b.f2是NULL的行，这样这个left join的语义就是“找到这两个表里面，f1、f2对应相同的行。对于表a中存在，而表b中匹配不到的行，就放弃”,这样，这条语句虽然用的是left join，但是语义跟join是一致的,最终MySQL 就会改写为`join`。

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/join/joinexplan.png!print " />

改写：

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/join/show.png!print " />

我们在看看`join`where条件和on 的条件

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/join/joinc.png!print " />

可以看到**join将判断条件是否全部放在on部分就没有区别**

## 总结

<font color=red>总体来看。在做连接查询时，我们需要根据条件区别出那个表记录是最少的。把他作为驱动表。然后在根据被驱动表的`join`字段建立索引，这样是比较好的。总之，整体的思路就是，尽量让每一次参与join的驱动表的数据集，越小越好，因为这样我们的驱动表就会越小。</font>

虽然BNL算法和Simple Nested Loop Join 算法都是要判断M*N次（M和N分别是join的两个表的行数），但是Simple Nested Loop Join 算法的每轮判断都要走全表扫描，而BNL是不会影响buffer pool这一块东西的。NLJ这个算法天然会对被驱动表的数据做多次访问，更容易将这些数据页放到Buffer Pool的头部。因此性能上BNL算法执行起来会快很多。
