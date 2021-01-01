# JVM调优学习之旅-(6)分析工具使用


## 工具介绍

### jstat

```shell
jstat -gc PID
参数说明：
S0C：这是From Survivor区的大小
S1C：这是To Survivor区的大小
S0U：这是From Survivor区当前使用的内存大小
S1U：这是To Survivor区当前使用的内存大小
EC：这是Eden区的大小
EU：这是Eden区当前使用的内存大小
OC：这是老年代的大小
OU：这是老年代当前使用的内存大小
MC：这是方法区（永久代、元数据区）的大小
MU：这是方法区（永久代、元数据区）的当前使用的内存大小
YGC：这是系统运行迄今为止的Young GC次数
YGCT：这是Young GC的耗时
FGC：这是系统运行迄今为止的Full GC次数
FGCT：这是Full GC的耗时
GCT：这是所有GC的总耗时
```

<!--more-->

常用

```
jstat -gccapacity PID：堆内存分析
jstat -gcnew PID：年轻代GC分析，这里的TT和MTT可以看到对象在年轻代存活的年龄和存活的最大年龄
jstat -gcnewcapacity PID：年轻代内存分析
jstat -gcold PID：老年代GC分析
jstat -gcoldcapacity PID：老年代内存分析
jstat -gcmetacapacity PID：元数据区内存分析
jstat -gc PID 1000 10 每隔一秒展示10次
```

### jmap、jhat

```
jmap -histo PID  可以看到对象内存分部从大到小排序
详细的 可以用
jmap -dump:live,format=b,file=dump.hprof PID 查看堆内存快照
jhat dump.hprof -port 7000 通过web界面查看默认7000端口

视图化分析工具我采用MAT
```



## jstat分析优化

### 模拟代码

```java
public class Demo2 {
    public static void main(String[] args) throws Exception {
        Thread.sleep(30000);
        while (true) {
            loadData();
        }

    }
    private static void loadData() throws InterruptedException {
        byte[] data = null;
        for (int i = 0; i < 4; i++) {
            data = new byte[10 * 1024 * 1024];
            data = null;
            byte[] datal = new byte[10 * 1024 * 1024];
            byte[] data2 = new byte[10 * 1024 * 1024];
            byte[] data3 = new byte[10 * 1024 * 1024];
            data3 = new byte[10 * 1024 * 1024];
            Thread.sleep(1000);
        }
    }
}
```

### jvm参数

```
-XX:NewSize=104857600
-XX:MaxNewSize=104857600
-XX:InitialHeapSize=209715200
-XX:MaxHeapSize=209715200
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
-XX:PretenureSizeThreshold=20971520
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:jstat.log
```

大对象我设置的20m。避免我们程序里分配的 大对象直接进入老年代

### 分析

jstat -gc pid 1000  每隔一秒打印分析 日志

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/jstat/TIM%E6%88%AA%E5%9B%BE20200811145958.png)

可以看到程序在第一次循环放了50M对象进入 此时EU是 57753 

而随后执行了一次YGC ，从代码可以看出此时只有10M进入 老年代

```
第一次循环 50m
第二次循环时：data：10m  data1：10m 这时已经 70m了 data2放不下了。执行YGC此时存活下来的就是data1还有引用
随后 data2 data3 总共30m 进入eden区
```

**从这一点 就可以看出 分配很不对了。回收的对象根本放不进 s区**

我们再看 FGC规律

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/jstat/fgc.png)

老年代总共就100MB左右，已经占用了60MB了，此时如果发生一次Young GC，有30MB存活对象要放入老年代的话，你还要放30MB对象，明显老年代就要不够了，或者老年代存在碎片放不下新进来的对象了，此时进行了Full GC，

可以看到，按照这段代码，几乎是每秒新增80MB左右，触发每秒1次Young GC，每次Young GC后存活下来 20MB~30MB的对象，老年代每秒新增20MB~30MB的对象，触发老年代几乎三秒一次Full GC。

**从上图也可以看出每次Full GC都是由Young GC触发的，因为Young GC过后存活对象太多要放入老年代，老年代内存不够了触 发Full GC，所以必须得等Full GC执行完毕了，Young GC才能把存活对象放入老年代，才算结束。这就导致Young GC也是速度非常慢**



### 改进参数

```
-XX:NewSize=209715200
-XX:MaxNewSize=209715200
-XX:InitialHeapSize=314572800
-XX:MaxHeapSize=314572800
-XX:SurvivorRatio=2
-XX:MaxTenuringThreshold=15
-XX:PretenureSizeThreshold=20971520
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:jstat.log
```

我们把堆大小调大为了300MB，年轻代给了200MB，同时“-XX:SurvivorRatio=2”表明，Eden:Survivor:Survivor的比例为2:1:1， 所以Eden区是100MB，每个Survivor区是50MB，老年代也是100MB

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/jstat/2A.png)

可以看出我们除了一些未知对象，我们每次YGC的对象都完美进入s区。每次YGC时间基本很短
