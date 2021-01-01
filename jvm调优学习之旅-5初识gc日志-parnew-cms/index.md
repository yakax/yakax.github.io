# JVM调优学习之旅-(5)初识gc日志 ParNew+CMS



## 模拟GC

### JVM参数

```java
-XX:NewSize=5242880 -XX:MaxNewSize=5242880 -XX:SurvivorRatio=8 
-XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 
-XX:PretenureSizeThreshold=10485760 
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```
<!--more-->

参数说明

-XX:NewSize **初始新生代大小 5M**

-XX:MaxNewSize **最大新生代大小 5M**

-XX:SurvivorRatio **Eden区与s1 s2 区的比例**

-XX:InitialHeapSize **初始堆大小 10M**

-XX:MaxHeapSize **最大堆大小 10M**

-XX:+UseParNewGC -XX:+UseConcMarkSweepGC 新生代parnew 老年代 cms

-XX:PretenureSizeThreshold=10485760 **指定了大对象阈值是10MB**

-XX:+PrintGCDetils：打印详细的gc日志 

-XX:+PrintGCTimeStamps：这个参数可以打印出来每次GC发生的时间 

-Xloggc:gc.log：这个参数可以设置将gc日志写入一个磁盘文件

### 示例代码

```java
public class Demo1 {
    public static void main(String[] args) {
        byte[] array1 = new byte[1024 * 1024];
        array1 = new byte[1024 * 1024];
        array1 = new byte[1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[2 * 1024 * 1024];
    }
}
```

### gc日志

```
Java HotSpot(TM) 64-Bit Server VM (25.231-b11) for windows-amd64 JRE (1.8.0_231-b11), built on Oct  5 2019 03:11:30 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 33359028k(19834688k free), swap 35456180k(14483660k free)
CommandLine flags: -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 -XX:MaxNewSize=5242880 -XX:NewSize=5242880 -XX:OldPLABSize=16 -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:-UseLargePagesIndividualAllocation -XX:+UseParNewGC 
0.103: [GC (Allocation Failure) 0.103: [ParNew: 3684K->512K(4608K), 0.0018922 secs] 3684K->1647K(9728K), 0.0020309 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 4608K, used 3746K [0x00000000ff600000, 0x00000000ffb00000, 0x00000000ffb00000)
  eden space 4096K,  78% used [0x00000000ff600000, 0x00000000ff928990, 0x00000000ffa00000)
  from space 512K, 100% used [0x00000000ffa80000, 0x00000000ffb00000, 0x00000000ffb00000)
  to   space 512K,   0% used [0x00000000ffa00000, 0x00000000ffa00000, 0x00000000ffa80000)
 concurrent mark-sweep generation total 5120K, used 1135K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3143K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 343K, capacity 388K, committed 512K, reserved 1048576K
```

前面都是一目了然 我们从第六行开始看

```
0.103: [GC (Allocation Failure) 0.103: [ParNew: 3684K->512K(4608K), 0.0018922 secs] 3684K->1647K(9728K), 0.0020309 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
这是一次GC概要说明：
GC (Allocation Failure) 由于分配失败 产生一次gc 系统运行到0.103秒时ParNew 产生一次年轻代gc
年轻代空间4.5M (Eden区是4MB，两个Survivor中只有一个是可以放存活对象的，另外一个是必须一致保持空闲的)
3684K 说明已经使用了多大空间 但是gc 过后 存活下来512k
3684K->1647K(9728K) 这是整个堆gc后整体情况 gc前占用 3684k gc后还占用 1647k
Times: user=0.00 sys=0.00, real=0.00 secs 由于单位是秒  所以忽略为0 了
```

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200808144151.png" style="zoom: 67%;" />

```
Heap
 par new generation   total 4608K, used 3746K [0x00000000ff600000, 0x00000000ffb00000, 0x00000000ffb00000)
  eden space 4096K,  78% used [0x00000000ff600000, 0x00000000ff928990, 0x00000000ffa00000)
  from space 512K, 100% used [0x00000000ffa80000, 0x00000000ffb00000, 0x00000000ffb00000)
  to   space 512K,   0% used [0x00000000ffa00000, 0x00000000ffa00000, 0x00000000ffa80000)
 concurrent mark-sweep generation total 5120K, used 1135K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3143K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 343K, capacity 388K, committed 512K, reserved 1048576K
```

这是内存结束后的情况

新生代总大小4608k目前还在使用3746k 年轻代目前占用78% 里面包含了2M数组 还有未知对象 s1也占用了一个

老年代也存在1135k对象 。

具体还存在哪些对象后续用工具看清楚,先了解个大概

