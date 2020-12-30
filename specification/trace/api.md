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

- 返回 `SpanContext`。一个 API 返回指定的 `Span` 的 `SpanContext`。即使在 Span 结束后，该接口也可以使用。返回的值必须整个 Span 寿命周期内保持不变。可以被称为 `GetContext` 。

#### IsRecording

返回 `true` 当该 `Span` 正在记录信息，例如使用 `AddEvent` 操作的事件，使用 `SetAttributes` 的属性，使用 `SetStatus` 的状态等。

当 `Span` 结束后，通常会变成非记录状态，因此对结束的 Span，`IsRecording`应当返回 `false`。注意：流数据的实现时，它不知道一个 `Span` 是否结束，这是一个预期的情况。`IsRecording` 在 `Span` 结束后无法被修改

`IsRecording` 不应当接受任何参数。

这个标记应当用来避免在 Span 没记录时，处理 Span 属性和事件（这两个操作是昂贵的计算）。注意任何子 span 的记录标记都是独立于父 span 的标记来决定的（通常基于 [SpanContext](#spancontext) 上的 TraceFlags 中的采样标记）。

这个标签可能是 `true`， 当整个事件再被采样的时候。这允许不需要发送到后端即可记录和处理单个 `Span` 的信息。这个情况的一个例子可能是记录和处理所有的传入请求，用于创建 SLA/SLO 延迟图，同事只向后端发送一个子集（采样的子集）。详情参见 [sampling section of SDK design](sdk.md#sampling)。

API 的使用者应当通过 instrumenting 代码访问 IsRecording 属性，除非在 context 传播器中使用，否则永远不要访问SampledFlag。

#### 设置属性 Set Attributes

`Span` 必须支持设置予以相关的  [`属性`](../common/common.md#attributes) 。

Span 接口必须提供：

- 一个 API 用于设置单个`属性`，其中的 `Span` 的具体属性可作为参数传递。这被称为 `SetAttribute`。为了避免额外的分配，一些实现可以为每个可能的值类型提供单独的 API。

注意：OpenTelemetry 项目中定义了一些具有语义的  ["标准属性"](semantic_conventions/README.md)。

注意： [Samplers](sdk.md#sampler) 只考虑在创建 `Span` 时已经存在的信息。创建后的任何改变，包括创建/修改属性，都不能改变原有的决定。

#### 增加事件 Add Events

`Span` 必须提供增加事件的功能。事件在添加进入 `span` 的时候需存在一个时间戳。

`Event` 的结构定义遵循以下的属性：

- 事件名称。
- 事件时间。事件被添加的时间或用户提供的自定义时间戳。
- 零或多个进一步描述事件的 [`属性`](../common/common.md#attributes) 属性。

Span 接口必须提供：

- 一个用于记录单个`事件`的 API，事件属性被作为参数进行传递。这个 API 可以被成为 `AddEvent`。这个 API 需要该事件的名称，可选的属性和一个可选的`时间戳`（用于指定事件的发生时间），这些既可以是单独的参数，也可以是封装他们的不可变对象，以实现语言最合适的方式为准。如果用户没有自定义时间戳，那么会自动设置时间戳为该 API 被调用的时间。

时间应按记录顺序排序。这通常和时间的时间戳相匹配，但时间可能使用自定义的时间戳进行乱序记录。

消费者 (Consumers) 应注意，用户可以在开始 `Span` 前或结束 `Span` 后 为事件提供时间，因此事件的时间可能早于 span 的开始时间，或晚于 span 的结束时间。本规范不对事件时间超出 `span` 时间范围的情况，进行任何标准化。

注意：OpenTelemetry 项目文档中定义了一些具有语义的  ["标准时间名称和键"](semantic_conventions/README.md) 。

注意： [`RecordException`](#record-exception) 是一种特殊的变体，用于记录 `AddEvent` 发生的异常时间。


#### 设置状态

为 `Span`设置状态，默认为 `Unset`。

`Status` 有以下数据定义。

- `StatusCode`，以下所列的数值之一。
-  `Description` 可选，提供状态的描述性信息。`描述`必须当 `StatusCode` 为 `error` 时使用。

`StatusCode` 是下列数值之一：

- `Unset`
  - 默认状态
- `Ok`
  - 表示该操作依据被软件开发者或操作者验证为完全
- `Error`
  - 表示所属操作存在异常。

Span 接口必须提供：

- 一个用于设置`状态`的 API。应当被称为 `SetStatus`。API 接受 `StatusCode` 与可选的 `Description`，这些既可以是单独的参数，也可以是封装他们的不可变对象，以各自编程语言自身最合适的方式为准。对于 `StatusCode` 值为 `Ok` 或 `Unset` 时候必须忽略 `Description`。

除以下情况外，状态码应保持不变：

​	当 Instrumentation 库将状态设置成 Error 时，状态码应当被记录下来并可预测。状态代码应该只能根据语义惯例中定义的规则设置为 ERROR。对于语义约定未涵盖的操作，Instrumentation Libraries 应发布自己的约定，包括状态代码。

通常而言，Instrumentation Libraries 不应将状态码设置为 OK，除非有明确合理的目的 。另外除非出现上述错误，否则 Instrumentation Libraries 应将状态码设置为 "未设置"。

软件开发者或操作者可以设置状态码为 `Ok` 。

分析工具应当对 `Ok` 状态做出响应，抑制它们因其他原因产生的任何错误。例如，为了抑制诸如 404s 错误。

只有最后一次调用的值被记录，实现可自由处理之前的调用。

#### 更新名称

更新 `Span` 名称。在名称更新后的任何基于名称的取样操作，将取决于实现者。

注意：  [Samplers](sdk.md#sampler)  只考虑在创建 `span` 期间已经存在的信息。任何创建后的修改，包括更新名称，都不能修改原有的决定。

 `span` 创建后名称更新的替代方法是：如果确定了最终 `span` 名称， 可以使用已明确的时间戳（过去的时间）创建 `Span`，或者将带有所需名称的 `Span` 报告为子 `Span`。

创建 `Span` 过程的后段，当 Span 已经使用明确的时间戳开始时

需要参数：

- 新的 span 名称，取代在 `span` 创建时创建的名称。

#### 结束

表示该 `Span` 所描述的操作到现在（或者可选的指定时间）已经结束。

实现者应当忽略任何在 `end` 调用发生后的所有操作。换言之，span 被结束后就变为不可记录状态。（存在例外，例如当 `Tracer` 是流式事件且没有可变的状态分配给 `Span`）。 

语言 SIGs 可能利用特定语言的语言特性，提供除 `End` 以外的方法用于结束 `span` 。例如 Python 提供了 `with` 形式结束 `span` 。然而，所有实现者的 API 必须在内部存在 `call` 方法并提供文档教导如何使用。

`End` 必须不会影响子 `Span`s。这些 `Span` 可能在父 `span` 结束后仍然在运行。

`End` 必须不会使任何 `Context` 中的 `span` 进入非活跃状态。必须仍然可以通过 Context 使用已经结束的 `span` 作为父 `span`。此外，任何将 `Span` 放入 `Context` 的逻辑都必须保证在 `Span` 结束后依旧有效。

参数:

- (可选) 一个时间戳用于明确结束时间。如果省略，则必须等价于调用时的时间戳。

API 必须是非阻塞的。

#### 记录异常

为了方便记录异常，实现者应该提供一个 `RecordException` 方法。该 API 可视为  [`AddEvent`](#add-events)的一个特殊变体，同时接口没有额外的参数要求，与 `AddEvent` 的要求是一样的。

该方法的签名由每种语言决定，并可酌情实现重载。该方法必须使用[异常语义约定](semantic_conventions/exceptions.md)文档中的规定，将异常记录为一个事件。所需的最小参数应该只是一个异常对象。

如果提供 `RecordException`，该方法必须接受一个可选参数，以提供任何附加的事件属性（这应该以与 `AddEvent` 方法相同的方式进行）。如果该方法已经生成了同名的属性，那么附加的属性将优先。

注意：`RecordException`  可以被看作是 `AddEvent` 的一个变体，它有额外的参数用于记录异常，而其他参数都是可选的（因为它们有异常语义约定的默认值）。

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