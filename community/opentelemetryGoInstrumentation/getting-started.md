欢迎来到 Go 的开放遥测入门指南！ 本指南将引导您完成安装、监测、配置和从 OpenTelemetry 导出数据的基本步骤。 在开始之前，请确保已安装 Go 1.16 或更新版本。

了解系统在出现故障时如何运行的，对于解决这些故障至关重要。 链路追踪可以做到这一点。 本指南展示了如何使用 OpenTelemetry Go 项目来跟踪示例应用程序。 您将从一个为用户计算斐波那契数的应用程序开始，然后您将添监测代码 以使用 OpenTelemetry Go 生成跟踪遥测。

作为参考，可以在 [此处](https://github.com/open-telemetry/opentelemetry-go/tree/main/example/fib) 找到您将构建的代码的完整示例。

要开始构建应用程序，请创建一个名为`fib`的新目录来存放我们的斐波那契项目。 接下来，将以下内容添加到该目录中名为`fib.go`的新文件中。

```go
package main

// Fibonacci returns the n-th fibonacci number.
func Fibonacci(n uint) (uint64, error) {
	if n <= 1 {
		return uint64(n), nil
	}

	var n2, n1 uint64 = 0, 1
	for i := uint(2); i < n; i++ {
		n2, n1 = n1, n1+n2
	}

	return n2 + n1, nil
}
```

添加核心逻辑后，您现在可以围绕它构建应用程序。 添加具有以下应用程序逻辑的新 `app.go` 文件。

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
)

// App is a Fibonacci computation application.
type App struct {
	r io.Reader
	l *log.Logger
}

// NewApp returns a new App.
func NewApp(r io.Reader, l *log.Logger) *App {
	return &App{r: r, l: l}
}

// Run starts polling users for Fibonacci number requests and writes results.
func (a *App) Run(ctx context.Context) error {
	for {
		n, err := a.Poll(ctx)
		if err != nil {
			return err
		}

		a.Write(ctx, n)
	}
}

// Poll asks a user for input and returns the request.
func (a *App) Poll(ctx context.Context) (uint, error) {
	a.l.Print("What Fibonacci number would you like to know: ")

	var n uint
	_, err := fmt.Fscanf(a.r, "%d\n", &n)
	return n, err
}

// Write writes the n-th Fibonacci number back to the user.
func (a *App) Write(ctx context.Context, n uint) {
	f, err := Fibonacci(n)
	if err != nil {
		a.l.Printf("Fibonacci(%d): %v\n", n, err)
	} else {
		a.l.Printf("Fibonacci(%d) = %d\n", n, f)
	}
}
```

在您的应用程序完全组合后，您需要一个 `main()` 函数来实际运行应用程序。 在新的 `main.go` 文件中添加以下运行逻辑。

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
)

func main() {
	l := log.New(os.Stdout, "", 0)

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)

	errCh := make(chan error)
	app := NewApp(os.Stdin, l)
	go func() {
		errCh <- app.Run(context.Background())
	}()

	select {
	case <-sigCh:
		l.Println("\ngoodbye")
		return
	case err := <-errCh:
		if err != nil {
			l.Fatal(err)
		}
	}
}
```

代码完成后，马上可以运行应用程序了。 在执行此操作之前，您需要将此目录初始化为 Go 模块。 在您的终端上，在 `fib` 目录中运行命令 `go mod init fib`。 这将创建一个`go.mod`文件，Go 使用该文件来管理导入。 现在您应该可以运行应用程序了！


```sh
$ go run .
What Fibonacci number would you like to know:
42
Fibonacci(42) = 267914296
What Fibonacci number would you like to know:
^C
goodbye
```

可以使用 CTRL+C 退出应用程序。 您应该会看到与上述类似的输出，如果不是的话，请检查并修复错误。

## Trace Instrumentation

