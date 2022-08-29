---
title: go进程启动分析
author: kinnylee
date: 2020-08-28
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

# Go进程启动分析

- 我们都知道，go语言执行的入口为：main包下面的main()函数，但是底层指令真的是从这开始执行的吗？
- 这一篇内容用到上一篇《go协程原理》的GMP模型

## Go进程启动概述

- main包的main函数并不是go语言的入口函数，入口函数是在asm_amd64.s中定义的
- main包的main函数是由runtime.main函数启动的
- go进程启动后，会调用runtime·rt0_go来执行程序的初始化和启动系统调度

go进程启动的四大步骤

- runtime.osinit：获取系统cpu个数
- runtime.schedinit：初始化调度系统，p初始化，m0和某个绑定
- runtime.newproc：新建groutine，执行runtime.main，建好后插入p的本地队列
- runtime.mstart：启动m，进入启动调度系统

## 概念介绍

### m0

表示进程启动的第一个线程，也叫主线程。它是进程启动通过汇编复制的，是个全局变量

### g0

- 每个m都有一个g0，因为每个线程都有一个系统堆栈。
- 和其他g的区别是栈的区别。
- g0上的栈是系统分配的，在linux上默认大小为8M，不能扩展也不能缩小。而普通g默认2k，可扩展
- g0上没有任何任务函数，也没有任何状态，它不能被调度程序抢占。
- 调度是在g0上跑的

源码位置：src/runtime/proc.go

```Go
// 全局变量，赋值是汇编实现的
var (
  // 主线程
  m0           m
  // 和m0绑定的g0，也可以理解成m0的堆栈
  g0           g
  raceprocctx0 uintptr
)
```

### 汇编入口

源码位置：src/runtime/asm_arm64.s

```armasm
TEXT runtime·rt0_go(SB),NOSPLIT,$0

  // 进程启动时的主线程
  // 当前栈和资源保存在全局变量runtime.g0中
  MOVD	$runtime·g0(SB), g

  // 当前线程保存在m0
  MOVD	$runtime·m0(SB), R0

  // g0绑定到m0
  MOVD	g, m_g0(R0)

  // m0绑定到g0
  MOVD	R0, g_m(g)

  // os初始化，获取cpu数量
  BL	runtime·osinit(SB)

  // 调度器初始化
  BL	runtime·schedinit(SB)

  // 这里进入runtime.main，开始执行用户程序
  MOVD	$runtime·mainPC(SB), R0		// entry

  // runtime.newproc启动一个groutine
  BL	runtime·newproc(SB)
  // 启动线程，启动调度系统
  BL  runtime·mstart(SB)
```

## Go进程启动源码解析

### osinit

- 源码位置：src/runtime/os_linux.go

```Go
func osinit() {
  // 获取cpu数量
  ncpu = getproccount()
  physHugePageSize = getHugePageSize()
  osArchInit()
}
```

### scheint

```Go
// 调度系统的初始化
// 进行P的初始化
// 也会把M0和某个P绑定
func schedinit() {
  // raceinit must be the first call to race detector.
  // In particular, it must be done before mallocinit below calls racemapshadow.
  _g_ := getg()
  if raceenabled {
    _g_.racectx, raceprocctx0 = raceinit()
  }

  sched.maxmcount = 10000

  tracebackinit()
  moduledataverify()
  stackinit()
  mallocinit()
  fastrandinit() // must run before mcommoninit
  mcommoninit(_g_.m)
  cpuinit()       // must run before alginit
  alginit()       // maps must not be used before this call
  modulesinit()   // provides activeModules
  typelinksinit() // uses maps, activeModules
  itabsinit()     // uses activeModules

  msigsave(_g_.m)
  initSigmask = _g_.m.sigmask

  goargs()
  goenvs()
  parsedebugvars()
  gcinit()

  sched.lastpoll = uint64(nanotime())

  // 确认P的个数，默认为CPU个数，可以通过 GOMAXPROCS 环境变量更改
  procs := ncpu
  if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    procs = n
  }

  // 一次性分配procs个P
  if procresize(procs) != nil {
    throw("unknown runnable goroutine during bootstrap")
  }

  // For cgocheck > 1, we turn on the write barrier at all times
  // and check all pointer writes. We can't do this until after
  // procresize because the write barrier needs a P.
  if debug.cgocheck > 1 {
    writeBarrier.cgo = true
    writeBarrier.enabled = true
    for _, p := range allp {
      p.wbBuf.reset()
    }
  }

  if buildVersion == "" {
    // Condition should never trigger. This code just serves
    // to ensure runtime·buildVersion is kept in the resulting binary.
    buildVersion = "unknown"
  }
  if len(modinfo) == 1 {
    // Condition should never trigger. This code just serves
    // to ensure runtime·modinfo is kept in the resulting binary.
    modinfo = ""
  }
}
```

