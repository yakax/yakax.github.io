# Vagrant基本命令



[镜像地址](https://app.vagrantup.com/boxes/search)
<!--more-->
#### 基本命令
```
vagrant init        # 初始化
vagrant up          # 启动虚拟机
vagrant halt        # 关闭虚拟机
vagrant reload      # 重启虚拟机
vagrant ssh         # SSH 至虚拟机
vagrant status      # 查看虚拟机运行状态
vagrant destroy     # 销毁当前虚拟机

vagrant suspend         # 挂起当前虚拟机
vagrant resume          # 恢复被挂起的vm
vagrant box list        # 列出所有box列表

vagrant box remove {base name}  # 删除
vagrant box add {title} {url}  # 添加镜像 

vagrant destroy         # 停止当前正在运行的虚拟机并销毁所有创建的资源
vagrant package         # 把当前的运行的虚拟机环境进行打包，可用于分发开发环境
vagrant plugin          # 安装卸载插件
vagrant provision       # 设置基本的环境，进一步设置可以使用Chef/Puppet进行搭建
vagrant ssh-config      # 输出ssh连接的一些信息
vagrant status          # 获取虚拟机状态
vagrant version         # 获取vagrant的版本
```









