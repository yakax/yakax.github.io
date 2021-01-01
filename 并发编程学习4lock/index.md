# 并发编程学习(4)Lock


### 初步认识JUC
Java.util.concurrent 是在并发编程中比较常用的工具类，里面包含很多用来在并发场景中使用的组件。比如线程池、阻塞队列、计时器、同步器、并发集合等等。并发包的作者是大名鼎鼎的 Doug Lea。


### lock
在 Lock 接口出现之前，Java 中的应用程序对于多线程的并发安全处理只能基于synchronized 关键字来解决。但是 synchronized 在有些场景中会存在一些短板，也就是它并不适合于所有的并发场景。但是在 Java5 以后，Lock 的出现可以解决synchronized 在某些场景中的短板，它比 synchronized 更加灵活。
Lock 本质上是一个接口，它定义了释放锁和获得锁的抽象方法，定义成接口就意味着它定义了锁的一个标准规范，也同时意味着锁的不同实现。实现 Lock 接口的类有很多，以下为几个常见的锁实现

#### Lock接口
```
void lock() // 如果锁可用就获得锁，如果锁不可用就阻塞直到锁释放
void lockInterruptibly() // 和lock()方法相似, 但阻塞的线程可中断，抛 出java.lang.InterruptedException 异常
boolean tryLock() // 非阻塞获取锁;尝试获取锁，如果成功返回 true
boolean tryLock(longtimeout, TimeUnit timeUnit)//带有超时时间的获取锁方法
void unlock() // 释放锁
```
类图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/6.png)

<!--more-->
#### ReentrantLock
表示重入锁，它是唯一一个实现了 Lock 接口的类。重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数，也就是如果当前线程 t1 通过调用 lock 方法获取了锁之后，再次调用 lock，是不会再阻塞去获取锁的，直接增加重试次数就行了。synchronized 和 ReentrantLock 都是可重入锁。
#####  重入锁的设计目的 
比如调用 demo 方法获得了当前的对象锁，然后在这个方法中再去调用demo2，demo2 中的存在同一个实例锁，这个时候当前线程会因为无法获得demo2 的对象锁而阻塞，就会产生死锁。重入锁的设计目的是避免线程的死锁。
```
public class ReentrantDemo{
  public synchronized void demo(){// main获得对象锁
      System.out.println("begin:demo");
      demo2();
 }
 public void demo2(){
         System.out.println("begin:demo1");
         // 在次获得对象锁
         synchronized (this){
         }
 }
 public static void main(String[] args) {
       ReentrantDemo rd=new ReentrantDemo();
       new Thread(rd::demo).start();
       }
}
```
#### ReentrantReadWriteLock(读写锁)
重入读写锁，它实现了 ReadWriteLock 接口，在这个类中维护了两个锁，一个是 ReadLock，一个是 WriteLock，他们都分别实现了 Lock接口。读写锁是一种适合读多写少的场景下解决线程安全问题的工具，基本原则是： 读和读不互斥、读和写互斥、写和写互斥。也就是说涉及到影响数据变化的操作都会存在互斥。

##### 读写锁的设计目的 
我们以前理解的锁，基本都是排他锁，也就是这些锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个线程访问，但是在写线程访问时，所有的读线程和其他写线程都会被阻塞。读写锁维护了一对锁，一个读锁、一个写锁;一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量.
```
public class RWLock {
    static ReentrantReadWriteLock wrl=new ReentrantReadWriteLock();
    static Map<String,Object> cacheMap=new HashMap<>();
    static Lock read=wrl.readLock();
    static Lock write=wrl.writeLock();

    //线程B/C/D
    public static final Object get(String key){
        System.out.println("begin read data:"+key);
        read.lock(); //获得读锁-> 阻塞
        try {
            return cacheMap.get(key);
        }finally {
            read.unlock();
        }
    }
    //线程A
    public static final Object put(String key,Object val){
        write.lock();//获得了写锁
        try{
            return cacheMap.put(key,val);
        }finally {
            write.unlock();
        }
    }

    public static void main(String[] args) {
        wrl.readLock();//B线程 ->阻塞
        wrl.writeLock(); //A线程
        //读->读是可以共享
        //读->写 互斥
        //写->写 互斥
        //读多写少的场景
    }
}
```
在这个案例中，通过 hashmap 来模拟了一个内存缓存，然后使用读写所来保证这个内存缓存的线程安全性。当执行读操作的时候，需要获取读锁，在并发访问的时候，读锁不会被阻塞，因为读操作不会影响执行结果。在执行写操作是，线程必须要获取写锁，当已经有线程持有写锁的情况下，当前线程会被阻塞，只有当写锁释放以后，其他读写操作才能继续执行。使用读写锁提升读操作的并发性，也保证每次写操作对所有的读写操作的可见性
-  读锁与读锁可以共享
-  读锁与写锁不可以共享（排他）
-  写锁与写锁不可以共享（排他）

