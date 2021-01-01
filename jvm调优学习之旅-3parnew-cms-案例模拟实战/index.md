# JVM调优学习之旅-(3)ParNew+CMS 案例模拟实战



#### 场景

500万日活用户，每个用户平均访问20来次的一个系统，按10%的付费转化率来计算，每天应该产生50万订单，50万订单一般集中在4小时，平均下来每秒几十个订单。

当在高峰期的时候每秒有1000个下单请求：按3台机器算，每台机器抗300个请求左右。机器4核8G。

<!--more-->

##### 估算内存

每个订单按1kb，300个订单300kb
算上订单连带对象 订单库存等，放大20倍  查询对象再放大10倍 

```
300kb*20*10=60mb  对象一秒后消失
```

系统给JVM内存是4G  我们分配给堆新生代1.5g 老年代1.5g 然后再给永久代256M内存,基本上这4G内存就差不多了,其他给JVM其他。

```
“-Xms3072M -Xmx3072M -Xmn1536M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M”
```

##### Minor GC

```
一秒60M那么我们差不多在25秒的时候就会占满新生代：
因为JDK1.6以后默认开启空间担保了的。主要就是比较 “老年代可用空间大小”和“历次Minor GC后进入老年代对象的平均大小”，刚开始肯定这个检查是可以通过的
```

由于设置了新生代各比例8:1:1将会在20秒的时候进入minor gc 过后会存在100m左右垃圾

```
-Xms3072M -Xmx3072M -Xmn1536M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -
XX:SurvivorRatio=8
```

**由于s区每个区域150m 每次垃圾回收会有100m进入过来 由于动态年龄规划问题，这些对象会直接进入老年代。**

**调整新生代2g 老年代1g 这时同年龄问题已经解决**

```
“-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -
XX:SurvivorRatio=8”
```

##### Full GC

由于有的对象还是会逃过15次年轻代GC进入老年代比如controller 得类等等需要长期存活的类

这个时候我们可以提前让这些对象进入老年代。让他们尽快不占用新生代内存。

```
“-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -
XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5”
```

##### 大对象问题

由于这系统大对象会比较少，我们设置1M

```
“-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -
XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 -XX:PretenureSizeThreshold=1M”
```

##### 大促时多久会执行一次Full GC

基本可以确认默认开启了空间担保参数。此外 现在 新生代 s1和s2区是200m 

此外就是Minor GC过后可能存活的对象超过200MB放不下Survivor了，或者是一下子占到超过Surviovr的50%，此时 会有一些对象进入老年代中。 但是我们之前对新生代的JVM参数进行优化，就是为了避免这种情况，经过我们的测算，这种概率应该是很低的。 但是虽说是很低，也不能完全是是没有这种情况，比如某一次GC过后可能刚好机缘巧合有超过200MB对象，就会进入 老年代里。我们可以做一个假设，大概就是这个订单系统在大促期间，**每隔5分钟会在Minor GC**之后有一小批对象进入老年代， 大概200MB左右的大小。

1. 每次Minor GC之前，都检查一下“老年代可用内存空间” < “历次Minor GC后升入老年代的平均对象大小”

2. 设置了“-XX:CMSInitiatingOccupancyFaction”参数，比如设定值为92%，那么此时可能前面几个条件都没满 足，但是刚好发现这个条件满足了，比如就是老年代空间使用超过92%了，此时就会自行触发Full GC

   ```
   可以大概估算一哈 差不多40分钟左右才有可能会产生full GC 目前就先估算
   经过前面的推算，我们基本可知道，老年代大概有900MB的对象了，剩余可用空间仅仅只有100MB了，此时就会触发一次Full GC，这个时候新生代正好有200M进入过来。这种可能 但是概率相对较少。
   ```

   ```
   “-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 -XX:PretenureSizeThreshold=1M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFaction=92 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0”
   ```

   UseCMSCompactAtFullCollection 与 CMSFullGCsBeforeCompaction 是搭配使用的；前者目前默认就是true了CMSFullGCsBeforeCompaction 说的是，到底还要再执行多少次full GC才会做压缩。默认是0.

##### 指定垃圾回收器

```
“-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -
XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 -XX:PretenureSizeThreshold=1M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC”
```

#### 总结

根据业务合理分配老年代新生代比例。优化之前尽量去估算一下内存大概多少。根据新生代每次产生对象与GC后剩余对象来估算老年代存储大小。**Full GC优化的前提是Minor GC的优化，Minor GC的优化的前提是合理分配内存空间，合理分 配内存空间的前提是对系统运行期间的内存使用模型进行预估**

```
优化思路其实简单来说就一句话：尽量让每次Young GC后的存活对象小于Survivor区域的50%，
都留存在年轻代里。尽量别让对象进入老年代。尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响。
```


