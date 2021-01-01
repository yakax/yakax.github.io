# 数据结构学习-动态数组




> 学习数据结构 -动态数组 模仿 ArrayList



<!--more-->

```java
public class ArrayList<E> {
    /**
     * 大小
     */
    private int size;

    /**
     * 原始数组
     */
    private Object[] elementData;
    /**
     * 空数组
     */
    private static final Object[] EMPTY_DATA = {};
    /**
     * 默认数组长度
     */
    private static final int DEFAULT_CAPACITY = 10;


    public ArrayList(int initialCapacity) {
        // 大于0 设置为用户大小  等于0 给空数组  否则设置错误
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_DATA;
        } else {
            throw new IllegalArgumentException("长度设置错误" + initialCapacity);
        }
    }

    public ArrayList() {
        // 没设置给个空数组
        this.elementData = EMPTY_DATA;
    }

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 添加数据
     *
     * @param e 数据
     * @return 添加成功
     */
    public boolean add(Object e) {
        // 确保我的数组长度是当前实际大小+1
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }

    /**
     * 添加方法 带下标位置
     *
     * @param index 下标
     * @param e     数据
     * @return 添加成功
     */
    public boolean add(int index, Object e) {
        // 校验位置
        rangeCheckForAdd(index);
        // 确保数组长度我能放进去
        ensureCapacityInternal(size + 1);
        //添加指定位置并移动指定位置后数据  系统方法 复制数组很有趣  可以实现自己复制给自己
        // 比如 0 1 2 3 4 5   我插入下标为3 的数据 变成 0 1 2 e 3 4 5 意思就是  把源数组 下标为3起始位置 开始 copy 到目标数组index+1 起始位置后的数据
        // copy长度size-index
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);
        elementData[index] = e;
        size++;
        return true;
    }

    /**
     * 获取指定下标数据
     *
     * @param index 下标
     * @return 数据
     */
    public E get(int index) {
        // 验证下标是否在正常范围
        rangeCheck(index);
        return (E) elementData[index];
    }

    /**
     * 设置 指定下标数据
     *
     * @param index 下标
     * @param e     修改数据
     */
    public E set(int index, int e) {
        // 验证下标是否在正常范围
        rangeCheck(index);
        // 返回旧值
        E oldElementData = (E) elementData[index];
        elementData[index] = e;
        return oldElementData;
    }

    /**
     * 删除指定下标数据
     *
     * @param index 下标
     * @return 旧数据
     */
    public E remove(int index) {
        // 检测下标是否越界
        rangeCheck(index);
        //返回旧值
        E oldElementData = (E) elementData[index];
        // 上面index能过检测 最大的大小就是size -1 就是数组最后一个值的下标
        int move = size - index - 1;
        if (move > 0) {
            // 说明需要移动下标 不是最后一位   比如 我有 1,2,3,4 数组 我要修改 数组为2下标的数据 我就要移动4 移动到 3 的位置 长度为 size-2-1
            System.arraycopy(elementData, index + 1, elementData, index, move);
        }
        // 移动过后最后一位需要清空 不然数据不会消失
        elementData[--size] = null;
        return oldElementData;
    }

    /**
     * 返回数据指定下标
     *
     * @param e 数据
     * @return 下标
     */
    public int indexOf(Object e) {
        if (e != null) {
            for (int i = 0; i < size; i++) {
                if (e.equals(elementData[i])) {
                    return i;
                }
            }
        } else {
            return -1;
        }
        return -1;
    }


    @Override
    public String toString() {
        StringJoiner stringJoiner = new StringJoiner(", ", "[", "]");
        for (Object element : elementData) {
            if (element != null) {
                stringJoiner.add(String.valueOf(element));
            }
        }
        return stringJoiner.toString();
    }

    /**
     * 下标长度位置是否合法判断 添加
     *
     * @param index 下标
     */
    private void rangeCheckForAdd(int index) {
        // 只能添加下边小于等于 当前list长度的 可以添加等于size下标位置的数据
        if (index > size || index < 0) {
            throw new IndexOutOfBoundsException("index 越界(添加时)" + index);
        }
    }

    /**
     * 验证下标是否越界 获取
     *
     * @param index 下标
     */
    private void rangeCheck(int index) {
        // 获取仅存在的下标
        if (index >= size) {
            throw new IndexOutOfBoundsException("下标越界异常" + index);
        }
    }

    /**
     * 确保容量有多大
     *
     * @param minLength 最小长度
     */
    private void ensureCapacityInternal(int minLength) {
        // 判断 大小 给予 默认大小
        if (elementData == EMPTY_DATA) {
            //表示是空数组给个默认长度
            minLength = DEFAULT_CAPACITY;
        }
        // 如果 数组长度 已经小于我们需要的最小长度 就扩容
        if (minLength > elementData.length) {
            // 旧长度
            int oldLength = elementData.length;
            // 新长度等于旧长度 的 1.5倍   旧长度+旧长度*0.5
            int newLength = oldLength + (oldLength >> 1);
            Object[] copy = new Object[newLength];
            // 把旧数组 赋值给新数组
            System.arraycopy(elementData, 0, copy, 0,
                    oldLength);
            log.warn("扩容原始长度{},扩容后长度{}", oldLength, newLength);
            elementData = copy;
        }
    }
```











