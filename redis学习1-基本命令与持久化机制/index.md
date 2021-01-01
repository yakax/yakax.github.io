# Redis学习(1)-基本命令与持久化机制


### docker简单安装设置密码并开启持久化
```
docker run -d --name myredis -p 6379:6379 redis --requirepass "156967" --appendonly yes
```

### 文档

[文档学习](http://redisdoc.com/)

### 特性
1. 速度快
**数据放内存中是速度快的主要原因、C语言实现，与操作系统距离近、使用了单线程架构，预防多线程可能产生的竞争问题**
2. 丰富的功能：value可以为string、hash、list、set、zset等多种数据结构，可以满足很多应用场景。还提供了键过期，发布订阅，事务，流水线。(流水线: Redis 的流水线功能允许客户端一次将多个命令请求发送给服务器， 并将被执行的多个命令请求的结果在一个命令回复中全部返回给客户端， 使用这个功能可以有效地减少客户端在执行多个命令时需要与服务器进行通信的次数)
4. 高可用和分布式：哨兵机制实现高可用，保证redis节点故障发现和自动转移
5. 键值对的数据结构服务器
6. 持久化：发生断电或机器故障，数据可能会丢失，可以持久化到硬盘通过(aof、rdb)
7. 主从复制：实现多个相同数据的redis副本

<!--more-->
### 使用场景
1. 缓存：合理使用缓存加快数据访问速度，降低后端数据源压力
2. 排行榜：按照热度排名，按照发布时间排行，主要用到列表和有序集合
3. 计数器应用：视频网站播放数，网站浏览数，使用redis计数
4. 社交网络：赞、踩、粉丝、下拉刷新
5. 消息队列：发布和订阅

### 工具说明
| 可执行文件       |            作用             |
| ---------------- | :-------------------------: |
| redis-server     |          启动redis          |
| redis-cli        |      redis命令行客户端      |
| redis-benchmark  |        基准测试工具         |
| redis-check-aof  | AOF持久化文件检测和修复工具 |
| redis-check-dump | RDB持久化文件检测和修复工具 |
| redis-sentinel   |          启动哨兵           |

### 数据类型
#### 数据类型(String)
```
设值命令：
//10秒后过期  px 10000 毫秒过期
set age 23 ex 10
//不存在键name时，返回1设置成功；存在的话失败0
setnx name test  
//存在键age时，返回1成功
set age 25 xx 
//存在则返回value, 不存在返回nil
获值命令：get age 
// mset k v k v
批量设值：mset country china city beijing
//返回china  beigjin, address为nil
批量获取：mget country city address      
```
常用命令--字符串计数
```
incr age   //必须为整数自加1，非整数返回错误，无age键从0自增返回1
decr age   //整数age减1
incrby age 2 //整数age+2
decrby age 2//整数age -2
incrbyfloat score 1.1 //浮点型score+1.1
```
常用命令--字符串追加
```
append追加指令：
    set name hello; append name world //追加后成helloworld
字符串长度：
   set hello “世界”；strlen hello         //结果6，每个中文占3个字节
截取字符串：
   set name helloworld ; getrange name 2 4//返回 llo
```
#### 数据类型(Hash)
哈希hash是一个String类型的field和value的映射表，hash特适合用于存储对象
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/0.png)

```
hmset user:1 name yakax age 18 设值
   命令  hset key field value
   设值：hset user:1 name yakax           //成功返回1，失败返回0
   取值：hget user:1 name                 //返回yakax
   删值：hdel user:1 age                  //返回删除的个数
   计算个数：hset user:1 name yakax; hset user:1 age 23; 
             hlen user:1                  //返回2，user:1有两个属性值
   批量设值：hmset user:2 name yakax age 23 sex boy //返回OK
   批量取值：hmget user:2 name age sex    //返回三行：yakax 23 boy
   判断field是否存在：hexists user:2 name         //若存在返回1，不存在返回0
   获取所有field: hkeys user:2            // 返回name age sex三个field
   获取user:2所有value：hvals user:2                    // 返回yakax 23 boy
   获取user:2所有field与value：hgetall user:2    //name age sex yakax 23 boy值
   增加1：hincrby user:2 age 1                                  //age+1
          hincrbyfloat user:2 age 2       //浮点型加2
```

**三种方案实现用户信息存储优缺点**

1. 原生：
set user:1:name yakax;
set user:1:age  23;
set user:1:sex boy;
优点：简单直观，每个键对应一个值
缺点：键数过多，占用内存多，用户信息过于分散，不用于生产环境

2. 将对象序列化存入redis
set user:1 serialize(userInfo);
优点：编程简单，若使用序列化合理内存使用率高
缺点：序列化与反序列化有一定开销，更新属性时需要把userInfo全取出来进行反序列化，更新后再序列化到redis

3. 使用hash类型：
hmset user:1 name yakax age 23 sex boy
优点：简单直观，使用合理可减少内存空间消耗
缺点：要控制ziplist与hashtable两种编码转换，且hashtable会消耗更多内存serialize(userInfo);

