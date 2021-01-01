# 原型设计模式


#### 应用场景
原型模式就是从一个对象再创建另外一个可定制的对象，而且不需要知道任何创建的细节。
所谓原型模式，就是 Java 中的克隆技术，以某个对象为原型。复制出新的对象。显然新的对象具备原
型对象的特点，效率高（<font color=red>避免了重新执行构造过程步骤,而且实例的创建开销比较大或者需要输入较多参数</font>）。

#### 浅克隆
##### 说明
浅复制仅仅复制所克隆的对象，而不复制它所引用的对象。 Object类提供的方法clone只是拷贝本对象，其对象内部的数组、引用对象等都不拷贝，还是指向原生对象的内部元素地址
<!--more-->
##### 代码(使用lombok插件省去get与set)
student类---clone都需要实现Cloneable这个类
```
@Getter
@Setter
public class Student implements Cloneable {
    private String name;
    private int id;
    private Lesson lesson;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```
Lesson类
```
@Setter
@Getter
public class Lesson {
    private String name;
}
```
浅克隆测试类
```
public class ShallowTest {
    public static void main(String[] args) {
        Student student = new Student();
        Lesson lesson = new Lesson();
        student.setLesson(lesson);

        try {
            Student cloneStuden = (Student) student.clone();
            System.out.println(cloneStuden == student);
            System.out.println(cloneStuden.getLesson() == student.getLesson());
        } catch (CloneNotSupportedException e) {
            System.out.println("克隆失败!");
            e.printStackTrace();
        }
    }
}
-------------------------------
输出
false
true
```
从上面可以看出Object类提供的方法clone只是拷贝本对象，其中对象内部的引用对象还是指向原来的地址

#### 深克隆
##### 说明
<font color=red>深复制把要复制的对象所引用的对象都复制了一遍</font>
深复制需要把对象序列化并且所引用的对象也需要序列化
##### 代码
```
@Getter
@Setter
public class Student implements Serializable {
    private String name;
    private int id;
    private Lesson lesson;

    protected Object deepClone() throws IOException, ClassNotFoundException {
        /* 写入当前对象的二进制流 */
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(this);

        /* 读出二进制流产生的新对象 */
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(byteArrayOutputStream.toByteArray());
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
        return (Student) objectInputStream.readObject();
    }
}
```
Lesson类
```
@Setter
@Getter
public class Lesson implements Serializable {
    private String name;
}
```
浅克隆测试类
```
public class deepTest {
    public static void main(String[] args) {
        Student student = new Student();
        Lesson lesson = new Lesson();
        student.setLesson(lesson);

        try {
            Student cloneStuden = (Student) student.clone();
            System.out.println(cloneStuden == student);
            System.out.println(cloneStuden.getLesson() == student.getLesson());
        } catch (CloneNotSupportedException e) {
            System.out.println("克隆失败!");
            e.printStackTrace();
        }
    }
}
-------------------------------
输出
false
false
```



