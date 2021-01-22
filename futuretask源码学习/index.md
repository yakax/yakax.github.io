# FutureTask源码学习



#### **基本属性**

```java
  	/* Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state; //线程状态
    private static final int NEW          = 0; // 进来未执行的状态
    private static final int COMPLETING   = 1; // 表示正在结束，临时状态，用于结束前cas操作;异常和有结果前都会有
    private static final int NORMAL       = 2; // 正常结束
    private static final int EXCEPTIONAL  = 3; // 异常结束
    private static final int CANCELLED    = 4; // 任务被取消
    private static final int INTERRUPTING = 5; // 中断中 也是临时状态
    private static final int INTERRUPTED  = 6; // 已中断

	// 运行任务
    private Callable<V> callable; 
    // 整个任务周期中 存放值的对象，会有异常或者正常的值
    private Object outcome; 
    // 正在执行任务的线程 
    private volatile Thread runner;
    // get任务阻塞的线程 会存在这个队列里面
    private volatile WaitNode waiters;
```

<!--more-->
#### 构造方法 

```java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        //callable就是程序员自己实现的业务类
        this.callable = callable;
        //设置当前任务状态为 NEW
        this.state = NEW;       // ensure visibility of callable
    }
    
public FutureTask(Runnable runnable, V result) {
        // 利用适配器模式 把runnable 换成Callable
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```

#### 状态相关方法

```java
// 是否中断   
public boolean isCancelled() {
    return state >= CANCELLED;
}
// 只要是不等于new 都属于线程已经开始执行的标记
public boolean isDone() {
    return state != NEW;
}
```

##### 取消任务

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 如果状态等于new 并且 修改状态成功 说明可以取消  否则返回false 取消是失败
    // mayInterruptIfRunning 是否会发生中断情况 说白了就是以中断抛异常结束，
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {	
                    Thread t = runner;
                    if (t != null)
                        // 加中断标记，具体看线程是否响应中断，捕获异常 自己处理
                        t.interrupt();
                } finally { // final state
                    //设置任务状态为 中断完成。
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 使阻塞队列的线程全部唤醒退出
            finishCompletion();
        }
        return true;
    }
```



#### **执行任务方法**

```java
 public void run() {
     // 判断线程状态 是否已经执行过或 cas失败 说明runnerOffset 已经被别的线程修改了
     // runnerOffset 时futuretask的局部变量
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            // 判断任务是否存在 状态是否是新状态 
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 执行任务
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    // 抛异常
                    result = null;
                    ran = false;
                    // 设置异常值
                    setException(ex);
                }
                if (ran)
                    // 设置返回值
                    set(result);
            }
        } finally {
           //设置当前线程引用为空
            runner = null;
            int s = state;
              // 如果正在中断中
            if (s >= INTERRUPTING)
                 // 直到中断结束 或者改变成其他状态
                handlePossibleCancellationInterrupt(s);
        }
    }
```

##### **设置返回值**

```java
protected void setException(Throwable t) {
        //使用CAS方式设置当前任务状态为 完成中..
        //失败的话 外部线程等不及了，直接在set执行CAS之前 将  task取消了。  很小概率事件。
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //引用的是 callable 向上层抛出来的异常。
            outcome = t;
            //将当前任务的状态 修改为 EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            // 唤醒获取数据的节点
            finishCompletion();
        }
    }
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            //将结果赋值给 outcome之后，马上会将当前任务状态修改为 NORMAL 正常结束状态。
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); 
            finishCompletion();
        }
    }
```

##### 唤醒所有节点  注意 done();方法

```java
private void finishCompletion() {
        //q指向waiters 链表的头结点。
        for (WaitNode q; (q = waiters) != null;) {
   		 //使用cas设置 waiters 为 null 为了保证只有一个线程在循环这个唤醒操作 因为外部方法cancel也会触发唤醒节点操作
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                //自循环 唤醒所有获取数据的线程
                for (;;) {
                    //获取当前node节点封装的 thread
                    Thread t = q.thread;
                    //条件成立：说明当前线程不为null
                    if (t != null) {

                        q.thread = null;//help GC
                        //唤醒当前节点对应的线程
                        LockSupport.unpark(t);
                    }
                    //next 当前节点的下一个节点 继续唤醒
                    WaitNode next = q.next;
                    // 当下一个几点为空时 说明是最后节点咯
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }

                break;
            }
        }
    // 这个方法 就是我们执行完task任务的回调方法 一般我们都会重写逻辑在里面 
        done();

        //将callable 设置为null helpGC
        callable = null;      
    }
```

#### get **方法阻塞队列**

```java
// 可以带有超时时间
public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state
           // 只有小于COMPLETING 正在结束 状态的线程才会进行阻塞  直到唤醒线程 如果唤醒后状态还是没变 就超时
        if (s <= COMPLETING &&
                (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
    	// 上面执行完 就能返回值
        return report(s);
    }
```

##### **report 返回值**

```java
private V report(int s) throws ExecutionException {
        //正常情况下，outcome 保存的是callable运行结束的结果
        //非正常，保存的是 callable 抛出的异常。
        Object x = outcome;
        //条件成立：当前任务状态正常结束
        if (s == NORMAL)
            //直接返回callable运算结果
            return (V)x;
        //被取消状态
        if (s >= CANCELLED)
            throw new CancellationException();
        //执行到这，说明callable接口实现中，是有bug的...
        throw new ExecutionException((Throwable)x);
    }
```



##### **awaitDone 阻塞分析**

```java
private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
        // 超时时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        //引用当前线程 封装成 WaitNode 对象
        WaitNode q = null;
        //表示当前线程 waitNode对象 有没有 入队/压栈
        boolean queued = false;
        //自旋
        for (;;) {
            //条件成立：说明当前线程唤醒 是被其它线程使用中断这种方式喊醒的。interrupted()
            //返回true 后会将 Thread的中断标记重置回false.
            if (Thread.interrupted()) {
                //当前线程node出队
                removeWaiter(q);
                //抛出中断异常
                throw new InterruptedException();
            }

            //假设当前线程是被其它线程 使用unpark(thread) 唤醒的话。会正常自旋，走下面逻辑。
            int s = state;
            //条件成立：说明当前任务 已经有结果了.. 可能有结果 可能有异常
            if (s > COMPLETING) {
                 // 这里正常情况下都是外面自循环已经创建好了节点才会出现
                //条件成立：说明已经为当前线程创建过node了，此时需要将 node.thread = null helpGC
                if (q != null)
                    q.thread = null;
                //直接返回当前状态.
                return s;
            }
            //条件成立：说明当前任务接近完成状态...这里让当前线程再释放cpu ，进行下一次抢占cpu。
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //条件成立：第一次自旋，当前线程还未创建 WaitNode 对象，此时为当前线程创建 WaitNode对象
            else if (q == null)
                q = new WaitNode();
            //条件成立：第二次自旋，当前线程已经创建 WaitNode对象了，但是node对象还未入队
            else if (!queued){
                //当前线程node节点 next 指向 原 队列的头节点   waiters 一直指向队列的头！
                q.next = waiters;
                //cas方式设置waiters引用指向 当前线程node， 成功的话 queued == true 否则，可能其它线程先你一步入队了。
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset, waiters, q);
            }
            //第三次自旋，会到这里。
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                // 阻塞 等待唤醒或中断
                LockSupport.parkNanos(this, nanos);
            }
            else
                //当前get操作的线程就会被park了。  线程状态会变为 WAITING状态，相当于休眠了..
                //除非有其它线程将你唤醒  或者 将当前线程 中断。
                LockSupport.park(this);
        }
    }
```


