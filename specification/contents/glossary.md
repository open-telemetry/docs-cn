# 术语表

此文档定义了本规范中使用的一些术语。

其他一些基本术语的定义请查阅 [Overview](overview.md) 。



<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Common](#common)
  * [带内带外数据 In-band and Out-of-band Data](#带内带外数据-in-band-and-out-of-band-data)
  * [Telemetry SDK](#telemetry-sdk)
  * [Exporter Library](#exporter-library)
  * [Instrumented Library](#instrumented-library)
  * [Instrumentation Library](#instrumentation-library)
  * [Tracer Name / Meter Name](#tracer-name--meter-name)
- [Logs](#logs)
  * [日志记录 Log Record](#日志记录-log-record)
  * [日志 Log](#日志-log)
  * [嵌入日志 Embedded Log](#嵌入日志-embedded-log)
  * [独立日志 Standalone Log](#独立日志-standalone-log)
  * [日志属性 Log Attributes](#日志属性-log-attributes)
  * [结构化日志 Structured Logs](#结构化日志-structured-logs)
  * [扁平化日志 Flat File Logs](#扁平化日志-flat-file-logs)

<!-- tocstop -->

## Common

<a name="in-band"></a>
<a name="out-of-band"></a>

### 带内带外数据 In-band and Out-of-band Data 

> 在通信领域，**带内信令 **[in-band signaling] 是指使用和数据（如语音或视频）相同的频段或信道内发送控制信息。这与**带外信令**[out-of-band signaling]相反，带外信令的发送使用不同的信道，甚至是通过单独的网络 ([Wikipedia](https://en.wikipedia.org/wiki/In-band_signaling))。

在 OpenTelemetry 中，我们将**带内数据**定义为: 在分布式系统的组件之间传递的数据，且该数据是业务消息的一部分。例如使用 trace 或 baggages 以 HTTP Header 形式包含在 HTTP 请求中。这种数据虽然通常不包含遥感信息，但常用于关联和连接各个组件产生的遥感信息。遥测本身被称为**带外数据**: 从应用程序发出，通过专用消息传输，通常由后台程序异步传输，而不是使用业务逻辑的关键路径传输。

例如:  带外数据, 导出到遥测后端的 Trace，Log 与 Metric。

### Telemetry SDK

基于 *OpenTelemetry API* 实现的库。

详情请参考 [Library Guidelines](library-guidelines.md#sdk-implementation) 与 [Library resource semantic conventions](resource/semantic_conventions/README.md#telemetry-sdk)。

### Exporter Library

兼容 [Telemetry SDK](#telemetry-sdk)  的库，提供向消费者 [Consumers] 发送遥感的功能。

### Instrumented Library

收集遥感信号（追踪，指标，日志）的库。

可以从 Instrumented Library 自身或从另一个 Instrumentation Library 调用 OpenTelemetry Api 。	

例如: `org.mongodb.client`

### Instrumentation Library

为指定的 Instrumented Libray 提供性能测量。

如果内置OpenTelemetry instrumentation，则 *Instrumented Library* 与 *Instrumentation Library* 可能是同一个库。

更详细的定义和命名指南请参见 [Overview](overview.md#instrumentation-libraries)。

例如: `io.opentelemetry.contrib.mongodb`

同义词: *Instrumenting Library*

### Tracer Name / Meter Name

指定的名称和(可选)版本参数用于创建新的`追踪 [Tracer]`或者`仪表 [Meter]`（请参见 [Obtaining a Tracer](trace/api.md#tracerprovider)/[Obtaining a Meter](metrics/api.md#meter-interface)）。

 [Instrumentation Library](#instrumentation-library)  名称/版本标识对。

## Logs

### 日志记录 Log Record

对一起事件的记录。通常情况一个日志记录包含：

- 一个时间戳，指明事件的发生时间。

- 一些数据，用于描述事件如何发生，发生在何处等信息。

同义词: *Log Entry*.

### 日志 Log

通常日志是一系列日志记录的集合。本术语容易产生歧义，错误的将 Log 等价于一条消息记录[Record]。因此本术语请谨慎使用，同时在可能存在歧义的情况下使用本术语时，应当使用额外的限定词（例如: Log Record）。

### 嵌入日志 Embedded Log

一条嵌入在 Span 对象中，[Events](trace/api.md#add-events) 列表中的 Log Record。

### 独立日志 Standalone Log

一条未被嵌入在 Span 中且被记录在其他地方的 Log Record。

### 日志属性 Log Attributes

键值对，包含在 Log Record 中。 

### 结构化日志 Structured Logs

具有明确结构定义的 Logs，可以让你的日志结构更加清晰，从而区分 Log Record 的不同元素（如 时间戳，属性等） 。

例如在 The _Syslog protocol_ ([RFC 5425](https://tools.ietf.org/html/rfc5424)) 中, 定义了 `structured-data`格式。


### 扁平化日志 Flat File Logs

在文本文件中的 Logs，通常一行一条 Log Record（尽管可以进行多行记录）。

但对使用更结构化的格式（例如 JSON 文件）的文本文件是否可以被认作是扁平化日志，业内尚无共识。在这种区分很重要的情况下，推荐使用相对应的名称用于区分。
