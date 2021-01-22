# Condition源码学习


总体Condition产生的图

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/MQ/QQ%E6%88%AA%E5%9B%BE20200917105341.png" style="zoom: 33%;" />

<!--more-->

1. **signal() 唤醒队列是从condition阻塞队列取第一个** ，不过是**放进AQS队列的队尾**, 并且入队后会判断是否需要挂起
2. condition队列node 进入AQS队列后 会判断上一个节点是否为head节点 然后尝试加锁 **否则依然会挂起**
3. condition队列的node状态是CONDITION=-2
4. **await是**会阻塞当前线程并且会完全释放锁，然后唤醒**AQS**头结点来获取锁。也就是阻塞在lock外面的线程。
5. condition队列存储的就是曾经获取到锁，但是await的节点。
6. **Condition队列的node节点直至被signal/signalAll后才会使得当前线程从等待队列中移至到同步队列中去，直到获得了lock后才会从await方法后面开始执行**
7. lock() 是可以new 多个条件队列的。意思是可以维护多个条件队列 然后根据条件进入 AQS

## 代码分析

#### await()

```java
public final void await() throws InterruptedException {
            // 判断是否中断 中断的线程 直接抛异常
            if (Thread.interrupted())
                throw new InterruptedException();

            // 将当前线程封装成node 添加进入单向条件队列
            Node node = addConditionWaiter();
            // 完全释放 aqs持有的锁
            int savedState = fullyRelease(node);

            // 是否有出现过中断信号
            // -1:在condition 出现过中断信号
            // 1:在aqs队列出现过中断信号
            int interruptMode = 0;

            // 是否在阻塞队列 如果不在阻塞队列 那么将线程挂起
            while (!isOnSyncQueue(node)) {
                // 挂起
                LockSupport.park(this);
//1.常规路径：外部线程获取到lock之后，调用了 signal()方法 转移条件队列的头节点到 阻塞队列， 当这个节点获取到锁后，会唤醒。
                //2.转移至阻塞队列后，发现阻塞队列中的前驱节点状态 是 取消状态，此时会唤醒当前节点
                //3.当前节点挂起期间，被外部线程使用中断唤醒..

                // 唤醒后 判断中断信号
                // 如果不等于0 那么说明没中断过 则退出   这里面condition队列中断的话依然会把node 转到 AQS队列
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //执行到这里，就说明 当前node已经迁移到 “阻塞队列”了 传入了你的加锁状态
            // 如果上一个节点是头结点 回去竞争锁，如果不是 会找到一个signal的节点作为node的上一个节点
            // parkAndCheckInterrupt 并且会返回 当前线程的中断状态
            // 如果是中断状态 并且不是THROW_IE 说明在condition 没有中断过
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                // 这里方法 看lock的AQS
                // 设置成 REINTERRUPT
                interruptMode = REINTERRUPT;
            //其实是node在条件队列内时 如果被外部线程 中断唤醒时，会加入到阻塞队列，但是并未设置nextWaiter = null。
            if (node.nextWaiter != null) // clean up if cancelled
                // 从头开始 向后遍历 把不是condition 状态的节点都剔除掉
                unlinkCancelledWaiters();
            // 如果不等于 0 说明被中断过
            if (interruptMode != 0)
                // 根据 THROW_IE 和 REINTERRUPT 中断状态返回信息
                // condition 中断会抛异常 AQS中断会做标记
                reportInterruptAfterWait(interruptMode);
        }
        /**
         *唤醒后 判断中断信号
         */
        private int checkInterruptWhileWaiting(Node node) {
            // 如果被中断过 就执行transferAfterCancelledWait(node)
            // condition情况中断返回 THROW_IE   -1
            // AQS返回 REINTERRUPT 1
            return Thread.interrupted() ? (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT)
                    : 0;
        }
```

##### 进入condition队列

```java
private Node addConditionWaiter() {
            // 拿到尾结点
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            // 如果不为空 并且还不是condition状态 那就只能是取消状态了
            if (t != null && t.waitStatus != Node.CONDITION) {
                // 清理取消状态节点  这里面会让lastWaiter 变成一个condition状态的节点
                unlinkCancelledWaiters();
                // 赋值局部变量
                t = lastWaiter;
            }
            // 封装当前node 为condition条件队列节点
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            // 如果这时 lastWaiter 为空 -- 说明条件队列没有node
            if (t == null)
                // node 赋值头结点
                firstWaiter = node;
            else
                // 否则lastWaiter的下一个节点附上node
                t.nextWaiter = node;
            // 最后 lastWaiter 改为node
            lastWaiter = node;
            return node;
        }
```

