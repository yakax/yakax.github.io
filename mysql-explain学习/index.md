# MySQL-Explain学习


## 执行计划各列详解

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/explain.png!print " />

<!--more-->
### id

在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的，出现在前边的表表示驱动表，出现在后边的表表示被驱动表，**每出现一个SELECT关键字，就为它分配一个唯一的id值**

> 比如子查询 可能id就不想同，但查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询，id就相同了。



### select_type

> MySQL为每一个SELECT关键字代表的小查询都定义了一个称之为select_type的属性，意思是我们只要知道了某个小查询的select_type属性，就知道了这个小查询在整个大查询中扮演了一个什么角色。

- SIMPLE

  查询语句中不包含UNION或者子查询的查询都算作是SIMPLE类型、当然，连接查询也算是SIMPLE类型

  

- PRIMARY

  对于包含UNION、UNION ALL或者子查询的大查询来说，它是由几个小查询组成的，其中最左边的那个查询的select_type值就是PRIMARY

  ```mysql
  EXPLAIN SELECT * FROM info_flow UNION SELECT * FROM info_flow
  ```

  <img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/union.png!print " />

  

- UNION

  对于包含UNION或者UNION ALL的大查询来说，它是由几个小查询组成的，其中除了最左边的那个小查询以外，其余的小查询的select_type值就是UNION

  

- UNION RESULT

  MySQL选择使用临时表来完成UNION查询的去重工作，针对该临时表的查询的select_type就是UNION RESULT

  

- SUBQUERY

  > 物化：在SQL执行过程中，第一次需要子查询结果时执行子查询并将子查询的结果保存为临时表，后续对子查询结果集的访问将直接通过临时表获得
  >
  > semi-join：只需要满足匹配一张表。半连接

  如果包含子查询的查询语句不能够转为对应的semi-join的形式,一般在in语句带子查询时。**并且该子查询是不相关子查询,** 并且查询优化器决定采用将该子查询物化的方案来执行该子查询时。子查询由于会被物化，所以只需要执行一遍

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
  ```

- DEPENDENT SUBQUERY

  如果包含子查询的查询语句不能够转为对应的semi-join的形式，**并且该子查询是相关子查询**，则该子查询的第一个SELECT关键字代表的那个查询的select_type就是DEPENDENT SUBQUERY。需要注意这个查询类型没有被物化，查询可能会被执行多次

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE s1.key2 = s2.key2) OR key3 = 'a';
  ```

- DEPENDENT UNION

  在包含UNION或者UNION ALL的大查询中，如果各个小查询都依赖于外层查询的话，那除了最左边的那个小查询之外，其余的小查询的select_type的值就是DEPENDENT UNION。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE key1 = 'a' UNION SELECT key1 FROM s1 WHERE key1 = 'b')
  ```

  **这个查询比较复杂啊，大查询里包含了一个子查询，子查询里又是由UNION连起来的两个小查询。**

  **SELECT key1 FROM s2 WHERE key1 = 'a'这个小查询由于是子查询中第一个查询，所以它的select_type是DEPENDENT SUBQUERY，**

  **而SELECT key1 FROM s1 WHERE key1 = 'b'这个查询的select_type就是DEPENDENT UNION**

  

- DERIVED

  对于采用物化的方式执行的包含派生表的查询，该派生表对应的子查询的select_type就是DERIVED

  ```mysql
  EXPLAIN SELECT * FROM (SELECT id, count(*) as count FROM info_flow GROUP BY id) AS derived_s1 where id > 1
  ```

  <img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/derived.png!print " />

  从执行计划中可以看出，id为2的记录就代表子查询的执行方式，它的select_type是DERIVED，说明该子查询是以物化的方式执行的。id为1的记录代表外层查询，大家注意看它的table列显示的是<derived2>，表示该查询是针对将派生表物化之后的表进行查询的。

  

- MATERIALIZED

  当查询优化器在执行包含子查询的语句时，选择将子查询物化之后与外层查询进行连接查询时，该子查询对应的select_type属性就是MATERIALIZED

  ```mysql
  EXPLAIN SELECT * FROM info_flow where area_id in(SELECT biz_id FROM biz_dispatch_record)
  ```

  <img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/materilized.png!print " />

  执行计划的第三条记录的id值为2，说明该条记录对应的是一个单表查询，从它的select_type值为MATERIALIZED可以看出，查询优化器是要把子查询先转换成物化表。然后看执行计划的前两条记录的id值都为1，说明这两条记录对应的表进行连接查询，需要注意的是第二条记录的table列的值是<subquery2>，**说明该表其实就是id为2对应的子查询执行之后产生的物化表，然后将s1和该物化表进行连接查询。**

  

### table

**不论我们的查询语句有多复杂，里边儿包含了多少个表，到最后也是需要对每个表进行单表访问的**，<font color=red>所以MySQL规定EXPLAIN语句输出的每条记录都对应着某个单表的访问方法，该条记录的table列代表着该表的表名</font>

### partitions

分区

### type 主要关注

>  执行计划的一条记录就代表着MySQL对某个表的执行查询时的访问方法，其中的type列就表明了这个访问方法是个啥
>
>  一般来说，这些访问方法按照我们介绍它们的顺序性能依次变差。其中除了All这个访问方法外，其余的访问方法都能用到索引，除了index_merge访问方法外，其余的访问方法都最多只能用到一个索引。

- const

  当我们根据**主键**或者**唯一**二级索引列与常数进行等值匹配时，对单表的访问方法就是const。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE id = 5;
  ```

  

