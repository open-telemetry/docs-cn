# Using instrumentation libraries
# 使用监测库

Go does not support truly automatic instrumentation like other languages today. Instead, you'll need to depend on [instrumentation libraries](/docs/reference/specification/glossary/#instrumentation-library) that generate telemetry data for a particular instrumented library. For example, the instrumentation library for `net/http` will automatically create spans that track inbound and outbound requests once you configure it in your code.
Go 不像当前其他语言那样支持真正的自动检测。 相反，您需要依赖为特定监测库生成遥测数据的 [监测库](https://opentelemetry.io/docs/reference/specification/glossary/#instrumentation-library)。 例如，当你将`net/http` 的监测库配置到你的代码中后，监测库将自动创建 `Span` 在接收和发出请求时。

## Setup
## 设置

Each instrumentation library is a package. In general, this means you need to `go get` the appropriate package:
每个监测库都是一个包。 一般来说，这意味着您需要 `go get` 适当的包：

```console
go get go.opentelemetry.io/contrib/instrumentation/{import-path}/otel{package-name}
```

And then configure it in your code based on what the library requires to be activated.
然后根据相应库的需求，在你的代码中配置和启动。

## Example with `net/http`
## `net/http` 示例
As an example, here's how you can set up automatic instrumentation for inbound HTTP requests for `net/http`:
例如，以下是如何为 `net/http` 在接收 `HTTP` 请求时设置自动检测：

First, get the `net/http` instrumentation library:
首先，获取 `net\http` 监测库

```console
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

Next, use the library to wrap an HTTP handler in your code:
下一步，在你的代码中用这个库包裹 HTTP 处理器

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
)

// Package-level tracer.
// This should be configured in your code setup instead of here.
var tracer = otel.Tracer("github.com/full/path/to/mypkg")

// sleepy mocks work that your application does.
func sleepy(ctx context.Context) {
	_, span := tracer.Start(ctx, "sleep")
	defer span.End()

	sleepTime := 1 * time.Second
	time.Sleep(sleepTime)
	span.SetAttributes(attribute.Int("sleep.duration", int(sleepTime)))
}

// httpHandler is an HTTP handler function that is going to be instrumented.
func httpHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, World! I am instrumented automatically!")
	ctx := r.Context()
	sleepy(ctx)
}

func main() {
	// Wrap your httpHandler function.
	handler := http.HandlerFunc(httpHandler)
	wrappedHandler := otelhttp.NewHandler(handler, "hello-instrumented")
	http.Handle("/hello-instrumented", wrappedHandler)

	// And start the HTTP serve.
	log.Fatal(http.ListenAndServe(":3030", nil))
}
```

Assuming that you have a `Tracer` and [exporter]({{< relref "exporting_data" >}}) configured, this code will:
建设你已经有 `Tracer` 和 [exporter](exporting_data.md) 的相应配置 ，以上代码将：

* Start an HTTP server on port `3030`
* 在端口 `3030` 启动 HTTP 服务
* Automatically generate a span for each inbound HTTP request to `/hello-instrumented`
* 自动生成 `Span` 在每次请求 `/hello-instrumented` 时
* Create a child span of the automatically-generated one that tracks the work done in `sleepy`
* 创建一个自动生成的子 `Span`  , 当 `sleepy` 执行完成后

Connecting manual instrumentation you write in your app with instrumentation generated from a library is essential to get good observability into your apps and services.
将您在应用程序中编写的手动监测代码与监测库生成的检测代码连接起来，这对于让你的应用程序和服务具有良好的可观察性是至关重要的。

## Available packages
## 可用的监测库

A full list of instrumentation libraries available can be found in the [OpenTelemetry registry](/registry/?language=go&component=instrumentation).
可以在 [OpenTelemetry 注册表](https://opentelemetry.io/registry?language=go&component=instrumentation) 中找到可用的监测库的完整列表。

## Next steps
## 下一步

Instrumentation libraries can do things like generate telemetry data for inbound and outbound HTTP requests, but they don't instrument your actual application.
检测库可以监测诸如为接收和发出 HTTP 请求生成遥测数据之类的事情，但它们不会检测你的应用程序逻辑。

To get richer telemetry data, use [manual instrumentation]({{< relref "manual" >}}) to enrich your telemetry data from instrumentation libraries with instrumentation from your running application.
要获得更丰富的遥测数据，请使用 [手动监测库](manual.md) 来监测你的代码，这会丰富来自默认监测库的遥测数据。
