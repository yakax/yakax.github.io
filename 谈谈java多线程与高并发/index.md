# 谈谈Java多线程与高并发

#### 目前我接触的多线程编程基本上都是基于java.util.concurrent 这个包下的开发的，下面就以这个包来分析Java的多线程与高并发

#### 分析面试题
1. 在java中wait和sleep方法的不同？
> 最大的不同是在等待时wait会释放锁，而sleep一直持有锁。Wait通常被用于线程间交互，sleep通常被用于暂停执行。
2. 创建多线程的三种方法
> 1.继承Thread()
> 2.实现Runnable()接口
> 3.实现Callable接口

3. 继承Thread与实现Runnable区别
> 类可能只要求可执行即可,因此继承整个Thread类的开销过大

4. Runnable和Callable的区别
> Runnable接口中的run()方法的返回值是void;而Callable接口中的call()方法是有返回值的,和Future/FutureTask配合可以用来获取异步执行的结果。

5. notify作用
>1.notify：唤醒一个正在wait当前对象锁的线程，并让它拿到对象锁
>2.notifyAll：唤醒所有正在wait前对象锁的线程
>3.在调用wait，notify，notifyall的时候当前线程必须获得这个对象的锁。

6. 线程的5种状态
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/8.png)
<!--more-->
7. 你有哪些多线程开发良好的实践
> 1.考虑使用线程池
> 2.优先使用volatile 保证可见性
> 3.最小化同步范围
> 4.给线程命名

#### 来认识认识java5以后提供多线程的东西：

```
ExecutorService cachedThreadPool = Executors.线程池方法
```
1. Java通过Executors提供四种线程池:
>newCachedThreadPool创建一个可缓存线程池，需要就增加，不需要就减少。
>newFixedThreadPool 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
>newSingleThreadExecutor 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
>newScheduledThreadPool 创建一个定长线程池，支持定时及周期性任务执行。(这个一般不用:一般框架就会有定时器)

2. 同步工具
>Semaphone(信号量)可以将任何一种容器变为有界阻塞容器：比如服务器登录人数过多排队等待。
>CyclicBarrier(循环屏障)他可以做出让几个线程都执行到某个地方之后，才让几个线程同时继续执行。
>CountDownLatch(倒计时锁)这个类能够使一个线程等待其他线程完成各自的工作后再执行。
>ReentrantLock(重入锁)把锁机制分为读锁、写锁；让锁更灵活
>Condition用于让指定线程等待与唤醒，按预期顺序执行，他必须和ReentrantLock重入锁配合使用。
>Callable配合Future/FutureTask获取线程信息。


3. 原子对象
> AtomicInteger、AtomicLong、AtomicBoolean（Atomic则通过 CAS（乐观锁）实现自动同步）
> 主要是i++不是原子操作，非线程安全的，多线程访问的时候需要用到synchronized关键字保持线程同步。synchronized是悲观锁，在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，代价就是效率低下

4. 并发容器
> CopyOnWriteArrayList （写复制列表）
> CopyOnWriteArraySet （写复制集合）
>  底层实现主要是操作时复制了一份出来，底层采用的是重入锁解决并发更改问题。
> ConcurrentHashMap （分段锁映射）
> hashtable锁住的是一整张hash表而ConcurrentHashMap底层默认分段分成16份分别锁住，利用乐观锁来操作效率问题。
> ConcurrentSkipListMap、ConcurrentSkipListSet、ConcurrentLinkedQueue、这三个都是有序的支持并发的。
> 具体可以看[Jdk文档](http://www.matools.com/api/java8)





