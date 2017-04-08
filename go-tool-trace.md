你想知道你的Go程序在做什么吗？ `go tool trace`可以向你揭示：Go程序运行中的所有的运行时事件。 这种工具是Go生态系统中用于诊断性能问题时（如延迟，并行化和竞争异常）最有用的工具之一。 在我之前的[博客文章](https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/)中，我提到我们在Pusher中使用`go tool trace`来跟踪为何Go垃圾收集器有很长的停顿时间。 在这篇博文中，我更加深入的介绍`go toll trace`。

## `go tool trace` 试用
`go tool trace`可以显示大量的信息，所以从哪里开始是个问题。 我们首先简要介绍使用界面，然后我们将介绍如何查找具体问题。

![image](https://making.pusher.com/images/2017-03-22-go-tool-trace/tour.svg)

`go tool trace`UI是一个Web应用程序。 下面我已经嵌入了一个这个web程序的实例！ [此示例](https://gist.github.com/WillSewell/3246161e67f897a530a8120db8917bee)是可视化并行快速排序实现的追踪信息：

<iframe src="https://making.pusher.com/js/2017-03-22-go-tool-trace/trace.html" align="center" width="100%" height="578px" scrolling="no" style="border: solid lightgrey; box-shadow: 0 0 20px lightgrey; margin: 2em 0;"></iframe>

请尝试这个例子！有关导航UI的帮助，请单击右上角的“？”。单击屏幕上的任何事件可以在下面获取更多信息。这里有一些你可以从这个追踪中找到的有价值的信息：

- 这个程序运行多长时间？ 
- 有多少goroutines运行872微秒？ 
- 该进程何时第一次升级到使用三个OS线程？ 
- 什么时候主要调用qSortPar？ 
- 是什么导致额外的过程（1,2和3）开始工作？ 
- proc＃2什么时候停止？

## 太棒了! 我应该怎么在我的程序中使用`go tool trace`?

您必须调整程序以将运行时事件写入二进制文件。 这涉及从标准库导入[runtime/trace](https://golang.org/pkg/runtime/trace/)，并添加几行样板代码。 这个快速的视频将引导您：

[视频](https://youtu.be/Xq5HDH8y0CE)

以下是需要复制粘贴的代码：

```
package main

import (
	"os"
	"runtime/trace"
)

func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	err = trace.Start(f)
	if err != nil {
		panic(err)
	}
	defer trace.Stop()

  // Your program here
}

```

这将使您的程序以[二进制格式](https://docs.google.com/document/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/edit#heading=h.h5l8z9xnjq7)在文件trace.out中写入事件数据。 然后可以运行`go tool trace trace.out`。 这将解析跟踪文件，并使用可视化程序打开浏览器。 该命令还将启动服务器，并使用跟踪数据来响应可视化操作。 在浏览器中加载初始页面后，单击“View trace”。 这将加载跟踪查看器，如上面嵌入的那样。

## 使用go tool trace能解决什么问题?

我们来看一个如何使用这个工具跟踪典型问题的例子。

### 诊断延迟问题

当完成关键任务的goroutine被阻止运行时，可能会引起延迟问题。 可能的原因有很多：做系统调用时被阻塞; 被共享内存阻塞（通道/互斥等）; 被runtime系统（例如GC）阻塞，甚至可能调度程序不像您想要的那样频繁地运行关键goroutine。

所有这些都可以使用`go tool trace`来识别。 您可以通过查看PROCs时间线来跟踪问题，并发现一段时间内goroutine被长时间阻塞。 一旦你确定了这段时间，应该给出一个关于根本原因的线索。

作为延迟问题的一个例子，让我们看看上一篇博文中[长时间的GC暂停](https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/)：

![image](https://making.pusher.com/images/2017-03-22-go-tool-trace/gc-latency.png)

红色的事件代表了唯一的程序goroutine正在运行。 在所有四个线程上并行运行的goroutines是垃圾收集器的MARK阶段。 这个MARK阶段阻止了主要的goroutine。 你能出到阻止runtime.main goroutine的时间长短吗？

[在Go团队宣布GC暂停时间少于100微秒后](https://groups.google.com/forum/#!msg/golang-dev/Ab1sFeoZg_8/_DaL0E8fAwAJ),我很快就调查了这个延迟问题。 我看到的漫长的停顿时间，`go tool trace`的结果看起来很奇怪，特别是可以看到它们(暂停)是在收集器的并发阶段发生的。 [我在go-nuts 邮件列表中提到了这个问题](https://groups.google.com/forum/#!msg/golang-nuts/nOD0fGmRp_g/4FEThB1bBQAJ)，似乎与[这个问题](https://github.com/golang/go/issues/16528)有关，现在已经在Go 1.8中修复了。 我的基准测试又出现了[另一个GC暂停问题](https://github.com/golang/go/issues/18155)，这在写本文时依然会出现。 如果没有`go tool trace`这一工具，我是无法完成调查工作的。

### 诊断并行问题

假设您已经编写了一个程序，您希望使用所有的CPU，但运行速度比预期的要慢。 这可能是因为您的程序不像您所期望的那样并行运行。 这可能是由于在很多关键路径上串行运行太多，而很多代码原本是可以异步（并行）运行的。

假设我们有一个pub/sub消息总线，我们希望在单个goroutine中运行，以便它可以安全地修改没有加互斥锁的用户map。 请求处理程序将消息写入消息总线队列。 总线从队列中读取消息，在map中查找订阅者，并将消息写入其套接字。 让我们看看单个消息的`go tool trace`中的内容：

![image](https://making.pusher.com/images/2017-03-22-go-tool-trace/parallelism-bad.png)

最初的绿色事件是http处理程序读取发布的消息并将其写入消息总线事件队列。 之后，消息总线以单个线程运行 - 第二个绿色事件 - 将消息写给订阅者。

红线显示消息写入订户的套接字的位置。 写入所有订阅者的过程需要多长时间？

 问题是四分之一的线程正在闲置。 有没有办法利用它们？ 答案是肯定的 我们不需要同步写入每个用户; 写入可以在单独的goroutine中同时运行。 让我们看看如果我们作出这个变化，会发生什么：

![image](https://making.pusher.com/images/2017-03-22-go-tool-trace/parallelism-good.png)

正如你所看到的，写给订阅者消息的过程正在许多goroutines的上同步进行。

但它是否更快？

有趣的是，鉴于我们使用4X的CPU，加速是适合的。 这是因为并行运行代码有更多的开销：启动和停止goroutines; 共享内存以及单独的缓存。 加速的理论上限使得我们无法实现4倍延迟降低：[阿姆达尔定律](https://en.wikipedia.org/wiki/Amdahl's_law)。

实际上，并行运行代码往往效率较低; 特别是在goroutine是非常短暂的，或者他们之间有很多的竞争的情况下。 这是使用此工具的另一个原因：尝试这两种方法，并检查哪种工作最适合您的用例。

## 什么时候`go tool trace`不合适？

当然，`go tool trace`不能解决一切问题。 如果您想跟踪运行缓慢的函数，或者找到大部分CPU时间花费在哪里，这个工具就是不合适的。 为此，您应该使用`go tool pprof`，它可以显示在每个函数中花费的CPU时间的百分比。 `go tool trace`更适合于找出程序在一段时间内正在做什么，而不是总体上的开销。 此外，还有“view trace”链接提供的其他可视化功能，这些对于诊断争用问题特别有用。 了解您的程序在理论上的表现（使用老式Big-O分析）也是无可替代的。

## 结论
希望这篇文章可以让您了解如何使用`go tool trace`诊断问题。 即使您没有解决具体问题，可视化您的程序是检查程序运行时特性的好方法。 我在这篇文章中使用的例子很简单，但更复杂的程序中的症状应该与此惊人的相似。

## 附录
这个博客文章给了你一个使用`go tool trace`的介绍，但你可能希望更深入地深入了解该工具。 目前正在进行的[官方`go tool trace`文档](https://golang.org/cmd/trace/)相当稀少。 有一个[Google文档](https://docs.google.com/document/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/edit?usp=sharing)更详细。 除此之外，我发现参考源代码是很有用，可以找出`go tool trace`如何工作：

- [go tool trace 源代码](https://github.com/golang/go/tree/master/src/cmd/trace)
- [二进制跟踪解析器的源代码](https://github.com/golang/go/blob/master/src/internal/trace/parser.go)
- [trace 源代码](https://github.com/golang/go/blob/master/src/runtime/trace.go)
- `go tool trace`的Web界面来自[Catapult项目的跟踪查看器](https://github.com/catapult-project/catapult/blob/master/tracing/README.md)。 该查看器可以从许多跟踪格式生成可视化。 go工具跟踪使用[基于JSON](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview)的跟踪事件格式。
