
**关键词：**
trx_id
roll_pointer
ReadView


MVCC（MultiVersion Concurrency Control）就是通过一定机制生成一个数据请求时间点的一致性数据快照，并用这个快照来提供一定级别（语句级或事务级）的一致性读取。

记录的是某个时间点上的数据快照，用来实现不同事务之间数据的隔离性。

MVCC的实现方式通过两个隐藏列trx_id（最近一次提交事务的ID）和roll_pointer（上个版本的地址），建立一个版本链。并在事务中读取的时候生成一个ReadView（读视图），在Read Committed隔离级别下，每次读取都会生成一个读视图（ReadView），而在Repeatable Read隔离级别下，只会在第一次读取时生成一个读视图

![[Pasted image 20221228172104.png]]

  ![[Pasted image 20221228160333.png]]