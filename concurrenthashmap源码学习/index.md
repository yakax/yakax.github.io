# ConcurrentHashMap源码学习



### 整体结构图

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/HashMap/chm.png" style="zoom:33%;" />

<!--more-->
### put 插入

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    	// k v 不能为空
        if (key == null || value == null) throw new NullPointerException();
    	//  让高位和地位进行异或运算 充分利用所有位数
        int hash = spread(key.hashCode());
    	// 0表示当前可以放值进去，2表示可能是红黑
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // ----1 这个判断初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            //----2 算出下标 并且下标下面无头结点
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 失败会再次自旋 走其他条件
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // ----3 能到这个条件说明数组存在 并且 对应下标头结点存在 判断当前节点是否正在扩容
            else if ((fh = f.hash) == MOVED)
                // 协助扩容
                tab = helpTransfer(tab, f);
            else { // ----4 当前节点存在头结点 并且可能是链表 也可能是红黑树
                V oldVal = null;
                // 加锁 头结点 头结点赋值在第二个条件赋值的
                synchronized (f) {
                    // 再次判断是否是我想要操作的下标头节点
                    if (tabAt(tab, i) == f) {
                        // fh 默认是0 大于0说明是正常链表 因为treebin头结点hash是 -2
                        if (fh >= 0) {
                            // 会影响到addcount()
                            binCount = 1;
                            // 循环链表 每次加1
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 判断key 是否相同 
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // onlyIfAbsent 是否覆盖原有的值 true 不覆盖
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 找到尾结点 
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果是红黑树 加入红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            // 会影响到addcount()
                            binCount = 2; 
                            // 判断 节点与红黑树节点是否冲突、不冲突会返回null值
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val; 
                                if (!onlyIfAbsent) //覆盖
                                    p.val = value;
                            }
                        }
                    }
                }
                //binCount默认0 如果不为0说明是链表或者红黑   
                if (binCount != 0) {
                    // 如果链表长度数量大于等于8了转红黑 !但是里面也判断了整个数组长度64的限制
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
    	// 增加count数亮 
        addCount(1L, binCount);
        return null;
    }
