# Docker分布式部署zookeeper集群


#### 准备
- 3台机器
- [安装docker](https://yakax.github.io/2018/11/05/%E5%AD%A6%E4%B9%A0docker%E5%B9%B6%E5%AE%89%E8%A3%85%E4%B8%80%E4%B8%AA%E8%87%AA%E5%90%AF%E5%8A%A8MySQL8/)

```
HOST1: centos7.5 :172.16.217.135 zk1
HOST2: centos7.5 :172.16.217.136 zk2
HOST3: centos7.5 :172.16.217.137 zk3
```

<!--more-->

#### 先把防火墙开放三个端口
```
sudo firewall-cmd --zone=public --add-port=2181/tcp --permanent
sudo firewall-cmd --zone=public --add-port=2888/tcp --permanent
sudo firewall-cmd --zone=public --add-port=3888/tcp --permanent
sudo firewall-cmd --reload
2181：对cline端提供服务
3888：选举leader使用
2888：集群内机器通讯使用（Leader监听此端口）
```
检查防火墙规则
```
firewall-cmd --list-all
```
防火墙的一些命令
```
//临时关闭防火墙,重启后会重新自动打开
systemctl restart firewalld
//检查防火墙状态
firewall-cmd --state
firewall-cmd --list-all
//Disable firewall
systemctl disable firewalld
systemctl stop firewalld
systemctl status firewalld
//Enable firewall
systemctl enable firewalld
systemctl start firewalld
systemctl status firewalld
```

#### 分别安装容器 并跟随docker启动
```
HOST1
docker run -d --restart=always  --net=host   --name=zookeeper1 zookeeper
HOST2
docker run -d --restart=always  --net=host   --name=zookeeper2 zookeeper
HOST3
docker run -d --restart=always  --net=host   --name=zookeeper3 zookeeper
```

#### 编辑zoo.cfg 与 myid
我这里是先编辑其中一个然后copy到容器里面 三个容器 zoo.cfg是一样的 文件是在/conf/下面
```
docker cp 文件 容器id:文件  //copy文件到容器
docker cp 容器id:文件 文件  //copy容器文件到宿主机
```

```
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
clientPort=2181
server.1=172.16.217.135:2888:3888
server.2=172.16.217.136:2888:3888
server.3=172.16.217.137:2888:3888
```
myid修改为对应server后面的id 
```
重启
docker restart 容器id
查看日志
docker logs -f 容器id
```
**注意myid是否是正确就说明编辑的东西是否生效了日志也有打印**
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/zookeeper/%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/2.png)

#### 观察状态
进入容器
```
docker exec -it 容器id bash
bin/zkServer.sh status
```

最后效果
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/zookeeper/%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/1.png)
