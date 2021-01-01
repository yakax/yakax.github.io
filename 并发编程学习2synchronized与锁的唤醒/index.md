# 并发编程学习(2)synchronized与锁的唤醒


### synchronized 的基本认识
在多线程并发编程中 synchronized 一直是元老级角色，很多人都会称呼它为重量级锁。但是，随着 Java SE 1.6 对synchronized 进行了各种优化之后，有些情况下它就并不那么重，Java SE 1.6 中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁。

### synchronized 的基本语法 
synchronized 有三种方式来加锁，分别是
1. 修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁
2. 静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
3. 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。
```
public class SyncDemo {
    Object lock = new Object();

    public synchronized void demo1() {

    }

    public void demo2() {
        // TODO
        synchronized (this) {

        }
        // TODO
    }

    public synchronized static void demo3() {

    }

    public void demo4() {
        synchronized (SyncDemo.class) {

        }
    }

    public void demo5() {
        synchronized (lock) {

        }
    }
}
```
不同的修饰类型，代表锁的控制粒度
<!--more-->

### 锁如何存储
#### 对象在内存中的布局 
在 Hotspot 虚拟机中，对象在内存中的存储布局，可以分为三个区域:对象头(Header)、实例数据(Instance Data)、对齐填充(Padding)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/1.png)

#### MarkWord
普通对象的对象头由两部分组成，分别是markOop以及类元信息，markOop官方称为Mark Word,在Hotspot中，markOop的定义在 markOop.hpp文件中，代码如下
```
class markOopDesc: public oopDesc {
 private:
  // Conversion
  uintptr_t value() const { return (uintptr_t) this; }

 public:
  // Constants
  enum { age_bits                 = 4, // 分代年龄
         lock_bits                = 2, // 所表示
         biased_lock_bits         = 1, // 是否为偏向锁
         max_hash_bits            = BitsPerWord - age_bits - lock_bits - biased_lock_bits,
         hash_bits                = max_hash_bits > 31 ? 31 : max_hash_bits,
         cms_bits                 = LP64_ONLY(1) NOT_LP64(0),
         epoch_bits               = 2 // 偏向锁的时间戳
  };
```
Mark word记录了对象和锁有关的信息，当某个对象被synchronized关键字当成同步锁时，那么围绕这个锁的一系列操作都和Mark word有关系。Mark Word在32位虚拟机的长度是32bit、在64位虚拟机的长度是64bit。 Mark Word里面存储的数据会随着锁标志位的变化而变化，Mark Word可能变化为存储以下5中情况
32位
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/2.png)
64位的变化
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/3.png)
锁标志位的表示意义
- 锁标识 lock=00 表示轻量级锁
- 锁标识 lock=10 表示重量级锁
- 偏向锁标识 biased_lock=1表示偏向锁
- 偏向锁标识 biased_lock=0且锁标识=01表示无锁状态

<font color=red>总结一下前面的内容，synchronized(lock)中的lock可以用Java中任何一个对象来表示，而锁标识的存储实际上就是在lock这个对象中的对象头内</font>
### 为什么任何对象都可以实现锁 
首先，Java中的每个对象都派生自Object类，而每个Java Object在JVM内部都有一个native的C++对象 oop/oopDesc进行对应。 其次，线程在获取锁的时候，实际上就是获得一个监视器对象(monitor) ,monitor可以认为是一个同步对象，所有的Java对象是天生携带monitor. 在hotspot源码的 markOop.hpp文件中；多个线程访问同步代码块时，相当于去争抢对象监视器修改对象中的锁标识。
### synchronized 锁的升级
前面提到了锁的几个概念，偏向锁、轻量级锁、重量级锁。在JDK1.6之前，synchronized是一个重量级锁，性能比较差。从JDK1.6开始，为了减少获得锁和释放锁带来的性能消耗，synchronized进行了优化，引入了 偏向锁和 轻量级锁的概念。所以从JDK1.6开始，锁一共会有四种状态，锁的状态根据竞争激烈程度从低到高分别是:无锁状态->偏向锁状态->轻量级锁状态->重量级锁状态。这几个状态会随着锁竞争的情况逐步升级。为了提高获得锁和释放锁的效率，锁可以升级但是不能降级。 下面就详细讲解synchronized的三种锁的状态及升级原理

#### 偏向锁的基本原理
在大多数的情况下，锁不仅不存在多线程的竞争，而且总是由同一个线程获得。因此为了让线程获得锁的代价更低引入了偏向锁的概念。偏向锁的意思是如果一个线程获得了一个偏向锁，如果在接下来的一段时间中没有其他线程来竞争锁，那么持有偏向锁的线程再次进入或者退出同一个同步代码块，不需要再次进行抢占锁和释放锁的操作。
> 当一个线程访问加了同步锁的代码块时，会在对象头中存储当前线程的 ID，后续这个线程进入和退出这段加了同步锁的代码块时，不需要再次加锁和释放锁。而是直接比较对象头里面是否存储了指向当前线程的偏向锁。如果相等表示偏向锁是偏向于当前线程的，就不需要再尝试获得锁了

