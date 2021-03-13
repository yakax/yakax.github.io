# HashMap 常见问题Java8


### HashMap并发修改异常,或者循环时删除异常

`ConcurrentModificationException`基本都是modCount修改后与原modCount不相等而暴露的异常。很多集合都会有这个问题

编写者抛这个异常的是想 快速失败，暴露异常。让开发者自己解决
<!--more-->
当我们在for循环map 时，会编译成迭代器的方式 去运行。而删除我们调用的是hashmap自身的remove方法。

```java
        HashMap<Integer, String> host = new HashMap();
        host.put(1, "2");
        host.put(2, "2");
        host.put(3, "2");
        Iterator var2 = host.keySet().iterator();

        while(var2.hasNext()) {
            Integer integer = (Integer)var2.next();
            if (integer.equals(1)) {
                host.remove(integer);
            }
        }
```

var2.next() 这行代码执行时 会判断count而报错。 解决方法就是 用 **迭代器的remove** 方法,会在remove时重新给modCount赋值

```java
	public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            removeNode(p.hash, p.key, null, false, false);
            expectedModCount = modCount;
        }
```

这个不是线程安全的，所以并发还是用ConcurrentHashMap



### HashMap 底层结构

1.7:数组+链表  (由于是链表长度长了。查询效率不高。所以在设计初衷1.7的hash算法更复杂。数据也更散列)

1.8:数组+链表+红黑树（JDK8中即使用了单向链表，也使用了双向链表，双向链表主要是为了红黑树相关链表操作方便，应该在插入，扩容，链表转红黑树，红黑树转链表的过程中都要操作链表）

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/sets/QQ%E6%88%AA%E5%9B%BE20200905112823.png" style="zoom:50%;" />

### HashMap 转红黑条件

只有**当链表中的元素个数大于8(此时 node有9个)，并且数组的长度大于等于64时才会将链表转为红黑树**。

```java
// putVal 片段
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        // 这里放入第九个元素 只有在binCount在第七次并且当前元素下一个元素为null时
        p.next = newNode(hash, key, value, null); 
        if (binCount >= TREEIFY_THRESHOLD - 1) // TREEIFY_THRESHOLD=8
            treeifyBin(tab, hash);
        break;
    }
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}

// 这是在treeifyBin方法 树化前的判断 MIN_TREEIFY_CAPACITY=64
 if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();// 利用扩容来缩减链表长度

```

### HashMap 扩容机制

当前容量大于等于阈值 或 在树化之前，当前数组的长度小于64，链表长度大于等于8 也会发生扩容。

默认 长度是16 阀值是16*0.75 
每次扩容时阀值=`旧阀值 << 1` 容量=`旧容量<<1` 也就是两倍

```java
// 部分代码
 else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
```

1. 在扩容时通过对新的数组进行与运算新下标  这是下标对应node节点只有一个时
2. 如果node 节点是链表，都会把这个链表拆分成两个高低node节点的链表`loHead和hiHead`
3. 拆分后在放入数组、这样只需要改变链表头结点的位置就完成转移。
4. 如果是红黑树 也是类似链表 先拆分 在移动只改表头结点的位置。**有可能拆分后树太小会转链表**



### HashMap  put过程

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    // 进来前已经把key的hash算好了
        Node<K,V>[] tab; Node<K,V> p; int n, i;
    //判断当前hashmap对象是否为空，为空就初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    // 通过与运算 算出下标 并判断这个下标是否存在值 不存在就赋值新值 存在就进入下面else
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 当通过与运算算出下标对应小标存在值 进入
            Node<K,V> e; K k;
            // 如果key相等 说明就是一个值 则覆盖
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
            // 如果节点是树类型 那么直接向红黑树插入值 在putTreeVal的循环中第一次循环就赋值到树中。并且有覆盖的值会返回原有值
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
               // 进入这里说明还是链表通过binCount计数从头遍历到尾结点,新kv会封装成node插入到链表尾部 条件满足就树化。
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // ！注意 这里的赋值引用将会影响上面的e相关的赋值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // 如果设置了onlyIfAbsent为false 说明要替换原有的值 会把旧值的vale赋值给e 
                // 旧值为空 说明 没得覆盖的值将赋值并返回
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e); // 空方法 LinkedHashMap的方法
                return oldValue;
            }
        }
    // 修改次数增加 
        ++modCount;
    // hashMap的元素个数size加1 如果size大于扩容的阈值，则进行扩容
        if (++size > threshold)
            resize(); 
        afterNodeInsertion(evict);// 空方法 LinkedHashMap的方法
        return null;
    }
