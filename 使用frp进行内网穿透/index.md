# 使用frp进行内网穿透


> 从公网ip访问自己的机器-以前一直用的花生壳、NATAPP之类的;要么付费要么随机变化域名要么速度慢。frp可以解决这一类问题。

**搭建frp服务器进行内网穿透，可用且推荐，可以达到不错的速度，且理论上可以开放任何想要的端口，可以实现的功能远不止远程桌面或者文件共享**

<!--more-->

准备: 有公网ip的服务器(有域名可以配合ip映射访问如果要映射80端口需要配合nginx)


[frp地址](https://github.com/fatedier/frp/blob/master/README_zh.md)

#### 服务端配置
```
在 github 找到你电脑架构对应的版本下载
wget https://github.com/fatedier/frp/releases/download/v0.31.0/frp_0.31.0_linux_amd64.tar.gz
解压
tar -zxvf frp_0.31.0_linux_amd64.tar.gz
重命名文件夹
mv frp_0.31.0_linux_amd64  frp 
cd frp
ls 
```

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2020-01-06/1.png)
这里服务端，我们只需要关注 frps 、frps.ini
编辑 frps.ini 文件 
```
[common]
bind_port = 7000
dashboard_port = 7500
token = 12345678
dashboard_user = admin
dashboard_pwd = admin
```
1. bind_port表示用于客户端和服务端连接的端口，这个端口号我们之后在配置客户端的时候要用到。
2. dashboard_port是服务端仪表板的端口，若使用7500端口，在配置完成服务启动后可以通过浏览器访问 公网ip:7500 查看frp服务运行信息。
3. token是用于客户端和服务端连接的口令，请自行设置并记录，稍后会用到。
4. dashboard_user和dashboard_pwd表示打开仪表板页面登录的用户名和密码，自行设置即可。

运行
```
./frps -c frps.ini 运行 
nohup ./frps -c frps.ini & 后台运行
ps -ef|grep frp 找frp应用进程 
```

#### 客户端设置
frp的客户端就是我们想要真正进行访问的那台设备，大多数情况下应该会是一台Windows主机，这里使用Windows主机做例子；Linux配置方法类似。

```
在 github 找到你电脑架构对应的版本下载 解压
https://github.com/fatedier/frp/releases/download/v0.31.0/frp_0.31.0_windows_amd64.zip
编辑 frpc.ini 这个文件 
[common]
server_addr = 服务器地址
server_port = 7000
token = 12345678

[rdp]  规则名称自定义
type = tcp 用什么协议
local_ip = 127.0.0.1           
local_port = 8521 映射本地端口
remote_port = 8521  服务器端口 映射本地端口
```

运行
```
./frpc -c frpc.ini 
```


#### 完整配置

[frps的完整配置文件（服务器）](https://github.com/fatedier/frp/blob/master/conf/frps_full.ini)

[frpc的完整配置文件（客户端）](https://github.com/fatedier/frp/blob/master/conf/frpc_full.ini)








