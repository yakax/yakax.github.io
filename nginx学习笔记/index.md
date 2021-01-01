# Nginx学习笔记


## 编译
相关依赖

```shell
yum update
yum install wget
wget http://nginx.org/download/nginx-1.19.1.tar.gz ## nginx 源码包
wget https://ftp.pcre.org/pub/pcre/pcre-8.44.tar.gz ## 正则会用到
wget https://zlib.net/zlib-1.2.11.tar.gz  ##gzip 模块
tar xf 指定文件解压
```

<!--more-->

```shell
./configure --help  查看编译帮助命令
```

```shell
 --help                             print this message

  --prefix=PATH                      nginx总相对路径 下面都可以 默认以相对路径创建
  --sbin-path=PATH                   set nginx binary pathname
  --modules-path=PATH                set modules path
  --conf-path=PATH                   set nginx.conf pathname
  --error-log-path=PATH              set error log pathname
  --pid-path=PATH                    set nginx.pid pathname
  --lock-path=PATH                   set nginx.lock pathname

  --user=USER                        nginx配置文件生效启动用户
  
  --with开头都是可以单独指定需要使用的模块

  --without 开头指定不需要的模块
	还有一些指定模块路径的参数

```

我使用的配置参数

```
./configure  --prefix=/usr/local/nginx \
--with-pcre=/usr/local/nginx_rely/rely/pcre-8.44 \
--with-zlib=/usr/local/nginx_rely/rely/zlib-1.2.11  \
--with-http_ssl_module  \
--with-http_image_filter_module \
--with-http_stub_status_module
```

期间根据环境可能需要安装下面三个东西

```
yum -y install gcc gcc-c++ openssl openssl-devel gd gd-devel
```

出现这样 差不多就检测成功了

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200809201537.png" style="zoom:50%;" />

```
make 编译 生成Makefile
make install
```

去安装目录启动

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200809202041.png)

设置软连接

```
ln -s /usr/local/nginx/sbin/nginx /usr/bin/nginx
nginx -h 查看具体命令
注意：请在运行 nginx 时，使用绝对路径。因为热部署USR2是通过绝对路径寻找
```


## 虚拟主机配置与热部署

由于 nginx运行时 一个master 和多个 worker进程以线程的方式运行

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200809212450.png)

```
worker_processes 就代表worker进程数，我设置的是根据cpu核数。
user 代表配置文件启动生效用户 也可以加上用户组
worker_connections worker进程连接数 
```

> http://nginx.org/en/docs/ nginx 配置文件文档



### 基于多Ip的虚拟主机

```nginx
location /  ##说明映射的是本地nginx安装路径
root ## 路径下的文件夹
index ## 文件夹下的首页
```

前提主机是多网卡：不同ip对应不同路径 端口监听默认80

```nginx
server {
        listen       192.168.1.110; 
        server_name  localhost;

        location / {
            root   server1/html;
            index  index.html index.htm;
        }
   }
 server {
        listen       192.168.1.111; 
        server_name  localhost;

        location / {
            root   server2/html;
            index  index.html index.htm;
        }
   }
```

### 基于多端口的虚拟主机

```nginx
 server {
        listen       9011; 
        server_name  localhost;

        location / {
            root   server3/html;
            index  index.html index.htm;
        }
   }
```

### 基于域名的虚拟主机

```nginx
 server {
        listen       80; 
        server_name  域名; //匹配优先级 精确>左侧通配符>右侧通配符>正则

        location / {
            root   server4/html;
            index  index.html index.htm;
        }
   }
```

### 热部署

当从老版本替换为新版本的 nginx 的时候，如果不热部署的话，会需要取消 nginx 服务并重启服务才能替换成功，这样的话会使正在访问的用户在断开连接，所以为了在不影响用户的体验下进行版本升级，就需要**通过信号量**热部署来升级版本。

**注意一点：新的nginx编译后的 logs、sbin、conf这些目录名字不能变**

