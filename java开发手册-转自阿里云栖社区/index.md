# Java开发手册-(转自阿里云栖社区)




[Java开发手册下载链接](https://github.com/alibaba/p3c)

<font color=red>1-0.9=0.1是天经地义的，但在计算机的世界里，0.1恰恰是无法精确表示的一个小数，只有2的幂次倍小数才能够精确表示，如：0.5、0.25、0.125等。由于0.1是近似表达，在各种情形中的计算存在数位的取舍精度不一样，所以1-0.9未必等于0.9-0.8，所以浮点数之间的等值判断，基本数据类型不能用==来比较，包装数据类型不能用equals来判断。包装类型运算也是调用的浮点类型的计算</font>
```
public class FloatPrimitiveTest {
    public static void main(String[] args) {
        float a = 1.0f - 0.9f;
        float b = 0.9f - 0.8f;
        if (a == b) {
            System.out.println("true");
        } else {
            System.out.println("false");
        }
    }
}
输出------------false


public class FloatWrapperTest {
    public static void main(String[] args) {
        Float a = Float.valueOf(1.0f - 0.9f);
        Float b = Float.valueOf(0.9f - 0.8f);
        if (a.equals(b)) {
            System.out.println("true");
        } else {
            System.out.println("false");
        }
    }
}
输出--------false
正例--------------------------------
(1) 指定一个误差范围，两个浮点数的差值在此范围之内，则认为是相等的。
 float a = 1.0f - 0.9f;
 float b = 0.9f - 0.8f;
 float diff = 1e-6f;
 if (Math.abs(a - b) < diff) {
     System.out.println("true");
 }
(2) 使用 BigDecimal 来定义值，再进行浮点数的运算操作。
 BigDecimal a = new BigDecimal("1.0");
 BigDecimal b = new BigDecimal("0.9");
 BigDecimal c = new BigDecimal("0.8");
 BigDecimal x = a.subtract(b);
 BigDecimal y = b.subtract(c);
 if (x.equals(y)) {
     System.out.println("true");
 }

```

<!--more-->

**当 switch 括号内的变量类型为 String 并且此变量为外部参数时，必须先进行 null判断。**
```
public class SwitchTest {
    public static void main(String[] args) {
        String param = null;
        switch (param) {
            case "null":
                System.out.println("null");
                break;
            default:
                System.out.println("default");
        }
    }
}
抛出NPE异常
```

**为了防止精度损失，禁止使用构造方法 BigDecimal(double)的方式把 double 值转化为 BigDecimal 对象。**
BigDecimal(double)存在精度损失风险，在精确计算或值比较的场景中可能会导致业务逻辑异常。
如：BigDecimal g = new BigDecimal(0.1f); 实际的存储值为：0.10000000149
优先推荐入参为 String 的构造方法，或使用 BigDecimal 的 valueOf 方法，此方法内部其实执行了Double 的 toString，而 Double 的 toString 按 double 的实际能表达的精度对尾数进行了截断
```
推荐使用的方式
BigDecimal recommend1 = new BigDecimal("0.1");
BigDecimal recommend2 = BigDecimal.valueOf(0.1);
```

**在使用阻塞等待获取锁的方式中，必须在 try 代码块之外，并且在加锁方法与 try 代码块之间没有任何可能抛出异常的方法调用，避免加锁成功后，在 finally 中无法解锁。**
<font color=red>如果 lock 方法在 try 代码块之内，可能由于其它方法抛出异常，导致在 finally代码块中，unlock 对未加锁的对象解锁，它会调用 AQS 的 tryRelease 方法（取决于具体实现类），抛出 IllegalMonitorStateException 异常。在 Lock 对象的 lock方法实现中可能抛出 unchecked 异常。而在使用尝试机制来获取锁的方式中，比如 tryLock()，在进入业务代码块之前，必须先判断当前线程是否持有锁。</font>
```
public class LockTest {
    private final static Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        try {
            lock.tryLock();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
所以这个unlock不会抛出异常
正确加锁写法
Lock lock = new XxxLock();
        // ...
    lock.lock();
    try {
         doSomething();
         doOthers();
    } finally {
         lock.unlock();
   }
```






