# MySQL-Buffer Pool 学习


#### 一条sql执行基本流程

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/mysql%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png" style="zoom: 33%;" />

>在老版本的MySQL中sql通过连接进来时会有查询缓存这一块的逻辑。
>
>查询解析器解析sql 是否正确后，然后基本优化sql，生成执行计划，选择引擎执行。
>
>而InnoDB 在执行时 为了保证对数据的执行更快，省去磁盘IO开销 引入了Buffer Pool

<!--more-->

## Buffer Pool

> 当需要访问**某个页的数据时**，就会把完整的页的数据全部加载到内存中，也就是说即使我们只需要访问一个页的一条记录，那也需要先把整个页的数据加载到内存中，从某种意义上来说buffer pool 完全可以替代了查询缓存。

默认情况下 Buffer Pool 只有**128M**大小，参数`innodb_buffer_pool_size`;

Buffer pool 通常存储的是一个个数据页，一个页大小通常16k,不过每个数据页里面会有**描述信息**。

### free与 flush链表

Buffer pool 里面每个数据页都会有描述两个链表 一个**free链表描述空闲缓存链** 一个**flush描述被修改过的数据块**。 

free链表 与flush链表都如下图

![](https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/free链表.png!print)

### 缓存数据的淘汰策略 LRU链表

正常的LRU链表可能会导致MySQL会频繁的淘汰末尾数据，然后去加载数据库的数据到缓存页。(主要是全表扫描、查询预读导致)

引入**冷热数据LRU链表结构** 冷热数据比例是根据 `innodb_old_blocks_pct` 默认37 冷数据占比
数据第一次会被加载进入冷数据区域，1s后你又继续访问，就会进入热数据区域。1s内访问是不会加载进去的。

若这个数据页在LRU链表中存在的时间超过了1秒，就把它移动到冷数据链表头部 

**这样也能 保证预读进来的数据、或者全表扫描进来的数据，不会进入热数据区域。**

> 还有个细节，热数据的位置移动：比如有100个热数据页，当你访问后面75个才会移动，前面25个是不会移动的。
>
> 冷数据只要在1s后被访问是会进入热数据链表的。并且会在头部

**上面这个思想，完全可以用在业务系统上面的、比如redis热数据的缓存预加载**

后台线程 定时把flush链表数据刷入磁盘，定时把冷数据删除并刷入磁盘，来腾出链表空间。



### 多个Buffer Pool实例

> 在多线程环境下，访问Buffer Pool中的各种链表都需要加锁处理啥的，为了应对吞吐量，我们需要多个buffer pool,每个Buffer Pool都称为一个实例，它们都是独立的，独立的去申请内存空间，独立的管理各种链表。

参数`innodb_buffer_pool_instances`

![](https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/bufferpool.png!print)

```mysql
当innodb_buffer_pool_size的值小于1G的 时候设置多个实例是无效的，InnoDB会默认把innodb_buffer_pool_instances 的值修改为1
innodb_buffer_pool_size = 8589934592  #8g
innodb_buffer_pool_instances = 4  #4个
innodb_buffer_pool_size必须是innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances的倍数
（这主要是想保证每一个Buffer Pool实例中包含的chunk数量相同）。
假设我们指定的innodb_buffer_pool_chunk_size的值是128M，innodb_buffer_pool_instances的值是16，那么这两个值的乘积就是2G，也就是说innodb_buffer_pool_size的值必须是2G或者2G的整数倍。
```

**buffer pool 内存由多个 check组成 默认128M大小**。 在运行期间，如果我们机器内存变大了。他会自己申请chunk来自动增加总体缓存大小。

<font color=red>在实际生产环境中：buffer pool的大小、数量、chunk的大小都是需要优化的东西。很影响并发情况；</font>

> 如果设置不好：比如说设置小了。当缓存满了。你会去频繁的刷入缓存页尾部数据到磁盘腾出空间，然后从磁盘加载你的数据到缓存
>
> 这就是典型的缓存也使用很快，刷数据到磁盘很慢的情况。如果要加快刷盘，因为这也是磁盘io频繁，影响性能。这关键在于你的buffer pool大小情况。如果够大就好。 所以一般数据库服务器是需要大内存的。**经验值3/4**



### 查看系统pool情况

```mysql
SHOW ENGINE INNODB STATUS
```