##### 完全释放锁 并且唤醒AQS头节点

```java
public final boolean release(int arg) {
        //尝试释放锁，tryRelease 返回true 表示当前线程已经完全释放锁
        //返回false，说明当前线程尚未完全释放锁.. 解锁需要成对出现
        if (tryRelease(arg)) {
            //头结点
            Node h = head;

            //条件一:成立，说明队列中的head节点已经初始化过了，ReentrantLock 在使用期间 发生过 多线程竞争了...
            // 只有在第二个线程进来时就会有head
            //条件二：条件成立，说明当前head后面一定插入过node节点。节点去唤醒
            // waitStatus 其他状态 后继节点在park前会改变前节点的值为-1 或者有其他状态
            if (h != null && h.waitStatus != 0)
                //唤醒后继节点..
                unparkSuccessor(h);
            return true;
        }

        return false;
    }
```

##### 判断是否在AQS队列 

```java
final boolean isOnSyncQueue(Node node) {
        // 因为node节点 如果在condition队列状态 肯定是condition
        //条件一：node.waitStatus == Node.CONDITION 条件成立，说明当前node一定是在
        //条件二：因为 signal方法时是先修改node的状态 然后 再调用enq的
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //  但是next不为空的话 说明一定在阻塞队列  因为enq的里面cas操作过后操作的next
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /**
         * 执行到这里，说明当前节点的状态为：node.prev != null 且 node.waitStatus ！= CONDITION
         * 那么只有可能是在aqs 队列
         * 从aqs尾结点向前查找里面的node 找到就返回
         */
        return findNodeFromTail(node);
    }
```

#### 唤醒 signal()

```java
	public final void signal() {
            // 判断是否是持有锁 未持有则抛出异常
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();

            // 第一个节点
            Node first = firstWaiter;
            // 如果不为空 则开始迁移
            if (first != null)
                doSignal(first);
        }
	private void doSignal(Node first) {
            do {
                // 判断当前节点 下个节点是否为空 如果为空 则开始把指针这些都指向空
                if ((firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
                // 直至迁移成功  如果失败first = firstWaiter 则将下一个节点赋值给first 继续循环
            } while (!transferForSignal(first) &&
                    (first = firstWaiter) != null);
        }
```

##### 迁移进入 AQS队列  这里面跟await唤醒后逻辑有关联

```java
final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        // 迁移前 先修改状态 如果修改失败则返回false
        // 比如 未持有锁就进condition队列的
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        // 自旋进入 AQS队列  会返回这个node 的前驱节点
        Node p = enq(node);
        //获取前驱节点状态
        int ws = p.waitStatus;

        // ws >0 说明前驱节点是取消状态  则需要唤醒当前节点
        // 当前驱node对应的线程 是 lockInterrupt入队的node时，是会响应中断的，外部线程给前驱线程中断信号之后，前驱node会将状态修改为 取消状态，并且执行 出队逻辑..

        // 前置条件(ws <= 0)， 基本只有等于0
        // compareAndSetWaitStatus(p, ws, Node.SIGNAL) 设置前驱节点是signal状态 以便于唤醒下一个节点

        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            // 唤醒当前线程  总之 前驱节点有问题 变成了取消状态  都会唤醒当前线程
            // !!这里 会配合线程唤醒后去 acquireQueued 尝试抢占锁时 会有个找正常节点的过程 shouldParkAfterFailedAcquire 这个方法
            // 并且 会在 parkAndCheckInterrupt 这个方法继续挂起
            LockSupport.unpark(node.thread);
        // 返回true 因为已经入AQS队列咯
        return true;
    }
```



------



#### 工具类BlockingQueue

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/aqs/QQ%E6%88%AA%E5%9B%BE20200922133220.png)

通过维护两个condition条件队列，实现唤醒发送数据队列和获取数据队列。Array是利用数组来存放定长的数据。

notFull 队列 存放生产数据阻塞的队列 由消费者消费数据来唤醒

notEmpty队列存放消费队列为空时阻塞的队列，由生产者生产数据后唤醒。

```
主要方法还是put() take() 方法 
新增add满了会抛异常  offer 会返回是否插入成功
获取poll 带超时时间在时间内没获取到返回null  peek会返回数据，但是不移除队列。
```

> 其实forkjoin就是把任务拆分成多个workqueue 在分线程对队列执行任务而已，而窃取 就是去遍历这些队列 未执行的任务。
