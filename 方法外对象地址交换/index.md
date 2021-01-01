# 方法外对象地址交换


#### 面试题如下
```
public class Cat {
    private int inr;
    private Cat(int inr) {
        this.inr = inr;
    }
    private static void print(Cat cat) {
        System.out.println(cat.inr);
    }
    private void setInr(int inr) {
        this.inr = inr;
    }
    public static void main(String[] args) {
        Cat A = new Cat(100);
        Cat B = new Cat(101);
        System.out.println(A);
        System.out.println(B);
        swap(A, B);
        Cat.print(A);
        Cat.print(B);
    }
    private static void swap(Cat A, Cat B) {
        Cat temp = A;
        A = B;
        B = temp;
        System.out.println(A.inr);
        System.out.println(B.inr);
//        temp = new Cat(400);
//        B.setInr(400);
//        System.out.println(temp.inr);
//        System.out.println(B.inr);
    }
}
```
<!--more-->
#### 输出
```
com.yakax.Cat@299a06ac
com.yakax.Cat@383534aa
101
100
100
101
```
> 从这里的输出我们可以理解到，对象的传值引用只在方法内起作用方法外的地址也是改变不了的。

#### 放掉注释输出
```
com.yakax.Cat@299a06ac
com.yakax.Cat@383534aa
101
100
400
400
400
101
```
#### 分析
1. teap已经是新的对象了输出400。
2. B.setInr 时B对象指向的是A对象的地址，也就是说他吧com.yakax.Cat@299a06ac 这个地址的对象的值改了。所以方法外A打印400。
3. 最后B的地址依旧没有变，打印101。
4. 这里我们只要记住改变指向地址对象的值只需利用属性方法更改。new对象是产生新的对象地址。

