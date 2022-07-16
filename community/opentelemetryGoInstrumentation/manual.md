---
title: Manual Instrumentation
weight: 3
linkTitle: Manual
aliases: [/docs/instrumentation/go/instrumentation, /docs/instrumentation/go/manual_instrumentation]
---

Instrumentation is the process of adding observability code to your application. There are two general types of instrumentation - automatic, and manual - and you should be familiar with both in order to effectively instrument your software.
`监测` 是向应用程序添加可观察性代码的过程。 有两种一般类型的监测 - 自动和手动 - 您应该熟悉这两种类型，以便有效地监测您的应用程序。

## Getting a Tracer
## 获取跟踪器

To create spans, you'll need to acquire or initialize a tracer first.
要创建 `Span`，您需要先获取或初始化跟踪器。

### Initializing a new tracer
### 初始化一个新的跟踪器

Ensure you have the right packages installed:
确保您安装了以下软件包：

```
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/trace \
  go.opentelemetry.io/otel/sdk \
```

Then initialize an exporter, resources, tracer provider, and finally a tracer.
接下来初始化导出器、资源、跟踪器提供商，最后是跟踪器

```go
package app

import (
	"context"
	"fmt"
	"log"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.10.0"
	"go.opentelemetry.io/otel/trace"
)

var tracer trace.Tracer

func newExporter(ctx context.Context)  /* (someExporter.Exporter, error) */ {
	// Your preferred exporter: console, jaeger, zipkin, OTLP, etc.
}

func newTraceProvider(exp sdktrace.SpanExporter) *sdktrace.TracerProvider {
	// Ensure default SDK resources and the required service name are set.
	r, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("ExampleService"),
		)
	)
	
	if err != nil {
		panic(err)
	}

	return sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(r),
	)
}

func main() {
	ctx := context.Background()

	exp, err := newExporter(ctx)
	if err != nil {
		log.Fatalf("failed to initialize exporter: %v", err)
	}

	// Create a new tracer provider with a batch span processor and the given exporter.
	tp := newTraceProvider(exp)

	// Handle shutdown properly so nothing leaks.
	defer func() { _ = tp.Shutdown(ctx) }()

	otel.SetTracerProvider(tp)

	// Finally, set the tracer that can be used for this package.
	tracer = tp.Tracer("ExampleService")
}
```

You can now access `tracer` to manually instrument your code.
现在你能访问到 `tracer 跟踪器` 来手动监测你的代码了。

## Creating Spans
## 创建 Span

Spans are created by tracers. If you don't have one initialized, you'll need to do that.
`Span` 是由跟踪器创建的，如果您没有初始化，则需要这样做。
To create a span with a tracer, you'll also need a handle on a `context.Context` instance. These will typically come from things like a request object and may already contain a parent span from an [instrumentation library][].
要使用跟踪器创建 `Span`，您还需要一个 `context.Context` 实例的句柄。 这些通常来自请求对象，还有可能是来自由[监测库]生成的父 `Span`。
```go
func httpHandler(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "hello-span")
	defer span.End()

	// do some work to track with hello-span
}
```

In Go, the `context` package is used to store the active span. When you start a span, you'll get a handle on not only the span that's created, but the modified context that contains it.
在 Go 中，`context` 包用于存储活跃的 `Span`。 当你创建一个 `Span` 时，你不仅会得到一个 `Span` 的句柄，还有包含它的 `context`。

Once a span has completed, it is immutable and can no longer be modified.
一旦`Span` 完成后，它是不可变的，不能再被修改。

### Get the current span

To get the current span, you'll need to pull it out of a `context.Context` you have a handle on:
要获取当前跨度，你需要将其从 `context.Context` 中取出：

```go
// This context needs contain the active span you plan to extract.
ctx := context.TODO()
span := trace.SpanFromContext(ctx)

// Do something with the current span, optionally calling `span.End()` if you want it to end
```

This can helpful if you'd like to add information to the current span at a point in time.
这有助于你在某个时间点想向当前跨度添加信息。

### Create nested spans
### 创建嵌套的 `Span`

You can create a nested span to track work in a nested operation.
您可以创建嵌套 `Span` 来跟踪工作中的嵌套操作。

If the current `context.Context` you have a handle on already contains a span inside of it, creating a new span makes it a nested span. For example:
如果当前的 `context.Context` 内部已经包含一个 `Span`， 你可以创建一个嵌套的 `Span`。 如下所示：

```go
func parentFunction(ctx context.Context) {
	ctx, parentSpan := tracer.Start(ctx, "parent")
	defer parentSpan.End()

	// call the child function and start a nested span in there
	childFunction(ctx)

	// do more work - when this function ends, parentSpan will complete.
}

func childFunction(ctx context.Context) {
	// Create a span to track `childFunction()` - this is a nested span whose parent is `parentSpan`
	ctx, childSpan := tracer.Start(ctx, "child")
	defer childSpan.End()

	// do work here, when this function returns, childSpan will complete.
}
```

Once a span has completed, it is immutable and can no longer be modified.
一旦`Span` 完成后，它是不可变的，不能再被修改。


### Span Attributes
### Span 属性

Attributes are keys and values that are applied as metadata to your spans and are useful for aggregating, filtering, and grouping traces. Attributes can be added at span creation, or at any other time during the lifecycle of a span before it has completed.
属性是作为元数据应用于 `Span` 的键和值，用户聚合、过滤、分组追踪信息。 属性可以在 `Span` 创建时添加，或者在它生命周期结束之前的其他时间点