```
内部编码：ziplist<压缩列表>和hashtable<哈希表>
当field个数少且没有大的value时，内部编码为ziplist
如：hmset user:3 name yakax age 24; object encoding user:3 //返回ziplist
当value大于64字节，内部编码由ziplist变成hashtable
如：hset user:4 address “fsgst64字节”; object encoding user:3 //返回hashtable
```
#### 数据类型(List)
用来存储多个有序的字符串，一个列表最多可存2的32次方减1个元素

```
添加命令：
 rpush yakax c b a    //从右向左插入cba, 返回值3
          lrange yakax 0 -1    //从左到右获取列表所有元素 返回 c b a
          lpush key c b a        //从左向右插入cba
          linsert yakax before b teacher //在b之前插入teacher, after为之后，使					                用lrange yakax 0 -1 查看：c teacher b a
查找命令：
 lrange key start end //索引下标特点：从左到右为0到N-1
          lindex yakax -1 //返回最右末尾a，-2返回b
          llen yakax         //返回当前列表长度
          lpop yakax        //把最左边的第一个元素c删除
          rpop yakax        //把最右边的元素a删除
```

#### 数据类型(Set)

用户标签，社交，查询有共同兴趣爱好的人,智能推荐,保存多元素，与列表不一样的是不允许有重复元素，且集合是无序，一个集合最多可存2的32次方减1个元素，除了支持增删改查，还支持集合交集、并集、差集；
```
 exists user       //检查user键值是否存在
 sadd user a b c//向user插入3个元素，返回3
 sadd user a b  //若再加入相同的元素，则重复无效，返回0
 smember user //获取user的所有元素,返回结果无序
 srem user a     //返回1，删除a元素
 scard user       //返回2，计算元素个数
```
场景
```
标签，社交，查询有共同兴趣爱好的人,智能推荐
使用方式：
给用户添加标签：
  sadd user:1:fav basball fball pq
  sadd user:2:fav basball fball   
或给标签添加用户
  sadd basball:users user:1 user:2
  sadd fball:users user:1 user:2
计算出共同感兴趣的人：
  sinter user:1:fav user:2:fav
```

#### 数据类型－有序集合(ZSet)

常用于排行榜，如视频网站需要对用户上传视频做排行榜，或点赞数与集合有联系，不能有重复的成员。
```
添加     键    分数  key 
ZADD page_rank 15 1223 
指令：   
   zadd key score member [score member......]
   zadd user:zan 200 yakax            //yakax的点赞数1, 返回操作成功的条数1
   zadd user:zan 200 yakax 120 mike 100 lee    // 返回3
   zadd test:1 nx 100 yakax                     //键test:1必须不存在，主用于添加
   zadd test:1 xx incr 200 yakax              //键test:1必须存在，主用于修改,此时为300
   zadd test:1 xx ch incr -299 yakax       //返回操作结果1，300-299=1
   zrange test:1 0 -1 withscores   //查看点赞（分数）与成员名
   zcard test:1       //计算成员个数， 返回1
排名场景：
   zadd user:3 200 yakax 120 mike 100 lee      //先插入数据
   zrange user:3 0 -1 withscores              //查看分数与成员
  zrank user:3 yakax           //返回名次：第3名返回2，从0开始到2，共3名
  zrevrank user:3 yakax     //返回0， 反排序，点赞数越高，排名越前

```
##### 差别

| **数据结构** | **是否允许元素重复** | **是否有序** | **有序实现方式** | **应用场景**         |
| ------------ | -------------------- | ------------ | ---------------- | -------------------- |
| **列表**     | **是**               | **是**       | **索引下标**     | **时间轴，消息队列** |
| **集合**     | **否**               | **否**       | **无**           | **标签，社交**       |
| **有序集合** | **否**               | **是**       | **分值**         | **排行榜，点赞数**   |

#### 位图-BitMaps
基于字符串的位操作的数据类型
```
set k1 a
获取 value 在 offset 处的值（a 对应的 ASCII 码是 97，转换为二进制数据是 01100001）
getbit k1 0
修改二进制数据（b 对应的 ASCII 码是 98，转换为二进制数据是 01100010）
setbit k1 6 1
setbit k1 7 0
get k1
统计二进制位中 1 的个数
bitcount k1
获取第一个 1 或者 0 的位置
bitpos k1 1
bitpos k1 0
BITOP 命令支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数：
BITOP AND destkey srckey1 … srckeyN ，对一个或多个 key 求逻辑与，并将结果保存到 destkey
BITOP OR destkey srckey1 … srckeyN，对一个或多个 key 求逻辑或，并将结果保存到 destkey
BITOP XOR destkey srckey1 … srckeyN，对一个或多个 key 求逻辑异或，并将结果保存到 destkey
BITOP NOT destkey srckey，对给定 key 求逻辑非，并将结果保存到 destkey
```

#### 约数统计-Hyperloglogs
用最少的内存统计大数据量的数据
```
redis> PFADD  nosql  "Redis"  "MongoDB"  "Memcached"
(integer) 1
redis> PFADD  RDBMS  "MySQL" "MSSQL" "PostgreSQL"
(integer) 1
redis> PFMERGE  databases  nosql  RDBMS
OK
redis> PFCOUNT  databases
(integer) 6
```

