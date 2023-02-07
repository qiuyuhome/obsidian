![[Pasted image 20230131102309.png]]

## 表级锁
### (1) 概念


当前操作的整张表加锁，最常使用的 MyISAM 与 InnoDB 都支持表级锁定。

MySQL 里面表级别的锁有两种：一种是**表锁**，一种是**元数据锁（meta data lock，MDL)**。

MySQL 表锁有两种类型：共享的读锁和排他的写锁。

### (2) 实现方式

表锁：lock tables … read/write；

例如lock tables t1 read, t2 write; 命令，则其他线程写 t1、读写 t2 的语句都会被阻塞。同时，线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。连写 t1 都不允许，自然也不能在unlock tables之前访问其他表。

元数据锁：MDL 不需要显式使用，在访问一个表的时候会被自动加上，在 MySQL 5.5 版本中引入了 MDL，当对一个表做增删改查操作的时候，加 MDL读锁；当要对表做结构变更操作的时候，加 MDL 写锁。

给一个表加字段，或者修改字段，或者加索引，需要扫描全表的数据。在对大表操作的时候，肯定会特别小心，以免对线上服务造成影响。而实际上，即使是小表，操作不慎也会出问题。

1. sessionA:

**begin;**

**select* from t limit 1;**

2. sessionB:

**select* from t limit 1;**

3. sessionC:

**altertable t add f int;**

> 会mdl锁住

4. sessionD:

**select* from t limit 1;**

![[Pasted image 20230131112055.png]]

我们可以看到 session A 先启动，这时候会对表 t 加一个 MDL 读锁。由于session B 需要的也是 MDL 读锁，因此可以正常执行。

之后 session C 会被 blocked，是因为 session A 的 MDL 读锁还没有释放，而 sessionC 需要MDL 写锁，因此只能被阻塞。

如果只有 session C 自己被阻塞还没什么关系，但是之后所有要在表 t 上新申请 MDL 读锁的请求也会被session C 阻塞。前面说了，`所有对表的增删改查操作都需要先申请MDL 读锁，而这时读锁没有释放，对表alter ，产生了mdl写锁，把表t锁住了，这时候就对表t完全不可读写了`。

如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新session 再请求的话，这个库的线程很快就会爆满。

事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。  
`**注 ：一般行锁都有锁超时时间。但是MDL锁没有超时时间的限制，只要事务没有提交就会一直锁注。**`

### (4) 解决办法

首先我们要解决长事务，事务不提交，就会一直占着 MDL 锁。在 MySQL 的information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。如果你要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务。这也是为什么需要在低峰期做ddl 变更。

![[Pasted image 20230131112356.png]]


**InnoDB 既实现了行锁，也实现了表锁。  
当有明确指定的主键/索引时候，是行级锁，否则是表级锁**

### InnoDB行锁

行锁的具体实现算法有三种：record lock、gap lock以及next-key lock。

- record lock是专门对索引项加锁；
- gap lock是对索引项之间间隙加锁；
- next-lock则是前面两种的组合，对索引项以及之间的间隙加锁。

只在可重复读或以上隔离级别下的特定操作才会取得 gap lock 或 next-key lock，在 Select、Update 和 Delete 时，除了基于唯一索引的查询之外，其它索引查询时都会获取 gap lock 或 next-key lock，即锁住其扫描的范围。主键索引也属于唯一索引，所以主键索引是不会使用 gap lock 或 next-key lock

### for update 的使用方式

**for update 仅适用于InnoDB**，并且**必须开启事务**，在begin与commit之间才生效。

```csharp
select * from table_name where ... for update
```

select 语句默认不获取任何锁，所以是可以读被其它事务持有排它锁的数据的！

在MySQL中，行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql语句操作了主键索引，MySQL就会锁定这条主键索引；如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引。在UPDATE、DELETE操作时，MySQL不仅锁定WHERE条件扫描过的所有索引记录，而且会锁定相邻的键值，即所谓的next-key locking。

**InnoDB行锁是通过给索引上的索引项加锁来实现的**。所以，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。其他注意事项：

-   在不通过索引条件查询的时候，InnoDB使用的是表锁，而不是行锁。
-   由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以即使是访问不同行的记录，如果使用了相同的索引键，也是会出现锁冲突的。
-   当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。
-   即便在条件中使用了索引字段，但具体是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。




![[Pasted image 20230131120951.png]]
![[Pasted image 20230131121019.png]]