```

### HashMap 树化 简单分析

```java
 final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
     // 是否满足树化条件
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // 循环节点 把单向链表转为双向链表
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                // 把双向链表 转树
                hd.treeify(tab);
        }
    }
// treeify 具体逻辑 不分析代码了，有点乱
1. 将头结点作为root节点，然后依次将next节点插入到根节点，转变红黑树。
2. 再插入时候key比较
   1. 如果key实现了comparable接口，通过实现方式比较
   2. 否则比较key的hashCode
   3. 否则比较key的class.getName
   4. 否则比较key的System.identityHashCode比较
3. 最后树化后，取出root节点（TreeNode），放到entry下标位置
```

### HashMap remove值后退化链表的条件

**有两个地方会判断并退化成链表的条件**

1. remove时退化

   ```java
   // 在 removeTreeNode的方法中
   // 在红黑树的root节点为空 或者root的右节点、root的左节点、root左节点的左节点为空时 说明树都比较小了
   if (root == null || (movable && (root.right == null || (rl = root.left) == null|| rl.left == null))) {
     	tab[index] = first.untreeify(map);  // too small
       return;
   }
   ```

2. 在扩容时 low、high 两个TreeNode 长度小于6时 会退化为链表。

### HashMap 为什么选择红黑树

1. 因为AVL树插入节点或者删除节点，整体的性能是不如红黑树的。AVL每个左右节点的高度是不能大于1的。所以维持这种结构比较消耗性能。主要还是左旋右旋改变与维护AVL高度问题，红黑树比他好的就是只需改变节点颜色就可以了。

2. 二分查找树，他的左右节点不平衡，一开始就固定了root，那么极端的情况下会成为链表结构。

3. 链表长度越长，那么他的插入和查询效率都很低。

4. 而红黑树他的整体查找，增删节点的效率都是比较高的。

   

### HashMap 的hash优化

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
举个例子
key="ykx"
key的hashCode是=119718  这是个10进制数
原始hashCode对应二进制11101001110100110==00000000000000011101001110100110
然后我再向右位移16位后的二进制==00000000000000000000000000000001 
异或运算
    0000000000000001 1101001110100110
    0000000000000000 0000000000000001
    0000000000000001 1101001110100111==119719 这个就是最终的hash值
```
#### 优化点在哪

```java
HashMap在get方法去获取值时是这样的
    hash(key)&(n-1) 进行与运算而找到数组位置
    0000000000000001 1101001110100111 上面优化后的hash值 hashMap默认数组长度16
    0000000000000000 0000000000001111 &
    0000000000000000 0000000000000111 =7
    这就是最终的位置下标  只要数组长度是2的n次方  你会发现：119719%16与119719&(16-1)的值时相同的
```

<font color=blue>因为与运算的性能会比取模效率高。</font>

#### 总结

其实这里最重要的几个点就是在**算hash值时高是16位与低16进行了异或运算**（因为很有可能有两个不同的值hash高16位不相同低16位相同）。这样就导致HashMap寻址运算时**低16位已经包含了高16位与低16 的特征**，因为在寻址的时候大多数都是低16位在运算，因为数组长度-1的数字大小一般情况都比较小。所以在get寻址时基本都是低16在运算，尽量避免hash冲突，寻址时用与运算代替取模运算也是比较大的优化，只要当HashMap的数组长度是2的n次方那么我们算出来的hash值取模这个长度等于与运算这个长度-1 的值。