- eq_ref

  在连接查询时，如果被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（**如果该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较**），则对该被驱动表的访问方法就是eq_ref

  ```mysql
  EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
  ```

  

- ref

  当通过普通的二级索引列与常量进行等值匹配时来查询某个表，那么对该表的访问方法就可能是ref

  ```mysql
  SELECT * FROM single_table WHERE key1 = 'abc';
  ```

  

- ref_or_null

  对普通二级索引进行等值匹配查询，该索引列的值也可以是NULL值时，那么对该表的访问方法就可能是ref_or_null。

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key1 IS NULL;
  ```

  

- index_merge

  [索引合并](https://yakax.gitee.io/mysql-索引-索引合并学习/)

- unique_subquery

  unique_subquery是针对在一些包**含IN子查询的查询语句中**，如果查询优化器决定将IN子查询转换为EXISTS子查询，而且**子查询可以使用到主键进行等值匹配的话**，那么该子查询执行计划的type列的值就是unique_subquery

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  ```

  

- index_subquery

  index_subquery与unique_subquery类似，只不过访问子查询中的表时使用的是普通的索引

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE common_field IN (SELECT key3 FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  ```

  

- range

  如果使用索引获取某些范围区间的记录，那么就可能使用到range访问方法

  ```mysql
  EXPLAIN SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c');
  EXPLAIN SELECT * FROM s1 WHERE key1 > 'a' AND key1 < 'b';
  ```

  **只要索引列和常数使用=、<=>、IN、NOT IN、IS NULL、IS NOT NULL、>、<、>=、<=、BETWEEN、!=（不等于也可以写成<>）或者LIKE操作符连接起来，就可以产生一个所谓的区间，（LIKE操作符比较特殊，只有在匹配完整字符串或者匹配字符串前缀时才可以利用索引）** IN操作符的效果和若干个等值匹配操作符`=`之间用`OR`连接起来是一样

  

  ##### 未使用索引的情况

  ```mysql
  SELECT * FROM single_table WHERE key2 > 100 AND common_field = 'abc';
  ```

  这个查询语句中能利用的索引只有idx_key2一个，而idx_key2这个二级索引的记录中又不包含common_field这个字段，所以在使用二级索引idx_key2定位记录的阶段用不到common_field = 'abc'这个条件，这个条件是在回表获取了完整的用户记录后才使用的,而范围区间是为了到索引中取记录中提出的概念，所以在确定范围区间的时候不需要考虑common_field = 'abc'这个条件。

  **这也就说说明如果我们强制使用idx_key2执行查询的话，对应的范围区间就是(-∞, +∞)而不是(100, +∞)，也就是需要将全部二级索引的记录进行回表，这个代价肯定比直接全表扫描都大了。所以mysql 很可能会直接扫描聚簇索引。**

  

- index

  当我们可以使用索引覆盖，但需要扫描全部的索引记录时，也就是要回表时。

  ```mysql
   EXPLAIN SELECT key_part2 FROM s1 WHERE key_part3 = 'a';
  ```

  上述查询中的搜索列表中只有key_part2一个列，而且搜索条件中也只有key_part3一个列，这两个列又恰好包含在idx_key_part这个索引中，可是搜索条件key_part3不能直接使用该索引进行ref或者range方式的访问，只能**扫描整个idx_key_part**索引的记录。

  对于使用InnoDB存储引擎的表来说，二级索引的记录只包含索引列和主键列的值，而聚簇索引中包含用户定义的全部列以及一些隐藏列，**所以扫描二级索引的代价比直接全表扫描，也就是扫描聚簇索引的代价更低一些**。

- all

  最直接的查询执行方式就是我们已经提了无数遍的**全表扫描**，对于InnoDB表来说也就是直接扫描聚簇索引，设计MySQL的大叔把这种使用全表扫描执行查询的方式称之为：all



### possible_keys和和key  主要关注

possible_keys列表示在某个查询语句中，对某个表执行单表查询时可能用到的索引有哪些，key列表示实际用到的索引有哪些

```mysql
EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key3 = 'a';
一般情况possible_keys列的值是idx_key1,idx_key3。后key列的值是其中一个：优化器会根据数据量 成本来确定用那个索引。
```



### key_len

key_len列表示当优化器决定使用某个索引执行查询时，该索引记录的最大长度

- 对于使用固定长度类型的索引列来说，它实际占用的存储空间的最大长度就是该固定值，对于指定字符集的变长类型的索引列来说，比如某个索引列的类型是VARCHAR(100)，使用的字符集是utf8那么该列实际占用的最大存储空间就是100 × 3 = 300个字节。
- 如果该索引列可以存储NULL值，则key_len比不可以存储NULL值时多1个字节。
- 对于变长字段来说，都会有2个字节的空间来存储该变长列的实际长度。

所以一般VARCHAR(100) 可以存储空 就会有303字节

### ref

当使用索引列等值匹配的条件去执行查询时，也就是在访问方法是**const、eq_ref、ref、ref_or_null、unique_subquery、index_subquery**其中之一时，**ref** 列展示的就是与索引列作等值匹配的东东是个啥。

```mysql
EXPLAIN SELECT* FROM lms_clue WHERE phone='15183904781'
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/ref.png!print " />

```mysql
EXPLAIN SELECT* FROM lms_clue s1 INNER JOIN app_user  s2 ON s1.phone = s2.phone
显示具体索引匹配字段
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/refkey.png!print " />

```mysql
EXPLAIN SELECT* FROM lms_clue s1 INNER JOIN app_user  s2 ON s1.phone = UPPER(s2.phone)
显示函数func
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/reffunc.png!print " />



### rows

如果查询优化器决定使用全表扫描的方式对某个表执行查询时，执行计划的rows列就代表预计需要扫描的行数，如果使用索引来执行查询时，执行计划的rows列就代表预计扫描的索引记录行数；但这个是预估的 不一定准。



### filtered

- 如果使用的是全表扫描的方式执行的单表查询，那么计算驱动表扇出时需要估计出满足搜索条件的记录到底有多少条。
- 如果使用的是索引执行的单表扫描，那么计算驱动表扇出的时候需要估计出满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

```mysql
EXPLAIN SELECT* FROM lms_clue s1 WHERE  biz_id>100000 and phone >'15183904781' 
从执行计划可以看出来，满足biz_id的记录有231284条。执行计划的filtered列就代表查询优化器预测在这266条记录中，有多少条记
录满足其余的搜索条件，也就是phone >'15183904781' 这个条件的百分比。此处filtered列的值是25.00。都是预估值

如果是连表查询 也就是驱动表与被驱动表的匹配条数。
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/explain/filter.png!print " />



### Extra

这个里面有很多类型，列举一些比较常见的。

- **No tables used** : 顾名思义 没有表

- **Impossible WHERE**：查询语句的WHERE子句永远为FALSE时将会提示

- **No matching min/max row**：当查询列表处有MIN或者MAX聚集函数，但是并没有符合WHERE子句中的搜索条件的记录时

  -  EXPLAIN SELECT MIN(key1) FROM s1 WHERE key1 = 'abcdefg';

- **Using index**：当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用索引覆盖的情况下

- **Using index condition**: 其实就是索引条件下推-在索引那篇文章学习过
  我们说回表操作其实是一个随机IO，比较耗时，所以上述修改虽然只改进了一点点，但是可以省去好多回表操作的成本。设计MySQL的大叔们把他们的这个改进称之为索引条件下推（英文名：IndexCondition Pushdown）。 

- **Using where**: 当我们使用全表扫描来执行对某个表的查询，并且该语句的WHERE子句中有针对该表的搜索条件时

- **Using join buffer (Block Nested Loop)**：在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，MySQL一般会为其分配一块名叫join buffer的内存块来加快查询速度，也就是我们所讲的基于块的嵌套循环算法--在连接查询详解文章里面有写。

- **Not exists**: 连接查询时发现where条件与on条件不匹配

- **Using intersect(...)、Using union(...)和Using sort_union(...)**:使用了索引合并

- **Zero limit:** limit 0

- **Using filesort**：排序操作无法在索引上，只能采用内存或者磁盘排序的时候，如果数据量大最好还是优化为按索引顺序排序。、

- **Using temporary**：在许多查询的执行过程中，MySQL可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含DISTINCT、GROUP BY、UNION等子句的查询过程中，如果不能有效利用索引来完成查询，MySQL很有可能寻求通过建立内部的临时表来执行查询。

  > **如果临时表分组了,mysql会默认加上orderby来排序。意味着：你查询某个没索引的字段，并且你要对这个字段分组。那么mysql 会建立临时表并且对记录进行分组排序。如果不想排序。我们需要显式指定order by null 来减少Using filesort这个成本**

- **Start temporary, End temporary**: 查询优化器会优先尝试将IN子查询转换成semi-join，而semi-join又有好多种执行策略，当执行策略为DuplicateWeedout时，**也就是通过建立临时表来实现为外层查询中的记录进行去重操作时**，驱动表查询执行计划的Extra列将显示Start temporary提示，被驱动表查询执行计划的Extra列将显示End temporary提示 

- **LooseScan**：在将In子查询转为semi-join时，如果采用的是LooseScan执行策略

- FirstMatch(tbl_name)：在将In子查询转为semi-join时，如果采用的是FirstMatch执行策略

  > semi-join 具体看连接查询篇

## 查看额外执行计划信息

在我们使用EXPLAIN语句查看了某个查询的执行计划后，紧接着还可以使用SHOW WARNINGS语句查看与这个查询的执行计划有关的一些扩展信息

```mysql
EXPLAIN SELECT* FROM lms_clue s1 INNER JOIN app_user  s2 ON s1.phone = s2.phone;
show WARNINGS;
```

分别是**Level、Code、Message**。我们最常见的就是Code为1003的信息，当Code值为1003时，Message字段展示的信息类似于查询优化器将我们的查询语句重写后的语句。

**Message**字段展示的信息类似于查询优化器将我们的查询语句重写后的语句，**并不是等价于**，也就是说Message字段展示的信息并不是标准的查询语句，在很多情况下并不能直接拿到黑框框中运行，它只能作为帮助我们理解查MySQL将如何执行查询语句的一个参考**依据**而已



## 查看执行计划优化过程

如果要看具体执行计划为什么选择某个索引，我们需要通过optimizer_trace来查看。

默认`optimizer_trace` 这个是关闭的。

```mysql
SET optimizer_trace="enabled=on";

SELECT * FROM s1 WHERE 
    key1 > 'z' AND 
    key2 < 1000000 AND 
    key3 IN ('a', 'b', 'c') AND 
    common_field = 'abc';
    
SELECT * FROM information_schema.OPTIMIZER_TRACE\G

## 当你停止查看语句的优化过程时，把optimizer trace功能关闭
SET optimizer_trace="enabled=off"
```

information_schema数据库下的OPTIMIZER_TRACE表字段有4个 

- **QUERY**：表示我们的查询语句。
- **TRACE**：表示优化过程的JSON格式文本。
- **MISSING_BYTES_BEYOND_MAX_MEM_SIZE**：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
- **INSUFFICIENT_PRIVILEGES**：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是1

> 优化过程大致分为了三个阶段：prepare阶段、optimize阶段、execute阶段
>
> 我们所说的基于成本的优化主要集中在optimize阶段。
> 对于单表查询来说，我们主要关注optimize阶段的"rows_estimation"这个过程
> 对于多表连接查询来说，我们更多需要关注"considered_execution_plans"这个过程





