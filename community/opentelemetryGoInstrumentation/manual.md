# 手动监测

监测是向应用程序添加可观察性代码的过程。 有两种一般类型的监测 - 自动和手动 - 您应该熟悉这两种类型，以便有效地监测您的应用程序。

## 获取跟踪器

要创建跨度，您需要先获取或初始化跟踪器。
### 初始化一个新的跟踪器

确保您安装了以下软件包：

```
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/trace \
  go.opentelemetry.io/otel/sdk \
```

接下来初始化导出器、资源、跟踪器提供商，最后是跟踪器。

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

现在你能访问到 `tracer` 来手动监测你的代码了。

## 创建跨度

跨度是由跟踪器创建的，如果您没有初始化，则需要这样做。
要使用跟踪器创建跨度，您还需要一个 `context.Context` 实例的句柄。 这些通常来自请求对象，还有可能是来自由[监测库](https://opentelemetry.io/docs/instrumentation/go/libraries/)生成的父跨度。
```go
func httpHandler(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "hello-span")
	defer span.End()

	// do some work to track with hello-span
}
```

在 Go 中，`context` 包用于存储活跃的跨度。 当你创建一个跨度时，你不仅会得到一个跨度的句柄，还有包含它的 `context`。

一旦跨度完成后，它是不可变的，不能再被修改。

### 获取当前跨度

要获取当前跨度，你需要将其从 `context.Context` 中取出：

```go
// This context needs contain the active span you plan to extract.
ctx := context.TODO()
span := trace.SpanFromContext(ctx)

// Do something with the current span, optionally calling `span.End()` if you want it to end
```

这有助于你在某个时间点想向当前跨度添加信息。

### 创建嵌套的跨度

您可以创建嵌套跨度来跟踪工作中的嵌套操作。

如果当前的 `context.Context` 内部已经包含一个跨度， 你可以创建一个嵌套的跨度。 如下所示：

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

一旦跨度完成后，它是不可变的，不能再被修改。


### Span 属性

属性是作为元数据应用于跨度的键和值，用户聚合、过滤、分组追踪信息。 属性可以在跨度创建时添加，或者在它生命周期结束之前的其他时间点。

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

#### 语义化属性

语义化属性是在 [OpenTelemetry Specification](https://opentelemetry.io/docs/reference/specification/) 中定义好的，为了提供一组共享的属性集合在多语言、框架和不同的运行时中，如 HTTP 方法、状态码、用户代理等等的。 这些属性可以在 `go.opentelemetry.io/otel/semconv/v1.10.0` 中找到。
更多细节，见 [Trace semantic conventions](https://opentelemetry.io/docs/reference/specification/trace/semantic_conventions/)。

### 事件

事件是跨度上的可读消息，表示其生命周期内“正在发生的事情”。 例如，想象一个函数需要独占访问互斥锁下的资源。这里可以创建两个事件，一个是我们尝试访问资源时，另一个是我们获取互斥锁时。

```go
span.AddEvent("Acquiring lock")
mutex.Lock()
span.AddEvent("Got lock, doing work...")
// do stuff
span.AddEvent("Unlocking")
mutex.Unlock()
```

事件的一个有用特性是它们的时间戳显示为从跨度开始的偏移量，使你可以轻松查看它们之间经过了多少时间。

事件也可以拥有他们自己的属性。

```go
span.AddEvent("Cancelled wait due to external signal", trace.WithAttributes(attribute.Int("pid", 4328), attribute.String("signal", "SIGHUP")))
```

### 设置跨度状态

`Span` 是可以设置状态的，通常用来说明在跨度的执行期间发生了错误。

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

默认情况下，跨度的状态是`Unset`。 在其他情况下，你可以将跨度的状态设置为 `Ok`。虽然这通常是没必要的。

### 记录错误

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

当你用`RecordError`时，把设置跨度的状态也设置为 `Error`是非常推荐的。 除非你不想在追踪到一个错误时的跨度将其标记为一个错误的跨度。
`RecordError` 函数不会自动设置跨度状态。

## 创建指标

指标API 当前是不稳定的，文档待补充。

## 传播器和上下文

追踪可以被扩张到单一进程之外，这里需要上下文传播，它是一种将标记的追踪传递给远端进程的机器。

为了通过网路传播追踪上下文，传播器必须在遥测API中注册。

```go
import (
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/propagation"
)
...
otel.SetTextMapPropagator(propagation.TraceContext{})
```

> 开放遥测也支持 B3 头部格式，为了兼容已经存在的追踪系统(`go.opentelemetry.io/contrib/propagators/b3`)，它不在支持W3C追踪上下文标准。

在配置了上下文传播之后，您很可能希望使用自动监测来处理实际管理序列化上下文的幕后工作。