```
Metaspace       used 3143K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 343K, capacity 388K, committed 512K, reserved 1048576K
```

这里牵涉到了操作系统的虚拟内存的概念！

首先  Jdk8开始把类的元数据 放到本地内存（native heap），称之为MetaSpace,理论上本地内存剩余多少，MetaSpace就有多大。 而 class space 是元空间内部的。不过设置了Metaspace大小后还是会有溢出情况。



## 触发动态年龄判断

### JVM参数 其他参数看前一篇解释

```
-XX:NewSize=10485760 -XX:MaxNewSize=10485760 -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 
-XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15 -XX:PretenureSizeThreshold=10485760 
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc1.log
```

-XX:MaxTenuringThreshold=15 年龄15 

新生代我们通过“-XX:NewSize”设置为10MB了 然后其中Eden区是8MB，每个Survivor区是1MB，Java堆总大小是20MB，老年代是10MB，大对象必须超过10MB才会直接进入老年 代 

但是我们通过“-XX:MaxTenuringThreshold=15”设置了，只要对象年龄达到15岁才会直接进入老年代。 一切准备就绪，先看看我们当前的内存分配情况

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200808155702.png" style="zoom: 33%;" />

### 代码

```java
public class Demo2 {
    public static void main(String[] args) {
        byte[] array1 = new byte[2*1024 * 1024];
        array1 = new byte[2*1024 * 1024];
        array1 = new byte[2*1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[128 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];
    }
}
```

当main 方法执行到第五行的时候 给array3分配时新生代肯定放不下，前面已经放了2m +2m+2m+128k 此时 eden区可分配剩余1m左右

### gc日志

```
Java HotSpot(TM) 64-Bit Server VM (25.231-b11) for windows-amd64 JRE (1.8.0_231-b11), built on Oct  5 2019 03:11:30 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 33359028k(19075240k free), swap 35456180k(13369076k free)
CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=15 -XX:NewSize=10485760 -XX:OldPLABSize=16 -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:-UseLargePagesIndividualAllocation -XX:+UseParNewGC 
0.100: [GC (Allocation Failure) 0.100: [ParNew: 8130K->642K(9216K), 0.0008885 secs] 8130K->642K(19456K), 0.0010790 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 9216K, used 3141K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  30% used [0x00000000fec00000, 0x00000000fee70c60, 0x00000000ff400000)
  from space 1024K,  62% used [0x00000000ff500000, 0x00000000ff5a08c8, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3230K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 388K, committed 512K, reserved 1048576K
```

分析

