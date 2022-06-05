<!-- # Baggage API -->
# Baggage API (背包 API)
<!-- **Status**: [Stable, Feature-freeze](../document-status.md) -->
**状态**： [稳定， 功能冻结](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/document-status.md)。

<details>
<summary>
目录
</summary>

- [概述](#概述)
- [操作](#操作)
  - [获取值](#获取值)
  - [获取全部值](#获取全部值)
  - [设置值](#设置值)
  - [删除值](#删除值)
- [与Context交互](#与Context交互)
  - [清除Context中的Baggage](#清除Context中的Baggage)
- [传播](#传播)
- [冲突解决](#冲突解决)
</details>


## 概述

 `Baggage` 用于注解遥测数据，给指标、追踪、日志增加上下文和信息。 它是一系列描述用户定义的属性的名字/值对。 `Baggage` 中的每个名称必须只关联一个值。

Baggage API 由以下组成:

  <!--- the `Baggage``-->
- `Baggage`

  <!--- functions to interact with the `Baggage` in a `Context`-->
- 用于 `Baggage` 与上下文交互的函数

<!-- The functions described here are one way to approach interacting with the -->
<!-- `Baggage` via having struct/object that represents the entire Baggage content. -->
<!-- Depending on language idioms, a language API MAY implement these functions by -->
<!-- providing the defined functionality interacting purely via the `Context`. -->
这里定义的函数是指一种与拥有代表整个 Baggage 的结构体或对象进行交互的方式。取决于语言习惯，取决于语言习惯，语言 API 可能仅实现与 `Context` 交互的函数。

<!-- The Baggage API MUST be fully functional in the absence of an installed SDK. -->
<!-- This is required in order to enable transparent cross-process Baggage -->
<!-- propagation. If a Baggage propagator is installed into the API, it will work -->
<!-- with or without an installed SDK. -->
Baggage API 必须在没有安装 SDK 的情况下提供完整功能。这是为实现透明的跨进程 Baggage 传播所必须的。如果在 API 中安装了 Baggage 传播者，无论是否安装了 SDK，它都可以工作。

<!-- The `Baggage` container MUST be immutable, so that the containing `Context` -->
<!-- also remains immutable. -->
`Baggage` 的容器必须是不可变的, 这样包含的 `Context` 也是不可变的。

<!-- ## Operations -->
## 操作

<!-- ### Get Value -->
### 获取值

<!-- To access the value for a name/value pair set by a prior event, the Baggage API -->
<!-- MUST provide a function that takes the name as input, and returns a value -->
<!-- associated with the given name, or null if the given name is not present. -->
Baggage API 必须提供一个函数用于要访问名称/值对中的值，该函数将上下文和名称作为参数输入，并返回一个值。返回的值与名称参数相关，如果提供的名称参数不存在，则返回空值。

<!-- REQUIRED parameters: -->
<!-- `Name` the name to return the value for. -->
必填参数:
 `Name` 要返回对应值的名称。

<!-- ### Get All Values -->
### 获取全部值

<!-- Returns the name/value pairs in the `Baggage`. The order of name/value pairs -->
<!-- MUST NOT be significant. Based on the language specifics, the returned -->
<!-- value can be either an immutable collection or an iterator on the immutable -->
<!-- collection of name/value pairs in the `Baggage`. -->
返回 `Baggage` 中的名称/值对。名称/值对的顺序必须不重要。取决于语言习惯，返回的值可以是一个不可变的集合，也可以是一个指向 `Baggage` 中的名称/值对集合的不可变迭代器。

<!-- ### Set Value -->
### 设置值

<!-- To record the value for a name/value pair, the Baggage API MUST provide a -->
<!-- function which takes a name, and a value as input. Returns a new `Baggage` -->
<!-- that contains the new value. Depending on language idioms, a language API MAY -->
<!-- implement these functions by using a `Builder` pattern and exposing a way to -->
<!-- construct a `Builder` from a `Baggage`. -->
Baggage API 必须提供一个函数用于记录名称/值中的值，该函数接受一个名称和一个值作为输入参数。返回一个带有新值 `Baggage` 的 `上下文`。取决于语言习惯, 语言API可以使用构建者模式来从 Builder 构建出一个 `Baggage`。

<!-- REQUIRED parameters: -->
必填参数:

<!-- `Name` The name for which to set the value, of type string. -->
`Name` 设置值的名称，字符串类型。

<!-- `Value` The value to set, of type string. -->
`Value` 要设置的值，字符串类型。

<!--OPTIONAL parameters:-->
可选参数：

<!-- `Metadata` Optional metadata associated with the name-value pair. This should be -->
<!-- an opaque wrapper for a string with no semantic meaning. Left opaque to allow -->
<!-- for future functionality. -->
`Metadata` 与名称/值对相关联的元数据。这应当是对一个没有具体语义的字符串的不透明包装。以便于未来的功能拓展。

<!-- ### Remove Value -->
###  删除值

<!-- To delete a name/value pair, the Baggage API MUST provide a function which -->
<!-- takes a name as input. Returns a new `Baggage` which no longer contains the -->
<!-- selected name. Depending on language idioms, a language API MAY -->
<!-- implement these functions by using a `Builder` pattern and exposing a way to -->
<!-- construct a `Builder` from a `Baggage`. -->
为了删除名称/值对，Baggage API 必须提供一个函数，接收一个名称作为输入参数。返回一个新的上下文，该上下文不再包含所选名称。取决于语言习惯, 语言 API 可以使用构建者模式来从 Builder 构建出一个 `Baggage`。

<!--REQUIRED parameters:-->
必填参数：

<!--Name the name to remove.-->
`Name` 要移除的名称。

<!-- ## Context Interaction -->
## 与Context交互

<!-- This section defines all operations within the Baggage API that interact with -->
<!-- the [`Context`](../context/context.md). -->
此节定义所有与 [`Context`](../context/context.md) 交互的有关 Baggage API 的操作

<!-- The API MUST provide the following functionality to interact with a `Context` -->
<!-- instance: -->
此 API 必须提供以下方法与 `Context` 实例交互。

<!-- - Extract the `Baggage` from a `Context` instance -->
<!-- - Insert the `Baggage` to a `Context` instance -->
- 从 `Context` 实例中提取 `Baggage`
- 向 `Context` 实例插入 `Baggage`

<!-- The functionality listed above is necessary because API users SHOULD NOT have -->
<!-- access to the [Context Key](../context/context.md#create-a-key) used by the -->
<!-- Baggage API implementation. -->
上面列出的函数是必要的, 因为 API 用户应该不需要访问 `Baggage` API 实现相关的 [Context 键](../context/context.md#create-a-key)

<!-- If the language has support for implicitly propagated `Context` (see -->
<!-- [here](../context/context.md#optional-global-operations)), the API SHOULD also -->
<!-- provide the following functionality: -->
如果一种语言支持隐式传播 `Context` (参考[此](../context/context.md#optional-global-operations)), 那么 API 应该提供以下方法:

<!-- - Get the currently active `Baggage` from the implicit context. This is -->
<!-- equivalent to getting the implicit context, then extracting the `Baggage` from -->
<!-- the context. -->
<!-- - Set the currently active `Baggage` to the implicit context. This is equivalent -->
<!-- to getting the implicit context, then inserting the `Baggage` to the context. -->
- 从隐式上下文中获取当前的 `Baggage`, 这等价于先获取隐式的上下文, 然后从上下文中提取 `Baggage`。
- 设置当前的 `Baggage` 至隐式上下文中, 这等价于先获取隐式的上下文, 然后插入 `Baggage` 至上下文中。

<!-- ### Clear Baggage in the Context -->
### 清除Context中的Baggage

<!-- To avoid sending any name/value pairs to an untrusted process, the Baggage API -->
<!-- MUST provide a way to remove all baggage entries from a context. -->
为了避免向不信任的进程发送任何的名称/值对，Baggage API 必须提供一种方式从 Context 中删除所有的 Baggage 条目。

<!-- This functionality can be implemented by having the user set an empty `Baggage` -->
<!-- object/struct into the context, or by providing an API that takes a `Context` as -->
<!-- input, and returns a new `Context` with no `Baggage` associated. -->
这个函数可以通过用户设置一个空的 `Baggage` 对象/结构体至 `Context` 中来实现。 或者提供一个API, 它以 `Context` 作为输入参数, 返回一个不包含任何相关 `Baggage` 的新的 `Context`.

<!-- ## Propagation -->
## 传播

<!-- `Baggage` MAY be propagated across process boundaries or across any arbitrary -->
<!-- boundaries (process, $OTHER_BOUNDARY1, $OTHER_BOUNDARY2, etc) for various -->
<!-- reasons. -->
`Baggage` 可能因为各种原因跨越进程边界或任意边界（进程，边界1, 边界2, 等等）进行传播。

<!-- The API layer or an extension package MUST include the following `Propagator`s: -->
API 层或扩展包必须包括以下 `Propagator` (传播者)。

<!-- * A `TextMapPropagator` implementing the [W3C Baggage Specification](https://w3c.github.io/baggage). -->

* 一个 `TextMapPropagator` 实现了 [W3C Baggage Specification](https://w3c.github.io/baggage)。

<!-- See [Propagators Distribution](../context/api-propagators.md#propagators-distribution) -->
<!-- for how propagators are to be distributed. -->
参考 [Propagators Distribution](../context/api-propagators.md#propagators-distribution) 了解传播者如何被分发。

<!-- Note: The W3C baggage specification does not currently assign semantic meaning -->
<!-- to the optional metadata. -->
注：W3C baggage 规范目前没有给可选项的元数据赋予语义。

<!-- On `extract`, the propagator should store all metadata as a single metadata instance per entry. -->
<!-- On `inject`, the propagator should append the metadata per the W3C specification format. -->
当`提取`时，传播者应将所有元信息用一个元数据实例分配给各个条目。
当`注入`时，传播者应该根据W3C规范格式附加元数据。

<!-- Notes: -->
注：

<!-- If the propagator is unable to parse the incoming `baggage`, `extract` MUST return -->
<!-- a `Context` with no baggage entries in it. -->
如果传播者无法解析传过来的 `baggage` ，`提取` 函数必须返回一个没有 Baggage 条目的上下文。

<!-- If the incoming `baggage` is present, but contains no entries, `extract` MUST -->
<!-- return a `Context` with no baggage entries in it. -->
如果传过来的 `baggage` 存在，但不存在任何条目， `提取` 函数必须返回一个没有 Baggage 条目的上下文。

<!-- ## Conflict Resolution -->
## 冲突解决

<!-- If a new name/value pair is added and its name is the same as an existing name, -->
<!-- than the new pair MUST take precedence. The value is replaced with the added -->
<!-- value (regardless if it is locally generated or received from a remote peer). -->
如果添加新的名称/值对时已经存在同名，则新名称/值对的必须优先。值将被新添加的值替代（不论是本地生成还是从远处对等处接受）。
