# 使用监测库

Go 不像当前其他语言那样支持真正的自动检测。 相反，您需要依赖为特定监测库生成遥测数据的 [监测库](https://opentelemetry.io/docs/reference/specification/glossary/#instrumentation-library)。 例如，当你将`net/http` 的监测库配置到你的代码中后，监测库将自动创建跨度在接收和发出请求时。

## 设置

每个监测库都是一个包。 一般来说，这意味着您需要 `go get` 适当的包：

```console
go get go.opentelemetry.io/contrib/instrumentation/{import-path}/otel{package-name}
```

然后根据相应库的需求，在你的代码中配置和启动。

## `net/http` 示例
例如，以下是如何为 `net/http` 在接收 `HTTP` 请求时设置自动检测：

首先，获取 `net/http` 监测库：

```console
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

下一步，在你的代码中用这个库包裹 HTTP 处理器：

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

假设你已经有 `Tracer` 和 [exporter](exporting_data.md) 的相应配置 ，以上代码将：

* 在端口 `3030` 启动 HTTP 服务
* 自动生成跨度在每次请求 `/hello-instrumented` 时
* 创建一个自动生成的子跨度 , 当 `sleepy` 执行完成后

将您在应用程序中编写的手动监测代码与监测库生成的检测代码连接起来，这对于让你的应用程序和服务具有良好的可观察性是至关重要的。

## 可用的监测库

可以在 [OpenTelemetry 注册表](https://opentelemetry.io/registry?language=go&component=instrumentation) 中找到可用的监测库的完整列表。

## 下一步

检测库可以监测诸如为接收和发出 HTTP 请求生成遥测数据之类的事情，但它们不会检测你的应用程序逻辑。

要获得更丰富的遥测数据，请使用 [手动监测库](manual.md) 来监测你的代码，这会丰富来自默认监测库的遥测数据。
