# ThreadPoolExecutor源码学习


#### 属性字段说明

```java
//高3位：表示当前线程池运行状态   除去高3位之后的低位：表示当前线程池中所拥有的线程数量
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //表示在ctl中，低COUNT_BITS位 是用于存放当前线程数量的位。
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //低COUNT_BITS位 所能表达的最大数值。 000 11111111111111111111 => 5亿多。
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    //111 000000000000000000  转换成整数，其实是一个负数
    private static final int RUNNING    = -1 << COUNT_BITS;
    //000 000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //001 000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;
    //010 000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;
    //011 000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    //获取当前线程池运行状态
    //~000 11111111111111111111 => 111 000000000000000000000
    //c == ctl = 111 000000000000000000111
    //111 000000000000000000111
    //111 000000000000000000000
    //111 000000000000000000000
    private static int runStateOf(int c)     { return c & ~CAPACITY; }

    //获取当前线程池线程数量
    //c == ctl = 111 000000000000000000111
    //111 000000000000000000111
    //000 111111111111111111111
    //000 000000000000000000111 => 7
    private static int workerCountOf(int c)  { return c & CAPACITY; }

    //用在重置当前线程池ctl值时  会用到
    //rs 表示线程池状态   wc 表示当前线程池中worker（线程）数量
    //111 000000000000000000
    //000 000000000000000111
    //111 000000000000000111
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

<!--more-->
#### 添加execute

```java
public void execute(Runnable command) {
        //非空判断..
        if (command == null)
            throw new NullPointerException();
        //获取ctl最新值赋值给c，ctl ：高3位 表示线程池状态，低位表示当前线程池线程数量。
        int c = ctl.get();
        //workerCountOf(c) 获取出当前线程数量
        //条件成立：表示当前线程数量小于核心线程数，此次提交任务，直接创建一个新的worker，对应线程池中多了一个新的线程。
        if (workerCountOf(c) < corePoolSize) {
            //addWorker 即为创建线程的过程，会创建worker对象，并且将command作为firstTask
            //core == true 表示采用核心线程数量限制  false表示采用 maximumPoolSize
            if (addWorker(command, true))
                //创建成功后，直接返回。addWorker方法里面会启动新创建的worker，将firstTask执行。
                return;

            //1.存在并发现象，execute方法是可能有多个线程同时调用的，当workerCountOf(c) < corePoolSize成立后，
            //其它线程可能也成立了，并且向线程池中创建了worker。这个时候线程池中的核心线程数已经达到，所以...
            //2.当前线程池状态发生改变了。 RUNNING SHUTDOWN STOP TIDYING　TERMINATION
            //当线程池状态是非RUNNING状态时，addWorker(firstTask!=null, true|false) 一定会失败。
            //SHUTDOWN 状态下，也有可能创建成功。前提 firstTask == null 而且当前 queue  不为空。特殊情况。
            c = ctl.get();
        }


        //执行到这里有几种情况？
        //1.当前线程数量已经达到corePoolSize
        //2.addWorker失败..

        //条件成立：说明当前线程池处于running状态，则尝试将 task 放入到workQueue中。
        if (isRunning(c) && workQueue.offer(command)) {
            //执行到这里，说明offer提交任务成功了..
            //再次获取ctl保存到recheck。
            int recheck = ctl.get();

    //条件一：! isRunning(recheck) 成立：说明你提交到队列之后，线程池状态被外部线程给修改 比如：shutdown()shutdownNow()
            //这种情况 需要把刚刚提交的任务删除掉。
            //条件二：remove(command) 有可能成功，也有可能失败
            //成功：提交之后，线程池中的线程还未消费（处理）
            //失败：提交之后，在shutdown() shutdownNow()之前，就被线程池中的线程 给处理。
            if (! isRunning(recheck) && remove(command))
                //提交之后线程池状态为 非running 且 任务出队成功，走个拒绝策略。
                reject(command);

            //有几种情况会到这里？
            //1.当前线程池是running状态(这个概率最大)
            //2.线程池状态是非running状态 但是remove提交的任务失败.

            //担心 当前线程池是running状态，但是线程池中的存活线程数量是0，这个时候，如果是0的话，会很尴尬，任务没线程去跑了,
            //这里其实是一个担保机制，保证线程池在running状态下，最起码得有一个线程在工作。
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }else if (!addWorker(command, false))
        //执行到这里，有几种情况？
        //1.offer失败
        //2.当前线程池是非running状态

        //1.offer失败，需要做什么？ 说明当前queue 满了！这个时候 如果当前线程数量尚未达到maximumPoolSize的话，会创建新的worker直接执行command
        //假设当前线程数量达到maximumPoolSize的话，这里也会失败，也走拒绝策略。
        //2.线程池状态为非running状态，这个时候因为 command != null addWorker 一定是返回false。
            reject(command); 
    }
