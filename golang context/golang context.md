# Golang Context 实现原理
- [[#0 前言|0 前言]]
- [[#**`Context` 是协程之间的生命周期联动属性吗？**|**`Context` 是协程之间的生命周期联动属性吗？**]]
- [[#1 核心数据结构|1 核心数据结构]]
	- [[#1 核心数据结构#1.1 `context.Context`|1.1 `context.Context`]]
	- [[#1 核心数据结构#1.2 标准 error|1.2 标准 error]]
- [[#2 `emptyCtx`|2 `emptyCtx`]]
	- [[#2 `emptyCtx`#2.1 类的实现|2.1 类的实现]]
	- [[#2 `emptyCtx`#2.2 `context.Background()` & `context.TODO()`|2.2 `context.Background()` & `context.TODO()`]]
- [[#3 `cancelCtx`|3 `cancelCtx`]]
	- [[#3 `cancelCtx`#Q1:协程创建时，context会形成一颗父子结构的树吗?|Q1:协程创建时，context会形成一颗父子结构的树吗?]]
	- [[#3 `cancelCtx`#Q2:父context终止，会形成从上往下的树的单向传递的终止是吗?|Q2:父context终止，会形成从上往下的树的单向传递的终止是吗?]]
	- [[#3 `cancelCtx`#Q3:也就是说context是协程之间的生命周期联动属性？|Q3:也就是说context是协程之间的生命周期联动属性？]]
		- [[#Q3:也就是说context是协程之间的生命周期联动属性？#q1 我们知道context之间的联动取消，但还不知道是如何通过context取消影响到goroutine的取消的。|q1 我们知道context之间的联动取消，但还不知道是如何通过context取消影响到goroutine的取消的。]]
			- [[#q1 我们知道context之间的联动取消，但还不知道是如何通过context取消影响到goroutine的取消的。#什么是多路复用？|什么是多路复用？]]
	- [[#3 `cancelCtx`#3.1 `cancelCtx` 数据结构|3.1 `cancelCtx` 数据结构]]
	- [[#3 `cancelCtx`#Q4：`cancelCtx`是怎么继承父context的能力的|Q4：`cancelCtx`是怎么继承父context的能力的]]
	- [[#3 `cancelCtx`#Q5: done方法本质上只是用来查看context是否存活是吗?|Q5: done方法本质上只是用来查看context是否存活是吗?]]
	- [[#3 `cancelCtx`#只读管道读取时会阻塞，直到他被关闭？|只读管道读取时会阻塞，直到他被关闭？]]
	- [[#3 `cancelCtx`#这就是为什么只读管道可以作为信号变量的原因？|这就是为什么只读管道可以作为信号变量的原因？]]
	- [[#3 `cancelCtx`#作为信号变量的原因：|作为信号变量的原因：]]
	- [[#3 `cancelCtx`#3.2 Deadline 方法|3.2 Deadline 方法]]
		- [[#3.2 Deadline 方法#Q1:cancelcontext本身不实现Deadline ，默认使用其父context的Deadline 方法，如果没有实现则返回nil？这个说法对吗？|Q1:cancelcontext本身不实现Deadline ，默认使用其父context的Deadline 方法，如果没有实现则返回nil？这个说法对吗？]]
	- [[#3 `cancelCtx`#3.3 Done 方法|3.3 Done 方法]]
- [[#1. 什么是“接收方 Context”？|1. 什么是“接收方 Context”？]]
- [[#2. “当 Context 被取消时，这个通道会被关闭” 是指 cancel 方法吗？|2. “当 Context 被取消时，这个通道会被关闭” 是指 cancel 方法吗？]]
- [[#3. 关于 Done 通道的懒加载和创建|3. 关于 Done 通道的懒加载和创建]]
- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？|Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#1. 调用 cancel 时的行为|1. 调用 cancel 时的行为]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#5. 监听 Done 通道是由编程者自己控制还是 Go 管理？|5. 监听 Done 通道是由编程者自己控制还是 Go 管理？]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#3.4 Err 方法|3.4 Err 方法]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#3.5 Value 方法|3.5 Value 方法]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#3.6 context.WithCancel()|3.6 context.WithCancel()]]
		- [[#3.6 context.WithCancel()#3.6.1 context.WithCancel()|3.6.1 context.WithCancel()]]
		- [[#3.6 context.WithCancel()#3.6.2 newCancelCtx|3.6.2 newCancelCtx]]
		- [[#3.6 context.WithCancel()#3.6.3 propagateCancel|3.6.3 propagateCancel]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#具体说明：|具体说明：]]
	- [[#Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？#3.6.4 cancelCtx.cancel|3.6.4 cancelCtx.cancel]]
- [[#4 `timerCtx`|4 `timerCtx`]]
	- [[#4 `timerCtx`#4.1 类|4.1 类]]
	- [[#4 `timerCtx`#4.2 `timerCtx.Deadline()`|4.2 `timerCtx.Deadline()`]]
	- [[#4 `timerCtx`#4.3 `timerCtx.cancel`|4.3 `timerCtx.cancel`]]
	- [[#4 `timerCtx`#4.4 `context.WithTimeout` & `context.WithDeadline`|4.4 `context.WithTimeout` & `context.WithDeadline`]]
- [[#5 `valueCtx`|5 `valueCtx`]]
	- [[#5 `valueCtx`#5.1 类|5.1 类]]
	- [[#5 `valueCtx`#5.2 `valueCtx.Value()`|5.2 `valueCtx.Value()`]]
	- [[#5 `valueCtx`#5.3 `valueCtx` 用法小结|5.3 `valueCtx` 用法小结]]
	- [[#5 `valueCtx`#5.4 `context.WithValue()`|5.4 `context.WithValue()`]]
- [[#3. **`cancelCtx` 本身不实现 `Deadline`，默认使用其父 `Context` 的 `Deadline` 方法，如果没有实现则返回 `nil`，这个说法对吗？**|3. **`cancelCtx` 本身不实现 `Deadline`，默认使用其父 `Context` 的 `Deadline` 方法，如果没有实现则返回 `nil`，这个说法对吗？**]]
- [[#4. **`cancelCtxKey` 是什么？**|4. **`cancelCtxKey` 是什么？**]]
- [[#1. `Context` 的树设计是否使用了组合模式？|1. `Context` 的树设计是否使用了组合模式？]]
- [[#2. 在 `valueCtx` 结构体中，`Context` 字段是否是原 `Context` 的指针？|2. 在 `valueCtx` 结构体中，`Context` 字段是否是原 `Context` 的指针？]]
- [[#3. 每次派生一个新的 `Context` 时，是否是在形成树的新节点时加上一个被包装好的指针壳？|3. 每次派生一个新的 `Context` 时，是否是在形成树的新节点时加上一个被包装好的指针壳？]]
- [[#4. 对于 `cancel` 方法，它的主要职责是什么？|4. 对于 `cancel` 方法，它的主要职责是什么？]]
- [[#5. 在 `value` 函数中，遇到非 `valueCtx` 类型的上下文时，做了什么？|5. 在 `value` 函数中，遇到非 `valueCtx` 类型的上下文时，做了什么？]]
- [[#6. 如果要找到最近的 `cancelCtx` 祖先，该如何操作？|6. 如果要找到最近的 `cancelCtx` 祖先，该如何操作？]]
- [[#7. `value` 函数的主要任务是什么？|7. `value` 函数的主要任务是什么？]]
- [[#解释一下：func Cause(c Context) error|解释一下：func Cause(c Context) error]]
		- [[#5.4 `context.WithValue()`#1. 父 `Context` 是否具备取消能力|1. 父 `Context` 是否具备取消能力]]
		- [[#5.4 `context.WithValue()`#2. 为什么要检查 `done` 通道是否相同？|2. 为什么要检查 `done` 通道是否相同？]]
		- [[#5.4 `context.WithValue()`#3. 一句话描述三个函数的作用|3. 一句话描述三个函数的作用]]
	- [[#解释一下：func Cause(c Context) error#总结|总结]]



---

## 0 前言

`context` 是 Golang 中的经典工具，主要在异步场景中用于实现并发协调以及对 goroutine 的生命周期控制。除此之外，`context` 还兼有一定的数据存储能力。本着知其然知其所以然的精神，本文和大家一起深入 `context` 源码一探究竟，较为细节地对其实现原理进行梳理。
![[context包概括.svg]]

##  **`Context` 是协程之间的生命周期联动属性吗？**

是的，`Context` 的核心功能之一就是**控制协程之间的生命周期联动**。当父 `Context` 被取消时，所有依赖这个父 `Context` 的子 `Context` 以及使用该 `Context` 的协程都会收到取消信号，从而能够统一处理取消操作。这种设计可以让多个协程在同一时刻被取消，确保它们协调终止，避免资源浪费和并发问题。

通过 `Done()` 通道，所有依赖 `Context` 的协程可以监听 `Context` 的状态，一旦 `Context` 被取消，`Done()` 通道会关闭，通知所有监听者。

## 1 核心数据结构

### 1.1 `context.Context`
![[Pasted image 20241011002200.png|825]]

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 返回 context 的截止时间和是否存在截止时间
    Done() <-chan struct{}                     // 返回一个 channel，当 context 被取消或超时时会关闭此 channel
    Err() error                                // 返回 context 被取消或超时的错误
    Value(key any) any                         // 返回 context 中与 key 关联的值
}
```

`Context` 为接口，定义了四个核心 API：

- **Deadline**：返回 context 的过期时间；
- **Done**：返回 context 中的 channel；
- **Err**：返回错误；
- **Value**：返回 context 中的对应 key 的值。

### 1.2 标准 error

```go
var Canceled = errors.New("context canceled") // 定义一个表示 context 被取消的标准错误

var DeadlineExceeded error = deadlineExceededError{} // 定义一个表示 context 超时的标准错误

type deadlineExceededError struct{} // 定义一个空结构体，用于实现 DeadlineExceeded 错误

func (deadlineExceededError) Error() string   { return "context deadline exceeded" } // 实现 Error 方法，返回错误信息
func (deadlineExceededError) Timeout() bool   { return true } // 实现 Timeout 方法，表示这是一个超时错误
func (deadlineExceededError) Temporary() bool { return true } // 实现 Temporary 方法，表示这个错误是暂时的
```

- **Canceled**：context 被 cancel 时会报此错误；
- **DeadlineExceeded**：context 超时时会报此错误。

## 2 `emptyCtx`

### 2.1 类的实现
[[context包概括.svg]]

```go
type emptyCtx int // 定义 emptyCtx 类型为 int，用于表示一个空的 context

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return // 返回空的时间和 false，表示没有截止时间
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil // 返回 nil channel，表示永远不会被取消
}

func (*emptyCtx) Err() error {
    return nil // 返回 nil 错误，表示没有错误发生
}

func (*emptyCtx) Value(key any) any {
    return nil // 返回 nil，表示没有值存储
}
```

- `emptyCtx` 是一个空的 context，本质上类型为一个整型；
- **Deadline** 方法会返回一个零值的时间以及 `false` 的 flag，标识当前 context 不存在过期时间；
- **Done** 方法返回一个 `nil` 值，用户无论往 `nil` 中写入或者读取数据，均会陷入阻塞；
- **Err** 方法返回的错误永远为 `nil`；
- **Value** 方法返回的 value 同样永远为 `nil`。

### 2.2 `context.Background()` & `context.TODO()`

```go
var (
    background = new(emptyCtx) // 创建一个 emptyCtx 实例，用于 background context
    todo       = new(emptyCtx) // 创建一个 emptyCtx 实例，用于 TODO context
)

func Background() Context {
    return background // 返回 background context 实例
}

func TODO() Context {
    return todo // 返回 TODO context 实例
}
```

- 我们所常用的 `context.Background()` 和 `context.TODO()` 方法返回的均是 `emptyCtx` 类型的一个实例。

## 3 `cancelCtx`
### Q1:协程创建时，context会形成一颗父子结构的树吗?
![[Pasted image 20241011002650.png|575]]

![[context树组成图示.svg|75]]

![[context树双向连接.svg]]

![[cancel向下传播.svg]]
![[Pasted image 20241013182218.png]]

![[Pasted image 20241013182226.png]]

### Q2:父context终止，会形成从上往下的树的单向传递的终止是吗?
![[Pasted image 20241011002941.png|825]]

### Q3:也就是说context是协程之间的生命周期联动属性？
![[Pasted image 20241011003209.png]]

#### q1 我们知道context之间的联动取消，但还不知道是如何通过context取消影响到goroutine的取消的。
`Context` 在 Go 中的取消机制是通过主动监听 `Context` 的 `Done` 通道实现的。
![[Pasted image 20241013183137.png|675]]
##### 什么是多路复用？

![[Pasted image 20241013183354.png|475]]

![[Pasted image 20241013183512.png|475]]
0
### 3.1 `cancelCtx` 数据结构
![[Pasted image 20241011002357.png]]


```go
type cancelCtx struct {
    Context // 嵌入一个父 context
    mu       sync.Mutex            // 定义一个互斥锁，用于保护后续字段的并发访问
    done     atomic.Value          // 使用 atomic.Value 存储一个 chan struct{}，用于表示 context 的完成状态，懒加载
    children map[canceler]struct{} // 存储所有子 canceler，首次 cancel 时设为 nil
    err      error                 // 存储 context 取消的错误，首次 cancel 时设置为非 nil
}

type canceler interface {
    cancel(removeFromParent bool, err error) // 定义 cancel 方法，移除子 context 并设置错误
    Done() <-chan struct{}                    // 定义 Done 方法，返回完成的 channel
}
```

- `cancelCtx` 嵌入了一个 `Context` 作为其父 context，可见，`cancelCtx` 必然为某个 context 的子 context；
- 内置了一把锁，用以协调并发场景下的资源获取；
- **done**：实际类型为 `chan struct{}`，即用以反映 `cancelCtx` 生命周期的通道；
- **children**：一个 set，指向 `cancelCtx` 的所有子 context；
- **err**：记录了当前 `cancelCtx` 的错误，必然为某个 context 的子 context。



### Q4：`cancelCtx`是怎么继承父context的能力的
- `cancelCtx` 通过嵌入一个 `Context` 接口，继承了父 `Context` 的所有方法（如 `Deadline`、`Value` 等）。这使得 `cancelCtx` 成为一个父子关系中的子节点，能够访问并继承父 `Context` 的特性。

Q：golang的结构体字段如果是一个接口类型，本质上就是对一个实现了这个接口的结构体的指针是吗？

是的，Go语言中如果结构体字段是接口类型，那么实际上它可以存储任何实现了该接口的类型的指针。这使得接口字段能够动态地指向不同的实现。


### Q5: done方法本质上只是用来查看context是否存活是吗?
是的，您理解得非常正确！`Done()` 方法的本质作用就是**用于检测 `Context` 是否已经被取消或完成**。它通过返回一个只读的 `chan struct{}` 通道，允许 goroutine 被动监听 `Context` 的状态，得知 `Context` 是否还“存活”。
### 只读管道读取时会阻塞，直到他被关闭？
是的，在 Go 中，从一个只读通道（管道）读取时，如果通道没有数据可供读取，读取操作会阻塞，直到以下两种情况之一发生：

1. **通道有数据写入**：当有数据被写入通道时，阻塞的读取操作会立刻获取到数据，并继续执行。
    
2. **通道被关闭**：如果通道被关闭，阻塞的读取操作会返回零值（通道的默认值），同时不再阻塞。通道被关闭后，即使没有数据，读取操作也不会再阻塞。
### 这就是为什么只读管道可以作为信号变量的原因？

是的，正因为只读通道在没有数据时会阻塞，直到有数据写入或通道关闭，这使得它非常适合用作信号变量。

### 作为信号变量的原因：

1. **阻塞特性**：当一个 Goroutine 等待从通道读取信号时，它会被阻塞，直到另一个 Goroutine 写入信号（如发送数据或关闭通道），这确保了同步行为。
    
2. **通道关闭表示信号完成**：当通道被关闭时，所有等待读取的 Goroutine 都会立刻从阻塞中被释放，收到信号表示某个事件（比如任务完成或取消）已经发生。
![[Pasted image 20241013183137.png|375]]
![[Pasted image 20241013191123.png|550]]



### 3.2 Deadline 方法
[[context包概括]]
#### Q1:cancelcontext本身不实现Deadline ，默认使用其父context的Deadline 方法，如果没有实现则返回nil？这个说法对吗？
![[Pasted image 20241011003431.png]]



- `cancelCtx` 未实现该方法，仅是嵌入了一个带有 `Deadline` 方法的 `Context` interface，因此倘若直接调用会报错。

### 3.3 Done 方法
![[Pasted image 20241011003256.png]]

```go
func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load() // 原子加载 done 字段的当前值
    if d != nil { // 如果 done 已经被初始化
        return d.(chan struct{}) // 直接返回已存在的 channel
    }
    c.mu.Lock() // 加锁，保护后续操作
    defer c.mu.Unlock() // 确保在函数结束时解锁
    d = c.done.Load() // 再次加载 done，防止并发初始化
    if d == nil { // 如果 done 仍未初始化
        d = make(chan struct{}) // 创建一个新的 channel
        c.done.Store(d) // 原子存储到 done 字段中
    }
    return d.(chan struct{}) // 返回初始化后的 channel
}
```

- 基于 atomic 包，读取 `cancelCtx` 中的 channel；倘若已存在，则直接返回；
- 加锁后，在此检查 channel 是否存在，若存在则返回；（双重检查）
- 初始化 channel 存储到 `atomic.Value` 当中，并返回。（懒加载机制）
## 1. 什么是“接收方 Context”？
**“接收方 Context”** 是指在函数或 goroutine 中接收和使用 Context 的部分。当函数或 goroutine 接收到一个 Context 参数时，它就成为了这个 Context 的“接收方”。这些接收方通常会监听 Context 的取消信号，以便在需要时及时终止操作。

**示例：**
```go
func DoWork(ctx context.Context) {
    // 接收方 Context
    select {
    case <-ctx.Done():
        // Context 被取消，终止操作
        return
    default:
        // 继续执行工作
    }
}
````

在这个例子中，`DoWork` 函数接收一个 Context，并通过 `select` 语句监听 `ctx.Done()` 通道，以便在 Context 被取消时终止操作。

## 2. “当 Context 被取消时，这个通道会被关闭” 是指 cancel 方法吗？

是的，当调用 cancel 方法时，Context 的 Done 通道会被关闭，从而通知所有监听该通道的接收方 Context 已被取消。

## 3. 关于 Done 通道的懒加载和创建

Done 通道的懒加载意味着只有在首次调用 Done() 方法时，Context 才会实际创建这个通道。这种设计可以节省资源，避免不必要的通道创建。

**创建 Context 时的行为：** 当通过 `WithCancel`、`WithDeadline`、`WithTimeout` 或 `WithValue` 等函数创建一个新的 Context 时，这些函数会返回一个新的 Context 实例和一个取消函数（CancelFunc）。在创建 Context 时，Done 通道并不会立即被创建，而是会在第一次调用 `Done()` 方法时才被创建。

**示例：**

go

```go

// WithCancel 返回父级的一个副本，并带有一个新的 Done 通道。  
// 返回的上下文的 Done 通道会在调用返回的取消函数时关闭，或当父上下文的 Done 通道关闭时关闭，以先发生者为准。  
//  
// 取消此上下文会释放与之关联的资源，因此代码应在此 Context 中运行的操作完成后立即调用 cancel。  
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {  
    c := withCancel(parent)  
    return c, func() { c.cancel(true, Canceled, nil) }  
}

-------------------
ctx, cancel := context.WithCancel(parent)
// 此时，ctx.Done() 尚未被调用，通道未被创建

// 只有在调用 ctx.Done() 时，通道才会被实际创建
doneChan := ctx.Done()
```

取消函数： 创建 Context 时返回的取消函数（如 `cancel`）是一个固定的函数，用于取消该 Context。调用取消函数会关闭 Done 通道，并通知所有监听者。

**示例：**

go

```go
ctx, cancel := context.WithCancel(parent)
defer cancel() // 确保在函数退出时取消 Context

go func() {
    select {
    case <-ctx.Done():
        // 接收到取消信号
    }
}()

// 在需要时调用取消函数
cancel()
```



## Q 调用cancel时，调用的context会从其父context的map集合里删除，并且关闭其done通道？
### 1. 调用 cancel 时的行为
当调用 cancel 方法时，会发生以下操作：

- **从父 Context 的子 Context 列表中移除**：子 Context 会从父 Context 的子 Context 列表中删除，释放资源。这通过 `removeChild` 方法实现。
- **关闭 Done 通道**：调用 cancel 后，子 Context 的 Done 通道会被关闭，通知所有监听该通道的 goroutine 停止操作。

**示例代码**：
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return
    }
    c.err = err
    d, _ := c.done.Load().(chan struct{})
    if d == nil {
        c.done.Store(closedchan) // 设置为已关闭的通道
    } else {
        close(d) // 关闭通道，通知所有监听者
    }
    for child := range c.children {
        child.cancel(false, err)
    }
    c.children = nil // 清空子 Context 集合
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c) // 从父 Context 的子列表中移除
    }
}
````


### 5. 监听 Done 通道是由编程者自己控制还是 Go 管理？

监听 Done 通道主要由编程者（开发者）控制。Go 语言本身并不会自动为您监听 Context 的取消信号。相反，开发者需要在代码中明确地编写逻辑来监听和响应 Context 的取消。

**责任划分：**

- 编程者：负责在需要取消信号的地方编写监听 Done 通道的代码，如通过 `select` 语句、单独的 goroutine 等方式。
- Go 语言：提供了 Context 接口及其相关函数，帮助管理取消信号和传递 Context，但不自动监听或响应这些信号。

**示例：**

go

```go
func fetchData(ctx context.Context) error {
    // 创建一个 HTTP 请求，绑定到 Context
    req, err := http.NewRequestWithContext(ctx, "GET", "https://example.com/data", nil)
    if err != nil {
        return err
    }

    // 发起 HTTP 请求
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    // 处理响应数据
    // ...

    return nil
}

func main() {
    // 创建一个带有超时的 Context
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // 调用 fetchData，并传入 Context
    err := fetchData(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            fmt.Println("请求超时")
        } else {
            fmt.Println("请求失败:", err)
        }
    }
}
```

在这个例子中：

- 创建 Context：通过 `context.WithTimeout` 创建一个带有超时功能的 Context。
- 传递 Context：将 Context 传递给需要取消信号的函数 `fetchData`。
- 监听和响应：在 `fetchData` 函数中，HTTP 请求绑定到 Context，当 Context 被取消或超时时，HTTP 请求会自动取消。


### 3.4 Err 方法

```go
func (c *cancelCtx) Err() error {
    c.mu.Lock() // 加锁，保护 err 字段
    err := c.err // 读取 err 字段的值
    c.mu.Unlock() // 解锁
    return err // 返回错误
}
```

- 加锁；
- 读取 `cancelCtx.err`；
- 解锁；
- 返回结果。

### 3.5 Value 方法

```go
func (c *cancelCtx) Value(key any) any {
    if key == &cancelCtxKey { // 如果 key 是 cancelCtxKey 的指针
        return c // 返回当前 cancelCtx 实例
    }
    return value(c.Context, key) // 否则，委托给父 context 进行查找
}
```

- 倘若 key 特定值 `&cancelCtxKey`，则返回 `cancelCtx` 自身的指针；
- 否则遵循 `valueCtx` 的思路取值返回。
![[Pasted image 20241013193909.png]]


### 3.6 context.WithCancel()
![[withcancel全流程.svg]]

#### 3.6.1 context.WithCancel()

```go
func withCancel(parent Context) *cancelCtx {  
    // 检查父 Context 是否为 nil，如果是则抛出恐慌  
    if parent == nil {  
       panic("cannot create context from nil parent") // 如果父 Context 为 nil，则抛出恐慌  
    }  
    // 创建一个新的 cancelCtx 实例  
    c := &cancelCtx{}  
    // 调用 propagateCancel 方法，将取消信号传播到子 Context    
    c.propagateCancel(parent, c) // 传播取消信号  
    // 返回新的 cancelCtx 实例  
    return c  
}


func (c *cancelCtx) propagateCancel(parent Context, child canceler) {  
    // 设置当前上下文的父级上下文  
    c.Context = parent  
  
    // 获取父级上下文的 Done 通道  
    done := parent.Done()  
    if done == nil {  
       // 如果父级上下文永不取消，直接返回  
       return  
    }  
  
    select {  
    case <-done:  
       // 如果父级上下文已被取消，取消子级上下文  
       child.cancel(false, parent.Err(), Cause(parent))  
       return  
    default:  
       // 如果父级上下文未被取消，继续执行  
    }  
  
    // 检查父级上下文是否为 *cancelCtx 类型  
    if p, ok := parentCancelCtx(parent); ok {  
       p.mu.Lock()  
       if p.err != nil {  
          // 如果父级上下文已被取消，取消子级上下文  
          child.cancel(false, p.err, p.cause)  
       } else {  
          // 如果父级上下文未被取消，将子级上下文添加到父级的子级集合中  
          if p.children == nil {  
             p.children = make(map[canceler]struct{})  
          }  
          p.children[child] = struct{}{}  
       }  
       p.mu.Unlock()  
       return  
    }  
  
    // 检查父级上下文是否实现了 afterFuncer 接口  
    if a, ok := parent.(afterFuncer); ok {  
       c.mu.Lock()  
       // 调用父级上下文的 AfterFunc 方法，安排取消子级上下文  
       stop := a.AfterFunc(func() {  
          child.cancel(false, parent.Err(), Cause(parent))  
       })  
       // 设置当前上下文为 stopCtx 类型  
       c.Context = stopCtx{  
          Context: parent,  
          stop:    stop,  
       }  
       c.mu.Unlock()  
       return  
    }  
  
    // 增加 goroutine 计数  
    goroutines.Add(1)  
    // 启动一个新的 goroutine，监听父级上下文的 Done 通道  
    go func() {  
       select {  
       case <-parent.Done():  
          // 如果父级上下文被取消，取消子级上下文  
          child.cancel(false, parent.Err(), Cause(parent))  
       case <-child.Done():  
          // 如果子级上下文被取消，退出 goroutine       }  
    }()  
}
```

- 校验父 context 非空；
- 注入父 context 构造好一个新的 `cancelCtx`；
- 在 `propagateCancel` 方法内启动一个守护协程，以保证父 context 终止时，该 `cancelCtx` 也会被终止；
- 将 `cancelCtx` 返回，连带返回一个用以终止该 `cancelCtx` 的闭包函数。
[[#3.6.3 propagateCancel]]


#### 3.6.2 newCancelCtx

```go
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent} // 返回一个新的 cancelCtx，父 context 为 parent
}
```

- 注入父 context 后，返回一个新的 `cancelCtx`。

#### 3.6.3 propagateCancel
![[Pasted image 20241013235955.png]]

```go
func propagateCancel(parent Context, child canceler) {
    done := parent.Done() // 获取父 context 的 Done channel
    if done == nil { // 如果父 context 不会被取消 （emptyctx）
        return // 直接返回，因为没有取消需要传播
    }

    select {
    case <-done: // 如果父 context 已经被取消
        child.cancel(false, parent.Err()) // 立即取消子 context，并传递父 context 的错误
        return
    default: // 如果父 context 未被取消，继续执行
    }

    if p, ok := parentCancelCtx(parent); ok { //寻找最近的父cancelctx （找有取消能力的ctx）
    //IsparentCancelctx（）
        p.mu.Lock() // 加锁父 cancelCtx
        if p.err != nil { // 如果父 context 已经被取消
            child.cancel(false, p.err) // 立即取消子 context，并传递父 context 的错误
        } else {
            if p.children == nil { // 如果父 context 的 children 集合尚未初始化
                p.children = make(map[canceler]struct{}) // 初始化 children 集合
            }
            p.children[child] = struct{}{} // 将子 context 添加到父 context 的 children 集合中
        }
        p.mu.Unlock() // 解锁父 cancelCtx
    } else { // 如果父 context 不是 cancelCtx 类型，但支持取消
        atomic.AddInt32(&goroutines, +1) // 原子增加 goroutine 计数
        go func() { // 启动一个新的 goroutine 监听父 context 的取消
            select {
            case <-parent.Done(): // 当父 context 被取消
                child.cancel(false, parent.Err()) // 取消子 context，并传递父 context 的错误
            case <-child.Done(): // 当子 context 被取消
            }
        }()
    }
}
```

- `propagateCancel` 方法顾名思义，用以传递父子 context 之间的 cancel 事件；
- 倘若 parent 是不会被 cancel 的类型（如 `emptyCtx`），则直接返回；
- 倘若 parent 已经被 cancel，则直接终止子 context，并以 parent 的 err 作为子 context 的 err；
- 假如 parent 是 `cancelCtx` 的类型，则加锁，并将子 context 添加到 parent 的 children map 当中；
- 假如 parent 不是 `cancelCtx` 类型，但又存在 cancel 的能力（比如用户自定义实现的 context），则启动一个协程，通过多路复用的方式监控 parent 状态，倘若其终止，则同时终止子 context，并透传 parent 的 err。

```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {  
    // 获取父级 Context 的 Done 通道  
    done := parent.Done()  
    // 如果 Done 通道已关闭或为 nil，返回 nil 和 false    if done == closedchan || done == nil {  
       return nil, false  
    }  
    // 尝试从父级 Context 中获取 *cancelCtx    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)  
    // 如果获取失败，返回 nil 和 false    if !ok {  
       return nil, false  
    }  
    // 获取 *cancelCtx 的 Done 通道  
    pdone, _ := p.done.Load().(chan struct{})  
    // 如果 Done 通道不匹配，返回 nil 和 false    if pdone != done {  
       return nil, false  
    }  
    // 返回 *cancelCtx 和 true    return p, true  
}
```

- 倘若 parent 的 channel 已关闭或者是不会被 cancel 的类型，则返回 false；
- 倘若以特定的 `cancelCtxKey` 从 parent 中取值，取得的 value 是 `cancelCtx`，则返回 true。
- ![[Pasted image 20241014002410.png]]
![[Pasted image 20241014003538.png]]![[Pasted image 20241014003607.png]]
### 具体说明：

1. **派生新的 `cancelCtx`**：
    
    - 当你调用 `context.WithCancel()`、`context.WithTimeout()` 或 `context.WithDeadline()` 时，会创建一个新的 `cancelCtx`。每个新的 `cancelCtx` 都会有一个**自己独立的 `Done` 通道**，用于处理取消信号。
    - 这些新的 `cancelCtx` 会**不复用**父 `Context` 的 `Done` 通道。它们创建了新的取消机制，意味着它们的 `Done` 通道是独立的，且有各自的生命周期。
2. **复用父级 `cancelCtx` 的 `Done` 通道**：
    
    - 如果没有派生新的 `cancelCtx`，子 `Context` 可能会复用父上下文的 `Done` 通道。这种情况发生在：
        - 父级上下文已经有一个 `cancelCtx`，即父上下文本身是通过 `context.WithCancel()` 或其他类似方法创建的，并且已经有了一个 `Done` 通道。
        - 当子 `Context` 没有通过 `context.WithCancel()` 等方法派生出新的取消上下文（例如直接用 `context.WithValue()` 派生子上下文时），子上下文会复用父级的 `Done` 通道。这意味着，当父上下文取消时，子上下文也会被取消。
首先parentCancelCtx会从调用的context找到最近的父cancelctx，换句话说要不就是找到它本身，要不就是找到最近的cancelctx父级，并且第二种情况默认了此context是一个没有被新包装cancelctx字段的新context，比如valuectx这样新建时不会新包装cancelctx字段的context。而对于这两种情况，他们的两个通道就应该是同一个，第一张情况就是自身通道等于自身通道，第二种情况就是因为此context一路上没有新建包装新的cancelctx，自然复用了第一个父context的通道。

### 3.6.4 cancelCtx.cancel

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil { // 检查传入的错误是否为 nil
        panic("context: internal error: missing cancel error") // 如果是 nil，抛出 panic，表示内部错误
    }
    
    c.mu.Lock() // 加锁以保护对共享状态的修改
    if c.err != nil { // 如果当前 Context 已经被取消
        c.mu.Unlock() // 解锁并返回，避免重复取消
        return
    }
    
    c.err = err // 设置取消的错误信息
    
    d, _ := c.done.Load().(chan struct{}) // 加载当前的 done channel
    if d == nil { // 如果 done channel 尚未初始化
        c.done.Store(closedchan) // 将其设置为已关闭的 channel，表示取消状态
    } else {
        close(d) // 关闭已初始化的 done channel，通知所有等待的 goroutine
    }
    
    // 遍历所有子 context，递归取消它们
    for child := range c.children { 
        // NOTE: 在持有父级锁的同时获取子级的锁，这里需要小心避免死锁。
        child.cancel(false, err) // 取消子 context，并传递父 context 的错误信息
    }
    
    c.children = nil // 清空 children 集合，释放引用
    c.mu.Unlock() // 解锁，允许其他 goroutine 访问

    if removeFromParent { // 如果需要从父 context 中移除当前 context
        removeChild(c.Context, c) // 调用 removeChild 方法进行移除
    }
}

```

- `cancelCtx.cancel` 方法有两个入参，第一个 `removeFromParent` 是一个 bool 值，表示当前 context 是否需要从父 context 的 children set 中删除；第二个 `err` 则是 cancel 后需要展示的错误；
- 进入方法主体，首先校验传入的 `err` 是否为空，若为空则 panic；
- 加锁；
- 校验 `cancelCtx` 自带的 `err` 是否已经非空，若非空说明已被 cancel，则解锁返回；
- 将传入的 `err` 赋给 `cancelCtx.err`；
- 处理 `cancelCtx` 的 channel，若 channel 此前未初始化，则直接注入一个 `closedChan`，否则关闭该 channel；
- ### **原因**
```go
func DoWork(ctx context.Context) {
    // 接收方 Context
    select {
    case <-ctx.Done():
        // Context 被取消，终止操作
        return
    default:
        // 继续执行工作
    }
}
````

1. **如果 `done` 通道未初始化，说明还没有 goroutine 在等待，可以直接使用已关闭的共享通道 `closedchan`，节省资源。**
    
    - **未初始化的含义**：当 `done` 通道为 `nil` 时，表示尚未有人调用过 `c.Done()` 方法，也就是说，没有 goroutine 正在等待这个上下文的取消信号。
    - **节省资源**：
        - **避免创建新的通道**：如果没有 goroutine 在等待，我们无需为此上下文专门创建一个新的通道。
        - **使用共享的已关闭通道**：`closedchan` 是一个全局共享的、已关闭的通道。将 `c.done` 设置为 `closedchan`，意味着所有未来调用 `c.Done()` 的 goroutine 都会立即收到取消信号，而无需额外的资源开销。
    - **优化效果**：这种做法减少了内存分配和垃圾回收的压力，提高了程序的性能，尤其是在大量创建和取消上下文的场景下。
2. **如果 `done` 通道已初始化，说明有 goroutine 在等待，关闭该通道会立即通知它们。**
    
    - **已初始化的含义**：当 `done` 通道不为 `nil` 时，表示已经有 goroutine 调用了 `c.Done()` 方法，并获取了该通道。这些 goroutine 正在等待通道的关闭信号，以得知上下文何时被取消。
    - **需要通知等待者**：
        - **关闭已有的通道**：通过调用 `close(d)`，关闭已初始化的 `done` 通道，所有等待在该通道上的 goroutine 都会立即从接收操作中返回，收到上下文已被取消的通知。
        - **保证正确性**：确保所有已经开始等待的 goroutine 都能及时得知上下文的取消状态，避免逻辑上的错误或死锁。
- 遍历当前 `cancelCtx` 的 children set，依次将 children context 都进行 cancel；
- 解锁。
- 根据传入的 `removeFromParent` flag 判断是否需要手动把 `cancelCtx` 从 parent 的 children set 中移除。

走进 `removeChild` 方法中，观察如何将 `cancelCtx` 从 parent 的 children set 中移除：

```go
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent) // 尝试获取父 cancelCtx
    if !ok { // 如果不是 cancelCtx 类型
        return // 直接返回
    }
    p.mu.Lock() // 加锁父 cancelCtx
    if p.children != nil { // 如果父 context 有 children 集合
        delete(p.children, child) // 从 children 集合中删除子 context
    }
    p.mu.Unlock() // 解锁
}
```

- 如果 parent 不是 `cancelCtx`，直接返回（因为只有 `cancelCtx` 才有 children set）；
- 加锁；
- 从 parent 的 children set 中删除对应 child；
- 解锁返回。

## 4 `timerCtx`

### 4.1 类

```go
type timerCtx struct {
    cancelCtx // 嵌入 cancelCtx，继承其取消能力
    timer    *time.Timer // 定时器，用于在 deadline 到达时取消 context，受 cancelCtx.mu 保护
    deadline time.Time    // 记录 context 的过期时间
}
```

- `timerCtx` 在 `cancelCtx` 基础上又做了一层封装，除了继承 `cancelCtx` 的能力之外，新增了一个 `time.Timer` 用于定时终止 context；另外新增了一个 `deadline` 字段用于记录 `timerCtx` 的过期时间。

### 4.2 `timerCtx.Deadline()`

```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true // 返回 deadline 字段和 true，表示存在截止时间
}
```

- `context.Context` 接口下的 `Deadline` API 仅在 `timerCtx` 中有效，用于展示其过期时间。

### 4.3 `timerCtx.cancel`

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err) // 调用嵌入的 cancelCtx 的 cancel 方法，取消 context
    if removeFromParent { // 如果需要从父 context 中移除
        removeChild(c.cancelCtx.Context, c) // 调用 removeChild 方法移除
    }
    c.mu.Lock() // 加锁，保护 timer 字段
    if c.timer != nil { // 如果 timer 已经存在
        c.timer.Stop() // 停止 timer，防止定时器回调
        c.timer = nil // 将 timer 置为 nil
    }
    c.mu.Unlock() // 解锁
}
```

- 复用继承的 `cancelCtx` 的 cancel 能力，进行 cancel 处理；
- 判断是否需要手动从 parent 的 children set 中移除，若是则进行处理；
- 加锁；
- 停止 `time.Timer`；
- 解锁返回。

### 4.4 `context.WithTimeout` & `context.WithDeadline`

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout)) // 调用 WithDeadline 方法，设置截止时间为当前时间加上 timeout
}
```

- `context.WithTimeout` 方法用于构造一个 `timerCtx`，本质上会调用 `context.WithDeadline` 方法：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil { // 检查父 context 是否为 nil
        panic("cannot create context from nil parent") // 如果是 nil，抛出 panic
    }
    if cur, ok := parent.Deadline(); ok && cur.Before(d) { // 如果父 context 有截止时间，并且父的截止时间早于新的截止时间
        // The current deadline is already sooner than the new one.
        return WithCancel(parent) // 仅调用 WithCancel，忽略新的截止时间
    }
    c := &timerCtx{ // 创建一个新的 timerCtx 实例
        cancelCtx: newCancelCtx(parent), // 初始化嵌入的 cancelCtx，父 context 为 parent
        deadline:  d, // 设置截止时间
    }
    propagateCancel(parent, c) // 传播取消信号，将 timerCtx 作为子 context 关联到 parent
    dur := time.Until(d) // 计算截止时间与当前时间的持续时间
    if dur <= 0 { // 如果截止时间已过
        c.cancel(true, DeadlineExceeded) // 立即取消 timerCtx，并设置错误为 DeadlineExceeded
        return c, func() { c.cancel(false, Canceled) } // 返回 timerCtx 和取消函数
    }
    c.mu.Lock() // 加锁，保护 timer 字段
    defer c.mu.Unlock() // 确保解锁
    if c.err == nil { // 如果 context 尚未被取消
        c.timer = time.AfterFunc(dur, func() { // 设置一个定时器，在持续时间后执行取消操作
            c.cancel(true, DeadlineExceeded) // 取消 timerCtx，并设置错误为 DeadlineExceeded
        })
    }
    return c, func() { c.cancel(true, Canceled) } // 返回 timerCtx 和取消函数
}
```

- 校验 parent context 非空；
- 校验 parent 的过期时间是否早于自己，若是，则构造一个 `cancelCtx` 返回即可；
- 构造出一个新的 `timerCtx`；
- 启动守护方法，同步 parent 的 cancel 事件到子 context；
- 判断过期时间是否已到，若是，直接 cancel `timerCtx`，并返回 `DeadlineExceeded` 的错误；
- 加锁；
- 启动 `time.Timer`，设定一个延时时间，即达到过期时间后会终止该 `timerCtx`，并返回 `DeadlineExceeded` 的错误；
- 解锁；
- 返回 `timerCtx`，以及一个封装了 cancel 逻辑的闭包 cancel 函数。

## 5 `valueCtx`

### 5.1 类

```go
type valueCtx struct {
    Context        // 嵌入一个父 context
    key, val any      // 存储一个键值对
}
```

- `valueCtx` 同样继承了一个 parent context；
- 一个 `valueCtx` 中仅有一组 kv 对。

### 5.2 `valueCtx.Value()`

```go
func (c *valueCtx) Value(key any) any {
    if c.key == key { // 如果当前 valueCtx 的 key 与传入的 key 相等
        return c.val // 返回对应的 value
    }
    return value(c.Context, key) // 否则，委托给父 context 进行查找
}
```

- 假如当前 `valueCtx` 的 key 等于用户传入的 key，则直接返回其 value；
- 假如不等，则从 parent context 中依次向上寻找。

```go
func value(c Context, key any) any {
    for { // 开始一个无限循环
        switch ctx := c.(type) { // 类型断言父 context
        case *valueCtx: // 如果是 valueCtx 类型
            if key == ctx.key { // 如果 key 匹配
                return ctx.val // 返回对应的 value
            }
            c = ctx.Context // 否则，继续向上查找父 context
        case *cancelCtx: // 如果是 cancelCtx 类型
            if key == &cancelCtxKey { // 如果 key 是 cancelCtxKey 的指针
                return c // 返回当前 cancelCtx 实例
            }
            c = ctx.Context // 否则，继续向上查找父 context
        case *timerCtx: // 如果是 timerCtx 类型
            if key == &cancelCtxKey { // 如果 key 是 cancelCtxKey 的指针
                return &ctx.cancelCtx // 返回嵌入的 cancelCtx 实例
            }
            c = ctx.Context // 否则，继续向上查找父 context
        case *emptyCtx: // 如果是 emptyCtx 类型
            return nil // 返回 nil，因为 emptyCtx 不存储值
        default: // 其他自定义 context 类型
            return c.Value(key) // 调用自定义的 Value 方法
        }
    }
}
```

- 启动一个 for 循环，由下而上，由子及父，依次对 key 进行匹配；
- 其中 `cancelCtx`、`timerCtx`、`emptyCtx` 类型会有特殊的处理方式；
- 找到匹配的 key，则将该组 value 进行返回。

### 5.3 `valueCtx` 用法小结

阅读源码可以看出，`valueCtx` 不适合视为存储介质，存放大量的 kv 数据，原因有三：

- 一个 `valueCtx` 实例只能存一个 kv 对，因此 n 个 kv 对会嵌套 n 个 `valueCtx`，造成空间浪费；
- 基于 key 寻找 value 的过程是线性的，时间复杂度 O(N)；
- 不支持基于 key 的去重，相同 key 可能重复存在，并基于起点的不同，返回不同的 value。由此得知，`valueContext` 的定位类似于请求头，只适合存放少量作用域较大的全局 meta 数据。

### 5.4 `context.WithValue()`

```go
func WithValue(parent Context, key, val any) Context {
    if parent == nil { // 检查父 context 是否为 nil
        panic("cannot create context from nil parent") // 如果是 nil，抛出 panic
    }
    if key == nil { // 检查 key 是否为 nil
        panic("nil key") // 如果是 nil，抛出 panic
    }
    if !reflectlite.TypeOf(key).Comparable() { // 检查 key 的类型是否可比较
        panic("key is not comparable") // 如果不可比较，抛出 panic
    }
    return &valueCtx{parent, key, val} // 返回一个新的 valueCtx，包含父 context 和键值对
}
```

- 倘若 parent context 为空，panic；
- 倘若 key 为空 panic；
- 倘若 key 的类型不可比较，panic；
- 包括 parent context 以及 kv 对，返回一个新的 `valueCtx`。




---



---

## 3. **`cancelCtx` 本身不实现 `Deadline`，默认使用其父 `Context` 的 `Deadline` 方法，如果没有实现则返回 `nil`，这个说法对吗？**

这个说法是正确的。`cancelCtx` 本身不实现 `Deadline` 方法。它是通过**嵌入父 `Context`** 的方式继承了父 `Context` 的 `Deadline` 行为。也就是说，**当 `cancelCtx` 调用 `Deadline` 时，它会直接调用其父 `Context` 的 `Deadline` 方法**。

如果父 `Context` 实现了 `Deadline`，`cancelCtx` 会返回父 `Context` 的截止时间；如果父 `Context` 没有实现 `Deadline`（例如 `context.Background()`），那么 `cancelCtx` 调用 `Deadline` 时将返回零值（`time.Time{}`）和 `false`，表示没有设置截止时间。

---

## 4. **`cancelCtxKey` 是什么？**

`cancelCtxKey` 是在 `Go` 的 `context` 包内部使用的一个**标识符**，用于在 `Context` 中存储和获取 `cancelCtx`。它是一个全局变量，通常作为 `Value()` 函数的键来检索 `cancelCtx`。

`cancelCtxKey` 的主要作用是在通过 `context.WithCancel()` 等函数创建 `cancelCtx` 时，能够在 `Context` 树中定位和传递取消操作。虽然 `cancelCtxKey` 在 `context` 包中是未导出的（私有的），但它在实现 `Context` 的取消机制时起到了重要作用。

---



---

## 1. `Context` 的树设计是否使用了组合模式？

**问**：`Context` 的树设计是否使用了组合模式？

**答**：是的，`Go` 的 `Context` 设计可以视为一种组合模式的应用。组合模式允许将对象组合成树形结构以表示“部分-整体”的层次结构，父 `Context` 和子 `Context` 形成了树形结构，通过嵌套组合实现键值对的管理和取消信号的传递。

---

## 2. 在 `valueCtx` 结构体中，`Context` 字段是否是原 `Context` 的指针？

**问**：在 `valueCtx` 结构体中，`Context` 字段是否是原 `Context` 的指针？

**答**：是的，`valueCtx` 中的 `Context` 字段实际上是指向原 `Context` 的指针，而不是将原 `Context` 完整地拷贝。每次创建新的 `valueCtx` 时，都是通过嵌套的方式，将父 `Context` 的指针保存到 `valueCtx` 中，以节省内存。

---

## 3. 每次派生一个新的 `Context` 时，是否是在形成树的新节点时加上一个被包装好的指针壳？

**问**：每次派生一个新的 `Context` 时，是否是在形成树的新节点时加上一个被包装好的指针壳？

**答**：是的，新节点通过包装对原 `Context` 的指针来形成树形结构。每次创建新的 `Context` 时，新的节点只增加自己的字段，不会复制整个原 `Context`，形成“壳子”加指针的结构，从而节省内存并保持高效。

---

## 4. 对于 `cancel` 方法，它的主要职责是什么？

**问**：对于 `cancel` 方法，它的主要职责是什么？

**答**：`cancel` 方法的主要职责包括：

1. 设置取消状态（记录错误）。
2. 关闭 `Done` 通道，通知所有监听者 `Context` 已取消。
3. 递归取消所有子 `Context`，确保整个 `Context` 树能够响应取消信号。
4. 从父 `Context` 的 `children` 集合中移除自己，以维护结构的清晰性。

---

## 5. 在 `value` 函数中，遇到非 `valueCtx` 类型的上下文时，做了什么？

**问**：在 `value` 函数中，遇到非 `valueCtx` 类型的上下文时，做了什么？

**答**：当遇到非 `valueCtx` 类型的 `Context`（如 `cancelCtx`、`withoutCancelCtx` 和 `timerCtx`）时，`value` 函数会检查传入的 `key`。对于 `cancelCtx` 和 `timerCtx`，如果 `key` 匹配特定的取消相关的键（`&cancelCtxKey`），则直接返回对应的 `cancelCtx`。这样做可以高效地访问取消相关的信息，而不需要遍历整个上下文链。

---

## 6. 如果要找到最近的 `cancelCtx` 祖先，该如何操作？

**问**：如果要找到最近的 `cancelCtx` 祖先，该如何操作？

**答**：您可以调用 `value` 方法，并将 `key` 设置为 `&cancelCtxKey`。这样，`value` 函数会遍历上下文链，找到最近的 `cancelCtx`，并返回该实例。例如，调用 `ctx.Value(&cancelCtxKey)` 可以有效地查找与取消相关的上下文信息。

---

## 7. `value` 函数的主要任务是什么？

**问**：`value` 函数的主要任务是什么？

**答**：`value` 函数的主要任务是从当前的 `Context` 节点开始，向上遍历上下文链，查找与指定的 `key` 关联的值。如果找到匹配的值，则返回该值；如果没有找到，则返回 `nil`。

---

## 解释一下：func Cause(c Context) error 

Q所以cause通过调用value逐级向上寻找最近的父cancelctx，找到了就复用他的cause，没有找到就使用自身的err字段？

没错，`cause` 通过 `Value` 方法逐级向上查找最近的父级 `cancelCtx`。如果找到了，就复用其 `cause` 字段；如果没有找到，就使用自身的 `Err` 字段返回错误信息。这样确保了能够正确获取取消的原因，或者返回上下文的标准错误。

Q所以value方法的主任务是向上查找valuectx的键值对是否存在，而在特地传入cancelctxkey的时候则是向上找到最近的父cancelctx？

是的，`Context` 的 `Value` 方法的主要任务是沿着上下文链向上查找键值对。每个 `Context` 实现都可以通过键值对存储信息，而 `Value` 方法会从当前的上下文开始，逐级向上查找，直到找到与指定键匹 配的值。

具体到你提到的情况：

1. **查找键值对的过程**：`Value` 方法是递归的，从当前的上下文（可能是某种包装上下文，如 `cancelCtx` 或 `valueCtx`）向其父级上下文查找是否存在与传入的键匹配的键值对。
    
2. **查找 `cancelCtxKey`**：当你传入 `cancelCtxKey` 时，`Value` 方法会从当前上下文开始，逐层查找。如果遇到某个上下文（通常是 `cancelCtx`）存储了与 `cancelCtxKey` 相关联的值（即 `cancelCtx` 本身），那么查找结束，并返回这个值。这个值通常是一个 `cancelCtx` 类型的对象，包含取消的相关信息（如 `cause`）。
    
3. **复用父 `cancelCtx` 的 `cause`**：如果 `Value` 找到了最近的父级 `cancelCtx`，那么函数 `Cause` 可以从该上下文中提取 `cause`。如果没有找到 `cancelCtx`，则函数会使用 `c.Err()` 来获取当前上下文的标准错误信息。
    

总结来说，当传入 `cancelCtxKey` 时，`Value` 方法的目标是沿着上下文链向上找到最近的 `cancelCtx`，然后提取它的取消原因或相关信息。这种设计支持了上下文的继承机制，确保子上下文能够从父上下文中获取必要的信息。


#### 1. 父 `Context` 是否具备取消能力

**判断标准**：

- **从 `*cancelCtx` 继承**：如果一个 `Context` 是通过 `WithCancel`、`WithDeadline` 或 `WithTimeout` 创建的，那么它内部会有一个 `*cancelCtx`，具备取消能力。
- **`Done` 通道匹配**：要确保这个 `*cancelCtx` 的 `Done` 通道与父 `Context` 的 `Done` 通道是同一个，这样取消信号才能正确传播。

**实现方式**：

- `parentCancelCtx` 函数首先调用 `parent.Done()` 获取父 `Context` 的 `Done` 通道。
- 然后通过 `parent.Value(&cancelCtxKey).(*cancelCtx)` 尝试从父 `Context` 中提取出内部的 `*cancelCtx`。
- 最后，检查提取出的 `*cancelCtx` 的 `done` 通道是否与父 `Context` 的 `Done` 通道相同。

如果以上条件都满足，说明父 `Context` 具备取消能力，并且能够正确管理其子 `Context` 的取消信号。

#### 2. 为什么要检查 `done` 通道是否相同？

**原因**：

- **自定义 `Context` 实现**：开发者可能创建了自定义的 `Context`，包装了原有的 `*cancelCtx`，并提供了不同的 `Done` 通道。
- **避免干扰**：如果 `Done` 通道不同，直接使用内部的 `*cancelCtx` 可能会导致取消信号无法正确传播，或引发其他不可预见的问题。

**具体说明**：

- 如果 `p.done.Load().(chan struct{})` （即 `*cancelCtx` 的 `done` 通道）与 `parent.Done()` 返回的 `done` 通道不一致，说明 `*cancelCtx` 已被自定义 `Context` 包装，提供了不同的 `Done` 通道。
- 这种情况下，直接使用内部的 `*cancelCtx` 可能会绕过自定义的 `Done` 通道，导致取消信号无法正确传播。
- 为了保证取消机制的完整性和正确性，`parentCancelCtx` 会返回 `nil, false`，表示无法从父 `Context` 中提取出有效的 `*cancelCtx`，从而避免潜在的问题。

#### 3. 一句话描述三个函数的作用

- **`propagateCancel`**：  
    **`propagateCancel`** 的
    
    建立父 `Context` 与子 `Context` 之间的取消关系，确保当父 `Context` 被取消时，子 `Context` 也会被取消。
    
- **`parentCancelCtx`**：
    
    检查并提取父 `Context` 中的内部 `*cancelCtx`，以确认父 `Context` 是否具备取消能力并能正确管理子 `Context` 的取消信号。
    
- **`cancel`**：
    
    取消当前 `Context`，关闭其 `Done` 通道，并将取消信号传播给所有子 `Context`。
    

### 总结

- **`parentCancelCtx` 的作用**：检查一个父 `Context` 是否具备取消能力，并确保其内部的 `*cancelCtx` 能正确管理子 `Context` 的取消信号。这是通过验证父 `Context` 是否包含 `*cancelCtx` 且其 `Done` 通道匹配来实现的。
    
- **取消信号的管理**：通过确保父 `Context` 和内部 `*cancelCtx` 的 `Done` 通道一致，`parentCancelCtx` 确保了取消信号能够正确地从父 `Context` 传播到所有子 `Context`，避免了在自定义实现中可能出现的信号传播问题。
    
- **防止绕过取消机制**：如果发现父 `Context` 的 `Done` 通道与内部 `*cancelCtx` 的 `done` 通道不一致，`parentCancelCtx` 会返回 `nil, false`，避免直接使用 `*cancelCtx`，从而维护取消机制的完整性和安全性。