先备份nginx二进制文件 并用新的二进制nginx文件覆盖
**cp nginx nginx.bak**

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200810114446.png" style="zoom:50%;" />

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/1.png" style="zoom:50%;" />

启动新的nginx  这里说明一下 nginx 启动时必须以绝对路径启动，不然SIGUSR2无法找到nginx
**kill -s SIGUSR2 21598**

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200810121822.png" style="zoom:50%;" />

<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200810122043.png" style="zoom:50%;" />

现在 新旧进程 都并存，并且在logs下存在了一个旧进程的master 进程pid
这时 可以优雅退出worker进程，具体用户超时时间连接参数 worker shutdown timeout time 在配置文件可以配置
**kill -s SIGWINCH 21598**

![](https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/nginx/TIM%E6%88%AA%E5%9B%BE20200810122613.png)

此时**旧master** 还存在的情况下可以开始验证**新worker**进程的nginx 是否满足需求 如果满足需求可以正式让就master退出了

<font color=blue>**kill -s  SIGQUIT 21598**</font>  至此平滑升级就可以了;如果新版有不满足需求，未通过测试 在老版本的master未退出时可以**回滚**

**kill -s  SIGHUP 21598** 此时新旧进程也是并存的 

**kill -s SIGQUIT 21603** 退出新master进程

## main、events段常用参数



```markdown
# main段核心参数：
### user USERNAME [GROUP]
	解释：指定运行nginx的worker子进程的属主和属组，其中属组可以不指定
	示例：
		user nginx nginx;

### pid DIR
	解释：指定运行nginx的master主进程的pid文件存放路径
	示例：
		pid /usr/local/nginx/logs/nginx.pid;

### worker_rlimit_nofile number
	解释：指定worker子进程可以打开的最大文件句柄数
	示例：
		worker_rlimit_nofile 20480; 
		备注：linux 最大打开65535个句柄 一般每个worker*cpu核心数=总数-10000

### worker_rlimit_core size
	解释：指定worker子进程异常终止后的core文件，用于记录分析问题
	示例：
		worker rlimit core 50M;
		working_directory /usr/local/nginx/tmp; 
		备注：当前worker所属用户需要有这个文件夹的写权限

### worker_processes number|auto
	解释：指定nginx启动的worker子进程数量
	示例：
	worker processes 4; worker_processes auto;

### worker_cpul_affinity cpumask1 cpumask2
	解释：将每个worker子进程与我们的CPU物理核心绑定
	示例：
	worker_cpu_affinity 0001 0010 0100 1000; #4个物理核心，4个worker子进程

	worker cpu affinity 
	00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
	#8物理核心，8个worker子进程

	worker_cpu_affinity 01 10 01 10; #2个物理核心，4个子进程
    备注：将每个worker子进程与特定CPU物理核心绑定，优势在于：避免同一个worker子进程
	在不同的CPU核心上切换，缓存失效，降低性能；其并不能真正的避免进程切换,
	主要还是cpu时间片切换开销大，又由于是多核，并不能充分使用cpu缓存，所以让nginx的worker进程绑定上固定的核心cpu 来切换。

### worker_priority number
	解释：指定worker子进程的nice值，以调整运行nginx的worker子进程的优先级，通常设定为负值，以优先调用nginx
	示例：
	worker_priority -10;
	备注：Linux默认进程的优先级值是120,值越小越优先；nice设定范围为-20到+19 如-20+100

### worker_shutdown_timeout time
	解释：指定worker子进程优雅退出时的超时时间(热部署时,nginx 切换pid进程时发送信号量关闭旧链接请求的强制条件)
	示例：
	worker_shutdown_timeout 5s;

### timer_resolution time
	解释：worker子进程内部使用的计时器精度，调整时间间隔越大，系统调用越少，有利于性能提升；反之，系统调用越多，性能下降，主要看系统对时间精度要求  
	示例：
	worker_resolution 100ms;

### daemon on|off
	解释：设定nginx的运行方式，前台还是后台，前台用户调试，后台用于生产
	示例：
	daemon off; 前台  默认on

```

