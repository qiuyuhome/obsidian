先来说一下大字段在mysql 中存储的原理。

数据页默认大小为 16 KB，是 16384 byte。
单行最大能存储的 65535 byte。

行格式为 COMPACT 时：
当列的类型为VARCHAR、 VARBINARY、 BLOB、TEXT时，该列超过 768byte 的数据放到其他数据页中去。

[MySQL :: MySQL 5.7 Reference Manual :: 8.4.7 Limits on Table Column Count and Row Size](https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html)

[MySQL :: MySQL 8.0 Reference Manual :: 15.10 InnoDB Row Formats](https://dev.mysql.com/doc/refman/8.0/en/innodb-row-format.html)

[Mysql-一行能存多少数据？ - 墨天轮](https://www.modb.pro/db/62018)


- InnoDB 的默认行格式是DYNAMIC    
- DYNAMIC存储可变字符串使用的方式和COMPACT格式基本相同，动态行格式不会在记录的真实数据处存储字段数据的前768个字节，而是把所有数据都存储到其他页面中，只记录存储在其他页面的地址。


每页最少存储 2 行数据，一夜是 16k，所以每行数据的最大字符是 16k/2 = 8192 byte

当一行的数据真实字节数大于 8192 字节时，就会报错。（例如 COMPACT 行格式，创建了11 个 varchar(1000) 字段，因为每个字段都会存储 768 字节的数据，其他数据放到溢出页中了，768 * 11 = 8448，已经大于 8192 了，就会报错。）


```sql
INSERT INTO big_table (var_1,var_2,var_3,var_4,var_5,var_6,var_7,var_8,var_9,var_10，var_11)
SELECT
    REPEAT('a', 1000),
    REPEAT('b', 1000),
    REPEAT('c', 1000),
    REPEAT('d', 1000),
    REPEAT('e', 1000),
    REPEAT('f', 1000),
    REPEAT('g', 1000),
    REPEAT('h', 1000),
    REPEAT('i', 1000),
    REPEAT('j', 1000),
    REPEAT('k', 1000);
```


行溢出  
innodb采用B+树结构，每个数据页至少要存储两个记录(否则变成了链表)，如果不能满足这个需求，则会出现行溢出，innodb只保留字段前20字节

(pre-plugin保留768字节)，剩余数据分配新的数据页uncompress blob page；  
行溢出只与记录长度有关系，即便行包含blob或text，若一个数据页可存储两条记录，也不会出现行溢出；  
具体可参看《MySQL技术内幕-InnoDB存储引擎的》的4.4章节 innodb行记录格式；

对于pre-plugin，行溢出时保留每个字段的前768字节，如果保留的字节总数仍然超出数据页的一半大小，就会直接报错1030- Got error 139 from storage engine。譬如一个表建立11个text或varachar(1000)字段且行记录长度>8k，则会产生行溢出，保留在原数据页的记录长度=11*768>8k，此时不能再继续行溢出了，因此只能报错。


[sql - What is off page in Mysql? - Stack Overflow](https://stackoverflow.com/questions/48298502/what-is-off-page-in-mysql)

