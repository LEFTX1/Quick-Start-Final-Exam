# Go语言Goroutine状态字段详解

本文档详细介绍Go语言中goroutine的状态字段及相关源码实现。

## 1. Goroutine结构体中的状态字段

```go
// src/runtime/runtime2.go

type g struct {
    // 省略其他字段...
    
    // 原子状态字段，存储当前goroutine的状态
    // 这是一个原子操作的32位无符号整数，用于表示goroutine的当前状态
    // 状态包括基本状态和GC扫描状态的组合
    atomicstatus atomic.Uint32
    
    // 栈锁，用于sigprof/scang锁定；TODO: 合并到atomicstatus中
    stackLock    uint32
    
    // goroutine的唯一ID
    goid         uint64
    
    // 调度链接，用于在各种队列中链接goroutine
    schedlink    guintptr
    
    // 大约的阻塞时间，当goroutine变为阻塞状态时的时间戳
    waitsince    int64
    
    // 如果status==Gwaiting，表示等待的原因
    waitreason   waitReason
    
    // 省略其他字段...
}
```

## 2. Goroutine状态常量定义

```go
// src/runtime/runtime2.go

const (
    // G状态
    // _Gidle 表示此goroutine刚被分配，尚未初始化
    _Gidle = iota // 0
    // _Grunnable 表示此goroutine在运行队列中。
    // 它当前没有执行用户代码。栈不被拥有。
    _Grunnable // 1
    // _Grunning 表示此goroutine可能执行用户代码。
    // 栈被此goroutine拥有。它不在运行队列中。
    // 它被分配了一个M和一个P（g.m和g.m.p是有效的）。
    _Grunning // 2
    // _Gsyscall 表示此goroutine正在执行系统调用。
    // 它没有执行用户代码。栈被此goroutine拥有。
    // 它不在运行队列中。它被分配了一个M。
    _Gsyscall // 3
    // _Gwaiting 表示此goroutine在运行时中被阻塞。
    // 它没有执行用户代码。它不在运行队列中，
    // 但应该记录在某处（例如，channel等待队列），
    _Gwaiting // 4
    // _Gmoribund_unused 当前未使用，但在gdb脚本中硬编码
    _Gmoribund_unused // 5
    // _Gdead 表示此goroutine当前未使用。
    // 它可能刚刚退出，在空闲列表中，或刚被初始化。
    // 它没有执行用户代码。它可能有也可能没有分配栈。
    // G和其栈（如果有）被正在退出G的M或从空闲列表
    // 获得G的M拥有。
    _Gdead // 6
    // _Genqueue_unused 当前未使用
    _Genqueue_unused // 7
    // _Gcopystack 表示此goroutine的栈正在被移动。
    // 它没有执行用户代码且不在运行队列中。
    // 栈被将其置于_Gcopystack状态的goroutine拥有。
    _Gcopystack // 8 
    // _Gpreempted 表示此goroutine因suspendG抢占而自己停止。
    // 它像_Gwaiting，但还没有任何东西负责ready()它。
    // 一些suspendG必须CAS状态为_Gwaiting来承担
    // ready()此G的责任。
    _Gpreempted // 9
    // _Gscan 与上述状态之一（除了_Grunning）结合使用，
    // 表示GC正在扫描栈。goroutine没有执行用户代码，
    // 栈被设置_Gscan位的goroutine拥有。
    //
    // _Gscanrunning 不同：它用于在GC信号G扫描自己的栈时
    // 简短地阻塞状态转换。这在其他方面像_Grunning。
    //
    // atomicstatus&~Gscan 给出扫描完成时goroutine将
    // 返回的状态。
    _Gscan          = 0x1000
    _Gscanrunnable  = _Gscan + _Grunnable  // 0x1001
    _Gscanrunning   = _Gscan + _Grunning   // 0x1002
    _Gscansyscall   = _Gscan + _Gsyscall   // 0x1003
    _Gscanwaiting   = _Gscan + _Gwaiting   // 0x1004
    _Gscanpreempted = _Gscan + _Gpreempted // 0x1009
)
```

## 3. 状态字符串映射

```go
// src/runtime/traceback.go

// goroutine状态到字符串的映射，用于调试和打印
var gStatusStrings = [...]string{
    _Gidle:      "idle",        // 空闲状态
    _Grunnable:  "runnable",    // 可运行状态
    _Grunning:   "running",     // 运行状态
    _Gsyscall:   "syscall",     // 系统调用状态
    _Gwaiting:   "waiting",     // 等待状态
    _Gdead:      "dead",        // 死亡状态
    _Gcopystack: "copystack",   // 栈复制状态
    _Gpreempted: "preempted",   // 被抢占状态
}
```

## 4. 状态读取函数

