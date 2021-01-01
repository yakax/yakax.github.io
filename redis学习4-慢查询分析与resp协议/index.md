# Redis学习(4)-慢查询分析与RESP协议



### 慢查询

#### Redis慢查询分析

与MySQL一样:当执行时间超过极大值时，会将发生时间、耗时、 命令记录；

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/10.png)

redis命令生命周期：发送 排队 执行 返回,慢查询只统计**第3个**执行步骤的时间

#### Redis如何设置

1. 动态设置6379:> config set slowlog-log-slower-than 10000  //10毫秒
使用config set完后,若想将配置持久化保存到redis.conf，要执行config rewrite ;前提是你根据redis.conf 执行

2. redis.conf修改：找到slowlog-log-slower-than 10000 ，修改保存即可
注意：slowlog-log-slower-than =0记录所有命令 -1命令都不记录

<!--more-->
#### Redis慢查询原理

慢查询记录也是存在队列里的，slow-max-len 存放的记录最大条数，比如设置的slow-max-len＝10，当有第11条慢查询命令插入时，队列的第一条命令就会出列，第11条入列到慢查询队列中， 可以config set动态设置，也可以修改redis.conf

```
获取队列里慢查询的命令：slowlog get
获取慢查询列表当前的长度：slowlog len
对慢查询列表清理（重置）：slowlog reset
对于线上slow-max-len配置的建议：线上可加大slow-max-len的值，记录慢查询存 长命令时redis会做截断，不会占用大量内存，线上可设置1000以上
对于线上slowlog-log-slower-than配置的建议：默认为10毫秒，根据redis并发量来调整，对于高并发比建议为1毫秒
慢查询是先进先出的队列，访问日志记录出列丢失，需定期执行slowlog get,将结果          存储到其它设备中
```

#### Redis性能测试(在外部请求)

1. redis-benchmark -h 192.168.42.111 -p 6379 -c 100 -n 10000
100个并发连接，10000个请求，检测服务器性能 
2. redis-benchmark -h 192.168.42.111 -p 6379 -q -d 100
测试存取大小为100字节的数据包的性能
3. redis-benchmark -h 192.168.42.111 -p 6379 -t set,get -n 100000 -q
只测试 set,lpush操作的性能
4. redis-benchmark -h 192.168.42.111 -p 6379 -n 100000 -q script load "redis.call('set','foo','bar')"
只测试某些数值存取的性能

### RESP协议

Redis服务器与客户端通过RESP（Redis Protocol specification）协议通信 (aof 文件就是通过RESP存储的)

- Simple to implement. 
- Fast to parse. 
- Human readable. 

好实现、解析快、易于理解

```
set test 5 我执行了这个命令 产生的RESP 可以在aof文件里面查看
*3  -----表示后面有几组数据 set test 5  所以就是3
$3  -----set 的长度 为3
set -----执行的命令
$4  -----test的长度 为4
test ----执行的命令
$1   ----5的长度
5    ----执行的命令
```

#### 应用

我们可以通过这个在客户端产生成这样的数据通过socket连接redis 发送给他(jedis客户端的原理就是这样)

我们也可以通过把数据库数据查询出来格式化成resp的格式请求redis(就做到了备份数据库)

```
mysql -utest -ptest stress --default-character-set=utf8 --skip-column-names --raw < order.sql | redis-cli -h 192.168.42.111 -p 6379 -a 12345678  --pipe
1. 使用用户名和密码登陆连接数据库
2. 登陆成功后执行order.sql的select语句得到查询结果集result
3. 使用密码登陆Redis
4. Redis登陆成功后, 使用PIPE管道将result导入Redis.
```

#### PIPELINE操作流程

大多数情况下，我们都会通过请求-相应机制去操作redis。只用这种模式的一般的步骤是，先获得jedis实例，然后通过jedis的get/put方法与redis交互。由于redis是单线程的，下一次请求必须等待上一次请求执行完成后才能继续执行。然而使用Pipeline模式，客户端可以一次性的发送多个命令，无需等待服务端返回。这样就大大的减少了网络往返时间，提高了系统性能。**但是要用好、用对还是得深度学习**

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/11.png)

流程

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/12.png)

##### 使用PIPELINE可以解决网络开销的问题

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/13.png)

原理也非常简单,流程如下, 将多个指令打包后,一次性提交到Redis, 网络通信只有一次

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/14.png)

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/15.png)
