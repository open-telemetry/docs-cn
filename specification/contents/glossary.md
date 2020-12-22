# 术语表

本文档定义了本规范中使用的一些术语。

其他一些基本术语的定义请查阅 Overview 。



<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [通用](#common)
  * [In-band and Out-of-band Data](#in-band-and-out-of-band-data)
  * [Telemetry SDK](#telemetry-sdk)
  * [Exporter Library](#exporter-library)
  * [Instrumented Library](#instrumented-library)
  * [Instrumentation Library](#instrumentation-library)
  * [Tracer Name / Meter Name](#tracer-name--meter-name)
- [Logs](#logs)
  * [Log Record](#log-record)
  * [Log](#log)
  * [Embedded Log](#embedded-log)
  * [Standalone Log](#standalone-log)
  * [Log Attributes](#log-attributes)
  * [Structured Logs](#structured-logs)
  * [Flat File Logs](#flat-file-logs)

<!-- tocstop -->

## Common

<a name="in-band"></a>
<a name="out-of-band"></a>

### 带内带外数据 In-band and Out-of-band Data 

> 在通信领域, **带内信令**是指使用和数据(如语音或视频)相同的频段或信道内发送控制信息。这与**带外信令**相反，带外信令的发送使用不同的信道，甚至是通过单独的网络。 ([Wikipedia](https://en.wikipedia.org/wiki/In-band_signaling)).

在 OpenTelemetry 中，我们将带内数据定义为: 分布式系统的组件之间传递的数据，该数据是业务消息的一部分。例如 当 trace 或者  baggages 以 HTTP Header 的形式包含在 HTTP 请求中。这种数据通常不包含遥感信息，但是用于关联和连接各个组件产生的遥感信息。遥测本身被称为带外数据: 从应用程序发出，通过专用消息传输，通常由后台程序异步传输，而不是使用业务逻辑的关键路径传输。

例如:  带外信令: 导出到遥测后端的 Trace，Log 与 Metric。

### Telemetry SDK

基于 *OpenTelemetry API* 实现的库

详情请参考 [Library Guidelines](library-guidelines.md#sdk-implementation) 与
[Library resource semantic conventions](resource/semantic_conventions/README.md#telemetry-sdk).

### Exporter Library

兼容 [Telemetry SDK](#telemetry-sdk)  的库，提供向消费者(Consumers)发送遥感的功能。

### Instrumented Library

收集遥感信号(追踪，指标，日志)的库

可以从 Instrumented Library 自身或从另一个 Instrumentation Library 调用 OpenTelemetry Api 。	

例如: `org.mongodb.client`.

### Instrumentation Library

为指定的 Instrumented Libray 提供性能测量。

如果内置OpenTelemetry instrumentation，则 *Instrumented Library* 与 *Instrumentation Library* 可能是同一个库。

更详细的定义和命名指南请参见[Overview](overview.md#instrumentation-libraries) 

例如: `io.opentelemetry.contrib.mongodb`.

同义词: *Instrumenting Library*.

### Tracer Name / Meter Name

指定的名称和(可选)版本参数用于创建新的`追踪[Tracer]`或者`仪表[Meter]` (请参见 [Obtaining a Tracer](trace/api.md#tracerprovider)/[Obtaining a Meter](metrics/api.md#meter-interface)).

 [Instrumentation Library](#instrumentation-library)  名称/版本标识对

## Logs

### 日志记录 Log Record

对一个事件的记录。通常情况一个日志记录包含: 一个时间戳，指明事件的发生时间。一些描述事件如何发生，发生在何处等信息的数据。

同义词: *Log Entry*.

### 日志 Log

通常日志是一系列日志记录的集合. 有事会有歧义,自从人们有时候会将 日志(log) 等价于一条记录(Record). 因此本术语请谨慎使用，同时在可能存在歧义的情况下使用本术语时，应当使用额外的限定词(例如: 日志记录 Log Record)

### 嵌入日志 Embedded Log

一条嵌入在 Span 中的日志记录 

`Log Records` embedded inside a [Span](trace/api.md#span)
object, in the [Events](trace/api.md#add-events) list.

### 独立日志 Standalone Log

一个未被嵌入在 Span 中且被记录在其他地方的日志记录

### 日志属性 Log Attributes

键值对，包含在日志记录中 

### 结构化日志 Structured Logs

具有明确结构定义的日志，可以让你的日志结构更加清晰，从而区分日志记录的不同元素（如 时间戳，属性等) 

例如在 The _Syslog protocol_ ([RFC 5425](https://tools.ietf.org/html/rfc5424)) 中, 定义了 `structured-data`格式


### 扁平化日志 Flat File Logs

在文本文件中的日志记录，通常一行一个日志记录(尽管可以进行多行记录)

但对使用更结构化的格式（例如json 文件)的文本文件是否可以被认作是扁平化的日志，业内尚无共识。在这种区分很重要的情况下，推荐使用特别的名称用于区分。