```go
// src/runtime/proc.go

// readgstatus 读取g的状态
// 所有对g状态的读取都通过此函数进行
//
//go:nosplit
func readgstatus(gp *g) uint32 {
    // 原子加载goroutine的状态
    return gp.atomicstatus.Load()
}
```

## 5. 状态转换函数

### 5.1 基本状态转换函数

```go
// src/runtime/proc.go

// casgstatus 原子地比较并交换goroutine状态
// 如果要求移动到或从Gscanstatus，此函数会抛出异常。
// 应该使用castogscanstatus和casfrom_Gscanstatus代替。
// casgstatus会循环，如果g->atomicstatus处于Gscan状态，
// 直到将其置于Gscan状态的例程完成。
//
//go:nosplit
func casgstatus(gp *g, oldval, newval uint32) {
    // 检查参数有效性：不能包含_Gscan位，且新旧值不能相同
    if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
        systemstack(func() {
            // 在systemstack上调用以防止print和throw
            // 计入nosplit栈预留
            print("runtime: casgstatus: oldval=", hex(oldval), " newval=", hex(newval), "\n")
            throw("casgstatus: bad incoming values")
        })
    }

    lockWithRankMayAcquire(nil, lockRankGscan)

    // 参见 https://golang.org/cl/21503 对yield延迟的解释
    const yieldDelay = 5 * 1000
    var nextYield int64

    // 如果gp->atomicstatus处于扫描状态，则循环，
    // 给GC时间完成并将状态更改为oldval
    for i := 0; !gp.atomicstatus.CompareAndSwap(oldval, newval); i++ {
        if oldval == _Gwaiting && gp.atomicstatus.Load() == _Grunnable {
            systemstack(func() {
                // 在systemstack上调用以防止throw计入
                // nosplit栈预留
                throw("casgstatus: waiting for Gwaiting but is Grunnable")
            })
        }
        if i == 0 {
            nextYield = nanotime() + yieldDelay
        }
        if nanotime() < nextYield {
            for x := 0; x < 10 && gp.atomicstatus.Load() != oldval; x++ {
                procyield(1)
            }
        } else {
            osyield()
            nextYield = nanotime() + yieldDelay/2
        }
    }

    // 如果goroutine有bubble（用于synctest），通知状态变化
    if gp.bubble != nil {
        systemstack(func() {
            gp.bubble.changegstatus(gp, oldval, newval)
        })
    }

    // 如果从_Grunning状态退出，进行性能跟踪
    if oldval == _Grunning {
        // 每gTrackingPeriod次跟踪一次goroutine从运行状态的转换
        if casgstatusAlwaysTrack || gp.trackingSeq%gTrackingPeriod == 0 {
            gp.tracking = true
        }
        gp.trackingSeq++
    }
    if !gp.tracking {
        return
    }

    // 处理各种类型的跟踪
    // 当前包括：
    // - 在可运行状态花费的时间
    // - 在sync.Mutex或sync.RWMutex上阻塞的时间
    // ... 省略跟踪代码 ...
}
```

### 5.2 扫描状态转换函数

```go
// src/runtime/proc.go

// castogscanstatus 将g从oldval状态转换为newval|_Gscan状态
// 这充当锁获取，而casfromgstatus充当锁释放
func castogscanstatus(gp *g, oldval, newval uint32) bool {
    switch oldval {
    case _Grunnable,    // 可运行状态
         _Grunning,     // 运行状态
         _Gwaiting,     // 等待状态
         _Gsyscall:     // 系统调用状态
        if newval == oldval|_Gscan {
            // 尝试原子设置扫描位
            r := gp.atomicstatus.CompareAndSwap(oldval, newval)
            if r {
                acquireLockRankAndM(lockRankGscan)
            }
            return r
        }
    }
    print("runtime: castogscanstatus oldval=", hex(oldval), " newval=", hex(newval), "\n")
    throw("castogscanstatus")
    panic("not reached")
}

// casfrom_Gscanstatus 从扫描状态释放g
// Gscanstatuses充当锁，此函数释放它们
func casfrom_Gscanstatus(gp *g, oldval, newval uint32) {
    success := false

    // 检查转换是否有效
    switch oldval {
    default:
        print("runtime: casfrom_Gscanstatus bad oldval gp=", gp, ", oldval=", hex(oldval), ", newval=", hex(newval), "\n")
        dumpgstatus(gp)
        throw("casfrom_Gscanstatus:top gp->status is not in scan state")
    case _Gscanrunnable,   // 扫描+可运行
         _Gscanwaiting,    // 扫描+等待
         _Gscanrunning,    // 扫描+运行
         _Gscansyscall,    // 扫描+系统调用
         _Gscanpreempted:  // 扫描+抢占
        if newval == oldval&^_Gscan {
            // 清除扫描位
            success = gp.atomicstatus.CompareAndSwap(oldval, newval)
        }
    }
    if !success {
        print("runtime: casfrom_Gscanstatus failed gp=", gp, ", oldval=", hex(oldval), ", newval=", hex(newval), "\n")
        dumpgstatus(gp)
        throw("casfrom_Gscanstatus: gp->status is not in scan state")
    }
    releaseLockRankAndM(lockRankGscan)
}
```

