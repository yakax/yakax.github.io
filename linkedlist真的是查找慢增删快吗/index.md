# LinkedList真的是查找慢增删快吗

**以前别人面试我,这个问题的时候我一般都是回答：linkendlist增删改块，arraylist查找块。直到最近我看了掘金的一篇博文，才发现，实践出真知啊。**

#### 测试结果
> 分别在ArrayList和LinkedList的头部、尾部和中间三个位置插入与查找100000个元素所消耗的时间来进行对比测试，下面是测试结果

| List | 插入 | 查找 |
| :------:| :------: | :------: |
| ArrayList头部 | 2859ms | 7ms |
| ArrayList尾部 | 26ms | 12ms |
| ArrayList中间 | 848ms | 13ms |
| LinkedList头部 | 15ms | 11ms |
| LinkedList尾部 | 28ms | 11ms |
| LinkedList中间 | 15981ms | 34928ms |
<!--more-->
#### 测试结论
- ArrayList的查找性能绝对是一流的，无论查询的是哪个位置的元素
- ArrayList除了尾部插入的性能较好外（位置越靠后性能越好），其他位置性能就不如人意了
- LinkedList在头尾查找头尾性能都很棒，但是在中间位置进行操作的话，性能就差很远了，而且跟ArrayList完全不是一个量级的，<font color=red>并且Linkedlist并不是插入哪里性能都比Arraylist快，越靠中间，插入越慢。</font>

#### 源码分析
我们把Java中的ArrayList和LinkedList就是分别对顺序表和双向链表的一种实现：

- 顺序表：需要申请连续的内存空间保存元素，可以通过内存中的物理位置直接找到元素的逻辑位置。在顺序表中间插入or删除元素需要把该元素之后的所有元素向前or向后移动。
- 双向链表：不需要申请连续的内存空间保存元素，需要通过元素的头尾指针找到前继与后继元素（查找元素的时候需要从头or尾开始遍历整个链表，直到找到目标元素）。在双向链表中插入or删除元素不需要移动元素，只需要改变相关元素的头尾指针即可。

所以我们潜意识会认为：ArrayList查找快，增删慢。LinkedList查找慢，增删快；<font color=red>但实际上并不是这样的。</font>

##### ArrayList尾部插入
add(E e)方法
```
public boolean add(E e) {
        // 检查是否需要扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 直接在尾部添加元素
        elementData[size++] = e;
        return true;
    }
```
##### LinkedList尾部插入
LinkedList中定义了头尾节点
```
    /**
     * Pointer to first node.
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     */
    transient Node<E> last;
```
add(E e)方法，该方法中调用了linkLast(E e)方法
```
public boolean add(E e) {
        linkLast(e);
        return true;
    }
```
linkLast(E e)方法，可以看出，在尾部插入的时候，并不需要从头开始遍历整个链表，因为已经事先保存了尾结点，所以可以直接在尾结点后面插入元素
```
    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        // 先把原来的尾结点保存下来
        final Node<E> l = last;
        // 创建一个新的结点，其头结点指向last
        final Node<E> newNode = new Node<>(l, e, null);
        // 尾结点置为newNode
        last = newNode;
        if (l == null)
            first = newNode;
        else
            // 修改原先的尾结点的尾结点，使其指向新的尾结点
            l.next = newNode;
        size++;
        modCount++;
    }
```
对于尾部插入而言，ArrayList与LinkedList的性能几乎是一致的

##### ArrayList头部插入
add(int index, E element)方法，可以看到通过调用系统的数组复制方法来实现了元素的移动。所以，插入的位置越靠前，需要移动的元素就会越多
```
public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);  
        // 把原来数组中的index位置开始的元素全部复制到index+1开始的位置（其实就是index后面的元素向后移动一位）
        System.arraycopy(elementData,index,elementData, index + 1,size - index);
        // 插入元素
        elementData[index] = element;
        size++;
    }
```
##### LinkedList头部插入
add(int index, E element)方法，该方法先判断是否是在尾部插入，如果是调用linkLast()方法，否则调用linkBefore()，那么是否真的就是需要重头开始遍历呢？我们一起来看看
```
public void add(int index, E element) {
        checkPositionIndex(index);
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
```
linkBefore方法
这个函数的工作就只是负责把元素插入到相应的位置而已，关键的工作在node()方法中已经完成了
```
 void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```
node方法
```
Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
在头尾以外的位置插入元素当然得找出这个位置在哪里，这里面的node()方法就是关键所在，这个函数的作用就是根据索引查找元素，但是它会先判断index的位置，如果index比size的一半(size >> 1,右移运算，相当于除以2)要小，就从头开始遍历。否则，从尾部开始遍历。从而可以知道，对于LinkedList来说，操作的元素的位置越往中间靠拢，效率就越低。

#### ArrayList、LinkedList查找
- 这就没啥好说的了，对于ArrayList，无论什么位置，都是直接通过索引定位到元素，时间复杂度O(1)
- 而对于LinkedList查找，其核心方法就是上面所说的node()方法，所以头尾查找速度极快，越往中间靠拢效率越低

#### 总结
- 对于LinkedList来说，头部插入和尾部插入时间复杂度都是O(1)
- 但是对于ArrayList来说，头部的每一次插入都需要移动size-1个元素，效率可想而知
- 但是如果都是在最中间的位置插入的话，ArrayList速度比LinkedList的速度快将近10倍


