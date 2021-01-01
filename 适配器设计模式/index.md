# 适配器设计模式


#### 应用场景
1. 已经存在的类，它的方法和需求不匹配（方法结果相同或相似）的情况
2. 适配器模式不是软件设计阶段考虑的设计模式，是随着软件维护，由于不同产品、不同厂家造成功能类似而接口不相同情况下的解决方案

<!--more-->
#### 代码
方法类
```
public interface Target {
    void methed();
}
```
准备适配的类（这里可以是抽象类也可以是普通类）
```
class ReadyAdapter {
    void methed() {
        log.info("适配的东西！");
    }
}
```
适配类：(如果有抽象类就继承来做适配)
```
public class Adapter implements Target {

    private ReadyAdapter readyAdapter;
    Adapter(ReadyAdapter readyAdapter) {
        this.readyAdapter = readyAdapter;
    }
    @Override
    public void methed() {
        readyAdapter.methed();
        log.info("可以在这里做适配操作");
    }
}
```
测试类
```
public static void main(String[] args) {
        Target target = new Adapter(new ReadyAdapter());
        target.methed();
    }
输出------------------------------------
ReadyAdapter - 适配的东西！
Adapter - 可以在这里做适配操作
```
#### 总结
1. 上面写这个属于对象适配器代码实现：这里写的是单个对象的适配，当要适配多个对象时我们可以把多个对象就继承一个类，然后在适配的时候通过InstanceOf来判断做适配。
2. 将目标类和适配者类解耦，通过引入一个适配器类来重用现有的适配者类，而无须修改原有代码。

#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/2.png"/>

