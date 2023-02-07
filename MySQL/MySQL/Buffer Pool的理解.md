[MySQL 5.7 参考手册 / Buffer Pool](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool.html)
[缓冲池(buffer pool)，这次彻底懂了！！！](https://juejin.cn/post/6844903874172551181)
[什么是数据库的“缓存池”？](https://www.toutiao.com/article/6923175484478079500/?wid=1672134744637)


关键词：
* 改良的 LRU 算法
* 预读失效
* 缓冲池污染
* new sublist
* old sublist
* 老生代停留时间窗口

**总结**
（1）缓冲池(buffer pool)是一种**常见的降低磁盘访问的机制；**
（2）缓冲池通常**以页(page)为单位缓存数据；**
（3）缓冲池的**常见管理算法是LRU**，memcache，OS，InnoDB都使用了这种算法；
（4）InnoDB对普通LRU进行了优化：
-   将缓冲池分为**老生代和新生代**，入缓冲池的页，优先进入老生代，页被访问，才进入新生代，以解决预读失效的问题
-   页被访问，且在老生代**停留时间超过配置阈值**的，才进入新生代，以解决批量数据访问，大量热数据淘汰的问题