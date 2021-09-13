# 导出器（Exporter）开发者指南

导出器（Exporter）将追踪（Tracing）和指标（metrics）发送到任何能够使用它们的后端。 使用 OpenTelemetry，您可以轻松添加和删除任何导出器（Exporter），而无需更改您的应用程序代码。


我们为多个开箱即用的开源后端和供应商提供支持，如 Zipkin、Jaeger 和 Prometheus，但 OpenTelemetry 导出器（Exporter）遵循任何人都可以实现的公共接口。 本文档描述了开发人员在提供的导出器（Exporter）不满足其需求时创建自己的导出器（Exporter）的过程。

标准的包结构：

```text
opentelemetry-exporter-myexporter
   ├── src
   │   └── index.ts
   │   └── transform.ts
   │   └── types.ts
   │   └── myexporter.ts
   └── test
       └── transform.test.ts
       └── myexporter.test.ts
```

## 追踪（Tracing）

`SpanExporter` 接口定义了特定于协议的追踪（Tracing）/Span 导出器（Exporters）必须实现的方法，以便它们可以插入 OpenTelemetry SDK。 Span 导出器（Exporters）必须遵循以下规则：

1. 实现 `SpanExporter` 接口。
2. 预期只接收已采样的 span。
3. 预计只接收已结束的 span。
4. 不抛出异常。
5. 不修改已接收的 spans。
6. 不实现排队或批处理逻辑，这些应该又 span 处理器解决。

当前的`SpanExporter`接口（`0.2.0`）包含 2 个方法：

- `export`: 导出一批 spans。在这个方法中，你可以处理和转换`ReadableSpan`的数据为你跟踪器所接收的数据，然后发送他们到你的接收器后端。

- `shutdown`: 关闭导出器（Exporter）。 可以在这里清理导出器（Exporter）的所有数据。 `Shutdown`只应该被每一个导出器（Exporter）实例调用一次。 调用 `Shutdown` 后，导出器（Exporter）不允许也不应该返回 `FailedNotRetryable` 错误。

更全面的例子请参考[Zipkin Exporter][zipkin-exporter]或[Jaeger Exporter][jaeger-exporter]。

## 指标（Metrics）

`MetricExporter` 定义了特定于协议的导出器（Exporter）必须实现的接口，以便它们可以插入 OpenTelemetry SDK 并支持发送指标（Metrics）数据。

当前`MetricExporter`接口 (`0.2.0`) 定义了2个方法:

- `export`:批量导出监测数据。在这个方法中，你可以处理和转化`MetricRecord`数据为你的指标（Mertric）服务可接收的数据。

- `shutdown`: 关闭导出器（Exporter）。 可以在这里清理导出器（Exporter）的所有数据。 `Shutdown`只应该被每一个导出器（Exporter）实例调用一次。 调用 `Shutdown` 后，导出器（Exporter）不允许也不应该返回 `FailedNotRetryable` 错误。

更全面的例子请参考[Zipkin Exporter][zipkin-exporter]或[Jaeger Exporter][jaeger-exporter]。

[zipkin-exporter]: https://github.com/open-telemetry/opentelemetry-js/blob/main/packages/opentelemetry-exporter-zipkin
[jaeger-exporter]: https://github.com/open-telemetry/opentelemetry-js/blob/main/packages/opentelemetry-exporter-jaeger
[prometheus-exporter]: https://github.com/open-telemetry/opentelemetry-js/blob/main/packages/opentelemetry-exporter-prometheus
