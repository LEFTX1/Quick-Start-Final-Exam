``` Go

// src/runtime/proc.go ready函数
func ready(gp *g, traceskip int, next bool) {
    // ... 状态处理 ...
    
    // 步骤1：标记goroutine为runnable
    casgstatus(gp, _Gwaiting, _Grunnable)
    
    // 步骤2：放入runnext字段
    runqput(mp.p.ptr(), gp, next)
    
    // 步骤3：立即检查是否需要唤醒空闲P
    wakep()
    
    releasem(mp)
}




// src/runtime/proc.go runqget函数
func runqget(pp *p) (gp *g, inheritTime bool) {
    // 1. 首先检查runnext（最高优先级）
    next := pp.runnext
    if next != 0 && pp.runnext.cas(next, 0) {
        return next.ptr(), true  // 继承时间片
    }

    // 2. 然后从runq队列头部取
    for {
        h := atomic.LoadAcq(&pp.runqhead)
        t := pp.runqtail
        if t == h {
            return nil, false  // 队列为空
        }
        gp := pp.runq[h%uint32(len(pp.runq))].ptr()
        if atomic.CasRel(&pp.runqhead, h, h+1) {
            return gp, false  // 新的时间片
        }
    }
}










src/runtime/runtime2.go

type p struct {
	m           muintptr   // 反向链接到关联的 m (如果空闲则为 nil)
	//...//
}

type m struct {
	g0      *g // 拥有调度栈的 goroutine (g0)
	//...//
	p  puintptr  // 关联的 p，用于执行 go 代码 
	//...//
}


src/runtime/proc.go

//优先根据GOMAXPROCS的配置决定p的个数
if n, ok := strconv.Atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
    procs = n
    sched.customGOMAXPROCS = true
} else {
    //默认使用cpu核心数
    procs = defaultGOMAXPROCS(numCPUStartup)
}


src/runtime/runtime2.go

type schedt struct {
    // …… //
    pidle      puintptr       // 空闲 P 的链表头，保存可分配的工作上下文
    npidle     atomic.Int32   // 空闲 P 的数量，用于快速判断是否有可用 P
    // …… //
}



src/runtime/proc.go

// 从当前的 g 的栈，切换到当前 M 的 g0 栈，并执行 fn(g)。
func mcall(fn func(*g))



// Go 调度器本地队列（runq）与 runnext 数据结构实现及设计理念详解  
// 基于 Go 源码 src/runtime/runtime2.go 和 src/runtime/proc.go//  
// 本文件展示了 Go 语言调度器中 P（Processor）的本地运行队列设计  
// 这是 Go 高效并发调度的核心组件之一  
  
package runtime  
  
import (  
    "runtime/internal/atomic"  
    "unsafe")  
  
// ============================================================================  
// 核心数据结构定义  
// ============================================================================  
  
// guintptr 是一个原子指针类型，用于安全地存储和操作 goroutine 指针  
// 在多线程环境下保证指针操作的原子性  
type guintptr uintptr  
  
// cas 原子比较并交换操作  
// 这是实现无锁数据结构的关键原语  
func (gp *guintptr) cas(old, new guintptr) bool {  
    return atomic.Casuintptr((*uintptr)(unsafe.Pointer(gp)), uintptr(old), uintptr(new))  
}  
  
// ptr 安全地将 guintptr 转换为 *g 指针  
func (gp guintptr) ptr() *g {  
    return (*g)(unsafe.Pointer(gp))  
}  
  
// set 原子地设置指针值  
func (gp *guintptr) set(g *g) {  
    atomic.Storeuintptr((*uintptr)(unsafe.Pointer(gp)), uintptr(unsafe.Pointer(g)))  
}  
  
// g 代表一个 goroutine 的结构体（简化版本）  
type g struct {  
    // ... 其他字段  
    schedlink guintptr // 用于链接到调度队列中的下一个 goroutine    // ... 其他字段  
}  
  
// ============================================================================  
// P（Processor）结构体中的队列相关字段  
// ============================================================================  
  
type p struct {  
    id     int32  // P 的唯一标识符  
    status uint32 // P 的状态（空闲、运行中等）  
    //...//
    runqhead uint32        // 队列头部索引（消费者读取位置）  
    runqtail uint32        // 队列尾部索引（生产者写入位置）  
    runq     [256]guintptr // 环形缓冲区，存储等待运行的 goroutine 指针  
    //...//
    runnext guintptr  
    // ... 其他字段  
}  
  
// ============================================================================  
// 队列操作函数实现  
// ============================================================================  
  
// runqput 将 goroutine 添加到本地运行队列  
//  
// 参数：  
//  
//  pp: 目标 P//  gp: 要添加的 goroutine//  next: 是否放入 runnext 优先队列  
//  
// 设计理念：  
// 1. 优先使用 runnext：提供低延迟调度  
// 2. 环形缓冲区：高效的队列实现  
// 3. 自动降级：本地队列满时转移到全局队列  
func runqput(pp *p, gp *g, next bool) {  
    // 随机化调度顺序，用于竞态检测时发现调度假设错误  
    if randomizeScheduler && next && randn(2) == 0 {  
       next = false  
    }  
  
    // ========================================================================  
    // runnext 优先队列处理  
    // ========================================================================  
    if next {  
    retryNext:  
       // 原子地获取当前 runnext 的值  
       oldnext := pp.runnext  
  
       // 尝试原子地将新 goroutine 设置为 runnext       // 这里使用 CAS 操作保证线程安全  
       if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {  
          // CAS 失败，可能是其他线程修改了 runnext，重试  
          goto retryNext  
       }  
  
       if oldnext == 0 {  
          // 之前 runnext 为空，直接设置成功  
          return  
       }  
  
       // runnext 之前有值，将旧值踢到常规队列中  
       // 这确保了 runnext 只有一个槽位，新的优先级更高  
       gp = oldnext.ptr()  
    }  
  
    // ========================================================================  
    // 常规环形队列处理  
    // ========================================================================  
retry:  
    // 使用 load-acquire 语义读取队列头部，与消费者同步  
    h := atomic.LoadAcq(&pp.runqhead)  
    t := pp.runqtail  
  
    // 检查队列是否还有空间  
    if t-h < uint32(len(pp.runq)) {  
       // 队列未满，将 goroutine 添加到尾部  
       // 使用模运算实现环形缓冲区  
       pp.runq[t%uint32(len(pp.runq))].set(gp)  
  
       // 使用 store-release 语义更新尾部索引，使元素对消费者可见  
       atomic.StoreRel(&pp.runqtail, t+1)  
       return  
    }  
  
    // 队列已满，需要将一部分工作转移到全局队列  
    if runqputslow(pp, gp, h, t) {  
       return  
    }  
  
    // 慢路径处理完成，队列现在应该有空间了，重试  
    goto retry  
}  
  
// runqputslow 处理本地队列满时的情况  
// 将本地队列的一半 goroutine 转移到全局队列  
//  
// 设计理念：  
// 1. 负载均衡：防止某个 P 积累过多工作  
// 2. 工作窃取：其他空闲的 P 可以从全局队列获取工作  
// 3. 批量操作：减少全局队列的锁竞争  
func runqputslow(pp *p, gp *g, h, t uint32) bool {  
    // 创建批处理数组，大小为队列一半加上当前要添加的 goroutine    var batch [len(pp.runq)/2 + 1]*g  
  
    // 计算要转移的 goroutine 数量（队列的一半）  
    n := t - h  
    n = n / 2  
  
    if n != uint32(len(pp.runq)/2) {  
       throw("runqputslow: queue is not full")  
    }  
  
    // 从本地队列头部取出一半的 goroutine    for i := uint32(0); i < n; i++ {  
       batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()  
    }  
  
    // 原子地更新队列头部，提交消费操作  
    if !atomic.CasRel(&pp.runqhead, h, h+n) {  
       // CAS 失败，可能是有其他线程在窃取工作，返回 false 重试  
       return false  
    }  
  
    // 将当前要添加的 goroutine 也加入批次  
    batch[n] = gp  
  
    // 可选的随机化，用于测试  
    if randomizeScheduler {  
       for i := uint32(1); i <= n; i++ {  
          j := cheaprandn(i + 1)  
          batch[i], batch[j] = batch[j], batch[i]  
       }  
    }  
  
    // 将 goroutine 链接成链表，准备添加到全局队列  
    for i := uint32(0); i < n; i++ {  
       batch[i].schedlink.set(batch[i+1])  
    }  
  
    // 创建队列结构，包含头、尾和大小信息  
    q := gQueue{  
       head: batch[0].guintptr(),  
       tail: batch[n].guintptr(),  
       size: int32(n + 1),  
    }  
  
    // 将批次添加到全局队列（需要获取全局锁）  
    lock(&sched.lock)  
    globrunqputbatch(&q)  
    unlock(&sched.lock)  
  
    return true  
}  
  
// runqget 从本地运行队列获取可执行的 goroutine//  
// 返回值：  
//  
//  gp: 获取到的 goroutine//  inheritTime: 是否继承当前时间片  
//  
// 调度优先级：  
// 1. 首先检查 runnext 优先队列  
// 2. 然后从常规环形队列头部取出  
func runqget(pp *p) (gp *g, inheritTime bool) {  
    // ========================================================================  
    // 优先检查 runnext 队列  
    // ========================================================================  
  
    // 获取 runnext 中的 goroutine    next := pp.runnext  
  
    // 如果 runnext 非空，尝试原子地将其取出  
    // 注意：如果 CAS 失败，说明被其他 P 窃取了，不需要重试  
    // 因为其他 P 只能将 runnext 设为 0，只有当前 P 能设为非零值  
    if next != 0 && pp.runnext.cas(next, 0) {  
       // 成功取出 runnext，返回 true 表示继承时间片  
       // 这是 runnext 优化的关键：减少上下文切换开销  
       return next.ptr(), true  
    }  
  
    // ========================================================================  
    // 从常规环形队列获取  
    // ========================================================================  
  
    for {  
       // 使用 load-acquire 语义读取队列头部，与其他消费者同步  
       h := atomic.LoadAcq(&pp.runqhead)  
       t := pp.runqtail  
  
       // 检查队列是否为空  
       if t == h {  
          return nil, false  
       }  
  
       // 从队列头部取出 goroutine       gp := pp.runq[h%uint32(len(pp.runq))].ptr()  
  
       // 原子地更新队列头部索引，提交消费操作  
       if atomic.CasRel(&pp.runqhead, h, h+1) {  
          // 成功获取，返回 false 表示开始新的时间片  
          return gp, false  
       }  
       // CAS 失败，可能是其他线程也在消费，重试  
    }  
}  
  
// runqempty 检查本地队列是否为空  
// 需要同时检查 runnext 和环形队列，并处理可能的竞态条件  
func runqempty(pp *p) bool {  
    // 需要防止竞态条件：  
    // 1. P 在 runnext 有 G1，但 runqhead == runqtail    // 2. runqput 将 G1 踢到 runq 中  
    // 3. runqget 清空了 runnext    // 简单地观察 runqhead == runqtail 和 runnext == nil 并不能保证队列真的为空  
  
    for {  
       head := atomic.Load(&pp.runqhead)  
       tail := atomic.Load(&pp.runqtail)  
       runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&pp.runnext)))  
  
       // 重新检查 runqtail，确保读取的一致性  
       if tail == atomic.Load(&pp.runqtail) {  
          return head == tail && runnext == 0  
       }  
       // 读取不一致，重试  
    }  
}  
  
// ============================================================================  
// 工作窃取相关函数  
// ============================================================================  
  
// runqsteal 从其他 P 的本地队列窃取工作  
// 这是 Go 调度器实现负载均衡的重要机制  
//  
// 参数：  
//  
//  pp: 目标 P（窃取者）  
//  p2: 源 P（被窃取者）  
//  stealRunNextG: 是否可以窃取 runnext//  
// 设计理念：  
// 1. 负载均衡：忙碌的 P 分担工作给空闲的 P// 2. 局部性保持：只窃取一半，保持原 P 的缓存局部性  
// 3. 最小化竞争：通过 CAS 操作实现无锁窃取  
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {  
    t := pp.runqtail  
  
    // 尝试从 p2 窃取 goroutine 到 pp 的队列尾部  
    n := runqgrab(p2, &pp.runq, t, stealRunNextG)  
    if n == 0 {  
       return nil // 没有窃取到任何工作  
    }  
  
    // 窃取成功，返回其中一个 goroutine 立即执行  
    n--  
    gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()  
  
    if n == 0 {  
       // 只窃取到一个，直接返回  
       return gp  
    }  
  
    // 窃取到多个，更新队列尾部索引，使其他的可用于后续调度  
    h := atomic.LoadAcq(&pp.runqhead)  
    if t-h+n >= uint32(len(pp.runq)) {  
       throw("runqsteal: runq overflow")  
    }  
    atomic.StoreRel(&pp.runqtail, t+n)  
  
    return gp  
}  
  
// runqgrab 执行实际的窃取操作  
// 从源队列抓取约一半的 goroutine 到目标批次中  
func runqgrab(pp *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {  
    for {  
       // 原子地读取源队列的头尾索引  
       h := atomic.LoadAcq(&pp.runqhead)  
       t := atomic.LoadAcq(&pp.runqtail)  
  
       // 计算要窃取的数量（约一半）  
       n := t - h  
       n = n - n/2  
  
       if n == 0 {  
          // 常规队列为空，尝试窃取 runnext          if stealRunNextG {  
             if next := pp.runnext; next != 0 {  
                // 如果源 P 正在运行，稍等一下避免干扰其调度 runnext                if pp.status == _Prunning {  
                   if !osHasLowResTimer {  
                      usleep(3) // 等待约 3 微秒  
                   } else {  
                      osyield() // 让出 CPU                   }  
                }  
  
                // 尝试原子地窃取 runnext                if !pp.runnext.cas(next, 0) {  
                   continue // CAS 失败，重试  
                }  
  
                batch[batchHead%uint32(len(batch))] = next  
                return 1  
             }  
          }  
          return 0 // 没有可窃取的工作  
       }  
  
       // 检查读取的一致性  
       if n > uint32(len(pp.runq)/2) {  
          continue // 读取不一致，重试  
       }  
  
       // 复制 goroutine 到批次数组  
       for i := uint32(0); i < n; i++ {  
          g := pp.runq[(h+i)%uint32(len(pp.runq))]  
          batch[(batchHead+i)%uint32(len(batch))] = g  
       }  
  
       // 原子地更新源队列头部，提交窃取操作  
       if atomic.CasRel(&pp.runqhead, h, h+n) {  
          return n // 窃取成功  
       }  
       // CAS 失败，重试  
    }  
}  
  
// ============================================================================  
// 设计理念总结  
// ============================================================================  
  
/*  
Go 调度器本地队列设计的核心理念：  
  
1. 【性能优化】  
   - 无锁设计：使用原子操作和 CAS 避免锁竞争  
   - 缓存友好：连续内存布局和局部性优化  
   - 批量操作：减少全局队列访问频率  
  
2. 【并发安全】  
   - 原子操作：保证多线程环境下的数据一致性  
   - 内存屏障：使用 acquire/release 语义确保可见性  
   - ABA 问题处理：通过精心设计的 CAS 操作避免  
  
3. 【负载均衡】  
   - 工作窃取：空闲 P 可以从忙碌 P 窃取工作  
   - 全局降级：本地队列满时转移到全局队列  
   - 公平调度：定期检查全局队列确保公平性  
  
4. 【延迟优化】  
   - runnext 优先队列：为相关 goroutine 提供低延迟调度  
   - 时间片继承：减少上下文切换开销  
   - 局部性利用：提高缓存命中率  
  
5. 【可扩展性】  
   - 分布式设计：每个 P 独立的本地队列  
   - 最小化竞争：大部分操作在本地完成  
   - 动态适应：根据负载情况自动调整  
  
这种设计使得 Go 能够高效地调度大量的 goroutine，  
实现了高并发、低延迟和良好可扩展性的完美平衡。  
*/


type p struct {
    id     int32  // P 的唯一标识符
    status uint32 // P 的状态（空闲、运行中等）
    runqhead uint32        // 队列头部索引（消费者读取位置）
    runqtail uint32        // 队列尾部索引（生产者写入位置）
    runq     [256]guintptr // 环形缓冲区，存储等待运行的 goroutine 指针
    runnext guintptr
    // ... 其他字段
}


func currGoroutineDo (){
    go func() { // <-- 在这行代码的背后，一个_Gidle的G被初始化并变为_Grunnable
    fmt.Println("Hello from a new goroutine!")
    }()
}




// src/runtime/proc.go findRunnable函数片段
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
//...//
if pp.schedtick%61 == 0 && !sched.runq.empty() {
    // %61 == 0: 每61次调度检查一次，这是一个经验值，用于平衡性能和公平性
    lock(&sched.lock)           // 获取调度器全局锁，保护全局队列操作
    gp := globrunqget()         // 从全局队列获取一个goroutine
    unlock(&sched.lock)         // 释放调度器全局锁
    if gp != nil {
        // 成功获取到goroutine，立即返回执行
        // 第二个参数false: 不继承时间片，使用完整的时间片
        // 第三个参数false: 不需要唤醒额外的P
        return gp, false, false
    }
    }
    //...//
    }
```
