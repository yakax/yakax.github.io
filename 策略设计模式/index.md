# 策略设计模式


#### 应用场景
一个系统需要动态地在几种算法中选择一种的情况

#### 优点
1. 易于扩展, 增加一个新的策略只需要添加一个具体的策略类即可,基本不需要改变原有的代码,<font color=red>符合开闭原则;</font>
2. 避免使用多重条件选择语句(if else),充分体现面向对象设计思想。

<!--more-->
#### 缺点
<font color=red>客户端必须知道所有的策略类，并自行决定使用哪一个策略类</font>

#### 代码示例
Strategy接口 定义策略方法
```
public interface Strategy {
    /**
     * 计算方法
     *
     * @param a
     * @param b
     * @return
     */
    int calculate(int a, int b);
}
```
具体策略方法 实现策略接口，提供具体算法
```
public class AddCalculate implements Strategy{
    @Override
    public int calculate(int a, int b) {
        return a+b;
    }
}
```
```
public class SubtractCalculate implements Strategy {
    @Override
    public int calculate(int a, int b) {
        return a - b;
    }
}
```
Context类，持有具体策略类的实例，负责调用具体算法
```
class Context {
    private Strategy strategy;
    Context(Strategy strategy) {
        this.strategy = strategy;
    }
    int calculate(int a, int b) {
        return strategy.calculate(a, b);
    }
}
```
测试类
```
public class Test {
    public static void main(String[] args) {
        Strategy addCalculate = new AddCalculate();
        Context context = new Context(addCalculate);
        log.info(String.valueOf(context.calculate(3, 2)));
        
        Strategy subtractCalculate = new SubtractCalculate();
        context = new Context(subtractCalculate);
        log.info(String.valueOf(context.calculate(3, 2)));
    }
}
输出---------------------------
5
1
```
**这里的context类可以改成枚举类更方便**
```
enum ContextEnum {
    // 加法
    ADD(new AddCalculate()),
    // 减法
    SUBTRACT(new SubtractCalculate());
    private Strategy strategy;
    ContextEnum(Strategy strategy) {
        this.strategy = strategy;
    }
    Strategy getStrategy() {
        return this.strategy;
    }
}
```
测试
```
public class Test {
    public static void main(String[] args) {
        log.info(String.valueOf(ContextEnum.ADD.getStrategy().calculate(3, 2)));
        log.info(String.valueOf(ContextEnum.SUBTRACT.getStrategy().calculate(3, 2)));
    }
}
输出-------------------
5
1
```

#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/8.png"  />





