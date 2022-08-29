---
title: go协程原理
author: kinnylee
date: 2020-08-25
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

# 协程原理

## 并发编程模型

- 从内存的角度看，并行计算只有两种：共享内存、消息通讯。
- 目的是解决多线程的数据一致性

## CSP模型

- go语言的并发特性是由1978年发布的CSP理论演化而来，另外一个知名的CSP实现语言是Eralng，大名鼎鼎的Rabbitmq就是用erlang实现的。
- CSP是Communicating Sequential Processes(顺序通信进程)的缩写
- CSP模型用于描述两个独立的并发实体通过共享的通讯 channel(管道)进行通信的并发模型。

### go对CSP模型的实现

- go底层使用goroutine做为并发实体，goroutine非常轻量级可以创建几十万个实体。
- 实体间通过 channel 继续匿名消息传递使之解耦，在语言层面实现了自动调度，这样屏蔽了很多内部细节
- 对外提供简单的语法关键字，大大简化了并发编程的思维转换和管理线程的复杂性。
- 通过GMP调度模型实现语言层面的调度

## 理解调度

- 操作系统不认识goroutine，只认识线程
- 调度是由golang的runtime实现的
- 最开始只有GM模型，Groutine都在全局队列，因为性能问题引入了P的概念，将G放入P本地
- 操作系统为什么调度：一个cpu只有一组寄存器，不同线程要用这组寄存器，只能换着使用。本质是寄存器、栈帧的保存和恢复
- Go调度的理解：寻找合适的

## 调度发展历程

1.14之前：不支持抢占式，存在调度缺陷问题


源码位置：src/runtime/proc.go

```Go
// 获取sudog
func acquireSudog() *sudog {
  // Delicate dance: the semaphore implementation calls
  // acquireSudog, acquireSudog calls new(sudog),
  // new calls malloc, malloc can call the garbage collector,
  // and the garbage collector calls the semaphore implementation
  // in stopTheWorld.
  // Break the cycle by doing acquirem/releasem around new(sudog).
  // The acquirem/releasem increments m.locks during new(sudog),
  // which keeps the garbage collector from being invoked.
  // 获取当前G对应的M
  mp := acquirem()
  // 获取M对应的P
  pp := mp.p.ptr()
  // 如果P本地的G列表为空
  if len(pp.sudogcache) == 0 {
    // 尝试从调度器的全局队列获取G，所以需要上锁
    lock(&sched.sudoglock)
    // First, try to grab a batch from central cache.
    for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
      s := sched.sudogcache
      sched.sudogcache = s.next
      s.next = nil
      pp.sudogcache = append(pp.sudogcache, s)
    }
    unlock(&sched.sudoglock)
    // If the central cache is empty, allocate a new one.
    // 如果调度器也获取不到G，生成一个新的G，并放到本地G列表中
    if len(pp.sudogcache) == 0 {
      pp.sudogcache = append(pp.sudogcache, new(sudog))
    }
  }
  n := len(pp.sudogcache)
  s := pp.sudogcache[n-1]
  pp.sudogcache[n-1] = nil
  pp.sudogcache = pp.sudogcache[:n-1]
  if s.elem != nil {
    throw("acquireSudog: found s.elem != nil in cache")
  }
  releasem(mp)
  return s
}

// 释放sudog
func releaseSudog(s *sudog) {
  if s.elem != nil {
    throw("runtime: sudog with non-nil elem")
  }
  if s.isSelect {
    throw("runtime: sudog with non-false isSelect")
  }
  if s.next != nil {
    throw("runtime: sudog with non-nil next")
  }
  if s.prev != nil {
    throw("runtime: sudog with non-nil prev")
  }
  if s.waitlink != nil {
    throw("runtime: sudog with non-nil waitlink")
  }
  if s.c != nil {
    throw("runtime: sudog with non-nil c")
  }
  gp := getg()
  if gp.param != nil {
    throw("runtime: releaseSudog with non-nil gp.param")
  }
  mp := acquirem() // avoid rescheduling to another P
  pp := mp.p.ptr()
  // 如果本地G列表已满
  if len(pp.sudogcache) == cap(pp.sudogcache) {
    // Transfer half of local cache to the central cache.
    var first, last *sudog
    // 将本地G列表的一半数据放到全局队列
    for len(pp.sudogcache) > cap(pp.sudogcache)/2 {
      n := len(pp.sudogcache)
      p := pp.sudogcache[n-1]
      pp.sudogcache[n-1] = nil
      pp.sudogcache = pp.sudogcache[:n-1]
      if first == nil {
        first = p
      } else {
        last.next = p
      }
      last = p
    }
    lock(&sched.sudoglock)
    last.next = sched.sudogcache
    sched.sudogcache = first
    unlock(&sched.sudoglock)
  }
  // 将需要释放被调度的G放到P本地队列
  pp.sudogcache = append(pp.sudogcache, s)
  releasem(mp)
}
```

