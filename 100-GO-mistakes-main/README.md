# 如何避免Go语言的100个错误

- 作者：[Teiva Harsanyi](http://teivah.io)
- 原名：[《100 Go Mistakes and How to Avoid Them》](https://www.manning.com/books/100-go-mistakes-and-how-to-avoid-them)
- 译者：[李天时](https://github.com/lts8989)
- 校订：[上杉夏香](https://github.com/ZMbiubiubiu)

> 使用 [Github Pages](https://lts8989.github.io/) 以获取最佳阅读体验。
>
> 本地：你可在项目根目录中执行 `make`，并通过浏览器阅读（[在线预览](https://lts8989.github.io/)）。

## 法律声明

从原作者处得知，已经有简体中文的翻译计划。[购买地址](https://search.jd.com/Search?keyword=go%20100%20mistakes)

译者纯粹出于 **学习目的** 与 **个人兴趣** 翻译本书，不追求任何经济利益。

译者保留对此版本译文的署名权，其他权利以原作者和出版社的主张为准。

本译文只供学习研究参考之用，不得公开传播发行或用于商业用途。有能力阅读英文书籍者请购买正版支持。

- [1 Go:入门容易，掌握难](chapter/1-go-simple-to-learn-but-hard-to-master/1-0-go-simple-to-learn-but-hard-to-master.md)
  - [1.1 Go 大纲](chapter/1-go-simple-to-learn-but-hard-to-master/1-1-go-outline.md)
  - [1.2 简单并不意味着容易](chapter/1-go-simple-to-learn-but-hard-to-master/1-2-simple-doesnt-mean-easy.md)
  - [1.3 100 个 Go 错误](chapter/1-go-simple-to-learn-but-hard-to-master/1-3-100-go-mistakes.md)
  - [1.4 本章总结](chapter/1-go-simple-to-learn-but-hard-to-master/1-4-chapter-summary.md)
- [2 代码和项目组织](chapter/2-Code-and-project-organization/2-0-Code-and-project-organization.md)
  - [2.1 意外的阴影变量](chapter/2-Code-and-project-organization/2-1-unintended-variable-shadowing.md)
  - [2.2 不必要的嵌套代码](chapter/2-Code-and-project-organization/2-2-unnecessary-nested-code.md)
  - [2.3 滥用的 init 函数](chapter/2-Code-and-project-organization/2-3-misusing-init-functions.md)
  - [2.4 过度使用 getters 和 setters](chapter/2-Code-and-project-organization/2-4-overusing-getters-and-setters.md)
  - [2.5 接口污染](chapter/2-Code-and-project-organization/2-5-interface-pollution.md)
  - [2.6 在生产者端的接口](chapter/2-Code-and-project-organization/2-6-interface-on-the-producer-side.md)
  - [2.7 返回接口](chapter/2-Code-and-project-organization/2-7-Returning-interfaces.md)
  - [2.8 `any` say nothing](chapter/2-Code-and-project-organization/2-8-any-says-nothing.md)
  - [2.9 令人困惑的泛型](chapter/2-Code-and-project-organization/2-9-being-confused-about-when-to-use-generics.md)
  - [2.10 没有意识到类型嵌入可能存在的问题](chapter/2-Code-and-project-organization/2-10-not-being-aware-of-the-possible.md)
  - [2.11 不使用函数式选项模式](chapter/2-Code-and-project-organization/2-11-not-using-the-functional-options-pattern.md)
  - [2.12 项目组织不善](chapter/2-Code-and-project-organization/2-12-project-misorganization.md)
  - [2.13 创建公共包](chapter/2-Code-and-project-organization/2-13-creating-utility-packages.md)
  - [2.14 忽略软件包名称冲突](chapter/2-Code-and-project-organization/2-14-ignoring-package-name-collisions.md)
  - [2.15 缺少代码文档](chapter/2-Code-and-project-organization/2-15-missing-code-documentation.md)
  - [2.16 不使用代码检查工具](chapter/2-Code-and-project-organization/2-16-not-using-linters.md)
- [3 数据类型](chapter/3-Data-types/3-0-Data-types.md)
  - [3.1 与八进制文字混淆](chapter/3-Data-types/3-1-creating-confusion-with-octal-literals.md)
  - [3.2 忽略整数溢出](chapter/3-Data-types/3-2-neglecting-integer-overflows.md)
  - [3.3 不被理解的浮点型](chapter/3-Data-types/3-3-not-understanding-floating-points.md)
  - [3.4 不被理解的切片长度和容量](chapter/3-Data-types/3-4-not-understanding-slice-length-and-capacity.md)
  - [3.5 低效的切片初始化](chapter/3-Data-types/3-5-inefficient-slice-initialization.md)
  - [3.6 令人困惑的切片，nil vs 空切片](chapter/3-Data-types/3-6-being-confused-about-nil-vs-empty-slice.md)
  - [3.7 没有正确检查切片是否为空](chapter/3-Data-types/3-7-not-properly-checking-if-a-slice-is-empty.md)
  - [3.8 无法正确复制切片](chapter/3-Data-types/3-8-not-making-slice-copy-correctly.md)
  - [3.9 使用切片附加的意外副作用](chapter/3-Data-types/3-9-unexpected-side-effects-using-slice-append.md)
  - [3.10 切片与内存泄漏](chapter/3-Data-types/3-10-slice-and-memory-leaks.md)
  - [3.11 低效的 map 初始化](chapter/3-Data-types/3-11-inefficient-map-initialization.md)
  - [3.12 映射和内存泄漏](chapter/3-Data-types/3-12-map-and-memory-leaks.md)
  - [3.13 错误的比较值](chapter/3-Data-types/3-13-comparing-values-incorrectly.md)
- [4 控制结构](chapter/4-control-structures/4-0-control-structures.md)
  - [4.1 在 `range` 循环中不被重视的元素副本](chapter/4-control-structures/4-1-ignoring-that-elements-are.md)
  - [4.2 在 `range` 循环中被忽视的参数求值方式](chapter/4-control-structures/4-2-ignoring-how-arguments-are-evaluated.md)
  - [4.3 在 `range` 循环中使用指针元素被忽略的影响](chapter/4-control-structures/4-3-ignoring-the-impacts-of-using-pointer.md)
  - [4.4 在 map 迭代过程中做出错误的假设](chapter/4-control-structures/4-4-making-wrong-assumptions-during.md)
  - [4.5 忽略 break 语句的工作方式](chapter/4-control-structures/4-5-ignoring-how-the-break-statement-work.md)
  - [4.6 在循环中使用 defer](chapter/4-control-structures/4-6-using-defer-inside-a-loop.md)
- [5 字符串](chapter/5-strings/5-0-strings.md)
  - [5.1 Go 中超难理解的符文(runes)概念](chapter/5-strings/5-1-not-understanding-the-concept-of-rune.md)
  - [5.2 字符串中有捣乱格式的字符，导致的字符串迭代混乱](chapter/5-strings/5-2-Inaccurate-string-iteration.md)
  - [5.3 被滥用的 trim 函数](chapter/5-strings/5-3-misusing-trim-functions.md)
  - [5.4 欠优化的字符串连接](chapter/5-strings/5-4-under-optimized-strings-concatenation.md)
  - [5.5 无用的字符串转换](chapter/5-strings/5-5-useless-string-conversion.md)
  - [5.6 子字符串与内存泄漏](chapter/5-strings/5-6-substring-and-memory-leaks.md)
- [6 函数与方法](chapter/6-functions-and-method/6-0-functions-and-method.md)
  - [6.1 接收器应该定义为值类型还是指针类型？](chapter/6-functions-and-method/6-1-Not-knowing-which-type-of-receiver-to-use.md)
  - [6.2 什么时候使用返回值参数](chapter/6-functions-and-method/6-2-Never-using-named-result-parameters.md)
  - [6.3 定义返回值参数的坑](chapter/6-functions-and-method/6-3-Unintended-side-effects-with-named-result-parameters.md)
  - [6.4 返回一个 nil 接收器引发的坑爹事件](chapter/6-functions-and-method/6-4-Returning-a-nil-receiver.md)
  - [6.5 不要使用文件名作为函数参数](chapter/6-functions-and-method/6-5-Using-a-filename-as-a-function-input.md)
  - [6.6 defer 函数中参数和接收器是如何被赋值的](chapter/6-functions-and-method/6-6-Ignoring-how-defer-arguments.md)
- [7 错误管理](chapter/7-error-management/7-0-error-management.md)
  - [7.1 恐慌(panicking) ](chapter/7-error-management/7-1-panicking.md)
  - [7.2 被忽略的包装(wrap)错误](chapter/7-error-management/7-2-Ignoring-when-to-wrap-an-error.md)
  - [7.3 错误类型比较的坑](chapter/7-error-management/7-3-Comparing-an-error-type-inaccurately.md)
  - [7.4 错误值比较的坑](chapter/7-error-management/7-4-Comparing-an-error-value-inaccurately.md)
  - [7.5 重复处理了两次错误](chapter/7-error-management/7-5-Handling-an-error-twice.md)
  - [7.6 忽略错误，引入的坑](chapter/7-error-management/7-6-Not-handling-a-error.md)
  - [7.7 不处理 defer 的错误，引入的坑](chapter/7-error-management/7-7-Not-handling-defer-errors.md)
- [8 并发：基础](chapter/8-Concurrency-Foundations/8-0-Concurrency-Foundations.md)
  - [8.1 并发和并行，傻傻分不清楚](chapter/8-Concurrency-Foundations/8-1-Mixing-concurrency-and-parallelism.md)
  - [8.2 并发并不总是更快](chapter/8-Concurrency-Foundations/8-2-concurrency-isnt-always-faster.md)
  - [8.3 并发业务应该使用 channel 还是 mutex](chapter/8-Concurrency-Foundations/8-3-Being-puzzled-about-when-to-use-channels-or-mutexes.md)
  - [8.4 竞争问题引入的坑](chapter/8-Concurrency-Foundations/8-4-Not-understanding-race-problems.md)
  - [8.5 goroutines 数量取决于工作负载类型](chapter/8-Concurrency-Foundations/8-5-Not-understanding-the-concurrency.md)
  - [8.6 令人头大的 Go 上下文](chapter/8-Concurrency-Foundations/8-6-Misunderstanding-Go-contexts.md)
- [9 并发：实践](chapter/9-Concurrency-Practice/9-0-Concurrency-Practice.md)
  - [9.1 传递不恰当的上下文](chapter/9-Concurrency-Practice/9-1-Propagating-an-inappropriate-context.md)
  - [9.2 goroutine 已启动，啥时候停，不确定](chapter/9-Concurrency-Practice/9-2-Starting-a-goroutine-without.md)
  - [9.3 goroutine 和循环变量，一不小心就出错](chapter/9-Concurrency-Practice/9-3-Not-being-careful-with-goroutines-and-loop-variables.md)
  - [9.4 期望使用 select 和 channel 的确定性执行](chapter/9-Concurrency-Practice/9-4-Expecting-a-deterministic-behavior-using-select-and-channels.md)
  - [9.5 不要使用布尔类型的通道](chapter/9-Concurrency-Practice/9-5-Not-using-notification-channels.md)
  - [9.6 不要使用零值通道](chapter/9-Concurrency-Practice/9-6-Not-using-nil-channels.md)
  - [9.7 通道缓冲区尺寸引入的坑](chapter/9-Concurrency-Practice/9-7-Being-puzzled-about-a-channel-size.md)
  - [9.8 忘记字符串格式化引入的坑](chapter/9-Concurrency-Practice/9-8-Forgetting-about-possible-side-effects.md)
  - [9.9 append 数据引发的数据竞争](chapter/9-Concurrency-Practice/9-9-Creating-data-races-with-append.md)
  - [9.10 切片和 map 使用互斥锁引入的坑](chapter/9-Concurrency-Practice/9-10-Using-mutexes-inaccurately-with-slices-and-maps.md)
  - [9.11 使用 sync.WaitGroup 引入的坑](chapter/9-Concurrency-Practice/9-11-Misusing-sync-WaitGroup.md)
  - [9.12 被遗忘的 sync.Cond](chapter/9-Concurrency-Practice/9-12-Forgetting-about-sync-Cond.md)
  - [9.13 多个 goroutine 并行执行，使用 errgroup 聚合返回错误](chapter/9-Concurrency-Practice/9-13-not-using.md)
  - [9.14 拷贝同步类型变量引入的坑](chapter/9-Concurrency-Practice/9-14-Copying-a-sync-type.md)
- [10 标准库](chapter/10-Standard-library/10-0-Standard-library.md)
  - [10.1 提供错误的时间周期(time Duration)](chapter/10-Standard-library/10-1-Providing-a-wrong-time-duration.md)
  - [10.2 time.After 引发的内存泄漏](chapter/10-Standard-library/10-2-time-After-and-memory-leak.md)
  - [10.3 JSON 编码和解码引入的坑](chapter/10-Standard-library/10-3-Unexpected-behavior-because-of-type-embedding.md)
  - [10.4 SQL 类库的常见错误](chapter/10-Standard-library/10-4-SQL-common-mistakes.md)
  - [10.5 瞬态资源一定要关闭](chapter/10-Standard-library/10-5-Not-closing-transient-resources.md)
  - [10.6 回复 HTTP 请求后忘记调用 return 语句](chapter/10-Standard-library/10-6-Forgetting-the-return-statement-after-replying-to-an-HTTP-request.md)
  - [10.7 不要使用默认的 HTTP 客户端和服务器](chapter/10-Standard-library/10-7-Using-the-default-HTTP-client-and-server.md)
- [11 测试](chapter/11-Testing/11-0-Testing.md)
  - [11.1 测试也需要分类](chapter/11-Testing/11-1-Not-categorizing-tests.md)
  - [11.2 启用 race 标志检查是否存在数据竞争](chapter/11-Testing/11-2-Not-enabling-the-race-flag.md)
  - [11.3 测试的 parallel 与 shuffle 模式](chapter/11-Testing/11-3-Not-using-test-execution-modes.md)
  - [11.4 表驱动测试，简化测试代码的神器](chapter/11-Testing/11-4-Not-using-table-driven-tests.md)
  - [11.5 在单元测试中使用 sleep 引入的坑](chapter/11-Testing/11-5-Sleeping-in-unit-tests.md)
  - [11.6 测试不稳定，都是 time API 在捣乱](chapter/11-Testing/11-6-Not-dealing-with-the-time-API-efficiently.md)
  - [11.7 原生测试包，好处多](chapter/11-Testing/11-7-Not-using-testing-utility-packages.md)
  - [11.8 测试代码中编写不准确的基准](chapter/11-Testing/11-8-Writing-inaccurate-benchmarks.md)
  - [11.9 没有探索所有的 Go 测试特性](chapter/11-Testing/11-9-Not-exploring-all-the-Go-testing-features.md)
- [12 优化](chapter/12-Optimizations/12-0-Optimizations.md)
  - [12.1 基于 CPU 缓存的优化](chapter/12-Optimizations/12-1-Not-understanding-CPU-caches.md)
  - [12.2 并发代码导致 CPU 缓存的虚假共享](chapter/12-Optimizations/12-2-Writing-concurrent-code-leading-to-false-sharing.md)
  - [12.3 CPU 指令级并行性对程序的优化](chapter/12-Optimizations/12-3-Not-taking-into-account-instruction-level-parallelism.md)
  - [12.4 数据在内存中对齐，有什么好处](chapter/12-Optimizations/12-4-Not-being-aware-of-data-alignment.md)
  - [12.5 逃逸分析到底是啥](chapter/12-Optimizations/12-5-Not-understanding-stack-vs-heap.md)
  - [12.6 减少内存分配对程序的优化](chapter/12-Optimizations/12-6-Not-knowing-how-to-reduce-allocations.md)
  - [12.7 Go 语言的内联优化](chapter/12-Optimizations/12-7-Not-relying-on-inlining.md)
  - [12.8 pprof 诊断工具](chapter/12-Optimizations/12-8-Not-using-Go-diagnostics-tooling.md)
  - [12.9 Go 的垃圾回收是如何工作的](chapter/12-Optimizations/12-9-Not-understanding-how-the-GC-works.md)
  - [12.10 Docker 和 Kubernetes 内部机制对 Go 程序的影响](chapter/12-Optimizations/12-10-Not-understanding-the-impacts-of-running-Go-inside-of-Docker-and-Kubernetes.md)

## 协议

[CC-BY 4.0](https://github.com/lts8989/lts8989.github.io/blob/main/LICENSE)

## 翻译心得

翻译要做到信、达、雅。

举个例子

![](https://img.exciting.net.cn/127.png)

Forever21 是一个美国服装品牌，怎么翻译成中文？

* 信：准确的翻译，永远21。
* 达：阅读通顺，永远21岁。
* 雅：接地气，永远18岁。