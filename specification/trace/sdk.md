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

所有的配置（比如： [Span处理器](#span-processor), [Id生成器](#id-generators),[Span的限制](#span-limits) 和 [`采样器`](#sampling)）
都是被`TracerProvider`所管理的，`TracerProvider`也提供了一些途径去配置这些实现自SDK的元素，至少在创建或初始化的时候是可以配置的

TracerProvider可以提供更新配置的方法。如果配置已更新（例如，添加“ SpanProcessor”），则更新后的配置还必须应用于所有已返回的`Tracer`（也就 是说，在配置更改之前或之后，从`TracerProvider`
获取`Tracer`都没关系）。注意：在实现上，这可能意味着`Tracer`实例具有对其`TracerProvider`的引用，并且只能通过该引用来访问配置。

### Shutdown(关闭)

此方法可以让provider做任何有必要的清理工作

对于每个`TracerProvider`实例，`Shutdown`只能被调用一次。 在`Shutdown`被调用之后, 就不再允许获取`Tracer`。如果可以的话，在被调用时，SDK 可以返回一个无操作的Tracer

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
  中定义的完整的span API， 以及能够检索到所有添加到span中的信息（像*readable span*那样）

  调用此接口的方法能够获得与[span creation API](api.md#span-creation)相同的已返回给用户的`Span`实例和类型（例如：`Span`可以作为方法的一个参数， 或者也可以提供getter方法）

## Sampling（采样）

采样是OpenTelemetry引入的一种控制噪点和开销的机制，这种机制会减少trace的收集量、发送到后端的trace数量

采样可以在trace收集的不同的阶段进行。最早可以在trace创建之前，最晚可以在收集器Collector处理之后

OpenTelemetry API有两个负责数据收集的属性：

* `Span`的`IsRecording`属性。如果是`false`， 当前的`Span`会丢弃所欲的tracing数据（属性、事件、状态等）。用户可以通过该属性来决定是否收集高成本的trace数据。
  [Span Processor（Span处理器）](#span-processor)只会接收`IsRecording`是`true`的span。但是，如果`Sampled`
  没有设置的话，[Span Exporter（Span 导出器）](#span-exporter)同样不会接收。

* `SpanContext`中的`TraceFlags`标识：`Sampled`。这个标识会通过`SpanContext`
  传播给子span。了解更多请查看[W3C Trace Context specification](https://www.w3.org/TR/trace-context/#sampled-flag) 。 这个标识表明这个`Span`
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
    * `RECORD_ONLY` - `IsRecording == true`， 同时`Sampled`标记不能设置
    * `RECORD_AND_SAMPLE` - `IsRecording == true`，且设置`Sampled`标记
* span属性的集合，这些属性同样被添加到`Span`中。返回的对象必须是不可变的（多次调用将返回不同的不可变对象）
* Trace状态`Tracestate`，在new `SpanContext`的时候被关联到`Span`。采样器在这如果返回一个空的`Tracestate`， 那么`Tracestate`将会被清除；
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

* 采样算法是确定性的。通过traceId来表明是否要采样的trace是独立于语言、时间等条件的。为此，当计算采样决策的时候，实现上就需要一个确定的关于`TraceId`的hash算法。 因此，在子`Span`
  上运行任何采样器都会产生同样的采样决策

* 一个指定采样率rate的`TraceIdRatioBased`采样器必需采集那些采样率更低的`TraceIdRatioBased`的trace。这在一个后端系统希望比前端系统采用更高的采样率的时候很重要，这样前端系统还是正常采样，
  但是一些额外的trace就只会被后端系统采样。

* **警告：** 由于确切的算法还没有指定（查看上面的TODO），很有可能在某个语言的SDK中会做改动，这就会打破依赖于该算法结果的代码。当前只有配置和创建的API是可以认为是稳定的。 建议只在根span上采用这种算法(in
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

[以下待翻译]()

|Parent| parent.isRemote() | parent.IsSampled()| Invoke sampler| |--|--|--|--| |absent| n/a | n/a |`root()`|
|present|true|true|`remoteParentSampled()`| |present|true|false|`remoteParentNotSampled()`|
|present|false|true|`localParentSampled()`| |present|false|false|`localParentNotSampled()`|

## Span Limits

Erroneous code can add unintended attributes, events, and links to a span. If these collections are unbounded, they can
quickly exhaust available memory, resulting in crashes that are difficult to recover from safely.

To protect against such errors, SDK Spans MAY discard attributes, links, and events that would increase the number of
elements of each collection beyond the configured limit.

It the SDK implements the limits above it MUST provide a way to change these limits, via a configuration to the
TracerProvider, by allowing users to configure individual limits like in the Java example bellow.

The name of the configuration options SHOULD be `AttributeCountLimit`,
`EventCountLimit` and `LinkCountLimit`. The options MAY be bundled in a class, which then SHOULD be called `SpanLimits`.
Implementations MAY provide additional configuration such as `AttributePerEventCountLimit`
and `AttributePerLinkCountLimit`.

```java
public final class SpanLimits {
    SpanLimits(int attributeCountLimit, int linkCountLimit, int eventCountLimit);

    public int getAttributeCountLimit();

    public int getEventCountLimit();

    public int getLinkCountLimit();
}
```

**Configurable parameters:**

* `AttributeCountLimit` (Default=128) - Maximum allowed span attribute count;
* `EventCountLimit` (Default=128) - Maximum allowed span event count;
* `LinkCountLimit` (Default=128) - Maximum allowed span link count;
* `AttributePerEventCountLimit` (Default=128) - Maximum allowed attribute per span event count;
* `AttributePerLinkCountLimit` (Default=128) - Maximum allowed attribute per span link count;

There SHOULD be a log emitted to indicate to the user that an attribute, event, or link was discarded due to such a
limit. To prevent excessive logging, the log should not be emitted once per span, or per discarded attribute, event, or
links.

## Id Generators

The SDK MUST by default randomly generate both the `TraceId` and the `SpanId`.

The SDK MUST provide a mechanism for customizing the way IDs are generated for both the `TraceId` and the `SpanId`.

The SDK MAY provide this functionality by allowing custom implementations of an interface like the java example below (
name of the interface MAY be
`IdGenerator`, name of the methods MUST be consistent with
[SpanContext](./api.md#retrieving-the-traceid-and-spanid)), which provides extension points for two methods, one to
generate a `SpanId` and one for `TraceId`.

```java
public interface IdGenerator {
    byte[] generateSpanIdBytes();

    byte[] generateTraceIdBytes();
}
```

Additional `IdGenerator` implementing vendor-specific protocols such as AWS X-Ray trace id generator MUST NOT be
maintained or distributed as part of the Core OpenTelemetry repositories.

## Span processor

Span processor is an interface which allows hooks for span start and end method invocations. The span processors are
invoked only when
[`IsRecording`](api.md#isrecording) is true.

Built-in span processors are responsible for batching and conversion of spans to exportable representation and passing
batches to exporters.

Span processors can be registered directly on SDK `TracerProvider` and they are invoked in the same order as they were
registered.

Each processor registered on `TracerProvider` is a start of pipeline that consist of span processor and optional
exporter. SDK MUST allow to end each pipeline with individual exporter.

SDK MUST allow users to implement and configure custom processors and decorate built-in processors for advanced
scenarios such as tagging or filtering.

The following diagram shows `SpanProcessor`'s relationship to other components in the SDK:

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

### Interface definition

#### OnStart

`OnStart` is called when a span is started. This method is called synchronously on the thread that started the span,
therefore it should not block or throw exceptions.

**Parameters:**

* `span` - a [read/write span object](#additional-span-interfaces) for the started span. It SHOULD be possible to keep a
  reference to this span object and updates to the span SHOULD be reflected in it. For example, this is useful for
  creating a SpanProcessor that periodically evaluates/prints information about all active span from a background
  thread.
* `parentContext` - the parent `Context` of the span that the SDK determined
  (the explicitly passed `Context`, the current `Context` or an empty `Context`
  if that was explicitly requested).

**Returns:** `Void`

#### OnEnd(Span)

`OnEnd` is called after a span is ended (i.e., the end timestamp is already set). This method MUST be called
synchronously within the [`Span.End()` API](api.md#end), therefore it should not block or throw an exception.

**Parameters:**

* `Span` - a [readable span object](#additional-span-interfaces) for the ended span. Note: Even if the passed Span may
  be technically writable, since it's already ended at this point, modifying it is not allowed.

**Returns:** `Void`

#### Shutdown()

Shuts down the processor. Called when SDK is shut down. This is an opportunity for processor to do any cleanup required.

`Shutdown` SHOULD be called only once for each `SpanProcessor` instance. After the call to `Shutdown`, subsequent calls
to `OnStart`, `OnEnd`, or `ForceFlush`
are not allowed. SDKs SHOULD ignore these calls gracefully, if possible.

`Shutdown` SHOULD provide a way to let the caller know whether it succeeded, failed or timed out.

`Shutdown` MUST include the effects of `ForceFlush`.

`Shutdown` SHOULD complete or abort within some timeout. `Shutdown` can be implemented as a blocking API or an
asynchronous API which notifies the caller via a callback or an event. OpenTelemetry client authors can decide if they
want to make the shutdown timeout configurable.

#### ForceFlush()

Exports all spans that have not yet been exported to the configured `Exporter`.

`ForceFlush` SHOULD provide a way to let the caller know whether it succeeded, failed or timed out.

`ForceFlush` SHOULD only be called in cases where it is absolutely necessary, such as when using some FaaS providers
that may suspend the process after an invocation, but before the `Processor` exports the completed spans.

`ForceFlush` SHOULD complete or abort within some timeout. `ForceFlush` can be implemented as a blocking API or an
asynchronous API which notifies the caller via a callback or an event. OpenTelemetry client authors can decide if they
want to make the flush timeout configurable.

### Built-in span processors

The standard OpenTelemetry SDK MUST implement both simple and batch processors, as described below. Other common
processing scenarios should be first considered for implementation out-of-process
in [OpenTelemetry Collector](../overview.md#collector)

#### Simple processor

This is an implementation of `SpanProcessor` which passes finished spans and passes the export-friendly span data
representation to the configured
`SpanExporter`, as soon as they are finished.

**Configurable parameters:**

* `exporter` - the exporter where the spans are pushed.

#### Batching processor

This is an implementation of the `SpanProcessor` which create batches of finished spans and passes the export-friendly
span data representations to the configured `SpanExporter`.

**Configurable parameters:**

* `exporter` - the exporter where the spans are pushed.
* `maxQueueSize` - the maximum queue size. After the size is reached spans are dropped. The default value is `2048`.
* `scheduledDelayMillis` - the delay interval in milliseconds between two consecutive exports. The default value
  is `5000`.
* `exportTimeoutMillis` - how long the export can run before it is cancelled. The default value is `30000`.
* `maxExportBatchSize` - the maximum batch size of every export. It must be smaller or equal to `maxQueueSize`. The
  default value is `512`.

## Span Exporter

`Span Exporter` defines the interface that protocol-specific exporters must implement so that they can be plugged into
OpenTelemetry SDK and support sending of telemetry data.

The goal of the interface is to minimize burden of implementation for protocol-dependent telemetry exporters. The
protocol exporter is expected to be primarily a simple telemetry data encoder and transmitter.

### Interface Definition

The exporter must support two functions: **Export** and **Shutdown**. In strongly typed languages typically there will
be 2 separate `Exporter`
interfaces, one that accepts spans (SpanExporter) and one that accepts metrics
(MetricsExporter).

#### `Export(batch)`

Exports a batch of [readable spans](#additional-span-interfaces). Protocol exporters that will implement this function
are typically expected to serialize and transmit the data to the destination.

Export() will never be called concurrently for the same exporter instance. Export() can be called again only after the
current call returns.

Export() MUST NOT block indefinitely, there MUST be a reasonable upper limit after which the call must time out with an
error result (`Failure`).

Any retry logic that is required by the exporter is the responsibility of the exporter. The default SDK SHOULD NOT
implement retry logic, as the required logic is likely to depend heavily on the specific protocol and backend the spans
are being sent to.

**Parameters:**

batch - a batch of [readable spans](#additional-span-interfaces). The exact data type of the batch is language specific,
typically it is some kind of list, e.g. for spans in Java it will be typically `Collection<SpanData>`.

**Returns:** ExportResult:

ExportResult is one of:

* `Success` - The batch has been successfully exported. For protocol exporters this typically means that the data is
  sent over the wire and delivered to the destination server.
* `Failure` - exporting failed. The batch must be dropped. For example, this can happen when the batch contains bad data
  and cannot be serialized.

Note: this result may be returned via an async mechanism or a callback, if that is idiomatic for the language
implementation.

#### `Shutdown()`

Shuts down the exporter. Called when SDK is shut down. This is an opportunity for exporter to do any cleanup required.

`Shutdown` should be called only once for each `Exporter` instance. After the call to `Shutdown` subsequent calls
to `Export` are not allowed and should return a `Failure` result.

`Shutdown` should not block indefinitely (e.g. if it attempts to flush the data and the destination is unavailable).
OpenTelemetry client authors can decide if they want to make the shutdown timeout configurable.

### Further Language Specialization

Based on the generic interface definition laid out above library authors must define the exact interface for the
particular language.

Authors are encouraged to use efficient data structures on the interface boundary that are well suited for fast
serialization to wire formats by protocol exporters and minimize the pressure on memory managers. The latter typically
requires understanding of how to optimize the rapidly-generated, short-lived telemetry data structures to make life
easier for the memory manager of the specific language. General recommendation is to minimize the number of allocations
and use allocation arenas where possible, thus avoiding explosion of allocation/deallocation/collection operations in
the presence of high rate of telemetry data generation.

#### Examples

These are examples on what the `Exporter` interface can look like in specific languages. Examples are for illustration
purposes only. OpenTelemetry client authors are free to deviate from these provided that their design remain true to the
spirit of `Exporter` concept.

##### Go SpanExporter Interface

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

##### Java SpanExporter Interface

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