OpenTelemetry 分为两部分：用于监测代码的 API 和实现 API 的 SDK。 在将 OpenTelemetry 集成到任何项目中时，API 用于定义遥测的生成方式。 要在您的应用程序中生成跟踪遥测，您可以使用 `go.opentelemetry.io/otel/trace` 包中的 OpenTelemetry Trace API。

首先，您需要为 Trace API 安装必要的包。 在您的工作目录中运行以下命令。

```sh
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/trace
```

现在已经安装了包，你现在可以将要使用的包导入到到 `app.go` 文件中。

```go
import (
	"context"
	"fmt"
	"io"
	"log"
	"strconv"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)
```

添加导入后，您可以开始监测。

OpenTelemetry Tracing API 提供了一个 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)  来创建跟踪。 这些 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)  旨在与一个`监测库`相关联。 这样产生的遥测数据，可以理解为是由代码库生产的。 为了让 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)  唯一标识您的应用程序，您需要在 `app.go` 中创建一个带有包名的常量。


```go
// name is the Tracer name used to identify this instrumentation library.
const name = "fib"
```

使用完全限定的包名（对于 Go 包来说是唯一的）是识别 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)  的标准做法。 如果您的示例包名称不是 `fib`，请将它更新到这里。

现在一切都应该准备就绪，可以开始跟踪您的应用程序了。 但什么是链路呢？ 又应该如何在应用程序中构建它们呢？

我们回头看一下，跟踪是一种遥测，遥测服务正在完成的工作。 跟踪是一种记录，记录处理请求参与者之间的连接信息，通常指的是客户端和服务器之间的请求或其他形式的通信。

服务执行工作的每个部分在跟踪中由跨度表示。 这些跨度不仅仅是一个无序的集合。 就像我们应用程序的调用堆栈一样，这些跨度相互之间是有关系的。 根跨度是唯一没有跨度的跨度，它表示服务请求从哪开始的。 在同一个跟踪中其他所有跨度之间都是有父子关系的。

如果对最后一部分关于跨度关系的解释还没有完全理解，请不要担心。 最重要的一点是代码的每个部分，它做的一些工作，都应该表示为一个跨度。 集成你的代码后，您将对这些跨度关系有更好的理解，所以让我们开始吧。

首先运行 `Run` 方法启动监测。

```go
// Run starts polling users for Fibonacci number requests and writes results.
func (a *App) Run(ctx context.Context) error {
	for {
		// Each execution of the run loop, we should get a new "root" span and context.
		newCtx, span := otel.Tracer(name).Start(ctx, "Run")

		n, err := a.Poll(newCtx)
		if err != nil {
			span.End()
			return err
		}

		a.Write(newCtx, n)
		span.End()
	}
}
```