#### StampedLock
stampedLock 是 JDK8 引入的新的锁机制，可以简单认为是读写锁的一个改进版本，读写锁虽然通过分离读和写的功能使得读和读之间可以完全并发，但是读和写是有冲突的，如果大量的读线程存在，可能会引起写线程的饥饿。stampedLock 是一种乐观的读策略，使得乐观锁完全不会阻塞写线程

### ReentrantLock 的实现原理
#### AQS 
在 Lock 中，用到了一个同步队列 AQS，全称 AbstractQueuedSynchronizer，它是一个同步工具也是 Lock 用来实现线程同步的核心组件独占锁，每次只能有一个线程持有锁，比如前面给大家演示的 ReentrantLock 就是以独占方式实现的互斥锁共 享 锁 ， 允 许 多 个线程同时获取锁，并发访问共享资源，比如ReentrantReadWriteLock
AQS 队列内部维护的是一个 FIFO 的双向链表，这种结构的特点是每个数据结构都有两个指针，分别指向直接的后继节点和直接前驱节点。所以双向链表可以从任意一个节点开始很方便的访问前驱和后继。每个 Node 其实是由线程封装，当线程争抢锁失败后会封装成 Node 加入到 ASQ 队列中去；当获取锁的线程释放锁以后，会从队列中唤醒一个阻塞的节点(线程)
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/4.png)
#### Node 的组成 
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/5.png)
#### 释放锁以及添加线程对于队列的变化 
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/1.png)
**里会涉及到两个变化**
- 新的线程封装成 Node 节点追加到同步队列中，设置 prev 节点以及修改当前节点的前置节点的 next 节点指向自己
- 通过 CAS 讲 tail 重新指向新的尾部节点

head 节点表示获取锁成功的节点，当头结点在释放同步状态时，会唤醒后继节点，如果后继节点获得锁成功，会把自己设置为头结点，节点的变化过程如下
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/7.png)
**涉及到两个变化**
- 修改head节点指向下一个获得锁的节点
- 新的获得锁的节点，将prev的指针指向null

<font color=red>设置 head 节点不需要用 CAS，原因是设置 head 节点是由获得锁的线程来完成的，而同步锁只能由一个线程获得，所以不需要 CAS 保证，只需要把 head 节点设置为原首节点的后继节点，并且断开原 head 节点的 next 引用即可</font>

#### ReentrantLock 的源码分析
<font color=red>**ReentrantLock 默认是非公平锁，因为无论在什么场景非公平锁的效率都是会大于公平锁的。**</font>
时序图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/10.jpg)
```
public void lock(){
    sync.lock();
}
```
sync是一个静态内部类，它继承了AQS这个抽象类，前面说过AQS是一个同步工具，主要用来实现同步控制。我们在利用这个工具的时候，会继承它来实现同步控制功能。 通过进一步分析，发现Sync这个类有两个具体的实现，分别是 NofairSync(非公平锁), FailSync(公平锁).
**公平锁** 表示所有线程严格按照FIFO来获取锁
**非公平锁** 表示可以存在抢占锁的功能，也就是说不管当前队列上是否存在其他线程等待，新线程都有机会抢占锁
**NonfairSync.lock**
1. 非公平锁和公平锁最大的区别在于，在非公平锁中我抢占锁的逻辑是，不管有没有线程排队，我先上来 cas 去抢占一下
2. CAS 成功，就表示成功获得了锁
3. CAS 失败，调用 acquire(1)走锁竞争逻辑

