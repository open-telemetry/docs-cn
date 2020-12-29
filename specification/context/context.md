<!--# Context-->

# 上下文

**状态**: [功能冻结](../document-status.md)。

<details>
<summary>
目录
</summary>

- [概述](#概述)
- [创建键](#创建键)
- [获得值](#获得值)
- [设置值](#设置值)
- [可选全局操作](#可选全局操作)
  - [获得当前上下文](#获得当前上下文)
  - [连接上下文](#连接上下文)
  - [断开上下文](#断开上下文)

</details>

## 概述

<!--A `Context` is a propagation mechanism which carries execution-scoped values across API boundaries and between logically associated execution units. Cross-cutting concerns access their data in-process using the same shared `Context` object.-->

`上下文 `是一种传播机制，它跨越 API 边界，在逻辑执行单元间传递执行范围的值。贯穿各领域的问题通过共享 `上下文` 对象在进程中访问其数据。

<!--A `Context` MUST be immutable, and its write operations MUST result in the creation of a new `Context` containing the original values and the specified values updated.-->

`上下文` 必须是不可改变的，其写操作必须导致创建一个新的 `上下文`，其中包含原始值和更新的值。

<!--Languages are expected to use the single, widely used Context implementation if one exists for them. In the cases where an extremely clear, pre-existing option is not available, OpenTelemetry MUST provide its own Context implementation. Depending on the language, its usage may be either explicit or implicit.-->

希望各语言能实现单一的、被广泛使用的 `上下文`。在没有非常明确，预先存在的选项的情况下，OpenTelemetry 必须提供自己的 `Context` 实现。根据不同语言的实现，其用法可以是显式，也可以是隐式。

<!--Users writing instrumentation in languages that use `Context` implicitly are discouraged from using the `Context` API directly. In those cases, users will manipulate `Context` through cross-cutting concerns APIs instead, in order to perform operations such as setting trace or baggage entries for a specified `Context`.-->

不鼓励用户使用已隐式调用上下文的 instrumentation 时，直接使用上下文 API。在这些情况下，用户将通过横切关注点的 API 作为替代，以执行诸如设置 trace 或 设置 baggage 条目等操作。

<!--A `Context` is expected to have the following operations, with their respective language differences:-->

`上下文` 将有以下操作，并有各自的语言差异：

<!--## Create a key-->

## 创建键

<!--Keys are used to allow cross-cutting concerns to control access to their local state. They are unique such that other libraries which may use the same context cannot accidentally use the same key. It is recommended that concerns mediate data access via an API, rather than provide direct public access to their keys.-->

键用于横切关注点控制对其本地状态的访问。它们是唯一的，因此其他可能使用相同上下文的库不能意外地使用相同的键。建议有关方面通过 API 控制数据访问，而不是提供对其键的直接公开访问。

<!--The API MUST accept the following parameter:-->

API必须接受以下参数：

<!--The key name. The key name exists for debugging purposes and does not uniquely identify the key. Multiple calls to `CreateKey` with the same name SHOULD NOT return the same value unless language constraints dictate otherwise. Different languages may impose different restrictions on the expected types, so this parameter remains an implementation detail.-->

- 键的名称。钥匙名称存在的目的是便于调试，并不是键的唯一标识。对 `CreateKey` 的使用相同名称多次调用不应该返回相同的值，除非语言限制另有规定。不同的语言可能会对预期的类型施加不同的限制，所以这个参数请具体参考实现细节。

<!--The API MUST return an opaque object representing the newly created key.-->

API必须返回一个不透明的对象，代表新创建的密钥。

<!--## Get value-->

## 获得值

<!--Concerns can access their local state in the current execution state represented by a `Context`.-->

关注点可以在由`上下文`表示的当前执行状态下访问其本地状态。

<!--The API MUST accept the following parameters:-->

API必须接受以下参数：

- `上下文`。
- 键。

<!--The API MUST return the value in the `Context` for the specified key.-->

API 必须返回上下文中指定键的值。

<!--## Set value-->



## 设置值

<!--Concerns can record their local state in the current execution state represented by a `Context`.-->

关注点可以用 "上下文 "表示当前的执行状态中记录其本地状态。

<!--The API MUST accept the following parameters:-->

API必须接受以下参数：

- `上下文`。
- 键。
- 需要被设置的值。

<!--The API MUST return a new `Context` containing the new value.-->

API 必须返回一个包含新值的新`上下文`。



<!--## Optional Global operations-->

## 可选全局操作

<!--These operations are expected to only be implemented by languages using `Context` implicitly, and thus are optional. These operations SHOULD only be used to implement automatic scope switching and define higher level APIs by SDK components and OpenTelemetry instrumentation libraries.-->

这些操作预计只能由隐式使用上下文的语言实现，因此是可选的。这些操作应该只用于实现自动范围切换，并由 SDK 组件和 OpenTelemetry instrumentation 定义的更高级别的 API。

<!--### Get current Context-->

### 获得当前上下文

<!--The API MUST return the `Context` associated with the caller's current execution unit.-->

AP I必须返回与调用者当前执行单元相关的`上下文`。

<!--### Attach Context-->

### 连接上下文

<!--Associates a `Context` with the caller's current execution unit.-->

将上下文与调用者的当前执行单元连接起来。

<!--The API MUST accept the following parameters:-->

API必须接受以下参数：

- 上下文。

<!--The API MUST return a value that can be used as a `Token` to restore the previous `Context`-->

API 必须返回一个值，该值可以作为`标记` 来恢复上一个的 `上下文'。

<!--Note that every call to this operation should result in a corresponding call to [Detach Context](#detach-context).-->

注意：每次调用这个操作都应该产生对 [断开上下文](#断开上下文) 的相应调用。

<!--### Detach Context-->

### 断开上下文

<!--Resets the `Context` associated with the caller's current execution unit to the value it had before attaching a specified `Context`.-->

将与调用者当前执行单元相连接的`上下文`重置为连接`上下文`操作之前的值。

<!--This operation is intended to help making sure the correct `Context` is associated with the caller's current execution unit. Users can rely on it to identify a wrong call order, i.e. trying to detach a `Context` that is not the current instance. In this case the operation can emit a signal to warn users of the wrong call order, such as logging an error or returning an error value.-->

本操作旨在帮助确保正确关联`上下文`与调用者的当前执行单元。用户可以依靠它来识别错误的调用顺序，例如：试图分离一个不是当前实例的`上下文`。在这种情况下，该操作可以发出信号，警告用户调用顺序错误，如记录一个错误或返回一个错误值。

<!--The API MUST accept the following parameters:-->

API必须接受以下参数：

<!--A `Token` that was returned by a previous call to attach a `Context`.-->

- 上次调用连接 "上下文 "时返回的 "令牌"。

<!--The API MAY return a value used to check whether the operation was successful or not.-->

API可以返回一个值，用于检查操作成功与否。