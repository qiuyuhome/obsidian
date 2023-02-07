![[Pasted image 20221227111722.png]]


**Server层**又分为连接器、缓存、分析器、优化器、执行器。所有跨存储引擎的功能都在这层实现，比如：函数、存储过程、触发器、视图等。

**存储引擎**是可插拔式的，常见的存储引擎有MyISAM、InnoDB、Memory等，MySQL5.5之前默认的是MyISAM，之后默认的是InnoDB。

