# AQS应用工具类源码学习


## AQS的共享锁 CountDownLatch

### 基本结构图

```
基本就是当state值更改为0时就开始唤醒aqs队列第一个节点 然后，就会调用 setHeadAndPropagate 这个方法 唤醒持续唤醒后继节点1。
```

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/aqs/QQ%E6%88%AA%E5%9B%BE20200922142430.png)

<!--more-->
<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/Concurrent/CountDownLatch-%E5%94%A4%E9%86%92%E6%B5%81%E7%A8%8B%E5%9B%BE.png!print" style="zoom:50%;" />


### 简单案例

```java
CountDownLatch latch = new CountDownLatch(2);

        Thread t1 = new Thread(()-> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException ignore) {
            }
            // 休息 5 秒后(模拟线程工作了 5 秒)，调用 countDown()
            latch.countDown();
        }, "t1");

        Thread t2 = new Thread(()-> {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException ignore) {
            }
            // 休息 10 秒后(模拟线程工作了 10 秒)，调用 countDown()
            latch.countDown();
        }, "t2");

        t1.start();
        t2.start();

        Thread t3 = new Thread(()-> {
            try {
                // 阻塞，等待 state 减为 0
                latch.await();
                System.out.println("线程 t3 从 await 中返回了");
            } catch (InterruptedException e) {
                System.out.println("线程 t3 await 被中断");
                Thread.currentThread().interrupt();
            }
        }, "t3");

        Thread t4 = new Thread(()-> {
            try {
                // 阻塞，等待 state 减为 0
                latch.await();
                System.out.println("线程 t4 从 await 中返回了");
            } catch (InterruptedException e) {
                System.out.println("线程 t4 await 被中断");
                Thread.currentThread().interrupt();
            }
        }, "t4");

        t3.start();
        t4.start();
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/Concurrent/栅栏demo-入队效果.png!print" style="zoom: 80%;" />

### 源码分析

CountDownLatch 里面队列也是继承aqs队列，在构造方法时把**count值赋值给state**

#### countDown()分析

```java
    // 默认传值1进来
    public final boolean releaseShared(int arg) {
   //条件成立：说明当前调用latch.countDown() 方法线程 正好是 state - 1 == 0 的这个线程，需要做触发唤醒 await状态的线程。
        if (tryReleaseShared(arg)) {
            // 唤醒wait() 阻塞队列的线程  只有正好修改为0的线程可以进行操作  
            // 这里一但唤醒后 就开始向下传播
            doReleaseShared();
            return true;
        }
        return false;
    }
	// 尝试更新 AQS.state 值，每调用一次，state值减一，当state -1 正好为0时，返回true
    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            //获取当前AQS.state
            int c = getState();
            //条件成立：说明前面已经有线程 触发 唤醒操作了，这里返回false
            if (c == 0)
                return false;

            //执行到这里，说明 state > 0
            int nextc = c-1;

            //cas成功，说明当前线程执行 tryReleaseShared 方法 c-1之前，没有其它线程 修改过 state。
            if (compareAndSetState(c, nextc))
                //nextc == 0 ：true ，说明当前调用 countDown() 方法的线程 就是需要触发 唤醒操作的线程.
                return nextc == 0;
        }
    }
```

#### await() 阻塞

```java
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        /**
         * protected int tryAcquireShared(int acquires) {
         * 		return (getState() == 0) ? 1 : -1;
         * }
         */
        // 如果不等于0  将进入开始阻塞
        if (tryAcquireShared(arg) < 0)
            // 阻塞逻辑
            doAcquireSharedInterruptibly(arg);
    }
