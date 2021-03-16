# Metrics API

<!-- toc -->

- [总览](#总览)
  * [测量](#测量)
  * [Metric Instruments](#metric-instruments)
  * [标签](#标签)
  * [Meter Interface](#meter-interface)
  * [聚合](#聚合)
  * [时间](#时间)
  * [Metric Event格式](#Metric-Event格式)
- [Meter provider](#meter-provider)
  * [获取一个Meter](#获取一个Meter)
  * [全局 Meter provider](#全局-Meter-provider)
    + [获取全局的MeterProvider](#获取全局的MeterProvider)
    + [设置全局的MeterProvider](#设置全局的MeterProvider)
- [Instrument属性](#Instrument属性)
  * [Instrument命名要求](#Instrument命名要求)
  * [同步和异步instruments比较](#同步和异步instruments比较)
  * [Adding instruments与grouping instruments比较](#Adding-instruments与grouping-instruments比较)
  * [单调和非单调instruments的比较](#单调和非单调instruments的比较)
  * [方法名称](#方法名称)
- [Instrument说明](#Instrument说明)
  * [Counter](#counter)
  * [UpDownCounter](#updowncounter)
  * [ValueRecorder](#valuerecorder)
  * [SumObserver](#sumobserver)
  * [UpDownSumObserver](#updownsumobserver)
  * [ValueObserver](#valueobserver)
  * [Interpretation](#解释)
  * [构造函数](#构造函数)
- [标签集合](#标签集合)
  * [标签性能](#标签性能)
  * [Option: Ordered labels](#option-ordered-labels)
- [同步instrument详情](#同步instrument详情)
  * [同步调用约定](#同步调用约定)
    + [绑定调用instrument约定](#绑定调用instrument约定)
    + [直接调用instrument约定](#直接调用instrument约定)
    + [RecordBatch调用约定](#RecordBatch调用约定)
  * [Association with distributed context](#与分布式上下文关联)
    + [Baggage中的metric标签](#Baggage中的metric标签)
- [异步instrument细节](#异步instrument细节)
  * [异步调用约定](#异步调用约定)
    + [单一instrument的监测](#单一instrument的监测)
    + [Batch observer](#batch-observer)
  * [Asynchronous observations form a current set](#asynchronous-observations-form-a-current-set)
    + [Asynchronous instruments define moment-in-time ratios](#asynchronous-instruments-define-moment-in-time-ratios)
- [并发](#并发)
- [Related OpenTelemetry work](#related-opentelemetry-work)
  * [Metric Views](#metric-views)
  * [OTLP Metric protocol](#otlp-metric-protocol)
  * [Metric SDK default implementation](#metric-sdk-default-implementation)

<!-- tocstop -->

## 总览
`OpenTelemetry Metrics API`支持在运行时获取有关计算机程序执行情况的参数，
`Metrics API`是专门为处理原始测量数据而设计的，通常是为了高效且并行的对这些测量结果进行连续的统计分析。
以下，`API`是指`OpenTelemetry Metrics API`。

`API`通过提供不同级别性能的几个调用约定，用于捕获原始度量的函数。
无论什么级别的调用约定, 我们都将`metric event`定义为捕获新的测量时发生的逻辑事件。
捕获的这一时间（在“运行时”）定义了一个隐式时间戳，这是SDK在该时刻从系统时钟读取的时间。

这里使用的`semantic`或者`semantics`一词是指我们如何为`metric events`赋予含义，
该术语在本文档中得到广泛使用，以定义和解释这些API函数以及我们应如何解释它们。
尽可能地，此处使用的术语试图传达预期的语义，下面将描述一个标准实现，以帮助我们理解它们的含义。
标准实现对每种`metric event`执行与默认解释对应的聚合。

监控和报警系统一般使用`metric events`所提供的数据, 这些数据经过[聚合](#聚合)并转换为各种展示格式的数据。
然而，我们还发现`metric events`的数据还有许多其他用途，例如在Tracing和日志系统中记录聚合的或原始的度量。
因此，
[OpenTelemetry需要将API与SDK分开](../library-guidelines.md#requirements)
以便可以在运行时配置不同的SDK。

### 在没有安装SDK情况下的API行为

在没有安装Metrics SDK的情况下，Metrics API仅可以由`无操作`组成，API的调用都将不具备任何的意义。 
`Meters`必须返回任何`instruments`的无操作实现，从用户的角度来看，应在不引发错误的情况下忽略对它们的调用
（即，如果一个语言会在返回`null`引用时触发异常，则不要这样做）。
API不得在对其进行的任何调用上引发异常。

### 测量
本文档中使用术语`捕获`（capture）来描述将用户测量出的数据传递给API时所执行的操作。
`捕获`（capture）的结果取决于所配置的SDK，如果SDK没有被安装，则默认操作是不会执行捕获的事件。
这种用法的目的是，根据不同的SDK来传达任何可能会发生的测量结果，但是这也意味着用户正在进行着某些测量。
出于性能和语义（semantic）方面的考虑，API允许用户在这两种测量方式之间进行选择。

术语`添加`（adding）用于指定某些测量的特征，在该种测量下，仅将总和的量视为有用信息。
这些是可以使用算术加法（通常是某些事物的实际数量，例如字节数）进行组合的度量。

分组度量在值集时使用，如果一个值的集合自身具有一定的意义，则使用分组度量。
分组度量是一种您不会使用算术加法自然组合的度量（例如请求延迟），或者，当您的目的是监视值的分布时，也会添加这种度量（例如队列的大小）。
在分组度量中，中位数也是有用的信息。

分组工具在功能上比添加工具捕获更多的信息。根据此定义，分组测量会消耗比添加测量更高的系统性能。
用户应该优先选择添加工具，除非他们希望从有关单个数据的额外信息中获取到其他价值。

### Metric Instruments

`metric instrument` 是用于在API中捕获原始测量值的组件。
下表中列出的标准的`instrument`，每一种都有特定的用途，在API中刻意避免了改变`instrument`的可选特性; 
相反，API更喜欢支持单一方法的`instrument`，每个方法都有固定的实现。

API捕获的所有度量数据都与用于进行度量的`instrument`相关联，
因此赋予了度量其语义属性。`instrument`是通过调用`Meter`API来创建和定义的，API是SDK面向用户的入口点。

`instrument`通过以下的几种方式进行区分：
1.同步性：同步`instrument`由用户在分布式
  [上下文](../context/context.md)
  中调用（即，具有关联的`Span`，`Baggage`等）。在没有上下文的情况下，SDK会在每个收集间隔调用一次异步工具。
2.`添加`与`分组`：添加工具是一种记录添加测量值的工具，与上述的分组工具相反。
3.单调性：`monotonic instrument`是一种`adding instrument`，其总和的进度不减。`monotonic instrument`对于监视速率信息很有用。

度量工具名称以及它们是否`synchronous`，`adding`或`monotonic`显示在下面表格中。

| 名称 | Synchronous | Adding | Monotonic |
| ---- | ----------- | -------- | --------- |
| Counter           | 是 | 是 | 是 |
| UpDownCounter     | 是 | 是 | 否 |
| ValueRecorder     | 是 | 否 | 否 |
| SumObserver       | 否 | 是 | 是 |
| UpDownSumObserver | 否 | 是 | 否 |
| ValueObserver     | 否 | 否 | 否 |

`synchronous instruments`对于在分布式
[上下文](../context/context.md)
中（即，具有关联的`span`，`Baggage`等）收集的测量很有用。
当测量成本很高时，`asynchronous instruments`很有用，因此应定期收集。在下面阅读更多
[同步和异步instruments的特性](#同步和异步instruments比较)

同步和异步`adding instruments`有很大的区别：同步`instruments`用于捕获总和的变化，而异步工具用于直接捕获总和。在下面阅读
[`adding instruments`的特征](#Adding-instruments与grouping-instruments比较)。

`monotonic adding instrument`是重要的工具，因为他们支持比例计算。阅读
[选择`metric instruments`](#单调和非单调instruments的比较)
以获取更多的信息。

一个`instrument`的定义，描述了这个`instrument`的一些特性，包括他的名称和种类。
`metric instrument`的一些其他属性是可配置的，包括其说明和度量单位。
一个`instrument`的定义与其产生的数据之间相关联。

### 标签

标签是用于指代与`metric event`关联的Key-Value属性的术语，类似于`Tracing API`中的
[Span attribute](../trace/api.md#span)。
每个标签对于进行`metric event`分类，从而可以对事件进行过滤和分组以进行分析。

每一个`instrument`的调用约定（在下面详细说明）都接受一组标签作为参数。
标签的集合定义为从Key到Value的唯一映射。
通常，标签以key：values列表的形式传递给API，在这种情况下，规范规定通过在列表中显示最后一个值来解析键的重复项。
（此处翻译存疑，原文：in which case the specification dictates that duplicate entries for a key are resolved by taking the last value to appear in the list.）

如果将同步`instrument`的测量与具有完全相同标记集的其他测量相结合，那么它将得到显著的优化。
相关的内容可以阅读[通过聚合来组合度量](#聚合)

### Meter Interface

API定义了一个Meter接口。该接口由一组`instrument`构造器，和一个使用原子方式批量获取测量值的工具组成。

有一个全局Meter实例可供使用，它有助于自动检测第三方代码。
使用此实例可以使代码静态初始化`metric instruments`，而无需显式依赖注入。
全局`Meter`实例充当无操作实现，直到应用程序`Meter`通过服务提供商接口或其他特定于语言的支持显式，安装SDK来初始化全局实例。
请注意，没有必要直接使用全局实例：OpenTelemetry SDK的多个实例可以同时运行。

作为强制性的步骤，API要求调用者在获得`Meter`实现时提供`instrumenting`库的名称（可选，版本）。
库名称旨在用于标识从该库产生的检测，例如禁用检测，配置聚合和应用采样策略。有关更多详细信息，请参见
[TracerProvider](../trace/api.md#tracerprovider)
上的规范。

### 聚合

聚合是指将多个维度的度量，合并为程序执行期间的时间间隔内发生的具体或者预估的测量数据的统计信息的过程。
每个`instrument`都指定一个适合该工具语义的默认聚合，用于解释其属性并使用户了解如何使用它。
在没有任何配置替代的情况下，`instrument`可以提供开箱即用的简约的汇总。

`adding instruments` (`Counter`, `UpDownCounter`, `SumObserver`,`UpDownSumObserver`)在默认的情况下使用的是求和聚合
用于计算聚合数据的详细信息有所不同，但是从用户的角度来说，这意味着他们将能够监测捕获的值的总和。
同步于异步的`instruments`之间的区别，对于指定到处工具的工作方式至关重要。
这是
[SDK 规范 (WIP)](https://github.com/open-telemetry/opentelemetry-specification/pull/347)
涵盖的主题。

`ValueRecorder``instrument`采用
[TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636)
默认聚集。

`ValueObserver``instrument`默认使用LastValue聚集。此聚合跟踪所观察到的最后一个值和它的时间戳。

还可以使用其他标准聚合，尤其是对于`grouping instruments`，
我们普遍关心的各种不同的报表，例如直方图，分位数报表，基数估计和其他种类的数据结构。

默认的OpenTelemetry SDK实现了[Views API (WIP)](https://github.com/open-telemetry/oteps/pull/89)，
该API支持在单个`instrument`的级别上配置非默认的聚合行为。
即使OpenTelemetry SDK可以配置为以非标准方式处理`instruments`，但仍希望用户根据其语义来选择`instruments`，使用默认聚合进行处理。

### 时间

时间是`Metric Event`的基本属性，但不是必须的属性。用户不为`Metric Event`提供显式的时间戳。
不建议SDK捕获每个事件的当前时间戳（通过读取时钟），除非明确需要每个`Event`都需要计算高精度的时间戳。

源于针对`metric`报告的常见优化，该优化是将`metric`数据收集配置为具有相对较小的时间段（例如1秒），并使用单个时间戳来描述一批导出的数据，
因为所需要的精度是在跨数分钟或数小时的时间段内的数据汇总，所以无需在数据精度上进行担忧。

聚合通常通过一系列事件进行计算，这些事件落入连续的时间区间内，称为收集间隔。
由于SDK控制着开始收集的决定，因此可以收集汇总的指标数据，而每个收集间隔仅读取一次时钟。
默认的SDK就是使用这种方法。

同步仪器产生的`Metric events`会在瞬间发生，因此，它们属于同一个收集间隔，其中它们与来自同一`instrument`和标签集的其他事件聚合在一起。
因为`events`可能彼此同时发生，最近的事件没有在技术上有着很好的定义。

异步`instruments`允许SDK通过每个收集间隔进行一次观察来评估`metric instruments`。
由于这种与集合的耦合(与同步`instruments`不同)，这些`instruments`明确地定义了最近的事件。
我们将会把对某个时刻的`instruments`和标签集的“最后一个值”定义为最近一次收集间隔内测得的值。

由于`metric events`带有隐式时间戳，因此我们可以将一系列`metric events`称为`时间序列`。
但是，我们保留使用此术语作为SDK规范，以指代在序列中显式表示有时间戳的值的一种数据格式，
这是由于原始测量值随时间推移而产生的。

### Metric Event格式

`Metric events`具有相同的数据表现形式，无论`instrument`是那种类型。
通过任何工具捕获的`Metric events`都具有以下属性：

- timestamp (隐式的时间戳)
- instrument definition (`instrument`的定义，包括名称/种类/描述/计量单位)
- label set (标签集合，keys和values)
- value (有符号整数或浮点数)
- [资源](../resource/sdk.md) （启动时与SDK相关的资源）.

同步的`events`还有一个附加属性，即当时处于活动状态的分布式
[上下文](../context/context.md) (包括 `Span`，`Baggage`等等)。

## Meter provider

`MeterProvider`通过初始化和配置`OpenTelemetry Metrics SDK`，可以获得具体的实现。
本文档未指定如何构建SDK，仅说明它们必须实现 `MeterProvider`。
配置完成后，应用程序或库将选择是使用`MeterProvider` 接口的全局实例，还是选择使用依赖项注入来更好地控制配置提供程序。

### 获取一个Meter

`Meter`可以通过`MeterProvider`和 `GetMeter(name, version)`方法创建新实例。 `MeterProvider`通常被期望作为单例来使用。
实现应提供单个全局默认值`MeterProvider`。该`GetMeter`方法需要两个字符串参数：

- name（必填）：此名称必须标识`instrumentation library`（例如`io.opentelemetry.contrib.mongodb`），而不标识`instrumented library`。
  如果指定了无效的名称（`null`或空字符串`""`），Meter则将可用的默认实现作为后备返回，而不是返回null或引发异常。
  如果应用程序所有者配置SDK来废弃这个库的`telemetry produced`，一个“MeterProvider”也可以返回一个无操作的“Meter”，

- version（选填）：指定检测库的版本（例如1.0.0）。

每个单独命名的“Meter”都为其`metric instruments`建立了一个单独的命名空间，
这使得多个`instrument`库可以用其他库使用的相同`instrument`名称报告`metrics`。

`Meter`的名称明显并不是`instrument`名称的一部分，因为这将阻止`instrumentation`库捕获同名的`metrics`。

### 全局 Meter provider

在许多情况下，全局实例的使用可能被视为一种反模式（anti-pattern），但是在大多数情况下，它是`telemetry`数据的正确形式，
以便将相互依赖的库中的`telemetry`数据组合在一起而不使用依赖注入，因此，`penTelemetry`语言API应该提供一个全局实例。
一种语言在提供全局实例时，必须保证通过这些`Meter`实例分配的全局的`MeterProvider`和`instruments`，
并且通过这些`Meter`实例分配的`instruments`的初始化过程延迟到全局SDK的第一次初始化。

#### 获取全局的MeterProvider

由于全局`MeterProvider`变量是单例并且支持单个方法，因此调用者可以`Meter`使用全局`GetMeter` 调用获取全局变量。
例如， 在全局`global.GetMeter(name, version)`调用全局`MeterProvider`上的`GetMeter`方法，并返回一个命名的`Meter`的实例。

#### 设置全局的MeterProvider

全局函数将MeterProvider安装为全局SDK。
例如，用`global.SetMeterProvider(MeterProvider)`在初始化SDK后安装它。

## Instrument属性

因为API与SDK是分开的，所以实现的逻辑最终决定了如何处理`metric events`的事件。
因此`instrument`的选择应该以语义和预设实现（原文：the individual instruments）作为指导。
各个`instrument`的语义是由几个属性定义的，此处有详细的介绍，以帮助选择`instrument`。

### Instrument命名要求

`Metric instruments`主要由其名称定义，这是我们在外部系统中对它们的称呼方式。`Metric instruments`的命名应符合以下语法：
1. 它们不是空字符串
2. 它们不区分大小写
3. 第一个字符必须是非数字，非空格，非标点符号
4. 后续字符必须属于字母数字字符`_`, `.`和`-`。

`Metric instruments`的名称属于一个命名空间`namespace`，由关联的“Meter”实例建立。
当多个`instruments`以相同的名称注册时，`Meter`的实现必须要返回一个错误。

TODO: [关于`metric`命名的更规范的指导](https://github.com/open-telemetry/opentelemetry-specification/issues/600)

`Metric instrument`的名称在语义上应该是有意义的，独立于最初的`Meter`的名称。
例如，在检测http服务器库时，"latency"就不是一个合适的`instrument`名称，因为这个名称太笼统了。
作为示例，我们应该喜欢使用"http_request_latency"之类的名称，因为它会告知查看者延迟测量的语义。
可以编写多个工具库来生成此`metric`。

### 同步和异步instruments比较

同步`instruments`在请求内被调用，这意味着它们具有关联的分布式
[上下文](../context/context.md)
（包括`Span`，`Bagage`等）。
在给定的收集间隔内，同步`instruments`可能会发生多个`metric events`。

异步`instruments`由回调来进行上报收集，每个收集间隔上报一次，并且缺少上下文信息。每个时期每个标签组只能报告一个值。
如果应用程序在同一回调中观察到同一标签集的多个值，则最后一个值是唯一保留的值。

为确保最后一个值的定义在异步工具中保持一致，与异步事件关联的时间戳，是固定到计算时间间隔结束时的时间戳。
所有的异步事件都在时间间隔的结束上打上时间戳，也就是它们成为仪器和标签集合对应的最后一个值的那一刻。
（由于这个原因，SDK应该在收集间隔的末尾运行异步工具回调。）

### Adding instruments与grouping instruments比较

`Adding instruments`用于获取有关总和的数据，根据其定义，只有总和才具有意义。
单个的事件，对于这些`instruments`是没有意义的，也没有对于事件的计数。
这也就意味着，两个`Counter`事件 `Add(N)`和`Add(M)`等效于一个`Counter`事件`Add(N + M)`。
之所以会出现这种情况，是因为`Counter`是同步的，而`Adding instruments`是用来捕获变化到一个总和。

异步的`adding instruments`（例如`SumObserver`）用于直接捕获总和。
例如，这意味着在`SumObserver`给定`instrument`和标签集的任何观察序列中，最后一个值定义了`instrument`的总和。

在同步和异步的情况下，在不丢失信息的情况下，`adding instruments`以低廉的成本在每个收集间隔内收集的数据聚合成一个单独的数字。
这个特性使添加工具的性能比分组工具的性能更高

`Grouping instruments`在默认情况下，使用了与记录完成数据相比更简单的聚合工具，
但是仍然比`adding instruments`的默认的默认值消耗更多的性能（Sum）。
与添加工具不同，在`adding instruments`中，只需要定义定义的总和即可，而`Grouping instruments`可以配置甚至更昂贵的聚合器。

### 单调和非单调instruments的比较

单调性仅适用于`adding instruments`。 `Counter`和 `SumObserver`工具被定义为单调的，因为任一工具捕获的总和都不会减少。
`UpDown-` 这两种工具的变化是非单调的，这意味着总和可以增加，减少或保持恒定。

单调性的`instruments`用于捕获有关要作为速率监视的总和的信息。
此API将`Monotonic`（单调）属性定义为表示不减总和。不增加总和不被视为`Metric API`中的功能。

### 方法名称

每个`instrument`都支持一个方法，命名该功能有助于传达该`instrument`的语义。

同步`adding instruments`支持一项`Add()`方法，表示它们加到总和上而不直接捕获总和。

同步`Synchronous grouping`支持一项`Record()`方法，表示它们捕获单个事件，而不仅仅是一个和。

异步`instruments`均支持一项`Observe()`方法，这表明它们在每个测量间隔内仅捕获一个值。

## Instrument说明

### Counter

`Counter` 是最常见的同步`instrument`。该`instrument`支持`Add(increment)`报告总和的功能，并且仅限于非负增量。
对于任何的`adding instrument`，默认聚合为`Sum`。

`Counter`使用示例:

- 统计收到的字节数
- 统计已完成的请求数
- 统计创建的帐户数
- 统计运行的检查点数量
- 统计5xx错误的数量

上述的这些示例的`instruments`对于监测这些数量的速率是非常有用的。
在这些情况下，通常会比较方便地报告和发生的变化量，而不是计算和报告每次测量的和。

### UpDownCounter

`UpDownCounter`与`Counter`相似，`Counter`除了`Add(increment)`支持负增量。
这`UpDownCounter`对于计算速率聚合没有用。它汇总一个Sum，只有总和数据，是非单调的。
通常，对于捕获所用资源量或请求期间上升和下降的任何量的变化很有用。

示例用于`UpDownCounter`：

- 计算活动请求的数量
- 通过仪器计数使用中的内存，new并delete
- 通过检测enqueue和计算队列大小dequeue
- 计算信号量up和down操作。

上述的这些示例的`instruments`对于监视一组进程中的资源级别将很有用。

### ValueRecorder

`ValueRecorder`是一个同步的`grouping instrument`，用于记录一些正数或者负数的分组编号。
`Record(value)`方法捕获的值，被视为属于正在汇总的分布的单个事件。
如果数据的总和没有意义，或者数据的分布是具有意义的，则应该使用`ValueRecorder`。

`ValueRecorder`的常见用法之一是用于捕获延迟数据，因为延迟的测量数据不需要了解延迟的总和数据。
我们通常使用`ValueRecorder`来捕获延迟的测量值，单个事件的平均值，中位数和其他摘要信息是具有意义的。

`ValueRecorder`的默认聚合计算最小值和最大值、事件值和事件总数，
从而可以监视输入值的速率，平均值和范围。

用于`ValueRecorder`分组的示例用法：

- 捕获任何类型的时间信息
- 记录飞行员的加速度
- 捕获喷油器的喷嘴压力
- 捕获MIDI按键的速度

例如在`adding`使用`ValueRecorder`的捕获度量数据的示例，这些数据正在相加，
但是我们不仅对值的总和感兴趣，可能对值的分布也比较感兴趣：

- 捕获请求大小
- 取得帐户余额
- 捕获队列长度
- 捕获一些木材板的大小

上述这些例子表明，尽管它们本质上是相加的，但选择`ValueRecorder`而不是使用`Counter`或`UpDownCounter`，
暗示着我们不仅对于总和的统计存在兴趣。如果你对收集这些属性不感兴趣，则可以选择一种`adding instruments`，
使用`ValueRecorder`的意义在于对数据的分布进行分析。

请谨慎的使用`ValueRecorder`，因为它会消耗比`adding instruments`更多的性能。

### SumObserver

`SumObserver`是对应的异步的`instrument``Counter`，用于捕获`Observe(sum)`单调求和数据。
名称中会出现“总和”，以提醒用户直接用于捕获总和。
使用`SumObserver`捕获任何从零开始并在整个过程生命周期中上升并且永不下降的值。

`SumObserver`的示例用法.

- 捕获进程用户/系统CPU秒
- 捕获高速缓存未命中的数量

`SumObserver`度量的计算成本很高，因此对每个请求都进行计算是一种浪费。
例如，需要一个系统调用来捕获进程的CPU占用率，因此，它应该定期执行，而不是针对每个请求。
在不实用或浪费的`instrument`的个别变化，构成一个总和时`SumObserver`也时一个不错的选择。
例如，即使缓存未命中的数量是单个缓存未命中事件的总和，使用`Counter`同步捕获每个事件的代价太大了。

### UpDownSumObserver

`UpDownSumObserver`是对应的异步工具`UpDownCounter`，
用于捕获的非单调计数 `Observe(sum)`。名称中会出现`sum`，以提醒用户直接用于捕获总和。
使用`UpDownSumObserver`捕获在整个过程生命周期中从零开始并且上升或下降的任何值。

的示例用法UpDownSumObserver：

- 捕获进程堆大小
- 捕获活动碎片的数量
- 捕获已启动/已完成的请求数
- 捕获当前队列大小。

关于选择`SumObserver`还是`Counter`的考虑，也同样适用于在选择`UpDownSumObserver`还是`UpDownCounter`的场景。
如果度量值的计算成本很高，或者相应的更改发生得太频繁以至于无法进行测量，请使用`UpDownSumObserver`。

### ValueObserver
ValueObserver是ValueRecorder与之对应的异步工具，使用`Observe(value)`来捕获分组测量值。
这些`instruments`对于捕获计算成本很高的测量数据特别有用，因为使用SDK可以控制它们数据采集的频率。

`ValueObserver`的示例用法:

- 捕获CPU风扇速度
- 捕获CPU温度。

注意，这些例子使用了分组度量。在上面的ValueRecorder案例中，给出了在请求期间捕获同步`adding instrument`的示例用法（获取当前的队列大小）。
但是，在异步情况下，用户应如何决定是否使用`ValueObserver`而不是`UpDownSumObserver`？
（此处翻译可能存在问题，下方为原文）
_Note that these examples use grouping measurements.  In the
`ValueRecorder` case above, example uses were given for capturing
synchronous adding measurements during a request (e.g.,
current queue size seen by a request).  In the asynchronous case,
however, how should users decide whether to use `ValueObserver` as
opposed to `UpDownSumObserver`?_

考虑如何异步报告队列的大小。无论`ValueObserver`和`UpDownSumObserver`的逻辑都适用于这种情况。
异步仪器每个时间间隔仅捕获一次测量，因此在此示例中，UpDownSumObserver报告一个当前统计值，
`ValueObserver`报告一个当前的统计值（等于最大或最小值）和一个count = 1。
当没有聚合时，这些结果是相等的。

当只有一个数据点时，定义默认聚合似乎没有意义。在执行空间聚合时指定应用默认聚合，这意味着可以组合跨标签集或分布式设置的测量值。
尽管在`ValueObserver`每个收集间隔中仅观察一个值，但是默认聚合指定了如何在没有其他任何配置的情况下将其与其他值聚合。

因此，考虑到`ValueObserver`和`UpDownSumObserver`之间的选择，建议选择具有更合适的默认汇总的工具。
如果您正在观察一组机器的队列大小，而您唯一想知道的就是聚合队列大小，使用`SumObserver`是因为它产生的是总和数据，而不是分布数据。

如果您正在观察一组机器上的队列大小，并且您想知道这些机器上队列大小的分布情况，使用`ValueObserver`。

### 解释

这些工具有根本的不同，为什么只有三个？为什么不是一个`instrument`？为什么不十个？

正如我们所看到的，这些工具根据它们是否同步，相加和/或单调而分类。
该方法为提高`metric events`的性能和解释方式，为每种工具提供了独特的语义。

建立不同种类的`instrument`很重要，因为在大多数情况下，它允许SDK“开箱即用”提供良好的默认功能，而无需配置其他行为。
`instrument`的选择不仅决定事件的含义，还决定用户调用的功能的名称。
函数名称`Add()`用于添加工具，`Record()`用于`grouping instruments`和`Observe()`用于异步工具，
不同名称帮助传达这些动作的含义。

下表概述了每种仪器的特性和标准实施。

| **名称** | Instrument种类 | 方法(参数) | 默认聚合 | 备注 |
| --------------------- | ----- | --------- | ------------- | --- |
| **Counter**           | 同步 单调 adding  | Add(increment) | Sum | Per-request, part of a monotonic sum |
| **UpDownCounter**     | 同步 adding      | Add(increment) | Sum | Per-request, part of a non-monotonic sum |
| **ValueRecorder**     | 同步             | Record(value)  | [TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636)  | Per-request, any grouping measurement |
| **SumObserver**       | 异步 单调 adding  | Observe(sum)   | Sum | Per-interval, reporting a monotonic sum |
| **UpDownSumObserver** | 异步 adding      | Observe(sum)   | Sum  | Per-interval, reporting a non-monotonic sum |
| **ValueObserver**     | 异步             | Observe(value) | LastValue | Per-interval, any grouping measurement |

### 构造函数

该`Meter`接口支持创建新的注册`metric instruments`的功能。
`instruments`构造函数的命名方式是在它构造的类型、构造器模式或语言中的其他惯用方法上添加`New`前缀

在本规范中每种`instrument`都会有至少一种对应的构造器， (查看 [上文Metric Instruments](#metric-instruments))
并且可能由于语言的种类而规定更多的构造器。例如，如果提供了整型和浮点型指针数的专用处理，
OpenTelemetry API将支持每种`instrument`类型的2个构造函数。

将`instrument`绑定到单个`Meter`实例有两个好处::

1. `instrument`可以从零状态导出，在第一次使用之前，没有明确的注册调用
2. 库的名称和版本与`metric event`隐式关联

一些现有的`metric`系统支持支持静态分配`metric instruments`，并在使用时提供等效的`Meter`接口。
在一个示例中，典型的statsd客户端，现有的代码可能没有一个方便的地方来存储新的度量工具。
如果这成为一种负担，建议使用全局`MeterProvider`来构建静态`Meter`，并构建和使用全局范围的度量工具。

`Prometheus`现有客户端的用户也面临着类似的情况，他们可以将`instruments`分配给全局注册器。
这样的代码可能无法访问定义`instruments`时使用的那个适当的`MeterProvider`或`Meter`实例。
如果这成为一种负担，建议使用全局`MeterProvider`来构建静态`Meter`，并构建和使用全局范围的度量工具。

预期中，应用程序将会构造一个长生命周期的`instruments`。

`Instruments`在SDK的生命周期内被认为是永久的，没有办法删除它们。

## 标签集合

从语义上讲，一组标签是从字符串键到值的唯一映射。在整个API中，必须以相同的数据格式传递一组标签。
常见的表示形式包括键: 包含key与values属性的对象列表，或Map结构的key与value的映射。

如果将标签作为key：values的有序列表传递，并且存在重复的键，则将获取给定键的列表中的最后一个值，以形成唯一的映射。

标签值的类型通常被导出工具假定为字符串，尽管作为编程语言级别的决定，
标签值类型可以是该语言中用于表示字符串的任何常用类型。

用户不需要预先声明将与API中的`metric instruments`一起使用的一组标签键。
调用API时，用户可以针对任何`metric event`自由使用任何标签集。

### 标签性能

在整个`metric`数据的生产中，标签处理可能会产生巨大的成本。

SDK对进程内聚合的支持取决于找到仪器、标签集组合活动记录的能力。这样可以将测量组合在一起。
通过使用绑定的同步`instruments`和批处理方法(`RecordBatch`, `BatchObserver`)，可以降低标签处理成本。

### Option: Ordered labels

作为编程语言级别的决定，API可以支持标签键排序。
在这种情况下，用户可以指定标签键的有序序列，该序列是用来从一个类似有序的标签值序列中创建一个无序的标签集的。

```golang

var rpcLabelKeys = OrderedLabelKeys("a", "b", "c")

for _, input := range stream {
    labels := rpcLabelKeys.Values(1, 2, 3)  // a=1, b=2, c=3

    // ...
}
```

之所以将其指定为语言可选功能，是由于其安全性和因此作为监视输入的值会受到源语言中类型的可用性检查。
传递无序标签（即，从键到值的映射）被认为是更安全的选择。

## 同步instrument详情

为同步`instruments`指定了以下详细信息。

### 同步调用约定

`metrics`API提供了三种语义上等效的方式来使用同步`instruments`捕获测量值：

- 绑定调用`instruments`，它们具有一组预先关联的标签
- 直接调用`instruments`，传递相关的标签集
- 使用一组标签对多个`instruments`进行批量记录测量。

这三种方法均会生成等效的`metric events`，但会提供不同程度的性能和便利性。

`metric`的API性能取决于输入新度量标准所完成的工作，通常由处理标签的成本决定。
调用绑定的`instruments`是性能最高的调用约定，因为它们可以分摊在许多用途中处理标签的成本。
通过另一种调用约定‘RecordBatch()’记录多个度量，这是提高性能的一个很好的选择，因为处理标签的成本分散在多个度量中。
直接调用约定是最方便的，但通过API输入度量值是性能最差的调用约定。

#### 绑定调用instrument约定

在需要性能的情况下，并且使用同一套标签重复使用`metric instrument`的情况下，开发人员可以选择使用绑定调用`instruments`约定作为优化。
为了使绑定的`instrument`受益，它要求特定的`instrument`将被重新使用并带有特定的标签。
如果一个`instrument`将多次使用相同的标签，则获得与这些标签相对应的绑定`instrument`可确保获得最高的性能。

要绑定一个`instrument`，请使用`Bind(labels...)`方法返回支持相应同步API（即 `Add()`或`Record()`）的接口。
绑定的`instrument`在没有标签的情况下被调用；相应的`metric event`与绑定到`instrument`的标签相关联。

由于其性能优势，绑定的`instrument`也会消耗SDK中的资源。绑定的`instrument`必须支持一种`Unbind()`方法，以指示用户完成绑定并释放关联的资源。
请注意，`Unbind()`这并不意味着删除时间序列，它仅允许SDK在没有待处理的更新后忘记存在的时间序列。

例如，要重复更新具有相同标签的计数器：

```golang
func (s *server) processStream(ctx context.Context) {

  // The result of Bind() is a bound instrument
  // (e.g., a BoundInt64Counter).
  counter2 := s.instruments.counter2.Bind(
      kv.String("labelA", "..."),
      kv.String("labelB", "..."),
  )
  defer counter2.Unbind()

  for _, item := <-s.channel {
     // ... other work

     // High-performance metric calling convention: use of bound
     // instruments.
     counter2.Add(ctx, item.size())
  }
}
```

#### 直接调用instrument约定

当调用时的便利性比性能更重要，或者提前不知道值时，用户可以选择直接在`metric instruments`上进行操作，
这意味着在调用的时候提供标签。这种方法提供了最大的便利。

例如，要更新单个计数器：

```golang
func (s *server) method(ctx context.Context) {
    // ... other work

    s.instruments.counter1.Add(ctx, 1,
        kv.String("labelA", "..."),
        kv.String("labelB", "..."),
        )
}
```
直接调用很方便，因为它们不需要分配和存储绑定的`instrument`。
它们适用于很少使用`instrument`或很少使用同一组标签的情况。
与绑定`instrument`不同，使用直接调用约定时，不会长期消耗SDK资源。

#### RecordBatch调用约定

有一个用于输入测量值的最终API，就像直接访问调用约定一样，但支持多个同时测量。
`RecordBatch`API的使用支持同时输入多个测量值，这意味着对几种`instrument`进行了语义上的原子更新。
对`RecordBatch`的调用可以跨多个度量摊销标签处理的成本。

例如:

```golang
func (s *server) method(ctx context.Context) {
    // ... other work

    s.meter.RecordBatch(ctx, labels,
        s.instruments.counter.Measurement(1),
        s.instruments.updowncounter.Measurement(10),
        s.instruments.valuerecorder.Measurement(123.45),
    )
}
```

另一个用于批量记录的有效接口使用了一个构建器模式：

```java
    meter.RecordBatch(labels).
        put(s.instruments.counter, 1).
        put(s.instruments.updowncounter, 10).
        put(s.instruments.valuerecorder, 123.45).
        record();
```

使用批量处理调用约定在语义上与一系列直接调用相同，只是增加了原子性。
因为值是在单个调用中输入的，从导出程序的角度来看，SDK有可能实现原子更新，因为SDK可以对单个批量更新进行排队，或者仅进行一次锁定。
像直接调用约定一样，使用批处理调用约定时也不会长期消耗SDK资源。

### 与分布式上下文关联

同步`instruments`在运行时与分布式
[上下文](../context/context.md)
存在着隐式关联，它可能包括一个`Span`和`Baggage entries`。
Metric SDK可以以多种方式使用这些信息，但OpenTelemetry中有一个有趣的特性（这里原文中未继续描述这个有趣的特性是什么）。

#### Baggage中的metric标签

`Baggage`在`OpenTelemetry`中得到支持，它是一种让标签在分布式计算中从一个进程传播到另一个进程的方法。
有时，使用分布式`baggage entries`作为`metric`标签来聚合`metric`数据是有用的。

必须显式地配置`Baggage`的使用，使用[Views API (WIP)](https://github.com/open-telemetry/oteps/pull/89)
来选择应作为标签来使用的特定关键`baggage entries`。默认SDK不会在出口管道中自动使用`Baggage`标签，
因为使用`Baggage`标签可能会产生很大的消耗。

为应用`Baggage`标签配置视图是一项[正在进行的工作](https://github.com/open-telemetry/oteps/pull/89)。

## 异步instrument细节

以下描述信息仅限于异步的`instruments`。

### 异步调用约定

`metrics`的API提供了两种语义上等价的方法来使用异步`instruments`捕获度量数据，
或者通过单一`instruments`回调，或者通过多个`instruments`进行批回调。

无论是单个的还是批量的，异步`instruments`必须通过一个回调来观察。构造函数为`null``observer callbacks`返回无操作的`instruments`。
任何一个异步`instruments`被指定了多个回调，都会被认为是一个异常。

`Instruments`在每组不同的标签中，只能监测到一个值。
当单个`Instruments`和标号组监测到多个值时，只会取最后一个监测值，并无错误地丢弃较早的数据。

#### 单一instrument的监测

一个单独的回调被绑定到一个`instrument`上。它的回调接收一个`ObserverResult`和一个`Observe(value, labels…)`函数。

```golang
func (s *server) registerObservers(.Context) {
     s.observer1 = s.meter.NewInt64SumObserver(
         "service_load_factor",
          metric.WithCallback(func(result metric.Float64ObserverResult) {
             for _, listener := range s.listeners {
                 result.Observe(
                     s.loadFactor(),
                     kv.String("name", server.name),
                     kv.String("port", listener.port),
                 )
             }
          }),
          metric.WithDescription("The load factor use for load balancing purposes"),
    )
}

```

#### Batch observer

一个`BatchObserver`支持在一个回掉中监测多个`instruments`。
它的回调接收一个`BatchObserverResult`和一个`Observe(value, observations…)`函数。

在异步`instrument`上，通过调用`Observation(value)`返回监测结果。

```golang
func (s *server) registerObservers(.Context) {
     batch := s.meter.NewBatchObserver(func (result BatchObserverResult) {
          result.Observe(
             []kv.KeyValue{
                 kv.String("name", server.name),
                 kv.String("port", listener.port),
             },
             s.observer1.Observation(value1),
             s.observer2.Observation(value2),
             s.observer3.Observation(value3),
          },
    )

     s.observer1 = batch.NewSumObserver(...)
     s.observer2 = batch.NewUpDownSumObserver(...)
     s.observer3 = batch.NewValueObserver(...)
}
```

### Asynchronous observations form a current set

异步仪器回调允许对每个`instrument`、每个不同的标签集、每个回调调用观察一个值。
由一次回调调用记录的一组值代表`instrument`的当前快照；这一组值定义了`instrument`直到下一个收集间隔的最后一个值。

异步`instrument`应记录每一个它认为是当前的标签集的监测结果。这意味着异步回调应该持续监测一个值，
即使这个值自上次回调调用以来没有改变。不监测标签集意味着某个值不再是当前值。
当最后一个值在一个收集时间间隔内没有被监测到时，它就不再是当前值，因此变成`undefined`。

对于异步`instruments`来说，最后一个值的定义是可以存在的，因为它们是由SDK协调收集的，而且它们应该上报所有的当前测量值。
该属性的另一个说法是，SDK可以在内存中保存一个集测量间隔区间，以查找任何`instrument`和标签集的当前最后值。
通过这种方式，异步`instruments`支持使用单个时间点收集的数据查询当前值，而不依赖于采集间隔的持续时间。

回想一下，没有为同步`instruments`定义一个"最后一个值"的概念，这恰好是因为它没有一个明确的定义"当前"这个概念。
因为没有机制可以确保在每个间隔内都记录当前值，所以要确定同步`instruments`的"最后记录"值，就可能需要检查多个数据收集窗口，。

#### Asynchronous instruments define moment-in-time ratios

上面所描述的异步`instruments`的概念对于目前开发的检测率是用的。
当`instruments`的一组测量值加起来是一个整体，那么每一个测量值可以除以同一区间的测量值的总和，以计算其这次测量的结果的相对贡献比例。
目前相对贡献是这样定义的，多亏了异步`instruments`的特性做到了与收集间隔时间无关。

## 并发

对于支持并发执行的语言，Metrics API提供了一些具体的保障和安全性。但是并非所有的API函数都可以安全地被并发调用。

**MeterProvider** - 所有方法都可以安全地被并发调用。

**Meter** - 所有方法都可以安全地被并发调用。

**Instrument** - 任何`Instrument`的所有方法都可以安全地同时调用。

**Bound Instrument** - 任何`Bound Instrument`的所有方法都可以安全地同时调用。

## 相关的OpenTelemetry工作

在编写本规范时，正在进行一些持续的努力。

### Metric视图

该API不支持`metric instruments`的可配置聚合。

`View API`定义为SDK方式的接口，该方式支持配置聚合。
包括应用哪个操作符（sum, p99, last-value, 等）和使用哪个维度。

可以查看[当前有关这个话题的讨论](https://github.com/open-telemetry/opentelemetry-specification/issues/466) 
和[当前的OTEP草案](https://github.com/open-telemetry/oteps/pull/89)。

### OTLP Metric协议

如上所述，OTLP协议旨在以无侵入的方式导出`metric`数据。该协议的几个细节正在制定中。请参阅
[当前协议](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto)。

### Metric SDK default implementation

OpenTelemetry SDK包含对`metric API`的默认支持。默认SDK的规范正在进行中，请参阅
[current draft](https://github.com/open-telemetry/opentelemetry-specification/pull/347).