```go
// setting attributes at creation...
ctx, span = tracer.Start(ctx, "attributesAtCreation", trace.WithAttributes(attribute.String("hello", "world")))
// ... and after creation
span.SetAttributes(attribute.Bool("isTrue", true), attribute.String("stringAttr", "hi!"))
```

Attribute keys can be precomputed, as well:
属性可以预先设置，如下所示：

```go
var myKey = attribute.Key("myCoolAttribute")
span.SetAttributes(myKey.String("a value"))
```

#### Semantic Attributes
#### 语义化属性

Semantic Attributes are attributes that are defined by the [OpenTelemetry Specification][] in order to provide a shared set of attribute keys across multiple languages, frameworks, and runtimes for common concepts like HTTP methods, status codes, user agents, and more. These attributes are available in the `go.opentelemetry.io/otel/semconv/v1.10.0` package.
语义化属性是在 [OpenTelemetry Specification](https://opentelemetry.io/docs/reference/specification/) 中定义好的，为了提供一组共享的属性集合在多语言、框架和不同的运行时中，如 HTTP 方法、状态码、用户代理等等的。 这些属性可以在 `go.opentelemetry.io/otel/semconv/v1.10.0` 中找到。
For details, see [Trace semantic conventions][].
更多细节，见 [Trace semantic conventions](https://opentelemetry.io/docs/reference/specification/trace/semantic_conventions/)。

### Events
### 事件

An event is a human-readable message on a span that represents "something happening" during it's lifetime. For example, imagine a function that requires exclusive access to a resource that is under a mutex. An event could be created at two points - once, when we try to gain access to the resource, and another when we acquire the mutex.
事件是 `Span` 上的可读消息，表示其生命周期内“正在发生的事情”。 例如，想象一个函数需要独占访问互斥锁下的资源。这里可以创建两个事件，一个是我们尝试访问资源时，另一个是我们获取互斥锁时。

```go
span.AddEvent("Acquiring lock")
mutex.Lock()
span.AddEvent("Got lock, doing work...")
// do stuff
span.AddEvent("Unlocking")
mutex.Unlock()
```

A useful characteristic of events is that their timestamps are displayed as offsets from the beginning of the span, allowing you to easily see how much time elapsed between them.
事件的一个有用特性是它们的时间戳显示为从 `Span` 开始的偏移量，使你可以轻松查看它们之间经过了多少时间。

Events can also have attributes of their own -
事件也可以拥有他们自己的属性

```go
span.AddEvent("Cancelled wait due to external signal", trace.WithAttributes(attribute.Int("pid", 4328), attribute.String("signal", "SIGHUP")))
```

### Set span status
### 设置 `Span` 状态

A status can be set on a span, typically used to specify that there was an error in the operation a span is tracking - .`Error`.
`Span` 是可以设置状态的，通常用来说明在 `Span` 的执行期间发生了错误。

```go
import (
	// ...
	"go.opentelemetry.io/otel/codes"
	// ...
)

// ...

result, err := operationThatCouldFail()
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
}
```

By default, the status for all spans is `Unset`. In rare cases, you may also wish to set the status to `Ok`. This should generally not be necessary, though.
默认情况下，`Span` 的状态是`Unset`。 在其他情况下，你可以将 `Span` 的状态设置为 `Ok`。虽然这通常是没必要的。

### Record errors
### 记录错误

If you have an operation that failed and you wish to capture the error it produced, you can record that error.
如果在一个操作中发生了错误，并且你想捕获这个错误，你可以记录这个错误。

```go
import (
	// ...
	"go.opentelemetry.io/otel/codes"
	// ...
)

// ...

result, err := operationThatCouldFail()
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
	span.RecordError(err)
}
```

It is highly recommended that you also set a span's status to `Error` when using `RecordError`, unless you do not wish to consider the span tracking a failed operation as an error span.
当你用`RecordError`时，把设置 `Span` 的状态也设置为 `Error`是非常推荐的。 除非你不想在追踪到一个错误时的`Span`将其标记为一个错误的`Span`。
The `RecordError` function does **not** automatically set a span status when called.
`RecordError` 函数不会自动设置`Span` 状态。

## Creating Metrics
## 创建指标

The metrics API is currently unstable, documentation TBA.
指标API 当前是不稳定的，文档待补充

## Propagators and Context
## 传播器和上下文

Traces can extend beyond a single process. This requires _context propagation_, a mechanism where identifiers for a trace are sent to remote processes.
追踪可以被扩张到单一进程之外，这里需要上下文传播，它是一种将标记的追踪传递给远端进程的机器。

In order to propagate trace context over the wire, a propagator must be registered with the OpenTelemetry API.
为了通过网路传播追踪上下文，传播器必须在遥测API中注册。

```go
import (
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/propagation"
)
...
otel.SetTextMapPropagator(propagation.TraceContext{})
```

> OpenTelemetry also supports the B3 header format, for compatibility with existing tracing systems (`go.opentelemetry.io/contrib/propagators/b3`) that do not support the W3C TraceContext standard.
> 开放遥测也支持 B3 头部格式，为了兼容已经存在的追踪系统，它不在支持W3C追踪上下文标准。

After configuring context propagation, you'll most likely want to use automatic instrumentation to handle the behind-the-scenes work of actually managing serializing the context.
在配置了上下文传播之后，您很可能希望使用自动监测来处理实际管理序列化上下文的幕后工作。

[OpenTelemetry Specification]: {{< relref "/docs/reference/specification" >}}
[Trace semantic conventions]: {{< relref "/docs/reference/specification/trace/semantic_conventions" >}}
[instrumentation library]: {{< relref "libraries" >}}
