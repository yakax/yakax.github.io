# 并发编程学习(1)线程的创建、启动、停止


### 单核cpu执行程序的流程

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/2.png)

<font color=red>**有了进程以后，可以让操作系统从宏观层面实现多应用并发。而并发的实现是通过 CPU 时间片不端切换执行的。对于单核 CPU 来说，在任意一个时刻只会有一个进程在被CPU 调度**</font>

<!--more-->

#### 线程的出现 

1. 在多核 CPU 中，利用多线程可以实现真正意义上的并行执行
2. 在一个应用进程中，会存在多个同时执行的任务，如果其中一个任务被阻塞，将会引起不依赖该任务的任务也
   被阻塞。通过对不同任务创建不同的线程去处理，可以提升程序处理的实时性
3. 线程可以认为是轻量级的进程，所以线程的创建、销毁比进程更快

### 线程的创建
1. Runnable 接口
2. Thread类(本质上是Runnable 的实现)
3. 实现 Callable 接口通过 FutureTask 包装器来创建 Thread 线程 
4. ThreadPool
具体说明后面文章会单独学习

### Java线程的生命周期

Java 线程既然能够创建，那么也势必会被销毁，所以线程是存在生命周期的，那么我们接下来从线程的生命周期开始去了解线程。Thread类里面有个state枚举展示了所有生命周期的状态
线程一共有 6 种状态（NEW、RUNNABLE、BLOCKED、WAITING、TIME_WAITING、TERMINATED）

![](<https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/1.png>)

- NEW：初始状态，线程被构建，但是还没有调用 start 方法
- RUNNABLED：运行状态，JAVA 线程把操作系统中的就绪和运行两种状态统一称为“运行中”
- BLOCKED：阻塞状态，表示线程进入等待状态,也就是线程因为某种原因放弃了 CPU 使用权，阻塞也分为几种情况
  - 等待阻塞：运行的线程执行 wait 方法，jvm 会把当前线程放入到等待队列
  - 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被其他线程锁占用了，那么 jvm 会把当前的线程放入到锁池中
  - 其他阻塞：运行的线程执行 Thread.sleep 或者 t.join 方法，或者发出了 I/O 请求时，JVM 会把当前线程设置为阻塞状态，当 sleep 结束、join 线程终止、io 处理完毕则线程恢复
- WAITING: 无限期等待另一个线程执行特定操作的线程处于此状态
- TIME_WAITING：超时等待状态，超时以后自动返回
- TERMINATED：终止状态，表示当前线程执行完毕

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/3.png)

```java
public class ThreadStatsDemo {
    public static void main(String[] args) {
        new Thread(()->{
            while (true){
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"Time_Waiting_Thread").start();


        new Thread(()->{
            while (true){
                synchronized (ThreadStatsDemo.class){
                    try {
                        ThreadStatsDemo.class.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        },"Waiting_Thread").start();


        new Thread(new BlockDemo(),"blockThread1").start();
        new Thread(new BlockDemo(),"blockThread2").start();
    }

    static  class BlockDemo extends Thread{
        @Override
        public void run() {
            synchronized (BlockDemo.class){
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

我们jstack 进入终端看状态

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/2.gif)

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/4.png)

可以看到blockThread1线程锁住了blockdemo 所以blockThread2 无限期等待 

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/5.png)

这里线程一个个都看的很清楚，<font color=red>所以我们在定义线程时一定要定义名称！</font>

### 线程的启动原理

我们进入Thread类的start方法查看

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/6.png)

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/7.png)

用native修饰的方法只能下载hotspot源码查看  
这也就是java能在多平台运行，因为他针对不同的操作系统有不同的处理

```
http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/00cd9dc3c2b5/src/share/native/java/lang/Thread.c
```

先通过这个地址找到方法在jvm里面的对应关系

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/8.png)

在hotspot源码里面找到jvm.cpp的文件 找到JVM_StartThread 这个方法 我根据他调用的方法找到

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/9.png)

之后就没必要看了。我们只需要知道启动线程是通过JVM调用底层操作系统操作启动线程。然后回调run方法。

### 线程的终止 

```
线程的终止，并不是简单的调用 stop 命令去。虽然 api 仍然可以调用，但是和其他的线程控制方法如 suspend、
resume 一样都是过期了的不建议使用，就拿 stop 来说，stop 方法在结束一个线程时并不会保证线程的资源正常释
放，因此会导致程序可能出现一些不确定的状态。要优雅的去中断一个线程，在线程中提供了一个 interrupt方法
```

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/10.png)

这是之前通过循环读取一个值来判断是否中断循环的操作：其实thread里面的中断也是类似的思想

#### thread.isInterrupted() 中断

```java
    public static void main(String[] args) throws InterruptedException {
        Thread thread=new Thread(()->{
            while(!Thread.currentThread().isInterrupted()){//默认是false 
                i++;
            }
            System.out.println("i:"+i);
        });
        thread.start();

        TimeUnit.SECONDS.sleep(1);
        thread.interrupt(); //把isInterrupted设置成true
        System.out.println(thread.isInterrupted()); //true
    }
```

这里抛出了一个InterruptedException 这个后面要分析
我们查看 isInterrupted 这个源代码 在jvm.cpp 里面找到 JVM_IsInterrupted方法 也是调用的OS的方法 不同平台不同操作

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/1/11.png)

### 线程的复位

```java
 public static void main(String[] args) throws InterruptedException {
        Thread thread=new Thread(()->{
            while(true){//默认是false 
                if(Thread.currentThread().isInterrupted()){
                    System.out.println("before:"+Thread.currentThread().isInterrupted());
                    Thread.interrupted(); //复位- 回到初始状态
                    System.out.println("after:"+Thread.currentThread().isInterrupted());
                }
            }
        });
        thread.start();
        TimeUnit.SECONDS.sleep(1);
        thread.interrupt(); //把isInterrupted设置成true
    }
----------------
before:true
after:false

```

### InterruptedException复位异常

```java
 public static void main(String[] args) throws InterruptedException {
        Thread thread=new Thread(()->{
            while(!Thread.currentThread().isInterrupted()){//默认是false  
                try {
                    TimeUnit.SECONDS.sleep(10); //中断一个处于阻塞状态的线程。包括join/wait/queue.take..
                    System.out.println("demo");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    break;
                }
            }
        });
        thread.start();
        TimeUnit.SECONDS.sleep(1);
        thread.interrupt(); //把isInterrupted设置成true
        System.out.println(thread.isInterrupted()); //true
    }
-------
    java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.gupaoedu.vip.ExceptionThreadDemo.lambda$main$0(ExceptionThreadDemo.java:17)
	at java.lang.Thread.run(Thread.java:748)
false
```

<font color=red>这也就是所有阻塞方法都会抛出一个InterruptedException 的异常他会复位这个线程（这也就是预防阻塞线程一直阻塞的原因）</font>

但是 这个方法是不会中断线程的 他只是告诉一个信号，所以我们需要在抛出异常时 自己在catch方法里面自己选择是否终止；
Interrupted 的底层源码做了很多事情，后面还需分析。


