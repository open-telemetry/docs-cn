# Tracing API

**状态**: [功能冻结](../document-status.md).

<details>
<summary>
Table of Contents
</summary>

* [数据类型](#数据类型)
  * [时间](#time)
    * [时间戳](#时间戳-timestamp)
    * [时长](#时长-duration)
* [TracerProvider](#tracerprovider)
  * [TracerProvider 操作](#tracerprovider-operations)
* [Context Interaction](#context-interaction)
* [Tracer](#tracer)
  * [Tracer operations](#tracer-operations)
* [SpanContext](#spancontext)
  * [检索 TraceId 与 SpanId](#检索-traceid-与-spanid)
  * [IsValid](#isvalid)
  * [IsRemote](#isremote)
  * [TraceState](#TraceState)
* [Span](#span)
  * [创建 Span](#创建-Span)
    * [通过 Context 创建父 Span](#通过-Context-创建父-Span)
    * [指定链接](#指定链接)
  * [Span 操作](#span-operations)
    * [获得 Context](#获得-context)
    * [IsRecording](#isrecording)
    * [设置属性](#设置属性-Set-Attributes)
    * [新增事件](#新增事件-Add-Events)
    * [设置状态](#设置状态)
    * [更新名称](#更新名称)
    * [结束](#结束)
    * [记录异常](#记录异常)
  * [Span 生命周期](#span-生命周期)
  * [用 Span 包装 SpanContext](#用-Span-包装-SpanContext)
* [跨度种类](#跨度种类-SpanKind)
* [并发性](#并发性)
* [包含传播者](#包含传播者-Included-Propagator)

</details>

Tracing API 由以下三种类组成:

- [`TracerProvider`](#tracerprovider) 是 API 的入口点 (Entry Point) 。
  它提供对 Tracers 的访问。
- [`Tracer`](#tracer) 是负责创建 Spans 的类。
- [`Span`](#span) 是跟踪操作的 API。

## 数据类型

不同编程语言和平台有不同的数据表示方式，本节定义了 Tracing API 的一些通用规定。

### 时间 Time

OpenTelemetry 可以处理精度为纳秒(ns)的时间值。这些值的表现方式是语言特定的。

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

通常而言， 应该从中心位置访问 `TracerProvider` 。因此，建议 API 提供一种可以设置/注册和访问全局默认 `TracerProvider` 的方法。

然而在全局 `TracerProvider` 的情况下，一些应用可能还是希望使用多个 `TracerProvider` 实例，
例如：为每个实例加载不同配置（例如 `SpanProcessor`），或更好地与依赖注入框架进行协作

to have different configuration (like `SpanProcessor`s) for each
(and consequently for the `Tracer`s obtained from them),
or because its easier with dependency injection frameworks.
因此，`TracerProvider` 应当允许创建任意数量的 `TracerProvider` 实例。

### TracerProvider 操作

`TracerProvider` 必须包含提供以下 API：

- 获得一个 `Trace`

#### 获得一个 Trace

本 API 必须接受以下参数：

- `name` (required):  该值必须是 [instrumentation library](../overview.md#instrumentation-libraries) 的标识
  (例如 `io.opentelemetry.contrib.mongodb`)，而不是 instrumented library 的标识.
  如果指定了一个无效的名称（空或者空字符串），将会返回一个可工作的默认 Trace ，而不是返回 null 或者抛出异常。

  当一个实现 OpenTelemetry API 的库不支持“命名”功能时，该库可以忽略该命名，并对所有调用返回一个默认实例。(例如，一个甚至与可观察性无关的实例)。
  TracerProvider 也可以返回一个无操作 (no-op) Tracer，当应用程序所有者配置了 SDK 来抑制该库产生遥感数据。

- `version` (optional): Instrumentation library 的版本 (例如 `1.0.0`).

该接口不确保在相同/不同的情况下，返回相同/不同的 `Trace` 实例。

该接口的实现禁止要求用户通过使用相同的名称+版本参数重复获取 `Tracer`来接收配置的变更。
这可以通过允许过时的配置继续工作或者确保新配置也适用于之前返回的 `Tracer`来实现。

注: 这是可行的，例如，在 `TracerProvider` 中存储可变化的配置，同时从这个 `TracerProvider` 生成的 `Tracer` 对象持有该 `TracerProvider` 的引用。
如果配置必须按照每个 Tracer 存储（如禁止某个 Tracer），则 `Tracer` 可以通过name+version在 `TracerProvider` 提供的map中查找，
或者在 `TracerProvider` 中维护一个包含所有返回的 `Tracer`的注册表 ，并在配置发生改变时进行主动更新。

## Context Interaction

本节定义了 Tracing API 与[`Context`](../context/context.md) 交互的所有操作。

API 必须提供以下功能来与 `Context` 实例进行交互:

- 提取 `Span` 从一个 `Context` 实例中
- 插入 `Span` 从一个`Context` 实例中

以上罗列的功能是必要的，因为 API 用户不应当通过 Tracing API 的实现访问 [Context Key](../context/context.md#create-a-key)。

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

`TraceFlags` 包含该 trace 的详情。不像 TraceState，TraceFlags 影响所有的 traces。当前版本和定义的 Flags 只有 [sampled](https://www.w3.org/TR/trace-context/#sampled-flag) 。

`TraceState` 携带特定 trace 标识数据，通过一个 KV 对数组进行标识。TraceState允许多个跟踪系统参与同一个 Trace。完整定义请参考 [W3C Trace Context
specification](https://www.w3.org/TR/trace-context/#tracestate-header) 。

本 API 必须实现创建 `SpanContext` 的方法。这些方法应当是唯一的方法用于创建 `SpanContext`。这个功能必须在 API 中完全实现，并且不应当可以被覆盖。

### 检索 TraceId 与 SpanId

本 API 必须支持通过以下方式检索 `TraceId` 与 `SpanId`：

* Hex - 返回十六进制格式 `TraceID` （结果必须是一个 32 个 十六进制字符的小写字母）或 `SpanID` （结果必须是一个 16 个十六进制字符的小写字母）
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

这些操作必须遵循 [W3C Trace Context specification](https://www.w3.org/TR/trace-context/#mutating-the-tracestate-field) 中的定义规则。
所有转变操作都必须返回新的修改生效的`TraceState`。`TraceState` 必须所有时候都按照[W3C Trace Context specification](https://www.w3.org/TR/trace-context/#tracestate-header-field-values) 
规范中指定的规则进行验证。每个转变操作必须验证输出的参数。如果操作被传入了无效值，必须不能返回带有无效数据的 `TraceState`。
并且必须遵循 [一般错误处理准则](../error-handling.md) 。（例如，通常不得返回 null 或抛出异常）

注意: 由于 `SpanContext` 不可变，所以不可能用新的 `TraceState` 更新 `SpanContext`。 
因此这种更改只有在 [`SpanContext` 传播 ](../context/api-propagators.md)或[遥感数据导出](sdk.md#span-exporter)发生前进行才有意义。
在这两种情况下， `Propagators` 和 `SpanExporters`  可能在序列化到线上之前，创建更改后的 `TraceState` 副本。

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
span 名称应当是一种具有通用性字符串，便于后续的统计学处理。而不是单个 Span 实例，同时仍然是人类可读的。
也就是说，"get_user" 是一个合理的名称，而 "get_user/314159"，当中 "314159" 是一个用户 ID，这不是一个好的名称不具备高基数率 (high cardinality)。通用型应当优先于人的可读性。

例如，以下是获得账户信息的端点 API 备选跨度名称列表。

| Span Name                 | Guidance                                                     |
| ------------------------- | ------------------------------------------------------------ |
| `get`                     | 过于普遍                                                     |
| `get_account/42`          | 过于特殊                                                     |
| `get_account`             | 不错, account_id=42 可以作为 Span 的 attribute |
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

可以通过创建 Child spans（或者在一些情况下用 events）来更详细观察描述子操作。Child spans 应当衡量各个子操作的时间，并可以添加相应的属性。

Span 的开始时间应当设置为创建 Span 时的当前时间。Span 创建后应当可以更改名称，设置属性，添加事件和设置状态。在 Span 的结束时间被设置后，这些都不允许被改变。

`Span`s 没有在进程中传播的功能。为了防止被误用，实现中除了 `Span` 自己的 `SpanContext` 外不能访问到 `Span` 的属性。

厂商可以通过实现 `Span` 接口来满足厂商自身特定的逻辑，然而这些可供替代的实现禁止允许调用者直接创建 `Span`。所有的 `Span` 必须由 `Tracer` 创建。

### 创建 Span

除了 [`Tracer`](#tracer)  外，不准许任何其他 API 创建 `Span` 。

`Span` 的创建不准许将新创建的 `Span` 默认作当前活跃的 `Span`，但该功能可以作为单独的操作提供。

API 必须接受以下参数：

- Span 名称。这是必须的参数。

- 父 `Context` 或者表明该新的 `Span` 是 `root Span`。
  API 需要提供一个选项，用于设置默认行为：将当前的 `Context` 作为父级。
  API 禁止接收 `Span` 或 `SpanContext` 作为父级，只能是完整的 `Context`。
  
  Span 的语义父级必须遵守  [Determining the Parent Span from a Context](#determining-the-parent-span-from-a-context) 中描述的规则。
  
- [`SpanKind`](#spankind)，默认值为: `SpanKind.Internal`。

- [`Attributes`](../common/common.md#attributes)。此外，这些属性还可用于定义[取样详情](sdk.md#sampling)。如果没有指定，该字段将被假定是一个空的集合。
  
  只要有可能，使用者应该在创建跨度时设置相应属性，而不是在创建之后，调用 `SetAttribute` 。
  
- `Link`s - 一个有序的链接序列，详情见 [here](#specifying-links).
- `Start timestamp`，默认为当前时间。应当只能在创建时间已经发生的 Span 时才可以使用本参数。如果 API 在 Span 逻辑发生时被调用，API 使用者必须能设置该参数。


每个 span 都有零或一个父 span 和零或多个子 span，这用于记录操作的因果关系。Spans 的关联树构成了 Trace。如果一个 span 没有父 span，那它被定义成一个 *根 (root) span*。
每个 Trace 有且只有一个的 root span，它是所有其他的 Trace 中的 span 的祖先。实现必须提供一个选项用于创建一个 `Span` 作为 root span，并且必须每次创建 root span 时生成一个新的 `TraceId`。
对于有父 span 的 `Span`，`TraceId` 必须与父 span 相同。此外，子 span 必须默认继承其父 span 的所有 `TraceState` 值。

如果一个 `Span` 是被另一个进程中创建的 `Span` 的子代，那么它就被称为有一个 *remote parent*。每个传播者的反序列化时必须在父 `SpanContext` 上将 `IsRemote` 设置为 true，这样在 `Span` 的创建时就知道父 `Span` 是否是远程的。

任何被创建的 `Span` 也必须被结束。这是使用者的责任。如果使用者忘记结束 `Span`，API 实现可能会泄漏内存或其他资源（例如，适用于所有 `Span` 的周期性工作的 CPU 时间）。

#### 通过 Context 创建 Parent Span

当新的 Span 通过 Context 创建时，Context 可能已经包含一个代表当前活跃实例的 Span，并将其设置为新的 Span 的父 Span。如果 Context 中没有 Span，那么新创建的 Span 将是一个 Root Span。

`SpanContext` 不能被直接设置成活跃状态，而是通过 [包装成 Span](#wrapping-a-spancontext-in-a-span)。

例如：提取 context 的 `Propagator` 可能需要这个。

#### 指定链接

在创建 `Span` 时，用户必须能够记录与其他 `Span`s 的链接。链接的 `Span` 可以来自相同或不同的 `Trace`。请看链接的[描述](../overview.md#links-between-spans)。Span 创建后不能添加链接。

一个 `Link` 由以下属性组成。

- `SpanContext` ，要链接的 `Span` 所属的 `SpanContext`。
- 零或多个 [`属性`](../common/common.md#attributes) 用于描述链接。

Span 创建 API 必须提供：

- 一个用于记录单个链接的 API，链接的属性可以作为参数进行传递。这可以被称为 `AddLink`。这个 API 需要参数: `SpanContext` （需要被链接的 `Span`）和可选的 `Attributes`，可以是单独的参数，也可以是封装它们的不可变对象，以各自编程语言自身最合适的方式为准。

链接应该保留其设置的顺序。

### Span 操作

除了从 `SpanContext` 取出 `Span` 与记录状态的操作外，其他操作都不能在 `Span` 结束后被调用

#### 获得 Context

Span 接口必须提供：

- 返回 `SpanContext`。一个 API 返回指定的 `Span` 的 `SpanContext`。即使在 Span 结束后，返回的值也可以使用。返回的值必须整个 Span 寿命周期内保持不变。可以被称为 `GetContext` 。

#### IsRecording

返回 `true` 当该 `Span` 正在记录信息，例如使用 `AddEvent` 操作的事件，使用 `SetAttributes` 的属性，使用 `SetStatus` 的状态等。

当 `Span` 结束后，通常会变成非记录状态，因此对结束的 Span，`IsRecording`应当返回 `false`。注意：流式实现（它不知道一个 `Span` 是否结束）是一种预期的情况。在这种情况下，`IsRecording` 在 `Span` 结束后无法被修改。

`IsRecording` 不应当接受任何参数。

这个标记应当用来避免在 Span 没记录时，处理 Span 属性和事件（这两个操作是昂贵的计算）。注意任何子 span 的记录标记都是独立于父 span 的标记来决定的（通常基于 [SpanContext](#spancontext) 上的 TraceFlags 中的采样标记）。

即使整个链路都不被采样，这个标记仍然可能为 `true`。这允许不需要发送到后端即可记录和处理单个 `Span` 的信息。这个情况的一个例子可能是记录和处理所有的传入请求，用于创建 SLA/SLO 延迟图，同时只向后端发送一个子集（采样的子集）。详情参见 [sampling section of SDK design](sdk.md#sampling)。

API 的使用者应当通过 instrumenting 代码访问 IsRecording 属性，除非在 context 传播器中使用，否则永远不要访问SampledFlag。

#### 设置属性 Set Attributes

`Span` 必须支持设置与其相关的  [`属性`](../common/common.md#attributes) 。

Span 接口必须提供：

- 一个 API 用于设置单个`属性`，其中的 `Span` 的具体属性可作为参数传递。这被称为 `SetAttribute`。为了避免额外的分配，一些实现可以为每个可能的值类型提供单独的 API。

注意：OpenTelemetry 项目中定义了一些具有语义的  ["标准属性"](semantic_conventions/README.md)。

注意： [Samplers](sdk.md#sampler) 只考虑在创建 `Span` 时已经存在的信息。创建后的任何改变，包括创建/修改属性，都不能改变原有的决定。

#### 新增事件 Add Events

`Span` 必须提供新增事件的功能。事件在添加进入 `span` 的时候需存在一个时间戳。

`Event` 的结构定义遵循以下的属性：

- 事件名称。
- 事件时间。事件被添加的时间或用户提供的自定义时间戳。
- 零或多个进一步描述事件的 [`属性`](../common/common.md#attributes) 属性。

Span 接口必须提供：

- 一个用于记录单个`事件`的 API，事件属性被作为参数进行传递。这个 API 可以被成为 `AddEvent`。这个 API 需要该事件的名称，可选的属性和一个可选的`时间戳`（用于指定事件的发生时间），这些既可以是单独的参数，也可以是封装他们的不可变对象，以实现语言最合适的方式为准。如果用户没有自定义时间戳，那么会自动设置时间戳为该 API 被调用的时间。

事件应按记录顺序排序。这通常和事件的时间戳相匹配，但事件可能使用自定义的时间戳进行乱序记录。

消费者 (Consumers) 应注意，用户可以在开始 `Span` 前或结束 `Span` 后 为事件提供时间，因此事件的时间可能早于 span 的开始时间，或晚于 span 的结束时间。本规范不对事件时间超出 `span` 时间范围的情况，进行任何标准化。

注意：OpenTelemetry 项目文档中定义了一些具有语义的  ["标准时间名称和键"](semantic_conventions/README.md) 。

注意： [`RecordException`](#record-exception) 是 `AddEvent` 的一种特殊变体，用于记录异常事件。


#### 设置状态

为 `Span`设置状态，默认为 `Unset`。

`Status` 有以下数据定义。

- `StatusCode`，以下所列的数值之一。
- `Description` 可选，提供状态的描述性信息。`描述`必须当 `StatusCode` 为 `error` 时使用。

`StatusCode` 是下列数值之一：

- `Unset`
  - 默认状态。
- `Ok`
  - 表示该操作依据被软件开发者或操作者验证为成功完成。
- `Error`
  - 表示所属操作存在异常。

Span 接口必须提供：

- 一个用于设置`状态`的 API。应当被称为 `SetStatus`。API 接受 `StatusCode` 与可选的 `Description`，这些既可以是单独的参数，也可以是封装他们的不可变对象，以各自编程语言自身最合适的方式为准。对于 `StatusCode` 值为 `Ok` 或 `Unset` 时候必须忽略 `Description`。

除以下情况外，状态码应保持不变：

当 Instrumentation 库将状态设置成 Error 时，状态码应当被记录成文档并可预测。状态代码应该只能根据语义惯例中定义的规则设置为 ERROR。对于语义约定未涵盖的操作，Instrumentation Libraries 应发布自己的约定，包括状态代码。

通常而言，除非显示配置，否则 Instrumentation Libraries 不应将状态码设置为 OK。另外除非出现上述错误，否则 Instrumentation Libraries 应将状态码设置为 "未设置"。

软件开发者或操作者可以设置状态码为 `Ok` 。

分析工具应当通过抑制它们因其他原因产生的错误来响应 `Ok` 状态。例如抑制嘈杂的404错误。
（这里感觉还是翻译不到位，原文：Analysis tools SHOULD respond to an `Ok` status by suppressing any errors they
would otherwise generate. For example, to suppress noisy errors such as 404s.）

只有最后一次调用的值被记录，实现可自由处理之前的调用。

#### 更新名称

更新 `Span` 名称。在名称更新后的任何基于名称的取样操作，将取决于实现者。

注意：  [Samplers](sdk.md#sampler)  只考虑在创建 `span` 期间已经存在的信息。任何创建后的修改，包括更新名称，都不能修改原有的决定。

更新名称的替代方法有：使用过去的时间戳和最终的 `Span` 名来延迟创建 `Span`，或者上报一个带有更新名称的子 `Span`。

需要参数：

- 新的 span 名称，取代在 `span` 创建时创建的名称。

#### 结束

表示该 `Span` 所描述的操作到现在（或者可选的指定时间）已经结束。

实现者应当忽略任何在 `end` 调用发生后的所有操作。换言之，span 被结束后就变为不可记录状态。（存在例外，例如当 `Tracer` 是流式事件且没有可变的状态分配给 `Span`）。 

语言 SIGs 可能利用特定语言的语言特性，提供除 `End` 以外的方法用于结束 `span` 。例如 Python 提供了 `with` 形式结束 `span` 。然而，此类方法的 API 实现必须在内部调用 `End` 方法并记录在文档中。

`End` 必须不会影响子 `Span`s。这些 `Span` 可能在父 `span` 结束后仍然在运行。

`End` 必须不会使任何 `Context` 中的 `span` 进入非活跃状态。必须仍然可以通过 Context 使用已经结束的 `span` 作为父 `span`。此外，任何将 `Span` 放入 `Context` 的逻辑都必须保证在 `Span` 结束后依旧有效。

参数:

- (可选) 一个时间戳用于明确结束时间。如果省略，则必须等价于调用时的时间戳。

API 必须是非阻塞的。

#### 记录异常

为了方便记录异常，实现者应该提供一个 `RecordException` 方法。该 API 可视为  [`AddEvent`](#add-events)的一个特殊变体，同时接口没有额外的参数要求，与 `AddEvent` 的要求是一样的。

该方法的签名由每种语言决定，并可酌情实现重载。该方法必须使用[异常语义约定](semantic_conventions/exceptions.md)文档中的规定，将异常记录为一个事件。所需的最小参数应该只是一个异常对象。

如果提供 `RecordException`，该方法必须接受一个可选参数，以提供任何附加的事件属性（这应该以与 `AddEvent` 方法相同的方式进行）。如果该方法已经生成了同名的属性，那么附加的属性将优先。

注意：`RecordException`  可以被看作是 `AddEvent` 的一个变体，它有额外的参数用于记录异常，而其他参数都是可选的（因为它们有异常语义约定的默认值）。

### Span 生命周期

span 生命周期表示从 span 开始被记录时间戳到结束时间戳整个过程 。

- Span 创建时需要记录开始时间。
- Span 结束时需要记录结束时间。

开始和结束时间以及事件的时间戳必须通过调用相应的 API 时记录。

### 用 Span 包装 SpanContext

API 必须提供一个操作，用实现了 Span 接口的对象包装 SpanContext。这样做的目的是为了在进程内 Span 传播等操作中把 SpanContext 作为 Span 暴露。

如果需要一个新的类型来支持本功能，应当尽力不公开暴露该操作（例如，只暴露一个返回 Span 接口类型对象的函数）。如果一个新类型要求被公开暴露，应当被命名成 `NonRecordingSpan`。

其行为应当准许以下定义:

- `GetContext()` 必须返回被包装的 `SpanContext`.
- `IsRecording` 必须返回 false。用于表示时间，属性和其他元素都未被记录（等价于他们被删除）。

Span 的其余功能必须被定义为无操作行为。注意：这包括 End，因此作为常规的例外，不要求（甚至更加建议）结束这样的 Span。

本功能必须在 API 中完全实现，而且不能被覆盖。

## 跨度种类 SpanKind

`跨度种类` 描述 Span 与父母和子女 Span 之间在 Trace 中的关系。  `Span 种类` 描述了两个独立的属性，便于分析时发掘追踪系统的特性。

跨度种类`描述的第一个属性反映了 Span 是远程的子代或父代。 具有远程父级的 Span 很有趣，因为它们是外部负载的来源。 有远程子代的 Span 也很有趣的，因为它们反映了一个非本地系统的依赖性。

`跨度种类`描述的第二个属性反映了一个子 Span 是否同步调用。 当一个子 Span 是同步时，一般而言，父 Span 应等待其完成。 理解这个属性对于跟踪系统来说是很有帮助的，因为同步的 Span 可能对整个跟踪延迟有所影响。异步方案可以是远程，也可以是本地。

为了使 跨度种类有意义，调用者应当分配一个 Span 不超过一个目的。例如，一个服务器端的 Span 不应直接用作另一个远程 span 的父类。 作为一个简单的准则，在提取和序列化远程调用的 SpanContext 之前，instrumentation 应该创建一个新的 Span。

以下是一些可选的 跨度种类:

* `SERVER` 表示该 span 包含服务端处理同步 RPC 或其他远程请求。该 span 是远程 `CLIENT` span 的子 span。`CLIENT` span 预期会等待响应。
* `CLIENT` 表示该 span 对某些远程服务发送了同步请求。该 span 是远程 `SERVER` 的父 span，应当等待其响应。
* `PRODUCER` 表明该 span 产生异步请求的父 span。其父 span 预计在相应的子 `CONSUMER` span 完成前结束，甚至有可能在子 span 开始前结束。在有批处理消息的场景中，跟踪不同的消息为每个消息创建相应的 `PRODUCER` span。
* `CONSUMER` 表明该 span 是被异步 `PRODUCER` 创建的子 span。
* `INTERNAL` 跨度种类默认值。表示该 span 代表应用程序内部的操作，而不是远程的父或子操作。

To summarize the interpretation of these kinds:

| `跨度种类` | 同步 | 异步 | 远端接受 | 远端发送 |
| ---------- | ----------- | ------------ | --------------- | --------------- |
| `CLIENT`   | 是        |              |                 | 是             |
| `SERVER`   | 是        |              | 是            |                 |
| `PRODUCER` |             | 是         |                 | 可能           |
| `CONSUMER` |             | 是         | 可能         |                 |
| `INTERNAL` |             |              |                 |                 |

## 并发性

对于支持并发执行的语言，Tracing APIs 提供了特定的保证和安全保障。并非所有的 API 函数都可以安全地被并发调用。

**TracerProvider** - 所有方法可以安全的并行调用。

**Tracer** - 所有方法可以安全的并行调用。

**Span** - 所有方法可以安全的并行调用。

**Event** - Events 为不可变，可安全的并行调用。

**Link** - Links 为不可变，可安全的并行调用。

## 包含传播者 Included Propagator

API 层或扩展包必须包括以下 传播者。

* 一个 `TextMapPropagator` 基于 [W3C TraceContext Specification](https://www.w3.org/TR/trace-context/) 实现。

查看 [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
了解 Propagators 如何被分发。

## 未安装 SDK 时，API 的行为

通常而言，在没有安装 SDK 时，Trace API 为一个 无操作("no-op") 的 API 。这意味着对 Tracer 或 Spans 的任何操作都应当是无副作用且无用的。

但是，该原则有个重要的例外，这与 `SpanContext` 的传播有关。API 必须使用 Span 的父 Context 的SpanContext 创建一个非记录 [Span](#wrapping-a-spancontext-in-a-span) （不管这是通过显示还是隐式设定的），或者如果父 Context 是一个非记录 Span（如果没有 SDK，通常会是这样），这可能会创建方法中返回父 Span。

如果父 `Context` 不包含 `Span`，则必须返回一个空的非记录 Span（拥有一个 SpanContext，其 SpanID 和 TraceIDs 全设为零，空的Tracestate 与未采样的 TraceFlags）。这意味着由配置的传播者提供的 SpanContext 将被传播到任何子 span，并最终也会被 `Inject`，但并不会创建新的 `SpanContext`。