上面的代码为 for 循环的每次循环创建了一个跨度。跨度是使用 [global `TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel#GetTracerProvider) 中的 [`Tracer`] 创建的。 当您在后面的部分安装 SDK 时，你需要设置一个全局 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) ，这将让你从另一个角度了解有关 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 的更多信息。 现在，作为监测作者，您只需要关注的是，当您编写 `otel.Tracer(name)` 时，您需要给来自 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider)的 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer) 设置适当的名字。

接下来，检测 `Poll` 方法。
```go
// Poll asks a user for input and returns the request.
func (a *App) Poll(ctx context.Context) (uint, error) {
	_, span := otel.Tracer(name).Start(ctx, "Poll")
	defer span.End()

	a.l.Print("What Fibonacci number would you like to know: ")

	var n uint
	_, err := fmt.Fscanf(a.r, "%d\n", &n)

	// Store n as a string to not overflow an int64.
	nStr := strconv.FormatUint(uint64(n), 10)
	span.SetAttributes(attribute.String("request.n", nStr))

	return n, err
}
```

类似于 `Run` 方法检测，这为方法添加了一个跨度以跟踪计算的执行。 另外，它还添加了一个属性来注解跨度。 当您认为应用程序的用户在查看遥测数据时希望查看有关运行环境的状态或详细信息时，您可以添加此注解。

最后，检测 `Write` 方法。

```go
// Write writes the n-th Fibonacci number back to the user.
func (a *App) Write(ctx context.Context, n uint) {
	var span trace.Span
	ctx, span = otel.Tracer(name).Start(ctx, "Write")
	defer span.End()

	f, err := func(ctx context.Context) (uint64, error) {
		_, span := otel.Tracer(name).Start(ctx, "Fibonacci")
		defer span.End()
		return Fibonacci(n)
	}(ctx)
	if err != nil {
		a.l.Printf("Fibonacci(%d): %v\n", n, err)
	} else {
		a.l.Printf("Fibonacci(%d) = %d\n", n, f)
	}
}
```

此方法使用两个跨度进行检测。 一个跟踪 `Write` 方法本身，另一个跟踪使用 `Fibonacci` 函数对核心逻辑的调用。 你看到上下文是如何通过跨度传递的吗？ 您是否看到这也定义了跨度之间的关系？

在 OpenTelemetry 中，跨度关系是使用 `context.Context` 显式定义的。 创建跨度时，会和跨度一起返回一个 `Context`。 该 `Context` 将包含对创建的跨度的引用。 如果在创建另一个跨度时使用该上下文，则两个跨度将相关。 原始跨度将成为新跨度的父跨度，反之，新跨度被称为原始跨度的子级。 这种层次结构提供了链路跟踪结构，该结构有助于显示系统计算路径。 根据您在上面检测相关的代码以及对跨度关系的理解，您的代码将展示如下所示的链路追踪信息：

```
Run
├── Poll
└── Write
    └── Fibonacci
```

`Run`跨度将是 `Poll` 和 `Write` 跨度的父级，而 `Write` 跨度将是 `Fibonacci` 的父级。

现在你如何真正看到产生的跨度？ 为此，您需要配置和安装 SDK。

## SDK 设置

`开放遥测` 在其 API 的实现中被设计成模块化的。 `OpenTelemetry Go` 项目提供了一个 SDK 包 [`go.opentelemetry.io/otel/sdk`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk)，它实现了这个 API 并遵守 `开放遥测规范`。 要开始使用此 SDK，您首先需要创建一个导出器，在这之前，我们需要安装一些包。 在 `fib` 目录中运行以下命令以安装`标准输入输出导出器`和 SDK。

```sh
$ go get go.opentelemetry.io/otel/sdk \
         go.opentelemetry.io/otel/exporters/stdout/stdouttrace
```

现在将其导入到 `main.go`

```go
import (
	"context"
	"io"
	"log"
	"os"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.10.0"
)
```

### 创建一个控制台导出器

SDK 将来自 OpenTelemetry API 的遥测数据连接到导出器。 导出器是允许将遥测数据发送到某处的包 - 发送到控制台（这是我们在这里所做的），或者发送到远程系统或收集器以进行进一步分析或丰富。
开放遥测通过其生态系统支持各种供应商，包括流行的开源工具，如 [Jaeger](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/jaeger)、[Zipkin](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/zipkin) 和 [Prometheus](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/prometheus)。

要初始化控制台导出器，请将以下函数添加到 `main.go` 文件中：

```go
// newExporter returns a console exporter.
func newExporter(w io.Writer) (trace.SpanExporter, error) {
	return stdouttrace.New(
		stdouttrace.WithWriter(w),
		// Use human-readable output.
		stdouttrace.WithPrettyPrint(),
		// Do not print timestamps for the demo.
		stdouttrace.WithoutTimestamps(),
	)
}
```

这将创建一个具有基本选项的控制台导出器。 稍后您将使用此函数当你配置 SDK 向其发送遥测数据时。

### 创建一个资源

遥测数据对于解决服务问题至关重要。 问题是，您需要一种方法来识别数据来自哪个服务，甚至是哪个服务实例。 OpenTelemetry 使用 [`Resource`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/resource#Resource) 来表示产生遥测的实体。 将以下函数添加到 `main.go` 文件以为应用程序创建适当的 [`Resource`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/resource#Resource)。

```go
// newResource returns a resource describing this application.
func newResource() *resource.Resource {
	r, _ := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("fib"),
			semconv.ServiceVersionKey.String("v0.1.0"),
			attribute.String("environment", "demo"),
		),
	)
	return r
}
```

你添加到 [`Resource`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/resource#Resource) 中的任何信息，都会有SDK 产生的所有遥测数据相关联。这是通过向 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 注册 [`Resource`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/resource#Resource) 来完成的。 下面我们来创建 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider)。

### 创建并设置 `Tracer Provider`

您的应用程序被监测以便生成遥测数据，并且您有一个导出器将该数据发送到控制台，但是它们是如何连接的？ 这就是 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 的作用。 它是个中转点，仪器将从这里获取 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)， 遥测数据是从这些 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer)s 中汇集到`导出管道`。

接收数据并最终将数据传输到导出器的管道称为 [`SpanProcessor`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#SpanProcessor)。 可以将 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 配置为具有多个[`SpanProcessor`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#SpanProcessor)，但对于此示例，这里您只需要配置一个。 使用以下内容更新 `main.go` 中的 `main` 函数。


```go
func main() {
	l := log.New(os.Stdout, "", 0)

	// Write telemetry data to a file.
	f, err := os.Create("traces.txt")
	if err != nil {
		l.Fatal(err)
	}
	defer f.Close()

	exp, err := newExporter(f)
	if err != nil {
		l.Fatal(err)
	}

	tp := trace.NewTracerProvider(
		trace.WithBatcher(exp),
		trace.WithResource(newResource()),
	)
	defer func() {
		if err := tp.Shutdown(context.Background()); err != nil {
			l.Fatal(err)
		}
	}()
	otel.SetTracerProvider(tp)

    /* … */
}
```

main函数做了很多工作。 首先，您正在创建一个将导出到文件的`导出器`。 然后，您在新的 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 中注册导出器。 通过 [`trace.WithBatcher`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#WithBatcher) 选项时将 [`BatchSpanProcessor`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#NewBatchSpanProcessor) 注册到导出器。 批处理数据是一种很好的做法，有助于避免下游系统过载。 最后，在创建 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 后，您将调研一个延迟函数来刷新和停止它，并将其注册为全局的 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 中。

你还记得在之前的检测部分中我们使用全局 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 来获取 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer) 吗？ 这里的最后一步，将 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 注册到全局变量，是将检测的 [`Tracer`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#Tracer) 与此 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 连接起来。 这种使用全局 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 的模式很方便，但并不总是合适的。 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider)s 可以显式通过函数参数传递或从包含跨度的 `Context` 中推断出来。 对于这个简单的示例，使用全局提供程序是有意义的，但对于更复杂或分布式的代码库，这些其他传递 [`TracerProvider`](https://pkg.go.dev/go.opentelemetry.io/otel/trace#TracerProvider) 的方式可能更有意义。

## 把以上代码放到一起

您现在应该有一个正常工作的应用程序，他可以生成跟踪遥测数据了！快试试吧。

```sh
$ go run .
What Fibonacci number would you like to know:
42
Fibonacci(42) = 267914296
What Fibonacci number would you like to know:
^C
goodbye
```

你的工作目录中应该已经创建了一个名为 `traces.txt` 的新文件。 运行应用程序所创建的所有跟踪数据应该都在那这里！

## 错误处理

此时，您有一个正在工作的应用程序，它正在生成跟踪遥测数据。 不幸的是，`fib` 模块中的核心功能存在错误。

```sh
$ go run .
What Fibonacci number would you like to know:
100
Fibonacci(100) = 3736710778780434371
# …
```

第 100 个斐波那契数是`354224848179261915075`，而不是`3736710778780434371`！ 此应用程序仅用作演示，但不应返回错误值。 更新 `Fibonacci` 函数以返回错误，而不是计算不正确的值。

```go
// Fibonacci returns the n-th fibonacci number. An error is returned if the
// fibonacci number cannot be represented as a uint64.
func Fibonacci(n uint) (uint64, error) {
	if n <= 1 {
		return uint64(n), nil
	}

	if n > 93 {
		return 0, fmt.Errorf("unsupported fibonacci number %d: too large", n)
	}

	var n2, n1 uint64 = 0, 1
	for i := uint(2); i < n; i++ {
		n2, n1 = n1, n1+n2
	}

	return n2 + n1, nil
}
```

太好了，您已经修复了代码，但最好在遥测数据中包含返回给用户的错误。 幸运的是，可以配置跨度来传达此信息。 使用以下代码更新 `app.go` 中的 `Write` 方法。

```go
// Write writes the n-th Fibonacci number back to the user.
func (a *App) Write(ctx context.Context, n uint) {
	var span trace.Span
	ctx, span = otel.Tracer(name).Start(ctx, "Write")
	defer span.End()

	f, err := func(ctx context.Context) (uint64, error) {
		_, span := otel.Tracer(name).Start(ctx, "Fibonacci")
		defer span.End()
		f, err := Fibonacci(n)
		if err != nil {
			span.RecordError(err)
			span.SetStatus(codes.Error, err.Error())
		}
		return f, err
	}(ctx)
    /* … */
}
```

通过此更改，从 `Fibonacci` 函数返回的任何错误都会在跨度中标记，并记录描述错误的事件。

这是一个很好的开始，但它并不是应用程序返回的唯一错误。 如果用户请求一个无符号整数值，应用程序将失败。 使用类似的修复更新 `Poll` 方法，以便在遥测数据中捕获此错误。

```go
// Poll asks a user for input and returns the request.
func (a *App) Poll(ctx context.Context) (uint, error) {
	_, span := otel.Tracer(name).Start(ctx, "Poll")
	defer span.End()

	a.l.Print("What Fibonacci number would you like to know: ")

	var n uint
	_, err := fmt.Fscanf(a.r, "%d\n", &n)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return 0, err
	}

	// Store n as a string to not overflow an int64.
	nStr := strconv.FormatUint(uint64(n), 10)
	span.SetAttributes(attribute.String("request.n", nStr))

	return n, nil
}
```

剩下的就是更新 `app.go` 文件的导入以包含 [`go.opentelemetry.io/otel/codes`](https://pkg.go.dev/go.opentelemetry.io/otel/codes) 的包。


```go
import (
	"context"
	"fmt"
	"io"
	"log"
	"strconv"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)
