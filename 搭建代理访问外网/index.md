# 搭建代理访问外网


## shadowsocks

**文档总结自秋水逸冰大神 更详细可以看**

```
https://teddysun.com/(需要外网访问)
```

1. 首先，**买服务器我就不说**了，一般只翻墙的话就买搬瓦工或者vultr都行；里面最便宜的都行（除了vultr2.5刀的）；找一个ip如果ping 的延时在200以内算是挺可以的了；
2. 这里我采用Vultr3.5刀的；选哪个地方主要看你的网络对他们哪个服务器友好,下面附上测速脚本。

```
@echo off
echo ===========================================
echo 东京
ping hnd-jp-ping.vultr.com
 
echo ============================================
echo 新加坡
ping sgp-ping.vultr.com
 
echo ===========================================
echo (AU) Sydney, Australia[悉尼]
ping syd-au-ping.vultr.com
 
echo ===========================================
echo 德国 法兰克福
ping fra-de-ping.vultr.com
 
echo ===========================================
echo 荷兰 阿姆斯特丹
ping ams-nl-ping.vultr.com
 
echo ===========================================
echo 英国 伦敦
ping lon-gb-ping.vultr.com
 
echo ===========================================
echo 法国 巴黎
ping par-fr-ping.vultr.com
 
echo ===========================================
echo 美东 华盛顿州 西雅图
ping wa-us-ping.vultr.com
 
echo ===========================================
echo 美西 加州 硅谷
ping sjo-ca-us-ping.vultr.com
 
echo ===========================================
echo 美西 加州 洛杉矶
ping lax-ca-us-ping.vultr.com
 
echo ===========================================
echo 美东 芝加哥
Chicago, Illinois[美东 芝加哥]
ping il-us-ping.vultr.com
 
echo ===========================================
echo 美中 德克萨斯州 达拉斯
ping tx-us-ping.vultr.com
 
echo ===========================================
echo 美东 新泽西
ping nj-us-ping.vultr.com
 
echo ===========================================
echo 美东 乔治亚州 亚特兰大
ping ga-us-ping.vultr.com
 
echo ===========================================
echo 美东 佛罗里达州 迈阿密
ping fl-us-ping.vultr.com   
pause
```

3. 服务器信息：用于Xshell连接服务器
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/1.png"  />
<!--more-->
4. 服务器连接
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/2.png" />
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/3.png"  />
5. 使用一键安装ss 脚本 
```
wget --no-check-certificate -O shadowsocks-all.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
chmod +x shadowsocks-all.sh
./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log
```
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/4.png" />
安装成功图
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/5.png" />
6. 用ss软件连接
> https://github.com/shadowsocks
> https://github.com/shadowsocksrr
7. 打开软件填入数据 代理即可
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/ss/6.png" />
> 这里要注意全局代理是把本地出入ip完全代理你设置的ip；PAC模式是访问国内用本地ip访问外网会自动代理；
8. 配置加速
> 使用root用户登录，运行以下命令：
```
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh && chmod +x bbr.sh && ./bbr.sh
```
> 安装完成后，脚本会提示需要重启 VPS，输入 y 并回车后重启。
重启完成后，进入 VPS，验证一下是否成功安装最新内核并开启 TCP BBR，输入以下命令：
```
uname -r
```
> 查看内核版本，显示为最新版就表示 OK 了
```
sysctl net.ipv4.tcp_available_congestion_control
```
> 返回值一般为：
net.ipv4.tcp_available_congestion_control = bbr cubic reno
或者为：
net.ipv4.tcp_available_congestion_control = reno cubic bbr
```
sysctl net.ipv4.tcp_congestion_control
```
> 返回值一般为：
net.ipv4.tcp_congestion_control = bbr
```
sysctl net.core.default_qdisc
```
> 返回值一般为：
net.core.default_qdisc = fq
```
lsmod | grep bbr
```
> 返回值有 tcp_bbr 模块即说明 bbr 已启动。<font color=red >注意：并不是所有的 VPS 都会有此返回值，若没有也属正常。</font>
> 
配置完后，基本上洛杉矶硅谷东京这几个地方的ip跑7m左右秒没问题。



## V2Ray 

**目前我用的方式**

```shell
bash <(curl -s -L https://git.io/v2ray.sh) 一键脚本
bash <(curl -Ls https://blog.sprov.xyz/v2-ui.sh) 带有控制台的一键脚本
```


