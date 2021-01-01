# 并发编程学习(6)CountDownLatch、Semaphore、CyclicBarrier


### CountDownLatch
countdownlatch 是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完毕再执行,countdownlatch 提供了两个方法，一个是 countDown，一个是 await。countdownlatch 初始化的时候需要传入一个整数，在这个整数倒数到 0 之前，调用了 await 方法的程序都必须要等待，然后通过 countDown 来倒数

#### 示例代码
```
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(5);
        new Thread(() -> {
            System.out.println("Thread1");
            countDownLatch.countDown(); //3-1=2
            System.out.println("Thread1执行完毕");
        }).start();
        new Thread(() -> {
            System.out.println("Thread2");
            countDownLatch.countDown();//2-1=1
            System.out.println("Thread2执行完毕");
        }).start();
        new Thread(() -> {
            System.out.println("Thread3");
            countDownLatch.countDown();//1-1=0
            System.out.println("Thread3执行完毕");
        }).start();
        countDownLatch.await();
    }
输出--------------------
Thread1
Thread2
Thread2执行完毕
Thread1执行完毕
Thread3
Thread3执行完毕
-----------不会结束
```
<!--more-->

模拟高并发
```
public class CountDownLatchDemo extends Thread {

    static CountDownLatch countDownLatch = new CountDownLatch(1);

    public static void main(String[] args) {
        for(int i=0;i<10;i++){
            new CountDownLatchDemo().start();
        }
        countDownLatch.countDown();
    }

    @Override
    public void run() {
        try {
            countDownLatch.await(); //阻塞  10个线程 Thread.currentThread
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //TODO 业务代码
        System.out.println("ThreadName:" + Thread.currentThread().getName());
    }
 }
 输出-----------------
ThreadName:Thread-2
ThreadName:Thread-4
ThreadName:Thread-3
ThreadName:Thread-8
ThreadName:Thread-0
ThreadName:Thread-1
ThreadName:Thread-6
ThreadName:Thread-5
ThreadName:Thread-9
ThreadName:Thread-7
```
#### CountDownLatch 分析
我们只需要关系两个方法，一个是 countDown() 方法，另一个是 await() 方法，countDown() 方法每次调用都会将 state 减 1，直到state 的值为 0；而 await 是一个阻塞方法，当 state 减为 0 的时候，await 方法才会返回。await 可以被多个线程调用，大家在这个时候脑子里要有个图：所有调用了await 方法的线程阻塞在 AQS 的阻塞队列中，等待条件满足（state == 0），将线程从队列中一个个唤醒过来

##### await
**await()**
```
 public void await() throws InterruptedException {
     // 可中断的共享锁
    sync.acquireSharedInterruptibly(1);
 }
```
**acquireSharedInterruptibly**
```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
    //state 如果不等于 0，说明当前线程需要加入到共享锁队列中
        doAcquireSharedInterruptibly(arg);
}
```
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/6/1.png)

**doAcquireSharedInterruptibly**
```
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //创建一个共享模式的节点添加到队列中
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
    //  通过自选不断判断
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
            // 就判断尝试获取锁
                int r = tryAcquireShared(arg);
                //r>=0 表示获取到了执行权限，这个时候因为 state!=0，所以不会执行这段代码
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //在阻塞线程，这也就是为什么会捕获这个异常
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
**setHeadAndPropagate**
通过doReleaseShared()来解决唤醒 把全部节点改为head头结点
```
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
##### countDown
由于线程被 await 方法阻塞了，所以只有等到countdown 方法使得 state=0 的时候才会被唤醒
1. 只有当 state 减为 0 的时候，tryReleaseShared 才返回 true, 否则只是简单的 state = state - 1
2. 如果 state=0, 则调用 doReleaseShared唤醒处于 await 状态下的线程