```
0.100: [GC (Allocation Failure) 0.100: [ParNew: 8130K->642K(9216K), 0.0008885 secs] 8130K->642K(19456K), 0.0010790 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

可以看出gc前就已经占用了8130k  垃圾回收后还有642k存在

```
 par new generation   total 9216K, used 3141K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  30% used [0x00000000fec00000, 0x00000000fee70c60, 0x00000000ff400000)
  from space 1024K,  62% used [0x00000000ff500000, 0x00000000ff5a08c8, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
```

我们 正常剩余存在的对象应该就是2m+128k左右 

由于在128k产生后 分配内存失败引起gc 所以 那一部分去了 s1 也就是日志中的from  后面分配的2048k 去了 eden 区。

此时array2的对象 应该是**1岁**



### 更改代码

```java
public class Demo2 {
    public static void main(String[] args) {
        byte[] array1 = new byte[2*1024 * 1024];
        array1 = new byte[2*1024 * 1024];
        array1 = new byte[2*1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[128 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];

        array3=new byte[2*1024*1024];
        array3=new byte[2*1024*1024];
        array3=new byte[128*1024];
        array3=null;
        byte[] array4 = new byte[2 * 1024 * 1024];
    }
}
```

此时的内存分布图 **还没执行最后一行**

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200808161814.png" style="zoom:33%;" />

### 上面代码再次执行gc日志

```
0.090: [GC (Allocation Failure) 0.090: [ParNew: 8130K->637K(9216K), 0.0009362 secs] 8130K->637K(19456K), 0.0011031 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.092: [GC (Allocation Failure) 0.092: [ParNew: 7150K->366K(9216K), 0.0033473 secs] 7150K->990K(19456K), 0.0033991 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 9216K, used 2552K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee225d0, 0x00000000ff400000)
  from space 1024K,  35% used [0x00000000ff400000, 0x00000000ff45bb38, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 concurrent mark-sweep generation total 10240K, used 623K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3230K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 388K, committed 512K, reserved 1048576K
```

从这里可以看出 cms 已经有600多k的老年代了 这600多k其实就是第一次gc的时候进入s区的对象，

CMS管理的老年代，此时使用空间刚好是600多k，证明此时Survivor里的对象触发了动态年龄判定规则，虽然没有达到15岁，但是全部进入老年代了。

**接着其实此时会发现Survivor区域中的对象都是存活的，而且总大小超过s区50%了，之前对象年龄都是1岁 此时根据动态年龄判定规则：年龄1+年龄2+年龄n的对象总大小超过了Survivor区域的50%，年龄n及以上的对象进入老 年代。 当然这里的对象都是年龄1的，所以老年龄进入老年代了。**



### 再次改变代码

```java
public class Demo2 {
    public static void main(String[] args) {
        byte[] array1 = new byte[2 * 1024 * 1024];
        array1=new byte[2*1024*1024];
        array1=new byte[2*1024*1024];
        byte[] array2 = new byte[128 * 1024];
        array2=null;
        byte[] array3=new byte[2*1024*1024];
    }
}
```

### gc日志

```
CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=15 -XX:NewSize=10485760 -XX:OldPLABSize=16 -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:-UseLargePagesIndividualAllocation -XX:+UseParNewGC 
0.119: [GC (Allocation Failure) 0.119: [ParNew: 8130K->659K(9216K), 0.0020862 secs] 8130K->2709K(19456K), 0.0022548 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 9216K, used 3158K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  30% used [0x00000000fec00000, 0x00000000fee70c60, 0x00000000ff400000)
  from space 1024K,  64% used [0x00000000ff500000, 0x00000000ff5a4cc0, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 2050K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3230K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 388K, committed 512K, reserved 1048576K
```

这里出现一个**特点** <font color=red>Young GC过后存活对象放不下Survivor区域，从而部分对象会进入老年代</font>

在gc 的时候会发现 有659k未知对象和2M的数组要放进  Survivor 区 放不下 会有部分对象进入老年代。



## fullGc之老年代放不下

### JVM参数 其他参数看前一篇解释

```
-XX:NewSize=10485760 -XX:MaxNewSize=10485760 -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 
-XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=15 
-XX:PretenureSizeThreshold=3145728 
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc2.log
```

上面注意一点 **大对象给的 3M**

### 代码

```java
public class Demo3 {
    public static void main(String[] args) {
        byte[] array1 = new byte[4 * 1024 * 1024];
        array1=null;
        byte[] array2 = new byte[2 * 1024 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];
        byte[] array4 = new byte[2 * 1024 * 1024];
        byte[] array5 = new byte[128 * 1024];
        byte[] array6 = new byte[2 * 1024 * 1024];
    }
}
```

### gc 日志

```
0.130: [GC (Allocation Failure) 0.130: [ParNew (promotion failed): 8130K->8803K(9216K), 0.0034184 secs]0.134: [CMS: 8194K->6774K(10240K), 0.0030527 secs] 12226K->6774K(19456K), [Metaspace: 3223K->3223K(1056768K)], 0.0070386 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 par new generation   total 9216K, used 2422K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  29% used [0x00000000fec00000, 0x00000000fee5d898, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 6774K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3230K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 388K, committed 512K, reserved 1048576K
```

最后一句代码还没执行时 gc前

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200808171219.png" style="zoom:50%;" />

### 分析

接着会执行如下代码：byte[] array6 = new byte[2 * 1024 * 1024];。此时，Eden区 已经放不下了。因此此时会直接触发一次Young GC。 

我们看下面的GC日志：0.130: [GC (Allocation Failure) 0.130: [ParNew (promotion failed): 8130K->8803K(9216K), 0.0034184 secs]

<font color=red>这行日志显示了，Eden区原来是有8130K的对象，但是回收之后发现一个都回收不掉，因为上述几个数组都被变 量引用了。</font>

此时，一定会直接把这些对象放入到老年代里去，但是此时老年代里已经有一个4MB的数组了，明显放不下3个2MB的数组和1个128KB的数组；

[CMS: 8194K->6774K(10240K), 0.0030527 secs] 12226K->6774K(19456K), [Metaspace: 3223K->3223K(1056768K)], 0.0070386 secs]

此时执行了CMS垃圾回收器的Full GC，Full GC其实就是会对老年代进行Old GC， 同时一般会跟一次Young GC关联，还会触发一次元数据区（永久代）的GC。

这里看到老年代从8MB左右的对象占用，变成了6MB左右的对象占用 为什么？？？

```
首先 执行young gc后 放了两个2M的数组进入老年代 ，发现无法继续再次放数据了 
这时 执行了 old gc 回收了4m的大对象 因为此时4m的大对象是没引用的,可以回收。
然后把剩余的对象放进老年代 差不多2M+2m+2m+128k和一些未知对象存活在老年代。
```


