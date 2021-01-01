# JVM调优学习之旅-(8)P+CMS案例优化分析记录与优化总结


## 参数案例

### 案例1-线上频繁Metadata GC

错误参数 -XX:SoftRefLRUPolicyMSPerMB=0

```
由于参数设置错误，导致线上gc日志  
【Full GC（Metadata GC Threshold）xxxxx, xxxxx】
频繁清空元数据区，而这个区是存放一些加载类的信息的。
通过jstat，或者其他可视化工具
看起来Metaspace区域的内存呈现一个波动的状态，他总是会先不断增加，达到一个顶点之后，
就会把Metaspace区域给占满，然后自然就会触发一次Full GC，Full GC会带着Metaspace区域的垃圾回收，
所以接下来Metaspace区域的内存占用又变得很小了。
```

<!--more-->

#### 怎么查看到底什么类加载进入

```
-XX:+TraceClassLoading
-XX:+TraceClassUnloading
```

通过这两个参数看看加载和卸载类的情况

```
【Loaded sun.reflect.GeneratedSerializationConstructorAccessor from __JVM_Defined_Class】
```

在JVM运行期间不断地加载这个类

```
这是反射的时候会创建的一个类的信息。可以看出，JVM就是会动态的去生成一些类放入Metaspace区域里的
并且JVM自己创建的奇怪的类，他们的Class对象都是SoftReference，也就是软引用的
举个例子
Student student = new Student(）
这时Metaspace就会存在一个student类 而年轻代会存在一个student类对象并且
```

#### 错误原因


clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB。
这个公式的意思就是说，“clock - timestamp”代表了一个软引用对象他有多久没被访问过了freespace代表JVM中的空闲内存空间，SoftRefLRUPolicyMSPerMB代表每一MB空闲内存空间可以允许SoftReference对象存活多久。

举个例子，假如说现在JVM创建了一大堆的奇怪的类出来，这些类本身的Class对象都是被SoftReference软引用的。然后现在JVM里的空间内存空间有3000MB，SoftRefLRUPolicyMSPerMB的默认值是1000毫秒，那么就意味着，此时那些奇怪的

SoftReference软引用的Class对象，可以存活3000 * 1000 = 3000秒，就是50分钟左右。
当然上面都是举例而已，大家都知道，一般来说发生GC时，其实JVM内部或多或少总有一些空间内存的，所以基本上如果不是快要发生OOM内存溢出了，一般软引用也是等到内存实在放不下对象才开始回收。

所以大家就知道了，按理说JVM应该会随着反射代码的执行，动态的创建一些奇怪的类，他们的Class对象都是软引用的，正常情况下不会被回收，但是也不应该快速增长才对

但是如果 把这个参数改为0的话
直接导致clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB这个公式的右半边是
0，就导致所有的软引用对象，比如JVM生成的那些奇怪的Class对象，刚创建出来就可能被一次Young GC给带着立马回收掉一些。接着在反射调用时又不断的创建类信息到元空间(因为JDK源码里的实现有一些问题,所以导致并发环境下会重复创建一些Class)。元空间满了就FGC。

这个参数一般设置大一些就可以了，没必要频繁的去对软引用的对象做回收。



### 案例2-大对象分析

分析案例前先看运行参数

```
-Xms1536M -Xmx1536M -Xmn512M -Xss256K -XX:SurvivorRatio=5 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:CMSInitiatingOccupancyFraction=68 -XX:+UseCMSInitiatingOccupancyOnly
-XX:+CMSParallelRemarkEnabled 
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
```

堆大小1536M  年轻代512M  线程的栈大小 256k  年轻代比例5:1:1 意思就是 365:70:70左右    老年代1000m左右

> -XX:CMSInitiatingOccupancyFraction=68 -XX:+UseCMSInitiatingOccupancyOnly 老年代占用68% 是启用回收 在占用680M时

> CMSParallelRemarkEnabled 这是老年代在初始标记时采用并发标记 

**当时这个系统运行差不多30分钟左右进行一次FGC  意思就是差不多30分钟有680m左右对象进入老年代。**

我们通过jstat 在线上观察JVM运行数据

并不是每次Young GC后都有几十MB对象进入老年代的，而是偶尔一次Young GC才 会有几十MB对象进入老年代，记住，是偶尔一次！

这也就是说600多M 进入老年代是比较困难的。

继续观察发现系统运行着，会有段时间会有几百M的数据直接进入老年代。



#### <font color=red>**大对象问题**</font>

定位大对象 我是通过jstat 观察系统，一旦有大对象进入老年代。我就用jmap打印当时的内存快照。通过可视化工具 查看具体信息。分析对象来源。

这次大对象的来源是一个sql语句没加条件而把大量的数据查询出来造成的。



#### 针对本次优化

1. 让开发同学解决代码中的bug，避免一些极端情况下SQL语句里不拼接where条件，务必要拼接上where条件，不允许查询表 中全部数据。彻底解决那个时不时有几百MB对象进入老年代的问题。 

