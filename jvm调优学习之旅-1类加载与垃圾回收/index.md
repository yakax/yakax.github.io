# JVM调优学习之旅-(1)类加载与垃圾回收


### 类加载过程

```java
public class Test {

    public static void main(String[] args) {
        A a = new A();
    }
}
```
<!--more-->

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200527110944.png" style="zoom: 50%;" />

加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载

1. 验证：加载类时验证类语法问题，符合JVM规范,并且把需要用到的类同时加载

2. 准备：给类分配内存空间，给变量分配默认值 比如int类型给0 对象给开辟空间

3. 解析：对字节码进行符号转化，符号引用替换为直接引用

4. 初始化：对类里面的值进行赋值，比如给对象赋值new A()-如果有父类还需要先初始化父类; 对静态块变量赋值。

   

### 类加载器

上面这些过程都需要通过类加载器加载进来

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200527111406.png" style="zoom:50%;" />

比如我们应用程序要加载一个类：Application ClassLoader会问扩展类加载器 扩展类加载器会问启动类加载器，然后启动类加载器找了自己目录半天没找到这类就会告诉扩展类让他加载，扩展类找了没找到就会让应用程序类加载。也就是**先问父类父类加载不了再压到子类 直到加载**

<font color=red>类加载器本身就是做的父子关系模型 ,这也是代码设计思想，保证代码的可扩展性,各司其职！顶层类加载器就不会硬编码规定了，任何加载器都可以扩展。</font>

#### **双亲委派机制的一点重要性**

双亲委派模型设计的出发点很重要，对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。
也就是说:判断2个类是否 <font color=red>“相等”</font>，只有在这2个类是由同一个类加载器加载的前提下才有意义，否则即使这2个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，这2个类必定不相等。
**基于双亲委派模型设计，那么Java中基础的类，Object类似Object类重复多次的问题就不会存在了，因为经过层层传递，加载请求最终都会被Bootstrap ClassLoader所响应。加载的Object类也会只有一个，否则如果用户自己编写了一个java.lang.Object类，并把它放到了ClassPath中，会出现很多个Object类，这样Java类型体系中最最基础的行为都无法保证，应用程序也将一片混乱**

### tomcat的双亲委派机制

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200805180233.png" style="zoom:50%;" />

Tomcat自定义了Common、Catalina、Shared等类加载器，其实就是用来加载Tomcat自己的一些核心基础类库的。 然后Tomcat为每个部署在里面的Web应用都有一个对应的WebApp类加载器，负责加载我们部署的这个Web应用的类 至于Jsp类加载器，则是给每个JSP都准备了一个Jsp类加载器。 而且大家一定要记得，Tomcat是**打破了双亲委派机制**的 每个WebApp负责加载自己对应的那个Web应用的class文件，也就是我们写好的某个系统打包好的war包中的所有class文 件，不会传导给上层类加载器去加载。

#### 打破双亲委派的原因

1. **tomcat中的需要支持不同web应用依赖同一个第三方类库的不同版本，jar类库需要保证相互隔离**；

2. 同一个第三方类库的相同版本在不同web应用可以共享 

3. tomcat自身依赖的类库需要与应用依赖的类库隔离 

4. jsp需要支持修改后不用重启tomcat即可生效 为了上面类加载隔离 和类更新不用重启，定制开发各种的类加载器



#### 怎么让别人不能反编译你的代码

在对字节码进行加密，然后自定义类加载器来解密

### JVM内存划分

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200527115000.png" style="zoom:50%;" />

还有一个区域，是不属于JVM的，通过NIO中的allocateDirect这种API，可以在Java堆外分配内存空间。然后，通过Java虚拟机里的DirectByteBuffer来引用和操作堆外内存空间。



### 怎么判定对象是否可以回收

**可达性分析**

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200806111558.png" style="zoom:33%;" />

在一个方法中创建了一个对象，然后有一个局部变量引用了这个对象，这种情况是最常见的

此时main()方法的栈帧入栈，然后调用"loadReplicasFromDisk()"方法，栈帧入栈，接着让局部变量 "replicaManager"引用堆内存里的"ReplicaManager"实例对象

当回收线程分析时 **局部变量就是可以作为GC Roots的**只要一个对象被局部变量引用了，那么就说明他有一个GC Roots，此时就不能被回收了。（其实就是栈中变量表引用的）

同理在JVM的规范里，**静态变量也可以看做是一种GC Roots**，此时只要一个对象被GC Roots引用了，就不会去回收他。(其实就是方法区引用的)

`Native方法引用的也可以`

首先该类的所有实例（堆中）都已经被回收；其次该类的ClassLoader已经被回收；最后，对该类对应的Class对象 没有任何引用

```
为什么没有用引用计数器的问题
 Step1：GcObject实例1的引用计数加1，实例1的引用计数=1； 
 Step2：GcObject实例2的引用计数加1，实例2的引用计数=1； 
 Step3：GcObject实例2的引用计数再加1，实例2的引用计数=2； 
 Step4：GcObject实例1的引用计数再加1，实例1的引用计数=2； 
 执行到Step 4，则GcObject实例1和实例2的引用计数都等于2。
 Step5：栈帧中obj1不再指向Java堆，GcObject实例1的引用计数减1，结果为1； 
 Step6：栈帧中obj2不再指向Java堆，GcObject实例2的引用计数减1，结果为1。 
 到此，发现GcObject实例1和实例2的计数引用都不为0，那么如果采用的引用计数算法的话，那么这两个实例所占的内存将得不到释放，这便产生了内存泄露。
```



### Java中对象不同的引用类型

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200806112426.png" style="zoom:50%;" />

**弱引用**：就跟没引用是类似的，如果发生垃圾回收，就会把这个对象回收掉。 

**强引用**：强引用就是代表绝对不能回收的对象，

**软引用**：就是说有的对象可有可无，如果内存实在不 够了，可以回收他。

虚引用，很少用，暂时忽略。

##### **finalize()** 

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/jvm/TIM%E6%88%AA%E5%9B%BE20200806113018.png" style="zoom: 50%;" />

假设有一个Kafka对象要被垃圾回收了，那么假如这个对象重写了Object类中的finialize()方法 此时会先尝试调用一下他的finalize()方法，看是否把自己这个实例对象给了某个GC Roots变量，比如说代码中就给了kafka类的静态变量。 如果重新让某个GC Roots变量引用了自己，那么就不用被垃圾回收了。

