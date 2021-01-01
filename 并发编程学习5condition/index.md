# 并发编程学习(5)Condition


## Condition
在前面学习 synchronized 的时候，有wait/notify 的基本使用，结合 synchronized 可以实现对线程的通信。那么，既然 J.U.C 里面提供了锁的实现机制，那 J.U.C 里面应该也有提供线程通信的机制,Condition 是一个多线程协调通信的工具类，可以让某些线程一起等待某个条件(condition)，只有满足条件时，线程才会被唤醒。

<!--more-->
### 测试代码
Signal
```
public class Signal implements Runnable{
    private Lock lock;
    private Condition condition;

    public Signal(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            System.out.println("Signal--开始");
            condition.signal();
            System.out.println("Signal--结束");
        }finally {
            lock.unlock();
        }
    }
}

```
Wait
```
public class Wait implements Runnable{
    private Lock lock;
    private Condition condition;

    public Wait(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            try {
                System.out.println("wait--开始");
                condition.await();
                System.out.println("wait--结束");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }finally {
            lock.unlock();
        }
    }
}
```
测试
```
public static void main(String[] args) {
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
    new Thread(new Wait(lock, condition)).start();
    new Thread(new Conditions(lock, condition)).start();
}
------------------------输出
wait--开始
Signal--开始
Signal--结束
wait--结束
```

### condition分析
调用 Condition，需要获得 Lock 锁，所以意味着会存在一个 AQS 同步队列。先进入await方法分析
```
public final void await() throws InterruptedException {
        if (Thread.interrupted())     //这也就是表示await的线程允许被中断 这是lock的一大特性
            throw new InterruptedException(); //如果当前线程被中断，则抛出InterruptedException 
        Node node = addConditionWaiter();  //创建一个新的节点，节点状态为 condition，采用的数据结构仍然是链表
        int savedState = fullyRelease(node); //释放当前的锁，得到锁的状态，并唤醒 AQS 队列中的一个线程
        int interruptMode = 0; //如果当前节点没有在同步队列上，即还没有被 signal，则将当前线程阻塞
        while (!isOnSyncQueue(node)) { //判断这个节点是否在 AQS 队列上，第一次判断的是 false，因为前面已经释放锁了
            LockSupport.park(this); //通过 park 挂起当前线程
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        // 当这个线程醒来,会尝试拿锁, 当 acquireQueued返回 false 就是拿到锁了.
       // interruptMode != THROW_IE -> 表示这个线程没有成功将 node 入队,但 signal 执行了 enq 方法让其入队了.
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            // 将这个变量设置成 REINTERRUPT.
            interruptMode = REINTERRUPT;
       // 如果 node 的下一个等待者不是 null, 则进行清理,清理 Condition 队列上的节点 如果是 null ,就没有什么好清理了.
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        // 如果线程被中断了,需要抛出异常.或者什么都不做
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }
```
**addConditionWaiter**
这个方法的主要作用是把当前线程封装成 Node，添加到等待队列。这里的队列不再是双向链表，而是单向链表
```
private Node addConditionWaiter() {
//如 果 lastWaiter 不 等 于 空 并 且waitStatus 不等于 CONDITION 时，把冲好这个节点从链表中移除
        Node t = lastWaiter;
        // If lastWaiter is cancelled, clean out.
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();
            t = lastWaiter;
        }
   //构建一个 Node，waitStatus=CONDITION。这里的链表是一个单向的，所以相比 AQS 来说会简单很多
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        return node;
}
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/5/2.png)

**fullyRelease**
彻底的释放锁，什么叫彻底呢，就是如果当前锁存在多次重入，那么在这个方法中只需要释放一次就会把所有的重入次数归零。
```
final int fullyRelease(Node node) {
    boolean failed = true;
        try {
             // 获得重入的次数
            int savedState = getState();
            // 释放锁并且唤醒下一个同步队列中的线程
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/5/3.png)

**isOnSyncQueue**
判断当前节点是否在同步队列中，返回 false 表示不在，返回 true 表示在
如果不在 AQS 同步队列，说明当前节点没有唤醒去争抢同步锁，所以需要把当前线程阻塞起来，直到其他的线程调用 signal 唤醒
如果在 AQS 同步队列，意味着它需要去竞争同步锁去获得执行程序执行权限为什么要做这个判断呢？原因是在 condition 队列中的节
点会重新加入到 AQS 队列去竞争锁。也就是当调用 signal的时候，会把当前节点从 condition 队列转移到 AQS 队列
1. 如果 ThreadA 的 waitStatus 的状态为 CONDITION，说明它存在于 condition 队列中，不在 AQS 队列。因为AQS 队列的状态一定不可能有 CONDITION
2. 如果 node.prev 为空，说明也不存在于 AQS 队列，原因是 prev=null 在 AQS 队列中只有一种可能性，就是它是head 节点，head 节点意味着它是获得锁的节点。
3. 如果 node.next 不等于空，说明一定存在于 AQS 队列中，因为只有 AQS 队列才会存在 next 和 prev 的关系
4. findNodeFromTail，表示从 tail 节点往前扫描 AQS 队列，一旦发现 AQS 队列的节点和当前节点相等，说明节点一定存在于 AQS 队列中
```
   final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev可以是非空的，但尚未在队列中，
         * 因为将CAS放在队列中的CAS可能会失败。因此，我们必须从尾部进行遍历，
         * 以确保它实际成功。在调用这种方法时，总是接近尾部，除非CAS失败（这不太可能），
         * 它将在那里，所以我们几乎不会遍历很多
         */
        return findNodeFromTail(node);
    }
```
**Condition.signal**
await 方法会阻塞 wait，然后 Signal 抢占到了锁获得了执行权限，这个时候在 Signal 中调用了 Condition的 signal()方法，将会唤醒在等待队列中节点
```
 public final void signal() {
  //先判断当前线程是否获得了锁，这个判断比较简单，直接用获得锁的线程和当前线程相比即可 
  所以你如果没获得锁来调用这个方法是不对的，会抛这个异常
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
  //拿到 Condition队列上第一个节点
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
  }
```
**doSignal**
对 condition 队列中从首部开始的第一个 condition 状态的节点，执行 transferForSignal 操作，将 node 从 condition队列中转换到 AQS 队列中，同时修改 AQS 队列中原先尾节点的状态
```
    private void doSignal(Node first) {
        do {
        // 从 Condition 队列中删除 first 节点
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
           // 将 next 节点设置成 null
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                 (first = firstWaiter) != null);
 }
```
**AQS.transferForSignal**
```
  final boolean transferForSignal(Node node) {
  //更新节点的状态为 0，如果更新失败，只有一种可能就是节点被 CANCELLED(取消) 了
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
  //调用 enq--这是aqs的方法了，把当前节点添加到AQS 队列。并且返回返回按当前节点的上一个节点，也就是原tail 节点
        Node p = enq(node);
        int ws = p.waitStatus;
 // 如果上一个节点的状态被取消了, 或者尝试设置上一个节点的状态为 SIGNAL 失败了(SIGNAL 表示: 他的 next节点需要停止阻塞)
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        // 唤醒节点上的线程
            LockSupport.unpark(node.thread);
        return true; //如果 node 的 prev 节点已经是signal 状态，那么被阻塞的 wait 的唤醒工作由 AQS 队列来完成 
  }
```
这里执行完毕就变成了执行finally块里面的 lock.unlock();来释放锁
这个时候会判断 wait 的 prev 节点也就是 head 节点的 waitStatus，如果大于 0 或者设置 SIGNAL 失败，表示节点被设置成了 CANCELLED 状态。这个时候会唤醒wait 这个线程。否则就基于 AQS 队列的机制来唤醒，也就是等到 Signal 释放锁之后来唤醒 wait线程

**被阻塞的线程唤醒后的逻辑**
前面在 await 方法时，线程会被阻塞。而通过 signal被唤醒之后又继续回到上次执行箭头指向的checkInterruptWhileWaiting 这个方法，其实从名字就可以看出来判断线程在等待状态下是否收到中断的请求，就是 Wait线程 在 condition 队列被阻塞的过程中，有没有被其他线程触发过中断请求 
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/5/4.png)

