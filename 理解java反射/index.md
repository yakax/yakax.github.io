# 理解java反射

**Java 反射是可以让我们在运行时获取类的方法、属性、父类、接口等类的内部信息的机制**

#### 为什么用反射
假如你写了一段代码：Object o=new Object();
运行了起来！扔给jvm去跑，跑完就over了，jvm关闭，你的程序也停止了;
想想上面的程序对象是自己new的，程序相当于写死了给jvm去跑。假如一个服务器上突然遇到某个请求哦要用到某个类，哎呀但没加载进jvm，是不是要停下来自己写段代码，new一下，哦启动一下服务器，（脑残）！
反射是什么呢？当我们的程序在运行时，需要动态的加载一些类这些类可能之前用不到所以不用加载到jvm，而是在运行时根据需要才加载，这样的好处对于服务器来说不言而喻。
> 反射是框架的灵魂
<!--more-->
#### 怎么创建
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/11.png)
> 但是这个怎么实现动态的呢，这里的forName方法里面的字符串就是动态的。不同的类我们用不同的字符串 这样就可以在运行时加载类来用了。
```
 Object object = c.newInstance();
 // 所有共有方法，这就包括自身的所有public方法，和从基类继承的、从接口实现的所有public方法
 Method[] methods = c.getMethods();
 // 自身声明的所有方法，包含public、protected和private方法
 Method[] declaredMethods = c.getDeclaredMethods();
 //获取methodClass类的add方法
 Method method = c.getMethod("add",int.class,int.class);
 Object result = method.invoke(obj,1,4);
```
#####   反射调用一般分为3个步骤：
1. 得到要调用类的class
2. 得到要调用的类中的方法(Method)
3. 方法调用(invoke)








