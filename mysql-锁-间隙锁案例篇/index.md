# MySQL-锁-间隙锁案例篇




## 间隙加锁规则

1. 原则1：加锁的基本单位是next-key lock。希望你还记得，next-key lock是前开后闭区间。
2. 原则2：查找过程中访问到的对象才会加锁。
3. 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁。
4. 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。
<!--more-->



>**注意！有行**才会加行锁。如果查询条件没有命中行，那就加next-key lock。当然，等值判断的时候，需要加上优化2
>
>（即：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。）
>
><=到底是间隙锁还是行锁？其实，这个问题，你要跟“执行过程”配合起来分析。
>**在InnoDB要去找“第一个值”的时候，是按照等值去找的，用的是等值判断的规则；**
>**找到第一个值以后，要在索引内找“下一个值”，对应于我们规则中说的范围查找。**



## 案例SQL

所有案例都是在可重复读隔离级别(repeatable-read)下验证的

```MYSQL
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```



## 案例1-等值查询间隙锁

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock1.png!print" style="zoom: 33%;" />

>由于表t中没有id=7的记录，所以用我们上面提到的加锁规则判断一下的话：
>
>1. 根据原则1，加锁单位是next-key lock，session A加锁范围就是(5,10]；
>2. 同时根据优化2，这是一个等值查询(id=7)，而id=10不满足查询条件，next-key lock退化成间隙锁，因此最终加锁的范围是(5,10)。
>
>所以，session B要往这个间隙里面插入id=8的记录会被锁住，但是session C修改id=10这行是可以的。

## 案例2-非唯一索引等值锁

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock2.png!print" style="zoom: 33%;" />

这里session A要给索引c上c=5的这一行加上读锁。

1. 根据原则1，加锁单位是next-key lock，因此会给(0,5]加上next-key lock。
2. **要注意c是普通索引**，因此仅访问c=5这一条记录是不能马上停下来的，需要向右遍历，查到c=10才放弃。根据原则2，访问到的都要加锁，因此要给(5,10]加next-key lock。
3. 但是同时这个符合优化2：等值判断，向右遍历，最后一个值不满足c=5这个等值条件，因此退化成间隙锁(5,10)。
4. 根据原则2 ，**只有访问到的对象才会加锁**，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么session B的update语句可以执行完成。

但session C要插入一个(7,7,7)的记录，就会被session A的间隙锁(5,10)锁住。

**在这个例子中，lock in share mode只锁覆盖索引，但是如果是for update就不一样了。 执行 for update时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁**。

**这个例子说明,锁是加在索引上的,** 同时，它给我们的指导是，如果你要用lock in share mode来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段。

**所以，如果一个select * from ... for update 语句，优化器决定使用全表扫描，那么就会把主键索引上next-key lock全加上。**



## 案例3：主键索引范围锁

```mysql
mysql> select * from t where id=10 for update;
mysql> select * from t where id>=10 and id<11 for update;
这两条查语句肯定是等价的，但是它们的加锁规则不太一样
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock3.png!print" style="zoom: 33%;" />

1. 开始执行的时候，要找到第一个id=10的行，因此本该是next-key lock(5,10]。 根据优化1， 主键id上的等值条件，退化成行锁，只加了id=10这一行的行锁。
2. **范围查找就往后继续找，找到id=15这一行停下来，因此需要加next-key lock(10,15]。**

>session A这时候锁的范围就是主键索引上，行锁id=10和next-key lock(10,15]。这样，session B和session C的结果你就能理解了。
>
>这里需要注意一点，首次session A定位查找id=10的行的时候，是当做等值查询来判断的，而向右扫描到id=15的时候，用的是范围查询判断。



## 案例4：非唯一索引范围锁

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock4.png!print" style="zoom: 33%;" />

**session A用字段c来判断，加锁规则跟案例三唯一的不同是：在第一次用c=10定位记录的时候，索引c上加了(5,10]这个next-key lock后，由于索引c是非唯一索引，没有优化规则，也就是说不会蜕变为行锁，因此最终sesion A加的锁是，索引c上的(5,10] 和(10,15] 这两个next-key lock**。

> 所以从结果上来看，sesson B要插入（8,8,8)的这个insert语句时就被堵住了。
>
> 这里需要扫描到c=15才停止扫描，是合理的，因为InnoDB要扫到c=15，才知道不需要继续往后找了。



## 案例5：唯一索引范围锁bug

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock5.png!print" style="zoom: 33%;" />

**session A是一个范围查询，按照原则1的话，应该是索引id上只加(10,15]这个next-key lock，并且因为id是唯一键，所以循环判断到id=15这一行就应该停止了。但是实现上，InnoDB会往前扫描到第一个不满足条件的行为止，也就是id=20。而且由于这是个范围扫描，因此索引id上的(15,20]这个next-key lock也会被锁上。**

> 所以你看到了，session B要更新id=20这一行，是会被锁住的。同样地，session C要插入id=16的一行，也会被锁住。
>
> 照理说，这里锁住id=20这一行的行为，其实是没有必要的。因为扫描到id=15,就可以确定不用往后再找了,但实现上还是这么做了



## 案例6：非唯一索引上存在"等值"的例子

```mysql
insert into t values(30,10,30);
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/next6.png!print" style="zoom: 33%;" />

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock6.png!print" style="zoom: 33%;" />