```markdown
# events段核心参数
### use 
	解释:nginx使用何种事件驱动模型
	method可选值：select、poll、kqueue、epoll、/dev/poll、eventpor
	默认配置：无 推荐配置：不指定，让nginx自己选择

### worker_connections 
	解释：worker子进程能够处理的最大并发连接数
	默认配置：worker_connections 1024 推荐配置：worker_connections 65535/worker_processes|65535

### accept_mutex 
	解释:是否打开负载均衡互斥锁 （就是在worker子进程都加了个互斥锁，新请求进来只给其中一个处理）
	可选值：on、off
	默认配置：accept_mutex off 推荐配置：accept_mutext on

### accept_mutex_delay 200ms
	解释：新连接分配给worker子进程的超时时间  当accept_mutex参数打开才有意义

### lock file 
	解释：负载均衡互斥锁文件存放路径
	默认配置：lock_file logs/nginx.lock 推荐配置：lock_file logs/nginx.lock

### muti_accept on|off
	解释:子进程一次性可以接收的新连接个数 默认off
```

## location学习与sub_status 模块
root与alias区别**

```json
/usr/local/nginx  这个路径是nginx主路径
server_name 匹对路径地址
location 后面是匹对监听端口后的路径地址
location /image {
		root /usr/local/nginx/image
}
客户端请求www.test.com/image/1.jpg,
则对应磁盘映射路径/usr/local/nginx/image/image/1.jpg
location /image {
		root image 相对地址
}
则对应磁盘映射路径/usr/local/nginx/image/image/1.jpg

location /picture{
		alias /opt/nginx/html/picture; 
}
客户端请求www.test.com/picture/1.jpg,则对应磁盘映射
则对应磁盘映射路径/opt/nginx/html/picture/1.jpg
```

**alias只能放在 location中**

**root能用在 http server location if中 类似全局参数一样**



### location 匹配规则

| 规则 | 含义 (优先级从高到低) | 示例                                      |
| ---- | --------------------- | ----------------------------------------- |
| =    | 精确匹配              | location = /images/ {...}                 |
| ^~   | 匹配到即停止搜索      | location  ^~ /images/ {...}               |
| ~    | 正则匹配 区分大小写   | location  ~  \. (jpg l gif)$ { ... }     |
| ~*   | 正则匹配 不区分大小写 | loaction  ~*\.(jpg l gif)$ {...} 这个用得少 |
|      | 不带任何符号          | location  / {..}                          |

结尾反斜线区别

```
/test 与 /test/
/test 先找目录  找不到 会尝试找test文件
/test/ 只会当做文件目录
```

### sub_status 模块

```
location /uri{
		stub_status; 
}
也可以写在server层级
```

| 状态项             | 含义                     |
| ------------------ | ------------------------ |
| Active Connections | 活跃的连接数量           |
| accepts            | 接受的客户端连接总数量   |
| handled            | 处理的客户端连接总数量   |
| requests           | 客户端总的请求数量       |
| Reading            | 读取客户端的连接数       |
| Writing            | 响应数据到客户端的连接数 |
| Waiting            | 空闲客户端请求连接数量   |

## 请求、链接、权限模块
### 请求与连接区别

```
connection是连接:即常说的tcp连接，三次握手，状态机
request是请求:例如http请求，无状态的协议
request是必须建立在connection之上
```

#### connection限制客户端并发连接数

Nginx默认是有这个模块，可以通过`--without-http_limit_conn_module` 禁用

由于限制客户端需要每个worker子进程对客户端有标识，所以需要用到共享内存。

##### 常用指令

- limit_conn_zone

  ```
  上下文 http
  写法 limit_conn_zone $binary_remote_addr zone=addr:10m
  通常1m共享内存空间大小都能维护3万多并发连接 10m已经很大了 addr是名称
  binary_remote_addr 相比 remote_addr 空间使用更少 前者4 后面7-8字节
  ```

  

