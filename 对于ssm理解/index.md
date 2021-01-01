# 对SSM的理解

- MyBatis：持久层框架，负责数据库访问。
- Spring MVC：表现层框架，把模型、视图、控制器分离，组合成一个灵活的系统。
- Spring： 整合项目的所有框架，管理各种Java Bean（mapper、service、controller），事务控制。


#### Spring中的IOC（控制反转）、DI（依赖注入）、AOP（切面）
1. IOC（Spring核心）可以认为是一个生产和管理bean的容器（可以管理bean的生命周期）；在原来需要new对象时，现在不需要了。
2. DI注入方式有三种
> - set注入（根据属性注入）
> - 构造方法注入
> - 根据接口注入 
3. IOC与DI可以理解为：
> 控制反转是动态的向某个对象提供他所需要的对象，这一点是通过DI依赖注入实现的。
4. AOP面向切面:是一种编程思想，将程序中的交叉业务逻辑（比如安全，日志，事务等），封装成一个切面,注入到目标对象中。
5. AOP是通过代理实现的：可以是JDK动态代理，也可以是cglib代理，前者基于接口，后者基于子类。
<!--more-->
#### Spring Mvc （Model、View、Controller）
1. SpringMvc是基于过滤器对servlet进行了封装的一个框架，我们使用的时候就是在web.xml文件中配置DispatcherServlet类。
2. SpringMvc工作时主要是通过DispatcherServlet管理接收到的请求并进行处理。
3. 流程
> 发送请求到DispatcherServlet->前端控制器通过HandlerMapping找映射地址->通过映射地址执行方法->返回ModelAndView（视图）->通过视图渲染返回界面

#### Mybatis
> 一个持久层框架,负责数据库操作；
