# Servlet面试题总结

**事实上，Servlet就是一个Java接口：具体可以看这里：**


![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/15.png)
<!--more-->
#### Servlet是干嘛的
Servlet接口定义的是一套处理网络请求的规范，所有实现Servlet的类，都需要实现它那五个方法，其中最主要的是两个生命周期方法 init()和destroy()，还有一个处理请求的service()，也就是说，所有实现servlet接口的类，或者说，所有想要处理网络请求的类，都需要回答这三个问题：
> - 你初始化时要做什么
> - 你销毁时要做什么
> - 你接受到请求时要做什么 

Servlet只是定义了一套规范（接口即是规范），处理请求还是需要一个容器，不然Servlet不会起作用，比如Tomcat 他监听了端口，根据url确定交给那个Servlet处理，然后才有了Servlet的操作。

#### Servlet生命周期
1. 发送请求
2. 解析请求
3. 加载和实例化init（）方法
>  第一次创建Servlet调用在后续每次请求这个Servlet都不会调用。
4. 响应客户端请求调用Service（）方法具体执行doGet()或者doPost()方法
5. 输出响应信息
6. 返回响应信息
7. 终止阶段，调用destroy()方法
> destroy() 方法只会被调用一次，在 Servlet 生命周期结束时被调用。destroy() 方法可以让您的 Servlet 关闭数据库连接、停止后台线程、等等清理操作。
8. 最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的

#### 请求转发（Forward）和重定向（sendredirect）的区别
>1. Forward是在服务器端的跳转，就是客户端一个请求发给服务器，服务器直接将请求相关的参数的信息原封不动的传递到该服务器的其他jsp或Servlet去处理。
>2. sendredirect是在客户端的跳转，服务器会返回给客户端一个响应报头和新的URL地址，原来的参数什么的信息如果服务器端没有特别处理就不存在了，浏览器会访问新的URL所指向的Servlet或jsp：**重定向可以防止表单重复提交**。
>3. 转发的是同一次请求；重定向是两次不同请求
>4. 转发地址栏没有变化；重定向地址栏有变化

#### 讲讲filter listener 与Servlet 
1. 执行顺序
>  listener(监听器) -> filter（过滤器） -> Servlet
2. Servlet
> 当客户端向服务器发出HTTP请求时，首先会由服务器中的 Web 容器（如Tomcat）对请求进行路由，交给该URL对应的 Servlet 进行处理，Servlet 所要做的事情就是返回适当的内容给用户
3. filter
> Filter 是介于 Web 容器和 Servlet 之间的过滤器，用于过滤未到达 Servlet 的请求或者由 Servlet 生成但还未返回响应。
4. listener
>基于观察者模式：Listener 用于监听 java web程序中的事件，例如创建、修改、删除Session、request、context等，并触发响应的事件。比如显示在线人数；单态登录：一个账号只能在一台机器上登录；等等全局的操作。


