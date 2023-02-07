## 目的
降低写操作的磁盘IO

## 原理
1. Change Buffer是一种特殊的内存结构，它是一种应用在非唯一普通索引页（non-unique secondary index page）不在缓冲池中，对页进行了写操作，并不会立刻将磁盘页加载到缓冲池，而仅仅记录缓冲变更（Buffer Changes），等未来数据被读取时，再将数据合并（Merge）恢复到缓冲池中的技术。写缓冲的目的是降低写操作的磁盘IO，提升数据库性能；

2. 如果索引设置了唯一（Unique）属性，在进行修改操作时，InnoDB必须进行唯一性检查，是不能使用到Change Buffer的；

3. 根据业务使用情况，可以指定Change Buffer占整个缓冲池的比例。同时，也可以指定配置哪些写操作启用写缓冲，可以设置成all/none/inserts/deletes等；

4. Change Buffer不会出现数据不一致的情况，原因有二：
	- 数据库异常奔溃，能够从redo log中恢复数据；
	- 写缓冲不只是一个内存结构，它也会被定期刷盘到写缓冲系统表空间。

https://www.modb.pro/db/112469