- limit_conn_status

  ```
  上下文：http、server、location
  limit_conn_status code 
  默认状态码 limit_conn_status 503  超过连接数后返回的状态码
  ```

  

- limit_conn_log_level

  ```
  上下文：http、server、location
  limit_conn_ log_level info|notice|warn|error
  默认error 当限速行为发生时会产生日志
  ```

  

- limit_conn

  ```
  limit_conn 上面的名称 限速多少个客户端并发连接
  ```

  限制的只是并发连接才会出现。

### request限制客户端请求平均速率

**写法跟conn一样**

Nginx默认是有这个模块，可以通过`--without-http_limit_req_module` 禁用

由于限制客户端需要每个worker子进程对客户端有标识，所以需要用到共享内存。

使用 限流算法是leaky_bucket算法，漏桶算法，使流量平滑进入

```
limit_req_zone $binary_remote_addr zone=addr:10m rate=2r/m
一分钟处理两个请求
limit_req_log_level 限速日志级别
limit_conn_status 状态码
limit_req zone=addr burst=7 nodelay;
设置了 rate 可以不设置burst=7 nodelay 连续请求
```

### 限定指定ip 访问

```
allow 允许
deny 拒绝  会直接返回403 forbidden
http、server、location、limit_except 上下文 
可以组合编写,从上到下范围筛选 
location/{
	deny 192.168.1.1;
	allow 192.168.1.0/24;
	allow 10.1.1.0/16;
	allow 2001:0db8::/32;
	deny all;
}
```



### 限制特定用户访问auth_basic模块

默认已经有了的模块：可以通过`--without-http_auth_basic_module`禁用

```
语法：auth_basic 浏览器提示语|off;
默认值：auth_basic off;
上下文：http、server、location、limit_except

语法：auth_basic_user_file 配置的密钥文件绝对地址;
上下文：http、server、location、limit_except
```

制作用户名密码

```
可执行程序：htpasswd
所属软件包：httpd-tools
生成新的密码文件：htpasswd -bc encrypt_pass jack 123456
添加新用户密码：htpasswd -b encrypt_pass mike 123456
```

### 给予HTTP响应码做权限

需要增加模块`--with-http_auth_request_module`

```
语法：auth_request uri|off;
默认值：auth_request off;
上下文：http、server、location

location /private/{
	auth_request /auth;
}
location /auth {
	//授权服务器，根据返回状态码做后续操作 成功状态200
	proxy_pass http://127.0.0.1:8080/verify;
	// 带上请求信息可以
	proxy_pass_request_body off;
	proxy_set_header Content-Length "";
	proxy_set_header X-Original-URI $request_uri;
}
```

 ## tcp与http相关变量学习
 ### TCP相关变量

| 变量名              | 含义                                                  |
| :------------------ | ----------------------------------------------------- |
| remote_addr         | 客户端IP地址                                          |
| remote_port         | 客户端端口                                            |
| server addr         | 服务端IP地址                                          |
| server_port         | 服务端端口                                            |
| server protocol     | 服务端协议                                            |
| binary_remote_addr  | 二进制格式的客户端IP地址 固定4字节                    |
| connection          | TCP连接的序号，递增                                   |
| connection_request  | TCP连接当前的请求数量                                 |
| proxy_protocol_addr | 若使用了proxy_protocol协议则返回协议中地址 否则返回空 |
| proxy_protocol_port | 若使用了proxy_protocol协议则返回协议中端口 否则返回空 |

### HTTP 请求过程中相关变量

| 变量名         | 含义                                        |
| -------------- | ------------------------------------------- |
| uri            | 请求的URL,不包含参数                        |
| request_uri    | 请求的URL,包含参数                          |
| scheme         | 协议名，http或https                         |
| request_method | 请求方法                                    |
| request_length | 全部请求的长度，包括请求行、请求头、请求体  |
| args           | 全部参数字符串                              |
| arg_参数名     | 特定参数值                                  |
| is_args        | URL中有参数，则返回？否则返回空             |
| query_string   | 与args相同                                  |
| remote_user    | 由HTTP Basic Authentication协议传入的用户名 |

