# 单例设计模式


#### 单例的好处
单例模式的初衷就是为了使资源能够共享，只需要赋值或者初始化一次，大家都能够重复利用。
一个类Class只有一个实例存在。 使用Singleton的好处还在于可以节省内存，因为它限制了实例的个数，有利于Java垃圾回收；常见的单例有枚举常量类、IOC容器、配置项等等。

#### 普通单例模式-饿汉式
```
public class Singleton {
    private static Singleton singleton = new Singleton();
    private Singleton() {
    }
    public static Singleton getInstance() {
        return singleton;
    }
}
```
饿汉式单例，就是一个私有的构造方法加一个私有的静态当前类实例对象和一个公有的静态获取实例方法组成由于类实例对象为静态变量，所以在加载类的时候我们就会创建类的实例对象，这样的话比较消耗内存，浪费性能。 
<!--more-->
#### 普通单例模式-懒汉式
```
public class Singleton {
    private static Singleton singleton;
    private Singleton() {
    }
    public static Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
懒汉式在饿汉式的基础上做了改进，加了判断，类实例对象做了懒加载，也就是所谓的延时加载，所以提升了一些性能。

#### 多线程
关键词：volatile 
> <font color=red>被volatile修饰的变量能够保证每个线程能够获取该变量的最新值，从而避免出现数据脏读的现象。</font>
> 因为在多线程情况下执行顺序与我们最初想的是不同的，导致数据可能不是最新的。
我们就强制每次都直接读内存，阻止重排序，确保voltile类型的值一旦被写入缓存必定会被立即更新到主存。
#### 多线程 使用双重校验机制
```
public class Singleton {
    private static volatile Singleton singleton;
    private Singleton() {
    }
    public static Singleton getInstance() {
        // 先判断是否有线程实例对象已经创建。这样会让其他线程不会去等待获取锁
        if (singleton == null) {
            // 让先进入线程的线程创建示例
            synchronized (Singleton.class) {
                // 可能在同时进入时别的线程已经创建成功了
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
volatile 关键词在jdk1.5之后才完善的。
#### 静态内部类实现 单例模式 （这种实现比较好）
```
public class Singleton {
    private Singleton() {
    }
    private static class SingleTonBuilder {
        private static final Singleton singleton = new Singleton();
    }
    public static Singleton getInstance() {
        return SingleTonBuilder.singleton;
    }
}
```
主要原理为：Java中静态内部类可以访问其外部类的静态成员属性，同时，静态内部类只有当被调用的时候才开始首次被加载，(也就是这个时候占用内存)利用了classloader的机制来保证初始化instance时只有一个线程，所以也是线程安全的，同时没有性能损耗。（这里说的损耗是synchronized带来的损耗）
这个静态内部类的东西也可以写在类里，不过这里就变成线程安全的饿汉式了。

#### 解决单例模式反序列化产生新对象的方法
<font color=red>只要在对象中加入readResolve方法</font>

这样当JVM从内存中反序列化地"组装"一个新对象时,就会自动调用这个 readResolve方法来返回我们指定好的对象了, 单例规则也就得到了保证.
```
private Object readResolve() {
      return INSTANCE;
}
```


