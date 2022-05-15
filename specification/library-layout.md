# OpenTelemetry 工程结构

本文档展示了 OpenTelemetry 项目的基本的工程结构布局。对于包结构仅展示通用的部分，对某个特定语言的包结构不做强制要求。

## API 包结构

OpenTelemetry API推荐的包结构。

从最顶层目录开始的推荐包结构布局如下:

```
api
   ├── context
   │   └── propagation
   ├── metrics
   ├── trace
   │   └── propagation
   ├── baggage
   │   └── propagation
   └── internal
```

> 选择全小写、驼峰命名还是蛇形命名规则取决于使用的语言。

### `/context`

本目录包含正在执行的 context propagation 的 API 。

### [/metrics](./metrics/api.md)

本目录包含用于记录应用 Metrics 数据的 Metrics API 。

### [/baggage](baggage/api.md)

本目录包含可以用于管理 context propagation 和与 Metrics 相关的标签的 Baggage API。

### [/trace](trace/api.md)

跟踪 API 包含一些主要的类:

- `Tracer` 用于所有的操作. 详见 [Tracer](trace/api.md#tracer) 章节。
- `Span` 是一个保存了当前执行的操作的可变对象. 详见 [Span](trace/api.md#span) 章节。

### `/internal` (_Optional_)

私有的库和应用代码。

### `/logs` (_In the future_)

> TODO: 日志操作

## SDK Package

OpenTelemetry SDK 推荐的包结构。

从最顶层目录开始的推荐包结构布局如下:

```
sdk
   ├── context
   ├── metrics
   ├── resource
   ├── trace
   ├── baggage
   ├── internal
   └── logs
```

> 选择全小写、驼峰命名还是蛇形命名规则取决于使用的语言。

### `/sdk/context`

本目录包含 api/context 的 SDK 实现。

### `/sdk/metrics`

本目录包含 api/metrics 的 SDK 实现。

### `/sdk/resource`

resource 目录主要定义了 [Resource](overview.md#resources) 类型。该类型包含了统计和跟踪实体的信息。比如，
Kubernetes 可以导出指向 Kubernetes 集群、命名空间、 Pod 和容器名称的 Metrics 。

### `/sdk/baggage`


### [/sdk/trace](trace/sdk.md)

本目录包含sdk/trace的SDK实现。

### `/sdk/internal` (_Optional_)

私有应用和库代码。

### `/sdk/logs` 

本目录包含sdk/logs的SDK实现。
