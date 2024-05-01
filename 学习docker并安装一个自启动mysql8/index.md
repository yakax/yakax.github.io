# 自启动docker并安装自启动镜像MySQL8

**因为在10月15号换了新工作，入职了半月左右，这半月时间一直在搞前端的ts和angular，再加上在忙着自考，后端的代码一直没有碰，正好这周在家时间就圆一下我的好奇心去搞一搞docker，顺便把我的服务器利用一下做一个MySQL在上面，后期会上点项目上去；**

#### 以后服务器就安装一个Docker就行了
1. 本次采用的是Centos7.5版本的Linux系统，镜像是[阿里云](https://opsx.alibaba.com/)下载的DVD版本；（因为Docker需要内核3版本以上的Linux）；
![内核.png](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/1.png)
2. 执行一键安装脚本
<!--more-->
![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/2.png)
```
curl -sSL https://get.docker.com/ | sh //这是官方的
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh - // 这是阿里云的
curl -sSL https://get.daocloud.io/docker | sh // 这是DaoCloud的 （实测这个最快）
或者采用离线安装指定版本:下载rpm安装包
https://download.docker.com/linux/centos/7/x86_64/stable/Packages/ 
然后把安装包上传至服务器执行yum install 安装包文件地址 
卸载干净docker 
 yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
```
官网方法安装
# 删除所有旧的数据
sudo rm -rf /var/lib/docker
#  安装依赖包
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
# 添加源，使用了阿里云镜像
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 配置缓存
sudo yum makecache fast
# 安装最新稳定版本的 docker
sudo yum install -y docker-ce
# 配置镜像加速器 阿里云自己的
# 启动docker引擎并设置开机启动
sudo systemctl start docker
sudo systemctl enable docker
```
3. 配置镜像加速器，不然下载速度会很慢;(采用阿里云提供的)
[个人自定义地址](https://cr.console.aliyun.com)
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/3.png"/>
4. 设置自启动
> 由于需要MySQL跟随linux自启动，MySql是依赖于docker启动，所以这里先设置docker自启动
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/4.png"  />
```
service docker start //启动docker（新安装默认启动了）
chkconfig docker on  //设置自动启动
```
5. 把MySQL的docker镜像下载到本地（这里就可以看到加速地址的作用了）
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/5.png"  />
```
docker pull mysql //下载的最新版MySQL
docker pull mysql：版本号 //
```
6. 运行一个MySQL镜像 并设置自动启动
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/6.png"  />
```
docker run --name 运行名称 -p 3306:3306 -v
/usr/local/mysql/conf:/etc/mysql/conf.d -v 
/usr/local/mysql/log:/var/log/mysql -v 
/usr/local/mysql/data:/var/lib/mysql -e 
MYSQL_ROOT_PASSWORD=密码 -d --restart=always  mysql
-d 是后台运行  
--restart=always 是docker官方的给的自启动方案
-v （导入外部文件到MySQL启动比如配置文件）/usr/local/mysql/conf/my.cnf:/etc/my.cnf
-e 映射MySQL 文件到本地 
```
7. 这个时候差不多就可以测试是否可以连接成功了
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/7.png" />
如果出现这样，下载最新版数据库连接工具 [Navicat](https://www.navicat.com/cht/download/navicat-premium)连接即可，我们就不去修改加密方式了；东西总得用新的嘛。
[Navicat最新破解教程](http://yakax.github.io/2018/11/05/破解Navicat最新版本12.1.9+/)
8. 如果遇到权限问题
```
docker exec -it mysql bash // 进入MySQL docker
mysql -u root -p  // 这个之后会叫你输入密码
mysql> SET GLOBAL innodb_fast_shutdown = 1;// 在MySQL 里面输入这个；然后ctrl+D 退出来
mysql_upgrade -u root -p   执行自动更新权限
```
