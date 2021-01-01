# Mybatis与hibernate

**总的来说，Hibernate使用的是封装好的，通用的SQL来应付所有场景，而Mybatis是针对响应的场景设计的SQL。Mybatis的SQL会更灵活、可控性更好、更优化**

> - Hibernate的查询会将表中的所有字段查询出来，这一点会有性能消耗。
> - Hibernate也可以自己写SQL来指定需要查询的字段，但这样就破坏了Hibernate开发的简洁性。
> - Mybatis的SQL是手动编写的，所以可以按需求指定查询的字段。
> - hibernate迁移性相对mybatis较好，因为hibernate不是依赖数据库。
<!--more-->
#### hibernate 一级缓存与二级缓存
1. 减少对数据库的访问次数！从而提升hibernate的执行效率！
2. 一级缓存又称为“Session的缓存”，它是内置的，不能被卸载（不能被卸载的意思就是这种缓存不具有可选性，必须有的功能，不可以取消session缓存）
3. 二级缓存又称为"SessionFactory的缓存"，由于SessionFactory对象的生命周期和应用程序的整个过程对应，因此Hibernate二级缓存是进程范围或者集群范围的缓存。
> 很少被修改的数据与不会被并发的数据放入二级缓存中
4. 访问顺序
> 一级缓存->二级缓存->数据库
5. 具体方法功能
> - session.flush(); 让一级缓存与数据库同步
  - session.evict(arg0);清空一级缓存中指定的对象
  - session.clear();清空一级缓存中缓存的所有对象
6. Hibernate缓存的是对象
7. Hibernate List与iterator查询的区别
>  - list()一次把所有的记录都查询出来，**会放入缓存，但不会从缓存中获取数据**
>  - Iterator N+1查询； N表示所有的记录总数，即会先发送一条语句查询所有记录的主键（1），再根据每一个主键再去数据库查询（N）！**会放入缓存，也会从缓存中取数据**

#### Mybatis缓存
1. 访问顺序
> 二级缓存->一级缓存->数据库
2. 一级缓存是SqlSession级别的缓存。在操作数据库时需要构造 sqlSession对象，在对象中有一个(内存区域)数据结构（HashMap）用于存储缓存数据
> 缓存作用域在用一个session之间
2. 二级缓存相对于一级缓存来说，实现了SqlSession之间缓存数据的共享，同时粒度更加的细，能够到namespace级别，通过Cache接口实现类不同的组合，对Cache的可控性也更强。
> 二级缓存是mapper级别的缓存，多个SqlSession去操作同一个Mapper的sql语句，多个SqlSession去操作数据库得到数据会存在二级缓存区域，多个SqlSession可以共用二级缓存，二级缓存是跨SqlSession的
3. MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
4. 在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis,Memcached等分布式缓存可能成本更低，安全性也更高。
5. 一般情况下，增删改后都会commit，这样要避免读取脏数据。也可以在具体mapper方法里加入flushCache="true"。

#### 缓存相对比较
>- 两者比较：因为Hibernate对查询对象有着良好的管理机制，用户无需关心SQL。所以在使用二级缓存时如果出现脏数据，系统会报出错误并提示。
>- 而MyBatis在这一方面，使用二级缓存时需要特别小心。如果不能完全确定数据更新操作的波及范围，避免Cache的盲目使用。否则，脏数据的出现会给系统的正常运行带来很大的隐患。



