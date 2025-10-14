+++
date = '2025-10-14T14:12:34+08:00'
draft = false
title = 'Go Channel源码解读'
comments = true
tags = ["Go", "Channel", "Source"]

+++



> **“Don’t communicate by sharing memory, share memory by communicating.”**
> 不要通过共享内存来通信，而要通过通信来共享内存。



# channel 介绍

channel是Go语言内置的一个非常重要的feature，相比其他语言，channel为Go提供了goroutine之间通信的独特方式。它和Linux的管道很像，goroutine可以向可写的channel写入数据，也可以从channel中读取数据，还可以关闭channel。这篇文章结合Go的源码来分析channel的实现原理，包括channel的创建、读、写和关闭。



# channel实现原理

## hchan结构体

Go语言的channel本质上是一个带锁的等待队列的循环缓冲区队列，它的源码在runtime/chan.go中，其实就是一个[hchan](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=34)结构体：

```Go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	timer    *timer // timer feeding this chan
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

`hchan`结构体各成员的含义如下：

* qcount: 循环缓冲区队列当前存放的数据量，`qcount = 0` 时缓冲区为**空**，`qcount = dataqsiz` 时缓冲区**满**；
* dataqsiz: 循环缓冲区队列的容量（即`make(chan T, N)`时指定的**`N`**）；
* buf: 指向循环缓冲区队列；
* elemsize: channel元素大小（注意这里是uint16，所以大小不能超过65535字节）；
* closed: 标识channel是否已经被关闭；
* timer: 与channel关联的定时器，主要用于select和超时机制；
* elemtype: channel元素的类型信息，runtime需要用它来确定elemsize；
* sendx: 下一个要写入的循环缓冲区索引；
* recvx: 下一个要读取的循环缓冲区索引；
* recvq: 正在等待接收数据的goroutine队列，当缓冲区空且者无发送者时，尝试以阻塞方式读取channel的goroutine会被加入此队列；
* sendq: 正在等待发送数据的goroutine队列，当无接收者且缓冲区已满时，尝试以阻塞方式读取channel的goroutine会被加入此队列；
* lock: 保护整个hchan中所有成员的锁；

这个结构体相对比较简单，各成员的功能也相对比较好理解，下面就分别介绍channel的创建、写入、读出、关闭的实现。



## 创建channel

Go语言中，使用channel之前需要先用`make`创建，有两种方式：

```Go
// 方式一，创建无缓冲的channel
make(chan int)

// 方式二，创建有缓冲的channel
make(chan int, 8)
```

无论是哪种方式创建的，Go的编译器都是调用`runtime.makechan()`来创建。从直觉上来看，channel的底层就是一个hchan结构体，创建channel就是创建一个结构体，并根据创建时提供的参数对结构体的相关成员初始化即可。以下是[`runtime.makechan()`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=73)的源码，makechan接受两个参数，第一个参数是channel元素类型相关的指针，第二个参数是缓冲区大小，无缓冲则为0。

在为`hchan`分配循环缓冲区时，源码考虑了三种情况：

* 无缓冲 或 缓冲区大小为0，这种情况不需要分配缓冲区，所以只创建一个`hchan`本体即可；
* 有缓冲区，但是元素的类型不是指针，这种情况把数据写入channel时，只需要将要写的数据拷贝一份到循环缓冲区即可，所以就把hchan本体和缓冲区放在一起，申请一整块内存，并让buf指向hchan结束的位置即可；
* 有缓冲，且元素类型是指针，这种情况就需要特殊考虑了，因为如果直接复制指针，就不会受GC的管理，可能存在UAF问题。

分配好缓冲区之后，再对`elemsize`等成员赋值即完成了channel的创建。

```Go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem

	// compiler checks this but be safe.
  // 元素的大小必须小于64K，因为hchan的elemsize类型为uint16，超过就会溢出
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}

  // 计算缓冲区大小（单位为字节），并检查大小是否溢出
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
  // 创建hchan结构体，并初始化c.buf，分为三种情况
	var c *hchan
	switch {
  case mem == 0:	// 无缓冲 或 元素大小为0（如chan struct{}）
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case !elem.Pointers():	// 元素类型不是指针，直接在hchan之后分配缓冲区
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:	// 有缓冲 且 元素类型是指针，需要特别对待（考虑GC）
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

  // 为其他成员赋值
	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c
}
```





## 写入channel

channel的写入可以分为两种：**阻塞写**和**非阻塞写**。

```Go
// 阻塞写，如果c不可写，则一直阻塞，直到c可写时再被唤醒
c <- v