- Total memory allocated，这就是说buffer pool最终的总大小是多少
- Buffer pool size，这就是说buffer pool一共能容纳多少个缓存页
- Free buffers，这就是说free链表中一共有多少个空闲的缓存页是可用的
- Database pages和Old database pages，就是说lru链表中一共有多少个缓存页，以及冷数据区域里的缓存页数量
- Modified db pages，这就是flush链表中的缓存页数量
- Pending reads和Pending writes，等待从磁盘上加载进缓存页的数量，还有就是即将从lru链表中刷入磁盘的数量、即将从flush链表中刷入磁盘的数量
- Pages made young和not young，这就是说已经lru冷数据区域里访问之后转移到热数据区域的缓存页的数量，以及在lru冷数据区域里1s内被访问了没进入热数据区域的缓存页的数量
- youngs/s和not youngs/s，这就是说每秒从冷数据区域进入热数据区域的缓存页的数量，以及每秒在冷数据区域里被访问了但是不能进入热数据区域的缓存页的数量
- Pages read xxxx, created xxx, written xxx，xx reads/s, xx creates/s, 1xx writes/s，这里就是说已经读取、创建和写入了多少个缓存页，以及每秒钟读取、创建和写入的缓存页数量
- Buffer pool hit rate xxx / 1000，这就是说每1000次访问，有多少次是直接命中了buffer pool里的缓存的
- young-making rate xxx / 1000 not xx / 1000，每1000次访问，有多少次访问让缓存页从冷数据区域移动到了热数据区域，以及没移动的缓存页数量
- LRU len：这就是lru链表里的缓存页的数量
- I/O sum：最近50s读取磁盘页的总数
- I/O cur：现在正在读取磁盘页的数量

### 普通索引与唯一索引在change buffer的情况

> change buffer用的是buffer pool里的内存，因此不能无限增大。change buffer的大小，
>
> 可以通过参数`innodb_change_buffer_max_size`来动态设置。
> 这个参数设置为50的时候，表示change buffer的大小最多只能占用bufferpool的50%。

```mysql
select id from T where k=5
```

查询过程

- 对于普通索引来说，查找到满足条件的第一个记录(5,500)后，需要查找下一个记录，直到碰到第一个不满足k=5条件的记录
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索

#### 更新过程:

1. 记录页在内存中

   - 对于唯一索引来说，找到3和5之间的位置，判断到没有冲突，插入这个值，语句执行结束；
   - 对于普通索引来说，找到3和5之间的位置，插入这个值，语句执行结束。

2. 记录也不在内存中

   - **对于唯一索引来说，需要将数据页读入内存**，判断到没有冲突，插入这个值，语句执行结束；
   - **对于普通索引来说，则是将更新记录在change buffer，语句执行就结束了**。

   **将数据从磁盘读入内存涉及随机IO的访问，是数据库里面成本最高的操作之一。change buffer因为减少了随机磁盘访问,所以对更新性能的提升是会很明显的。**

> 对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好
>
> 假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价
>
> 所以：如果所有的更新后面，都马上伴随着对这个记录的查询，那么你应该关闭change buffer。普通索引和change buffer的配合使用，对于数据量大的表的更新优化还是很明显。

#### change buffer 与redo log

1. Page 1在内存中，直接更新内存；
2. Page 2没有在内存中，就在内存的change buffer区域，记录下“我要往Page 2插入一行”这个信息
3. 将上述两个动作记入redo log中

<font color=red>**redo log主要节省的是随机写磁盘的I0消耗(转成顺序写) , 而change buffer主要节省的则是随机读磁盘的IO消耗**。</font>

### 总结

1. **磁盘太慢，用内存作为缓存很有必要。**
2. **Buffer Pool本质上是InnoDB向操作系统申请的一段连续的内存空间，可以通过innodb_buffer_pool_size来调整它的大小。**
3. **Buffer Pool向操作系统申请的连续内存由控制块和缓存页组成，每个控制块和缓存页都是一一对应的，在填充足够多的控制块和缓存页的组合后，BufferPool剩余的空间可能产生不够填充一组控制块和缓存页，这部分空间不能被使用，也被称为碎片。**
4. **InnoDB使用了许多链表来管理Buffer Pool。**
5. **free链表中每一个节点都代表一个空闲的缓存页，在将磁盘中的页加载到Buffer Pool时，会从free链表中寻找空闲的缓存页。**
6. **为了快速定位某个页是否被加载到Buffer Pool，使用表空间号 + 页号作为key，缓存页作为value，建立哈希表。**
7. **在Buffer Pool中被修改的页称为脏页，脏页并不是立即刷新，而是被加入到flush链表中，待之后的某个时刻同步到磁盘上。**
8. **LRU链表分为young和old两个区域，可以通过innodb_old_blocks_pct来调节old区域所占的比例。首次从磁盘上加载到Buffer Pool的页会被放到old区域的头部，在innodb_old_blocks_time间隔时间内访问该页不会把它移动到young区域头部。在Buffer Pool没有可用的空闲缓存页时，会首先淘汰掉old区域的一些页。** young 和old 分别对应热、冷
9. **我们可以通过指定innodb_buffer_pool_instances来控制Buffer Pool实例的个数，每个Buffer Pool实例中都有各自独立的链表，互不干扰。**
10. **自MySQL 5.7.5版本之后，可以在服务器运行过程中调整Buffer Pool大小。每个Buffer Pool实例由若干个chunk组成，每个chunk的大小可以在服务器启动时通过启动参数调整。**
11. **可以用下边的命令查看Buffer Pool的状态信息(命中情况、死锁等等)：`SHOW ENGINE INNODB STATUS`**





