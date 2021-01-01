# 委派设计模式


#### 应用场景
<font color=red>委派其实就是静态代理和策略模式一种特殊的组合,因为代理模式注重过程，而委派模式注重结果。</font>
在我们日常开发中的spring mvc中的DispatcherServlet类就是用了委派模式,其原理就是根据用户的url在handlerMapping里面找到对应的处理类,然后委派到具体方法。在 Spring 源码中，只要以 Delegate 结尾的都是实现了委派模式。例如：BeanDefinitionParserDelegate 根据不同类型委派不同的逻辑解析 BeanDefinition。

由于不属于GOF 23种设计模式之一
- 程序当中一般是精简程序逻辑提升代码可读性

<!--more-->
#### 代码
老板叫领导做事情,领导根据需求分发任务案例
创建老板类命令领导的方法：
```
class Boss {
    /**
     * 传命令给领导让他分发任务
     *
     * @param command 什么事情
     * @param leader 那个领导
     */
    void command(String command, Leader leader) {
        leader.doing(command);
    }
}
```
干活接口:领导跟员工都是打工的
```
public interface Action {
    /**
     * 接收方法执行
     *
     * @param command 命令信息
     */
    void doing(String command);
}
```
领导类：
```
public class Leader implements Action {
    private Map<String, Action> targets = new HashMap<>();
    Leader() {
        targets.put("加密", new ProgrammerA());
        targets.put("登录", new ProgrammerB());
    }
    @Override
    public void doing(String command) {
        targets.get(command).doing(command);
    }
}
```
员工类：
```
public class ProgrammerA implements Action {
    @Override
    public void doing(String command) {
        log.info("我是码农A：我开始工作" + command);
    }
}
public class ProgrammerB implements Action {
    @Override
    public void doing(String command) {
        log.info("我是码农B：我开始工作" + command);
    }
}
```
测试委派：
```
public class Test {
    public static void main(String[] args) {
        // boss
        Boss boss = new Boss();
        // 那个领导
        Leader leader = new Leader();
        // boss安排任务给这个领导
        boss.command("登录", leader);
        boss.command("加密", leader);
    }
}
输出----------------------------
 我是码农B：我开始工作登录
 我是码农A：我开始工作加密
```
#### 类图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/designPatterns/1.png"/>



