# Redis学习(3)-使用场景


#### Redis的常用数据类型

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/7.png)

<!--more-->

#### String 使用场景
#####  Key的设计注意事项

一般以业务功能模块： 比如购物车key: cart:001,表示1号用户的购物车,简短明了以主,节约内存。

##### 简单字符缓存

set key value   get key

##### 结构体或对象的存储

```
set  user:1 value   //value为XML或Json格式
mset user:1:name deer user:1:age 18
mget user1:name user:1:age
```

##### Redis分布式锁
##### 计数功能-点赞

INCR article:001   GET  article:001

##### 集群环境下的Session共享
使用spring session与redis完成session共享

#### Hash 使用场景

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/8.png)

##### 购物车之类--表结构的数据都可以用hash
1. 全选功能-获取所有该用户的所有购物车商品
2. 商品数量-购物车图标上要显示购物车里商品的总数
3. 删除-要能移除购物车里某个商品
4. 增加或减少某个商品的数量
如何设计实现？
```
hmset cart:001 prod:01 1 prod:02 1
指令说明：当前登录用户ID号做为KEY，商品ID号为Field, 加入购物车数量为value
```

#### List 使用场景

##### 利用List实现栈、队列
1. 先进后出：栈 =LPUSH+LPOP->FILO
2. 先进先出：队列=LPUSH + RPOP->FIFO
Blocking
MQ（阻塞队列）=LPUSH +BRPOP

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/redis-2019-926/9.png)
##### 订阅号发布消息之类的
#### Set 使用场景
##### 抽奖功能
```
微信有一个活动，活动ID为001，如何实现微信抽奖功能，基于Redis设计实现？
userId: 004 
当Lison点击参与抽奖时，数据放入set集合
      sadd act:001  004
开始抽奖2名中奖者
      srandmember act:001 2 取出两个但是不移除
      spop act:001 2 取出两个并移除
查看有多少用户参加了本次抽奖
      smembers act:001
```
##### 用户关系设计
set的特殊命令
```
setA={A,B,C}     setB={B, C}
集合与集合之间的交集
sinter setA setB－－>得到集合{B,C}
集合与集合之间的并集
sunion setA setB －－>得到集合{A,B,C}
集合与集合之间的差集
sdiff  setA setB－－>得到集合{A}
```
```
1）James老师关注的人
     sadd jamesCares lison,peter,king,av
2)  Lison老师关注的人
     sadd lisonCares james,av,cjk,king
3)  av老师关注的人
     sadd avCares deer,cjk,king
－－－－－－－－－－－－－－
1）James和lison共同关注的人
     sinter jamesCares lisonCares , 计算结果为 {av, king}
2) 我关注的人也关注他(king)
     sismember lisonCares king    lisonCares关注king没
     sismember avCares king      avCares关注king没
3）我可能认识的人
     SDIFF lisonCares jamesCares-> {james.cjk}
```
#### Zset 有序集合

##### 常用于排行榜，如视频网站需要对用户上传视频做排行榜，或点赞数与集合有联系，不能有重复的成员

##### 新闻话题榜

```
话题榜Redis设计实现, 以日期做为Key
1）点击话题增加1
zincrby topic:20190912 1 军嫂怒怼张馨予
2)  排行实现,展示今日前9排名 从大到小
zrevrange  topic:20190912 0 9  withscores
3）统计近3日点击数据
zunionstore 新key 统计的key数量  topic:20190910  topic:20190911 topic:20190912
可以加WEIGHTS改变数值
4) 展示近3日的排行前9名
zrevrange topic:20190910-20190912 0 9  withscores
```


