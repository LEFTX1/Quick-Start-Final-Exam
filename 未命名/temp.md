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
```
