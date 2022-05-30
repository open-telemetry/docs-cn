# 概述

<details>
<summary>
Table of Contents
</summary>
<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [分布式跟踪(distributed trace)](#分布式跟踪distributed-trace)
  * [Trace](#trace一种数据结构代表了分布式跟踪链路)
  * [Span](#span一种数据结构代表了trace中的某一个片段)
  * [SpanContext](#spancontextspan上下文)
  * [Span之间的Links](#span之间的links链接)
- [Metrics](#指标metrics)
  * [记录原始数据](#记录原始数据)
    + [Measure](#measure)
    + [Measurement](#measurement)
  * [使用预定义的聚合方式记录指标](#使用预定义的聚合方式记录指标)
  * [指标数据模型（Metrics Data Model）和SDK](#指标数据模型metrics-data-model和sdk)
- [Logs](#logs)
  * [数据模型](#数据模型)
- [Baggage](#baggage)
- [Resources](#resources)
- [Context Propagation](#context-propagation)
- [Propagators](#propagators传播者)
- [Collector](#collector)
- [Instrumentation Libraries](#instrumentation-libraries)
- [Semantic Conventions](#semantic-conventions)

<!-- tocstop -->

</details>

## 分布式跟踪(distributed trace)
一条分布式跟踪是一系列事件(Event)的顺序集合。这些事件分布在不同的应用程序中，可以
跨进程、网络和安全边界。例如，该链路可以从用户点击一个网页的按钮开始，在这种情况下，该跟踪会包含从点击按钮开始，所经过的所有下游服务，
最终串起来形成一条链路。

### Trace(一种数据结构，代表了分布式跟踪链路)

**Traces** 在OpenTelemetry中是通过**Spans**来进一步定义的.  我们可以把一个**Trace**想像成由
**Spans**组成的有向无环图（DAG）, 图的每条边代表了**Spans**之间的关系——父子关系。

For example, the following is an example **Trace** made up of 6 **Spans**:

```
下图展示了在一个Trace中Spans之间常见的关系


        [Span A]  ←←←(根Span(root span)))
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C是Span A的孩子)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] 
```

有时候我们可以用时间轴的方式来更简单的可视化**Traces**，如下图：

```

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> 时间轴

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··]
```

### Span(一种数据结构，代表了Trace中的某一个片段)

每个**Span**都封装了以下状态:

- 一个操作名An operation name
- 开始/结束时间戳A start and finish timestamp
- Key:Value形式的属性集合，Key必须是字符串，Value可以是字符串、布尔或者数字类型
- 0个或者多个事件(Event), 每个事件都是一个Key:Value Map和一个时间戳
- 该Span的父Span ID
- [**Links**(链接)](#Span之间的Links(链接))到0个或多个有因果关系的**Spans**
  (通过那些**Spans**的**SpanContext**).
- 一个Span的**SpanContext** ID

### SpanContext(Span上下文)

包含所有能够识别**Trace**中某个**Span**的信息，而且该信息必须要跨越进程边界传播到子Span中。
一个**SpanContext**包含了将会由父**Span**传播到子**Span**的跟踪ID和一些设置选项。

- **TraceId** 是一条Trace的全局唯一ID，由16个随机生成的字节组成,TraceID用来把该次请求链路的所有Spans组合到一起
- **SpanId** 是Span的全局唯一ID，由8个随机生成的字节组成，当一个SpanID被传播到子Span时，该ID就是子Span的父SpanID
- **TraceFlags** 代表了一条Trace的设置标志，由一个字节组成(里面8个bit都是设置位) 
  - 例如采样Bit位 -  设置了该Trace是否要被采样（掩码`0x1`).
- **Tracestate** 携带了具体的跟踪内容，表现为[{key:value}]的键指对形式,**Tracestate** 允许不同的APM提供商加入额外的自定义内容和对于旧ID的转换处理，更多内容请查看[这里](https://w3c.github.io/trace-context/#tracestate-field).

### Span之间的Links(链接) 

一个**Span**可以和多个其它**Span**产生具有因果关系的链接(通过**SpanContext**定义)。

这些链接可以指向某一个**Trace**内部的**SpanContexts**，也可以指向其它的**Traces**。

**Links**可以用来代表这种批量操作：一个**Span**需要多个**Span**进行初始化。    
另一个使用**Link**的例子是：申明原始trace和后续trace之间的关系，例如**Trace**进入了一个受信边界，进入后需要重新生成一个新的Trace。
新的被链接的Trace也可以用来代表被许多快速进入的请求之一所初始化的一个长时间运行的异步数据处理操作。

在上述分散/聚合模式下，当根操作开启多个下游处理时，这些都会最终聚合到一个**Span**中，这个最后的**Span**被链接到它所聚合
的多个操作中。最后的Span将会被进行聚合其操作的Span所链接，这些Span都是来自同一个Trace，类似于Span的parent字段。
但是建议在这个场景下，建议不要设置Span的parent字段，因为从语义上来说，parent字段表示单亲场景，父Span将
完全包含子Span的所有范围，但是在分散/聚合模式和批处理场景下并不是如此。

## 指标(Metrics)

OpenTelemetry允许用户使用预定义的聚合方法和标签记录原始数据与指标。

当使用OpenTelemetry API记录原始数据时，我们可以在最后决定使用什么样的聚合算法或者标签来记录指标。当我们采用客户端库比如gRPC时，
我们可以记录“服务器端延迟”或“接收字节数”等原始数据。因此终端用户就可以在大量的原始数据中，决定收集什么样的数据：可能是简单的
求平均或详细的直方图(histogram)计算。

这种预定义聚合的方式来进行指标收集实际上是非常重要的，例如我们可以用来收集CPU和内存使用信息，也可以收集简单的指标例如"队列长度"。

### 记录原始数据

用来记录原始数据主要的代码类是 `Measure` 和 `Measurement`。我们可以用OpenTelemetry API记录
`Measurement`列表，同时记录一些上下文信息。因此用户可以自定义如何聚合`Measurement`，同时可以使用上下文信息对
最终指标结果进行额外的定义。


#### Measure

`Measure` 表示通过使用某一个库采集到的独立的数据。同时它还在采集数据和聚合数据之间定义了一个合约，最终结果会进入
 `Metric`。
`Measure` 由名称、描述和一系列单位值组成。

#### Measurement

`Measurement` 描述了该如何采集数据（采集的数据就是 `Measure`）.

`Measurement` 是一个空的API接口，这个接口由具体的采集SDK实现。

### 使用预定义的聚合方式记录指标

 `Metric`代表了一种数据结构：指标预聚合. 它定义了基本的指标属性：名称、标签。那些继承自`Metric`的类
定义了聚合的类型同时也会定义一些独立的`Measurements`，我们的API预定义了一些预聚合的指标：
- Counter(计数器)指标，瞬时测量值。Counter的值可以增长也可以保持不变，但是不降低。Counter的值不能为负数，
Counter有两种类型：`double`和`long`
- Gauge(实时值)指标，它也是一种瞬时测量值。与Counter不同的是，Gauges可以上下变动，也可以为负数。Gauges也有两种类型：
`double`和`long`

API允许构造一个指定类型的`Metric`，SDK则定义了一个被导出的`Metric`的当前值该怎么查询。

每个`Metric`类型都有自己的API来记录数据，而且支持读取或设置`Metric`的值。


### 指标数据模型（Metrics Data Model）和SDK

指标数据模型是在SDK中定义的，基于
[metrics.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto)协议。

该数据模型被所有OpenTelemetry导出器(exporters)作为数据输入来使用。不同的导出器有不同的能力(例如支持哪种数据类型)，
还有不同的限制(例如在标签Key中哪些字符被允许)。所有的导出器都通过OpenTelemetry SDK中定义的指标生产者接口(Metric Producer interface)从数据指标模型中消费数据，

因为上面的原因，指标对于数据本身是做了最小限制的(例如在标签Key中支持哪些字符)，处理指标的代码应该避免对这些指标数据进行
验证和净化。你应该做的是：把这些数据传到服务器端，让服务器端来做验证，然后从服务器端接收错误。

## 日志 
[日志总览](/specification/logs/overview.md)
### 日志 RoadMap
 - [OTLP](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/logs/data-model.md)：`Opentelemetry Log Protocol` 日志协议标准现在已经是稳定版本 
 - [Log数据模型](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/logs/data-model.md)：现在已经是稳定版本
 - OpenTelemetry Collector Log：测试阶段
## Baggage
除了Trace传播之外，OpenTelemetry还提供了一个简单的机制来传播键值对，这一机制被称为**Baggage**。**Baggage**可以为一个服务查看可观测事件提供索引，
其属性则是由同一个事务的前一个服务提供。这有助于在各个事件之间建立因果关系。    

虽然**Baggage**也为其他横切关注点的实现提供了原型，但是这一机制主要还是为了在OpenTelemetry所观测的系统之间进行值传递。     

下面这些值可以在**Baggage**中进行使用，并提供额外维度的metric指标或者为trace追踪和log日志提供额外的上下文内容。下面是一些例子：
- 一个web服务提供方可以在上下文中得到服务的调用方的信息
- Saas提供者可以在上下文中记录API的使用者及其持有令牌的信息
- 可以确定特定的浏览器版本与图像处理服务的故障的关联关系

为了让OpenTracing向后兼容，当使用OpenTracing bridge的时候Baggage将以**Baggage**进行传播。具有不同标准的新的关注点应该去新建一个横切关注点来覆盖其用例。
这些新的关注点可能受益于W3C编码格式，但是需要使用新的HTTP头来在分布式追踪中传播数据。

## Resources

`Resource`记录当前发生采集的实体的信息(虚拟机、容器等)，例如，Metrics如果是通过Kubernetes容器导出的，那么resource可以记录
Kubernetes集群、命名空间、Pod和容器名称等信息。

`Resource`也可以包含实体信息的层级结构，例如它可以描述云服务器/容器和跑在其中的应用。

注意，一些OpenTelemetry的SDK或者特定的导出器也会自动采集`Resource`信息，具体例子请查看:
[proto](https://github.com/open-telemetry/opentelemetry-proto/blob/a46c815aa5e85a52deb6cb35b8bc182fb3ca86a0/src/opentelemetry/proto/agent/common/v1/common.proto#L28-L96)

## Context Propagation
所有OpenTelemetry中的关注点（比如trace和metric指标）都共享一个底层上下文机制，用于在分布式事务的整个生命周期内存储状态和访问数据。    

详情可以查看[这里](context/context.md)。

## Propagators(传播者)

OpenTelemetry使用`Propagators`来序列化和反序列化中的关注点，比如Span（通常仅限于SpanContext部分）和Baggage。
不同的`Propagator`的类型定义了特定传输方式下的限制并将其绑定到一个数据类型上。     
传播者Propagators API定义了一个`Propagator`类型：

- `TextMapPropagator` 可以将值注入文本，也可以从文本当中提取值

## Collector

OpenTelemetry collector是由一系列组件组成的，这些组件可以使用OpenTelemetry或三方SDK(Jaeger、Prometheus等)收集trace跟踪、metric指标和日志等数据，
然后对这些数据进行聚合、智能采样，再导出到一个监控后端。Collector将允许加工转换收集到的数据（比如添加额外属性或者删除隐私数据）。

OpenTelemetry服务由两个主要的模型：Agent(一个本地代理)和Collector(一个独立运行的服务)。

阅读更多 [Long-term
Vision](https://github.com/open-telemetry/opentelemetry-service/blob/master/docs/vision.md).


## Instrumentation Libraries
详情可以查看[Instrumentation Libraries](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumentation-library)。    

这个项目的灵感在于通过让各个库和应用程序通过直接调用OpenTelemetry的API来达到开箱即用的目标。但是，大部分库不会做这样的集成，因此需要一个单独的库来注入
这样这样的调用，通过诸如包装接口，订阅特定库的回调或者将当前的监测模型改为OpenTelemetry模型等机制来实现。
     
一个被让另一个库实现OpenTelemetry观测能力的库叫做[Instrumentation Library](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumentation-library)。    
一个Instrumentation Library的命名应该遵循任意关于对于这个库的命名规范(比如web框架的'中间件')。      
如果没有已经确定的名称，建议在包前面加上“opentelemetry instrumentation”前缀，然后加上被集成的库名称本身。示例包括：     

- opentelemetry-instrumentation-flask (Python)
- @opentelemetry/instrumentation-grpc (Javascript)

## Semantic Conventions
OpenTelemetry约定了Resource属性与Span属性的标准值与名称。 
- [Resource约定](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/semantic_conventions/README.md)
- [Span约定](trace/semantic_conventions/README.md)
- [Metrics约定](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/semantic_conventions/README.md)

 属性的类型应该在语义约定中指定。可以查阅[Attributes](common/common.md#属性)
 部分了解更多。