## 并发调度原理

go语言的线程MPG调度模型

M：machine，一个M直接关联一个内核线程
P：processor，代表M所需的上下文环境，也是处理用户级代码逻辑的处理器
G：groutine，协程，本质上是一种轻量级的线程

![线程MPG模型](https://i6448038.github.io/img/csp/GMPrelation.png)

- 一个M对应一个内核线程，也会连接一个上下文P
- 一个上下文P，相当于一个处理器。
- p的数量是在启动时被设置为环境变量`GOMAXPROCS`，意味着运行的线程数量是固定的
- 一个上下文连接一个或多个groutine
- p正在执行的Groutine为蓝色的，处于待执行的Groutine为灰色（保存在一个队列中）

### G 数据结构

- G对应Groutine

```Go
// 源码位置：src/runtime/runtime2.go
type g struct {
  // 栈变量，保存运行时的堆栈内存信息
  // 内部包含两个指针：lo和hi，分别指向栈的上下界
  stack       stack   // offset known to runtime/cgo
  // 下面两个变量
  stackguard0 uintptr // offset known to liblink
  stackguard1 uintptr // offset known to liblink
  ...
  // 当前的M，即在哪个线程上执行任务
  m            *m      // current m; offset known to arm liblink

  // 存放g的上下文信息（寄存器信息），g被停止调度时，会将上下文信息放到这里，唤醒后可以继续调度
  // 非常重要的数据信息，替换内核的上下文切换
  sched        gobuf
  syscallsp    uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
  syscallpc    uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
  stktopsp     uintptr        // expected sp at top of stack, to check in traceback
  // 用于参数传递
  param        unsafe.Pointer // passed parameter on wakeup
  atomicstatus uint32
  stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
  // g的id
  goid         int64
  schedlink    guintptr
  // 被阻塞的时间
  waitsince    int64      // approx time when the g become blocked
  waitreason   waitReason // if status==Gwaiting

  preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
  preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
  preemptShrink bool // shrink stack at synchronous safe point

  // asyncSafePoint is set if g is stopped at an asynchronous
  // safe point. This means there are frames on the stack
  // without precise pointer information.
  asyncSafePoint bool

  paniconfault bool // panic (instead of crash) on unexpected fault address
  gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
  throwsplit   bool // must not split stack
  // activeStackChans indicates that there are unlocked channels
  // pointing into this goroutine's stack. If true, stack
  // copying needs to acquire channel locks to protect these
  // areas of the stack.
  activeStackChans bool

  raceignore     int8     // ignore race detection events
  sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
  sysexitticks   int64    // cputicks when syscall has returned (for tracing)
  traceseq       uint64   // trace event sequencer
  tracelastp     puintptr // last P emitted an event for this goroutine

  // 被锁定只在这个M上执行
  lockedm        muintptr
  sig            uint32
  writebuf       []byte
  sigcode0       uintptr
  sigcode1       uintptr
  sigpc          uintptr
  // 创建groutine的入口指令
  gopc           uintptr         // pc of go statement that created this goroutine
  ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
  // 被执行函数
  startpc        uintptr         // pc of goroutine function
  racectx        uintptr
  waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
  cgoCtxt        []uintptr      // cgo traceback context
  labels         unsafe.Pointer // profiler labels
   // 缓存的定时器
  timer          *timer         // cached timer for time.Sleep
  selectDone     uint32         // are we participating in a select and did someone win the race?

  // Per-G GC state

  // gcAssistBytes is this G's GC assist credit in terms of
  // bytes allocated. If this is positive, then the G has credit
  // to allocate gcAssistBytes bytes without assisting. If this
  // is negative, then the G must correct this by performing
  // scan work. We track this in bytes to make it fast to update
  // and check for debt in the malloc hot path. The assist ratio
  // determines how this corresponds to scan work debt.
  gcAssistBytes int64
}

// 保存栈内存地址的数据结构
type stack struct {
  // 栈顶，低地址
  lo uintptr
  // 栈底，高地址
  hi uintptr
}

// 保存上下文寄存器信息的数据结构
// 核心就是保存寄存器信息
type gobuf struct {
  // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
  //
  // ctxt is unusual with respect to GC: it may be a
  // heap-allocated funcval, so GC needs to track it, but it
  // needs to be set and cleared from assembly, where it's
  // difficult to have write barriers. However, ctxt is really a
  // saved, live register, and we only ever exchange it between
  // the real register and the gobuf. Hence, we treat it as a
  // root during stack scanning, which means assembly that saves
  // and restores it doesn't need write barriers. It's still
  // typed as a pointer so that any other writes from Go get
  // write barriers.
  // 执行sp寄存器
  sp   uintptr
  // 执行pc寄存器
  pc   uintptr
  // 指向G本身
  g    guintptr
  ctxt unsafe.Pointer
  ret  sys.Uintreg
  lr   uintptr
  bp   uintptr // for GOEXPERIMENT=framepointer
}
```

### P 数据结构

- P是一个抽象的概念，并不是真正的物理cpu
- 当p有任务时，需要创建或唤醒一个系统线程来执行它队列里的任务，所以P和M需要进行绑定，构成一个可执行单元
- 可通过GOMAXPROCS限制同时执行用户级任务的操作系统线
- GOMAXPROCS默认为系统核数

#### P队列

P有两种队列，本地队列和全局队列：

- 本地队列：当前P的队列，本地队列是Lock-Free，没有数据竞争问题，无需加锁处理，可以提升处理速度。
- 全局队列：全局队列为了保证多个P之间任务的平衡。所有M共享P全局队列，为保证数据竞争问题，需要加锁处理。相比本地队列处理速度要低于全局队列

```go
// 源码位置：src/runtime/runtime2.go
type p struct {
  // 在所有p列表中的索引
  id          int32
  status      uint32 // one of pidle/prunning/...
  link        puintptr
  // 每调度一次加一
  schedtick   uint32     // incremented on every scheduler call
  // 每次系统调用加一
  syscalltick uint32     // incremented on every system call
  // 用于 sysmon 线程记录被监控p的系统调用时间和运行时间
  sysmontick  sysmontick // last tick observed by sysmon
  // 指向绑定的m，p是idle的话，该指针为空
  m           muintptr   // back-link to associated m (nil if idle)
  mcache      *mcache
  pcache      pageCache
  raceprocctx uintptr

  deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
  deferpoolbuf [5][32]*_defer

  // Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
  goidcache    uint64
  goidcacheend uint64

  // Queue of runnable goroutines. Accessed without lock.
  // 可运行的Groutine队列，队列头
  runqhead uint32
  // 可运行的Groutine队列，队列尾
  runqtail uint32
  // 保存q的数组，只能存放256个，超出的要放到全局队列
  runq     [256]guintptr
  // runnext, if non-nil, is a runnable G that was ready'd by
  // the current G and should be run next instead of what's in
  // runq if there's time remaining in the running G's time
  // slice. It will inherit the time left in the current time
  // slice. If a set of goroutines is locked in a
  // communicate-and-wait pattern, this schedules that set as a
  // unit and eliminates the (potentially large) scheduling
  // latency that otherwise arises from adding the ready'd
  // goroutines to the end of the run queue.
  // 下一个运行的g
  runnext guintptr

  // Available G's (status == Gdead)
  // 空闲的g
  gFree struct {
    gList
    n int32
  }

  sudogcache []*sudog
  sudogbuf   [128]*sudog

  // Cache of mspan objects from the heap.
  mspancache struct {
    // We need an explicit length here because this field is used
    // in allocation codepaths where write barriers are not allowed,
    // and eliminating the write barrier/keeping it eliminated from
    // slice updates is tricky, moreso than just managing the length
    // ourselves.
    len int
    buf [128]*mspan
  }

  tracebuf traceBufPtr

  // traceSweep indicates the sweep events should be traced.
  // This is used to defer the sweep start event until a span
  // has actually been swept.
  traceSweep bool
  // traceSwept and traceReclaimed track the number of bytes
  // swept and reclaimed by sweeping in the current sweep loop.
  traceSwept, traceReclaimed uintptr

  palloc persistentAlloc // per-P to avoid mutex

  _ uint32 // Alignment for atomic fields below

  // The when field of the first entry on the timer heap.
  // This is updated using atomic functions.
  // This is 0 if the timer heap is empty.
  timer0When uint64

  // Per-P GC state
  gcAssistTime         int64    // Nanoseconds in assistAlloc
  gcFractionalMarkTime int64    // Nanoseconds in fractional mark worker (atomic)
  gcBgMarkWorker       guintptr // (atomic)
  gcMarkWorkerMode     gcMarkWorkerMode

  // gcMarkWorkerStartTime is the nanotime() at which this mark
  // worker started.
  gcMarkWorkerStartTime int64

  // gcw is this P's GC work buffer cache. The work buffer is
  // filled by write barriers, drained by mutator assists, and
  // disposed on certain GC state transitions.
  gcw gcWork

  // wbBuf is this P's GC write barrier buffer.
  //
  // TODO: Consider caching this in the running G.
  wbBuf wbBuf

  runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

  // Lock for timers. We normally access the timers while running
  // on this P, but the scheduler can also do it from a different P.
  timersLock mutex

  // Actions to take at some time. This is used to implement the
  // standard library's time package.
  // Must hold timersLock to access.
  timers []*timer

  // Number of timers in P's heap.
  // Modified using atomic instructions.
  numTimers uint32

  // Number of timerModifiedEarlier timers on P's heap.
  // This should only be modified while holding timersLock,
  // or while the timer status is in a transient state
  // such as timerModifying.
  adjustTimers uint32

  // Number of timerDeleted timers in P's heap.
  // Modified using atomic instructions.
  deletedTimers uint32

  // Race context used while executing timer functions.
  timerRaceCtx uintptr

  // preempt is set to indicate that this P should be enter the
  // scheduler ASAP (regardless of what G is running on it).
  preempt bool

  pad cpu.CacheLinePad
}
```

### M 数据结构

- M代表一个线程，每次创建一个M时，都有一个底层线程创建
- 所有的G任务，都是在M上执行
- 两个特殊的G
  - g0:带有调度栈的Groutine，是一个比较特殊的Groutine, g0的栈是对应的M对应的线程的真实内核栈
  - curg: 结构体M当前绑定的结构体G

```Go
// 源码位置：src/runtime/runtime2.go
type m struct {
  // 带有调度栈的Groutine，是一个比较特殊的Groutine
  // g0的栈是对应的M对应的线程的真实内核栈
  g0      *g     // goroutine with scheduling stack
  morebuf gobuf  // gobuf arg to morestack
  divmod  uint32 // div/mod denominator for arm - known to liblink

  // Fields not known to debuggers.
  procid        uint64       // for debuggers, but offset not hard-coded
  gsignal       *g           // signal-handling g
  goSigStack    gsignalStack // Go-allocated signal handling stack
  sigmask       sigset       // storage for saved signal mask
  // 线程本地存储，实现m与工作线程的绑定
  tls           [6]uintptr   // thread-local storage (for x86 extern register)
  mstartfn      func()
  // 当前绑定的、正在运行的groutine
  curg          *g       // current running goroutine
  caughtsig     guintptr // goroutine running during fatal signal
  // 关联p，需要执行的代码
  p             puintptr // attached p for executing go code (nil if not executing go code)
  nextp         puintptr
  oldp          puintptr // the p that was attached before executing a syscall
  id            int64
  mallocing     int32
  throwing      int32
  // 如果不为空字符串，保持curg始终在这个m上运行
  preemptoff    string // if != "", keep curg running on this m
  locks         int32
  dying         int32
  profilehz     int32
  // 自旋标志，true表示正在从其他线程偷g
  spinning      bool // m is out of work and is actively looking for work
  // m阻塞在note上
  blocked       bool // m is blocked on a note
  newSigstack   bool // minit on C thread called sigaltstack
  printlock     int8
  // 正在执行cgo
  incgo         bool   // m is executing a cgo call
  freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
  fastrand      [2]uint32
  needextram    bool
  traceback     uint8
  // cgo调用总次数
  ncgocall      uint64      // number of cgo calls in total
  ncgo          int32       // number of cgo calls currently in progress
  cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
  cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
  // 没有groutine需要执行，工作线程睡眠在这个成员上
  // 其它线程通过这个 park 唤醒该工作线程
  park          note
  // 所有工作线程链表
  alllink       *m // on allm
  schedlink     muintptr
  mcache        *mcache
  lockedg       guintptr
  createstack   [32]uintptr // stack that created this thread.
  lockedExt     uint32      // tracking for external LockOSThread
  lockedInt     uint32      // tracking for internal lockOSThread
  // 正在等待锁的下一个m
  nextwaitm     muintptr    // next m waiting for lock
  waitunlockf   func(*g, unsafe.Pointer) bool
  waitlock      unsafe.Pointer
  waittraceev   byte
  waittraceskip int
  startingtrace bool
  syscalltick   uint32
  freelink      *m // on sched.freem

  // these are here because they are too large to be on the stack
  // of low-level NOSPLIT functions.
  libcall   libcall
  libcallpc uintptr // for cpu profiler
  libcallsp uintptr
  libcallg  guintptr
  syscall   libcall // stores syscall parameters on windows

  // 寄存器信息，用于恢复现场
  vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
  vdsoPC uintptr // PC for traceback while in VDSO call

  // preemptGen counts the number of completed preemption
  // signals. This is used to detect when a preemption is
  // requested, but fails. Accessed atomically.
  preemptGen uint32

  // Whether this is a pending preemption signal on this M.
  // Accessed atomically.
  signalPending uint32

  dlogPerM

  mOS
}
```

## go func()到底做了什么

> 源码中的 _StackMin 指明了栈的大小为2k

- 首选创建G对象，G对象保存到P本地队列（超出256）或者全局队列
- P此时去唤醒一个M。
- P继续执行
- M寻找是否有可以执行的P，如果有则将G挂到P上
- 接下来M执行调度循环
  - 调用G
  - 执行
  - 清理现场
  - 继续找新的G
- M执行过程中，会进行上下文切换，切换时将现场寄存器信息保存到G对象的成员里
- G下次被调度到执行时，从自身对象上恢复寄存器信息继续执行

```go
// 源码位置：sr/runtime/proc.go
func newproc(siz int32, fn *funcval) {
  argp := add(unsafe.Pointer(&fn), sys.PtrSize)
  gp := getg()
  pc := getcallerpc()
  systemstack(func() {
    // 内部调用了newproc1函数
    newproc1(fn, argp, siz, gp, pc)
  })
}

// _StackMin = 2048 默认栈大小为2k
// newg = malg(_StackMin) 初始化栈大小
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
  _g_ := getg()

  if fn == nil {
    _g_.m.throwing = -1 // do not dump full stacks
    throw("go of nil func value")
  }
  acquirem() // disable preemption because it can be holding p in a local var
  siz := narg
  siz = (siz + 7) &^ 7

  // We could allocate a larger initial stack if necessary.
  // Not worth it: this is almost always an error.
  // 4*sizeof(uintreg): extra space added below
  // sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
  if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
    throw("newproc: function arguments too large for new goroutine")
  }

  _p_ := _g_.m.p.ptr()
  newg := gfget(_p_)
  // 初始化栈大小
  if newg == nil {
    newg = malg(_StackMin)
    casgstatus(newg, _Gidle, _Gdead)
    allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
  }
  if newg.stack.hi == 0 {
    throw("newproc1: newg missing stack")
  }

  if readgstatus(newg) != _Gdead {
    throw("newproc1: new g is not Gdead")
  }

  totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
  totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
  sp := newg.stack.hi - totalSize
  spArg := sp
  if usesLR {
    // caller's LR
    *(*uintptr)(unsafe.Pointer(sp)) = 0
    prepGoExitFrame(sp)
    spArg += sys.MinFrameSize
  }
  if narg > 0 {
    memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
    // This is a stack-to-stack copy. If write barriers
    // are enabled and the source stack is grey (the
    // destination is always black), then perform a
    // barrier copy. We do this *after* the memmove
    // because the destination stack may have garbage on
    // it.
    if writeBarrier.needed && !_g_.m.curg.gcscandone {
      f := findfunc(fn.fn)
      stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
      if stkmap.nbit > 0 {
        // We're in the prologue, so it's always stack map index 0.
        bv := stackmapdata(stkmap, 0)
        bulkBarrierBitmap(spArg, spArg, uintptr(bv.n)*sys.PtrSize, 0, bv.bytedata)
      }
    }
  }

  memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
  newg.sched.sp = sp
  newg.stktopsp = sp
  newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
  newg.sched.g = guintptr(unsafe.Pointer(newg))
  gostartcallfn(&newg.sched, fn)
  newg.gopc = callerpc
  newg.ancestors = saveAncestors(callergp)
  newg.startpc = fn.fn
  if _g_.m.curg != nil {
    newg.labels = _g_.m.curg.labels
  }
  if isSystemGoroutine(newg, false) {
    atomic.Xadd(&sched.ngsys, +1)
  }
  casgstatus(newg, _Gdead, _Grunnable)

  if _p_.goidcache == _p_.goidcacheend {
    // Sched.goidgen is the last allocated id,
    // this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
    // At startup sched.goidgen=0, so main goroutine receives goid=1.
    _p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
    _p_.goidcache -= _GoidCacheBatch - 1
    _p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
  }
  newg.goid = int64(_p_.goidcache)
  _p_.goidcache++
  if raceenabled {
    newg.racectx = racegostart(callerpc)
  }
  if trace.enabled {
    traceGoCreate(newg, newg.startpc)
  }
  runqput(_p_, newg, true)

  if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
    wakep()
  }
  releasem(_g_.m)
}
```

## 全局调度器

- 通过lock来全局管控一些全局操作

```Go
type schedt struct {
  // accessed atomically. keep at top to ensure alignment on 32-bit systems.
  // 需要原子访问
  goidgen   uint64
  lastpoll  uint64 // time of last network poll, 0 if currently polling
  pollUntil uint64 // time to which current poll is sleeping

  lock mutex

  // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
  // sure to call checkdead().
	// 空闲的工作线程组成的列表
  midle        muintptr // idle m's waiting for work
  // 空闲的工作线程数量
  nmidle       int32    // number of idle m's waiting for work
  // 空闲的、且被lock的m计数
  nmidlelocked int32    // number of locked m's waiting for work
  // 创建的工作线程的数量
  mnext        int64    // number of m's that have been created and next M ID
  // 最多能创建的工作线程数量
  maxmcount    int32    // maximum number of m's allowed (or die)
  nmsys        int32    // number of system m's not counted for deadlock
  nmfreed      int64    // cumulative number of freed m's

  // 系统中所有Groutine的数量，自动更新
  ngsys uint32 // number of system goroutines; updated atomically
	// 空闲的p结构体对象组成的链表
  pidle      puintptr // idle p's
  // 空闲的p结构体对象数量
  npidle     uint32
  nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

  // Global runnable queue.
  // 全局可运行的G队列
  runq     gQueue
  runqsize int32

  // disable controls selective disabling of the scheduler.
  //
  // Use schedEnableUser to control this.
  //
  // disable is protected by sched.lock.
  disable struct {
    // user disables scheduling of user goroutines.
    user     bool
    runnable gQueue // pending runnable Gs
    n        int32  // length of runnable
  }

  // Global cache of dead G's.
  // 全局的已退出goroutine，缓存下来，避免每次都要重新分配内存
  gFree struct {
    lock    mutex
    stack   gList // Gs with stacks
    noStack gList // Gs without stacks
    n       int32
  }

  // Central cache of sudog structs.
  sudoglock  mutex
  sudogcache *sudog

  // Central pool of available defer structs of different sizes.
  deferlock mutex
  deferpool [5]*_defer

  // freem is the list of m's waiting to be freed when their
  // m.exited is set. Linked through m.freelink.
  freem *m

  gcwaiting  uint32 // gc is waiting to run
  stopwait   int32
  stopnote   note
  sysmonwait uint32
  sysmonnote note

  // safepointFn should be called on each P at the next GC
  // safepoint if p.runSafePointFn is set.
  safePointFn   func(*p)
  safePointWait int32
  safePointNote note

  profilehz int32 // cpu profiling rate

  procresizetime int64 // nanotime() of last change to gomaxprocs
  totaltime      int64 // ∫gomaxprocs dt up to procresizetime
}

type gQueue struct {
	head guintptr
	tail guintptr
}

// 全局变量
// src/runtime/proc.go
var (
  // 进程的主线程
	m0           m
  // m0的g0
	g0           g
	raceprocctx0 uintptr
)

// 全局变量
// src/runtime/runtime2.go
var (
  // 所有g的长度
	allglen    uintptr
  // 保存所有的m
	allm       *m
  // 保存所有的p
	allp       []*p  // len(allp) == gomaxprocs; may change at safe points, otherwise immutable
	allpLock   mutex // Protects P-less reads of allp and all writes
  // p的最大值，默认等于ncpu
	gomaxprocs int32
  // 程序启动时，osinit函数获取该值
	ncpu       int32
	forcegc    forcegcstate
  // 全局调度器
	sched      schedt
	newprocs   int32

	// Information about what cpu features are available.
	// Packages outside the runtime should not use these
	// as they are not an external api.
	// Set on startup in asm_{386,amd64}.s
	processorVersionInfo uint32
	isIntel              bool
	lfenceBeforeRdtsc    bool

	goarm                uint8 // set by cmd/link on arm systems
	framepointer_enabled bool  // set by cmd/link
)
```

## 参考

- https://github.com/talkgo/night/issues/450