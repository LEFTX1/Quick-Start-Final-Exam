``` Go
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
```
