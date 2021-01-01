# 代理设计模式


#### 应用场景
为其他对象提供一种代理以控制对这个对象的访问。即通过代理对象访问目标对象.这样做的好处是:可以在目标对象实现的基础上,增强额外的功能操作,即扩展目标对象的功能。
<font color=red>关键:代理对象与目标对象.代理对象是对目标对象的扩展,并会调用目标对象;这里使用到编程中的一个思想:不要随意去修改别人已经写好的代码或者方法,如果需改修改,可以通过代理的方式来扩展该方法;</font>

#### 静态代理

##### 说明
静态代理在使用时,需要定义接口或者父类,被代理对象与代理对象一起实现相同的接口或者是继承相同父类.
<font color=red>关键：在编译期确定代理对象，在程序运行前代理类的.class文件就已经存在了。</font>

<!--more-->
##### 代码(我让媒婆代理我帮我找个女朋友)
接口类Person
```
public interface Person {
    /**
     * 我喜欢什么
     */
    void findLove();
}
```
代理类Meipo
```
class Meipo implements Person {
    private Person person;
    Meipo(Person person) {
        this.person = person;
    }
    @Override
    public void findLove() {
        System.out.println("媒婆在帮你寻找");
        this.person.findLove();
        System.out.println("目前没找到");
    }
}
```
被代理类Me
```
public class Me implements Person {
    @Override
    public void findLove() {
        System.out.println("年龄比我小");
        System.out.println("有共同爱好");
        System.out.println("好相处");
    }
}
```
测试类
```
public class Test {
    public static void main(String[] args) {
        // Meipo是代理对象 Me是被代理对象
        Meipo meipo = new Meipo(new Me());
        meipo.findLove();
    }
}
输出
-----------------------------------
媒婆在帮你寻找
年龄比我小
有共同爱好
好相处
目前没找到
```

##### 总结
1. 可以做到在不修改目标对象的功能前提下,对目标功能扩展.
2. 缺点:代理类和被代理类实现相同的接口，同时要实现相同的方法。这样就出现了大量的代码重复。如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度。


#### 字节码重组
<font color=red>以下这个过程就叫字节码重组</font>
1. 拿到被代理对象的引用，并且获取到它的所有的接口，反射获取
2. JDK Proxy类重新生成一个新的类、同时新的类要实现被代理类所有实现的所有的接口
3. 动态生成Java代码，把新加的业务逻辑方法由一定的逻辑代码去调用（在代码中体现）
4. 编译新生成的Java代码.class
5. 再重新加载到JVM中运行

#### jdk动态代理
##### 说明
在运行时，通过反射机制创建一个实现了一组给定接口的新类

