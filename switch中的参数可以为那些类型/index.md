# switch中的参数可以为那些类型

**之前面试被问懵了一个很简单的问题，switch中的参数支持string吗，我当时想都没想，直接回答string可以为switch的参数，因为在我写的代码中我的确拿string当过参数：然而面试官说不可以。
(经过我的验证，我发现jdk1.7以后switch就支持String类型了；面试官傻逼！)**

#### 官方示例
```
https://docs.oracle.com/javase/tutorial/java/nutsandbolts/switch.html
```

<!--more-->
#### 语法格式如下：
```
switch(expression){
    case value :
       //语句
       break; //可选
    case value :
       //语句
       break; //可选
    //你可以有任意数量的case语句
    default : //可选
       //语句
}
```
#### 这里的 expression 都支持哪些类型呢？
- 基本数据类型：byte, short, char, int
- 包装数据类型：Byte, Short, Character, Integer
- 枚举类型：Enum
- 字符串类型：String（Jdk 7+ 开始支持）

#### 基本数据类型和字符串很简单不用说，下面举一个使用包装类型和枚举的，其实也不难，注意只能用在 switch 块里面。
```
// 使用包装类型
Integer value = 5;
switch (value) {
    case 3:
        System.out.println("3");
        break;
    case 5:
        System.out.println("5");
        break;
    default:
        System.out.println("default");
}

// 使用枚举类型
Status status = Status.PROCESSING;
// 也可以直接把枚举类放进去（官方文档有讲）
switch (status) {
    case OPEN:
        System.out.println("open");
        break;
    case PROCESSING:
        System.out.println("processing");
        break;
    case CLOSE:
        System.out.println("close");
        break;
    default:
        System.out.println("default");
}
```
#### 使用 switch case 语句也有以下几点需要注意。
- case 里面必须跟 break，不然程序会一个个 case 执行下去，直到最后一个 break 的 case 或者 default 出现。
- case 条件里面只能是常量或者字面常量。
- default 语句可有可无，最多只能有一个。

