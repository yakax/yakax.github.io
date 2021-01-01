# 装饰设计模式


#### 应用场景
1. 用于扩展一个类的功能或给一个类添加附加职责
2. 动态的给一个对象添加功能，这些功能可以再动态的撤销
3. 在spring mvc中 HttpHeadResponseDecorator类就是使用的装饰者
4. 在mybatis中(org.apache.ibatis.cache.Cache)里面就有许多关于缓存的装饰者类(后续我会在mybatis源码分析里面看)。
5. 在jdk中io相关的类(比如InputStream)都是使用了装饰设计模式。

<!--more-->
#### 代码
创建一个方法基类
```
public abstract class BaseBattercake {
    /**
     * 我要买什么
     *
     * @return
     */
    protected abstract String Msg();

    /**
     * 多少钱
     *
     * @return
     */
    protected abstract int price();
}
```
创建默认信息
```
public class Battercake extends BaseBattercake {
    @Override
    protected String Msg() {
        return "煎饼";
    }
    @Override
    protected int price() {
        return 5;
    }
}
```
创建基础装饰
```
public abstract class BaseBattercakeDecotator extends BaseBattercake {

    private BaseBattercake baseBattercake;

    BaseBattercakeDecotator(BaseBattercake baseBattercake) {
        this.baseBattercake = baseBattercake;
    }
    @Override
    protected String Msg() {
        return this.baseBattercake.Msg();
    }
    @Override
    protected int price() {
        return this.baseBattercake.price();
    }
}
```
加蛋装饰
```
public class EggDecorator extends BaseBattercakeDecotator {
    EggDecorator(BaseBattercake baseBattercake) {
        super(baseBattercake);
    }
    @Override
    protected String Msg() {
        return super.Msg() + "加一个蛋";
    }
    @Override
    protected int price() {
        return super.price() + 1;
    }
}
```
加香肠装饰
```
public class SausageDecorator extends BaseBattercakeDecotator {
     SausageDecorator(BaseBattercake baseBattercake) {
        super(baseBattercake);
    }
    @Override
    protected String Msg() {
        return super.Msg() + "加一根香肠";
    }
    @Override
    protected int price() {
        return super.price() + 2;
    }
}
```
测试
```
public class Test {
    public static void main(String[] args) {
        // 买一个煎饼
        BaseBattercake baseBattercake = new Battercake();
        // 加鸡蛋
        baseBattercake = new EggDecorator(baseBattercake);
        log.info(baseBattercake.Msg() + baseBattercake.price());
        // 加香肠
        baseBattercake = new SausageDecorator(baseBattercake);
        log.info(baseBattercake.Msg() + baseBattercake.price());
    }
}
输出-----------------------
煎饼加一个蛋6
煎饼加一个蛋加一根香肠8
// 通过得到父类的信息来做成动态的扩展父类功能，灵活增加业务需求 这也就不需要多继承
```
#### 优点
1. 是继承的有力补充，比继承灵活，不改变原有对象 的情况下动态地给一个对象扩展功能，即插即用。
2. 使用不同装饰类以及这些装饰类的排列组合，可以实 现不同效果。
3. 者完全遵守开闭原则。

#### 缺点
1. 多的代码，更多的类，增加程序复杂性。
2. 装饰时，多层装饰时会更复杂。

#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/3.png"/>