```

#### 添加工作线程

从这里就可以看出线程是不区分核心不核心的，知识一个判断而已

成功：创建worker 成功 并且 启动成功

失败：线程数量判断、线程池状态、worker创建失败、worker启动失败

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
    // 自旋 并且有retry这个跳出
        for (;;) {
            // 获取ctl 这个ctl是能算出线程状态和线程数量
            int c = ctl.get();
            // 获取状态
            int rs = runStateOf(c);
            // 大于等于SHUTDOWN 说明不是运行状态
            // 第二个条件是当前线程是SHUTDOWN 但是还有任务没执行完 也不运行添加任务
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN && firstTask == null &&! workQueue.isEmpty()))
                return false;

            for (;;) {
                // 获取线程数
                int wc = workerCountOf(c);
                // 线程数判断 
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 到这里说明线程可以加入进来 cas 尝试修改ctl来标识线程进来了  成功就跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // cas 失败 再次获取ctl
                c = ctl.get();  // Re-read ctl
                // 判断状态是否变化，变了 再次循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

    // 准备开始添加进入worker 工作
    // worker运行标识
        boolean workerStarted = false;
    // worker 添加标识
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 加锁 来对线程池做操作。
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    // 获取最近状态
                    int rs = runStateOf(ctl.get());
                    
                    //条件1 判断是否是运行状态 
           //条件2 说明是shutdown状态 但是firsttask为空 跟上面添加线程的时候情况一样，可以运行任务，但是不让添加任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 线程是否存活的native方法 就是怕给线程池之前 线程就已经start了
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 添加到worker
                        workers.add(w);
                        int s = workers.size();
                        // 新高size判断
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 标识添加成功
                        workerAdded = true;
                    }
                } finally {
                    // 释放锁
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 启动  会调用worker里面那个run方法
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // 如果启动失败 
            if (! workerStarted)
                // 清理工作。主要是对worker 和 ctl删减
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

```java
private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        //持有线程池全局锁，因为操作的是线程池相关的东西。
        mainLock.lock();
        try {
            //条件成立：需要将worker在workers中清理出去。
            if (w != null)
                workers.remove(w);
            //将线程池计数恢复-1，前面+1过，这里因为失败，所以要-1，相当于归还令牌。
            decrementWorkerCount();
            tryTerminate();
        } finally {
            //释放线程池全局锁。
            mainLock.unlock();
        }
    }
```

runWorker       start会调用worker的run方法

```java
final void runWorker(Worker w) {
    // 获取线程信息
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
     	// 初始化worker state == 0 和 exclusiveOwnerThread ==null 
    // 最终会调用到 worker里面的tryRelease  不是aqs的
        w.unlock(); // allow interrupts
    // 退出通知
        boolean completedAbruptly = true;
        try {
            // 拿到任务 会阻塞
            while (task != null || (task = getTask()) != null) {
                // 加锁
                w.lock();
              
                // 线程池状态是大于stop 说明线程状态STOP/TIDYING/TERMINATION 需要中断
                // 都是判断线程池状态或者当提前线程状态 来给予线程标记位
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 钩子方法 我们可以实现 执行任务前操作
                    beforeExecute(wt, task);
                    // 异常
                    Throwable thrown = null;
                    try {
                        // 运行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 钩子 任务结束
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 清空task
                    task = null;
                    // 线程池执行任务计数
                    w.completedTasks++;
                    // 释放锁  如果任务执行失败 会退出getTask自循环
                    w.unlock();
                }
            }
            //正常退出
            completedAbruptly = false;
        } finally {
            // 异常退出
            processWorkerExit(w, completedAbruptly);
        }
    }
```

#### getTask拿到线程超时机制

```java
 private Runnable getTask() {
     // 是否超时标志  会根据线程数比较得出
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //是否有任务 线程池状态是否是运行状态 
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                // 死循环更改ctl-1
                decrementWorkerCount();
                return null;
            }
            // 获取线程池线程数量
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            // 是否是核心线程数 如果不是就需要具有闲置超时机制
          //条件成立：当前线程池中的线程数量是大于核心线程数的，此时让所有路过这里的线程，都是用poll 支持超时的方式去获取任务，
            //这样，就会可能有一部分线程获取不到任务，获取不到任务 返回Null，然后..runWorker会执行线程退出逻辑。
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 先判断是否大于线程池数量 线程数始终要存活1个
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                // cas 让ctl-1 成功
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 根据是否超时状态 来获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                // 说明超时 那么还是会循环
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

#### 线程运行退出

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    	//条件成立：代表当前w 这个worker是发生异常退出的，task任务执行过程中向上抛出异常了..
        //异常退出时，ctl计数，并没有-1 
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
	
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 将当前worker完成的task数量，汇总到线程池的completedTaskCount
            completedTaskCount += w.completedTasks;
            // 移除队列
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();
    // 保证线程池最低线程数量
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // 线程没了 但是还有任务 这时最下面顶一个线程进去
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 留一个线程保持线程池存活状态
            addWorker(null, false);
        }
    }
```

#### 线程池关闭

```java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        //获取线程池全局锁
        mainLock.lock();
        try {
            // 线程安全管理方面 我们少用
            checkShutdownAccess();
            //设置线程池状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断空闲线程
            interruptIdleWorkers();
            //空方法，子类可以扩展
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            //释放线程池全局锁
            mainLock.unlock();
        }
        tryTerminate();
    }
```

#### 设置线程池状态 (转化)

```java
final void tryTerminate() {
        //自旋
        for (;;) {
            //获取最新ctl值
            int c = ctl.get();
            //条件一：isRunning(c)  成立，直接返回就行，线程池很正常！
   //条件二：runStateAtLeast(c, TIDYING) 说明 已经有其它线程 在执行 TIDYING -> TERMINATED状态了,当前线程直接回去。
            //条件三：(runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())
            //SHUTDOWN特殊情况，如果是这种情况，直接回去。得等队列中的任务处理完毕后，再转化状态。
            if (isRunning(c) ||
                    runStateAtLeast(c, TIDYING) ||
                    (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;

            //条件成立：当前线程池中的线程数量 > 0
            if (workerCountOf(c) != 0) { // Eligible to terminate
               // 中断空闲线程 其实就是通过中断信号 唤醒阻塞的线程 
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            //获取线程池全局锁
            mainLock.lock();
            try {
                //设置线程池状态为TIDYING状态。
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //调用钩子方法
                        terminated();
                    } finally {
                        //设置线程池状态为TERMINATED状态。
                        ctl.set(ctlOf(TERMINATED, 0));
                        //唤醒调用 awaitTermination() 方法的线程。
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                //释放线程池全局锁。
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```


