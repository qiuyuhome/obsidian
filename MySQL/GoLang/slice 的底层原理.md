[Go Slice探秘——slice作为函数参数传递时，若修改函数中的slice，到底会不会改变原slice的值？ - 掘金](https://juejin.cn/post/6844904177022271501)


## slice 的底层结构
```go
type slice struct {
	array unsafe.Pointer // 指向底层数组的指针
	len   int            // 切片长度
	cat   int            // 切片容量
}
```

## 扩容机制
当向切片中 append 时，如果容量不足，会发生扩容。原 slice 的容量小于 1024，则会扩容为原来的 2 倍。大于 1024，则扩容为 1.25 倍。这里后面的版本可能有所更改。
扩容的方式是：重新申请一个内存空间，把原来的元素赋值到新的内存空间中，然后 append 新的元素，返回这个新的 slice。所以只要发生扩容，就已经和原来的 slice 使用的不是一个底层数组了。
之后再修改原来的 slice 的元素，不会影响到扩容后的 slice。

## 复制切片
```go
s1 := []int{1, 2}
s2 := s1
```
想这种，`s2 := s1`，实际上，使用的还是同一个底层数组。改变 s1 的元素值，会影响到 s2。

```go
func main() {  
   s1 := []int{1, 2}  
   s2 := s1  
   fmt.Println(s1, s2) // [1 2] [1 2]  
   s1[0] = 0  
   fmt.Println(s1, s2) // [0 2] [0 2]  
}
```

## 当参数传递
> Go的slice类型中包含了一个array指针以及len和cap两个int类型的成员。

> Go中的参数传递实际都是值传递，将slice作为参数传递时，函数中会创建一个slice参数的副本，这个副本同样也包含array,len,cap这三个成员。

> 副本中的array指针与原slice指向同一个地址，所以当修改副本slice的元素时，原slice的元素值也会被修改。但是如果修改的是副本slice的len和cap时，原slice的len和cap仍保持不变。

> 如果在操作副本时由于扩容操作导致重新分配了副本slice的array内存地址，那么之后对副本slice的操作则完全无法影响到原slice，包括slice中的元素。


## 并发问题
slice 是并发不安全的类型。是不会报错的。但结果可能意想不到。
在多个 groutine 中并发的处理 slice 时，要加锁。


## 考题
```go
// main_pointer.go  
package main  
  
import "fmt"  
  
func main() {  
   s1 := []int{1, 2}  
   // 这是复制的 slice 结构体，包含了 数组的指针，长度，容量。  
   // 数组的指针，s1 和 s2 是同一个。  
   s2 := s1  
  
   // s2 发生扩容：  
   // 先扩容，重新分配一个内存空间，len 是 2，cap 是 4，将原来的数据 1、2 复制到这个新的内存空间，  
   // 再把 3 追加进去，len 变为 3，再返回给 s2 变量。  
   s2 = append(s2, 3)  
  
   // 这个时候，s1 没有发生任何变化。  
   // 上面对 s2 的 append，因为发生了扩容，已经使用了新的内存地址，所以追加的 3 对于原本 s1 和 s2 公用的底层数组是没有任何影响的。  
   // 在 SliceRise(s1) 里面，因为 s = append(s, 0)，会发生扩容，使用新的底层数组，所以 for 里面的元素自增，不会对 s1 有影响。  
   SliceRise(s1)  
  
   // 这个时候的 s2，len 是3，cap 是 4。进入函数以后，append，不会发生扩容，所以，经过 for 后，s2 的值就发生了变化。  
   SliceRise(s2)  
  
   // [1, 2] [2, 3, 4]  
   fmt.Println(s1, s2)  
}  
  
func SliceRise(s []int) {  
   s = append(s, 0)  
   for i := range s {  
      s[i]++  
   }  
}
```


```go
s1 := []int{1, 2}  
fmt.Printf("%p, %p\n", s1, &s1) // s1指的是底层数组的地址，&s1是切片的地址
```