// 非阻塞写，用select + default，如果c不可写，则不会阻塞，而是直接执行default分支
select {
case c <- v:
  println("writen")
default:
  println("default")
}
```

对于**非阻塞写**的情况，Go会将其[优化](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=736)成如下方式，即调用[`selectnbsend()`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=752)函数，`selectnbsend()`和阻塞写一样——最终调用`chansend()`。

```Go
// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
```

以下是[`chansend()`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=171)函数简化版。总结来说写入逻辑为：

* 对于*非阻塞*写，先尝试fast path，即在不加锁的情况下判断，如果channel未关闭，且循环缓冲区已满，则立即返回；
* 以下操作在加锁情况下执行
  * 若channel已经关闭，则触发panic；（禁止向closed的channel写就是这么来的）；
  * 若当前接收等待队列不为空，则直接把数据交给接受队列中的goroutine，省去了写入循环缓冲区的开销，也导致了channel不能保证FIFO。否则向下执行；
  * 判断循环缓冲区是否已满，如果未满，则将数据写入循环缓冲区，并更新`sendx`和`qcount`；如果循环缓冲区已满，对于非阻塞写直接返回；
  * 对于阻塞写，将自己加入等待写入队列，然后调用gopark触发调度；

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  // 如果c是nil，且以非阻塞方式写入，则立即返回false，调用gopark让出执行权
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	// ......
  
  // 如果是非阻塞方式读，则执行Fast path, 即在 [不加锁] 的情况下判断：
  // 如果channel未close，且循环缓冲区已满，则立即返回
	if !block && c.closed == 0 && full(c) {
		return false
	}

	// ......
	// 以下操作加锁了！
	lock(&c.lock)

  // 不允许向已经closed的channel写入，违反则触发panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  // 如果接受队列部位空，则直接从接受队列取出一个接收者
  // 避免了将数据写入循环缓冲区，所以可以看出channel不保证先进先出的顺序 :)
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

  // 如果循环缓冲区未满，则将数据写入循环缓冲区，更新qcount和sendx
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

  // 如果缓冲区还是空，且以非阻塞方式写入，则直接返回
	if !block {
		unlock(&c.lock)
		return false
	}

	// ......
  
  // 将自己加入发送等待队列
	c.sendq.enqueue(mysg)
	// 调用gopark触发调度
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
  
	// ......
	return true
}
```





## 读出channel

理解了前面的写入channel就很好理解读出channel的操作了。和写入channel一样，读出channel也分为**阻塞读**和**非阻塞读**两种情况。

```go
// 阻塞读
v := <-c
// 非阻塞读，用select + default，如果c暂时没有数据，则不会啧啧，而是直接执行default分支
select {
case v := <-ch:
	println(v)
default:
	println("default")
}
```

对于**非阻塞读**的情况，Go会将其[优化](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=756)成如下形式，即**非阻塞读**会调用`selectnbrecv()`，最终还是和阻塞读一样——调用`chanrecv()`。

```go
// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selected, ok = selectnbrecv(&v, c); selected {
//		... foo
//	} else {
//		... bar
//	}
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
	return chanrecv(c, elem, false)
}
```

以下是[`chanrecv()`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=504)函数的简化版。总结来说，读出逻辑为：

* 对于非阻塞读，先尝试fast path，即在不加锁的情况下判断，如果循环接受队列为空，则立即返回；
* 以下操作在加锁的情况下执行：
  * 若channel已经关闭，且循环缓冲区为空，则返回。否则继续执行；
  * 如果发送等待队列不为空，则直接从发送等待队列的goroutine获取数据。否则继续执行；
  * 如果循环缓冲区不为空，则从循环缓冲区读取数据并返回；
  * 如果循环缓冲区为空，且是非阻塞读，则立即返回，否则将自己加入接收等待队列并调用gopark触发调度。

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 如果channel为nil，且是非阻塞读，直接返回，否则调用gopark触发调度
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	// 如果非阻塞读，且循环缓冲区为空，则直接返回
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 { // 如果channel已关闭，直接return
			return
		}
	}
	// 加锁保护
	lock(&c.lock)

  // 如果通道已关闭，且循环缓冲区没有数据，则返回
	if c.closed != 0 {
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
		// The channel has been closed, but the channel's buffer have data.
	} else {
		// Just found waiting sender with not closed.
    // 若发送等待队列不为空，则直接从发送等待队的goroutine接收数据
		if sg := c.sendq.dequeue(); sg != nil {
			// Found a waiting sender. If buffer is size 0, receive value
			// directly from sender. Otherwise, receive from head of queue
			// and add sender's value to the tail of the queue (both map to
			// the same buffer slot because the queue is full).
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

  // 如果循环缓冲区不为空，则从循环缓冲区读取
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

  // 如果是非阻塞读，且未读到，则直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 如果是阻塞读，将自己放入接收等待队列
	c.recvq.enqueue(mysg)
	// 触发调度
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

	// ......
}
```





## 关闭channel

关闭channel在语法是是`close(c)`，实际上是调用了[`runtime.closechan()`](https://cs.opensource.google/go/go/+/refs/tags/go1.23.4:src/runtime/chan.go;l=397)。关闭channel的逻辑如下：

* 禁止关闭nil的channel，违反则触发panic；
* 禁止关闭已经被关闭的channel，这就是著名的**close of closed channel**；
* 



```go
func closechan(c *hchan) {
  // 禁止关闭nil
	if c == nil {
		panic(plainError("close of nil channel"))
	}

  // 加锁执行
	lock(&c.lock)
  // 禁止关闭已经被关闭的channel
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	// 标记为已关闭
	c.closed = 1

	var glist gList

	// release all readers
  // 遍历接收等待队列，将队列中的goroutine加入唤醒队列
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
  // 遍历发送等待队列，将队列中的goroutine加入唤醒队列
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
  // 遍历唤醒队列，唤醒队列中的goroutine
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

