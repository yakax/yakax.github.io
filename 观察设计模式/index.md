# 观察设计模式


#### 应用场景
1. 观察者模式有时也叫做发布订阅模式。
2. 主要用于在关联行为之间建立一套触发机制的场景(例如一些提醒业务、MQ等等)
3. java.awt.Event 就是观察者模式的一种，只不过 Java 很少被用来写桌面程序。上面的比如点击事件等等都是通过发布订阅绑定来触发事件。
4. 在spring 中ContextLoaderListener是实现了ServletContextListener接口,这个接口也是继承的EventListener。这也是观察者模式的代表。

<!--more-->
java语言中，有一个接口Observer，以及它的实现类Observable，对观察者角色进行了实现。我们通过这个api写一个发布订阅
#### 代码
观察者
```
public class EyeOfGod implements Observer {

    private String name;

    @Override
    public void update(Observable o, Object arg) {
        // 被观察者信息(观察多个的话可以通过类型判断来最强转)
        ObserveAims observeAims = (ObserveAims) o;
        // 给被观察者发布的信息
        Question question = (Question)arg;
        System.out.println(this.name+"给"+observeAims.getName()+"发布了"+question.getMsg());
    }
}
```
被观察者
```
class ObserveAims extends Observable {
    private String name;

    void publishQuestion(Question question) {
        System.out.println(this.name+ "收到了一条" + question.getMsg() + "的消息。");
        setChanged();
        notifyObservers(question);
    }
}
```
消息信息类
```
class Question {
    private String msg;
}
```
测试
```
public class Test {
    public static void main(String[] args) {
        EyeOfGod eyeOfGod=new EyeOfGod();
        eyeOfGod.setName("老师");
        ObserveAims observeAims =new ObserveAims();
        observeAims.setName("小明");
        Question question=new Question();
        question.setMsg("写作业");
        eyeOfGod.update(observeAims,question);
        observeAims.publishQuestion(question);
    }
}
输出--------------------
老师给小明发布了写作业
小明收到了一条写作业的消息。
```

#### 优点
1. 观察者和被观察者之间建立了一个抽象的耦合。
2. 观察者模式支持广播通信。

#### 缺点
1. 观察者之间有过多的细节依赖、提高时间消耗及程序的复杂度。
2. 使用要得当，要避免循环调用。

#### 总结
观察者模式有很多写法，现在主流比较简单的写法是通过google的guava包的@Subscribe注解 
```
public class GuavaEvent {
    @Subscribe
    public void subscribe(String str){
       // 可以编写业务逻辑
        System.out.println("执行subscribe方法，传入的参数是：" + str);
    }
}

    public static void main(String[] args) {
        //消息总线
        EventBus eventBus = new EventBus();
        GuavaEvent guavaEvent = new GuavaEvent();
        eventBus.register(guavaEvent);
        eventBus.post("传入信息");
    }
输出------------------
执行subscribe方法，传入的参数是：传入信息
```
#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/5.png" />
