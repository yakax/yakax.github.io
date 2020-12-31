# JVM初始化顺序


> 类的静态成员 --> 类的实例成员 --> 类的构造方法
> 说到这里，有一个很好的问题。比如一个类中静态变量是一个该类的实例，那么此时类的初始化顺序是什么样的呢？

#### 昨天在群里看到这个问题，很有趣，今天来分析一波
```java
public class test {
    public static test  t1=new test();
    {
        System.out.println("执行1");
    }
    static
    {
        System.out.println("执行2");
    }
    public static void main(String[] args) {
        test t2=new test();
    }
}
```
<!--more-->
首先我们得理解的执行顺序
1. 由于test类是JVM启动类，所以首先被加载，也就是执行加载、连接、初始化这些流程。
2. 进入初始化阶段，该阶段会对static域进行初始化。
3. 同为static按顺序执行所以这时只初始化了t1
4. <font color=red>由于这时test类已经处于初始化阶段了，static无需再次初始化，否则会导致static域多次初始化情况，所以暂时忽略static域初始化操作，对非static域进行初始化操作</font>这时打印了第一句话
5. 然后这时再执行t2的操作由于t1已经加载，所以加载静态块然后在加载构造块。

最后执行结果
```
执行1
执行2
执行1
```

如果理解了上面：来看一哈Alibaba的14年的一道校招附加题
```
public class Alibaba {

    public static int k = 0;
    public static Alibaba t1 = new Alibaba("t1");
    public static Alibaba t2 = new Alibaba("t2");
    public static int i = print("i");
    public static int n = 99;
    private int a = 0;
    public int j = print("j");
    {
        print("构造块");
    }
    static {
        print("静态块");
    }

    public Alibaba(String str) {
        System.out.println((++k) + ":" + str + "   i=" + i + "    n=" + n);
        ++i;
        ++n;
    }

    public static int print(String str) {
        System.out.println((++k) + ":" + str + "   i=" + i + "    n=" + n);
        ++n;
        return ++i;
    }

    public static void main(String args[]) {
        Alibaba t = new Alibaba("init");
    }
}
```
1. 由于Alibaba是 JVM 的启动类，属于主动调用，所以会依此进行 loading(加载)、linking(连接)、initialization(初始化)三个过程。
2. 经过 loading与 linking 阶段后，所有的属性都有了默认值，然后进入最后的 initialization 阶段。
3. 在 initialization 阶段，先对 static 属性赋值，然后在非 static 的。k 第一个显式赋值为 0 。
4. 接下来是t1属性，由于这时Alibaba这个类已经处于initialization 阶段，static 变量无需再次初始化了，所以忽略 static 属性的赋值，只对非 static 的属性进行赋值，所有有了开始的：
```
// 这时调用的是第9行的代码进入print方法
// 再执行构造块
// 执行构造方法
1:j   i=0    n=0
2:构造块   i=1    n=1
3:t1   i=2    n=2
```
5. 接着对t2进行赋值，过程与t1相同
```
// 这时调用的是第9行的代码进入print方法
// 再执行构造块
// 执行构造方法
4:j   i=3    n=3
5:构造块   i=4    n=4
6:t2   i=5    n=5
```
6. 之后到了 static 的 i ：
```
// 这时调用的是第9行的代码进入print方法
7:i   i=6    n=6
```
7. 执行static 的 n 赋值
8. 到现在为止，所有的static的成员变量已经赋值完成，接下来就到了 static 代码块
```
// 调用静态块
8:静态块   i=7    n=99
```
9. 至此，所有的 static 部分赋值完毕，接下来是非 static 的 j
```
// 调用print
9:j   i=8    n=100
```
10. 所有属性都赋值完毕，然后是构造块
```
// 调用构造块{}
10:构造块   i=9    n=101
```
11. 执行构造函数
```
// 最后构造函数
11:init   i=10    n=102
```
Alibaba这个类的初始化过程就算完成了。这里面比较容易出错的是认为会再次初始化 static 变量或代码块。而实际上是没必要，否则会出现多次初始化的情况。
最后执行结果
```
1:j   i=0    n=0
2:构造块   i=1    n=1
3:t1   i=2    n=2
4:j   i=3    n=3
5:构造块   i=4    n=4
6:t2   i=5    n=5
7:i   i=6    n=6
8:静态块   i=7    n=99
9:j   i=8    n=100
10:构造块   i=9    n=101
11:init   i=10    n=102
```








