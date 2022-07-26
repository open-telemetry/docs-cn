
# 处理和导出数据

当你的代码被监测之后，就需要导出数据以便用它做一些有用的事情。 此页面将介绍处理管道和导出管道的基础知识。
## 采样

采样是一个限制系统生成的追踪链路数量的过程。 如何设置采样器取决于你的需求，但通常你应该在链路开始时做出决定，并允许将采样策略传播到其他服务。  
在配置跟踪器提供程序时，需要在其上设置采样器，如下所示： 
```go
provider := sdktrace.NewTracerProvider(
	sdktrace.WithSampler(sdktrace.AlwaysSample()),
)
```

`AlwaysSample` 和 `NeverSample` 可以从名字中看出功能。 `Always` 意味着将对每条链路进行采样，`Never`则与之相反。 当你刚刚开始使用时，或者在开发环境中，你应该一直使用`AlwaysSample`。  
其他采样器还包括：

* `TraceIDRatioBased`，它将根据提供给采样器的分数对部分链路进行采样。 例如，如果将其设置为 0.5，将采样一半的链路。
* `ParentBased`，根据父代传入的采样决策表现不同。 一般来说，如果父代的 Span 被采样，那么当前的 Span 才会被采样。如果父代的 Span 不被采样，那么当前的 Span 也不会被采样。
在生产环境中，你应该考虑将 `TraceIDRatioBased` 采样器与 `ParentBased` 采样器一起使用。

## 资源

资源是一种特殊类型的属性，用于进程生成的所有跨度。 用于表示和进程有关的非临时的底层元数据 -- 例如，进程的主机名或实例 ID。

资源应该在初始化时分配给跟踪器提供者，并像属性一样被创建：

```go
resources := resource.NewWithAttributes(
	semconv.SchemaURL,
	semconv.ServiceNameKey.String("myService"),
	semconv.ServiceVersionKey.String("1.0.0"),
	semconv.ServiceInstanceIDKey.String("abcdef12345"),
)

provider := sdktrace.NewTracerProvider(
	...
	sdktrace.WithResource(resources),
)
```

请注意使用 `semconv` 包为资源属性提供惯用名称。 这有助于使用这些遥测数据的消费者可以轻松的通过语义约定发现相关属性并理解其含义。

资源也可以通过 `resource.Detector` 实现自动检测。 这些`Detector`可以发现有关当前正在运行的进程的信息，正在运行的操作系统信息，托管该操作系统实例的云提供商信息，以及其他一些资源属性。

```go
resources := resource.New(context.Background(),
	resource.WithFromEnv(), // pull attributes from OTEL_RESOURCE_ATTRIBUTES and OTEL_SERVICE_NAME environment variables
	resource.WithProcess(), // This option configures a set of Detectors that discover process information
	resource.WithDetectors(thirdparty.Detector{}), // Bring your own external Detector implementation
	resource.WithAttributes(attribute.String("foo", "bar")), // Or specify resource attributes directly
)
```

## OTLP 导出器

OpenTelemetry 数据传输协议 (OTLP) 导出相关的信息可以在  `go.opentelemetry.io/otel/exporters/otlp/otlptrace` 和 `go.opentelemetry.io/otel/exporters/otlp/otlpmetrics` 包中找到。

在 [GitHub](https://github.com/open-telemetry/opentelemetry-go/tree/main/exporters/otlp) 中可以找到更多文档

## Jaeger 导出器

Jaeger 导出相关信息可以在  `go.opentelemetry.io/otel/exporters/jaeger` 中找到。

在 [GitHub](https://github.com/open-telemetry/opentelemetry-go/tree/main/exporters/jaeger) 中可以找到更多文档

## Prometheus 导出器

Prometheus 导出相关信息可以在 `go.opentelemetry.io/otel/exporters/prometheus` 中找到。  
在 [GitHub](https://github.com/open-telemetry/opentelemetry-go/tree/main/exporters/prometheus) 中可以找到更多文档
