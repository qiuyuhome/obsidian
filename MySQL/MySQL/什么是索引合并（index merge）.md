[MySQL 是怎样运行的：从根儿上理解 MySQL - 小孩子4919 - 掘金课程](https://juejin.cn/book/6844733769996304392/section/6844733770050830350?enter_from=course_center)


一般情况下，对于单表查询MySQL一次只能利用一个索引。但是，如果条件允许，MySQL也可以利用「Index Merge」的方式利用多个索引完成一次查询。MySQL支持三种索引合并的方式，分别是Intersection、Union、Sort Union，其实就是利用二级索引中的主键值取交集、并集后再回表查询。其中Intersection和Union使用条件比较严苛，要求从二级索引取出的记录必须是根据主键排好序的。有时候条件不满足，但是MySQL又很想使用Index Merge，就会尝试自己在内存中手动排序，这就是Sort Union，它只比Union多了个手动排序的过程。

