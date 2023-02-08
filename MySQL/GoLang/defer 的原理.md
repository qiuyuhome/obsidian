[2.1 defer · GitBook](https://books.studygolang.com/GoExpertProgramming/chapter02/2.1-defer.html)
[【Golang】defer1.13和1.14的优化策略_哔哩哔哩_bilibili](https://www.bilibili.com/list/567195437?sid=426378&desc=1&oid=456052143&bvid=BV1b5411W7ih)


## 规则一：延迟函数的参数在defer语句出现时就已经确定下来了

官方给出一个例子，如下所示：

```
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```

defer语句中的fmt.Println()参数i值在defer出现时就已经确定下来，实际上是拷贝了一份。后面对变量i的修改不会影响fmt.Println()函数的执行，仍然打印"0"。

注意：对于指针类型参数，规则仍然适用，只不过延迟函数的参数是一个地址值，这种情况下，defer后面的语句对变量的修改可能会影响延迟函数。


## 规则二：延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

## 规则三：延迟函数可能操作主函数的具名返回值

## 作用域问题

```go
package main

func main() {
	func() {
		defer println("--- defer ---")
	}()
	println("--- ending ---")
}
```

```
--- defer ---
--- ending ---
```

`defer` 处于一个匿名函数中，就 main 函数本身来讲，匿名函数 `fun(){}()` 先调用且返回，然后再调用 `println("--- ending ---")`

## 原理
```go
type g struct {  
   stack       stack   // offset known to runtime/cgo  
   stackguard0 uintptr // offset known to liblink  
   stackguard1 uintptr // offset known to liblink  
  
   _panic    *_panic // innermost panic - offset known to liblink  
   _defer    *_defer // innermost defer  重点看这个
   // ...
   // ...
}
```

```go
type _defer struct {  
   started bool  
   heap    bool   
   openDefer bool  
   sp        uintptr // sp at time of defer   
   pc        uintptr // pc at time of defer   
   fn        func()  // can be nil for open-coded defers  
   _panic    *_panic // panic that is running defer  
   // 看这个
   link      *_defer // next defer on G; can point to either heap or stack! 
   
   fd      unsafe.Pointer
   varp    uintptr  
   framepc uintptr  
}
```

可以看到，`g` 中有一个 `_defer` 类型的指针，`link *_defer` 表明了是一个链表。

defer 信息会注册到一个链表，而当前执行的 goroutine 持有这个链表的头指针。

新注册的 defer 会添加到链表头。所以 defer 才会表现为倒序执行。

如图：
![[Pasted image 20230208163848.png]]

![[Pasted image 20230208164630.png]]


在 1.14 版本，把 defer 函数的执行逻辑，展开在所属的函数内，从而免于创建 `_defer` 结构体，而且不需要注册到 defer 链表。这个就是 **open coded defer**。

open coded defe 不适用于循环中的 defer。


##总结

- defer定义的延迟函数参数在defer语句出时就已经确定下来了
- defer定义顺序与实际执行顺序相反
- return不是原子操作，执行过程是: 保存返回值(若有)-->执行defer（若有）-->执行ret跳转
- 申请资源后立即使用defer关闭资源是好习惯