### mstart

```Go
// 启动线程，并且启动调度系统
func mstart() {
  // 这里获取的是g0，在系统堆栈
  _g_ := getg()

  osStack := _g_.stack.lo == 0
  if osStack {
    // Initialize stack bounds from system stack.
    // Cgo may have left stack size in stack.hi.
    // minit may update the stack bounds.
    size := _g_.stack.hi
    if size == 0 {
      size = 8192 * sys.StackGuardMultiplier
    }
    _g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
    _g_.stack.lo = _g_.stack.hi - size + 1024
  }
  // Initialize stack guard so that we can start calling regular
  // Go code.
  _g_.stackguard0 = _g_.stack.lo + _StackGuard
  // This is the g0, so we can also call go:systemstack
  // functions, which check stackguard1.
  _g_.stackguard1 = _g_.stackguard0
  mstart1()

  // Exit this thread.
  switch GOOS {
  case "windows", "solaris", "illumos", "plan9", "darwin", "aix":
    // Windows, Solaris, illumos, Darwin, AIX and Plan 9 always system-allocate
    // the stack, but put it in _g_.stack before mstart,
    // so the logic above hasn't set osStack yet.
    osStack = true
  }
  mexit(osStack)
}

func mstart1() {
  _g_ := getg()

  // 确保g是系统栈上的g0，调度器只在g0上执行
  if _g_ != _g_.m.g0 {
    throw("bad runtime·mstart")
  }

  // Record the caller for use as the top of stack in mcall and
  // for terminating the thread.
  // We're never coming back to mstart1 after we call schedule,
  // so other calls can reuse the current frame.
  save(getcallerpc(), getcallersp())
  asminit()

  // 初始m
  minit()

  // Install signal handlers; after minit so that minit can
  // prepare the thread to be able to handle the signals.
  // 如果当前g的m是m0，执行mstartm0
  if _g_.m == &m0 {
    // 对于初始m，需要一些特殊处理
    mstartm0()
  }

  // 如果有m的起始函数执行，先执行它
  if fn := _g_.m.mstartfn; fn != nil {
    fn()
  }

  if _g_.m != &m0 {
    // 如果不是m0，需要绑定p
    acquirep(_g_.m.nextp.ptr())
    _g_.m.nextp = 0
  }

  // 开始进入调度
  schedule()
}
```

### schedule

调度的本质是：尽力找可以运行的G，然后运行G上的任务函数

具体流程包括：

- 如果GC需要STW，就休眠M
- 每隔61次从全局队列获取G，避免全局队列的g被饿死
- 从p的本地队列获取G
- 调用findrunnable 找G，找不到的话就将M休眠，等待唤醒
- 找到G后，调用execute去执行G

> main函数也是放入到G中的