## 6. 关键状态转换场景

### 6.1 Goroutine执行

```go
// src/runtime/proc.go

// execute 开始执行goroutine
func execute(gp *g, inheritTime bool) {
    mp := getg().m

    // 省略其他代码...

    // 在进入_Grunning之前分配gp.m，这样运行的Gs有一个M
    mp.curg = gp
    gp.m = mp
    gp.syncSafePoint = false // 清除标志，可能由morestack设置
    
    // 将goroutine状态从_Grunnable转换为_Grunning
    casgstatus(gp, _Grunnable, _Grunning)
    
    gp.waitsince = 0      // 清除等待时间
    gp.preempt = false    // 清除抢占标志
    gp.stackguard0 = gp.stack.lo + stackGuard
    if !inheritTime {
        mp.p.ptr().schedtick++
    }

    // 省略其他代码...

    // 跳转到goroutine执行
    gogo(&gp.sched)
}
```

### 6.2 Goroutine就绪

```go
// src/runtime/proc.go

// ready 标记gp准备运行
func ready(gp *g, traceskip int, next bool) {
    status := readgstatus(gp)

    // 标记为可运行
    mp := acquirem() // 禁用抢占，因为它可能在局部变量中持有p
    if status&^_Gscan != _Gwaiting {
        dumpgstatus(gp)
        throw("bad g->status in ready")
    }

    // 状态是Gwaiting或Gscanwaiting，使其变为Grunnable并放入运行队列
    trace := traceAcquire()
    casgstatus(gp, _Gwaiting, _Grunnable)
    if trace.ok() {
        trace.GoUnpark(gp, traceskip)
        traceRelease(trace)
    }
    runqput(mp.p.ptr(), gp, next)  // 放入运行队列
    wakep()                        // 唤醒P
    releasem(mp)
}
```

### 6.3 Goroutine休眠

```go
// src/runtime/proc.go

// park_m 将当前goroutine置于waiting状态
func park_m(gp *g) {
    mp := getg().m

    trace := traceAcquire()

    // 省略synctest相关代码...

    if trace.ok() {
        // 在转换前跟踪事件。可能会获取栈跟踪，
        // 但转换后我们将不再拥有栈
        trace.GoPark(mp.waitTraceBlockReason, mp.waitTraceSkip)
    }
    
    // 注意：这里不使用casGToWaiting，因为waitreason
    // 由park_m的调用者设置
    casgstatus(gp, _Grunning, _Gwaiting)
    
    if trace.ok() {
        traceRelease(trace)
    }

    dropg()  // 解除M和G的绑定

    // 省略unlock函数处理...

    schedule()  // 重新调度
}
```

## 7. 状态转换图

```
          newproc
             ↓
          _Gdead ←─────────────────┐
             ↓                    │
         _Grunnable ←──────────┐   │ goexit
             ↓                │   │
          _Grunning           │   │
        ↙    ↓    ↘          │   │
_Gsyscall  park  preempt     │   │
    ↓        ↓      ↓        │   │
    └→ _Gwaiting ←──┘        │   │
           ↓                 │   │
       ready() ──────────────┘   │
                                 │
          任何状态 ───────────────────┘
```

## 8. 状态详细说明

| 状态 | 值 | 描述 | 栈所有权 | 队列位置 |
|------|----|----|---------|---------|
| `_Gidle` | 0 | 刚分配，未初始化 | 无 | 无 |
| `_Grunnable` | 1 | 在运行队列中，等待执行 | 无 | 运行队列 |
| `_Grunning` | 2 | 正在执行用户代码 | goroutine | 无 |
| `_Gsyscall` | 3 | 执行系统调用 | goroutine | 无 |
| `_Gwaiting` | 4 | 被阻塞等待 | 特殊 | 等待队列 |
| `_Gdead` | 6 | 已结束或在空闲列表 | M | 空闲列表 |
| `_Gcopystack` | 8 | 栈正在复制 | 复制者 | 无 |
| `_Gpreempted` | 9 | 被抢占停止 | 特殊 | 无 |

## 9. 总结

Goroutine的状态管理是Go运行时调度器的核心组成部分。通过`atomicstatus`字段和相关的原子操作函数，Go运行时能够：

1. **安全地管理并发状态转换**：使用原子操作确保状态变更的线程安全性
2. **支持垃圾收集器扫描**：通过`_Gscan`位协调GC扫描和goroutine执行
3. **实现精确的调度控制**：不同状态决定了goroutine的调度行为
4. **提供调试信息**：状态信息用于运行时诊断和调试

理解这些状态及其转换对于深入理解Go语言的并发模型和调度器实现至关重要。 