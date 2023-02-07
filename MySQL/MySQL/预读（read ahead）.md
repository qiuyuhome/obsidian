目的：通过预读策略，减少磁盘IO。

两种预读算法来提高性能：

1.  线性预读：以extent为单位，将下一个extent提前读取到buffer pool中；
2.  随机预读：以extent中的page为单位，将当前extent中的剩余的page提前读取到buffer pool中；

**线性预读**
一个重要参数：innodb_read_ahead_threshold，控制什么时间（访问extent中多少页的阈值）触发预读；

1.  默认：56，范围：0～64，值越高，访问模式检查越严格；
2.  没有该变量之前，当访问到extent最后一个page时，innodb会决定是否将下一个extent放入到buffer pool中；

**随机预读说明**

1.  当同一个extent的一些page在buffer pool中发现时，innodb会将extent中剩余page一并读取到buffer pool中；
2.  随机预读给innodb code带来一些不必要的复杂性，性能上也不稳定，在5.5版本已经废弃，如果启用，需要修改变量：innodb_random_read_ahead为ON；



