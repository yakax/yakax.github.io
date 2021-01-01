# MySQL性能调优(5)Innodb引擎的MVCC


#### MVCC 
<font color=red>Multiversion concurrency control (多版本并发控制)
并发访问(读或写)数据库时，对正在事务内处理的数据做多版本的管理。以达到用来避免写操作的堵塞，从而引发读操作的并发问题。</font>
这里看一个案例
```
begin
// 这里默认是加上X锁的，在未做commit/rollback操作之前
update users set lastUpdate=now() where id =1; 
// 在其他的事务我们能不能进行对应数据的查询 但是我这时查询是能查到的当前时间这条记录
select * from users where id = 1;
```
先来看看MVCC的处理机制
<!--more-->
##### 插入数据
<font color=red>同一个事务，事务行版本号相同</font>
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/1.png" />
##### 删除数据
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/2.png"/>
##### 修改数据
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/3.png"/>
##### 查询数据
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/4.png" />
<font color=red>这也就是之前案例为什么能查询数据的原因所在</font>
```
记住图上的规则
1. 查找数据行版本号早于当前事务版本数据行
// 行的系统版本号小于或等于事务的系统版本号,这里就已经筛选出id等于1的两条数据
2. 查找删除版本号要么为null要么大于当前事务版本号记录
//为null说明没删除,大于当前事务版本号说明是事务开启过后 别的事务开启做的数据,这个是为了再次判断当前事务更改数据
```
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/5.png"  />

那这些数据时怎么来的呢
#### Undo Log
- undo意为取消，以撤销操作为目的，返回指定某个状态的操作
- <font color=red>UndoLog是为了实现事务的原子性而出现的产物</font>
- Undo Log实现事务原子性：事务处理过程中如果出现了错误或者用户执行了 ROLLBACK语句,MySQL可以利用Undo Log中的备份将数据恢复到事务开始之前的状态。
- <font color=red>UndoLog在MySQL innodb存储引擎中用来实现多版本并发控制</font>
- Undo log实现多版本并发控制：事务未提交之前，Undo保存了未提交之前的版本数据，Undo 中的数据可作为数据旧版本快照供其他并发事务进行快照读

##### 执行流程
<font color=red>这就是为什么加了X锁的数据我们还能查询的原因，他实际上是读取undo log里面的数据进行快照读</font>
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/6.png"  />
##### 当前读与快照读的区别
<font color=red>Innodb引擎的普通select都是快照读。只有增删改和加了锁的才是当前读</font>
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/7.png" />
#### Redo Log
- Redo，顾名思义就是重做。以恢复操作为目的，重现操作；
- Redo log指事务中操作的任何数据,将最新的数据备份到一个地方 (Redo Log)
- Redo log的持久：不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo 中。具体的落盘策略可以进行配置
- <font color=red>RedoLog是为了实现事务的持久性而出现的产物</font>
- Redo Log实现事务持久性：防止在发生故障的时间点，尚有脏页未写入磁盘，在重启MySQL服务的时候，根据redo log进行重做，从而达到事务的未入磁盘数据进行持久化这一特性。

##### 执行流程
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/8.png" />
##### Redo Log补充说明
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/9.png" />
这里可以看到redo log 默认大小是48M默认位置是在数据库根路径下面

```
指定Redo log 记录在{datadir}/ib_logfile1&ib_logfile2 可通过innodb_log_group_home_dir 配置指定目录存储
一旦事务成功提交且数据持久化落盘之后，此时Redo log中的对应事务数据记录就失去了意义，所以Redo log的写入是日志文件循环写入的
指定Redo log日志文件组中的数量 innodb_log_files_in_group 默认为2
指定Redo log每一个日志文件最大存储量innodb_log_file_size 默认48M
指定Redo log在cache/buffer中的buffer池大小innodb_log_buffer_size 默认16M
Redo buffer 持久化Redo log的策略， innodb_flush_log_at_trx_commit：
取值 0 每秒提交 Redo buffer --> Redo log OS cache -->flush cache to disk[可能丢失一秒内的事务数据]
取值 1 默认值，每次事务提交执行Redo buffer --> Redo log OS cache -->flush cache to disk[最安全，性能最差的方式]
取值 2 每次事务提交执行Redo buffer --> Redo log OS cache 再每一秒执行 ->flush cache to disk操作
```
#### 配置优化
##### 改配置说明
```
如果是docker可以通过启动选择配置文件启动
如果不想重启的话可以利用 set global
set global 参数=? 
```
##### 寻找配置文件位置和加载顺序
```
mysql --help | grep -A 1 'Default options are read from the following files in the given order'
```
这是我docker中MySQL的配置文件位置和加载顺序
<img src="https://yakax.oss-cn-hangzhou.aliyuncs.com/blog/mysql/10.png" />
##### MySQL服务器参数类型
```
全局参数
       set global autocommit = ON/OFF;
会话参数(<font color=red >会话参数不单独设置则会采用全局参数</font>)
       set session autocommit = ON/OFF;
注意：
全局参数的设定对于已经存在的会话无法生效
会话参数的设定随着会话的销毁而失效
全局类的统一配置建议配置在默认配置文件中，否则重启服务会导致配置失效
```
##### 全局配置文件配置
```
最大连接数配置（如果连接数过大会有约束，就需要考虑改句柄数了）
max_connections 
系统句柄数配置
/etc/security/limits.conf
ulimit -a
MySQL句柄数配置
/usr/lib/systemd/system/mysqld.service
```
##### 常用配置说明
```
port = 3306 #端口号
socket = /tmp/mysql.sock  #为MySQL客户程序与服务器之间的本地通信指定一个套接字文件
basedir = /usr/local/mysql #安装 MySQL 的安装路径
datadir = /data/mysql #数据库文件路径
pid-file = /data/mysql/mysql.pid #记录当前MySQL进程的pid
user = mysql  #表示MySQL的管理用户
bind-address = 0.0.0.0 #MySQL服务器的IP地址。如果MySQL服务器所在的计算机有多个IP地址，这个选项将非常重要
max_connections=2000 #MySQL服务器同时处理的数据库连接的最大数量
lower_case_table_names = 0 #表名区分大小写
server-id = 1 #数据库id号一般主从复制的时候会用到
tmp_table_size=16M #临时表大小
transaction_isolation = REPEATABLE-READ #隔离级别
ready_only=1 #0是读写1是只读
```
##### 其他参数配置
```
wait_timeout #服务器关闭非交互连接之前等待活动的秒数
innodb_open_files #限制Innodb能打开的表的个数
innodb_write_io_threads #innodb使用后台线程处理innodb缓冲区数据页上的写 I/O(输入输出)请求数 
innodb_read_io_threads #innodb使用后台线程处理innodb缓冲区数据页上的读 I/O(输入输出)请求数 
innodb_lock_wait_timeout #InnoDB事务在被回滚之前可以等待一个锁定的超时秒数
```







