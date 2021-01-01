# 谈谈对JVM内存分配

**JVM的分区可分为三个：堆（heap）、栈（stack）和方法区（method）：堆主要用来存放对象的，栈主要是用来执行程序的**

#### 堆区
1. 存储的是对象，并且JVM只有一个堆区，被所有线程共享。
2. 比如new string等等对象
3. 由于要在运行时动态分配内存，存取速度较慢
4. 先进先出

#### 栈区
1. 每个线程包含自己的一个栈区，栈中只保存基本数据类型的对象和自定义对象的**引用**记住是引用地址，指向堆中的对象
2. 栈与栈之间不能直接访问
3. 存取速度快，但是生命周期是固定的区域。
4. 先进后出
>栈有一个很重要的特殊性，就是存在栈中的数据可以共享。假设我们同时定义： 
int a = 3; 
int b = 3； 
编译器先处理int a = 3；首先它会在栈中创建一个变量为a的引用，然后查找栈中是否有3这个值，如果没找到，就将3存放进来，然后将a指向3。接着处理int b = 3；在创建完b的引用变量后，因为在栈中已经有3这个值，便将b直接指向3。这样，就出现了a与b同时均指向3的情况。
<!--more-->
#### 方法区
>又称为‘静态区’，和堆一样，被所有的线程共享。

#### 分析
```
String str1 = "abc"; 
String str2 = "abc"; 
System.out.println(str1==str2); //true 
```
```
String str1 =new String ("abc"); 
String str2 =new String ("abc"); 
System.out.println(str1==str2); // false 
```
> 两种的形式来创建，第一种是用new()来新建对象的，它会在存放于堆中。每调用一次就会创建一个新的对象。
这样就清晰了，new 会是新的对象地址：==比较的是内存地址
另一种就是先在栈中创建一个对String类的对象引用变量str，然后查找栈中有没有存放"abc"，如果没有，则将"abc"存放进栈，并令str指向”abc”，如果已经有”abc” 则直接令str指向“abc”。

```
String s1 = "ja"; 
String s2 = "va"; 
String s3 = "java";
String s4 = s1 + s2; 
System.out.println(s3 == s4);//false 
System.out.println(s3.equals(s4));//
```
>比较类里面的数值是否相等时，用equals()方法；s4最后指向的是新的一个java字符串地址。
String s3 = "ja"+"va"在编译期就已经变成了String s3 = "java"
String s4 = s1 + s2 在编译期无法确定 所以在运行的时候动态分配新的地址给他。
如果s1与s2是final修饰，那么"ja"+"va"跟s1 + s2 是没区别的，因为final修饰的变量会解析为常量值。




