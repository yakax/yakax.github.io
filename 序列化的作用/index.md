# 序列化的作用


#### 序列化的作用
1. 序列化
>有的时候我们想要把一个Java对象变成字节流的形式以便存储在文件或者网络传输：比如序列化成json格式通过http发送出去。
2. 反序列化
> 有的时候我们想要从一个字节流中恢复一个Java对象。：比如在网络中接受到一个json格式字符串，把它转化为某个对象。
<!--more-->

#### 序列化的什么特点：
>如果某个类能够被序列化，其子类也可以被序列化。声明为static和transient类型的成员数据不能被序列化。因为static代表类的状态， transient代表对象的临时数据。

#### 序列化的方式
1. 是相应的对象实现了序列化接口Serializable，这个使用的比较多，对于序列化接口Serializable接口是一个空的接口，它的主要作用就是标识这个对象时可序列化的。
```
package com.shop.domain;

import java.util.Date;


public class Article implements java.io.Serializable {
    private Integer id; 
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
}
```
2. 实现接口Externalizable
> 由于这个类实现了Externalizable 接口，在writeExternal()方法里定义了哪些属性可以序列化，

序列化的方式有很多，这上面的是最基本的序列化，我们在项目中常用的一般是json转换对象，对象转换json也算是序列化。


