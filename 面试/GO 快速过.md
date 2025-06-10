### Context / Cancel / CSP 要点

1. CSP 模型：Goroutine 通过 Channel 通信来共享数据，而不是直接共享彼此内存。
    
2. 取消信号传递：父 Context 关闭 Done() channel 后会级联关闭所有子 Context 的 Done()，select 即可立刻响应退出。
    
3. context 包作用：统一在跨 goroutine 间传递取消/超时控制与请求级别的少量数据。
    
4. 取消与超时机制：触发 cancel/timeout 时关闭 Done() channel，监听它的 select 分支瞬间执行。
    
5. 常用创建方式：WithCancel 手动取消；WithTimeout/WithDeadline 按时限取消；WithValue 仅附带键值，全都生成子 Context。
    
6. WithValue 使用要点：只传请求范围的小数据，用私有类型作 key，切勿当一般参数或隐式全局用。
    
7. vs TODO：Background() 是正式根 Context，TODO() 只是占位提示“待补合适 Context”。
    
8. 最佳实践与坑：ctx 置首参显式传、goroutine 里选 ctx.Done、defer cancel()、慎用 WithValue、别把 ctx 存进 struct。
    

---

### GMP 模型

- **Q1: GMP 模型是什么？**  
    GMP是Go的调度模型，P(处理器)负责将海量的G(协程)高效地调度到少量的M(系统线程)上执行。
    
- **Q2: G, M, P 如何协作？**  
    M绑定P后，从P的本地队列取G执行，队列空则从全局队列或其它P窃取。
    
- **Q3: 调度器与工作窃取？**  
    工作窃取指一个P的本地队列为空时，其M会从其它P的队列末尾偷走一半G来执行，以实现负载均衡。
    
- **Q4: Goroutine 切换为何快？**  
    Goroutine切换快因其在用户态完成，不陷内核且仅保存少量寄存器，成本远低于内核调度的线程。
    
- **Q5: Go 如何处理阻塞的系统调用？**  
    Sysmon监控到M因系统调用阻塞过久时，会抢走其P给其他M用，防止P被闲置。
    
- **Q6: g0 是什么？**  
    g0是每个M上代表调度器本身的特殊协程，运行在M的系统栈上，负责执行调度、GC等runtime任务。
    
- **Q7: 需要手动实现协程池吗？**  
    Go的runtime会自动复用G对象，因此手动协程池主要目的不是节约开销，而是为了控制并发数量。
    
- **Q8: 什么是线程自旋？**  
    线程自旋是一种忙等待优化，M在短时等待（如等锁或等G）时会空转而非休眠，以避免昂贵的线程切换。
    
- **Q9: GMP 模型带来了哪些优势？**  
    GMP模型通过轻量级协程、高效调度、工作窃取及对阻塞的智能处理，实现了极高并发和对多核的充分利用。
    
- **Q10: Goroutine 的栈如何管理？**  
    Goroutine的栈非固定大小，初始很小，当空间不足时runtime会分配更大连续内存并拷贝旧栈内容，实现动态扩容。
    
- **Q11: 栈扩容的开销与场景？**  
    栈扩容主要开销是拷贝旧栈内存，最常见触发场景是无限/过深的递归调用。
    
- **Q18: 什么是 Cgo？**  
    Cgo是Go与C语言交互的机制，主要用于复用成熟的C/C++库，如数据库驱动、算法库或特定领域SDK。