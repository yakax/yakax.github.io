# 抽象工厂模式


> 因为抽象工厂模式在spring中用的比较多，例如spring的心脏BeanFactory就是利用抽象工厂模式来管理bean的。最主要的就是把具体功能抽象出来

#### 上代码
先建立产品的接口方法类
```
/**
 * 定义一个牛奶的接口
 */
public interface Milk {
    String getName();
}
```
产品实现功能
```
/**
 * 伊利产品
 */
public class Erie implements Milk {
    @Override
    public String getName() {
        return "伊利";
    }
}
/**
 * 蒙牛产品
 */
public class Mengniu implements Milk {
    @Override
    public String getName() {
        return "蒙牛";
    }
}
```
<!--more-->
创建抽象工厂类
```
/**
 * 抽象工厂生产那些产品
 */
public abstract class AbstractFactory {
    abstract Milk getMengniu();
    abstract Milk getErie();
}
```
创建具体产品类
```
/**
 * 具体怎么生产
 */
public class MilkFactory extends AbstractFactory {
    @Override
    Milk getMengniu() {
        return new Mengniu();
    }
    @Override
    Milk getErie() {
        return new Erie();
    }
}
```
对外的调用类
```
 MilkFactory milkFactory = new MilkFactory();
 // 当增加产品时，调用方根本不需要改东西，只管有没有具体方法
 milkFactory.getSanlu().getName();
```

#### 优点
- **降低耦合** 抽象工厂模式将具体产品的创建延迟到具体工厂的子类中，这样将对象的创建封装起来，可以减少客户端与具体产品类之间的依赖，从而使系统耦合度低，这样更有利于后期的维护和扩展；
- **更符合开-闭原则**新增一种产品类时，只需要增加相应的具体产品类和**相应**的工厂子类即可；而简单工厂模式需要修改工厂类的判断逻辑
- **符合单一职责原则**每个具体工厂类只负责创建对应的产品

#### 缺点
抽象工厂模式很难支持**新种类**产品的变化。 
这是因为抽象工厂接口中已经确定了可以被创建的产品集合，如果需要添加新产品，此时就必须去修改抽象工厂的接口，这样就涉及到抽象工厂类的以及所有子类的改变，这样也就违背了“开发——封闭”原则

#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/7.png" />
