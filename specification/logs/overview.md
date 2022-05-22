# OpenTelemetry 日志 总览

**状态**: [起草阶段](https://opentelemetry.io/status/)

<details>
<summary>内容大纲</summary>

<!-- toc -->

- [简介](#introduction)
- [非OpenTelemetry解决方案目前的瓶颈](#limitations-of-non-opentelemetry-solutions)
- [OpenTelemetry解决方案介绍](#opentelemetry-solution)
- [日志如何做关联](#log-correlation)
- [事件和日志](#events-and-logs)
- [传统和现在日志源](#legacy-and-modern-log-sources)
  * [系统日志](#system-logs)
  * [组件日志](#infrastructure-logs)
  * [第三方应用日志](#third-party-application-logs)
  * [老系统自身业务日志](#legacy-first-party-applications-logs)
    + [Via File or Stdout Logs](#via-file-or-stdout-logs)
    + [Direct to Collector](#direct-to-collector)
  * [新系统自身业务日志](#new-first-party-application-logs)
- [OpenTelemetry Collector](#opentelemetry-collector)
- [自动检测已存在日志行为](#auto-instrumenting-existing-logging)


<!-- tocstop -->

</details>

## 简介

目前日志遥测数据是一个很大遗留问题. 大部分开发语言都有广泛的日志库去很好支撑日志功能。
Opentelemetry 相对于在 metrics、traces 目前有清晰设计，详细定义了全新的API，完整开发了多语言的API。
但是，日志考量上会有很大的差异：一方面，我们要提供改进、更好的可观测领域集成方案，同时还要兼容好过去系统存在的不同日志库。
对于Opentelemetry 这个目标难度极大，我们也欣然接受现状挑战：
致力于让已经存在日志库，日志收集系统，和日志方案都能很好运转。

## 非OpenTelemetry解决方案目前的瓶颈

很不幸运的事，目前整合日志解决方案对于可观测信号集成是比较弱的。很典型是在链路和监控工具中，比如时间戳、源头属性等相关信息的不完整，
日志支持受到限制。想要关联起来非常脆弱，因为属性值往往写入日志，链路和指标通过不同方式采集，比如不同采集器。
采集链路和指标时，没有一个通用方式能带上日志(比如应用系统和运行的基础组件)的源头信息，同时让遥测数据精准和强有力方式关联起来。

同时，日志没有通用方式去产生和记录请求的上下文。在分布式系统中经常产生不同的系统组件日志收集是不相关的。
这里展示没有 Opentelemetry 方式做可观测流程图的现状



![Separate Collection Diagram](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/img/separate-collection.png)

这里用到了不同库，不同的采集器，用到不同协议和数据模型，遥测数据采集完最后送到不同的独立后台，甚至都不知道一起工作是否正常


## OpenTelemetry 解决方案


分布式链路引进了请求上下文传播的概念

根本上讲，日志也有上下文传播的概念。日志保存下上下文标识(比如链路和Span的id或者用户自定义标识)，它就可以和链路形成丰富的关联。
而且分布式系统各个不同组件暴露日志，日志间也形成关联。这样，分布式系统日志变得非常有价值了。

这里有一种很有前景的可观测工具。标准化的metrics、traces和logs关联，支持了分布式上下文日志的传播，
统一metrics、traces和logs的源信息，提高了传统和现代系统可观测性信息的单个和组合的价值。
这是Opentelemetry 收集 logs, traces and metrics 整体的构想



![Unified Collection Diagram](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/img/unified-collection.png)

我们用Opentelemetry 约定数据格式去暴露logs, traces and metrics，发送采集数据到OpenTelemetry Collector，
Collector 它可以统一方式丰富处理数据。比如，一个Kubernetes Pod 的描述，不需要应用程序单独做任何处理，
这些描述Pod的属性自动通过Collector的 k8sprocessor 就能自动添加到遥测数据中。更重要的事，三个维度数据还被关联统一起来了。
Collector 保证了 logs, traces and metrics 里包含了同样的属性名、熟悉值，这样能够准确无误描述这个Kubernetes Pod来做于哪里。
在后端就能够拿到这个Pod 清晰、准确的关联起来的Pod 数据。



## 日志如何做关联



## Events and Logs



## Legacy and Modern Log Sources



### System Logs



### Infrastructure Logs



### Third-party Application Logs



### Legacy First-Party Applications Logs



#### Via File or Stdout Logs


#### Direct to Collector



### New First-Party Application Logs


## OpenTelemetry Collector


## Auto-Instrumenting Existing Logging



## Trace Context in Legacy Formats