```Go
func schedule() {
  _g_ := getg()

top:
  pp := _g_.m.p.ptr()
  pp.preempt = false

  // GC(垃圾回收)需要STW（stop the world)，休眠当前m
  if sched.gcwaiting != 0 {
    gcstopm()
    goto top
  }
  if pp.runSafePointFn != 0 {
    runSafePointFn()
  }

  // Sanity check: if we are spinning, the run queue should be empty.
  // Check this before calling checkTimers, as that might call
  // goready to put a ready goroutine on the local run queue.
  if _g_.m.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
    throw("schedule: spinning with local work")
  }

  checkTimers(pp, 0)

  var gp *g
  var inheritTime bool

  // Normal goroutines will check for need to wakeP in ready,
  // but GCworkers and tracereaders will not, so the check must
  // be done here instead.
  tryWakeP := false
  if trace.enabled || trace.shutdown {
    gp = traceReader()
    if gp != nil {
      casgstatus(gp, _Gwaiting, _Grunnable)
      traceGoUnpark(gp, 0)
      tryWakeP = true
    }
  }
  if gp == nil && gcBlackenEnabled != 0 {
    gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
    tryWakeP = tryWakeP || gp != nil
  }
  if gp == nil {
    // Check the global runnable queue once in a while to ensure fairness.
    // Otherwise two goroutines can completely occupy the local runqueue
    // by constantly respawning each other.
    // 每隔61次调度，从全局队列获取G
    if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
      lock(&sched.lock)
      gp = globrunqget(_g_.m.p.ptr(), 1)
      unlock(&sched.lock)
    }
  }
  if gp == nil {
    // 从P的本地队列获取G
    gp, inheritTime = runqget(_g_.m.p.ptr())
    // We can see gp != nil here even if the M is spinning,
    // if checkTimers added a local goroutine via goready.
  }
  if gp == nil {
    // 阻塞住，直到找到G
    gp, inheritTime = findrunnable() // blocks until work is available
  }

  // This thread is going to run a goroutine and is not spinning anymore,
  // so if it was marked as spinning we need to reset it now and potentially
  // start a new spinning M.
  if _g_.m.spinning {
    resetspinning()
  }

  if sched.disable.user && !schedEnabled(gp) {
    // Scheduling of this goroutine is disabled. Put it on
    // the list of pending runnable goroutines for when we
    // re-enable user scheduling and look again.
    lock(&sched.lock)
    if schedEnabled(gp) {
      // Something re-enabled scheduling while we
      // were acquiring the lock.
      unlock(&sched.lock)
    } else {
      sched.disable.runnable.pushBack(gp)
      sched.disable.n++
      unlock(&sched.lock)
      goto top
    }
  }

  // If about to schedule a not-normal goroutine (a GCworker or tracereader),
  // wake a P if there is one.
  if tryWakeP {
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
      wakep()
    }
  }
  if gp.lockedm != 0 {
    // Hands off own p to the locked m,
    // then blocks waiting for a new p.
    startlockedm(gp)
    goto top
  }
  // 执行G上的任务
  execute(gp, inheritTime)
}
```

#### findrunnable

函数的实现非常复杂，这个 300 多行的函数通过以下的过程。获取可运行的 Goroutine，获取不到就阻塞住

- 从本地运行队列、全局运行队列查找
- 从网络轮询器中查找是否有等待的groutine
- 通过 runtime.runqsteal 函数尝试从其他随机的处理器中窃取待运行的 Goroutine，在该过程中还可能窃取处理器中的计时器；

### execute

- execute中调用gogo函数将groutine调度到当前线程上

```Go
func execute(gp *g, inheritTime bool) {
  _g_ := getg()

  // Assign gp.m before entering _Grunning so running Gs have an
  // M.
  _g_.m.curg = gp
  gp.m = _g_.m
  casgstatus(gp, _Grunnable, _Grunning)
  gp.waitsince = 0
  gp.preempt = false
  gp.stackguard0 = gp.stack.lo + _StackGuard
  if !inheritTime {
    _g_.m.p.ptr().schedtick++
  }

  // Check whether the profiler needs to be turned on or off.
  hz := sched.profilehz
  if _g_.m.profilehz != hz {
    setThreadCPUProfiler(hz)
  }

  if trace.enabled {
    // GoSysExit has to happen when we have a P, but before GoStart.
    // So we emit it here.
    if gp.syscallsp != 0 && gp.sysblocktraced {
      traceGoSysExit(gp.sysexitticks)
    }
    traceGoStart()
  }

  gogo(&gp.sched)
}
```

### main函数