`特殊变量`

| host                 | 先看请求行，再看请求头，最后找server_name    |
| -------------------- | -------------------------------------------- |
| http_user_agent      | 用户浏览器                                   |
| http_referer         | 从哪些链接过来的请求                         |
| http_via             | 经过一层代理服务器，添加对应代理服务器的信息 |
| http_x_forwarded_for | 获取用户真实IP                               |
| http_cookie          | 用户cookie                                   |



### 处理http请求中的变量

| 变量名             | 含义                                  |
| ------------------ | ------------------------------------- |
| request_time       | 处理请求已耗费的时间                  |
| request_completion | 请求处理完成返回OK,否则返回空         |
| server_name        | 匹配上请求的server_name值             |
| https              | 若开启https,则返回on,否则返回空       |
| request_filename   | 磁盘文件系统待访问文件的完整路径      |
| document_root      | 由URI和root/alias规则生成的文件夹路径 |
| realpath_root      | 将document_root中的软链接换成真实路径 |
| limit_rate         | 返回响应时的速度上限值                |


## rewrite模块中的return指令
```
### return指令

`return 执行后 后续的执行是不会执行的`

```
语法：return code [text];
	 return code URL;
	 return URL;
上下文：server、location、if
1XX:消息类
2XX:成功
3XX:重定向
4XX:客户端错误
5XX:服务器错误
```

### rewrite指令

```
语法：rewrite regex replacement [flag];
上下文：server、location、if
示例：rewrite /images/(.*\.jpg)$/pic/$1;
last：这是讲images下面.jpg的文件 都重定到pic路径去查看对应的location 
break ：这是讲images下面.jpg的文件 都重定到pic路径去查看对应的文件路径
```

**flag说明**

- last 重写后的URL发起新请求，再此进入server段，尝试location中的匹配
- break 直接使用重写后的URL,不再匹配其他location中语句
- redirect 返回302临时重定向
- permanent 返回301永久重定向

示例

​```nginx
location /search {
	rewrite ^/(.*)http://www.cctv.com permanent;
location /images {
	rewrite /images/(.*) /pics/$1 break;
location /pics {
	rewrite /pics/(.*)/photos/$1 ;
location /photos {
}
```



### if语句

```
$variable 仅为变量时，值为空或以0开头字符串都会被当做false处理 类似布尔值
=或！= 相等或不等比较
~或！~正则匹配或非正则匹配
~*正则匹配，不区分大小写
-f或！-f检查文件存在或不存在
-d或！-d检查目录存在或不存在
-e或！-e检查文件、目录、符号链接等存在或不存在
-x或！-X检查文件可执行或不可执行
```

示例

```nginx
location /search/ {
if ($remote_addr = "192.168.184.1" ) {
	return 200 "test if OK in URL /search/";
}
location / {
	if ($uri = "/images /") {
	rewrite (.*) /pics/ break;
}
	return 200"test if failed
}
```

### autoindex

```
autoindex on|off 打开关闭 默认off
autoindex_exact_size on|off 文件大小是否精确到字节 默认on
autoindex_format json|html|xml|jsonp  默认html
autoindex_localtime 文件时间格式 默认off
```

示例

```nginx
location /download/{
	root /opt/source;
	index a.html;
	autoindex on;
	autoindex exact size on;
	autoindex format html;
	autoindex_localtime off;
}
访问 /opt/source/download/下面的文件
如果存在a.html直接访问这个页面 
```

## 反向代理服务器

- 反向代理服务器介于用户和真实服务器之间，提供请求和响应的中转服务
- 对于用户而言，访问反向代理服务器就是访问真实服务器
- 反向代理可以有效降低服务器的负载消耗，提升效率

### upstream模块

> 定义上游服务器的连接信息，请求控制之类的基本信息。

| 指令                | 含议                                      |
| ------------------- | ----------------------------------------- |
| upstream            | 段名，以{开始, }结束，中间定义上游服务URL |
| server              | 定义上游服务地址                          |
| zone                | 定义共享内存，用于跨worker子进程          |
| keepalive           | 对上游服务启用长连接                      |
| keepalive_ requests | 一个长连接最多请求个数                    |
| keepalive_timeout   | 空闲情形下,一个长连接的超时时长           |
| hash                | 哈希负载均衡算法                          |
| ip_hash             | 依据IP进行哈希计算的负载均衡算法          |
| least_conn          | 最少连接数负载均衡算法                    |
| least_time          | 最短响应时间负载均衡算法                  |
| random              | 随机负载均衡算法                          |

#### 上游服务器server 配置参数

| weight=number      | 权重值，默认为1                     |
| ------------------ | ----------------------------------- |
| max_conns= number  | 上游服务器的最大并发连接数          |
| fail_timeout= time | 服务器不可用的判定时间              |
| max_fails= number  | 服务器不可用的检查次数              |
| backup             | 备份服务器仅当其他服务 器都不可用时 |
| down               | 标记服务器长期不可用,离线维护       |

默认已被编译进Nginx 禁用须通过`--without-http_upstream_module`

**配置示例**

> 就是说在10秒内有两次请求失败 就会认为服务器不可用

```nginx
upstream 名称{
	server 代理地址 weight=3 max_conns= 1000 fail_timeout=10s max_fails=2;
		keepalive 32; # 上游空闲长连接最大数量
		keepalive_requests 50; # 单个长连接可以处理的最多http请求个数
		keepalive_timeout 30s; # 空闲长连接超过这个时间没请求就销毁
}
upstream 与 server一个层级
```

#### 负载均衡配置示例

> nginx默认是轮询server

```nginx
upstream 名称{
	server 代理地址;
    server 代理地址;
    #------不考虑上游服务器处理能力来负载
    hash $变量名;# 这个就是单纯客户端配置以什么算hash值来负载 
    ip_hash; #单纯以ip算hash值
    #------考虑上游服务器能力 怎么维护服务器的信息,是多个worker子进程通过共享内存
    zone 内存名称 10M; # 通过共享内存维护上游服务器信息
    least_conn; #挑选服务器处理连接数最少的，如果都一样 退化轮询；
    least_time; #最短响应时间
}
```

#### 当上游服务器错误

> 当 upstream上游服务器出错后请求转发到 proxy_next_upstream

```nginx
语法:
proxy_next_upstream error| timeout|
invalid_header|http_500| http_502|http_503|http_504|http_403| http_404| http_429|no_idempotent|off
# no_idempotent: 对非幂等请求失败是否需要转发 加上就会重试转发
默认值: proxy_next_upstream error timeout ;
上下文:http、server、 location

需要配置 proxy_next_upstream_timeout 等待时间  proxy_next_upstream_tries 重试转发次数 
两个参数 默认都是0

proxy_intercept_errors on|off #上游返回响应码大于300 是直接响应还是按照error_page处理
默认on打开会使用error_page 关闭会直接返回上游信息
需要增加配置 error_page 503 /503.html  
也可以error_page 503 =200 /503.html 这样用户看到的就是200状态码
```



#### proxy_pass指令

默认已被编译进Nginx 禁用须通过`--without-http_proxy_module`

```
语法: proxy_ pass URL  url必须http或者https开头
上下文: location、 if、limit_except
```

```nginx
upsteam back_end {
	server 192.168.184.20:8080 weight=2 max_conns=1000 fail_timeout=10s max_fails=3;
		keepalive 32;
		keepalive requests 80;
		keepalive timeout 20s;
}
server{
    listen 80;
	server_name proxy.kutian.edu;
	location /proxy/ {
	proxy_pass http://back_end/proxy;
}
## proxy_pass http://back_end/proxy/ 
## 尾部带反斜线nginx会更改url删除location后面地址 
## 不带反斜线会原封不动请求到上游服务器
```



#### 客户端请求代理服务器请求信息

> 客户端超过这些规则 会返回413

```nginx
location /proxy/ {
	proxy_pass http://back_end/proxy; #上游服务器
    proxy_request_buffering on; # 打开缓冲默认打开。请求接收完在转发：nginx处理比较快，上有服务器处理比较低 利用缓存来提升吞吐量
	client_max_body_size 250k; #最大body大小 
	client_body_buffer_size 100k;#客户端缓冲大小
	client_body_temp_path test_body_path;#当大于缓冲大小 小于body大小 会把请求持久化到磁盘的位置
	client_body_in_file_only on; #不管请求体多大 都会把请求持久化到磁盘 默认off处理完成会删除
	client_body_in_single_buffer on;#连续存储磁盘空间 默认off不连续
	client_body_timeout 30; #链接超时时间
}
```



#### 代理更改请求上游服务器

```nginx
location /proxy/ {
	proxy_pass http://back_end/proxy; #上游服务器
	proxy_method PUT; #请求方式修改
	proxy_http_version 1.1;# http版本  要支持长连接 必须是1.1 并且要与上游服务器使用长连接还需要设置header
	proxy_pass_request_headers off; #默认打开 全部转发上游服务器 
    proxy_pass_request_body off; #默认打开 
	proxy_set_body "body信息"; #  上面关闭不影响
    proxy_set_header test "header参数信息"; # 向头部插入信息 上面关闭不影响
}
注意 proxy_set_header 默认重定义两个 Header 头字段，
proxy_set_header Host $proxy_host;
proxy_set_header Connection close;

Host 初始值 $proxy_host，这是因为 HTTP/1.1 必须包含 Host 字段以指定主机；至于 $proxy_host 跟 $host 的区别，前者是 backend 即后端的主机名，后者是 frontend 即自身的主机名。该字段要不要改成 $host 或 $http_host，视后端会不会校验域名和端口而定

Connection 初始值 close，也是因为在 HTTP/1.1 中所有连接都是长连接 (keep-alive)，除非声明 close 表示不需要，而后端的 Web 应用程序 (HTTP/1.0 或更古早的) 未必支持连接复用，所以统统设成 close 以免出错。跟后端通讯真需要用到 keep-alive，先配置一组 upstream (上游服务器)，回头再清空 Connection 字段即可。即 proxy_set_header Connection "";
```

**连接指令**

```nginx
proxy_connect_timeout 60s; # 连接超时时间，比如3次握手   适用于http、server、location
proxy_socket_keepalive on|off;#默认off;通过socket 替换长连接 适用于http、server、location
proxy_send_timeout time;#默认值60s; 发送超时时间的超时时间 适用于http、server、location
proxy_ignore_client_bort on|off; #是否忽略客户端请求退出，on表示如果客户端断开连接，nginx到上游服务器也会断开连接；off表示如果客户端断开连接，nginx到上游服务器会到请求响应结束后再断开 默认关闭
```

## 缓存
#### 缓存文件-以便启动恢复

```nginx
语法：proxy_cache_path path keys_zone=name:size #文件路径与缓存名称大小
默认值：proxy_cache_path off;
上下文：http
```

| 可选参数          | 含义                                            |
| ----------------- | ----------------------------------------------- |
| level             | path的目录层级                                  |
| use_ temp_path    | off直接使用path路径; on使用proxy_temp_path路径  |
| inactive          | 在指定时间内没有被访问缓存会被清理;默认10分钟   |
| max_size          | 设定最大的缓存文件大小,超过将由CM清理           |
| manager_files     | CM清理一次缓存文件，最大清理文件数;默认100      |
| manager_sleep     | CM清理一次后进程的休眠时间;默认200毫秒          |
| manager_threshold | CM清理一 次最长耗时;默认50毫秒                  |
| loader_files      | CL载入文件到共享内存，每批最多文件数;默认100    |
| loader_sleep      | CL加载缓存文件到内存后,进程休眠时间;默认200毫秒 |
| loader_threshold  | CL每次载入文件到共享内存的最大耗时;默认50毫秒   |

> CM:处理进程

#### 缓存key设置


```nginx
语法：proxy_cache_key string;
默认值：proxy_cache_key $scheme $proxy_host$request_uri ; # http://host/uri
上下文：http、server、location

语法:proxy_cache_valid [code]time; # 对指定状态码进行缓存
上下文: http、 server、location
默认配置示例: proxy_cache_valid 60m; #只对200、301、 302响应码缓存

```

#### 缓存状态 upstream_ cache_status

```
MISS :未命中缓存
HIT :命中缓存
EXPIRED :缓存过期
STALE:命中了陈|旧缓存;
REVALIDDATED : Nginx验证陈旧缓存依然有效
UPDATING :内容陈旧，但正在更新
BYPASS :响应从原始服务器获取
```

配置示例

```nginx
proxy_cache_path /opt/nginx/cache_temp levels=2:2 keys_zone=cache_zone:30m max_size=32g inactive=60m use_temp_path=off;
upstream cache_server {
	server 192.168.184.20:1010;
	server 192.168.184.20: 1011;
}
server {
	listen 80;
	server_name cache.kutian.edu;
    location / {
	proxy_cache cache_zone ;
	proxy_cache_valid 200 5m;
    proxy_cache_key $scheme$proxy_host$request_uri;
	add_header Nginx-Cache-Status "$upstream_cache_status"; # 显示缓存状态
	proxy_pass http://cache_server;
	}
}
# 上游服务器可以在响应时修改header X-Accel-Expires 定义缓存时间。失效 状态会变为 EXPIRED
```

#### 对于 特定key 不缓存

```nginx
proxy_no_cache
	#表明用户访问login和search两个url的时候，变量$nocache 设置值
    if ($request_uri ~ ^/(login|search)){
        set $nocache 1;
    }
        location / {
         #当变量$nocache 有值，不缓存。
         proxy_no_cache $nocache;
    }
proxy_cache_bypass 
#该指令的每个参数都指定了一个条件，只有请求满足其中的任何一个条件，并且参数的值不是0，则Nginx会把请求转发到后端的服务而不会使用缓存。
```



#### 缓存大量失效解决

##### 限制请求

```nginx
proxy_cache_lock on|off 默认off  #限制相同路径同时请求 单个请求返回响应，后续相同路径才能再次请求
proxy_cache_lock_timeout time  默认5s  # 超时时间后 剩余请求同时请求
proxy_cache_lock_age time  默认5s #限制超时未返回 一个一个请求
```

##### 启用陈旧缓存

```nginx
proxy_cache_use_stale error|timeout|invalid_header|updating|http_500|http_502|http_503|http_504|http_403|http_404|off
默认值: proxy_cache_use_stale off ;
proxy_cache_background_update on|off 默认off  # 由nginx请求更新缓存，让客户端先用旧缓存
```

| 可选参数       | 含义                                       |
| -------------- | ------------------------------------------ |
| error          | 与上游建立连接、发送请求、读取响应头出错时 |
| timeout        | 与上游建立连接、发送请求、读取响应头超时时 |
| invalid_header | 无效头部时                                 |
| updating       | 缓存过期,正在更新时                        |
| http_状态码    |                                            |



#### 缓存清除操作

> 第三方nginx缓存清除模块。使用可以加--add-module ngx_cache_purge文件路径编译新版 

```nginx
location ~ /cache_purge(/.*) {
	proxy_cache_purge cache_zone $host$1;
}
# cache_zone 对应缓存 keys_zone 这个的名称
# $host$1; 对应缓存key
```