这里需要注意的地方是，如果第一次 CAS 失败了，则不能判断当前线程是先进行了中断还是先进行了 signal 方法的调用，可能是先执行了 signal 然后中断，也可能是先执行了中断，后执行了 signal，当然，这两个操作肯定是发生在 CAS 之前。这时需要做的就是等待当前线程的 node被添加到 AQS 队列后，也就是 enq 方法返回后，返回false 告诉 checkInterruptWhileWaiting 方法返回REINTERRUPT(1)，后续进行重新中断。简单来说，该方法的返回值代表当前线程是否在 park 的时候被中断唤醒，如果为 true 表示中断在 signal 调用之前，signal 还未执行，那么这个时候会根据await的语义，在 await时遇到中断需要抛出interruptedException，返回 true 就是告诉checkInterruptWhileWaiting 返回 THROW_IE(-1)。如果返回 false，否则表示signal 已经执行过了，只需要重新响应中断即可。<font color=red>这里的主要目的就是判断线程在等待的时候如果有中断请求，就一定要把它中断了</font>
```
  private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
    final boolean transferAfterCancelledWait(Node node) {
    //使用 cas 修改节点状态，如果还能修改成功，说明线程被中断时，signal 还没有被调用。
    // 这里有一个知识点，就是线程被唤醒，并不一定是在 java 层面执行了locksupport.unpark，
    也可能是调用了线程的 interrupt()方法，这个方法会更新一个中断标识，并且会唤醒处于阻塞状态下的线程。
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node); //如果 cas 成功，则把node 添加到 AQS 队列
            return true;
        }
    // 如果 cas 失败，则判断当前 node 是否已经在 AQS 队列上，如果不在，
    则让给其他线程执行当 node 被触发了 signal 方法时，node 就会被加到 aqs 队列上
        while (!isOnSyncQueue(node))
        //循环检测 node 是否已经成功添加到 AQS 队列中。如果没有，则通过 yield，
            Thread.yield();
        return false;
    }
```
**acquireQueued**
通过AQS队列修改等待队列的线程状态
当前被唤醒的节点wait线程 去抢占同步锁。并且要恢复到原本的重入次数状态。调用完这个方法之后，AQS 队列的状态如下
将 head 节点的 waitStatus 设置为-1，Signal 状态。

**reportInterruptAfterWait**
根据 checkInterruptWhileWaiting 方法返回的中断标识来进行中断上报。如果是 THROW_IE，则抛出中断异常如果是 REINTERRUPT，则重新响应中断

### 图
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/5/1.png)