<font color=red>非公平的AQS队列头结点的下一个节点不一定会获得锁，就是因为在非公平锁的释放后，如果节点在cas的时候。另外一个线程正好进来cas锁，如果这个节点没他快，那么锁就会被这个插队的获得</font>

```
final void lock() {
 if (compareAndSetState(0, 1))  //通过cas操作来修改state状态，表示争抢锁的操作
 setExclusiveOwnerThread(Thread.currentThread());//设置当前获得锁状态的线程
 else
 acquire(1);//尝试去获取锁
}
```
```
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
通过 cas 乐观锁的方式来做比较并替换，这段代码的意思是，如果当前内存中的state 的值和预期值 expect 相等，则替换为 update。更新成功返回 true，否则返回 false.
这个操作是原子的，不会出现线程安全问题，这里面涉及到Unsafe这个类的操作，以及涉及到 state 这个属性的意义。state 是 AQS 中的一个属性，它在不同的实现中所表达的含义不一样，对于重入锁的实现来说，表示一个同步状态。它有两个含义的表示
1. 当 state=0 时，表示无锁状态
2. 当 state>0 时，表示已经有线程获得了锁，也就是 state=1，但是因为ReentrantLock 允许重入，所以同一个线程多次获得同步锁的时候，state 会递增，比如重入 5 次，那么 state=5。而在释放锁的时候，同样需要释放 5 次直到 state=0其他线程才有资格获得锁

**Unsafe 类**
Unsafe 类是在 sun.misc 包下，不属于 Java 标准。但是很多 Java 的基础类库，包括一些被广泛使用的高性能开发库都是基于 Unsafe 类开发的，比如 Netty、Hadoop、Kafka 等；Unsafe 可认为是 Java 中留下的后门，提供了一些低层次操作，如直接内存访问、线程的挂起和恢复、CAS、线程同步、内存屏障而 CAS 就是 Unsafe 类中提供的一个原子操作，第一个参数为需要改变的对象，第二个为偏移量(即之前求出来的 headOffset 的值)，第三个参数为期待的值，第四个为更新后的值整个方法的作用是如果当前时刻的值等于预期值 var4 相等，则更新为新的期望值 var5，如果更新成功，则返回 true，否则返回 false；

**acquire(1)方法**
1. 通过 tryAcquire 尝试获取独占锁，如果成功返回 true，失败返回 false
2. 如果 tryAcquire 失败，则会通过 addWaiter 方法将当前线程封装成 Node 添加到 AQS 队列尾部
3. acquireQueued，将 Node 作为参数，通过自旋去尝试获取锁。

```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
**NonfairSync.tryAcquire**
这个方法的作用是尝试获取锁，如果成功返回 true，不成功返回 false它是重写 AQS 类中的 tryAcquire 方法，并且大家仔细看一下 AQS 中 tryAcquire方法的定义，并没有实现，而是抛出异常。
```
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```
**nonfairTryAcquire**
```
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread(); //获取当前执行的线程
            int c = getState(); //获得 state 的值
            if (c == 0) { //表示无锁状态
                if (compareAndSetState(0, acquires)) { //cas 替换 state 的值，cas 成功表示获取锁成功
                    setExclusiveOwnerThread(current); //保存当前获得锁的线程,下次再来的时候不要再尝试竞争锁
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) { //如果同一个线程来获得锁，直接增加重入次数
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
**addWaiter**
当 tryAcquire 方法获取锁失败以后，则会先调用 addWaiter 将当前线程封装成Node.
入参 mode 表示当前节点的状态，传递的参数是 Node.EXCLUSIVE，表示独占状态。意味着重入锁用到了 AQS 的独占锁功能
1. 将当前线程封装成 Node
2. 当前链表中的 tail 节点是否为空，如果不为空，则通过 cas 操作把当前线程的node 添加到 AQS 队列
3. 如果为空或者 cas 失败，调用 enq 将节点添加到 AQS 队列

```
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail; //tail 是 AQS 中表示同比队列队尾的属性，默认是 null
        if (pred != null) { //tail 不为空的情况下，说明队列中存在节点
            node.prev = pred; //把当前线程的 Node 的 prev 指向 tail
            if (compareAndSetTail(pred, node)) { //通过 cas 把 node加入到 AQS 队列，也就是设置为 tail
                pred.next = node; ;//设置成功以后，把原 tail 节点的 next指向当前 node
                return node;
            }
        }
        enq(node); ;//tail=null,把 node 添加到同步队列
        return node;
    }