这时，session A在遍历的时候，先访问第一个c=10的记录。同样地，根据原则1，这里加的是(c=5,id=5)到(c=10,id=10)这个next-key lock。然后，session A向右查找，直到碰到(c=15,id=15)这一行，循环才结束。根据优化2，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成(c=10,id=10) 到 (c=15,id=15)的间隙锁。

**所以最后加锁就是（5,5) 到（15,15）之间** 即(c=5,id=5)和(c=15,id=15)这两行上都没有锁。



## 案例7：limit 语句加锁

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock7.png!print" style="zoom: 33%;" />

session A的delete语句加了 limit 2。你知道表t里c=10的记录其实只有两条，因此加不加limit 2，删除的效果都是一样的，但是加锁的效果却不同。可以看到，session B的insert语句执行通过了，跟案例六的结果不同。这是因为，案例七里的delete语句明确加了limit 2的限制，因此在遍历到(c=10, id=30)这一行之后，满足条件的语句已经有两条，循环就结束了。

> 因此，索引c上的加锁范围就变成了从（c=5,id=5)到（c=10,id=30)这个前开后闭区间；



## 案例8：一个死锁的例子

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock8.png!print" style="zoom: 33%;" />

现在，我们按时间顺序来分析一下为什么是这样的结果。

1. session A 启动事务后执行查询语句加lock in share mode，在索引c上加了next-key lock(5,10] 和间隙锁(10,15)；
2. session B 的update语句也要在索引c上加next-key lock(5,10] ，进入锁等待；
3. 然后session A要再插入(8,8,8)这一行，被session B的间隙锁锁住。由于出现了死锁，InnoDB让session B回滚。

**其实是这样的，session B的“加next-key lock(5,10] ”操作，实际上分成了两步，先是加(5,10)的间隙锁，加锁成功；然后加c=10的行锁，这时候才被锁住的。** 也就是说，我们在分析加锁规则的时候可以用next-key lock来分析。但是要知道，具体执行的时候，是要分成间隙锁和行锁两段来执行的。



## 案例9：order by改变加锁方向

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock9.png!print" style="zoom: 33%;" />

1. 由于是order by c desc，第一个要定位的是索引c上“最右边的”c=20的行，所以会加上间隙锁(20,25)和next-key lock (15,20]。
2. 在索引c上向左遍历，要扫描到c=10才停下来，所以next-key lock会加到(5,10]，这正是阻塞session B的insert语句的原因。
3. 在扫描过程中，c=20、c=15、c=10这三行都存在值，由于是select *，所以会在主键id上加三个行锁。

因此，session A 的select语句锁的范围就是：**索引c上 (5, 25)；主键索引上id=10、15、20三个行锁。**

<font color=red>这里在说明一下！锁就是加在索引上的，这是InnoDB的一个基础设定，需要你在分析问题的时候要一直记得。</font>



## 案例10：再次分析范围查询

```mysql
begin;
select * from t where id>9 and id<12 order by id desc for update;
```

我们知道这个语句的加锁范围是主键索引上的 (0,5]、(5,10]和(10, 15)。也就是说，id=15这一行，并没有被加上行锁

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/next10.png!print" style="zoom: 33%;" />

我们说加锁单位是next-key lock，都是前开后闭区间，但是这里用到了优化2，即索引上的等值查询，向右遍历的时候id=15不满足条件，所以next-key lock退**化为了间隙锁** (10, 15)。

