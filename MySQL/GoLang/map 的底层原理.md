![[Pasted image 20230208105500.png]]

[由浅到深，入门Go语言Map实现原理 - TIGERB的技术博客 - SegmentFault 思否](https://segmentfault.com/a/1190000039101378)
[Dig101:Go之读懂map的底层设计 | 菜鸟Miao](http://blog.newbmiao.com/2020/02/04/dig101-golang-map.html)
[【Golang】Map长啥样儿？_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Sp4y1U7dJ/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=9fc264da94f6c0007e2537a3bfdfb2b9)

## 源码解析

### 结构体
```go
// runtime/map.go  
// A header for a Go map.  
type hmap struct {  
  //用于len(map)  
  count     int  
  //标志位  
  // iterator     = 1 // 可能有遍历用buckets  
  // oldIterator  = 2 // 可能有遍历用oldbuckets，用于扩容期间  
  // hashWriting  = 4 // 标记写，用于并发读写检测  
  // sameSizeGrow = 8 // 用于等大小buckets扩容，减少overflow桶  
  flags     uint8  
  
  // 代表可以最多容纳loadFactor * 2^B个元素（loadFactor=6.5）  
  B         uint8  
  // overflow桶的计数，当其接近1<<15 - 1时为近似值  
  noverflow uint16  
  // 随机的hash种子，每个map不一样，减少哈希碰撞的几率  
  hash0     uint32  
  
  // 当前桶，长度为（0-2^B）  
  buckets    unsafe.Pointer  
  // 如果存在扩容会有扩容前的桶  
  oldbuckets unsafe.Pointer  
  // 迁移数，标识小于其的buckets已迁移完毕  
  nevacuate  uintptr  
  
  // 额外记录overflow桶信息，不一定每个map都有  
  extra *mapextra  
}  
  
// 额外记录overflow桶信息  
type mapextra struct {  
  overflow    *[]*bmap  
  oldoverflow *[]*bmap  
  
  // 指向下一个可用overflow桶  
  nextOverflow *bmap  
}  
  
const(  
  // 每个桶8个k/v单元  
  BUCKETSIZE  = 8  
  // k或v类型大小大于128转为指针存储  
  MAXKEYSIZE  = 128  
  MAXELEMSIZE = 128  
)  
  
// 桶结构 （字段会根据key和elem类型动态生成，见下边bmap）  
type bmap struct {  
  // 记录桶内8个单元的高8位hash值，或标记空桶状态，用于快速定位key  
  // emptyRest      = 0 // 此单元为空，且更高索引的单元也为空  
  // emptyOne       = 1 // 此单元为空  
  // evacuatedX     = 2 // 用于表示扩容迁移到新桶前半段区间  
  // evacuatedY     = 3 // 用于表示扩容迁移到新桶后半段区间  
  // evacuatedEmpty = 4 // 用于表示此单元已迁移  
  // minTopHash     = 5 // 最小的空桶标记值，小于其则是空桶标志  
  tophash [bucketCnt]uint8  
}  
  
// cmd/compile/internal/gc/reflect.go  
// func bmap(t *types.Type) *types.Type {  
// 每个桶内k/v单元数是8  
type bmap struct{  
  topbits [8]uint8 //tophash  
  keys [8]keytype  
  elems [8]elemtype  
  // overflow 桶  
  // otyp 类型为指针*Type,  
  // 若keytype及elemtype不含指针，则为uintptr  
  // 使bmap整体不含指针,避免gc去scan此类map  
  overflow otyp  
}
```

例如：hmap.B = 2，则 2^2 = 4，有 4 个桶。
![[Pasted image 20230208104834.png]]



## 字段解释
这里有几个字段需要解释一下：
### hmap.B

这个为啥用2的对数来表示桶的数目呢？
这里是为了hash定位桶及扩容方便，常见的通过哈希值计算索引位置的方法有取模和与运算。
与运算要保证桶的个数为 2 的整次幂，否则 hash&(n-1) 会出现有些桶出现不会被选中的情况。
比方说，`hash%n`可以定位桶， 但`%`操作没有位运算快。
而利用 `n=2^B，则hash%n=hash&(n-1)`。
则可优化定位方式为: `hash&(1<<B-1)`， `(1<<B-1)`即源码中`BucketMask`。
再比方扩容，`hmap.B=hmap.B+1` 即为扩容到二倍。

### bmap.keys, bmap.elems

在桶里存储k/v的方式不是一个k/v一组, 而是k放一块，v放一块。
这样的相对k/v相邻的好处是，方便内存对齐。比如`map[int64]int8`, v是`int8`,放一块就避免需要额外内存对齐。
另外对于大的k/v也做了优化。

正常情况key和elem直接使用用户声明的类型，但当其size大于128(`MAXKEYSIZE/MAXELEMSIZE`)时，
则会转为指针去存储。（也就是`indirectkey、indirectelem`）

### hmap.extra

这个额外记录溢出桶意义在哪？

具体是为解决让`gc`不需要扫描此类`bucket`。
只要bmap内不含指针就不需gc扫描。

当`map`的`key`和`elem`类型都不包含指针时，但其中的`overflow`是指针。
此时bmap的生成函数会将`overflow`的类型转化为`uintptr`。
而`uintptr`虽然是地址，但不会被`gc`认为是指针，指向的数据有被回收的风险。
此时为保证其中的`overflow`指针指向的数据存活，就用`mapextra`结构指向了这些`buckets`，这样bmap有被引用就不会被回收了。

关于uintptr可能被回收的例子，可以看下 [go101 - Type-Unsafe Pointers](https://go101.org/article/unsafe.html) 中 Some Facts in Go We Should Know

## 哈希冲突
当多个 key 的哈希值一样，就是哈希冲突，解决哈希冲突有多种方式，如 开放定址法、再哈希法、 链地址法（拉链法）等等。
go 使用的是 **链地址法（拉链法）**。

## 负载因子
存储键值对的数目 / 桶的数目，结果就是 **负载因子**

Go则在在负载因子达到 **6.5** 时才会触发rehash。
因为 go 的每个桶中，可以存放 8 个键值对。因此负载因子比较高。

## 扩容

使用的是 **渐进式扩容**。是在每次读写 map 的时候，迁移 2 个。而不是一次性扩容，来避免扩容时的瞬时性能抖动。

### 翻倍扩容
**触发条件**
count / 2^B > 6.5，就会触发翻倍扩容。

扩容的时候，buckets 指向新分配的桶， oldbuckets 指向旧的桶。nevacuate 为 0，代表接下来要迁移编号为 0 的旧桶。每个桶的键值对，通过与运算的方式 **hash & (m - 1)**，会被重新分配到新的2个桶中。

### 等量扩容
负载因子没有超标，但溢出桶 noverflow 的个数比较多。就会触发等量扩容。
**触发条件**
**第一种：**
(B <= 15) && (noverflow >= 2^B)
**第二种**
(B > 15) && (noverflow >= 2^15)

等量扩容就是重新建立一个与原来一样大小的桶，重新分配。主要是为了让键值对排列的更加紧凑，减少溢出桶的使用，更节省空间。

## 总结
-   以2的对数存储桶数，便于优化hash模运算定位桶，也利于扩容计算
-   每个map都随机hash种子，减少哈希碰撞的几率
-   map以key的类型确定hash函数，对不同类型针对性优化hash计算方式
-   桶内部k/v并列存储，减少不必要的内存对齐浪费；对于大的k/v也会转为指针，便于内存对齐和控制桶的整体大小
-   桶内增加tophash数组加快单元定位，也方便单元回收（空桶）标记
-   当桶8个单元都满了，还存在哈希冲突的k/v，则在桶里增加overflow桶链表存储
-   桶内若只有overflow桶链表是指针，则overflow类型转为uintptr，并使用mapextra引用该桶，避免桶的gc扫描又保证其overflow桶存活
-   写操作增加新桶时如果需要扩容，只记录提交，具体执行会分散到写操作和删除操作中渐进进行，将迁移成本打散
-   哈希表的装载因子不满足要求是，扩容一倍，保证桶的装载能力
-   哈希表overflow桶过多，则内存重新整理，减少不必要的overflow桶，提升读写效率
-   对指定不同大小的map初始化，区别对待，不必要的桶预分配就避免；桶较多的情况下，也增加overflow桶的预分配
-   每次遍历起始位置随机，严格保证map无序语义
-   使用flags位标记检测map的并发读写，发现时panic，一定程度上预防数据不一致发生