**releaseShared**
```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
protected boolean tryReleaseShared(int releases) {
    // 递减计数;转换为零时发出信号
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
**doReleaseShared**
共享锁的释放和独占锁的释放有一定的差别前面唤醒锁的逻辑和独占锁是一样，先判断头结点是不是SIGNAL 状态，如果是，则修改为 0，并且唤醒头结点的下一个节点
PROPAGATE： 标识为 PROPAGATE 状态的节点，是共享锁模式下的节点状态，处于这个状态下的节点，会对线程的唤醒进行传播
```
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
            // 这个 CAS 失败的场景是：执行到这里的时候，刚好有一个节点入队，入队会将这个 ws 设置为 -1
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 如果到这里的时候，前面唤醒的线程已经占领了 head，那么再循环
        // 通过检查头节点是否改变了，如果改变了就继续循环
        if (h == head)                   // 等于head节点就退出
            break;
    }
}
```
h == head：说明头节点还没有被刚刚用unparkSuccessor 唤醒的线程（这里可以理解为ThreadB）占有，此时 break 退出循环。
h != head：头节点被刚刚唤醒的线程（这里可以理解为ThreadB）占有，那么这里重新进入下一轮循环，唤醒下一个节点（这里是 ThreadB ）。我们知道，等到ThreadB 被唤醒后，其实是会主动唤醒 ThreadC...
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/6/2.png)


### Semaphore
semaphore 也就是我们常说的信号灯，semaphore 可以控制同时访问的线程个数，通过 acquire 获取一个许可，如果没有就等待，通过 release 释放一个许可。有点类似限流的作用。叫信号灯的原因也和他的用处有关，比如某商场就 5 个停车位，每个停车位只能停一辆车，如果这个时候来了 10 辆车，必须要等前面有空的车位才能进入；比较常见的就是做限流操作；

#### 案例
```
public class SemaphoreDemo {
    //限流（AQS）
    //permits; 令牌(5)
    //公平和非公平
    static class Car extends  Thread{
        private int num;
        private Semaphore semaphore;

        public Car(int num, Semaphore semaphore) {
            this.num = num;
            this.semaphore = semaphore;
        }
        public void run(){
            try {
                semaphore.acquire(); //获得一个令牌, 如果拿不到令牌，就会阻塞
                System.out.println("第"+num+" 抢占一个车位");
                Thread.sleep(2000);
                System.out.println("第"+num+" 开走喽");
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
    // 可以基于公平与非公平锁
        Semaphore semaphore=new Semaphore(5);
        for(int i=0;i<10;i++){
            new Car(i,semaphore).start();
        }
    }

}
```
创建 Semaphore 实例的时候，需要一个参数 permits，这个基本上可以确定是设置给 AQS 的 state 的，然后每个线程调用 acquire 的时候，执行 state = state - 1，release 的时候执行 state = state + 1，当然，acquire 的时候，如果 state = 0，说明没有资源了，需要等待其他线程 release。


### CyclicBarrier
CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续工作。CyclicBarrier 默认的构造方法是 CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用 await 方法告诉 CyclicBarrier 当前线程已经到达了屏障，然后当前线程被阻塞

#### 案例
```
public class DataImportThread extends Thread{

    private CyclicBarrier cyclicBarrier;

    private String path;

    public DataImportThread(CyclicBarrier cyclicBarrier, String path) {
        this.cyclicBarrier = cyclicBarrier;
        this.path = path;
    }

    @Override
    public void run() {
        System.out.println("开始导入："+path+" 数据");
        //TODO 可以写业务
        try {
            cyclicBarrier.await(); //阻塞 condition.await()
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
public class CycliBarrierDemo extends Thread{
    @Override
    public void run() {
        System.out.println("开始进行数据分析");
    }

    //循环屏障
    //可以使得一组线程达到一个同步点之前阻塞.
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier=new CyclicBarrier
                (3,new CycliBarrierDemo());
        new Thread(new DataImportThread(cyclicBarrier,"file1")).start();
        new Thread(new DataImportThread(cyclicBarrier,"file2")).start();
        new Thread(new DataImportThread(cyclicBarrier,"file3")).start();

    }
    输出--------------------------
    开始导入：file3 数据
    开始导入：file2 数据
    开始导入：file1 数据
    开始进行数据分析
}
```
1. 对于指定计数值 parties，若由于某种原因，没有足够的线程调用 CyclicBarrier 的 await，则所有调用 await 的线程都会被阻塞
2. 同样的 CyclicBarrier 也可以调用 await(timeout, unit)，设置超时时间，在设定时间内，如果没有足够线程到达，则解除阻塞状态，继续工作
3. 通过 reset 重置计数，会使得进入 await 的线程出现BrokenBarrierException
4. 如果采用是 CyclicBarrier(int parties, RunnablebarrierAction) 构造方法，执行 barrierAction 操作的是最后一个到达的线程

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/6/3.png)
CyclicBarrier 相比 CountDownLatch 来说，要简单很多，源码实现是基于 ReentrantLock 和 Condition 的组合使用;CyclicBarrier 和 CountDownLatch 很像，只是 CyclicBarrier 可以有不止一个栅栏，因为它的栅栏（Barrier）可以重复使用（Cyclic）




