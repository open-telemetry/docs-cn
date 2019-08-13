# OpenTelemetry: 项目时间表

首先，我们的目标是把当前OpenTracing和OpenCensus都支持的语言SDK在OpenTelemetry实现，预计是
在**2019年9月**实现这些SDK,在**2019年11月**停止OpenTracing和OpenCensus的后续更新。

## 从OpenTracing/OpenCensus切换到OpenTelemetry，我们要做什么？

- 定义一个接口集合，实现OpenTelemetry的技术标准定义(specification)
- 多语言SDK，实现OpenTelemetry的技术标准定义
- 向后兼容OpenTracing/OpenCensus
- 为通用的操作定义统一的[语义约定](contents/data-semantic-conventions.md)
- 充分的测试，证明切换的互通性
- 充分的Benchmark, 证明CPU/内存资源的合理占用
- 使用文档

## 一致性

对于OpenTelemetry来说，一个极其重要的点就是一致性：对于用户来说，使用多语言的SDK时能获得一致性的体验，例如API等。
这种一致性就是在技术标准(specification)中定义的.

因此技术标准和多语言的SDK开发其实是同步，就是为了满足这种一致性。

### TL;DR;

Java和跨语言specification的时间表:

- 2019.6月底:
  - [实现基础的JAVA SDK](https://github.com/open-telemetry/opentelemetry-java/milestone/2)
  - 解决API使用上的反馈issue
- 7月中:
  - [实现JAVA SDK的数据导出](https://github.com/open-telemetry/opentelemetry-java/milestone/3)
  - [完成基础的SDK specification](https://github.com/open-telemetry/opentelemetry-specification/milestone/3)
  - [首个API修正文档](https://github.com/open-telemetry/opentelemetry-specification/milestone/2)
- 8月中:
  - [为SDK的扩展性功能添加文档](https://github.com/open-telemetry/opentelemetry-specification/milestone/4).
  - [API修正文档第二版](https://github.com/open-telemetry/opentelemetry-specification/milestone/5)
- 8月底:
  - [发布稳定版本的JAVA SDK的扩展性功能](https://github.com/open-telemetry/opentelemetry-java/milestone/4)
  - JAVA SDK生产可用
- 9月中 (或者在终端用户验证后):
  - API修订完成
- 9月底 (或者在终端用户验证后):
  - 发布1.0版本

### 当前状态

**API提案**:

- [Java已完成](https://github.com/open-telemetry/opentelemetry-java/milestone/1)
- API提案添加到[specification文档](https://github.com/open-telemetry/opentelemetry-specification/milestone/1)

**SDK提案**:

- Traces数据的基本处理流程完成
- 本月底完成限制范围内的工作
- SDK的Specification文档还没有开始

### 结束SDK提议

我们目前把SDK的提议工作限制在有限的范围内：

- Spans的流程处理:
  - SpanBuilder拦截接口
  - 内置采样频率控制(百分比采样)
  - SpanProcessor接口和实现:
  - 默认内置的处理器(processor)
    - Simple processor
    - Batching(批处理) processors
    - 队列满时阻塞 processor
  - 数据导出接口
  - 上报原始的spanData
- 分布式上下文(Distributed context)
  - 基础实现
- 指标(Metrics)
  - 实现指标聚合
  - MetricProducer接口

[**Java implementation**](https://github.com/open-telemetry/opentelemetry-java/milestone/2).
在限制范围内，6月底可以完成JAVA SDK的提案。

[**Specifications编写**](https://github.com/open-telemetry/opentelemetry-specification/milestone/3).
当前我们已经开始为SDK编写技术标准文档，时间节点上，大概是JAVA SDK完成两周后（7月中）。

### 基础的数据导出(exporters)

在默认情况下,OpenTelemetry导出到以下三个监控后端：Zipkin, Jaeger and Prometheus.

相关的文档依旧会在JAVA SDK完成两周后完成（7月中）。

具体参见
[Java](https://github.com/open-telemetry/opentelemetry-java/milestone/3).

### API迭代

随着API提案的发布，我们应开始针对收到的反馈进行修改，大概3个milestone内完成。

[**API修订 07-2019**](https://github.com/open-telemetry/opentelemetry-specification/milestone/2)
**(目标-7月中). API修订扫尾.**

- 简单的修正:  重命名, 代码优化, 移除不再需要的API接口
- 添加缺失的功能 - 添加histograms和getters
- 最新得到快速确认的Issues.例如为tracer添加组件

所以，该时间节点的目标是API扫尾.

**API修订 08-2019 (目标-8月中). 完成API修订.**

- 完成所有确认过的issues

**API修正 09-2019 (目标-9月中). API v1.0.**

- 保留终端用户反馈结果创建的issues

**未来:**

- 新的特性：这些特性要满足要求 - 能推迟到稳定版本后完成
- 新的采集资源类型支持

### SDK扩展

一旦JAVA的基础SDK完成，我们就会开始扩展它的功能集。

大概有以下扩展点：
1. OTSvc协议和实现
2. 添加缺失的功能
   1. 更多的SpanProcessors. 例如：实现数据选择性丢弃的非阻塞队列处理器
   2. 更多的采样控制. 例如：频率限制采样.
   3. 指标直方图(Histograms) – API和SDK
   4. 指标过滤器和处理器
   5. 其它
3. 讨论
   1.  原生(POJO)对象 vs. 使用SDK协议依赖库生成的对象
4. 可以处理Tracestate(译者注：网络协议传输中携带跟踪状态的对象)对象的回调函数
5. 其它

**8月中** 完成技术标准文档, **8月底** – JAVA SDK的首次迭代完成

### 准备发布

8月中，JAVA的基础SDK将会完成，这个时候我们会开始SDK稳定化工作。同时OpenCensus可以转为OpenTelemetry SDK。
同时，数据采集适配器将被实现。

截止9月初，我们将提交一个功能完整、生产可用的JAVA SDK，终端用户的反馈将会非常重要，决定了API和SDK的最终正确性。
因此这里我们可能不会称之为**V1.0**版本，而是**0.9**。因为OpenTelemetry是建立在OpenTracing/OpenCensus的基础智商，因此9月发布后，
理论上不会再有大的变动，然而，就像其它大型项目一样，这里会有一些关于API和SDK的问题存在

最终，在终端用户的协助下我们期待在9月底发布V1.0。

注意：由于技术标准定义工作(specification)的延后，其它语言的SDK可能无法在9月初发布生产可用的版本，因此每个语言的SDK需要单独设定时间节点。
