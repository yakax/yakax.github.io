# 学习docker常用命令


[docker的配置安装](https://yakax.github.io/2018/11/05/%E5%AD%A6%E4%B9%A0docker%E5%B9%B6%E5%AE%89%E8%A3%85%E4%B8%80%E4%B8%AA%E8%87%AA%E5%90%AF%E5%8A%A8MySQL8/)

#### 基本命令
```
docker info  docker基本信息
docker --help  帮助信息 比如docker search --help 这样就可以查看search语法
docker version  docker版本号
docker images  显示镜像
docker search 镜像名  搜索docker镜像
docker pull 镜像名  拉取镜像到本地
docker rmi 镜像id 删除镜像
```
<!--more-->
#### 容器命令
##### 启动容器命令
```
docker run 
-d  后台运行
-p  指定端口映射 宿主机端口:容器端口
-restart=always  是docker官方的给的自启动方案
--name 为容器指定一个名称
-it 通过伪终端交互方式运行
-P 随机端口映射
-v 启用数据卷 让外部文件在容器内运行
-e 把文件映射到宿主机
```
##### 镜像操作
```
docker ps -a 显示所有镜像包括停止了的
docker start (容器id或者名称) 启动镜像
docker stop 停止镜像
docker restart 重启镜像
docker rm 删除镜像 删除多个可以用空格隔开
docker kill 强制停止镜像
docker logs 查看日志 
docker logs -f --tail  容器id 可以动态查看
docker top 查看容器内进程
docker inspect 查看容器详细信息
docker exec -it 容器id bash  进入容器
docker cp  容器id:容器内路径:宿主机路径  拷贝文档
```

##### 创建镜像
```
docker commit -m='提交信息' -a='作者' 容器id 制作后的名称：标签
```

#### Dockerfile
通过dockerfile来构建镜像是最舒服最方便的
```
FROM java:8 指令指明了当前镜像的基镜像，编译当前镜像时自动下载基镜像
MAINTAINER bingo 作者
ADD demo-0.0.1-SNAPSHOT.jar demo.jar  复制jar文件到镜像中去并重命名为demo.jar
RUN yum -y install vim 构建时需要运行的命令
WORKDIR 容器创建后默认在那个目录
EXPOSE 8080  暴露8080端口
ENV JAVAHOME /usr/jvm/jdk1.8.0_181 设置环境变量
COPY 复制宿主机文件
VOLUME 数据卷
CMD 只执行最后一条 并且这个会被run 后面的命令替换
ENTRYPOINT ["java","-jar","demo.jar"]  启动时执行java -jar demo.jar 这个命令会追加到run命令后面
---------------------------------------------
然后运行 docker build 名称：标签 .  构建镜像
```
#### 容器间访问
**docker 容器访问是通过宿主机的NAT来转发的所以，docker容器内的ip是宿主机分配的B类私有地址**
容器间访问是通过172的私有地址访问，访问外网是通过宿主机转发访问。


