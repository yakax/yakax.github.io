# MySQL-事务学习



## 多事务并发处理数据产生的问题

**脏读脏写 都是因为读取了事务未提交时的事务回滚情况**

#### 脏写

事务A与事务B同时更新一条数据，由于其中一个事务回滚 导致另外事务执行成功的数据不存在了。

#### 脏读

事务A还没提交事务时事务B拿到值做了大量的操作，事务A回滚后出现的数据异常问题。

#### 不可重复读 与可重复读

事务A修改了值并且提交了事务    B读取到   然后B在执行事务期间 C又修改了值并且提交了。这时B读取到的值又变了。 这就时不可重复读问题。 可重复读就相反 并且读取到的值是相同的。

#### 幻读

其实就是在事务执行期间第二次查询相比第一次查询出现了之前没出现过的数据。

<!--more-->

### 事务的4种隔离级别

- **读未提交RU**  可以读未提交 但是不允许脏写 **READ UNCOMMITTED**
- **读已提交RC**  人家事务没提交的情况下修改的值，你是绝对读不到的  不会发生脏读脏写情况 **READ COMMITTED**
- **可重复读RR** 不会生脏写、脏读和不可重复读  但是可能发生幻读   **REPEATABLE READ**
- 串行化 **SERIALIZABLE**

```mysql
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE; 全局
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE; 会话
```

**MySQL默认都是 RR 可重复读级别 并且通过mvcc机制也不会发生幻读的情况。**



## MVCC 多版本并发控制机制

### 版本链

**trx_id**：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的事务id赋值给trx_id隐藏列。
**roll_pointer**：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

> 实际上insert undo只在事务回滚时起作用，当事务提交后，该类型的undo日志就没用了，它占用的Undo Log Segment也会被系统回收（也就是该undo日志占用的Undo页面链表要么被重用，要么被释放）。

**版本链的头节点就是当前记录最新的值 ,另外，每个版本中还包含生成该版本时对应的事务id，这个信息很重要** 

<img src="https://yakax-version2.oss-cn-chengdu.aliyuncs.com/blog/mysql/trx/version-lian.png!print " />

### ReadView

**每次执行一次事务会产生一个ReadView视图**

```
一个是m_ids，这个就是说此时有哪些事务在MySQL里执行还没提交的；
一个是min_trx_id，就是m_ids里最小的值；
一个是max_trx_id，这是说mysql下一个要生成的事务id，就是最大事务id；并不一定是m_ids 最大的事务id
  事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，m_ids就包	 括1和2，min_trx_id的值就是1，max_trx_id的值就是4。
一个是creator_trx_id，就是你这个事务的id。
```

- 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的trx_id属性值大于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断一下trx_id属性值是不是在m_ids列表中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

**简单来说MySQL在每次查询的时候会判断数据的trxid是否小于等于自己，如果小于等于自己就可以查询出来。**

**如果不等于 会基于这条数据去找undolog链条找指向的小于等于我当前事务id的这条数据的版本出来。** 

期间判断如果数据的事务id直接大于我当前视图最大事务id，会直接忽略。 事务id 只有在提交事务后才会到数据上面。

<font color=red>**RR 机制也就是判断m_ids中如果查询出来的数据事务版本号存在这里面说明 在开启视图的时候这些事务就存在了还没提交，我不能查询出来这也就解决了不可重复度问题，由于新插入进来的数据版本号必定大于我当前视图最大id，也查询不到，也就解决了幻读问题。**</font>

**RC 的机制是可以读取别人已经提交的事务数据 就是根据 判断事务id是否存在于m_ids中，存在就不查询出来，不存在说明可以查询出来**

### 小结

​		所谓的MVCC（Multi-Version Concurrency Control ，多版本并发控制）指的就是在使用READ COMMITTD、REPEATABLE READ这两种隔离级别的事务在执行普通的SEELCT操作时访问记录的版本链的过程，这样子可以使不同事务的读-写、写-读操作并发执行，从而提升系统性能。

​		READ COMMITTD、REPEATABLEREAD这两个隔离级别的一个很大不同就是：生成ReadView的时机不同，READ COMMITTD在每一次进行普通SELECT操作前都会生成一个ReadView，而REPEATABLEREAD只在第一次进行普通SELECT操作前生成一个ReadView，之后的查询操作都重复使用这个ReadView就好了。

​        之前说执行DELETE语句或者更新主键的UPDATE语句并不会立即把对应的记录完全从页面中删除，而是执行一个所谓的delete mark操作，相当于只是对记录打上了一个删除标志位，这主要就是为MVCC服务。

### 关于purge

​		我们说insert undo在事务提交之后就可以被释放掉了，而update undo由于还需要支持MVCC，不能立即删除掉。为了支持MVCC，**对于delete mark**操作来说，仅仅是在记录上打一个删除标记，并没有真正将它删除掉。随着系统的运行，在确定系统中包含最早产生的那个ReadView的事务不会再访问某些update undo日志以及被打了删除标记的记录后，有一个后台运行的purge线程会把它们真正的删除掉。