1. 首先这个查询语句的语义是order by id desc，要拿到满足条件的所有行，优化器必须先找到“第一个id<12的值”。
2. 这个过程是通过索引树的搜索过程得到的，在引擎内部，其实是要找到id=12的这个值，只是最终没找到，但找到了(10,15)这个间隙。
3. 然后向左遍历，在遍历过程中，就不是等值查询了，会扫描到id=5这一行，所以会加一个next-key lock (0,5]。



```mysql
begin;
select id from t where c in(5,20,10) lock in share mode;
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextexplan.png!print" style="zoom: 33%;" />

1. 在查找c=5的时候，先锁住了(0,5]。但是因为c不是唯一索引，为了确认还有没有别的记录c=5，就要向右遍历，找到c=10才确认没有了，这个过程满足优化2，所以加了间隙锁(5,10)。
2. 同样的，执行c=10这个逻辑的时候，加锁的范围是(5,10] 和 (10,15)；执行c=20这个逻辑的时候，加锁的范围是(15,20] 和 (20,25)。4
3. 通过这个分析，我们可以知道，这条语句在索引c上加的三个记录锁的顺序是：先加c=5的记录锁，再加c=10的记录锁，最后加c=20的记录锁。

需要注意的 `select id from t where c in(5,20,10) order by c desc for update;` **间隙锁虽然不互斥间隙锁，但是细化到记录上还是会出现互斥的。问题是in条件的加锁时一个个去扫描的。所以两个sql语句在并发时有可能会出现死锁**



## 怎么查看死锁

> 下面试间隙锁案例10出现死锁情况

```
在出现死锁后，执行show engine innodb status命令得到的部分输出。这个命令会输出很多信息，有一节LATESTDETECTED DEADLOCK，就是记录的最后一次死锁信息。
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/slock.png!print" style="zoom: 33%;" />

1. 这个结果分成三部分：
   - TRANSACTION，是第一个事务的信息；
   - TRANSACTION，是第二个事务的信息；
   - WE ROLL BACK TRANSACTION (1)，是最终的处理结果，表示回滚了第一个事务。
2. 第一个事务的信息中：
   - WAITING FOR THIS LOCK TO BE GRANTED，表示的是这个事务在等待的锁信息；
   - index c of table `test`.`t`，说明在等的是表t的索引c上面的锁；
   - lock mode S waiting 表示这个语句要自己加一个读锁，当前的状态是等待中；
   - Record lock说明这是一个记录锁；
   - n_fields 2表示这个记录是两列，也就是字段c和主键字段id；
   - 0: len 4; hex 0000000a; asc ;;是第一个字段，也就是c。值是十六进制a，也就是10；
   - 1: len 4; hex 0000000a; asc ;;是第二个字段，也就是主键id，值也是10；
   - 这两行里面的asc表示的是，接下来要打印出值里面的“可打印字符”，但10不是可打印字符，因此就显示空格。
   - 第一个事务信息就只显示出了等锁的状态，在等待(c=10,id=10)这一行的锁。
   - 当然你是知道的，既然出现死锁了，就表示这个事务也占有别的锁，但是没有显示出来。别着急，我们从第二个事务的信息中推导出来。
3. 第二个事务显示的信息要多一些：
   - “ HOLDS THE LOCK(S)”用来显示这个事务持有哪些锁；
   - index c of table `test`.`t` 表示锁是在表t的索引c上；
   - hex 0000000a和hex 00000014表示这个事务持有c=10和c=20这两个记录锁；
   - WAITING FOR THIS LOCK TO BE GRANTED，表示在等(c=5,id=5)这个记录锁。

从上面这些信息中，我们就知道：

1. “lock in share mode”的这条语句，持有c=5的记录锁，在等c=10的锁；
2. “for update”这个语句，持有c=20和c=10的记录锁，在等c=5的记录锁。

因此导致了死锁。这里，我们可以得到两个结论：

1. 由于锁是一个个加的，要避免死锁，对同一组资源，要按照尽量相同的顺序访问；
2. 在发生死锁的时刻，for update 这条语句占有的资源更多，回滚成本更大，所以InnoDB选择了回滚成本更小的lock in share mode语句，来回滚。

## 怎么看锁等待

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/lockwait.png!print" style="zoom: 33%;" />

