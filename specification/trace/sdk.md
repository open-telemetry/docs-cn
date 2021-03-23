# Tracing SDK

**状态**: [稳定](../document-status.md)

<details>

<summary>目录</summary>

* [Tracer Provider(Tracer提供者)](#tracer-provider)
* [Additional Span Interfaces（额外的Span接口）](#additional-span-interfaces)
* [Sampling（采样）](#sampling)
* [Span Limits（Span的限制）](#span-limits)
* [Id Generator（id生成器）](#id-generators)
* [Span Processor（span处理器）](#span-processor)
* [Span Exporter（span出口）](#span-exporter)

</details>

## Tracer Provider(Tracer提供者)

### Tracer的创建

新的`Tracer`实例都是通过`TracerProver`来创建的（查看[API](api.md#tracerprovider)）。需要提供`name(名称)`和`version(版本)`两个参数 给`TracerProvider`
来创建一个存储在`Tracer`实例中的[`InstrumentationLibrary`][otep-83]实例

所有的配置（比如： [Span处理器](#span-processor)，[Id生成器](#id-generators)，[Span的限制](#span-limits)和[`采样器`](#sampling)）
都是被`TracerProvider`所管理的，`TracerProvider`也提供了一些途径去配置这些实现自SDK的元素，至少在创建或初始化的时候是可以配置的

TracerProvider可以提供更新配置的方法。如果配置已更新（例如，添加“ SpanProcessor”），则更新后的配置还必须应用于所有已返回的`Tracer`（也就 是说，在配置更改之前或之后，从`TracerProvider`
获取`Tracer`都没关系）。注意：在实现上，这可能意味着`Tracer`实例具有对其`TracerProvider`的引用，并且只能通过该引用来访问配置。

### Shutdown(关闭)

此方法可以让provider做任何有必要的清理工作

对于每个`TracerProvider`实例，`Shutdown`只能被调用一次。在`Shutdown`被调用之后, 就不再允许获取`Tracer`。如果可以的话，在被调用时，SDK 可以返回一个无操作的Tracer

`Shutdown` 应该让调用者知道调用是否成功，还是失败或超时。

`Shutdown` 应该在一段时间内完成或中止。`Shutdown`可以实现成同步阻塞的方式或者异步回调的方式。OpenTelemetry的客户端作者可以决定是否将shutdown 超时做成可配置的

`Shutdown` 的实现中，至少要包含所有内部processor的`Shutdown`

### ForceFlush（强制冲刷）

此方法为provider提供了一种方式，可以立即导出所有内部processor尚未导出的span

`ForceFlush` 应该让调用者知道调用是否成功，还是失败或超时。

`ForceFlush` 应该在一段时间内完成或中止。`ForceFlush`可以实现成同步阻塞的方式或者异步回调的方式。OpenTelemetry的客户端作者可以决定是否将flush超时做成可配置的

`ForceFlush` 必须在所有已注册的`SpanProcessor`上调用`ForceFlush`。

## Additional Span Interfaces（额外的Span接口）

[API-level definition for span's interface ](api.md＃span-operations)
仅定义对span的只写访问。这很好，因为instrumentations和应用程序并不想要将存储在span的数据 应用的程序逻辑中。但是，在某些地方SDK最终还是需要读回这些数据的。因此，SDK规范中明确了`Span`的一些类似参数的要求：

* **Readable span接口**：以此作为参数的方法，能够访问span中所有在[in the API spec](api.md#span-data-memebers)列出的这些信息。特别是，
  它还必须能够（隐式地）访问与span相关联的`InstrumentationLibrary`和`Resource`信息。它还必须能够可靠地确定Span是否已结束 （某些语言可能通过将结束时间戳记为`null`
  来表示，而其他语言可能具有显式的布尔类型的`hasEnded`标志）。

  以此作为参数的方法不能修改Span 注意：通常这将使用新的接口或（不可变的）值类型来实现。在某些语言中，SpanProcessor可能具有与exporter不同的可读span类型（例如：`SpanData`
  可能包含一个不可变的快照，而`ReadableSpan`可能直接读取来自`Span`接口操作的底层数据结构）

* **Read/write span接口**: 以此作为参数的方法，必须有权限访问[API-level definition for span's interface ](api.md#span-operations)
  中定义的完整的span API，以及能够检索到所有添加到span中的信息（像*readable span*那样）

  调用此接口的方法能够获得与[span creation API](api.md#span-creation)相同的已返回给用户的`Span`实例和类型（例如：`Span`可以作为方法的一个参数，或者也可以提供getter方法）

## Sampling（采样）

采样是OpenTelemetry引入的一种控制噪点和开销的机制，这种机制会减少trace的收集量、发送到后端的trace数量

采样可以在trace收集的不同的阶段进行。最早可以在trace创建之前，最晚可以在收集器Collector处理之后

OpenTelemetry API有两个负责数据收集的属性：

* `Span`的`IsRecording`属性。如果是`false`，当前的`Span`会丢弃所欲的tracing数据（属性、事件、状态等）。用户可以通过该属性来决定是否收集高成本的trace数据。
  [Span Processor（Span处理器）](#span-processor)只会接收`IsRecording`是`true`的span。但是，如果`Sampled`
  没有设置的话，[Span Exporter（Span 导出器）](#span-exporter)同样不会接收。

* `SpanContext`中的`TraceFlags`标识：`Sampled`。这个标识会通过`SpanContext`
  传播给子span。了解更多请查看[W3C Trace Context specification](https://www.w3.org/TR/trace-context/#sampled-flag) 。这个标识表明这个`Span`
  已经被采样且将会被导出。
  [Span Exporters（Span 导出器）](#span-exporter) 将接收那些`Sampled`为true的span，而不会接收`Sampled`是false的span

`SampledFlag == false`且`IsRecording == true`意味着当前`Span`会记录信息，但是子`Span`很可能不会记录

`SampledFlag == true`且`IsRecording == false`会导致分布式追踪中出现间隙，因此，OpenTelemetry API不允许这种设置

<a name="recording-sampled-reaction-table"></a>

下面的表格总结出了`IsRecording`和`Sampled`所有的预期组合

| `IsRecording` | `Sampled` 标记  | Span处理器接收Span?            |  Span导出器接收Span?           |
| ------------- | -------------- | ----------------------------- | ---------------------------- |
| true          | true           | 是                            | 是                         |
| true          | false          | 是                            | 否                        |
| false         | true           | 不允许                         | 不允许                 |
| false         | false          | 否                            | 否                        |

SDK定义了[`Sampler`]（＃sampler）接口以及一组[内置采样器]（＃built-in-samplers），并将采样器与每一个[`TracerProvider`]关联。

### SDK Span的创建

当要求创建Span时，SDK必须按顺序执行以下操作：

1. 如果存在一个有效的父trace ID，那就直接使用。否则生成一个trace ID （注意：需要在`ShouldSample`被调用前完成，因为trane ID是它的入参）

2. 查询`Sampler`的[`ShouldSample`](#shouldsample)方法 （注意：[内置的`ParentBasedSampler`](#parentbased)）可以被用于使用父级的采样决策，
   将已设置的SampleFlag转换为REECORD，然后将未设置的SampleFlag转换为DROP）

3. 为`Span`生成一个新的Span ID，独立于采样决策。这样做是为了使其他组件（例如日志或异常处理）也可以依赖唯一的span ID，即便这个`Span`是no-recording实例。

4. 创建一个基于`ShouldSample`返回的决策的span： 有关如何在Span上设置`IsRecording`和`Sampled`的信息，请参见下面的[[ShouldSample]
   （＃shouldsample）返回值说明），`Span`是否会传递给`SpanProcessor`请参考[上面的表格](#recording-sampled-reaction-table)
   一个no-recording的span的实现机制类似于创建一个没有SDK的`Span`，
   参考[wrapping a SpanContext in a Span（将SpanContext包装进Span）](api.md#wrapping-a-spancontext-in-a-span)的描述

### Sampler（采样器）

`Sampler` 接口允许用户自定义采样器，该采样器将根据创建`Span`之前通常可用的信息来返回采样`SamplingResult`。

#### ShouldSample（采样方式）

返回被创建的`Span`的采样决策

**必填参数**

* [`Context`](../context/context.md)（带有父`Span`的）。Span的SpanContext可能无法表明一个根span。
* `TraceId`。如果父`SpanContext`包含了一个有效的`TraceId`，那么他们必须始终匹配。
* Name（名称）
* `SpanKind`
* `Span`的一系列初始化属性
* 一些要被关联到`Span`的链接集合。这个参数通常用于批量操作，请查看[Links Between Spans（链接多个Span）](../overview.md#links-between-spans).

**返回值:**

返回一个类型为`SamplingResult`的结果，该结果包含：

* 采样决策Decision。枚举值如下：
    * `DROP` - `IsRecording() == false`，span不会被记录，而且所有的事件和属性也会被丢弃
    * `RECORD_ONLY` - `IsRecording == true`，同时`Sampled`标记不能设置
    * `RECORD_AND_SAMPLE` - `IsRecording == true`，且设置`Sampled`标记
* span属性的集合，这些属性同样被添加到`Span`中。返回的对象必须是不可变的（多次调用将返回不同的不可变对象）
* Trace状态`Tracestate`，在new `SpanContext`的时候被关联到`Span`。采样器在这如果返回一个空的`Tracestate`，那么`Tracestate`将会被清除；
  所以如果采样器不打算修改它，那通常会直接返回传入的`Tracestate`。

#### GetDescription（获取描述）

返回采样器名称和配置的简短描述。这在调试页面或日志中可能会被展示。例如：
`"TraceIdRatioBased{0.000100}"`。

Description不会随着时间改变，所以调用者可以缓存返回的值。

### Built-in samplers（内置采样器）

OpenTelemetry有很多的内置采样器可供选择。默认的采样器是`ParentBased(root=AlwaysOn)`。

#### AlwaysOn（总是打开）

* 采样决策Decision为`RECORD_AND_SAMPLE`
* 描述Description为`AlwaysOnSampler`

#### AlwaysOff（总是关闭）

* 采样决策decision为`DROP`
* 描述Description为`AlwaysOffSampler`

#### TraceIdRatioBased(基于TraceId的比例)

* `TraceIdRatioBased`必须忽略父级的`SampledFlag`，为了体现父`SampledFlag`，
  `TraceIdRatioBased`应被用作下方详细说明的`ParentBased`采样器的代理。
* 描述Description为`TraceIdRatioBased{0.000100}`。

TODO（待办）：添加关于`TraceIdRatioBased`是如何通过`TraceID`
来实现的。[#1413](https://github.com/open-telemetry/opentelemetry-specification/issues/1413)

##### Requirements for `TraceIdRatioBased` sampler algorithm（`TraceIdRatioBased`采样算法的必需条件）

* 采样算法是确定性的。通过traceId来表明是否要采样的trace是独立于语言、时间等条件的。为此，当计算采样决策的时候，实现上就需要一个确定的关于`TraceId`的hash算法。因此，在子`Span`
  上运行任何采样器都会产生同样的采样决策

* 一个指定采样率rate的`TraceIdRatioBased`采样器必需采集那些采样率更低的`TraceIdRatioBased`的trace。这在一个后端系统希望比前端系统采用更高的采样率的时候很重要，这样前端系统还是正常采样，
  但是一些额外的trace就只会被后端系统采样。

* **警告：** 由于确切的算法还没有指定（查看上面的TODO），很有可能在某个语言的SDK中会做改动，这就会打破依赖于该算法结果的代码。当前只有配置和创建的API是可以认为是稳定的。建议只在根span上采用这种算法(in
  combination with [`ParentBased`](#parentbased))，因为不同语言的SDK，或者是同种语言的不同版本对于同样的输入都有可能产生不一致的结果。

#### ParentBased（基于父级）

* 这个一个复合采样器。`ParentBased`帮助区分以下情况：
    * No parent（根span）。
    * Remote parent（`SpanContext.IsRemote() == true`），且`SampledFlag`等于`true`
    * Remote parent（`SpanContext.IsRemote() == true`），且`SampledFlag`等于`false`
    * Local parent（`SpanContext.IsRemote() == false`），且`SampledFlag`等于`true`
    * Local parent（`SpanContext.IsRemote() == false`），且`SampledFlag`等于`false`

必需参数：

* root(Sampler) - 被无父级的span（根span）调用的采样器

可选参数：

* `remoteParentSampled(Sampler)` (默认值: AlwaysOn)
* `remoteParentNotSampled(Sampler)` (默认值: AlwaysOff)
* `localParentSampled(Sampler)` (默认值: AlwaysOn)
* `localParentNotSampled(Sampler)` (默认值: AlwaysOff)

|父级     | 父级isRemote？ | 父级IsSampled？| 触发的采样器          |
|--------|-----------------|-----------------|--------------------------| 
|无  | n/a             | n/a             |`root()`                  |
|有 |true             |true             |`remoteParentSampled()`   |
|有 |true             |false            |`remoteParentNotSampled()`|
|有 |false            |true             |`localParentSampled()`    | 
|有 |false            |false            |`localParentNotSampled()` |

## Span Limits（Span的限制）

错误的代码会添加预期之外的属性、事件、链接到span中。如果这些收集不被解绑，它们会迅速耗尽可用内存，从而导致奔溃，而且这些奔溃很难安全的恢复。

为了防止此类错误，SDK可能丢弃那些导致收集器的元素数量超过配置上限的属性、链接或事件。

如果SDK实现了上述限制，则必须提供更改这些限制的途径。通过TraceProvider的配置，可以像下面的Java示例一样让用户配置单独的限制。

配置的名称可以是：`AttributeCountLimit`,`EventCountLimit` 和 `LinkCountLimit`。这些选项捆绑在一个类中，称为`SpanLimits`。
实现上可以提供附加的配置，例如`AttributePerEventCountLimit`和`AttributePerLinkCountLimit`。

```java
public final class SpanLimits {
    SpanLimits(int attributeCountLimit, int linkCountLimit, int eventCountLimit);

    public int getAttributeCountLimit();

    public int getEventCountLimit();

    public int getLinkCountLimit();
}
```

**可配置参数：**

* `AttributeCountLimit` (默认值：128) - 允许的最大属性数量；
* `EventCountLimit` (默认值：128) - 允许的最大事件数量；
* `LinkCountLimit` (默认值：128) - 允许的最大链接数量；
* `AttributePerEventCountLimit` (默认值：128) - 一个span允许的最大属性个数；
* `AttributePerLinkCountLimit` (默认值：128) - 一个span允许的最大链接个数；

应该发出一条日志告知用户由于上限限制，属性、事件、链接已被丢弃。为防止过多的日志记录，不应为每个跨度或每个废弃的属性，事件或链接发送一次日志。

## Id Generators（Id生成器）

SDK默认情况下，随机生成`TraceId`和`SpanId`。

SDK提供自定义`TradeId`和`SpanId`的方式

SDK可以像下方Java代码一样提供一个接口供自定义实现（接口名称可以是`IdGenerator`，方法名必须和[SpanContext](./api.md#retrieving-the-traceid-and-spanid)
保持一致），提供两个方法的扩展点，一个方式是生成`SpanId`，一个方式是生成`TraceId`。

```java
public interface IdGenerator {
    byte[] generateSpanIdBytes();

    byte[] generateTraceIdBytes();
}
```

不得将实现自特定供应商协议（例如AWS X-Ray跟踪ID生成器）的`IdGenerator`作为OpenTelemetry核心来维护、分发

## Span processor（Span 处理器）

Span处理器是一个接口，它允许在start和end方法的调用上添加钩子函数。只有在[`IsRecording`](api.md#isrecording)为true的时候，span处理器才会被调用

内置的span处理器负责批处理操作和转换span为可导出的样式，以及最终批量将传递给exporter

Span处理器可以直接在SDK `TracerProvider`上注册，并以与注册时相同的顺序进行调用。

每一个注册到`TraceProvider`上的处理器都存在管道的开始端，而管道是由span处理器和可选的导出器exporter组成的。SDK必须允许以单独的导出器exporter来结束每一个管道

SDK必须允许用户实现和配置自定义处理器，以及在一些高级场景下装饰内置处理器（例如：打标操作tagging或过滤操作filtering）

下图显示了`SpanProcessor`与SDK中其他组件的关系：

```
  +-----+--------------+   +-------------------------+   +-------------------+
  |     |              |   |                         |   |                   |
  |     |              |   | Batching Span Processor |   |    SpanExporter   |
  |     |              +---> Simple Span Processor   +--->  (JaegerExporter) |
  |     |              |   |                         |   |                   |
  | SDK | Span.start() |   +-------------------------+   +-------------------+
  |     | Span.end()   |
  |     |              |
  |     |              |
  |     |              |
  |     |              |
  +-----+--------------+
```

### Interface definition（接口定义）

#### OnStart（启动时）

`OnStart`在span start后调用。此方法在start span的线程上同步调用，因此它不应阻塞或引发异常。

**入参:**

* `span` - 一个[read/write span object](#additional-span-interfaces)。它应该保存一个对此span对象的引用，并且应该在其中能反映此span的更新。
  例如，这对于创建那些定期计算/打印来自后台线程所有活跃span信息的Span处理器很有用。

* `parentContext` - 由SDK确定的span父级`Context`
  （显示传递的`Context`，当前`Context`，明确要求一个空`Context`）

**返回值:** `Void（空）`

#### OnEnd(Span)（结束时(Span)）

`OnEnd`在span结束时被调用（即设置了结束时间戳）。必须在[`Span.End（）`API]（api.md＃end）中同步调用此方法，这样不会阻塞或引发异常。

**入参：**

* `Span` - a [readable span object](#additional-span-interfaces)。注意：即使过去的Span在技术上是可写的，但由于它已经在此处结束，因此不允许对其进行修改。

**返回值:** `Void`（空）

#### Shutdown()（关闭）

关闭处理器。在SDK关闭时调用。这是处理器做清理工作的时机。

每一个`SpanProcessor`实例只能调用一次`Shutdown`。调用`Shutdown`后，随后调用`OnStart`，`OnEnd`，或`ForceFlush`都是不允许的。SDK应该优雅地忽略这些调用。

`Shutdown` 应该让调用者知道调用是成功、失败或是超时。

`Shutdown` 包含了`ForceFlush`同样的效果。

`Shutdown` 应该在一段时间内完成或者中止。`Shutdown`可以实现成同步阻塞或者异步回调的方式。OpenTelemetry客户端作者可以决定是否将shutdown超时配置做成可配置的。

#### ForceFlush()（强制冲刷）

导出所有未被导出的span到配置的导出器`Exporter`

`ForceFlush` 应该让调用者知道调用是成功、失败或是超时。

`ForceFlush` 仅在绝对必要的情况下才调用“ ForceFlush”，例如在使用某些FaaS提供程序时，这些提供程序可能在调用之后、`处理器`导出完整的跨度之前挂起该进程。

`ForceFlush` 应该在一段时间内完成或者中止。`ForceFlush`可以实现成同步阻塞或者异步回调的方式。OpenTelemetry客户端作者可以决定是否将flush超时配置做成可配置的。

### Built-in span processors（内置span处理器）

标准的OpenTelemetry SDK应该实现如下所述的简单处理器和批量的处理器。其他常见的处理场景应首先考虑下[OpenTelemetry Collector](../overview.md#collector)中的【进程外实现】

#### Simple processor（简单处理器）

这是`SpanProcessor`的一个实现，该处理器传递finished spans，并在完成后传递export-friendly的span的数据给配置的`SpanExporter`

**可配置参数：**

* `exporter（导出器）` - span数据推给导出器exporter

#### Batching processor（分批处理器）

这是`SpanProcessor`的一个实现，该处理器批量传递finished spans，并在完成后传递export-friendly的span的数据给配置的`SpanExporter`

**可配置参数：**

* `exporter` - span数据推给导出器exporter
* `maxQueueSize` - 最大队列长度。达到该值后span将会被丢弃。默认值是`2048`。
* `scheduledDelayMillis` - 两次连续导出之间的延迟间隔（以毫秒为单位）。默认值为5000。
* `exportTimeoutMillis` - 超时时间：在被超时取消之前，export能运行多久。默认值是`30000`.
* `maxExportBatchSize` - 每次批量export的最大数量。它必须小于等于`maxQueueSize`。默认值是`512`.

## Span Exporter（Span导出器）

`Span Exporter` 定义了一个特定协议exporter必须实现的接口，以便可以将其插入到OpenTelemetry SDK中并支持发送telemetry数据。

该接口的目的是最大程度地减少依赖协议的telemetry导出器的实现负担。协议导出器主要是一个简单的telemetry数据编码器和发送器。

### Interface Definition（接口定义）

导出器必须支持两个功能：**导出**和**关闭**。在强类型语言中，通常会有2个单独的导出器`Exporter`接口，一个接收span（SpanExporter），另一个接收metrics（MetricsExporter）

#### `Export(batch)（批量导出）`

导出一批[readable spans](#additional-span-interfaces)。实现此方法的协议导出器通常会序列化数据，并将数据发送到目的地

Export()方法不会被同一个exporter并发的调用。Export()方法只能在当前调用返回才会进行下一次调用。

Export()不能无限期地阻塞，必须有一个合理的上限，在此上限之后必须超时并返回错误结果（`Failure`）。

exporter需要有重试逻辑，这是exporter的责任。默认的SDK不应实现重试逻辑，因为所需的逻辑很大程度上取决于具体的协议和span要发送去的后端

**入参:**

batch - 一批[readable spans](#additional-span-interfaces)。批量数据的类型是特定于语言的，通常是某种列表。例如对于Java中的span通常为`Collection<SpanData>`

**返回值:** ExportResult（导出结果）:

ExportResult是下面的一种：

*`Success` - 该批次全部成功导出。对于协议导出器这通常意味着数据已经通过线路发送且已传递给目标服务器
*`Failure` - 导出失败。改批次必须被丢弃。例如当数据包含错误的数据不能被序列化

注意： 如果异步回调是当前语言的常用手段的话，那么结果可能就通过这种异步回调方式返回

#### `Shutdown()`（关闭）

关闭导出器exporter。在SDK关闭的时候被调用。这是exporter做相关清理工作的一个时机。

`Shutdown` 只会被`Exporter`实例调用一次。在`Shutdown`被调用之后，随后的`Export`调用不会被允许并返回一个`Failure`结果

`Shutdown` 不应无限期地阻塞（例如，如果它尝试冲刷数据但目标地址不可用）。OpenTelemetry客户端作者可以决定是否将shutdown超时配置做成可配置的。

### Further Language Specialization

基于上述通用的接口定义，library作者必须为特定的语言定义确切的接口

鼓励作者在接口上使用高效的数据结构，适合快速序列化为protocol exporter的线路传输格式以及较低内存管理器的压力。后者通常需要理解特定语言的内存管理器
如何优化出快速生成、短暂存活的数据结构，让生命周期更简单。通常的建议是减少分配数量和使用allocation arenas进行分配，从而避免在telemetry数据生成率很高时爆炸式的进行【分配/接触分配/回收】

#### Examples（示例）

这些是`Exporter`接口在一些特定语言上的例子展示。例子仅仅是说明性的。OpenTelemetry客户端作者可以自由的偏离这些遵循`Exporter`概念精神的例子

##### Go SpanExporter Interface（Golang的SpanExporter接口）

```go
type SpanExporter interface {
Export(batch []ExportableSpan) ExportResult
Shutdown()
}

type ExportResult struct {
Code         ExportResultCode
WrappedError error
}

type ExportResultCode int

const (
Success ExportResultCode = iota
Failure
)
```

##### Java SpanExporter Interface（Java的SpanExporter接口）

```java
public interface SpanExporter {
    public enum ResultCode {
        Success, Failure
    }

    ResultCode export(Collection<ExportableSpan> batch);

    void shutdown();
}
```

[trace-flags]: https://www.w3.org/TR/trace-context/#trace-flags

[otep-83]: https://github.com/open-telemetry/oteps/blob/main/text/0083-component.md