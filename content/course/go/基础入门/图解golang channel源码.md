---
title: 图解Golang channel源码
author: kinnylee
date: 2020-09-22
toc: true
featured: true
categories:
  - go
tags:
  - go
  - go基础入门
  - 源码分析
thumbnail: "images/go.png"
---

# 图解Golang channel源码

## 前言

先上一张channel布局图，channel的底层实际上并不复杂，没有用到很高深的知识，主要是围绕着一个环形队列和两个链表展开。相信你看完本篇文章一定能掌握channel的实现。

[高清地址](https://www.processon.com/view/link/5f6941057d9c087da1bae259)

![hchan结构](http://assets.processon.com/chart_image/5f694105637689248621dcf0.png)

## channel简介

- channel是一个类型管道，通过它可以在groutine之间发送和接收消息
- go语言层面提供的groutine之间的通讯方式

在日常开发中，对于channel的使用应该不陌生了，但是了解了基本的使用后，你是否对它的底层实现很好奇呢，为什么它就能实现并发的groutine之间通信呢？带着这个好奇，让我们研究一下channel底层的源码实现吧！

### channel使用

下面是channel的最简单的用法：

```Go
package main
import "fmt"

func main() {
  c := make(chan int)
  go func() {
    // 发送数据到channel
    c <- 1
  }()
  // 从channel取出数据
  x := <- c
  close(c)
  fmt.Println(x)
}
```

### channel源码入口

channel使用的make、<- 等符号，在源码中没有对应的实现，而是通过编译器将相关符号翻译为底层实现。
使用以下命令将go源码翻译为汇编

```bash
go tool compile -N -l -S main.go>hello.s
```

查看部分带有CALL指令的核心内容如下：

```armasm
0x0043 00067 (main.go:42) CALL  runtime.makechan(SB)
0x006a 00106 (main.go:44) CALL  runtime.newproc(SB)
0x008b 00139 (main.go:47) CALL  runtime.chanrecv1(SB)
0x0032 00050 (main.go:45) CALL  runtime.chansend1(SB)
0x00a3 00163 (main.go:48) CALL  runtime.closechan(SB)
```

可以猜测对应关系：

- make(chan int)对应：runtime.makechan函数
- 协程创建：runtime.newproc函数
- ch <- 1 写数据语句对应：runtime.chansend1函数
- x := <- ch 读数据语句对应：runtime.chanrecv1函数
- close(c) 关闭通道语句对应：runtime.closechan函数

相关源码只需要到runtime包下，全局搜索就可以找到在文件runtime/chan.go下

```Go
func makechan(t *chantype, size int) *hchan {}
func chansend1(c *hchan, elem unsafe.Pointer) {}
func chanrecv1(c *hchan, elem unsafe.Pointer) {}
func closechan(c *hchan) {}
```

## 源码分析

上述三个函数都用到一个hchan类型的参数，它就是channel的核心数据结构，我们先分析hchan

> 在IDE中下断点调试时，也能看出chan的内部数据结构

- 位置：src/runtime/chan.go

### chan数据结构

- channel内部数据结构是固定长度的双向循环列表
- 按顺利往里面写数据，写满之后又从0开始写
- chan中的两个重要组件是`buf`和`waitq`，所有的行为和实现都是围绕着两个组件进行的

github上Go夜读提供的这个图片比较形象，直接引用过来。

```Go
type hchan struct {
  // 当前队列中总元素个数
  qcount   uint           // total data in the queue
  // 环形队列长度，即缓冲区大小（申明channel时指定的大小）
  dataqsiz uint           // size of the circular queue
  // 环形队列指针
  buf      unsafe.Pointer // points to an array of dataqsiz elements
  // buf中每个元素的大小
  elemsize uint16
  // 当前通道是否处于关闭状态，创建通道时该字段为0，关闭时字段为1
  closed   uint32
  // 元素类型，用于传值过程的赋值
  elemtype *_type // element type
  // 环形缓冲区中已发送位置索引
  sendx    uint   // send index
  // 环形缓冲区中已接收位置索引
  recvx    uint   // receive index
  // 等待读消息的groutine队列
  recvq    waitq  // list of recv waiters
  // 等待写消息的groutine队列
  sendq    waitq  // list of send waiters
  // 互斥锁，为每个读写操作锁定通道（发送和接收必须互斥）
  lock mutex
}

// 等待读写的队列数据结构，保证先进先出
type waitq struct {
  first *sudog
  last  *sudog
}
```

### 创建channel

概述：

创建channel时，可以往channel中放入不同类型的数据，不同类型数据占用的空间大小也是不一样的，这决定了hchan和hchan中的buf字段需要开辟多大的存储空间。在go的源码中对不同的情况做不同的处理。包括三种情况：

> 总体的原则是：总内存大小 = hchan需要的内存大小 + 元素需要的内存大小

- 队列为空或元素大小为0：只需要开辟的内存空间为hchan本身的大小
- 元素不是指针类型：需要开辟的内存空间=hchan本身大小+每个元素的大小*申请的队列长度
- 元素是指针类型：这种情况下buf需要单独开辟空间，buf占用内存大小为每个元素的大小*申请的队列长度

输入：

- chantype：channel的类型
- size：channel大小

输出：

- 创建好的hchan对象

核心流程：

- 各种参数校验
- 数据赋值
- 创建缓冲区存储空间（区分元素为空、元素有指针、元素无指针三种情况）

[高清地址](https://www.processon.com/view/link/5f69ec4be0b34d2c46b8038c)

![channel创建](http://assets.processon.com/chart_image/5f69ec4b07912949af466a6d.png)

```Go
// 对应的源码为 c := make(chan int, size)
// c := make(chan int) 这种情况下，size = 0
func makechan(t *chantype, size int) *hchan {
  elem := t.elem

  // 总共需要的buff大小 = channel中创建的这种元素类型的大小（elem.size）* size
  mem, overflow := math.MulUintptr(elem.size, uintptr(size))

  var c *hchan
  // 下面是为buf创建并分配存储空间
  switch {
  case mem == 0:
    // size为0，或者每个元素占用的大小为0
    // 这时为buf分配大小时，只需要分配hchan结构体本身占用的大小即可
    // hchanSize是一个常量，表示空的hchan需要占用的字节大小
    // hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    // raceaddr内部实现为：return unsafe.Pointer(&c.buf)
    c.buf = c.raceaddr()
  case elem.ptrdata == 0:
    // 如果队列中不存在指针，那么每个元素都需要被存储并占用空间，占用大小为前面乘法算出来的mem
    // 同时还要加上hchan本身占用的空间大小，加起来就是整个hchan占用的空间大小
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    // 把buf指针指向空的hchan占用空间大小的末尾
    c.buf = add(unsafe.Pointer(c), hchanSize)
  default:
    // Elements contain pointers.
    // 如果chan中的元素是指针类型的数据，为buf单独开辟mem大小的空间，用来保存所有的数据
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
  }
  // 设置chan的总大小
  c.elemsize = uint16(elem.size)
  // 元素类型
  c.elemtype = elem
  // 环形队列的大小，即用户创建时设置的大小
  c.dataqsiz = uint(size)
  return c
}
```

### 发送数据到channel

概述：

发送数据到channel时，直观的理解是将数据放到chan的环形队列中，不过go做了一些优化：先判断是否有等待接收数据的groutine，如果有，直接将数据发给Groutine，唤醒groutine，就不放入队列中了。当然还有另外一种情况就是：队列如果满了，那就只能放到队列中等待，直到有数据被取走才能发送。

输入：

- chan对象
- 要发送的数据
- 是否阻塞
- 回调函数

输出：无

核心逻辑：

1. 如果recvq不为空，从recvq中取出一个等待接收数据的Groutine，将数据发送给该Groutine
2. 如果recvq为空，才将数据放入buf中
3. 如果buf已满，则将要发送的数据和当前的Groutine打包成Sudog对象放入sendq，并将groutine置为等待状态

#### 有等待接收数据的groutine

[高清地址](https://www.processon.com/view/link/5f69fba607912949af468b0a)

![channel发送-有receive](http://assets.processon.com/chart_image/5f69fba6e401fd61b050061e.png)

#### 无等待接收数据的groutine，环形队列未满

[高清地址](https://www.processon.com/view/link/5f69ff4a7d9c087da1bd5231)

![channel发送-无等待groutine-队列未满](http://assets.processon.com/chart_image/5f69ff4a1e0853769821c1d4.png)

#### 无等待接收数据的groutine，环形队列已满

[高清地址](https://www.processon.com/view/link/5f6a055be0b34d2c46b836dc)

![channel发送-无等待groutine-队列已满](http://assets.processon.com/chart_image/5f6a055ae401fd61b05017c7.png)

#### 发送数据源码

```GO
// ep指向要发送数据的首地址
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

  // 先上锁
  lock(&c.lock)

  // 如果channel已经关闭，抛出错误
  // 下面这个错误经常会遇到，都是对channel使用不当报出来的
  if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("send on closed channel"))
  }

  // 从接收队列中取出元素，如果取到数据，就将数据传过去
  if sg := c.recvq.dequeue(); sg != nil {
    // 调用send方法，将值传过去
    send(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true
  }

  // 走到这里，说明没有等待接收数据的Groutine
  // 如果缓冲区没有满，直接将要发送的数据复制到缓冲区
  if c.qcount < c.dataqsiz {
    // c.sendx是已发送的索引位置，这个方法通过指针偏移找到索引位置
    // 相当于执行c.buf(c.sendx)
    qp := chanbuf(c, c.sendx)
    if raceenabled {
      raceacquire(qp)
      racerelease(qp)
    }

    // 复制数据，内部调用了memmove，是用汇编实现的
    // 通知接收方数据给你了，将接收方协程由等待状态改成可运行状态，
    // 将当前协程加入协程队列，等待被调度
    typedmemmove(c.elemtype, qp, ep)

    // 数据索引前移，如果到了末尾，又从0开始
    c.sendx++
    if c.sendx == c.dataqsiz {
      c.sendx = 0
    }

    // 元素个数加1，释放锁并返回
    c.qcount++
    unlock(&c.lock)
    return true
  }

  // 走到这里，说明缓冲区也写满了
  // 同步非阻塞的情况，直接返回
  if !block {
    unlock(&c.lock)
    return false
  }

  // 以下为同步阻塞的情况
  // 此时会将当前的Groutine以及要发送的数据放入到sendq队列中，并且切换出该Groutine
  gp := getg()
  mysg := acquireSudog()
  mysg.releasetime = 0
  if t0 != 0 {
    mysg.releasetime = -1
  }
  // No stack splits between assigning elem and enqueuing mysg
  // on gp.waiting where copystack can find it.
  mysg.elem = ep
  mysg.waitlink = nil
  mysg.g = gp
  mysg.isSelect = false
  mysg.c = c
  gp.waiting = mysg
  gp.param = nil

  // 将Groutine放入sendq队列
  c.sendq.enqueue(mysg)

  // Groutine转入 waiting 状态，gopark是调度相关的代码
  // 在用户看来，向channel发送数据的代码语句会阻塞
  gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
  KeepAlive(ep)

  // G被唤醒
  if mysg != gp.waiting {
    throw("G waiting list is corrupted")
  }
  gp.waiting = nil
  gp.activeStackChans = false
  if gp.param == nil {
    if c.closed == 0 {
      throw("chansend: spurious wakeup")
    }
    panic(plainError("send on closed channel"))
  }
  gp.param = nil
  if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
  }
  mysg.c = nil

  // G被唤醒，状态改成可执行状态，从这里开始继续执行
  releaseSudog(mysg)
  return true
}
```

#### send函数

```Go
// 要发送的数据ep，被拷贝到接收者sg中，之后sg被唤醒继续执行
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {

  // 拷贝数据
  if sg.elem != nil {
    sendDirect(c.elemtype, sg, ep)
    sg.elem = nil
  }
  gp := sg.g
  unlockf()
  gp.param = unsafe.Pointer(sg)
  if sg.releasetime != 0 {
    sg.releasetime = cputicks()
  }
  // 放入调度队列，等待被调度
  goready(gp, skip+1)
}
```

### 读取数据

概述：

从channel读取数据的流程和发送的类似，基本是发送操作的逆操作。

从channel读取数据时，不是直接去环形队列中去数据，而是先判断是否有等待发送数据的groutine，如果有，直接将groutine出队列，取出数据返回，并唤醒groutine。如果没有等待发送数据的groutine，再从环形队列中取数据。

输入：

- chan对象
- 接收数据的指针
- 是否阻塞

输出：是否接收成功

核心逻辑：

1. 如果有等待发送数据的groutine，从sendq中取出一个等待发送数据的Groutine，取出数据
2. 如果没有等待的groutine，且环形队列中有数据，从队列中取出数据
3. 如果没有等待的groutine，且环形队列中也没有数据，则阻塞该Groutine，并将groutine打包为sudogo加入到recevq等待队列中

#### sendq中有等待的groutine

[高清地址](https://www.processon.com/view/link/5f6a0a4fe401fd61b0501f90)

![发送数据-有等待的groutine](http://assets.processon.com/chart_image/5f6a0a4f07912949af46a4f3.png)

#### sendq中无等待的groutine，队列不为空

[高清地址](https://www.processon.com/view/link/5f6a0b90637689248624570b)

![发送数据-无等待的groutine-队列不为空](http://assets.processon.com/chart_image/5f6a0b901e0853769821d519.png)

#### sendq中无等待的groutine，队列为空

[高清地址](https://www.processon.com/view/link/5f6a0c9d07912949af46a832)

![发送数据-无等待的groutine-队列为空](http://assets.processon.com/chart_image/5f6a0c9d1e0853769821d70f.png)

#### 读取数据源码

```Go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

  // 上锁
  lock(&c.lock)
  // 优先从发送队列中取数据，如果有等待发送数据的groutine,直接从发送数据的协程中取出数据
  if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
  }

  // chan环形队列中如果有有数据
  if c.qcount > 0 {
    // 从接收数据的索引出取出数据
    // 等价于 c.buf[c.recvx]
    qp := chanbuf(c, c.recvx)
    if raceenabled {
      raceacquire(qp)
      racerelease(qp)
    }
    // 将数据拷贝到接收数据的协程
    if ep != nil {
      typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    // 接收数据的索引前移
    c.recvx++
    // 环形队列，如果到了末尾，再从0开始
    if c.recvx == c.dataqsiz {
      c.recvx = 0
    }
    // 发送数据的索引移动位置
    c.qcount--
    unlock(&c.lock)
    return true, true
  }

  // 同步非阻塞，协程直接返回
  if !block {
    unlock(&c.lock)
    return false, false
  }

  // 同步阻塞
  // 如果代码走到这，说明没有任何数据可以获取到，阻塞住协程，并加入channel的接收队列中
  gp := getg()
  mysg := acquireSudog()
  mysg.releasetime = 0
  if t0 != 0 {
    mysg.releasetime = -1
  }
  // No stack splits between assigning elem and enqueuing mysg
  // on gp.waiting where copystack can find it.
  mysg.elem = ep
  mysg.waitlink = nil
  gp.waiting = mysg
  mysg.g = gp
  mysg.isSelect = false
  mysg.c = c
  gp.param = nil

  // 添加到接收队列中
  c.recvq.enqueue(mysg)
  // 调度
  gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

  // someone woke us up
  if mysg != gp.waiting {
    throw("G waiting list is corrupted")
  }
  gp.waiting = nil
  gp.activeStackChans = false
  if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
  }
  closed := gp.param == nil
  gp.param = nil
  mysg.c = nil

  // G被唤醒，从这里继续执行
  releaseSudog(mysg)
  return true, !closed
}
```

### 关闭channel

输入：channel

输出：无

核心流程：

- 设置关闭状态
- 唤醒所有等待读取chanel的协程
- 所有等待写入channel的协程，抛出异常

```Go
func closechan(c *hchan) {
  // channel为空，抛出异常
  if c == nil {
    panic(plainError("close of nil channel"))
  }

  // 上锁
  lock(&c.lock)

  // 如果channel已经被关闭，抛出异常
  if c.closed != 0 {
    unlock(&c.lock)
    panic(plainError("close of closed channel"))
  }

  // 设置关闭状态的值
  c.closed = 1

  // 申明一个存放g的list，把所有的groutine放进来
  // 目的是尽快释放锁，因为队列中可能还有数据需要处理，可能用到锁
  var glist gList

  // release all readers
  // 唤醒所有等待读取chanel数据的协程
  for {
    sg := c.recvq.dequeue()
    // 等待队列处理完毕，退出
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
    gp.param = nil
    if raceenabled {
      raceacquireg(gp, c.raceaddr())
    }
    // 加入临时队列
    glist.push(gp)
  }

  // release all writers (they will panic)
  // 处理所有要发送数据的协程，抛出异常
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
    gp.param = nil
    if raceenabled {
      raceacquireg(gp, c.raceaddr())
    }
    // 加入临时队列
    glist.push(gp)
  }
  unlock(&c.lock)

  // Ready all Gs now that we've dropped the channel lock.
  // 处理临时队列中所有的groutine
  for !glist.empty() {
    gp := glist.pop()
    gp.schedlink = 0

    // 放入调度队列，等待被调度
    goready(gp, 3)
  }
}
```

## 总结

初次使用channel时，感觉很复杂也很神奇，带着这份好奇去研究底层的源码实现，看完之后才发现，它其实没有那么复杂，底层实现逻辑很清晰。本文通过图文并茂的方式整理了底层的逻辑，包括创建channel，发送数据，接收数据等。当然，里面还涉及到调度等知识，后面专门再整理一篇文章加以分析。

## 参考

- https://www.cyhone.com/articles/analysis-of-golang-channel/
- https://github.com/talkgo/night/issues/450
- https://talkgo.org/t/topic/75