**可以看到，由于session A并没有锁住c=10这个记录，所以session B删除id=10这一行是可以的。但是之后，session B再想insert id=10这一行回去就不行了。**

`show engine innodb status`

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/lockwait1.png!print" style="zoom: 50%;" />

1. index PRIMARY of table `test`.`t` ，表示这个语句被锁住是因为表t主键上的某个锁。
2. lock_mode X locks gap before rec insert intention waiting 这里有几个信息：
   - insert intention表示当前线程准备插入一个记录，这是一个插入意向锁。为了便于理解，你可以认为它就是这个插入动作本身。
   - gap before rec 表示这是一个间隙锁，而不是记录锁。
3. 那么这个gap是在哪个记录之前的呢？接下来的0~4这5行的内容就是这个记录的信息。
4. n_fields 5也表示了，这一个记录有5列：
   - 0: len 4; hex 0000000f; asc ;;第一列是主键id字段，十六进制f就是id=15。所以，这时我们就知道了，这个间隙就是id=15之前的，因为id=10已经不存在了，它表示的就是(5,15)。
   - 1: len 6; hex 000000000513; asc ;;第二列是长度为6字节的事务id，表示最后修改这一行的是trx id为1299的事务。
   - 2: len 7; hex b0000001250134; asc % 4;; 第三列长度为7字节的回滚段信息。可以看到，这里的acs后面有显示内容(%和4)，这是因为刚好这个字节是可打印字符。
   - 后面两列是c和d的值，都是15。

<font color=red>因此，我们就知道了，由于delete操作把id=10这一行删掉了，原来的两个间隙(5,10)、(10,15）变成了一个(5,15)。</font>

**session A执行完select语句后，什么都没做，但它加锁的范围突然“变大”了；**

当我们执行select * from t where c>=15 and c<=20 order by c desc lock in share mode; 向左扫描到c=10的时候，要把(5, 10]锁起来。

**也就是说，所谓“间隙”，其实根本就是由“这个间隙右边的那个记录”定义的**。



## update间隙锁变大的例子



<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextupdata.png!print" style="zoom: 33%;" />

session A的加锁范围是索引c上的 (5,10]、(10,15]、(15,20]、(20,25]和(25,suprenum]。

之后session B的第一个update语句，要把c=5改成c=1，你可以理解为两步：

1. 插入(c=1, id=5)这个记录；
2. 删除(c=5, id=5)这个记录。

按照我们上面说的，索引c上(5,10)间隙是由这个间隙右边的记录。所以通过这个操作，session A间隙锁范围变成下层

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/nextlock11.png!print" style="zoom: 33%;" />

接下来session B要执行 update t set c = 5 where c = 1这个语句了，一样地可以拆成两步：

1. **插入(c=5, id=5)这个记录；**
2. 删除(c=1, id=5)这个记录。

**第一步试图在已经加了间隙锁的(1,10)中插入数据，所以就被堵住了。**



## kill锁住的线程

```mysql
 select * from t where id=1; # 长时间没返回
```

**分析语句之前 都应该执行`show processlist;查看` 后续操作找到了blocking_id kill掉**

### 长时间没返回

``` mysql
如果:performance_schema=on 打开着可以通过select blocking_pid from sys.schema_table_lock_waits; 
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/lock/showprocesslist.png!print " />



### 行锁

```mysql
 select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'
 ## 查询指定表里面的情况
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/lock/lock.png!print " />

```mysql
select * from t where c=5 for update
当级别为RR时，是可以解决幻读的，此时对于每条记录的间隙还要加上GAP锁。也就是说，表上每一条记录和每一个间隙都锁上了。
但是实际上 innodb先锁全表的所有行 返回server层,InnoDB就会把不满足条件的行行锁去掉。所以语句执行完只会锁c=5的行。
```



### 小总结

1. 表锁是主动在SQL 上面加的

2. MDL锁时自动加比如改变表结构

3. **如果在一个事务内，更新的数据条数过多。建议分开更新，因为在你事务开启时，其他session的查询都会查找你的undo log链条，如果你的链条太长会导致很慢**。

4. (DML、DDL语句时都会申请MDL锁，DML操作需要MDL读锁，DDL操作需要MDL写锁)

   > 增删改查，或者修改表之类的语句都会申请MDL锁，而增删改查 会自动加上MDL读锁。修改表会加上写锁