前面介绍过，main函数在和m绑定的P队列中。因此在调度时，先将main从本地队列取出来，然后传给execute，就可以执行main groutine的任务了。main groutine的任务函数为runtime.main()

主要流程包括：

- 新建线程执行sysmon，用于系统后台监控（定期GC和调度抢占）
- 确保在主线程上执行
- 执行runtime包下所有的init函数
- 启动执行GC的协程
- 执行用户定义的所有的init函数
- 然后在真正执行用户编写的main函数
- 最后exit(0)系统退出
- 如果没有退出，for循环一直访问非法地址，让操作系统杀死进程

源码位置：src/runtime/proc.go

```GO
func main() {
  g := getg()

  // Racectx of m0->g0 is used only as the parent of the main goroutine.
  // It must not be used for anything else.
  g.m.g0.racectx = 0

  // Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
  // Using decimal instead of binary GB and MB because
  // they look nicer in the stack overflow failure message.
  if sys.PtrSize == 8 {
    maxstacksize = 1000000000
  } else {
    maxstacksize = 250000000
  }

  // Allow newproc to start new Ms.
  mainStarted = true

  if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
    // 系统栈上分配一个新的m，运行sysmon(系统后台监控，定期垃圾回收和系统抢占）
    // p里的G是按照顺序执行的，放在某个G执行时间过长，阻塞其他的G
    systemstack(func() {
      newm(sysmon, nil)
    })
  }

  // Lock the main goroutine onto this, the main OS thread,
  // during initialization. Most programs won't care, but a few
  // do require certain calls to be made by the main thread.
  // Those can arrange for main.main to run in the main thread
  // by calling runtime.LockOSThread during initialization
  // to preserve the lock.
  lockOSThread()

  // 确保在主线程上执行
  if g.m != &m0 {
    throw("runtime.main not on m0")
  }

  // 执行runtime包下所有的init
  doInit(&runtime_inittask) // must be before defer
  if nanotime() == 0 {
    throw("nanotime returning zero")
  }

  // Defer unlock so that runtime.Goexit during init does the unlock too.
  needUnlock := true
  defer func() {
    if needUnlock {
      unlockOSThread()
    }
  }()

  // Record when the world started.
  runtimeInitTime = nanotime()

  // 启动一个groutine进行GC
  gcenable()

  main_init_done = make(chan bool)
  if iscgo {
    if _cgo_thread_start == nil {
      throw("_cgo_thread_start missing")
    }
    if GOOS != "windows" {
      if _cgo_setenv == nil {
        throw("_cgo_setenv missing")
      }
      if _cgo_unsetenv == nil {
        throw("_cgo_unsetenv missing")
      }
    }
    if _cgo_notify_runtime_init_done == nil {
      throw("_cgo_notify_runtime_init_done missing")
    }
    // Start the template thread in case we enter Go from
    // a C-created thread and need to create a new thread.
    startTemplateThread()
    cgocall(_cgo_notify_runtime_init_done, nil)
  }

  // 执行用户定义的所有init函数
  doInit(&main_inittask)

  close(main_init_done)

  needUnlock = false
  unlockOSThread()

  if isarchive || islibrary {
    // A program compiled with -buildmode=c-archive or c-shared
    // has a main, but it is not executed.
    return
  }

  // 真正执行main包下的main函数
  fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
  fn()
  if raceenabled {
    racefini()
  }

  // Make racy client program work: if panicking on
  // another goroutine at the same time as main returns,
  // let the other goroutine finish printing the panic trace.
  // Once it does, it will exit. See issues 3934 and 20018.
  if atomic.Load(&runningPanicDefers) != 0 {
    // Running deferred functions should not take long.
    for c := 0; c < 1000; c++ {
      if atomic.Load(&runningPanicDefers) == 0 {
        break
      }
      Gosched()
    }
  }
  if atomic.Load(&panicking) != 0 {
    gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
  }

  // 退出程序
  exit(0)

  // 确保程序崩溃，程序就一定会退出
  // or循环一直访问非法地址，让操作系统杀死进程
  for {
    var x *int32
    *x = 0
  }
}
```

## 参考

- https://zboya.github.io/post/go_scheduler/