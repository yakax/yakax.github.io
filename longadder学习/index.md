# LongAdder学习


<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/Concurrent/longadder/longadder.png" style="zoom: 50%;" />
<!--more-->

**LongAdder 更适合做全局计数统计**。而**AtomicLong适合做自增**，因为他能累加并实时返回值，而LongAdder统计的值时近实时。

AtomicLong 由于一直通过**cas自旋修改值**，在高并发情况下会有很多线程处于自旋，而LongAdder 利用空间换时间的方式把要增加的值先放另一边。

## 分析

### cell

```java
 @sun.misc.Contended static final class Cell {
     // 由于数组对象volatile 修饰，将只对数组对象可见，所以在cell 里面增加volatile修饰value
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                // 返回对象的value在内存中的偏移量
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

`因为由于驻留在数组中的原子对象往往彼此相邻,如果一个对象出现缓存行失效，将会影响另外的对象的缓存刷新。`
@sun.misc.Contended注解 消除了伪共享，由于缓存行一般是64k 这个注解通过填充对象大小来让缓存行只存在一个cell对象。

**伪共享**指的是多个线程同时读写同一个缓存行的不同变量时导致的 `CPU缓存失效`。尽管这些变量之间没有任何关系，但由于在主内存中邻近，存在于同一个缓存行之中，它们的相互覆盖会导致频繁的缓存未命中，引发性能下降

### add()

```java
public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        // 判断cells是否等于null 如果不等于null  存在cells数组就会进入 
        // 如果前面条件不满足：会执行cas操作base  存在竞争会进入
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            // 用于标识是否存在竞争 false 标识有竞争
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                // 算下标 找个cell
                (a = as[getProbe() & m]) == null ||
                // cas 写cell
                !(uncontended = a.cas(v = a.value, v + x)))
                // cells没初始化 || 数组对象下标不存在 || 初始化了更改数组对象值失败 
                longAccumulate(x, null, uncontended);
        }
    }
```

### longAccumulate()

如果cells 存在 那么尝试写入数组对象值  

如果 不存在 就初始化

如果 初始化 加锁失败 就再次 写base。

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;//通过线程引用拿到一个hash值 主要用于找下标
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
    	// 扩容意向 true 可能扩容，false不会扩容
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // cells 存在 可以先尝试写到cells数组内
            if ((as = cells) != null && (n = as.length) > 0) {
                // 如果找到的下标为空 ，尝试增加一个cell对象到数组
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // 尝试获取标记锁 将cellsBusy改为1
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                // 这里再次判断了下标值 是否存在 以防其他线程已经初始化了,因为上层if判断没有判断这个
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;  //如果失败 再次循环          // Slot is now non-empty
                        }
                    }
                    // 扩容意向改为false
                    // 因为 你的(a = as[(n - 1) & h]) == null 判断为真 说明你是能写数据进去的不需要扩容。
                    collide = false;
                }
                // 在外面cas cells数组元素值 失败进来的时候是false 
                // 会执行到h = advanceProbe(h); 重置hash
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                
                // 重置 hash 过后 再来cas 数组元素一次  
                // !! 还有一点就是下面条件不会 break说明都会执行h = advanceProbe(h);都会来尝试cas
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                // 判断数组是否已经不相等了。说明已经扩容了
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                
                // 扩容意向 改为true
                else if (!collide)
                    collide = true;
                // 开始扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1]; // 两倍
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            
            // 初始化cells 
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    // 再次加判断 以免覆盖  类似双重检查 
                    // 因为上层判断是分3部分的 并且 赋值cells与cellsBusy也是分开的，难免会出现问题
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        // 初始化成功
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                // 成功退出
                if (init)
                    break;
            }
            // 都不成功上面加锁失败 再次尝试修改base
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```


