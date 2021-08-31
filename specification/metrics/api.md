# Metrics API

<!-- toc -->

- [总览](#总览)
  * [在没有安装SDK情况下的API行为](#在没有安装SDK情况下的API行为)
  * [数据测量](#数据测量)
  * [Metric Instruments](#metric-instruments)
  * [标签](#标签)
  * [Meter Interface](#meter-interface)
  * [聚合](#聚合)
  * [时间](#时间)
  * [Metric Event格式](#Metric-Event格式)
- [Meter provider](#获取一个-MeterProvider)
  * [获取一个Meter](#获取一个-Meter)
  * [全局 MeterProvider](#全局-MeterProvider)
    + [获取全局的MeterProvider](#获取全局的-MeterProvider)
    + [设置全局的MeterProvider](#设置全局的-MeterProvider)
- [Instrument属性](#Instrument-属性)
  * [Instrument命名要求](#Instrument-命名要求)
  * [同步和异步instruments比较](#同步和异步-instruments-比较)
  * [Adding instruments与grouping instruments比较](#Adding-instruments-与-grouping-instruments-比较)
  * [单调和非单调instruments的比较](#单调和非单调-instruments-的比较)
  * [函数名称](#函数名称)
- [Instrument说明](#Instrument-说明)
  * [Counter](#Counter)
  * [UpDownCounter](#UpDownCounter)
  * [ValueRecorder](#ValueRecorder)
  * [SumObserver](#SumObserver)
  * [UpDownSumObserver](#UpDownSumObserver)
  * [ValueObserver](#ValueObserver)
  * [解释](#解释)
  * [构造函数](#构造函数)
- [标签集合](#标签集合)
  * [标签性能](#标签性能)
  * [可以选择的能力：经过排序的标签](#可以选择的能力：经过排序的标签)
- [同步instrument详情](#同步-instrument-详情)
  * [同步调用约定](#同步调用约定)
    + [绑定调用instrument约定](#绑定-instrument-调用约定)
    + [直接调用instrument约定](#直接-instrument-调用约定)
    + [RecordBatch调用约定](#RecordBatch-调用约定)
  * [与分布式上下文关联](#与分布式上下文关联)
    + [Baggage 作为 metric 的标签](#Baggage-作为-metric-的标签)
- [异步instrument细节](#异步-instrument-细节)
  * [异步调用约定](#异步调用约定)
    + [单一instrument的监测](#单一-instrument-的监测)
    + [Batch observer](#batch-observer)
  * [异步观察构成一个当前的值集](#异步观察构成一个当前的值集)
    + [Asynchronous instruments define moment-in-time ratios](#asynchronous-instruments-define-moment-in-time-ratios)
- [并发](#并发)
- [Related OpenTelemetry work](#相关的-OpenTelemetry-工作)
  * [Metric 视图](#metric-视图)
  * [OTLP Metric 协议](#OTLP-Metric-协议)
  * [Metric SDK 默认实现](#Metric-SDK-默认实现)

<!-- tocstop -->

## 总览
`OpenTelemetry Metrics API`（`OpenTelemetry`的指标`API`）支持获取计算机程序在运行时，有关执行情况的原始的指标测量数据。
这个`API`是专门为处理原始的测量指标数据采集而设计的，
`Metrics API`是专门为处理原始测量数据而设计，通常是为了采用并行的方式，高效的对这些测量结果进行连续地统计与记录。
在下文中，`API`是指`OpenTelemetry Metrics API`。

`API`通过提供基于不同性能级别的调用函数来采集原始测量指标数据。
无论什么样的调用约束, 我们都将`metric event`定义为指标被捕获时发生的逻辑事件。
当指标被获取到时，我们也在指标中定义了一个隐方的时间戳，这个时间戳是SDK从系统时钟中读取到的当前时间的时间戳。【该时间戳无法进行手动调整】。

在本文中使用`semantic`或者`semantics`或者`语义`来指代我们如何为指标事件来赋予他所应有的含义。
`语义`在本文档中得到广泛使用，用以定义和解释这些API函数。
在定义函数名称时，需要尽可能地贴合其原本所指代的含义。

监控与告警系统，一般会使用`指标事件`所提供的`指标`数据，这些数据经过[聚合](#聚合)并转换为各种不同展示格式的数据。
然而，我们还发现`指标事件`的数据还有许多其他用途，例如在链路和日志系统中记录聚合的或原始`指标数据`。
因此，
[OpenTelemetry需要将API与SDK分开](../library-guidelines.md#requirements)
以便可以在运行时配置不同的SDK。

### 在没有安装SDK情况下的API行为

在没有安装`Metrics SDK`（指标的SDK）的情况下，指标的`API`只可以由`无操作`的输出来构成，
任何的指标`API`的调用都将没有任何的实际意义，采集到的数据都会被直接丢弃。
在任何的`指标`的`API`都必须要默认返回一个由无操作实现构成的`instruments`（采集器）。从用户的角度来说，应该在引发任何错误的情况下，忽略对他们的调用。
例如：不要因为一个`API`返回了`null`引用时，触发异常。API在被调用时，不应该触发任何的异常。

### 数据测量

本文中使用术语`capture`（捕获）来描述用户将一份指标的测量数据传递给`API`时所执行的操作。
`capture`（捕获）的结果取决于所配置的SDK，如果SDK没有被安装，则默认操作是不会执行捕获的事件。
这种用法的目的是，根据不同的SDK来传达任何可能会发生的测量结果，但是这也意味着用户已经为了获取这一部分测量数据付出了成本。
出于性能和语义（semantic）方面的考虑，API允许用户在这两种测量方式之间进行选择。

术语`adding`（求和）用于指定某些测量的特征，在该种测量下，只有总和才会被认为是有意义的信息。
这些是可以使用算术加法进行组合的测量数据（通常是某些事物的实际数量，例如网络传输的IO字节数，可以是一个始终持续增长的指标）。

分组测量工具，在一组值的集合自身具有一定的意义时进行使用。
当一个测量指标，在进行累加后不具备含义，或者当你对于值的分布更加感兴趣时，需要采取的方案。（例如网络的延迟情况，某个队列当前的小）

分组工具在功能上比求和工具能够捕获更多的信息。根据其定义，分组测量数据比求和测量数据会消耗更高的性能。
用户应该优先选择求和工具，除非他们希望从有关单个数据的额外信息中获取到其他价值。
当然这些都不能阻止SDK根据配置重新组织与解释原始数据。原始数据可以被随意加工处理。

### Metric Instruments

`metric instrument`（指标的测量工具）是用于在API中捕获原始指标值的组件。
下表中列出的标准的`instrument`，每一种都有特定的用途，在API中刻意避免了改变`instrument`的可选特性; 
相反，API更喜欢支持单一方法的`instrument`，每个方法都有固定的实现。

API捕获的所有度量数据都与用于进行度量的`instrument`相关联，
因此赋予了度量其语义属性。`instrument`是通过调用`Meter`API来创建和定义的，API是SDK面向用户的入口点。

`instrument`通过以下的几种方式进行区分：
1.同步性：同步`instrument`由用户在处于分布式
  [上下文](../context/context.md)
  下的情况调用（即，具有关联的`Span`，`Baggage`等）。在不具备上下文的情况下，SDK会在每一个收集间隔中，调用异步工具，从`instrument`的缓存中去获取指标数据。
2.`求和`与`分组`：求和工具是一种记录添加测量值的工具，与上述的分组工具相反。
3.单调性：`monotonic instrument`是一种`adding instrument`，但是其值永远不会下降。`monotonic instrument`对于监视速率信息很有用。

指标测量工具的名称以及它们是否`synchronous`，`adding`或`monotonic`的情况显示在下面表格中。

| 名称 | Synchronous | Adding | Monotonic |
| ---- | ----------- | -------- | --------- |
| Counter           | 是 | 是 | 是 |
| UpDownCounter     | 是 | 是 | 否 |
| ValueRecorder     | 是 | 否 | 否 |
| SumObserver       | 否 | 是 | 是 |
| UpDownSumObserver | 否 | 是 | 否 |
| ValueObserver     | 否 | 否 | 否 |

`synchronous instruments`（异步测量工具）对于处于分布式
[上下文](../context/context.md)
中（即，具有关联的`span`，`Baggage`等）测量的指标很有用。
当指标的测量成本很高时，`asynchronous instruments`很有用，因此应定期收集。在此处阅读更多：
[同步和异步instruments的特性](#同步和异步instruments比较)

同步和异步加法指标测量工具`adding instruments`有很大的区别：同步指标测量工具用于捕获总和的变化，而异步工具用于直接捕获总和。在下面阅读
[`adding instruments`的特征](#Adding-instruments与grouping-instruments比较)。

`monotonic adding instrument`是重要的工具，因为他们支持速率计算。阅读
[选择`metric instruments`](#单调和非单调instruments的比较)
以获取更多的信息。

`instrument`（测量工具）的定义，描述了这个`instrument`的一些特性，包括它的类型与名称。
`metric instrument`（指标测量工具）中的一些属性是可以进行配置的，包括其名称、说明信息以及测量的单位。
一个`instrument`需要如何定义，取决于其产生的数据的也行。

### 标签

标签是用于指代与`metric event`关联的Key-Value属性的术语，类似于`Tracing API`中的
[Span attribute](../trace/api.md#span)。
每个标签可以对`metric event`进行分类，从而可以对其进行过滤和分组以进行分析。

每一个`instrument`的调用接口中，都可以填写标签信息。标签的集合定义为从Key到Value的唯一映射信息。
通常，标签以键值对列表的形式传递给API，在这种情况下，当key发生重复的时候以列表中的最后一个同名key为准。

同步`instrument`的测量数据通常可以与其他具有完全相同 label 集合数据进行组合来达到一定的性能优化的目的。
相关的内容可以阅读[通过聚合来组合度量](#聚合)

### Meter Interface

API定义了一个Meter接口。该接口由一组`instrument`构造器，和一个使用原子方式批量获取测量值的工具组成。

使用时建议创建一个全局Meter实例可供使用，以便在第三方代码的自动适配时使用。
使用这个实例，可以使用静态的方式来对`metric instruments`实例进行初始化，而无需要显示的依赖注入。

全局`Meter`实例在应用程序中服务提供者或者特定语言所支持的SDK被成功初始化之前将会不执行任何逻辑。
请注意，不要直接去使用全局实例：OpenTelemetry SDK的多个实例可能同时运行。

在使用时，API要求调用者在获得`Meter`实现时提供`instrumenting`库的名称（可选，版本）。
库名称旨在用于标识从该库产生的检测，例如禁用检测，配置聚合和应用采样策略。有关更多详细信息，请参见
[TracerProvider](../trace/api.md#tracerprovider)
上的规范。

### 聚合

聚合是指将多个维度的度量，合并为程序执行期间的时间间隔内发生的具体或者预估的测量数据的统计信息的过程。
每个`instrument`都指定一个适合该工具语义的默认聚合，用于解释其属性并使用户了解如何使用它。
在没有配置其他替代方案的情况下，`instrument`可以提供开箱即用的简单汇总。

`adding instruments`（加法测量工具，`Counter`, `UpDownCounter`, `SumObserver`,`UpDownSumObserver`)在默认的情况下使用的是求和聚合
用于计算聚合数据的详细信息有所不同，但是从用户的角度来说，这意味着他们将能够监测捕获的值的总和。
同步与异步的`instruments`之间的区别，对于指定 exporter 的工作方式至关重要。
这是
[SDK 规范 (WIP)](https://github.com/open-telemetry/opentelemetry-specification/pull/347)
涵盖的主题。

`ValueRecorder``instrument`采用
[TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636)
默认聚集。

`ValueObserver``instrument`默认使用最后一个采集到的指标值来进行聚合。此聚合跟踪所观察到的最后一个值和它的时间戳。

对于`grouping instruments`来说，其他聚合方式也是可行的，
我们普遍关心的各种不同的报表，例如直方图，分位数报表，基数估计和其他种类的数据结构。

默认的OpenTelemetry SDK实现了[Views API (WIP)](https://github.com/open-telemetry/oteps/pull/89)，
该API支持在单个`instrument`的级别上配置非默认的聚合行为。
即使OpenTelemetry SDK可以配置为以非标准方式处理`instruments`，但仍希望用户根据其语义来选择`instruments`，使用默认聚合进行处理。

### 时间

时间是 `Metric Event`（指标事件）的基本属性，但不是必须的属性。用户不需要为指标事件提供显式的时间戳。
不建议SDK在捕获每个事件时通过读取时钟来获取当前时间戳，除非明确需要每个 `Event` 都需要计算高精度的时间戳。

这是在上报 `metric` 的时候的常见优化，该优化是将 `metric` 数据收集间隔配置为相对较小的时间间隔（例如1秒），并使用单个时间戳来描述一批导出的数据，
因此在对所需要的精度是在分钟级别或者小时级别的时间段内的数据进行汇总的时候，精度损失很小。

聚合通常通过一系列事件进行计算，这些事件属于一个连续的时间区间，而这个时间区间称为收集间隔。
由于SDK控制着是否开始收集，因此可以收集聚合的指标数据的同时在每个收集间隔仅读取一次时钟。
默认的SDK就是使用这种方法。

`synchronous instruments`（同步测量工具）产生的`Metric events` 会在很短的时间内发生，因此，
它会和属于同一个收集间隔中来自同一`instrument`和标签集的其他事件组合在一起。
因为 `events` 可能在同一时间发生多次，所以并不能很好的定义最近的一次事件。

`asynchronous instruments`（异步指标采集工具）允许SDK通过每个收集间隔调用一次异步指标工具来进行数据采集。
由于这种数据采集方式，明确的定义了最近的指标采集事件，我们将会把某个时刻的指标与标签集的“最后一个值”定义为最近一次收集间隔内测得的值。

由于`metric events`带有隐式时间戳，因此我们可以将一系列`metric events`称为`时间序列`。
但是，我们保留使用此术语作为SDK规范，以指代在序列中显式表示有时间戳的值的一种数据格式，
这是由于原始测量值随时间推移而产生的。

### Metric Event格式

无论`instrument`（测量工具）是那种类型，`Metric events`（告警事件）都具有相同的逻辑形式。
通过任何工具捕获的`Metric events`都具有以下属性：

- timestamp (隐式的时间戳)
- instrument definition (测量工具的定义，包括名称、种类、描述、计量单位)
- label set (标签集合，keys和values)
- value (有符号整数或浮点数)
- [资源](../resource/sdk.md) （启动时与SDK相关的资源）.

同步的`events`还有一个附加属性，即当时所处于的有效的分布式
[上下文](../context/context.md) (包括 `Span`，`Baggage`等等)。

## 获取一个 MeterProvider

`MeterProvider` 通过初始化和配置 `OpenTelemetry Metrics SDK` 来获得具体实例。
本文档未指定如何构建SDK，仅说明它们必须实现 `MeterProvider`。
配置完成后，应用程序或库可以选择使用 `MeterProvider` 接口的全局实例的方式获取，也可以选择使用依赖项注入的方式来更好地根据配置进行控制。

### 获取一个 Meter

`Meter` 可以通过 `MeterProvider` 的 `GetMeter(name, version)`方法来创建一个新实例。 `MeterProvider` 通常被期望作为单例来使用。
其实现应作为`MeterProvider`全局的唯一实现。该 `GetMeter` 方法需要两个字符串类型的参数：

- name（必填）：这个名称应该作为集成库的名称（例如`io.opentelemetry.contrib.mongodb`），而不是被集成库的名称。
  如果指定了非法的名称（ `null` 或空字符串`""`），一个默认的 Meter 应该被返回，而不是返回 null 或引发异常。
  如果应用程序所有者配置SDK认为这个库已经过期而不应该产生任指标何观测数据，那么返回的 `MeterProvider` 应该不做任何操作。

- version（选填）：指定集成库的版本（例如 1.0.0）。

每个单独命名的 `Meter` 都为其 metric instrument 建立了一个单独的命名空间，
这使得某个 instrument 可以在不同的集成库使用的相同名名称上报 `metrics` 。
但是 `Meter` 的名称明显不会作为 instruments 的名称的一部分，因为这会导致集成库去捕获同名的 `metrics` 。

### 全局 MeterProvider

在许多情况下，全局实例的使用可能被视为一种反面模式，但是在大多数情况下，它却是观测数据采集的合理方案，
这种实现以便从一批相互依赖的库当中的获取观测数据，从而避免了依赖注入，因此，`openTelemetry` 所提供针对编程语言的 API 应该提供一个全局实例。
当在一种语言中提供全局实例时，必须保证通过这些 `Meter` 实例是全局的 `MeterProvider` 所分配的并且集成库所获取到的 `Meter` 实例的初始化将会
延迟到全局 SDK 的第一次初始化之后。

#### 获取全局的 MeterProvider

由于全局 `MeterProvider` 变量是单例并且支持通过单个方法获取，因此调用者可以通过全局的 `GetMeter` 方法调用来获取一个全局 
`Meter` 的变量。
例如， `global.GetMeter(name, version)` 通过调用全局 `MeterProvider` 的 `GetMeter` 方法并返回一个被命名的 `Meter` 的实例。

#### 设置全局的 MeterProvider

可以通过 SDK 的一个全局函数来设置 MeterProvider 。
例如，用 `global.SetMeterProvider(MeterProvider)` 可以在 SDK 初始化的过程中进行配置 MeterProvider 。

## Instrument 属性

因为 API 与 SDK 是分开的，所以其实现的逻辑最终决定了如何处理 `metric events` 的事件。
因此在选择 instrument 的过程中的选择应该以语义与需求作为指导。
各个 instrument 的语义是由几个属性定义的，此处有详细的介绍，以帮助选择合适的 instrument 。

### Instrument 命名要求

Metric instruments 主要由其名称定义，这是我们在外部系统中对它们的称呼方式。 Metric instruments 的命名应符合以下语法：
1. 它们不是空字符串
2. 它们不区分大小写
3. 第一个字符必须是非数字，非空格，非标点符号
4. 后续字符必须属于字母数字字符和`_`，`.`和`-`。

Metric instruments 的名称属于一个命名空间，这个命名空间由这关联的 `Meter` 实例建立。
当多个 `instruments` 以相同的名称注册时， `Meter` 的实现必须要返回一个错误。

TODO: [以下段落是为了后续更详细文档的占位符。](https://github.com/open-telemetry/opentelemetry-specification/issues/600)

Metric instrument 的名称在语义上应该是有意义的，独立于最初的 Meter 的名称。
例如，在检测 http 服务器库时，"latency" 就不是一个合适的 instrument 名称，因为这个名称太笼统了。
作为示例，我们应该喜欢使用 "http_request_latency" 之类的名称，因为它会直接展示给查看者其延迟数据的语义。
可以编写多个工具库来生成此`metric`。

### 同步和异步 instruments 比较

同步 `instruments` 在请求内被调用，这意味着它们处于在分布式
[上下文](../context/context.md)
（包括`Span`，`Bagage`等）之下。
在给定的间隔内，同步 `instruments` 可能会产生多个 `metric events` 。

异步 `instruments` 由回调来进行上报收集，每个收集间隔上报一次，并且缺少上下文信息。在每个时间间隔内每个标签组只能报告一个值。
如果应用程序在一次回调中观察到同一标签集的多个值，则最后一个值是唯一保留的值。

为确保最后一个值的定义在异步工具中保持一致，计算时间间隔结束时的时间戳将会固定作为异步事件关联的时间戳。
所有的异步事件都在时间间隔的结束上打上时间戳，同时这些 event 也成为了 instrument 下标签集合的最后一个值。
（由于这个原因， SDK 应该在收集间隔的末尾运行异步工具回调。）

### Adding instruments 与 grouping instruments 比较

Adding instruments 用于获取有关总和的数据，根据其定义只有数据总和才是值得感兴趣的结果。
个别的事件对于这些 instruments 是没有意义的，具体 event 数量的计算没有意义。
这也就意味着，两个 `Counter` 事件 `Add(N)` 和 `Add(M)` 等效于一个 `Counter` 事件 `Add(N + M)` 。
之所以会出现这种情况，是因为`Counter`是同步的，而`Adding instruments`是用来捕获变化到一个总和。

异步的 adding instruments（例如 `SumObserver` ）用于直接捕获总和。
这意味着在 `SumObserver` 给定 instrument 和标签集的任何观察序列中，最后一个值定义了`instrument`的总和。

在同步和异步的情况下，在不丢失信息的情况下，adding instruments 以低廉的成本在每个收集间隔内收集的数据聚合成一个单独的数字。
这个特性使 adding instrument 的性能比 grouping instruments 的性能更高。

Grouping instruments 在默认情况下，使用了与记录完整数据相比更简单的聚合工具，
但是仍然比 adding instruments 的默认的默认值消耗更多的性能（求和）。
在 adding instruments 中，只有总和是感兴趣的，而 grouping instruments 可以配置更消耗性能的聚合器。

### 单调和非单调 instruments 的比较

单调性仅适用于 `adding instruments` 。 `Counter` 和 `SumObserver` 工具被定义为单调的，因为任一工具捕获的总和都不会减少。
`UpDown-` 系列的两种工具的变化是非单调的，这意味着总和可以增加，减少或保持恒定。

单调性的 `instruments` 用于捕获有关要作为速率监视的总和的信息。
此API将 `Monotonic` （单调）属性定义为表示不减总和。不会增加的总和计数不被考虑为 `Metric API` 中的功能。

### 函数名称

每个 instrument 都支持一个函数，命名该功能有助于传达该 `instrument` 的语义。

同步 adding instruments 提供 `Add()` 方法来将目标数字加到总和上而不是直接捕获总和。

同步 grouping instruments 提供 `Record()` 方法，表示它们捕获单个独立事件，而不仅仅是一个数字和。

异步 instruments 提供 `Observe()` 函数，这表明它们在每个测量间隔内仅捕获一个值。

## Instrument 说明

### Counter

`Counter` 是最常见的同步 instrument 。该 instrument 支持通过 Add(increment) 函数报告总和的功能，并且仅限于非负增量。
对于任何的 adding instrument ，默认聚合为`Sum`。

`Counter`使用示例:

- 统计收到的字节数
- 统计已完成的请求数
- 统计创建的帐户数
- 统计运行的检查点数量
- 统计5xx错误的数量

上述的这些示例的 instruments 对于监测这些数量的速率是非常有用的。
在这些情况下，报告一个和的变化量通常比计算并报告每个测量数据的和更方便。

### UpDownCounter

`UpDownCounter` 与 `Counter` 相似， 但是相比 `Counter` 其的 `Add(increment)` 函数支持负增量。
这 `UpDownCounter` 对于计算速率聚合没有用。它汇总一个 `Sum` ，只有总和数据，是非单调的。
通常，这对于捕获所占用的资源量或请求期间上升和下降的数据的变化来说很有用。

示例用于 `UpDownCounter` ：

- 计算活跃请求的数量
- 通过对 `new` 和 `delete` 操作进行集成计算占用内存的数量
- 通过对 `enqueue` 和 `dequeue` 操作进行集成计算队列的长度
- 通过 `up` `down` 操作记录信号量

上述的这些示例的 instruments 对于监视一组进程中的资源级别的数据将很有用。

### ValueRecorder

`ValueRecorder` 是一个同步的 grouping instrument ，用于记录分组数据，不管是正数还是负数。
`Record(value)` 函数所捕获的值，被视为正在汇总的分布的单个事件。
如果数据的总和没有意义，或者数据的分布情况是具有意义的，则应该使用` ValueRecorder` 。

`ValueRecorder` 的常见用法之一是用于捕获延迟数据，因为对于每个正在执行的请求来说，它们的延迟之和是没有意义的。
我们通常使用 `ValueRecorder` 来捕获延迟的测量值的时候，我们可能会对单个事件的平均值，中位数和其他统计信息更感兴趣。

`ValueRecorder` 的默认聚合计算最小值和最大值、事件值和事件总数，
从而可以监视输入值的速率，平均值和范围。

用于 `ValueRecorder` 分组的示例用法：

- 捕获任何类型的时序信息
- 记录飞行员的加速度
- 捕获喷油器的喷嘴压力
- 捕获MIDI按键的速度

例如在 `adding` 工具中，这些数据会进行相加，但在有的时候，我们不仅对值的总和感兴趣，可能对值的分布也比较感兴趣。
在这时我们就可以使用 `ValueRecorder`来捕获所需要的数据。

- 捕获一个请求的大小
- 捕获一个帐户的余额
- 捕获一个队列的长度
- 捕获一些木材板的大小

在上述的这些例子中，尽管它们本质上是相加的，但应该选择 `ValueRecorder` ，而不是使用 `Counter` 或 `UpDownCounter`，
这表明着我们不仅对于总和的统计存在兴趣。如果你对收集数据的分布不感兴趣，则可以选择其中一种 `adding instruments` ，
使用`ValueRecorder`的意义在于对数据的分布进行分析。

请谨慎的使用 `ValueRecorder` ，因为它会消耗比 `adding instruments` 更多的性能。

### SumObserver

`SumObserver` 对应 `Counter` 的异步形式，用于捕获 `Observe(sum)` 单调求和数据。
名称中会出现 `Sum` ，以提醒用户直接用于捕获数据总和。
使用` SumObserver` 捕获任何从零开始并在整个过程生命周期中上升并且永不下降的值。

`SumObserver`的示例用法.

- 捕获进程用户/系统 CPU 秒数
- 捕获缓存未命中的数量

`SumObserver` 用来计算计算成本很高以致于多次计算将会导致性能浪费的数据。
例如，需要一个系统调用来捕获进程的 CPU 占用率，它应该定期执行，而不是针对每个请求进行一次计算。
在不实用或浪费的 `instrument` 的个别变化，构成一个总和时`SumObserver`也时一个不错的选择。
例如，即使缓存未命中的数量是单个缓存未命中事件的总和，使用`Counter`同步捕获每个事件的代价太大了。

### UpDownSumObserver

`UpDownSumObserver` 是 `UpDownCounter` 所对应的异步形式，
通过 `Observe(sum)` 函数进行非单调的计数。名称中会出现 "sum" ，以提醒用户这是用来直接用于捕获总和。
使用 `UpDownSumObserver` 捕获在整个过程生命周期中从零开始并且上升或下降的任何值。

`UpDownSumObserver` 的示例用法：

- 捕获进程堆大小
- 捕获活动碎片的数量
- 捕获已启动/已完成的请求数
- 捕获当前队列大小。

关于选择 `SumObserver` 还是`Counter`的考虑，也同样适用于在选择 `UpDownSumObserver` 还是 `UpDownCounter` 的场景。
如果测量数据的计算成本很高，或者相应的更改发生得太频繁以至于无法进行测量，请使用 `UpDownSumObserver` 。

### ValueObserver

`ValueObserver` 是 `ValueRecorder` 与之对应的异步工具，使用 `Observe(value)` 来捕获分组测量值。
这些 instruments 对于捕获计算成本很高的测量数据特别有用，因为使用SDK可以控制它们数据采集的频率。

`ValueObserver` 的示例用法:

- 捕获 CPU 风扇速度
- 捕获 CPU 温度。

注意，这些例子使用了分组度量。在上面的 `ValueRecorder` 案例中，给出了在请求期间捕获同步 adding instrument 的示例用法（获取当前的队列大小）。
但是，在异步情况下，用户应如何决定是否使用 `ValueObserver` 而不是 `UpDownSumObserver` ？

思考一下如何异步报告队列的大小。无论 `ValueObserver` 和 `UpDownSumObserver` 的使用都适用于这种情况的逻辑。
异步 instruments 每个时间间隔仅捕获一次测量，因此在此示例中，`UpDownSumObserver` 报告一个当前统计值，
 `ValueObserver` 报告一个当前的统计值（等于最大和最小值）和一个count = 1。
当没有聚合时，这些结果是相等的。

当只有一个数据点时，定义默认聚合似乎没有意义。在执行聚集性分析时将会使用应用默认聚合，这是为了跨标签集或在分布式配置下聚合数据。
尽管在 `ValueObserver` 每个收集间隔中仅观察一个值，但是默认聚合指定了其如何在没有其他任何配置的情况下将其与其他值聚合的方式。

因此，考虑到 `ValueObserver` 和 `UpDownSumObserver` 之间的选择，建议选择其中更适合当前聚合场景的工具。
如果你正在观察一组机器的队列大小，而你唯一想知道的就是聚合队列大小，使用 `SumObserver` 是因为它产生的是总和数据，而不包括数据的分布。
如果你正在观察一组机器上的队列大小，并且想知道这些机器上队列大小的分布情况，使用 `ValueObserver` 。

### 解释

这些工具有什么区别，为什么只有三个？为什么不是一个？那么又为什么不是十个呢？

正如我们所看到的，这些工具根据它们是否同步，支持相加以及与或关系和单调性而分类。
这种方式通过为每个工具赋予其语义，能够提高 metric events 的性能并使其更容易被理解。

建立不同种类的 instrument 很重要，因为在大多数情况下，它允许SDK“开箱即用”，在无需其他配置的前提下提供良好的默认功能。
instrument 的选择不仅决定事件的含义，还决定用户调用的函数的名称。
函数名称 `Add()` 用于 add instruments ，`Record()`用于 grouping instruments ，`Observe()`用于 asynchronous instruments ，
不同名称帮助传达这些动作的含义。

下表概述了每种 instruments 的特性和标准实施。

| **名称** | Instrument种类 | 方法(参数) | 默认聚合 | 备注 |
| --------------------- | ----- | --------- | ------------- | --- |
| **Counter**           | 同步 单调 adding  | Add(increment) | Sum | Per-request, part of a monotonic sum |
| **UpDownCounter**     | 同步 adding      | Add(increment) | Sum | Per-request, part of a non-monotonic sum |
| **ValueRecorder**     | 同步             | Record(value)  | [TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636)  | Per-request, any grouping measurement |
| **SumObserver**       | 异步 单调 adding  | Observe(sum)   | Sum | Per-interval, reporting a monotonic sum |
| **UpDownSumObserver** | 异步 adding      | Observe(sum)   | Sum  | Per-interval, reporting a non-monotonic sum |
| **ValueObserver**     | 异步             | Observe(value) | LastValue | Per-interval, any grouping measurement |

### 构造函数

`Meter` 接口支持创建并注册新的 metric instruments 的功能。
instruments 的构造函数的命名方式是在它构造的类型、构造器模式或语言中的其他惯用方法上添加 `New-` 前缀

在本规范中每种 instrument 都会有至少一种对应的构造器， (查看 [详细](#metric-instruments))
并且可能由于语言的种类而规定更多的构造器。例如，如果提供了整型和浮点型指针数的专用处理，
OpenTelemetry API将会为每种 instrument 支持2个不同类型的构造函数。

将 instrument 绑定到单个 `Meter` 实例有两个好处:

1. instrument 在第一次使用之可以在零状态的情况下导出，不需要显式注册调用
2. 库的名称和版本与 `metric event` 隐式关联

一些现有的 metric 系统支持支持静态分配 metric instruments ，并在使用时提供等效的 `Meter` 接口。
在一个示例中，典型的statsD客户端，现有的代码可能没有一个方便的地方来存储新的度量工具。
如果这成为一种负担，建议使用全局 `MeterProvider` 来构建静态 `Meter` ，并构建和使用全局范围的度量工具。

Prometheus 客户端的一些用户目前面临着类似的情况，他们可以将 instruments 分配给全局 `Register`。
这样的代码可能无法在定义 instruments 时访问适当的 `MeterProvider` 或 `Meter` 实例。
如果这成为一种负担，建议使用全局 `MeterProvider` 来构建静态 `Meter` 来构建静态 instruments 。

应用程序应当构造长生命周期的 instruments 。Instruments 在 SDK 的生命周期内被认为是永久的，不会提供删除它们的方法。

## 标签集合

从语义上讲，一组标签是一组字符串键值对的唯一映射。在整个API中，必须以相同的数据格式传递一组标签。
常见的表示形式包含 key values 键值对的对象列表，或 Map 结构的 key value 键值对。

如果将标签作为 key：values 的有序列表传递，如果存在重复的键，则将获取给定键的列表中的最后一个值，以形成唯一的映射。

标签值的类型通常被导出工具假定为字符串，但编程语言级别的考虑，
标签值类型可以是该语言中用于表示字符串的任何常用类型。

用户不需要预先声明将与 API 中的 metric instruments 一起使用的一组标签键。
调用 API 时，用户可以针对任何 metric event 自由使用任何标签集。

### 标签性能

在整个 metric 数据的生产中，标签处理可能会产生巨大的成本。

SDK 支持对进程进行聚合来找到源于一个 instrument 并具有相同标签集组合活跃记录的能力。这允许将测量数据组合在一起。
通过使用绑定的同步 instruments 和批处理方法( `RecordBatch` ， `BatchObserver` )可以降低标签处理成本。

### 可以选择的能力：经过排序的标签

考虑到编程语言的能力，API 可以支持对标签键进行排序。
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

## 同步 instrument 详情

为同步 instruments 指定了以下详细信息。

### 同步调用约定

metrics API提供了三种语义上等效的方式来使用同步`instruments`捕获测量值：

- 调用绑定 instruments ，它们具有一组预先关联的标签
- 直接调用 instruments ，传递相关的标签集
- 使用一组标签对多个 instruments 进行批量记录测量。

这三种方法均会生成等效的 metric events ，但会提供不同程度的性能和便利性。

metric 的 API 性能取决于输入新度量数据时的成本，通常由处理标签的成本决定。
调用绑定的 instruments 是性能最高的调用方式，因为它们可以在多次使用中分摊处理标签的成本。
通过另一种调用约定 `RecordBatch()` 记录多个原始数据，这是提高性能的一个很好的选择，因为处理标签的成本分散在多个度量中。
直接调用约定是最方便的，但通过 API 输入度量值是性能最差的调用约定。

#### 绑定 instrument 调用约定

在对性能有一定要求的情况下，并且常常会使用同一套标签集合 metric instrument 的情况下，开发人员可以选择使用调用绑定 instruments 作为优化。
为了从使用绑定的 instrument 当中受益，它要求与重复使用的标签绑定的特定的 instrument 被重新使用。
如果一个 instrument 将多次使用相同的标签，则使用与这些标签相对应的绑定 instrument 可确保获得最高的性能。

要绑定一个 `instrument`，使用 `Bind(labels...)` 方法，这个方法将会返回一个支持相应同步 API （即 `Add()` 或 `Record()` ）的接口。
绑定的 `instrument` 在调用的时候不需要声明标签；相应的 `metric event` 将会与绑定到 `instrument` 的标签相关联。

由于其性能优势，绑定的 `instrument` 也会消耗SDK中的资源。绑定的 `instrument` 必须支持 `Unbind()` 方法，使用户可以结束绑定并释放关联的资源。
请注意， `Unbind()` 这并不意味着删除时间序列，它仅允许 SDK 在没有待处理的更新后忘记时间序列的存在。

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

#### 直接 instrument 调用约定

当调用时的便利性比性能更重要，或者提前不知道值的时候，用户可以选择直接在 metric instruments 上进行操作，
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
直接调用很方便，因为它们不需要分配和存储绑定的 instrument 。
它们适用于很少使用的 instrument 或很少使用同一组标签的情况。
与绑定 instrument 不同，使用直接调用约定时，不会长期消耗SDK资源。

#### RecordBatch 调用约定

有一个用于输入测量值的最终 API，就像直接访问调用约定一样，但其支持多个同时测量数据。
`RecordBatch` API的使用支持同时输入多个测量值，这意味着对几种 instrument 进行了语义上的原子更新。
对 `RecordBatch` 的调用可以跨多个度量摊销标签处理的成本。

例如:

```golang
func (s *server) method(ctx context.Context) {
    // ... other work

    s.meter.RecordBatch(ctx, labels,
        s.instruments.counter.Measurement(1),
        s.instruments.upDownCounter.Measurement(10),
        s.instruments.valueRecorder.Measurement(123.45),
    )
}
```

另一个用于批量记录的有效接口使用了一个构建器模式：

```java
    meter.RecordBatch(labels).
        put(s.instruments.counter, 1).
        put(s.instruments.upDownCounter, 10).
        put(s.instruments.valueRecorder, 123.45).
        record();
```

使用批量处理调用约定在语义上与一系列直接调用相同，只是增加了原子性。
因为值是在单个调用中输入的，从导出程序的角度来看，SDK 有可能实现原子更新，因为打个比方， SDK 可以对单个批量更新进行排队，或者仅进行一次加锁。
像直接调用约定一样，使用批处理调用约定时也不会长期消耗 SDK 资源。

### 与分布式上下文关联

同步 instruments 在运行时与分布式
[上下文](../context/context.md)
存在着隐式关联，它可能包括 Span 和 Baggage entries 的信息。
Metric SDK 可以采用以多种方式来使用这些信息，但在 OpenTelemetry 中有一个特性十分重要。

#### Baggage 作为 metric 的标签

Baggage 在 OpenTelemetry 中得到支持，它是一种让标签在分布式计算中从一个进程传播到另一个进程的方法。
有时候使用分布式 baggage entries 作为 metric 标签来聚合 metric 数据是有用的。

必须显式地配置 Baggage 的使用，使用[Views API (WIP)](https://github.com/open-telemetry/oteps/pull/89)
来选择应作为标签来使用的特定关键 baggage entries 。默认情况下 SDK 不会在出口管道中自动使用 Baggage 标签，
因为使用 Baggage 标签可能会产生很大的消耗。

为应用 Baggage 标签配置视图是一项[正在进行的工作](https://github.com/open-telemetry/oteps/pull/89) 。

## 异步 instrument 细节

以下描述信息仅限于异步的 instruments 。

### 异步调用约定

metrics 的 API 提供了两种语义上等价的方法来使用异步 instruments 捕获度量数据，
通过单一 instruments 回调，或者通过多个 instruments 进行批回调。

无论是单个的还是批量的，异步 instruments 必须通过一个回调来进行观察。当使用 `null` 会调构造函数将会返回无操作的 instruments 。
任何一个异步 instruments 被指定了多个回调，都会被认为是一个异常操作。

Instruments 在每组不同的标签中，key 只能对应到一个 value。
当单个 Instruments 的标签集合中 key 重复的时候，只会取最后一个监测值，丢弃较早的数据，不会导致异常。

#### 单一 instrument 的监测

一个单独的回调与一个 instrument 所绑定。它的回调通过 `Observe(value, labels…)` 函数接受一个 `ObserverResult` 。

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

一个 `BatchObserver` 支持在一个回掉中监测多个 instruments 。
它的回调支持通过 `Observe(value, observations…)` 函数接收一个 `BatchObserverResult` 。

在异 instrument 上，通过调用 Observation(value) 返回监测结果。

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

### 异步观察构成一个当前的值集

异步 instrument 回调允许对每个 instrument 、每个不同的标签集、每次回调调用观察一个值。
由一次回调调用记录的一组值代表 instrument 的当前快照；这一组值定义了 instrument 直到下一个收集间隔的最后一个值。

异步 instrument 应记录每一个它认为是"当前"的标签集的监测结果。这意味着异步回调应该持续监测一个值，
即使这个值自上次回调调用以来没有改变。不监测标签集意味着某个值不再存在。
当最后一个值在一个收集时间间隔内没有被监测到时，它就不再是当前值而是变成未定义。

对于异步 instruments 来说，最后一个值的定义是可以存在的，因为它们是由 SDK 协调收集的，而且它们应该上报所有的当前测量值。
该属性的另一个说法是，SDK 可以在内存中保存一个集测量间隔区间，以查找任何 instrument 和标签集的当前最后值。
通过这种方式，异步 instruments 支持使用单个时间点收集的数据查询当前值，而不依赖于采集间隔的持续时间。

回想一下，没有为同步 instruments 定义一个"最后一个值"的概念，这恰好是因为它没有一个明确的定义"当前"这个概念。
因为没有机制可以确保在每个间隔内都记录当前值，所以要确定同步 instruments 的"最后记录"值，就可能需要检查多个数据收集窗口，。

#### Asynchronous instruments define moment-in-time ratios

上面所描述的异步 instruments 的当前值集的概念对于速率的检测是很有帮助的。
当 instruments 的一组测量值加起来是一个整体，那么每一个测量值可以除以同一区间的测量值的总和，以计算其这次测量的结果的相对增长比例。
受益于异步 instruments 的特性，当前相对增长就是这样定义做到与收集间隔无关。

## 并发

对于支持并发执行的语言，Metrics API 提供了一些具体的保障和安全性。但是并非所有的 API 函数都可以安全地被并发调用。

**MeterProvider** - 所有方法都可以安全地被并发调用。

**Meter** - 所有方法都可以安全地被并发调用。

**Instrument** - 任何 Instrument 的所有方法都可以安全地同时调用。

**Bound Instrument** - 任何 Bound Instrument 的所有方法都可以安全地同时调用。

## 相关的 OpenTelemetry 工作

在编写本规范时，正在进行一些持续的努力。

### Metric 视图

该 API 不支持 metric instruments 的可配置聚合。

View API 定义为 SDK 方式的接口，该方式支持配置聚合。
包括应用哪个操作符（sum, p99, last-value 等等）和使用哪个维度。

可以查看[当前有关这个话题的讨论](https://github.com/open-telemetry/opentelemetry-specification/issues/466) 
和[当前的 OTEP 草案](https://github.com/open-telemetry/oteps/pull/89) 。

### OTLP Metric 协议

如上所述， OTLP 协议旨在以无侵入的方式导出 metric 数据。该协议的几个细节正在制定中。请参阅
[当前协议](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto) 。

### Metric SDK 默认实现

OpenTelemetry SDK 包含对 metric API 的默认支持。默认 SDK 的规范正在进行中，请参阅 
[current draft](https://github.com/open-telemetry/opentelemetry-specification/pull/347).