##### 偏向锁的获取
1. 首先获取锁 对象的 Markword，判断是否处于可偏向状态。（biased_lock=1、且 ThreadId 为空）
2. 如果是可偏向状态，则通过 CAS 操作，把当前线程的 ID写入到 MarkWord
-- 如果 cas 成功，那么 markword 就会变成这样。表示已经获得了锁对象的偏向锁，接着执行同步代码块
-- 如果 cas 失败，说明有其他线程已经获得了偏向锁，这种情况说明当前锁存在竞争，需要撤销已获得偏向锁的线程，并且把它持有的锁升级为轻量级锁（这个操作需要等到全局安全点，也就是没有线程在执行字节码）才能执行
3. 如果是已偏向状态，需要检查 markword 中存储的ThreadID 是否等于当前线程的 ThreadID
-- 如果相等，不需要再次获得锁，可直接执行同步代码块
-- 如果不相等，说明当前锁偏向于其他线程，需要撤销偏向锁并升级到轻量级锁

> CAS:表示自旋锁，由于线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说性能开销很大。同时，很多对象锁的锁定状态指会持续很短的时间，因此引入了自旋锁，所谓自旋就是一个无意义的死循环，在循环体内不断的重行竞争锁。当然，自旋的次数会有限制，超出指定的限制会升级到阻塞锁。

##### 偏向锁的撤销
偏向锁的撤销并不是把对象恢复到无锁可偏向状态（因为偏向锁并不存在锁释放的概念），而是在获取偏向锁的过程中，发现 cas 失败也就是存在线程竞争时，直接把被偏向的锁对象升级到被加了轻量级锁的状态。
1. 原获得偏向锁的线程如果已经退出了临界区，也就是同步代码块执行完了，那么这个时候会把对象头设置成无锁状态并且争抢锁的线程可以基于 CAS 重新偏向当前线程
2. 如果原获得偏向锁的线程的同步代码块还没执行完，处于临界区之内，这个时候会把原获得偏向锁的线程升级为轻量级锁后继续执行同步代码块

在我们的应用开发中，绝大部分情况下一定会存在 2 个以上的线程竞争，那么如果开启偏向锁，反而会提升获取锁的资源消耗。所以可以通过 jvm 参数UseBiasedLocking 来设置开启或关闭偏向锁** -XX:+UseBiasedLocking开启或者关闭**
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/%E5%81%8F%E5%90%91%E9%94%81.png)

#### 轻量级锁的基本原理 
当存在超过一个线程在竞争同一个同步代码块时，会发生偏向锁的撤销。偏向锁撤销以后对象会可能会处于两种状态
##### 轻量级锁加锁
- JVM会先在当前线程的栈帧中创建用于存储锁记录的空间(LockRecord)
- 将对象头中的Mark Word复制到锁记录中，称为Displaced Mark Word.
- 线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针
- 如果替换成功，表示当前线程获得轻量级锁，如果失败，表示存在其他线程竞争锁，那么当前线程会尝试使用CAS来获取锁，当自旋超过指定次数(可以自定义)时仍然无法获得锁，此时锁会膨胀升级为重量级锁
##### 自旋锁
轻量级锁在加锁过程中，用到了自旋锁，所谓自旋，就是指当有另外一个线程来竞争锁时，这个线程会在原地循环等待，而不是把该线程给阻塞，直到那个获得锁的线程释放锁之后，这个线程就可以马上获得锁的。注意，锁在原地循环的时候，是会消耗 cpu 的，就相当于在执行一个啥也没有的 for 循环。所以，**轻量级锁适用于那些同步代码块执行的很快的场景**，这样，线程原地等待很短的时间就能够获得锁了。自旋锁的使用，其实也是有一定的概率背景，在大部分同步代码块执行的时间都是很短的。所以通过看似无异议的循环反而能提升锁的性能。但是自旋必须要有一定的条件控制，否则如果一个线程执行同步代码块的时间很长，那么这个线程不断的循环反而会消耗 CPU 资源。默认情况下自旋的次数是 10 次，可以通过 preBlockSpin 来修改
> 在 JDK1.6 之后，引入了自适应自旋锁，自适应意味着自旋的次数不是固定不变的，而是根据前一次在同一个锁上自旋的时间以及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源

##### 轻量级锁的解锁
- JVM会先在当前线程的栈帧中创建用于存储锁记录的空间(LockRecord)
- 将对象头中的Mark Word复制到锁记录中，称为Displaced Mark Word.
- 线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针
- 如果替换成功，表示当前线程获得轻量级锁，如果失败，表示存在其他线程竞争锁，那么当前线程会尝试使用CAS来获取锁，当自旋超过指定次数(可以自定义)时仍然无法获得锁，此时锁会膨胀升级为重量级锁
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%81.png)