2. 年轻代明显过小，Survivor区域空间不够，因为每次Young GC后存活对象在几十MB左右，如果Survivor就70MB很容易触发 动态年龄判定，让对象进入老年代中。所以直接调整JVM参数如下： 

   ```
   -Xms1536M -Xmx1536M -Xmn1024M -Xss256K -XX:SurvivorRatio=5 -XX:PermSize=256M -XX:MaxPermSize=256M 
   -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
   -XX:CMSInitiatingOccupancyFraction=92 
   -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly 
   -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
   ```

    年轻代设置

   >直接把年轻代空间调整为700MB左右，每个Surivor是150MB左右，此时YGC过后就几十M存活对象，一般不会进入老年代

   老年代设置

   > 反之老年代就留500MB左右就足够了，因为一般不会有对象进入老年代。
   >
   > 而且调整了参数“XX:CMSInitiatingOccupancyFraction=92”  避免老年代仅仅占用68%就触发GC，现在必须要占用到92%才会触 发GC。(1.6后这个值时默认值92)



### 案例3 不要在业务代码使用System.gc()

```
他每次执行都会指挥JVM去尝试执行一次Full GC，连带年轻代、老年代、永久代都会去回收。都知道FGC卡顿时间长
```

针对这个问题，一方面大家平时写代码的时候，不要自己使用“System.gc()”去随便触发GC，

一方面可以在JVM参数中加入这 个参数：-XX:+DisableExplicitGC。这个参数的意思就是禁止显式执行GC，不允许你来通过代码触发GC。 

所以推荐大家将“-XX:+DisableExplicitGC”参数加入到自己的系统的JVM参数中，或者是加入到公司的JVM参数模板中去。避免有 的开发工程师好心办坏事，代码中频繁触发GC就不好了。



## 阶段性总结



1. 系统承载高并发请求，或者处理数据量过大，导致Young GC很频繁，而且每次Young GC过后存活对象太多，内存分配不合理， Survivor区域过小，导致对象频繁进入老年代，频繁触发Full GC。

2. 系统一次性加载过多数据进内存，搞出来很多大对象，导致频繁有大对象进入老年代，必然频繁触发Full GC 系统发生了内存泄漏，

3. 莫名其妙创建大量的对象，始终无法回收，一直占用在老年代里，必然频繁触发Full GC 

4. Metaspace（永久代）因为加载类过多触发Full GC 

5. 误调用System.gc()触发Full GC

```markdown
- 如果jstat分析发现Full GC原因是第一种，那么就合理分配内存，调大Survivor区域即可。
- 如果jstat分析发现是第二种或第三种原因，也就是老年代一直有大量对象无法回收掉，年轻代升入老年代的对象并不多，那么就dump出来内存快照，然后用视图话工具分析。
- 通过分析，找出来什么对象占用内存过多，然后通过一些对象的引用和线程执行堆栈的分析，找到哪块代码弄出来那么多的对象的。接着优化代码即可。
- 如果jstat分析发现内存使用不多，还频繁触发Full GC，必然是第四种和第五种，此时对应的进行优化即可
```

公司最好所有jvm模板参数都加上

```
禁止System.gc()，打印出来GC日志 ：注意有些公司是禁止dump日志，因为可能会有几秒卡顿。
```

基本参数

```
-XX:+CMSParallelInitialMarkEnabled表示在初始标记的多线程执行，减少STW；
-XX:+CMSScavengeBeforeRemark：在重新标记之前执行minorGC减少重新标记时间；
-XX:+CMSParallelRemarkEnabled:在重新标记的时候多线程执行，降低STW；
-XX：CMSInitiatingOccupancyFraction=92和-XX:+UseCMSInitiatingOccupancyOnly配套使用，如果不设置后者，jvm第一次会采用92%但是后续jvm会根据运行时采集的数据来进行GC周期，如果设置后者则jvm每次都会在92%的时候进行gc；
-XX:+PrintHeapAtGC:在每次GC前都要GC堆的概况输出
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/app/oom  oom时自动dump文件 这个比较重要 分析很有用
XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 多少次进行压缩
+DisableExplicitGC -XX:+PrintGCDetails -Xloggc:gc.log 禁用手动gc 打印gc日志
```

参数一定要自己设置，因为默认的参数会给予永久代 新生代的内存大小比较少。



## 系统的排查问题体系

### 一种成熟的监控方案

```
如果公司拥有Zabbix、Open-Falcon之类的监控平台。当系统有异常出现是，通过钉钉、等能联系开发的方式发送提醒信息。
```

### 机器（CPU、磁盘、内存、网络）方面

cpu 负载高的问题，是否是GC太频繁导致 通过top看各方面指标

磁盘io问题 ,磁盘空间要有好的管控 ，这一块主要是怕其他操作把磁盘写满了。

内存这块，关注JVM 内存情况 gc频率。

代码异常这些捕获到的，都得上报到专门的分析、提醒平台。

实在监控方案没得, 就只能通过oom日志分析。能有当时的dump 最好。
