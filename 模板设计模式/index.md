# 模板设计模式


#### 应用场景
定义一个模板结构，将具体内容延迟到子类去实现,<font color=red>在不改变模板结构的前提下在子类中重新定义模板中的内容。</font>
比如JDBC；

#### 优点
- 提高代码复用性
  将相同部分的代码放在抽象的父类中，而将不同的代码放入不同的子类中
- 实现了反向控制 
  通过一个父类调用其子类的操作，通过对子类的具体实现扩展不同的行为，实现了反向控制 & 符合“开闭原则”

#### 缺点
引入了抽象类，每一个不同的实现都需要一个子类来实现，导致类的个数增加，从而增加了系统实现的复杂度。
<!--more-->
#### 代码
一个做家常菜的案例
创建一个模板：
```
abstract class AbstractComputer {
    final void cookProcess() {
        //第一步：倒油
        this.pourOil();
        //第二步：热油
        this.hotOil();
        //第三步：倒蔬菜
        this.pourVegetables();
        //第四步：倒调味料
        this.pourSauce();
        //第五步：翻炒
        this.fry();
    }
    private void pourOil() {
        System.out.println("开始倒油");
    }
    private void hotOil() {
        System.out.println("开始热油咯");
    }
    /**
     * 具体倒什么蔬菜
     */
    abstract void pourVegetables();
    /**
     * 具体放什么调料
     */
    abstract void pourSauce();
    private void fry() {
        System.out.println("炒熟");
    }
}
```
具体实现
```
/**
 * 炒白菜
 */
class FriedCabbage extends AbstractComputer {
    @Override
    void pourVegetables() {
        System.out.println("把白菜倒下去");

    }
    @Override
    void pourSauce() {
        System.out.println("放干辣椒");
    }
}
```
```
/**
 * 炖鸡汤
 */
class StewedChickenSoup extends AbstractComputer {
    @Override
    void pourVegetables() {
        System.out.println("放老母鸡下去");
    }
    @Override
    void pourSauce() {
        System.out.println("放枸杞");
    }
}
```
测试类
```
public class Test {
    public static void main(String[] args) {
        AbstractComputer friedCabbage=new FriedCabbage();
        friedCabbage.cookProcess();
        AbstractComputer stewedChickenSoup=new StewedChickenSoup();
        stewedChickenSoup.cookProcess();
    }
}
输出-------------------
开始倒油
开始热油咯
把白菜倒下去
放干辣椒
炒熟
开始倒油
开始热油咯
放老母鸡下去
放枸杞
炒熟
```

#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/4.png" />