```

**enq**
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
**图解分析**
假设 3 个线程来争抢锁，那么截止到 enq 方法运行结束之后，或者调用 addwaiter方法结束后，AQS 中的链表结构图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/11.png)

**acquireQueued**
通过 addWaiter 方法把线程添加到链表后，会接着把 Node 作为参数传递给acquireQueued 方法，去竞争锁
1. 获取当前节点的 prev 节点
2. 如果 prev 节点为 head 节点，那么它就有资格去争抢锁，调用 tryAcquire 抢占锁
3. 抢占锁成功以后，把获得锁的节点设置为 head，并且移除原来的初始化 head节点
4. 如果获得锁失败，则根据 waitStatus 决定是否需要挂起线程
5. 最后，通过 cancelAcquire 取消获得锁的操作

```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor(); ;//获取当前节点的 prev 节点
                if (p == head && tryAcquire(arg)) { //如果是 head 节点，说明有资格去争抢锁
                    setHead(node); //获取锁成功，也就是ThreadA 已经释放了锁，然后设置 head 为 ThreadB 获得执行权限
                    p.next = null; // help GC //把原 head 节点从链表中移除
                    failed = false;
                    return interrupted;
                }
                //ThreadA 可能还没释放锁，使得 ThreadB 在执行 tryAcquire 时会返回 false
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;//并且返回当前线程在等待过程中有没有中断过。
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
**shouldParkAfterFailedAcquire 争抢锁失败调用的方法**
如果 ThreadA 的锁还没有释放的情况下，ThreadB 和 ThreadC 来争抢锁肯定是会失败，那么失败以后会调用shouldParkAfterFailedAcquire 方法
Node 有 5 中状态，分别是：CANCELLED（1），SIGNAL（-1）、CONDITION（-2）、PROPAGATE(-3)、默认状态(0)
**CANCELLED**: 在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该 Node 的结点, 其结点的 waitStatus 为 CANCELLED，即结束状态，进入该状态后的结点将不会再变化
**SIGNAL**: 只要前置节点释放锁，就会通知标识为 SIGNAL 状态的后续节点的线程
**CONDITION**： 和 Condition 有关系，后续会讲解
**PROPAGATE**：共享模式下，PROPAGATE 状态的线程处于可运行状态
0:初始状态
1. 如果 ThreadA 的 pred 节点状态为 SIGNAL，那就表示可以放心挂起当前线程
2. 通过循环扫描链表把 CANCELLED 状态的节点移除
3. 修改 pred 节点的状态为 SIGNAL,返回 false.
<font color=red>返回 false 时，也就是不需要挂起，返回 true，则需要调用 parkAndCheckInterrupt挂起当前线程</font>

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; //前置节点的waitStatus
        if (ws == Node.SIGNAL)//如果前置节点为 SIGNAL，意味着只需要等待其他前置节点的线程被释放
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true; //返回 true，意味着可以直接放心的挂起了
        if (ws > 0) { //ws 大于 0，意味着 prev 节点取消了排队，直接移除这个节点就行
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev; //相当于: pred=pred.prev;node.prev=pred;
            } while (pred.waitStatus > 0); //这里采用循环，从双向列表中移除 CANCELLED 的节点
            pred.next = node;
        } else { //利用 cas 设置 prev 节点的状态为 SIGNAL(-1)

            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
**parkAndCheckInterrupt 中断**
如果shouldParkAfterFailedAcquire返回了true，则会执行： parkAndCheckInterrupt()方法，它是通过LockSupport.park(this)将当前线程挂起到WATING状态，它需要等待一个中断、unpark方法来唤醒它，通过这样一种FIFO的机制的等待，来实现了Lock的操作。
```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
LockSupport是Java6引入的一个类，提供了基本的线程同步原语。LockSupport实际上是调用了 Unsafe 类里的函数，归结到 Unsafe 里，只有两个函数--底层的就不分析了
```
    public native void unpark(Object var1);
    public native void park(boolean var1, long var2);
```
#### 锁的释放流程
**unlock**
```
public final boolean release(int arg) {
   if (tryRelease(arg)) { //释放锁成功
      Node h = head; //得到 aqs 中 head 节点
      if (h != null && h.waitStatus != 0) //如果 head 节点不为空并且状态！=0.调用 unparkSuccessor(h)唤醒后续节点
         unparkSuccessor(h);
        return true;
   }
  return false;
}
```
**tryRelease**
这个方法可以认为是一个设置锁状态的操作，通过将 state 状态减掉传入的参数值（参数是 1），如果结果状态为 0，就将排它锁的 Owner 设置为 null，以使得其它的线程有机会进行执行。
在排它锁中，加锁的时候状态会增加 1（当然可以自己修改这个值），在解锁的时候减掉 1，同一个锁，在可以重入后，可能会被叠加为 2、3、4 这些值，只有 unlock()的次数与 lock()的次数对应才会将 Owner 线程设置为空，而且也只有这种情况下才会返回 true
```
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
**unparkSuccessor**
```
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus; //获得 head 节点的状态
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0); ;// 设置 head 节点状态为 0

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next; //得到 head 节点的下一个节点
        if (s == null || s.waitStatus > 0) {
        //如果下一个节点为 null 或者 status>0 表示 cancelled 状态.
            //通过从尾部节点开始扫描，找到距离 head 最近的一个waitStatus<=0 的节点
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null) //next 节点不为空，直接唤醒这个线程即可
            LockSupport.unpark(s.thread);
    }
```
**为什么在释放锁的时候是从 最后节点 tail 进行扫描**
1. 将新的节点的 prev 指向 tail
2. 通过 cas 将 tail 设置为新的节点，因为 cas 是原子操作所以能够保证线程安全性
3. t.next=node；设置原 tail 的 next 节点指向新的节点

<font color=red>注意看else写的逻辑</font>
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/12.png)
<font color=red>在 cas 操作之后，t.next=node 操作之前。存在其他线程调用 unlock 方法从 head开始往后遍历，由于 t.next=node 还没执行意味着链表的关系还没有建立完整。就会导致遍历到 t 节点的时候被中断。所以从后往前遍历，一定不会存在这个问题。</font>

**图解分析**
通过锁的释放，原本的结构就发生了一些变化。head 节点的 waitStatus 变成了 0，ThreadB 被唤醒
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/13.png)
然后通过unlock后 原来被挂起的线程是在 acquireQueued 方法中自选，所以被唤醒以后继续从这个方法开始执行在acquireQueued方法中
- 设置新 head 节点的 prev=null
- 设置原 head 节点的 next 节点为 null

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/4/8.png)

#### 公平锁和非公平锁的区别
锁的公平性是相对于获取锁的顺序而言的，如果是一个公平锁，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是 FIFO。 在上面分析的例子来说，只要CAS 设置同步状态成功，则表示当前线程获取了锁，而公平锁则不一样，差异点有两个

**lock**
```
final void lock() {
 acquire(1);
}
非公平锁在获取锁的时候，会先通过 CAS 进行抢占，而公平锁则不会
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
**tryAcquire**
```
公平锁
 protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

非公平锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
不同的地方在于判断条件多了hasQueuedPredecessors()方法，也就是加入了**同步队列中当前节点是否有前驱节点**的判断，如果该方法返回 true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。


