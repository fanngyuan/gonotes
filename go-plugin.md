在Go 1.8版中引入了一个新的Go插件系统。 此功能使程序员可以使用动态链接库构建松散耦合的模块化程序，可以在运行时动态加载和绑定。

Go语言开发人员，不可避免地会遇到模块化代码的需要。 我们依靠多出来的流程设计如[OS exec calls](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/)，[sockets](https://docs.docker.com/engine/extend/plugin_api/)和[RPC/GRPC](https://github.com/hashicorp/go-plugin)（等等）的实现代码模块化。虽然这些方法可以很好地工作，但在许多情况下，它们的目的只是为了解决Go以前缺乏的插件系统。

在本文，我探讨了使用Go插件系统创建模块化软件的意义。 本文提供了一个功能齐全的示例，以及有关设计和相关影响的讨论。

# Go插件

Go插件是使用-buildmode = plugin 标记编译的一个包，用于生成一个共享对象（.so）库文件。 Go包中的导出的函数和变量被公开为ELF符号，可以使用plugin包在运行时查找并绑定ELF符号。

Go编译器能够使用build flag -buildmode = c-shared创建C风格的动态共享库。

## 限制
从1.8版开始，Go插件功能只能在Linux上使用。 很有可能在将来的版本中发生变化。

## 一个简单的可插拔程序
本节中介绍的代码展示了了如何创建一个简单的程序，该程序使用插件扩展其输出多种语言问候语的功能。 每种本地语言都被实现为一个Go插件。

请参阅Github repo - https://github.com/vladimirvivien/go-plugin-example

greeter.go使用包./eng和./chi中的插件分别打印中文和英文问候语。 下图显示了程序的目录布局。

![image](https://cdn-images-1.medium.com/max/1600/1*CKNiDcig6QccVwUI9C496A.png)

首先，我们来看看source eng/greeter.go，在执行时会打印一个英文消息。
文件./eng/greeter.go
```
package main

import "fmt"

type greeting string

func (g greeting) Greet() {
    fmt.Println("Hello Universe")
}

// exported as symbol named "Greeter"
var Greeter greeting
```

上面的代码代表了一个插件包的内容。 你应该注意以下几点：
- 插件包必须为main。
- 导出的包函数和变量成为共享库符号。 在上述中，变量 Greeter将作为共享库中的符号导出。

## 编译插件

插件包使用以下命令进行编译：
```
go build -buildmode=plugin -o eng/eng.so eng/greeter.go
go build -buildmode=plugin -o chi/chi.so chi/greeter.go
```
这一步将分别创建插件共享库文件./eng/eng.so和./chi/chi.so。 我们可以使用以下命令进行验证：

```
$> file chi/chi.so
chi/chi.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=20c823671954b4716d8d945c3b333fc18cdbf7fe, not stripped
```

## 使用插件

使用Go plugin 包动态加载插件。 驱动（或客户端）程序./greeter.go使用前面编译的插件，如下所示。

文件  ./greeter.go

```
package main

import "plugin"; ...


type Greeter interface {
	Greet()
}

func main() {
	// determine plugin to load
	lang := "english"
	if len(os.Args) == 2 {
		lang = os.Args[1]
	}
	var mod string
	switch lang {
	case "english":
		mod = "./eng/eng.so"
	case "chinese":
		mod = "./chi/chi.so"
	default:
		fmt.Println("don't speak that language")
		os.Exit(1)
	}

	// load module
	// 1. open the so file to load the symbols
	plug, err := plugin.Open(mod)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	// 2. look up a symbol (an exported function or variable)
	// in this case, variable Greeter
	symGreeter, err := plug.Lookup("Greeter")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	// 3. Assert that loaded symbol is of a desired type
	// in this case interface type Greeter (defined above)
	var greeter Greeter
	greeter, ok := symGreeter.(Greeter)
	if !ok {
		fmt.Println("unexpected type from module symbol")
		os.Exit(1)
	}

	// 4. use the module
	greeter.Greet()

}
```
从以前的源代码可以看出，动态加载插件模块访问其成员有几个步骤：

- 选择要加载的插件，基于os.Args和switch。
- 用plugin.Open()打开插件文件。
- 用plguin.Lookup("Greeter")查找导出的符号"Greeter"。 请注意，符号名称与插件模块中定义的变量名称相匹配。
- 声明符号是接口类型Greeter使用symGreeter.(Greeter)。
- 最后，调用Greet()方法从显示消息。

您可以在[这里](https://github.com/vladimirvivien/go-plugin-example/blob/master/README.md)找到有关源代码的详细信息。

## 运行程序

执行程序时，会根据传递的命令行参数以英文或中文打印问候语，如下所示。

```
> go run greeter.go english
Hello Universe
> go run greeter.go chinese
你好宇宙
```

通过使用插件在运行时扩展程序的功能，以不同语言显示问候语，而无需重新编译程序。

# 模块化程序设计

使用Go插件创建模块化程序需要遵循与常规Go软件包一样严格的软件实践。然而，插件引入了新的设计问题，因为它们的解耦性质被放大了。
## 清晰的负担
构建可插拔软件系统时，建立清晰的组件可用性很重要。系统必须为插件集成提供一个简单的封装层。另一方面，插件开发人员应将系统视为黑盒，不作为所提供的合约以外的假设。
## 插件独立
应该将插件视为与其他组件分离的独立组件。这允许插件独立于他们的消费者，并拥有自己的开发和部署生命周期。
## 应用Unix模块化原则
插件代码应该设计成只关注一个功能点。
## 清楚记录
由于插件是在运行时加载的独立组件，因此它们必须有很好的文档。例如，导出的函数和变量的名称应该清楚地文档化，以避免符号查找错误。
## 使用接口类型作为边界
Go插件可以导出任何类型的包函数和变量。您可以设计插件来将其功能解耦为一组松散的函数。缺点是您必须单独查找和绑定每个函数符号。

然而，更为简单的方法是使用接口类型。创建导出功能的接口提供了统一简洁的交互，并具有清晰的功能划分。解析到接口的符号将提供对该功能的整个方法集的访问，而不仅仅是一个方法。

## 新部署范式
插件有可能影响软件在Go中的构建和分发。例如，库作者可以将其代码作为可在运行时链接的预构建组件来分发。这将偏离传统的`go get`，build和链接循环。

## 信任和安全
如果Go社区习惯使用预构建的插件（二进制文件）作为分发库的一种方法，信任和安全自然会成为一个问题。幸运的是我们有已经建立起来的社区，还有信誉良好的分发基础设施，可以在这里提供帮助。

## 版本
插件是不透明而独立的实体，应该进行版本控制，以向用户提示其支持的功能。这里的一个建议是在命名共享对象文件时使用语义版本控制。例如，上面的文件编译插件可以命名为eng.so.1.0.0。

# Gosh：一个可插拔命令shell

自插件系统发布以来，我想创建一个可插拔的框架来创建交互式命令shell程序，其中使用Go插件实现命令。 所以我创建了Gosh（Go shell）。

在[这篇博客](https://medium.com/@vladimirvivien/gosh-a-pluggable-command-shell-in-go-cf25102c8439)文章中了解Gosh

Gosh使用shell在运行时加载命令插件。 当用户在提示符下键入命令时，驱动程序将分派已注册的插件来处理该命令。 这是一个早期的尝试，但它已经展示了Go插件系统的潜力。

# 结论
我很高兴在Go中添加插件功能。我认为它为Go程序的组装和构建提供了一个重要的方式。插件可以创建新类型的Go系统，利用动态共享对象二进制文件（如Gosh）的后期绑定性质，或根据需要将插件二进制文件推送到节点的分布式系统，或者在运行时完成装配的容器化系统等等

插件系统并不完美。现在还有缺陷，可以像软件开发中的任何东西一样被滥用。我可以预测插件版本控制将是一个痛点。除了这些问题，在Go中有一个模块化的解决方案，可以提供一个更健康的平台，可以同时支持单一二进制和模块化部署。
