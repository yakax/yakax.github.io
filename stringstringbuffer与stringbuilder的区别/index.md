# String、StringBuffer、StringBuilder的区别

**String 类型和 StringBuffer 类型的主要性能区别其实在于 String 是不可变的对象**

#### 简要的说
1. String 字符串常量
2. StringBuffer 字符串变量（线程安全）
3. StringBuilder 字符串变量（非线程安全）
4. String产生新的对象来操作 
5. Stringbuffer类型对自身操作

#### 论效率来说
> StringBuilder -> StringBuffer -> String


#### 区别
> 在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String 对象，所以经常改变内容的字符串最好不要用 String ，因为每次生成对象都会对系统性能产生影响，特别当内存中无引用对象多了以后,性能可想而知。

<!--more-->
#### StringBuffer与StringBuilder 的区别
> StringBuilder 不保证线程同步 方法跟StringBuffer 一样


#### 总结
> 如果要操作少量的数据，用String ；单线程操作大量数据，用StringBuilder ；多线程操作大量数据，用StringBuffer。