```

### addCount 算数

fullAddCount 这个方法可以直接看longAdder的**longAccumulate**这个方法 一模一样

```java
private final void addCount(long x, int check) {
        CounterCell[] cs; long b, s;
    	//当不存在counterCells  写basecount不成功会进入条件 当counterCells存在不会尝试写basecount
        if ((cs = counterCells) != null ||
            !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell c; long v; int m;
            boolean uncontended = true;
            // 判断counterCells长度和是否存在，存在就尝试写countcell 失败会进入fullAddCount 
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            // 说明是remove之类的方法进来的：可以直接返回
            if (check <= 1)
                return;
             //获取当前散列表元素个数，这是一个期望值 不会是最终值
            s = sumCount();
        }
    // 大于等于0说明一定是put之类的方法进来的 
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // s目前是总个数 
            //s >= (long)(sc = sizeCtl)
            // true-> 1.当前sizeCtl为一个负数 表示正在扩容中..可以进入帮助扩容
            //  true->  2.当前sizeCtl是一个正数，s大于扩容阈值 ..可以进入帮助扩容
            //另外条件都是正常成立  
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 获取扩容唯一标识 
                int rs = resizeStamp(n);
                // 说明正在扩容 可以协扩容
                if (sc < 0) {
                    // 判断扩容戳是否是本次 或者transferIndex小于0说明扩容结束
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // 让SIZECTL 低位+1
                    if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                        // 协助扩容 持有一个链表
                        transfer(tab, nt);
                }
                // 第一次 扩容 会把SIZECTL 改为负数，并且sc的低16位会表示扩容存在的线程 以便于上面if的判断
                else if (U.compareAndSetInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

#### 扩容固定标识说明

```java
    // 比如 14长度数组超过阀值 这一批的调用的扩容参数返回 都是1000000000011100
	static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
	}
```

### transfer 扩容代码

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //根据cpu核心数算出每个线程的扩容数量区间  stride 默认为 16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range

        // 说明第一次进入扩容方法 做准备工作
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                //创建了一个比扩容之前大一倍的table
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            //新的扩容后的Node[]
            nextTable = nextTab;
            //标记需要迁移的长度 这是原来的node长度
            transferIndex = n;
        }

        //表示新Node[]的长度
        int nextn = nextTab.length;
        // 新的node 设置为fwd 节点
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        //推进标记 用于迁移的时候自循环判断
        boolean advance = true;
        //完成标记
        boolean finishing = false; // to ensure sweep before committing nextTab
        // 表示执行到的区间与上限
        int i = 0, bound = 0;
        for (;;) {
            //头结点 与头结点hash
            Node<K,V> f; int fh;

            // 维护每个线程步长任务区间
            while (advance) {
                //分配任务的开始下标 nextBound代表长度
                int nextIndex, nextBound;

                //CASE1:
                //成立：表示当前线程的任务尚未完成，还有相应的区间的桶位要处理，--i 就让当前线程处理下一个 桶位.
                if (--i >= bound || finishing)
                    advance = false;
                //CASE2: 说明已经分配完毕
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                //CASE3:分配任务区间
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {

                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }

            //CASE1：未分配到任务
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 表示扩容完成 使新node 赋值给table
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    //sizeCtl 恢复 等于扩容阀值
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }

                //未分配到任务 依然要修改SIZECTL -1 代表自己退出了
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 如果分配成功 判断自己是否是最后一个线程 不是就正常退出 是的话就让赋值finishing 在上面条件退出
                    // 这里其实会检查 会执行到上面while循环做检查 如果有遗漏的还会执行 知道执行到上面条件
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }

            //CASE2:
            //条件成立：说明当前桶位未存放数据，只需要将此处设置为fwd节点即可。
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //CASE3:
            //条件成立：如果被迁移过 就再次循环
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            //CASE4:
            //前置条件：当前桶位有数据，而且node节点 不是 fwd节点，说明这些数据需要迁移。
            else {
                //sync 加锁当前桶位的头结点
                synchronized (f) {
                    //防止在你加锁头对象之前，当前桶位的头对象被其它写线程修改过，导致你目前加锁对象错误...
                    if (tabAt(tab, i) == f) {
                        //ln 表示低位链表引用
                        //hn 表示高位链表引用
                        Node<K,V> ln, hn;

                        //条件成立：表示当前桶位是链表桶位  TREEBIN 是-2
                        if (fh >= 0) {
                            //lastRun
                            //可以获取出 当前链表 末尾连续高位不变的 node
                            // 先算出头结点hash位置 这时这个n是原数组长度
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 这个循环可以拿到一个连续的节点 优化迁移
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }

                            //条件成立：说明lastRun引用的链表为 低位链表，那么就让 ln 指向 低位链表
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            //否则，说明lastRun引用的链表为 高位链表，就让 hn 指向 高位链表
                            else {
                                hn = lastRun;
                                ln = null;
                            }


                            // 遍历到lastRun 节点 说明已经到最后了 因为后面是指向一个地方的
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }

                            // 迁移节点
                            //get 的时候是通过(n - 1) & h 会发现 用原数组 长度 就可以判断出来了
                            // 新的长度就是原长度左移了一位 所以只需要把这一位加进去就ok 
                            // 重点关注图片中hash后面几位 变的其实就是加了旧数组长度
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        //条件成立：表示当前桶位是 红黑树 代理结点TreeBin
                        else if (f instanceof TreeBin) {
                            //转换头结点为 treeBin引用 t
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            //低位双向链表 lo 指向低位链表的头  loTail 指向低位链表的尾巴
                            TreeNode<K,V> lo = null, loTail = null;
                            //高位双向链表 lo 指向高位链表的头  loTail 指向高位链表的尾巴
                            TreeNode<K,V> hi = null, hiTail = null;


                            //lc 表示低位链表元素数量
                            //hc 表示高位链表元素数量
                            int lc = 0, hc = 0;

                            //迭代TreeBin中的双向链表，从头结点 至 尾节点
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                // h 表示循环处理当前元素的 hash
                                int h = e.hash;
                                //使用当前节点 构建出来的 新的 TreeNode
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);

                                //条件成立：表示当前循环节点 属于低位链 节点
                                if ((h & n) == 0) {
                                    //条件成立：说明当前低位链表 还没有数据
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    //说明 低位链表已经有数据了，此时当前元素 追加到 低位链表的末尾就行了
                                    else
                                        loTail.next = p;
                                    //将低位链表尾指针指向 p 节点
                                    loTail = p;
                                    ++lc;
                                }
                                //当前节点 属于 高位链 节点
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }

                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
// hash 这个原因主要是扩容后的容量实际上是原来容量的二进制位高位多了一个1 
// (还有个原因就是get的时候是扩容长度-1 意思就是全是1比如 31==11111 比如15=1111)相当于我们在获取的时候只是高位多了一个1
// 在与运算情况下 比如 10010与10000 会变成10000  
// 所以在hash已经确定的情况下 只需要确定高位是否是1 就能确定之后的hash位置
// 比如 你对应的那个高位是0 那么 肯定是原来的位置，如果是1那么就不是原来的位置
```

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/HashMap/%E6%89%A9%E5%AE%B9.png)

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/HashMap/%E6%B5%81%E7%A8%8B%E6%89%A9%E5%AE%B9.png)

### helpTransfer协助扩容代码

```java
 final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
            Node<K,V>[] nextTab; int sc;
            // 这些条件基本都恒成立
            if (tab != null && (f instanceof ForwardingNode) &&
                (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
                // 拿到表示戳
                int rs = resizeStamp(tab.length);
                // 等知道扩容完成 才返回 
                // sc=sizectl 小于0的情况下说明还可以协助 这一点是在第一次参与扩容时算的负值
                while (nextTab == nextTable && table == tab &&
                       (sc = sizeCtl) < 0) {
                    // 判断扩容戳是否是本次 或者transferIndex小于0说明扩容结束
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || transferIndex <= 0)
                        break;
                    // 如果还需要协助扩容 加入扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                        transfer(tab, nextTab);
                        break;
                    }
                }
                return nextTab;
            }
            return table;
    }
