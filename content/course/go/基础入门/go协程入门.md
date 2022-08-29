---
title: go协程入门
author: kinnylee
date: 2020-08-17
toc: true
categories:
  - go
tags:
  - go
  - go基础入门
thumbnail: "images/go.png"
---

# 协程在手，说Go就Go

## 协程概述

- 协程：轻量级、用户级线程。
- 协程的调度：
  - 由用户程序进行协作式调度，不像内核进程、线程一样是抢占式调度
- 协程的优势：
  - 占用空间少(只需2k，系统线程为2M)
  - 线程上下文切换成本少
- go：语言层面提出了Groutine的概念，支持协程

### 简单入门

- go语言中通过go关键字来调用

```Go
package main

import (
  "fmt"
  "time"
)

func One(){
  fmt.Println("1")
}

func Two(){
  fmt.Println("2")
}

func main(){
  go One()
  go Two()
}
```

> 上面的代码执行后，并不会如期打印结果，原因在于协程是并发的，协程调用前，主函数已经退出，协程也被销毁了。

```Go
func main(){
  go gorouting()
  // 可通过简单的sleep，让主线程等待协程执行完
  // 但是执行顺序不一定是按照1，2顺序输出
  time.Sleep(5 * 1e9)
}

```

前面的例子，可看到协程使用需要考虑：

- 如何控制协程调用顺序（特别是访问临界资源）
- 如何实现不同协程的并发通讯

实现思路：

- 同步问题：sync同步锁
- 通讯问题：channel

## sync同步锁

go中sync包提供了2个锁，互斥锁sync.Mutex和读写锁sync.RWMutex.我们用互斥锁来解决上述的不同的协程可能同时调度同一个资源的问题

## channel

### 概述

- channel是go语言中一种特殊的数据类型
- 可以通过chanel发送类型化数据，实现协程通讯
- channel是有方向的，包括流入和流出

### 基本使用

```Go
// 普通channel
var ch chan string
ch = make(chan string)
// 只写chanel(流入)
var writeCh chan <- int
// 只读chanel（流出）
var readCh <- chan int
```

### 上述chanel特点

上述channel不带缓冲区，或者说长度为1，有如下特点：

- 同一时间只有一条数据
- 一旦有数据放入，必须被取出才能继续放入

### channel控制执行顺序

- 下面的代码可以保证先执行One，再执行Two

```Go
package main

import (
  "fmt"
  "time"
)

func One(ch chan int){
  fmt.Println("1")
  ch <- 1
}

func Two(ch chan int){
  <- ch
  fmt.Println("2")
}

func main(){
  ch := make(chan int)
  go One(ch)
  go Two(ch)
  time.Sleep(5 * 1e9)
}
```

将main函数本身也看成协程，不用sleep实现同步

```Go
package main

import (
  "fmt"
)

func One(ch chan int){
  fmt.Println("1")
  ch <- 1
}

func Two(ch chan int){
  <- ch
  fmt.Println("2")
}

func main(){
  ch := make(chan int)
  go One(ch)
  <- ch
  //go Two(ch)
  //time.Sleep(5 * 1e9)
}
```

## range

> range用于列表的元素遍历，不仅仅可以遍历普通元素，还可以遍历channel

- 持续的访问数据源并检查channel是否已经关闭，并不高效。go中提供了range关键字。
- range关键字在使用channel的时候，会自动等待channel的动作一直到channel关闭。通俗点将就是channel可以自动开关。

## 带缓存的channel

申明channel时，可以指定长度，长度不为1的channel，可以称为带缓存的channel，数据可以连续写入，直到占满长度为止

```Go
ch = make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
```

### go携程实现生产者-消费者模型

下面的例子是结合协程和range实现的生产者-消费者模型

```Go
package main
import "fmt"

var count = 0

func main() {
  exitChan := make(chan int, 5)
  c := make(chan int, 50)

  // 这里任务数刚好是channel的大小，不会报错
  // 如果改成比chanel大，就会报错。但是如果把这段代码放到consumer后面，就不会报错
  for i := 0; i < 50; i++ {
    c <- i
  }

  for i := 0; i < 5; i++ {
    go Consumer(i, c, exitChan)
  }
  // 这里关闭任务channel，让调用Consumer函数的协程知道：自己的任务完成了，协程可以退出了
  // 如果不关闭，协程将不退出
  close(c)
  for i := 0; i < 5; i++ {
    <-exitChan
  }
  close(exitChan)
  fmt.Println(count)
}
func Consumer(index int, c chan int, exitChan chan int) {
  for target := range c {
    fmt.Printf("no.%d:%d\n", index, target)
    count++
  }
  // 执行close(c)后，协程会走到这里
  exitChan <- index
}
```

## select

在UNIX中，select()函数用来监控一组描述符，该机制常被用于实现高并发的socket服务器程序。Go语言直接在语言级别支持select关键字，用于处理异步IO问题。

首先要明确select做了什么？？

- select可以监听多个channel相关的io操作，当io发送时，触发相应操作
- select中存在着一种轮询机制，select监听进入通道的数据，也可以是通道发送值的时候，监听到相应的行为后就执行case里面的操作
- select默认是阻塞的，只有当监听的channel中有发送或接收可以进行时才会运行，当多个channel都准备好的时候，select是随机的选择一个执行的。

```Go
// 一个实现超时的例子
timeout := make(chan bool, 1)

go func() {
    time.Sleep(1e9)
    timeout <- true
}()

switch {
    case <- ch:
    // 从ch中读取到数据

    case <- timeout:
    // 没有从ch中读取到数据，但从timeout中读取到了数据
}
```

## 协程调度

go中的runtime包，提供了调度器的功能，runtime包提供了以下几个方法：

Gosched：让当前线程让出 cpu 以让其它线程运行,它不会挂起当前线程，因此当前线程未来会继续执行
NumCPU：返回当前系统的 CPU 核数量
GOMAXPROCS：设置最大的可同时使用的 CPU 核数
Goexit：退出当前 goroutine(但是defer语句会照常执行)
NumGoroutine：返回正在执行和排队的任务总数
GOOS：目标操作系统