#### 重量级锁的基本原理 
```
  private static Object Object;
    public static void main(String[] args) {
        Object=new Object();
        synchronized (Object){
        }
    }
```
我们通过javap -v 类 来查看信息
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/4.png)
加了同步代码块以后，在字节码中会看到一个monitorenter 和 monitorexit。每一个 JAVA 对象都会与一个监视器 monitor 关联，我们可以把它理解成为一把锁，当一个线程想要执行一段被synchronized 修饰的同步方法或者代码块时，该线程得先获取到 synchronized 修饰的对象对应的 monitor。monitorenter 表示去获得一个对象监视器。monitorexit 表示释放 monitor 监视器的所有权，使得其他被阻塞的线程可以尝试去获得这个监视器
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/5.png)
> 为什么重量级锁的开销比较大呢？

原因是monitor 依赖操作系统的 MutexLock(互斥锁)来实现的，当系统检查到是重量级锁之后，会把等待想要获取锁的线程阻塞，被阻塞的线程不会消耗CPU，但是阻塞或者唤醒一个线程，都需要通过操作系统来实现，也就是相当于从用户态转化到内核态，而转化状态是需要消耗时间的

### 回顾线程的竞争机制 
```
synchronized (lock) {
 // do something java 封装的东西越简单底层封装就越复杂
}
```
1. 只有 Thread#1 会进入临界区；
2. Thread#1 和 Thread#2 交替进入临界区,竞争不激烈；
3. Thread#1/Thread#2/Thread3… 同时进入临界区，竞争激烈

#### 偏向锁
此时当 Thread#1 进入临界区时，JVM 会将 lockObject 的对象头 Mark Word 的锁标志位设为“01”，同时会用 CAS 操作把 Thread#1 的线程 ID 记录到 Mark Word 中，此时进入偏向模式。所谓“偏向”，指的是这个锁会偏向于 Thread#1，若接下来没有其他线程进入临界区，则 Thread#1 再出入临界区无需再执行任何同步操作。也就是说，若只有Thread#1 会进入临界区，实际上只有 Thread#1 初次进入
临界区时需要执行 CAS 操作，以后再出入临界区都不会有同步操作带来的开销。

#### 轻量级锁 
偏向锁的场景太过于理想化，更多的时候是 Thread#2 也会尝试进入临界区， 如果 Thread#2 也进入临界区但是Thread#1 还没有执行完同步代码块时，会暂停 Thread#1并且升级到轻量级锁。Thread#2 通过自旋再次尝试以轻量级锁的方式来获取锁

#### 重量级锁
如果 Thread#1 和 Thread#2 正常交替执行，那么轻量级锁基本能够满足锁的需求。但是如果 Thread#1 和 Thread#2同时进入临界区，那么轻量级锁就会膨胀为重量级锁，意味着 Thread#1 线程获得了重量级锁的情况下，Thread#2就会被阻塞

### Synchronized 结合 Java Object 对象中的wait,notify,notifyAll
前面看synchronized的时候，发现被阻塞的线程什么时候被唤醒，取决于获得锁的线程什么时候执行完同步代码块并且释放锁。那怎么做到显示控制呢？我们就需要借助一个信号机制:在 Object 对 象 中，提供了wait/notify/notifyall，可以用于控制线程的状态
**先看代码**
```
public class ThreadA extends Thread{
    Object lock;

    public ThreadA(Object lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        synchronized (lock){
            try {
                System.out.println("start threadA");
                lock.wait();
                System.out.println("end threadA");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
```
public class ThreadB extends Thread{
    Object lock;

    public ThreadB(Object lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        synchronized (lock){
            System.out.println("start threadB");
            lock.notify();
            System.out.println("end threadB");
        }
    }
}
```
```
    public static void main(String[] args) {
        Object lock=new Object();
        Thread threadA=new ThreadA(lock);
        threadA.start();
        Thread threadB=new ThreadB(lock);
        threadB.start();
    }
输出----------------------
start threadA
start threadB
end threadB
end threadA
```
**执行流程**
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/2/6.png)

#### wait/notify/notifyall 基本概念
wait：表示持有对象锁的线程 A 准备释放对象锁权限，释放 cpu 资源并进入等待状态。
notify：表示持有对象锁的线程 A 准备释放对象锁权限，通知 jvm 唤 醒 某 个 竞 争 该 对 象 锁 的 线 程 X,线 程 A
synchronized 代码执行结束并且释放了锁之后，线程 X 直接获得对象锁权限，其他竞争线程继续等待(即使线程 X 同步完毕，释放对象锁，其他竞争线程仍然等待，直至有新的 notify ,notifyAll 被调用)。
notifyAll：notifyall 和 notify 的区别在于，notifyAll 会唤醒所有竞争同一个对象锁的所有线程，当已经获得锁的线程A 释放锁之后，所有被唤醒的线程都有可能获得对象锁权限
注意：三个方法都必须在synchronized同步关键字所限定的作用域中调用（一定要理解同步的原因），否则会报错java.lang.IllegalMonitorStateException，意思是因为没有同步，所以线程对对象锁的状态是不确定的，不能调用这些方法。