```

扩容的最细粒度是通过当前机器的核心算出来了。单核一个线程最小粒度是**16个长度**  协助不了的线程会在上面代码一直while。因为sc的值代表了能允许多少线程帮助扩容。

**扩容时会产生两个链 一个是要移动的链，一个是不移动的，因为扩容后有些数据已经不再原有位置。**



### get() 获取元素

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 算出此key 的hash 这里如果key为null 就会报空指针
    int h = spread(key.hashCode());
    // 判断 node是否存在并且 是否能定位到值
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断这个node头元素是否等于是否真的 等于我的key 
        // 因为就算你的下标相同 key也可能不相同 因为你的hash很长但是&的长度不一定很长 最终运算的只有后面几位
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                // 相同就返回
                return e.val;
        }
        // 只有一个两种情况 treebin节点或者fwd节点 
        else if (eh < 0)
            // 去查询值
            return (p = e.find(h, key)) != null ? p.val : null;
        // 如果是链表 就循环查询
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

#### ForwardingNode.find()

```java
Node<K,V> find(int h, Object k) {
    // loop to avoid arbitrarily deep recursion on forwarding nodes
    outer: for (Node<K,V>[] tab = nextTable;;) {
        Node<K,V> e; int n;
        // 数据不存在
        if (k == null || tab == null || (n = tab.length) == 0 ||
            (e = tabAt(tab, (n - 1) & h)) == null)
            return null;
        for (;;) {
            int eh; K ek;
            // 存在  并且key完全相同
            if ((eh = e.hash) == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
            // 红黑 或者fwd节点 
            if (eh < 0) {
                // 如果是fwd节点  就赋值 重新循环(高并发情况下 可能扩容后再次扩容)
                if (e instanceof ForwardingNode) {
                    tab = ((ForwardingNode<K,V>)e).nextTable;
                    continue outer;
                }
                // 调用红黑树查询
                else
                    return e.find(h, k);
            }
            // 赋值下一个循环元素-- 如果是最后节点还没找到就退出
            if ((e = e.next) == null)
                return null;
        }
    }
}
```



### remove() 删除元素

```java
// 要删除的key 删除对应的value 要替换的value 
// 意思就是如果传了value 和 cv 如果传值进来的cv与key对应的value 相同 就替换成传进来的value 
final V replaceNode(Object key, V value , Object  cv) {
    int hash = spread(key.hashCode());
    // 老规矩 赋值 自循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 不存在就退出
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        // 如果是在扩容的就协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 用于后面 判断是否删除值
            boolean validated = false;
            synchronized (f) {
                // 判断加锁对象是否已经改变
                if (tabAt(tab, i) == f) {
                     // 这是链表情况 fwd情况已经在加锁的时候规避了
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            // 判断key是否相同 
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                //然后判断是否需要替换值
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    // 如果不替换
                                    if (value != null)
                                        e.val = value;
                                   	// 改变链表情况 比如不是头结点情况下 让上一个结点指向当前节点的下一个节点完成链表链接
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        // 如果是头结点
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            // 再次赋值 循环
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        // 查询 是否存在
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                              // 是否替换
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                // 如果removeTreeNode 返回真 说明需要红黑转链表
                                else if (t.removeTreeNode(p))
                                    // 转链表
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            // 是否删除成功 
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        // 如果不是替换就减少元素个数
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

### removeTreeNode() 转链表代码

```java
            if (first == null) {
                root = null; // 如果头结点都为null 转链表 
                return true;
            }
			// root.right == null   // 右孩子为空 或
			// || root.left == null // 左孩子为空 或
			// || root.left.left == null  // 左孙子为空
            if ((r = root) == null || r.right == null || // too small
                (rl = r.left) == null || rl.left == null)
                return true;
```

这里有点没理解 网上答案也模糊 看面试问不问了，如果问了 直接问面试官。



### 红黑树基本说明

```
红黑树的头结点会设置hash 为-2 以代表这个节点下面是红黑树
红黑树可以允许多个读取，读取的时候会尝试修改 TreeBin的 lockState的值 增加4
红黑树写的时候就会尝试cas修改lockState 为1。如果是0就会成功，如果不为0，就会阻塞等待。 
锁住的都是头结点 并且修改的都是头结点的值,也就是TreeBin。
```



### 统计map 大小代码

通过两个值来存储元素个数

`baseCount` 与`CounterCell[]`数组  CounterCell里面有个value存储值 

1. 先通过cas操作basecount操作，如果存在冲突就
2. 然后通过随机函数下标修改contercell的值 如果失败
3. 进入fullAddCount 如果没初始化就初始化。 
4. 通过cas修改cellsBusy 标识加锁。修改CounterCell的值。

**总的来说就是通过cas修改`baseCount` 失败，再尝试修改CounterCell 数组里面的值，还失败就产生一个新的CounterCell 。**



### 退化情况

**在<font color=red>扩容时</font>小于6的时候退化 或者<font color=red>删除元素时</font>判断树太小的时候退化，也就是判断左右节点和父节点的左节点为空的情况。**



### 1.7与1.8版本结构对比

1.8 是数组 +单向链表+红黑  node链表  treebin头与TreeNode节点

1.7是数组+segment链表  每段数组是个segment 对象 里面存储是多个entry

主要优化还是**并发协助扩容**这里优化; 和查询性能优化

**元素个数** 1.7是遍历segment    1.8是遍历CounterCell数组+basecount 相对数量少

红黑树的查询在量大的情况下O(logN)肯定好于O(N)