##### 代码(我让我妈妈代理我去寄快递和代我点一份外卖)
接口类Person
```
public interface Person {
    /**
     * 点外卖
     */
    void pointTakeaway();
    /**
     * 送快递
     */
    void sendDelivery();
}
```
代理类Mom (这里也就是类似拦截器、AOP的实现过程)
```
public class Mom implements InvocationHandler {
    
    private Object object;

    Object getInstance(Object o) {
        this.object = o;
        Class<?> clazz = o.getClass();
        // 生成新的对象
        return Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), this);
    }

    /**
     * 类似动态代理拦截器
     *
     * @param proxy
     * @param method
     * @param args
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("调用代理类");

        if ("pointTakeaway".equals(method.getName())) {
            System.out.println("我儿子叫我帮他点外卖");
        } else if ("sendDelivery".equals(method.getName())) {
            System.out.println("我儿子叫我帮他寄快递");
        }
        method.invoke(this.object, args);
        return null;
    }
}
```
被代理类Me
```
public class Me implements Person {
    @Override
    public void pointTakeaway() {
        System.out.println("我想要一份炸鸡");
    }
    @Override
    public void sendDelivery() {
        System.out.println("帮我寄快递");
    }
}
```
测试类 (这里我通过反编译工具查看代理类具体代码)
```
public class Test {
    public static void main(String[] args) {
        // 被代理对象
        Me me = new Me();

        try {
            // 动态代理
            Person instance = (Person) new Mom().getInstance(me);
            System.out.println("生成的代理类"+instance.getClass());
            instance.pointTakeaway();
            instance.sendDelivery();

            //JDK中有个规范，只要要是$开头的一般都是自动生成的
            //通过反编译工具可以查看源代码

            byte [] bytes = ProxyGenerator.generateProxyClass("$Proxy0",new Class[]{Person.class});
            FileOutputStream os = new FileOutputStream("E://$Proxy0.class");
            os.write(bytes);
            os.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
输出
---------------------------
生成的代理类class com.sun.proxy.$Proxy0
调用代理类
我儿子叫我帮他点外卖
我想要一份炸鸡
调用代理类
我儿子叫我帮他寄快递
帮我寄快递
```
##### 查看自动生成的代理类
这个刚刚生成的代理类，我把他放进idea查看源代码
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/6.png" />
可以看出是通过反射生成的并且继承了Proxy和实现了接口
```
public final class $Proxy0 extends Proxy implements Person {
    private static Method m1;
    private static Method m4;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sendDelivery() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void pointTakeaway() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("com.example.designpatterns.proxy.jdkproxy.Person").getMethod("sendDelivery");
            m3 = Class.forName("com.example.designpatterns.proxy.jdkproxy.Person").getMethod("pointTakeaway");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

##### 总结
1. 代理对象不需要实现接口,但是目标对象一定要实现接口,否则不能用动态代理。
2. 接口中声明的所有方法都被转移到调用处理器一个集中的方法中处理（InvocationHandler.invoke）。这样，在接口方法数量比较多的时候，我们可以进行灵活处理，而不需要像静态代理那样每一个方法进行中转。而且动态代理的应用使我们的类职责更加单一，复用性更强。

#### Cglib代理

##### 说明
上面的静态代理和动态代理模式都是要求目标对象实现一个接口或者多个接口,但是有时候目标对象只是一个单独的对象,并没有实现任何的接口,这个时候就可以使用构建目标对象子类的方式实现代理,这种方法就叫做:Cglib代理；


##### 代码（代理我去寄快递和代我点一份外卖）
代理类CglibProxy
```
public class CglibProxy implements MethodInterceptor {
    private Object object;

    CglibProxy(Object o) {
        this.object = o;
    }

    Object getInstance() {
        Enhancer enhancer = new Enhancer();
        //要把哪个设置为即将生成的新类父类
        enhancer.setSuperclass(this.object.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        // 这里可以做代理类的操作
        methodProxy.invokeSuper(o, objects);
        return null;
    }
}
```
被代理类Me
```
public class Me {
    public void pointTakeaway() {
        System.out.println("我想要一份炸鸡");
    }
    /**
     * final 方法不会被生成的子类覆盖
     */
    public final void sendDelivery() {
        System.out.println("帮我寄快递");
    }
}
```
测试类
```
public class Test {
    public static void main(String[] args) {
        Me m=new Me();
        // 这里根据自己需要代理的类传值
        Me me = (Me)new CglibProxy(m).getInstance();
        me.pointTakeaway();
        me.sendDelivery();
    }
}
输出
---------------------------
我想要一份炸鸡
帮我寄快递
```

##### 总结
- Cglib包的底层是通过使用字节码处理框架ASM来转换字节码并生成新的子类.（动态代理都是通过字节码生成的）
<font color=red>
- Cglib是一个强大的高性能的代码生成包,它可以在运行期扩展java类与实现java接口.它广泛的被许多AOP的框架使用,例如Spring AOP和synaop,为他们提供方法的interception(拦截器)
</font>
- 为JDK的动态代理提供了很好的补充。通常可以使用Java的动态代理创建代理，但当要代理的类没有实现接口或者为了更好的性能，Cglib是一个好的选择。





