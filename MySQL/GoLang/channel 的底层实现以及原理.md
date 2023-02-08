## 结构体。

```go
type hchan struct {
    qcount   uint           // 当前队列中剩余元素个数
    dataqsiz uint           // 环形队列长度，即可以存放的元素个数
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 每个元素的大小
    closed   uint32            // 标识关闭状态
    elemtype *_type         // 元素类型
    sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint           // 队列下标，指示元素从队列的该位置读出
    recvq    waitq          // 等待读消息的goroutine队列
    sendq    waitq          // 等待写消息的goroutine队列
    lock mutex              // 互斥锁，chan不允许并发读写
}
```


channel由环形队列、类型信息、goroutine等待队列、状态标识 组成。
重点，channel 也是用互斥锁 mutex 来控制并发读写的。

## 环形队列
环形队列中存放的是这个 channel 保存的数据。
与环形队列相关的结构体属性有：
```go
type hchan struct {
    qcount   uint           // 当前队列中剩余元素个数
    dataqsiz uint           // 环形队列长度，即可以存放的元素个数
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 每个元素的大小
    closed   uint32            // 标识关闭状态
    elemtype *_type         // 元素类型
    sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint           // 队列下标，指示元素从队列的该位置读出
}
```

例如 

```go
c := make(chan int32, 3)
c <- 1
```

此时的结构体中对应的属性值：
```go
type hchan struct {
    qcount   uint           // 1
    dataqsiz uint           // 3
    buf      unsafe.Pointer // 执行一个内存地址
    elemsize uint16         // 32
    closed   uint32            // 标识关闭状态
    elemtype *_type         // int32
    sendx    uint           // 实现环形队列用的属性
    recvx    uint           // 实现环形队列用的属性
}
```

## 等待队列

等待队列的结构体属性：
```go
type hchan struct {
    recvq    waitq          // 等待读消息的goroutine队列
    sendq    waitq          // 等待写消息的goroutine队列
}
```

当从 channel 中读数据，如果 channel 的环形队列中没有数据，则会阻塞，会记录到 recvq 中。
当从 channel 中写数据，如果 channel 的环形队列中已经写满，则会阻塞，会记录到 sendq 中。

一般情况下，recvq 和 sendq 至少有一个为空。
例外就是同一个goroutine使用select语句向channel一边写数据，一边读数据。

- 因读阻塞的 goroutine 会被向 channel 写入数据的 goroutine 唤醒；
- 因写阻塞的 goroutine 会被从 channel 读数据的 goroutine 唤醒；

## 锁
结构体中有一个互斥锁 mutex。同一时刻，值允许被一个 groutine 读写。


## 创建 chan

创建 channel 的过程，实际上就是初始化 hchan 结构体的过程。

## 向 channel 写数据

1. 如果等待接收队列recvq（被阻塞的读队列）不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程；
2. 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；
3. 如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒；


## 从 channel 读数据

1. 如果等待发送队列sendq（被阻塞的写队列）不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程；
2. 如果等待发送队列sendq不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；
3. 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
4. 将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒；

## 关闭 channel
关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。

除此之外，panic出现的常见场景还有：
1. 关闭值为nil的channel
2. 关闭已经被关闭的channel
3. 向已经关闭的channel写数据