```

```java
// 阻塞方法 与向下传播方法
private void doAcquireSharedInterruptibly(int arg)
            throws InterruptedException {
        // 创建共享节点
        final Node node = addWaiter(Node.SHARED);
        // 失败标记
        boolean failed = true;
        try {
            for (; ; ) {
                // 拿到当前线程 前驱节点 理论上是头结点
                final Node p = node.predecessor();
                if (p == head) {
                    // countdown 再次判断state值  正常在这里是返回1 state=0 返回-1
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 唤醒后逻辑 并且里面有向下传播调用唤醒
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //条件1 找前置可用节点过程 主要是找到signal节点 或者更改前置节点为signal节点
                // 条件2 阻塞线程 并返回中断状态
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

---------------------
    // 添加共享节点
     private Node addWaiter(Node mode) {
        // 将当前线程封装成node
        Node node = new Node(Thread.currentThread(), mode);
        // 尾巴结点拿出来赋值  据了解 别人这样写完全是更直观的看代码
        Node pred = tail;
        // 如果node已经存在就尝试加入尾结点后面
        if (pred != null) {
            // 设置当前节点上一个节点为 之前的尾结点
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                // 加入尾结点成功 设置之前尾结点的下一个几点为当前节点
                pred.next = node;
                return node;
            }
        }
        // 1.当前队列是空队列  tail == null
        // 2.CAS竞争入队失败..会来到这里..
        enq(node);
        return node;
    }
-----------------------
	private void setHeadAndPropagate(Node node, int propagate) {
        // 拿到头节点
        Node h = head; // Record old head for check below
        // 讲当前节点设置为头结点
        setHead(node);
        // 如果 propagate=1 说明state 是0
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
                (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                // 唤醒 然后唤醒线程又会在 doAcquireSharedInterruptibly这个方法的循环里面进来
                doReleaseShared();
        }
    }
```

#### 唤醒逻辑

```java
private void doReleaseShared() {
        for (; ; ) {
            // 拿到阻塞队列头结点
            Node h = head;
            // 如果不为空并且不是尾结点 说明队列里面有元素
            // 如果h==tail 说明countdown完成了 然后 wait线程刚刚初始化完头结点 自己还没自选入队
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 如果是signal状态 就可以开始执行唤醒了
                if (ws == Node.SIGNAL) {
                    // 修改失败 就再次自旋
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    // 唤醒逻辑  跟AQS一样
                    unparkSuccessor(h);
                } else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                      // 如果ws 等于0  修改传播状态失败 继续循环
                    continue;                // loop on failed CAS
            }
            // 如果不等于head
            // 说明 唤醒成功 继续自循环
            if (h == head)                   // loop if head changed
                break;
        }
    }
```



------

## Condition条件应用 CyclicBarrier

### 基本结构图

```
利用 condition队列 加锁凑齐指定数量线程 并唤醒他们执行业务
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/Concurrent/CyclicBarrier.png!print" style="zoom: 67%;" />

### 简单案例

```java
public static void main(String[] args) {
        //第一步：定义玩家，定义5个
        String[] heros = {"安琪拉","亚瑟","马超","张飞", "刘备","安琪拉1","亚瑟1","马超1","张飞1", "刘备1"};

        //第二步：创建固定线程数量的线程池，线程数量为5
        ExecutorService service = Executors.newFixedThreadPool(5);

        //第三步：创建barrier，parties 设置为5
        CyclicBarrier barrier = new CyclicBarrier(5, new Runnable() {
            @Override
            public void run() {
                System.out.println("加载完对局");
            }
        });

        //第四步：通过for循环开启5任务，模拟开始游戏，传递给每个任务 英雄名称和barrier
        for(int i = 0; i < 10; i++) {
            service.execute(new Player(heros[i], barrier));
        }

        service.shutdown();
    }


    static class Player implements Runnable {
        private String hero;
        private CyclicBarrier barrier;

        public Player(String hero, CyclicBarrier barrier) {
            this.hero = hero;
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                //每个玩家加载进度不一样，这里使用随机数来模拟！
                TimeUnit.SECONDS.sleep(new Random().nextInt(10));
                System.out.println(hero + "：加载进度100%，等待其他玩家加载完成中...");
                barrier.await();
                System.out.println(hero + "：发现所有英雄加载完成，开始战斗吧！");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
```

### 源码分析

#### await().dowait 栅栏

```java
private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 获取锁对象 因为CyclicBarrier 依赖的是 condition组件
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 获取当前 版本 代
            final Generation g = generation;
            // 如果当前 代 被打破 直接抛异常,中断之类的会触发
            if (g.broken)
                throw new BrokenBarrierException();

            // 如果中断 则抛异常 并更改代被打破
            if (Thread.interrupted()) {
                //1.设置当前代的状态为broken状态  2.唤醒在trip 条件队列内的线程
                breakBarrier();
                throw new InterruptedException();
            }

            //假设 parties 给的是 5，那么index对应的值为 4,3,2,1,0
            int index = --count;
            //条件成立：说明当前线程是最后一个到达barrier的线程
            if (index == 0) {  // tripped
                //标记：true表示 最后一个线程 执行cmd时未抛异常。  false，表示最后一个线程执行cmd时抛出异常了.
                boolean ranAction = false;
                try {
                    // 是否有凑齐的执行任务
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    // 创建新的一代
                    nextGeneration();
                    return 0;
                } finally {
                    // 如果执行 凑齐任务出现异常 则把代打破并唤醒其他当代线程
                    if (!ranAction)
                        // 重置代 并打破当前代
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 自循环
            for (;;) {
                try {
                    // 如果有超时等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    // 如果超时 或 node 被外部中断 会进入
                    // 如果是当前代 并且没被打破 将重置并打破
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //执行到else有几种情况？
                        //1.代发生了变化，这个时候就不需要抛出中断异常了，因为 代已经更新了，这里唤醒后就走正常逻辑了..只不过设置下 中断标记。
                        //2.代没有发生变化，但是代被打破了，此时也不用返回中断异常，执行到下面的时候会抛出  brokenBarrier异常。也记录下中断标记位。
                        Thread.currentThread().interrupt();
                    }
                }
                // 代被更改 也会抛异常
                if (g.broken)
                    throw new BrokenBarrierException();

                //条件成立：说明当前线程挂起期间，最后一个线程到位了，然后触发了开启新的一代的逻辑，在创建新一代时已经唤醒之前代的线程了
                //返回当前线程的index。
                if (g != generation)
                    return index;
                // 超时异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```



------

## 信号量Semaphore复用工具

### 基本结构图

```
Semaphore与countdown比较明显的区别是Semaphore有释放的功能，并不是一次性使用完
他们都是基于AQS来设计的都是在sync抽象方法改变的逻辑
Semaphore典型的案例就是连接池之类的设计
```

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/Concurrent/semaphore.png!print" style="zoom: 50%;" />

### 简单案例

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/Concurrent/Semaphore 案例分析.png!print" style="zoom: 50%;" />

```java
public static void main(String[] args) throws InterruptedException {
        final Semaphore semaphore = new Semaphore(2, true);

        Thread tA = new Thread(() ->{
            try {
                semaphore.acquire();
                System.out.println("线程A获取通行证成功");
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
            }finally {
                semaphore.release();
            }
        });
        tA.start();
        //确保线程A已经执行
        TimeUnit.MILLISECONDS.sleep(200);

        Thread tB = new Thread(() ->{
            try {
                semaphore.acquire(2);
                System.out.println("线程B获取通行证成功");
            } catch (InterruptedException e) {
            }finally {
                semaphore.release(2);
            }
        });
        tB.start();
        //确保线程B已经执行
        TimeUnit.MILLISECONDS.sleep(200);

        Thread tC = new Thread(() ->{
            try {
                semaphore.acquire();
                System.out.println("线程C获取通行证成功");
            } catch (InterruptedException e) {
            }finally {
                semaphore.release();
            }
        });
        tC.start();
    }
```

### 源码分析

```
Semaphore 实现是默认是非公平拿信号量的逻辑
非公平与公平的逻辑在于 尝试获取锁时 会有hasQueuedPredecessors 
判断当前 AQS 阻塞队列内 是否有等待者线程，如果有直接返回-1，表示当前aquire操作的线程需要进入到队列等待..
```

#### acquire获取信号量

```java
// 前面调用法法跟countdown差不多  主要区别在这个方法
		/**
         * 尝试获取通行证，获取成功返回 >= 0的值;
         * 获取失败 返回 < 0 值
         */
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                //判断当前 AQS 阻塞队列内 是否有等待者线程，如果有直接返回-1，表示当前aquire操作的线程需要进入到队列等待..
                if (hasQueuedPredecessors())
                    return -1;
                //执行到这里，有哪几种情况？
                //1.调用aquire时 AQS阻塞队列内没有其它等待者
                //2.当前节点 在阻塞队列内是headNext节点

                //获取state ，state这里表示 通行证
                int available = getState();
                //remaining 表示当前线程 获取通行证完成之后，semaphore还剩余数量
                int remaining = available - acquires;

                //条件一：remaining < 0 成立，说明线程获取通行证失败..
                //条件二：前置条件，remaning >= 0, CAS更新state 成功，说明线程获取通行证成功，CAS失败，则自旋。
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

#### doAcquireSharedInterruptibly 阻塞逻辑

```java
private void doAcquireSharedInterruptibly(int arg)
                throws InterruptedException {
            //将调用Semaphore.aquire方法的线程 包装成node加入到 AQS的阻塞队列当中。
            final Node node = addWaiter(Node.SHARED);
            boolean failed = true;
            try {
                for (;;) {
                    //获取当前线程节点的前驱节点
                    final Node p = node.predecessor();
                    //条件成立，说明当前线程对应的节点 为 head.next节点
                    if (p == head) {
                        //head.next节点就有权利获取 共享锁了..
                        int r = tryAcquireShared(arg);


                        //站在Semaphore角度：r 表示还剩余的通行证数量
                        if (r >= 0) {
                            setHeadAndPropagate(node, r);
                            p.next = null; // help GC
                            failed = false;
                            return;
                        }
                    }
             //shouldParkAfterFailedAcquire  会给当前线程找一个好爸爸，最终给爸爸节点设置状态为 signal（-1），返回true
                    //parkAndCheckInterrupt 挂起当前节点对应的线程...
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

#### release释放信号量逻辑

```java
		/**
         *  countdown里面写的是-1
         *  semaphore 是增加指定数量
         * @param releases
         * @return
         */
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                // 增加
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
// 成功返回true 会调用 doReleaseShared唤醒逻辑 跟countdown一样
```




