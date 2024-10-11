# Golang Context 实现原理

---

## 0 前言

`context` 是 Golang 中的经典工具，主要在异步场景中用于实现并发协调以及对 goroutine 的生命周期控制。除此之外，`context` 还兼有一定的数据存储能力。本着知其然知其所以然的精神，本文和大家一起深入 `context` 源码一探究竟，较为细节地对其实现原理进行梳理。

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
![[Pasted image 20241011003029.png]]



### Q2:父context终止，会形成从上往下的树的单向传递的终止是吗?
![[Pasted image 20241011002941.png|825]]

### Q3:也就是说是协程之间的生命周期联动属性？
![[Pasted image 20241011003209.png]]

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


### Q5: done方法本质上只是用来查看context是否存活是吗?
是的，您理解得非常正确！`Done()` 方法的本质作用就是**用于检测 `Context` 是否已经被取消或完成**。它通过返回一个只读的 `chan struct{}` 通道，允许 goroutine 被动监听 `Context` 的状态，得知 `Context` 是否还“存活”。



### 3.2 Deadline 方法
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

### 3.6 context.WithCancel()

#### 3.6.1 context.WithCancel()

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil { // 检查父 context 是否为 nil
        panic("cannot create context from nil parent") // 如果是 nil，抛出 panic
    }
    c := newCancelCtx(parent) // 创建一个新的 cancelCtx，父 context 为 parent
    propagateCancel(parent, &c) // 传播取消信号，将 cancelCtx 作为子 context 关联到 parent
    return &c, func() { c.cancel(true, Canceled) } // 返回新的 cancelCtx 和一个取消函数
}
```

- 校验父 context 非空；
- 注入父 context 构造好一个新的 `cancelCtx`；
- 在 `propagateCancel` 方法内启动一个守护协程，以保证父 context 终止时，该 `cancelCtx` 也会被终止；
- 将 `cancelCtx` 返回，连带返回一个用以终止该 `cancelCtx` 的闭包函数。

#### 3.6.2 newCancelCtx

```go
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent} // 返回一个新的 cancelCtx，父 context 为 parent
}
```

- 注入父 context 后，返回一个新的 `cancelCtx`。

#### 3.6.3 propagateCancel

```go
func propagateCancel(parent Context, child canceler) {
    done := parent.Done() // 获取父 context 的 Done channel
    if done == nil { // 如果父 context 不会被取消
        return // 直接返回，因为没有取消需要传播
    }

    select {
    case <-done: // 如果父 context 已经被取消
        child.cancel(false, parent.Err()) // 立即取消子 context，并传递父 context 的错误
        return
    default: // 如果父 context 未被取消，继续执行
    }

    if p, ok := parentCancelCtx(parent); ok { // 尝试将父 context 转换为 cancelCtx
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

进一步观察 `parentCancelCtx` 是如何校验 parent 是否为 `cancelCtx` 的类型：

```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done() // 获取父 context 的 Done channel
    if done == closedchan || done == nil { // 如果 channel 已关闭或不存在
        return nil, false // 返回 nil 和 false，表示不是 cancelCtx 类型
    }
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx) // 尝试从父 context 中获取 cancelCtx
    if !ok { // 如果类型断言失败
        return nil, false // 返回 nil 和 false
    }
    pdone, _ := p.done.Load().(chan struct{}) // 原子加载父 cancelCtx 的 done channel
    if pdone != done { // 如果加载的 channel 不同
        return nil, false // 返回 nil 和 false
    }
    return p, true // 返回父 cancelCtx 和 true
}
```

- 倘若 parent 的 channel 已关闭或者是不会被 cancel 的类型，则返回 false；
- 倘若以特定的 `cancelCtxKey` 从 parent 中取值，取得的 value 是 `cancelCtx`，则返回 true。

### 3.6.4 cancelCtx.cancel

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil { // 检查传入的 err 是否为 nil
        panic("context: internal error: missing cancel error") // 如果是 nil，抛出 panic
    }
    c.mu.Lock() // 加锁，保护修改操作
    if c.err != nil { // 如果已经被取消
        c.mu.Unlock() // 解锁
        return // 直接返回，避免重复取消
    }
    c.err = err // 设置取消的错误
    d, _ := c.done.Load().(chan struct{}) // 加载 done channel
    if d == nil { // 如果 done channel 尚未初始化
        c.done.Store(closedchan) // 设置为已关闭的 channel
    } else {
        close(d) // 关闭 done channel，通知所有等待的 goroutine
    }
    for child := range c.children { // 遍历所有子 context
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err) // 取消子 context，并传递相同的错误
    }
    c.children = nil // 清空 children 集合
    c.mu.Unlock() // 解锁

    if removeFromParent { // 如果需要从父 context 中移除
        removeChild(c.Context, c) // 调用 removeChild 方法移除
    }
}
```

- `cancelCtx.cancel` 方法有两个入参，第一个 `removeFromParent` 是一个 bool 值，表示当前 context 是否需要从父 context 的 children set 中删除；第二个 `err` 则是 cancel 后需要展示的错误；
- 进入方法主体，首先校验传入的 `err` 是否为空，若为空则 panic；
- 加锁；
- 校验 `cancelCtx` 自带的 `err` 是否已经非空，若非空说明已被 cancel，则解锁返回；
- 将传入的 `err` 赋给 `cancelCtx.err`；
- 处理 `cancelCtx` 的 channel，若 channel 此前未初始化，则直接注入一个 `closedChan`，否则关闭该 channel；
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

# 结语

通过对 Golang `context` 包的源码解析，我们深入了解了 `context` 的核心数据结构及其实现原理。`context` 提供了强大的并发控制和数据传递能力，使得在复杂的异步编程场景中能够更加高效和安全地管理 goroutine 的生命周期和相关资源。理解其内部机制，有助于我们在实际开发中更好地运用 `context`，编写出更加健壮和高效的 Go 程序。