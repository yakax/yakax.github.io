# 初入学习git


#### git与svn的区别
1. 首先核心的点就是git是分布式的，而svn不是分布式的
2. git中每一个开发人员的电脑上都有一个Local Repository,所以即使没有网络也一样可以Commit，查看历史版本记录，创建项 目分支等操作，等网络再次连接上Push到Server端；这也就是为什么git适合分布式的原因。
3. svn相当于知识在本地拷贝了一份服务器上的代码，本地需要看日志记录是需要联网。
4. svn比较简单只是需要一个放代码的地方拿下来用就ok。这种属于集中式的代码管理。

<!--more-->
#### .git目录
HEAD: 当前指向的分支
config：配置目录
objects：git存储的文件对象，经过压缩编码
index：git的索引文件通过这个找到objects的文件

#### 创建git项目
```
不懂命令可以 git --help 你想查询的命令
比如 git --help push 就可以跳转浏览器帮助页面
```
1. 生成 ssh key 
```
ssh-keygen -t rsa -C '邮箱'
```
2. 提交的用户名与邮箱标识
```
// 这个是全局的
//windows下可以找到用户目录下.gitconfig这个文件更改
git config –-global user.name ‘xx’
git config –-global user.email ‘xx’
```
3. 开始项目初始化
```
git init //在项目文件下（会生成一个.git的目录）
git remote add origin git地址 // 这样会绑定远端地址
git pull origin master // 把远端的代码拉下来(这里的分支具体看公司的git flow)
git add . // 跟踪所有改动文件（可以指明具体文件）
git commit -m '提交说明' // 提交跟踪文件到本地工作目录
git push origin master //推动代码到远程分支
```

#### 我工作中常用命令
解决冲突（个人见解）
```
首先把握一点：就是把有冲突的代码pull下来与本地修改了然后再次commit 
我在开发中是拿我的分支与有冲突的分支 rebase head一下 
解决完代码后保存，再次提交commit 
git rabase -i HEAD~~ 删除我本地之前的版本（这里有多少次本地提交就有多少个~）
然后 git push -f 到远程分支上面去
再在远程仓库进行合并
```
切换分支
```
git checkout 分支名
git checkout .  // 可以撤销当前分支的更改
```

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/2018-11-5/14.png"/>



