```

有了这些修复和集成代码的更新，重新触发错误。

```sh
$ go run .
What Fibonacci number would you like to know:
100
Fibonacci(100): unsupported fibonacci number 100: too large
What Fibonacci number would you like to know:
^C
goodbye
```

非常好！ 应用程序不再返回错误值，并且查看`traces.txt`文件中的遥测数据，您将会看到错误被捕获为一个事件。
```
"Events": [
	{
		"Name": "exception",
		"Attributes": [
			{
				"Key": "exception.type",
				"Value": {
					"Type": "STRING",
					"Value": "*errors.errorString"
				}
			},
			{
				"Key": "exception.message",
				"Value": {
					"Type": "STRING",
					"Value": "unsupported fibonacci number 100: too large"
				}
			}
		],
        ...
    }
]
```

## 下一步

本指南引导您完成向应用程序添加跟踪检测并使用控制台导出器将遥测数据发送到文件。 OpenTelemetry 中还有许多其他主题需要介绍，但此时您应该准备好开始将 OpenTelemetry Go 添加到您的项目中。 去监控你的代码！

有关检测代码以及可以使用 span 执行的操作的更多信息，请参阅 [Instrumenting](https://opentelemetry.io/docs/instrumentation/go/manual/) 文档。 同样，有关处理和导出遥测数据的高级主题可以在 [处理和导出数据](https://opentelemetry.io/docs/instrumentation/go/exporting_data/) 文档中找到。