#### 地理位置-GEO
这个可以解决我们业务中用户的点位设置，以及点位附近的其他用户运算
```
redis> GEOADD Sicily 13.583333 37.316667 "Agrigento"
(integer) 1
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2
redis> GEORADIUSBYMEMBER Sicily Agrigento 100 km
1) "Agrigento"
2) "Palermo"
```

#### Streams-Redis流
5.0 推出的数据类型。支持多播的可持久化的消息队列，用于实现发布订阅功能，借鉴了 kafka 的设计。

### 全局命令
```
查看所有键：
  keys *
键总数 ：
  dbsize         //2个键，如果存在大量键，线上禁止使用此指令
检查键是否存在：
  exists key    //存在返回1，不存在返回0
删除键：
  del key       //del hello school, 返回删除键个数，删除不存在键返回0
键过期：
  expire key seconds   //set name test  expire name 10,表示10秒过期
  ttl key              // 查看剩余的过期时间
键的数据结构类型：
  type key      //返回string,键不存在返回none
```

### 数据库命令

| redis数据库管理方式 |            |
| --------------------------- | ---------- |
| select 0                    | 切换数据库 |
| flushdb                     | 清空当前库 |
| flushall                    | 清空全部   |
| dbsize                      | 数据库大小 |

### 持久化机制

redis是一个支持持久化的内存数据库,也就是说redis需要经常将内存中的数据同步到磁盘来保证持久化，持久化可以避免因进程退出而造成数据丢失；

```
RDB持久化把当前进程数据生成快照（.rdb）文件保存到硬盘的过程，有手动触发和自动触发
   手动触发有save和bgsave两命令 
   save命令：阻塞当前Redis，直到RDB持久化过程完成为止，若内存实例比较大会造成长时间阻塞，线上环境不建议用它
   bgsave命令：redis进程执行fork操作创建子线程，由子线程完成持久化，阻塞时间很短（微秒级），是save的优化,在执行redis-cli shutdown关闭redis服务时，如果没有开启AOF持久化，自动执行bgsave;
```

##### RDB
命令：config set dir /usr/local  //设置rdb文件保存路径
备份：bgsave  //将dump.rdb保存到usr/local下
恢复：将dump.rdb放到redis安装目录与redis.conf同级目录，重启redis即可

- 优点：
压缩后的二进制文，适用于备份、全量复制，用于灾难恢复
加载RDB恢复数据远快于AOF方式

- 缺点：
无法做到实时持久化，每次都要创建子进程，频繁操作成本过高
保存后的二进制文件，存在老版本不兼容新版本rdb文件的问题

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/2.png)

##### AOF
针对RDB不适合实时持久化，redis提供了AOF持久化方式来解决
开启：redis.conf设置：appendonly yes  (默认不开启，为no)
默认文件名：appendfilename "appendonly.aof"  
- 所有的写入命令(set hset)会append追加到aof_buf缓冲区中
- AOF缓冲区向硬盘做sync同步
- 随着AOF文件越来越大，需定期对AOF文件rewrite重写，达到压缩
- 当redis服务重启，可load加载AOF文件进行恢复
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/1.png)
配置详解
```
appendonly yes           //启用aof持久化方式
# appendfsync always //每收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用
appendfsync everysec //每秒强制写入磁盘一次，性能和持久化方面做了折中，推荐(默认)
no-appendfsync-on-rewrite  yes  //正在导出rdb快照的过程中,要不要停止同步aof
auto-aof-rewrite-percentage 100  //aof文件大小比起上次重写时的大小,增长率100%时,重写
auto-aof-rewrite-min-size 64mb   //aof文件,至少超过64M时,重写
aof如何恢复-----------------------------------------
1. 设置appendonly yes；
2. 将appendonly.aof放到dir参数指定的目录；
3. 启动Redis，Redis会自动加载appendonly.aof文件。
因为AOF的特性的原因，aof文件会越来越大，每次命令都是单独记录的所有redis提供了一个重写操作
BGREWRITEAOF 
执行一个 AOF文件 重写操作。重写会创建一个当前 AOF 文件的体积优化版本。
即使 BGREWRITEAOF 执行失败，也不会有任何数据丢失，因为旧的 AOF 文件在 BGREWRITEAOF 成功之前不会被修改。
```
**如果 AOF 文件出错了，怎么办**
服务器可能在程序正在对 AOF 文件进行写入时停机， 如果停机造成了 AOF 文件出错（corrupt）， 那么 Redis 在重启时会拒绝载入这个 AOF 文件， 从而确保数据的一致性不会被破坏。
```
为现有的 AOF 文件创建一个备份。
使用 Redis 附带的 redis-check-aof 程序，对原来的 AOF 文件进行修复。
$ redis-check-aof --fix
可以使用 diff -u 对比修复后的 AOF 文件和原始 AOF 文件的备份，查看两个文件之间的不同之处。
重启 Redis 服务器，等待服务器载入修复后的 AOF 文件，并进行数据恢复
```

##### 持久化顺序

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/3.png)
