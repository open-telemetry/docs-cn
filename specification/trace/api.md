# Tracing API

**状态**: [功能冻结](../document-status.md).

<details>
<summary>
Table of Contents
</summary>

* [Data types](#data-types)
  * [Time](#time)
    * [Timestamp](#timestamp)
    * [Duration](#duration)
* [TracerProvider](#tracerprovider)
  * [TracerProvider operations](#tracerprovider-operations)
* [Context Interaction](#context-interaction)
* [Tracer](#tracer)
  * [Tracer operations](#tracer-operations)
* [SpanContext](#spancontext)
  * [Retrieving the TraceId and SpanId](#retrieving-the-traceid-and-spanid)
  * [IsValid](#isvalid)
  * [IsRemote](#isremote)
* [Span](#span)
  * [Span creation](#span-creation)
    * [Determining the Parent Span from a Context](#determining-the-parent-span-from-a-context)
    * [Specifying Links](#specifying-links)
  * [Span operations](#span-operations)
    * [Get Context](#get-context)
    * [IsRecording](#isrecording)
    * [Set Attributes](#set-attributes)
    * [Add Events](#add-events)
    * [Set Status](#set-status)
    * [UpdateName](#updatename)
    * [End](#end)
    * [Record Exception](#record-exception)
  * [Span lifetime](#span-lifetime)
  * [Wrapping a SpanContext in a Span](#wrapping-a-spancontext-in-a-span)
* [SpanKind](#spankind)
* [Concurrency](#concurrency)
* [Included Propagators](#included-propagators)

</details>

 Tracing API 由以下三种类组成:

- [`TracerProvider`](#tracerprovider) 是 API 的入口点 (Entry Point) 。
  它提供对 Tracers 的访问。
- [`Tracer`](#tracer) 是负责创建 Spans 的类。
- [`Span`](#span) 是跟踪操作的 API。

## 数据类型

不同编程语言和平台有不同的数据表示方式。
本节定义了 Tracing API 的一些通用规定。

### 时间 Time

OpenTelemetry 可以处理精度为纳秒的时间值。
The representation of those values is language specific.

#### 时间戳 Timestamp

一个时间戳是指从 Unix 纪元 (Epoch) 以来所经过的时间量。

* 最小时间精度为毫秒 (milliseconds)。
* 最大时间精度为纳秒 (nanoseconds)。

#### 时长 Duration

一段时长是指两个事件间经过的时间量。

* 最小时间精度为毫秒。
* 最大时间精度为纳秒。

## TracerProvider

``TracerProvider`` 用于访问  `Tracer` 。

在本 API 实现中， `TracerProvider` 应当是一个有状态的对象，且可以容纳任意配置。

通常而言， `TracerProvider`  应当可以通过控制平面进行访问。因此，建议 API 提供一种可以设置/注册和访问全局默认 `TracerProvider` 的方法。

然而在全局 `TracerProvider` 的情况下，一些软件可能还是希望使用多个 `TracerProvider` 实例，
例如：为每个实例加载不同配置（例如 `SpanProcessor`），或更好地与依赖注入框架进行协作

to have different configuration (like `SpanProcessor`s) for each
(and consequently for the `Tracer`s obtained from them),
or because its easier with dependency injection frameworks.
因此，`TracerProvider` 应当允许创建任意数量的 `TracerProvider` 实例。

### TracerProvider 操作 TracerProvider operations

 `TracerProvider` 必须包含提供以下 API：

- 获得一个 `Trace` 

#### 获得一个 Trace

本 API 必须接受以下参数：

- `name` (required):  该输入必须是 [instrumentation library](../overview.md#instrumentation-libraries) 的名称 (identify) 
  (e.g. `io.opentelemetry.contrib.mongodb`)，而不是 instrumented library.
  如果指定了一个无效的名称（空或者空字符串），将会返回一个可工作的默认 Trace ，而不是返回 null 或者抛出异常。
  
  当一个实现 OpenTelemetry API 的库不支持“命名”功能时，该库可以忽略该命名，并对所有调用返回一个默认实例。(例如，一个甚至于可观察性无关的实例)。
  TracerProvider 也可以返回一个无操作 (no-op) Tracer，当应用程序所有者配置了 SDK 来抑制该库产生遥感数据。
  
- `version` (optional): Instrumentation library 的版本 (e.g. `1.0.0`).

该接口不确保在相同/不同的情况下，返回相同/不同的 `Trace` 实例。

该接口的实现禁止要求用户通过使用相同的名称+版本参数，重复获取 `Tracer`，接受 配置的变更。

这意味着使用过时的配置或通过确保新的配置，都可以获得到之前返回的 `Tracer`s

注: 这是可行的，例如，在 `TracerProvider` 中实现存储可变化的配置，同时实现 `Tracer`实现一个对 `TracerProvider` 引用的对象。如果配置必须按照每个 Tracer 存储（如禁止某个 Tracer），可以在 `TracerProvider` 中实现一个 name+version 的 map，或者实现一个注册表，用于存储所有返回的 `Tracer` 。当配置发生改变时进行主动更新。

## Context Interaction

本节定义了 Tracing API 与[`Context`](../context/context.md) 交互的所有操作。

API 必须提供以下功能来与 `Context` 实例进行交互:

- 提取 `Span` 从一个 `Context` 实例中
- 插入 `Span` 从一个`Context` 实例中

以上罗列的功能是必要的，因为 API 用户不应当使用 [Context Key](../context/context.md#create-a-key)  访问 Tracing API 的实现。

如果编程语言支持隐性传递 `Context` (see [here](../context/context.md#optional-global-operations))，本 API 应当也提供以下功能。

- 获得当前活跃 span  (隐式上下文中）。这等价于获得先获得隐式 Context，然后从上下文中提取 Span。
- 设置当前活跃 span（向隐式上下文中）。这等价于获得先获得隐式 Context，然后从上下文中提取 Span。

上述所有功能只能在 context API 中运行，他们可能作为 Trace 模块的静态方法，或者作为 Trace 模块内的类的静态方法暴露。这些功能都应当在 API 中完全实现。

## Tracer

 tracer 负责创建 `Span`s.

注意: `Tracers` 通常不负责进行配置，这是`TracerProvider` 的职责。

Tracer 操作

 `Tracer` 必须包含以下功能：

- [创建新的 `Span`](#span-creation) (请看 `Span` 章节)

## SpanContext

`SpanContext` 作为表示 `Span` 的一部分，他必须可以进行序列化，并沿着分布式上下文进行传播。`SpanContext`s 是不可变的。

OpenTelemetry `SpanContext` 符合 [W3C TraceContext 规范](https://www.w3.org/TR/trace-context/)。这包含两个标识符 - `TraceId` 和 `SpanId` - 一套通用 `TraceFlags` 和 系统特定 `TraceState`。

`TraceId` 一个有效的 `TraceId` 是一个 16 字节的数组，且至少有一个非零字节。

`SpanId` 一个有效的 SpanId 是一个 8 字节的数组，且至少有一个非零字节。

`TraceFlags` 包含该 trace 的详情。不像 TraceState，TraceFlags 影响所有的 traces。当前版本和定义的 Flags 只有 sampled 。

`TraceState` 携带特定 trace 标识数据，通过一个 KV 对数组进行标识。TraceState允许多个跟踪系统参与同一个 Trace。完整定义请参考 [W3C Trace Context
specification](https://www.w3.org/TR/trace-context/#tracestate-header).

本 API 必须实现创建 `SpanContext` 的方法。这些方法应当是唯一的方法用于创建 `SpanContext`。这个功能必须在 API 中完全实现，并且不应当可以被覆盖。

### 检索 TraceId and SpanId

本 API 必须支持通过以下方式检索 `TraceId` 与 `SpanId`：

* Hex - 返回十六进制格式 `TraceID` （结果必须是一个 32 个 十六进制字符的小写字母）或 `SpanID` 结果必须是一个 16 个十六进制字符的小写字母）
* Binary - 返回二进制格式 `TraceId` （结果必须是 16 字节数组）或 SpanId（结果必须是8字节数组）。

API 不应该暴露它们内部存储细节。

### IsValid

一个名为 `IsValid` 的 API，返回一个布尔值，当 `SpanContext` 存在不为零的 TraceID 与 不为零的 SpanID 时，返回 `true`。该 API 必须提供。

### IsRemote

一个名为  `IsRemote` 的 API，返回一个布尔值，当 `SpanContext` 是由远端的父节点传播而来时，返回 `true`。该 API 必须提供。

当通过 [Propagators API](../context/api-propagators.md#propagators-api) 获取到 `SpanContext` 时，`IsRemote`  必须返回 `true`。对于 `SpanContext` 的所有 child spans， 必须返回 `false`。

### TraceState

`TraceState` 是 [`SpanContext`](./api.md#spancontext) 的一部分，有不可变的键值对列表表示，由 [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#tracestate-header) 正式定义。
Tracing API 必须在 `TraceState` 上至少提供以下操作：

* 获取指定 key 的 value
* 更新指定 key 已存在的 value
* 添加新的 key/value pair
* 删除 key/value pair

这些操作必须遵循 [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#mutating-the-tracestate-field) 中的定义规则。所有转变操作都必须返回新的修改生效的`TraceState`。`TraceState`  必须所有时候都按照  [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#tracestate-header-field-values) 规范中指定的规则进行验证。每个转变操作必须验证输出的参数。如果操作被传入了无效值，必须不能返回带有无效数据的 `TraceState`。并且必须遵循 [一般错误处理准则](../error-handling.md) 。（例如，通常不得返回 null 或抛出异常）

注意: 由于 `SpanContext` 不可变，所以不可能用新的 `TraceState` 更新 `SpanContext`。 因此这种更改只有在 [`SpanContext` 传播 ](../context/api-propagators.md)或[遥感数据导出](sdk.md#span-exporter)发生前进行才有意义。在这两种情况下， `Propagators` 和 `SpanExporters`  可能在序列化到线上之前，创建更改后的 `TraceState` 副本。

## Span

`Span` 表示一个 Trace 中的单个操作。Spans 可以以嵌套形式组成一颗 trace tree。每个 trace 包含一个 root span，他通常用于描述整个操作。同时可选择为子操作提供一或多个 sub-spans。

<a name="span-data-members"></a>
`Span`s 囊括:

- Span 名称
- 一个不可变的 [`SpanContext`](#spancontext) ，作为 `Span ` 的唯一标识。
- 一个父 span，以 [`Span`](#span), [`SpanContext`](#spancontext) 形式。也可能是 null。
-  [`SpanKind`](#spankind)
- 开始时间 start timestamp
- 结束时间 end timestamp
- [`Attributes`](../common/common.md#attributes)
- [`Link`s](#specifying-links)  列表，用于链接其他 `Span`s
- 带有时间戳的 [`Event`s](#add-events) 列表
- 一个 [`Status`](#set-status)

span 名称应当简单扼要地表明该 Span 的工作内容。

例如，一个 RPC 方法名，一个函数名，一个庞大计算任务中子任务或者阶段的名称。

span 名称应当是一种具有通用性字符串，便于后续的统计学处理。而不是单个 span 实例同时人的可读性。
也就是说，"get_user" 是一个合理的名词，而 "get_user/314159"，当中 "314159" 是一个用户 ID，这不是一个好的名字不具备高基数率 (high cardinality)。通用型应当优先于人的可读性。

例如，以下是获得账户信息的端点 API 备选跨度名称列表。

| Span Name                 | Guidance                                                     |
| ------------------------- | ------------------------------------------------------------ |
| `get`                     | 过于普遍                                                     |
| `get_account/42`          | 过于特殊                                                     |
| `get_account`             | 不错, 同时 account_id=42 可以使一个很不错的a nice Span attribute |
| `get_account/{accountId}` | 同样不错 (使用 "HTTP route")                                 |

`Span` 的开始和结束时间戳反应了操作的实际时间。

例如，一个 span 代表一个请求-响应周期（例如, HTTP 或 RPC），那这个跨度的开始时间应该和第一个子操作的开始时间相同，结束时间应该和最后一个子操作的完成时间相同。
这其中包括:

- 从请求中接收数据
- 解析数据（例如从二进制文件或 JSON 格式）
- 中间件或额外的逻辑处理
- 业务逻辑
- 构建响应
- 发送响应

可以通过创建 Child spans（或者在一些情况下）来更详细观察描述子操作。Child spans 应当衡量各个子操作的时间，并可以添加相应的属性。

Span 的开始时间应当设置为创建 Span 时的当前时间。Span 创建后应当可以更改名称，设置属性，添加事件和设置状态。在 Span 的结束时间被设置后，这些都不允许被改变。

`Span`s 没有在进程中传播的功能。为了防止被无用，实现中除了 `SpanContext` 外不应当提供对 Span 属性的访问。

广商可以通过实现 `Span` 接口来满足厂商自身特定的逻辑，然而其他实现严禁允许调用者直接创建 `Span`。所有的 `Span` 必须由 `Tracer` 创建

### 创建 Span

除了 [`Tracer`](#tracer)  外，不准许任何其他 API 创建 `Span` 。

`Span` 的创建不准许将新创建的 `Span` 默认作当前活跃的 `Span`，但该功能可以作为单独的操作提供。

API 必须接受以下参数：

- Span 名称。这是必须的参数。

- 父 `Context` 或者表明该新的 `Span` 是 `root Span`。
  
  API 需要提供一个选项，用于设置默认行为：将当前的 `Context` 作为父级。
  
  API 禁止接收 `Span` 或 `SpanContext` 作为父级，只能是完整的 `Context`。
  
  Span 的语义父级必须要个遵守  [Determining the Parent Span from a Context](#determining-the-parent-span-from-a-context) 中描述的规则。
  
- [`SpanKind`](#spankind)，默认值为: `SpanKind.Internal`。

- `[Attributes](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)`。此外，这些属性还可用于定义[取样详情](sdk.md#sampling)。如果没有指定，该字段将被假定是一个空的集合。只要有可能，使用者应该在创建跨度时设置相应属性，而不是在创建之后，调用SetAttribute。
- `Link`s - 一个有序的链接序列，详情见 [here](#specifying-links).
- `Start timestamp`，默认为当前时间。应当只能在创建时间已经发生的 Span 时才可以使用本参数。如果 API 在 Span 逻辑发生时被调用，API 使用者必须能设置该参数。


每个 span 都有零或一个父 span 和零或多个子 span，这用于记录操作的因果关系。Spans 的关联树构成了 Trace。如果一个 span 没有父 span，那它被定义成一个 *根 (root) span*。每个 Trace 有且只有一个的 root span，它是所有其他的 Trace 中的 span 的祖先。实现必须提供一个选项用于创建一个 `Span` 作为 root span，并且必须每次创建 root span 时生成一个新的 `TraceId`。对于有父 span 的 `Span`，`TraceId` 必须与父 span 相同。此外，子 span 必须默认继承其父 span 的所有 `TraceState` 值。

如果一个 `Span` 是被另一个进程中创建的 `Span` 的子代，那么它就被称为有一个 *remote parent*。每个传播者的反序列化时必须在父 `SpanContext` 上将 `IsRemote` 设置为 true，这样在 `Span` 的创建时就知道父 `Span` 是否是远程的。

任何被创建的 `Span` 也必须被结束。这是使用者的责任。如果使用者忘记结束 `Span`，API 实现可能会泄漏内存或其他资源（例如，适用于所有 `Span` 的周期性工作的 CPU 时间）。

#### 通过 Context 创建 Parent Span

当新的 Span 通过 Context 创建时，Context 可能已经包含一个代表当前活跃实例的 Span，并将其设置为新的 Span 的父 Span。如果 Context 中没有 Span，那么新创建的 Span 将是一个 Root Span。

`SpanContext` 不能被直接设置成活跃状态，而是通过 [包装成 Span](#wrapping-a-spancontext-in-a-span)。

例如：提取 context 的 `Propagator` 可能需要这个。

#### 指定链接

在创建 `Span` 时，用户必须能够记录与其他 `Span`s 的链接。链接的 `Span` 可以来自相同或不同的 `Trace`。请看链接的[描述](../overview.md#links-between-spans)。Span 创建后不能添加链接。

一个 `Link` 由一下属性组成。

- `SpanContext` ，要链接的 `Span` 所属的 `SpanContext`。
- 零或多个 [`属性`](../common/common.md#attributes) 用于描述链接。

Span 创建 API 必须提供：

- 一个用于记录单个链接的 API，链接的属性可以作为参数进行传递。这可以被称为 `AddLink`。这个 API 需要参数: `SpanContext` （需要被链接的 `Span`）和可选的 `Attributes`，可以是单独的参数，也可以是封装它们的不可变对象，以各自编程语言自身最合适的方式为准。

链接应该保留其设置的顺序。

### Span 操作

除了从 `SpanContext` 取出 `Span` 与记录状态的操作外，其他操作都不能在 `Span` 结束后被调用

#### 获得 Context

Span 接口必须提供：

- 返回 `SpanContext`。一个 API 返回指定的 `Span` 的 `SpanContext`。即使在 Span 结束后，该接口也可以使用。返回的值必须整个 Span 寿命周期内保持不变。可以被称为 `GetContext` 。

#### IsRecording

返回 `true` 当该 `Span` 正在记录信息，例如使用 `AddEvent` 操作的事件，使用 `SetAttributes` 的属性，使用 `SetStatus` 的状态等。

Returns true if this `Span` is recording information like events with the
`AddEvent` operation, attributes using `SetAttributes`, status with `SetStatus`,
etc.

当 `Span` 结束后，通常会变成非记录状态，因此对结束的 Span，`IsRecording`应当返回 `false`。注意：流数据的实现时，它不知道一个 `Span` 是否结束，这是一个预期的情况。`IsRecording` 在 `Span` 结束后无法被修改

`IsRecording` 不应当接受任何参数。

这个标记应当用来避免在 Span 没记录时，处理 Span 属性和事件（这两个操作是昂贵的计算）。注意任何子 span 的记录标记都是独立于父 span 的标记来决定的（通常基于 [SpanContext](#spancontext) 上的 TraceFlags 中的采样标记）。

这个标签可能是 `true`， 当整个事件再被采样的时候。这允许不需要发送到后端即可记录和处理单个 `Span` 的信息。这个情况的一个例子可能是记录和处理所有的传入请求，用于创建 SLA/SLO 延迟图，同事只向后端发送一个子集（采样的子集）。详情参见 [sampling section of SDK design](sdk.md#sampling)。

API 的使用者应当通过 instrumenting 代码访问 IsRecording 属性，除非在 context 传播器中使用，否则永远不要访问SampledFlag。

#### Set Attributes

A `Span` MUST have the ability to set [`Attributes`](../common/common.md#attributes) associated with it.

The Span interface MUST provide:

- An API to set a single `Attribute` where the attribute properties are passed
  as arguments. This MAY be called `SetAttribute`. To avoid extra allocations some
  implementations may offer a separate API for each of the possible value types.

Setting an attribute with the same key as an existing attribute SHOULD overwrite
the existing attribute's value.

Note that the OpenTelemetry project documents certain ["standard
attributes"](semantic_conventions/README.md) that have prescribed semantic meanings.

Note that [Samplers](sdk.md#sampler) can only consider information already
present during span creation. Any changes done later, including new or changed
attributes, cannot change their decisions.

#### Add Events

A `Span` MUST have the ability to add events. Events have a time associated
with the moment when they are added to the `Span`.

An `Event` is structurally defined by the following properties:

- Name of the event.
- A timestamp for the event. Either the time at which the event was
  added or a custom timestamp provided by the user.
- Zero or more [`Attributes`](../common/common.md#attributes) further describing
  the event.

The Span interface MUST provide:

- An API to record a single `Event` where the `Event` properties are passed as
  arguments. This MAY be called `AddEvent`.
  This API takes the name of the event, optional `Attributes` and an optional
  `Timestamp` which can be used to specify the time at which the event occurred,
  either as individual parameters or as an immutable object encapsulating them,
  whichever is most appropriate for the language. If no custom timestamp is
  provided by the user, the implementation automatically sets the time at which
  this API is called on the event.

Events SHOULD preserve the order in which they are recorded.
This will typically match the ordering of the events' timestamps,
but events may be recorded out-of-order using custom timestamps.

Consumers should be aware that an event's timestamp might be before the start or
after the end of the span if custom timestamps were provided by the user for the
event or when starting or ending the span.
The specification does not require any normalization if provided timestamps are
out of range.

Note that the OpenTelemetry project documents certain ["standard event names and
keys"](semantic_conventions/README.md) which have prescribed semantic meanings.

Note that [`RecordException`](#record-exception) is a specialized variant of
`AddEvent` for recording exception events.

#### Set Status

Sets the `Status` of the `Span`. If used, this will override the default `Span`
status, which is `Unset`.

`Status` is structurally defined by the following properties:

- `StatusCode`, one of the values listed below.
- Optional `Description` that provides a descriptive message of the `Status`.
  `Description` MUST only be used with the `Error` `StatusCode` value.

`StatusCode` is one of the following values:

- `Unset`
  - The default status.
- `Ok`
  - The operation has been validated by an Application developer or Operator to
    have completed successfully.
- `Error`
  - The operation contains an error.

The Span interface MUST provide:

- An API to set the `Status`. This SHOULD be called `SetStatus`. This API takes
  the `StatusCode`, and an optional `Description`, either as individual
  parameters or as an immutable object encapsulating them, whichever is most
  appropriate for the language. `Description` MUST be IGNORED for `StatusCode`
  `Ok` & `Unset` values.

The status code SHOULD remain unset, except for the following circumstances:

When the status is set to `ERROR` by Instrumentation Libraries, the status codes
SHOULD be documented and predictable. The status code should only be set to `ERROR`
according to the rules defined within the semantic conventions. For operations
not covered by the semantic conventions, Instrumentation Libraries SHOULD
publish their own conventions, including status codes.

Generally, Instrumentation Libraries SHOULD NOT set the status code to `Ok`,
unless explicitly configured to do so. Instrumention libraries SHOULD leave the
status code as `Unset` unless there is an error, as described above.

Application developers and Operators may set the status code to `Ok`.

Analysis tools SHOULD respond to an `Ok` status by suppressing any errors they
would otherwise generate. For example, to suppress noisy errors such as 404s.

Only the value of the last call will be recorded, and implementations are free
to ignore previous calls.

#### UpdateName

Updates the `Span` name. Upon this update, any sampling behavior based on `Span`
name will depend on the implementation.

Note that [Samplers](sdk.md#sampler) can only consider information already
present during span creation. Any changes done later, including updated span
name, cannot change their decisions.

Alternatives for the name update may be late `Span` creation, when Span is
started with the explicit timestamp from the past at the moment where the final
`Span` name is known, or reporting a `Span` with the desired name as a child
`Span`.

Required parameters:

- The new **span name**, which supersedes whatever was passed in when the
  `Span` was started

#### End

Signals that the operation described by this span has
now (or at the time optionally specified) ended.

Implementations SHOULD ignore all subsequent calls to `End` and any other Span methods,
i.e. the Span becomes non-recording by being ended
(there might be exceptions when Tracer is streaming events
and has no mutable state associated with the `Span`).

Language SIGs MAY provide methods other than `End` in the API that also end the
span to support language-specific features like `with` statements in Python.
However, all API implementations of such methods MUST internally call the `End`
method and be documented to do so.

`End` MUST NOT have any effects on child spans.
Those may still be running and can be ended later.

`End` MUST NOT inactivate the `Span` in any `Context` it is active in.
It MUST still be possible to use an ended span as parent via a Context it is
contained in. Also, any mechanisms for putting the Span into a Context MUST
still work after the Span was ended.

Parameters:

- (Optional) Timestamp to explicitly set the end timestamp.
  If omitted, this MUST be treated equivalent to passing the current time.

This API MUST be non-blocking.

#### Record Exception

To facilitate recording an exception languages SHOULD provide a
`RecordException` method if the language uses exceptions.
This is a specialized variant of [`AddEvent`](#add-events),
so for anything not specified here, the same requirements as for `AddEvent` apply.

The signature of the method is to be determined by each language
and can be overloaded as appropriate.
The method MUST record an exception as an `Event` with the conventions outlined in
the [exception semantic conventions](semantic_conventions/exceptions.md) document.
The minimum required argument SHOULD be no more than only an exception object.

If `RecordException` is provided, the method MUST accept an optional parameter
to provide any additional event attributes
(this SHOULD be done in the same way as for the `AddEvent` method).
If attributes with the same name would be generated by the method already,
the additional attributes take precedence.

Note: `RecordException` may be seen as a variant of `AddEvent` with
additional exception-specific parameters and all other parameters being optional
(because they have defaults from the exception semantic convention).

### Span lifetime

Span lifetime represents the process of recording the start and the end
timestamps to the Span object:

- The start time is recorded when the Span is created.
- The end time needs to be recorded when the operation is ended.

Start and end time as well as Event's timestamps MUST be recorded at a time of a
calling of corresponding API.

### Wrapping a SpanContext in a Span

The API MUST provide an operation for wrapping a `SpanContext` with an object
implementing the `Span` interface. This is done in order to expose a `SpanContext`
as a `Span` in operations such as in-process `Span` propagation.

If a new type is required for supporting this operation, it SHOULD not be exposed
publicly if possible (e.g. by only exposing a function that returns something
with the Span interface type). If a new type is required to be publicly exposed,
it SHOULD be named `NonRecordingSpan`.

The behavior is defined as follows:

- `GetContext()` MUST return the wrapped `SpanContext`.
- `IsRecording` MUST return `false` to signal that events, attributes and other elements
  are not being recorded, i.e. they are being dropped.

The remaining functionality of `Span` MUST be defined as no-op operations.
Note: This includes `End`, so as an exception from the general rule,
it is not required (or even helpful) to end such a Span.

This functionality MUST be fully implemented in the API, and SHOULD NOT be overridable.

## SpanKind

`SpanKind` describes the relationship between the Span, its parents,
and its children in a Trace.  `SpanKind` describes two independent
properties that benefit tracing systems during analysis.

The first property described by `SpanKind` reflects whether the Span
is a remote child or parent.  Spans with a remote parent are
interesting because they are sources of external load.  Spans with a
remote child are interesting because they reflect a non-local system
dependency.

The second property described by `SpanKind` reflects whether a child
Span represents a synchronous call.  When a child span is synchronous,
the parent is expected to wait for it to complete under ordinary
circumstances.  It can be useful for tracing systems to know this
property, since synchronous Spans may contribute to the overall trace
latency. Asynchronous scenarios can be remote or local.

In order for `SpanKind` to be meaningful, callers should arrange that
a single Span does not serve more than one purpose.  For example, a
server-side span should not be used directly as the parent of another
remote span.  As a simple guideline, instrumentation should create a
new Span prior to extracting and serializing the SpanContext for a
remote call.

These are the possible SpanKinds:

* `SERVER` Indicates that the span covers server-side handling of a
  synchronous RPC or other remote request.  This span is the child of
  a remote `CLIENT` span that was expected to wait for a response.
* `CLIENT` Indicates that the span describes a synchronous request to
  some remote service.  This span is the parent of a remote `SERVER`
  span and waits for its response.
* `PRODUCER` Indicates that the span describes the parent of an
  asynchronous request.  This parent span is expected to end before
  the corresponding child `CONSUMER` span, possibly even before the
  child span starts. In messaging scenarios with batching, tracing
  individual messages requires a new `PRODUCER` span per message to
  be created.
* `CONSUMER` Indicates that the span describes the child of an
  asynchronous `PRODUCER` request.
* `INTERNAL` Default value. Indicates that the span represents an
  internal operation within an application, as opposed to an
  operations with remote parents or children.

To summarize the interpretation of these kinds:

| `SpanKind` | Synchronous | Asynchronous | Remote Incoming | Remote Outgoing |
| ---------- | ----------- | ------------ | --------------- | --------------- |
| `CLIENT`   | yes         |              |                 | yes             |
| `SERVER`   | yes         |              | yes             |                 |
| `PRODUCER` |             | yes          |                 | maybe           |
| `CONSUMER` |             | yes          | maybe           |                 |
| `INTERNAL` |             |              |                 |                 |

## Concurrency

For languages which support concurrent execution the Tracing APIs provide
specific guarantees and safeties. Not all of API functions are safe to
be called concurrently.

**TracerProvider** - all methods are safe to be called concurrently.

**Tracer** - all methods are safe to be called concurrently.

**Span** - All methods of Span are safe to be called concurrently.

**Event** - Events are immutable and safe to be used concurrently.

**Link** - Links are immutable and safe to be used concurrently.

## Included Propagators

The API layer or an extension package MUST include the following `Propagator`s:

* A `TextMapPropagator` implementing the [W3C TraceContext Specification](https://www.w3.org/TR/trace-context/).

See [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
for how propagators are to be distributed.

## Behavior of the API in the absence of an installed SDK

In general, in the absence of an installed SDK, the Trace API is a "no-op" API.
This means that operations on a Tracer, or on Spans, should have no side effects and do nothing. However, there
is one important exception to this general rule, and that is related to propagation of a `SpanContext`:
The API MUST create a [non-recording Span](#wrapping-a-spancontext-in-a-span) with the `SpanContext`
that is in the `Span` in the parent `Context` (whether explicitly given or implicit current) or,
if the parent is a non-recording Span (which it usually always is if no SDK is present),
it MAY return the parent Span back from the creation method.
If the parent `Context` contains no `Span`, an empty non-recording Span MUST be returned instead
(i.e., having a `SpanContext` with all-zero Span and Trace IDs, empty Tracestate, and unsampled TraceFlags).
This means that a `SpanContext` that has been provided by a configured `Propagator`
will be propagated through to any child span and ultimately also `Inject`,
but that no new `SpanContext`s will be created.