## 参考
[MySQL :: MySQL 5.7 Reference Manual :: 8.2.1.5 Index Condition Pushdown Optimization](https://dev.mysql.com/doc/refman/5.7/en/index-condition-pushdown-optimization.html)
[一文读懂什么是MySQL索引下推（ICP） - 简书](https://www.jianshu.com/p/31ceadace535)



## 目的
减少回表次数

5.6 以后才有的功能，默认是开启的。

## 触发条件
联合索引，包含了查询条件。
