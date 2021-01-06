# Baggage API

状态: [功能冻结](../document-status.md)。

<details>
<summary>
目录
</summary>

- [概述](#概述)
  - [Baggage](#baggage)
  - [获得全部 baggages](#获得全部-baggages)
  - [获得 baggage](#获得-baggage)
  - [设置 baggage](#设置-baggage)
  - [删除 baggage](#删除-baggage)
  - [清理](#清理)
- [Baggage 传递](#baggage-传递)
- [冲突解决](#[冲突解决)


</details>


## 概述

<!-- The Baggage API consists of: -->

Baggage API 由以下组成:

  <!--- Baggage`-->
- `Baggage`


  <!--- functions to interact with the `Baggage` in a `Context`-->
- 一些用于`Baggage` 与上下文交互的函数



<!--The functions described here are one way to approach interacting with the Baggage purely via the Context. Depending on language idioms, a language API MAY implement these functions by providing a struct or immutable object that represents the entire Baggage contents. This construct could then be added or removed from the Context with a single operation. -->

<!--For example, the Clear function could be implemented by having the user set an empty Baggage object/struct into the context. The Get all function could be implemented by returning the Baggage object as a whole from the function call. If an idiom like this is implemented, the Baggage object/struct MUST be immutable, so that the containing Context also remains immutable.-->

这里定义的函数是指一种通过上下文 与Baggage 进行交互的方式。根据编程语言的习惯，语言 API 可以通过实现一些函数，提供结构化或不可变对象来代表整个 Baggage 内容。其构造器可以通过一个操作对上下文进行增加或删除。
例如，通过让用户在上下文中设置一个空的 Baggage 对象/结构来实现一个[清理](#清理)函数。通过从返回整个 Baggage 对象来实现[获得全部](获取全部)函数。如果实现者按本段的建议实现，Baggage 对象/结构必须是不可变的，所包含的上下文也是不可变的。

<!--The Baggage API MUST be fully functional in the absence of an installed SDK. This is required in order to enable transparent cross-process Baggage propagation. If a Baggage propagator is installed into the API, it will work with or without an installed SDK.-->

在没有安装 SDK 情况时，Baggage API 提供完整功能。这是为实现透明的跨进程 Baggage 传播所必须的。如果在 API 中安装了 Baggage 传播者，无论是否安装了 SDB，它都可以工作。

### Baggage

<!--`Baggage` is used to annotate telemetry, adding context and information to metrics, traces, and logs. It is an abstract data type represented by a set of name/value pairs describing user-defined properties. Each name in `Baggage` MUST be associated with exactly one value.-->

`Baggage` 用于注释遥感，向指标，跟踪和日志中添加上下文与信息。这是一种抽象的数据类型，由一组描述用户定义属性的名称/值对组成。`Baggage` 中的每个名称必须与一个值相关联。

 <!--### Get ALL--> 

### 获得全部

<!--Returns the name/value pairs in the `Baggage`. The order of name/value pairs MUST NOT be significant. Based on the language specification, the returned value can be either an immutable collection or an immutable iterator to the collection of name/value pairs in the `Baggage`.-->

返回 `Baggage` 中的名称/值对。名称/值对的顺序必须不重要。基于语言规范，返回的值可以是一个不可变的集合，也可以是一个指向 `Baggage` 中的名称/值对集合的不可变迭代器。

<!--OPTIONAL parameters:--> 

可选参数：

<!--`Context` the context containing the `Baggage` from which to get the baggages.-->

`Context`：带有 `Baggage` 的上下文，从 baggages 中获得。

<!-- ### Get baggage--> 

### 获得 baggage

<!--To access the value for a name/value pair by a prior event, the Baggage API MUST provide a function that takes a context and a name as input, and returns a value. Returns the value associated with the given name, or null if the given name is not present.-->

Baggage API 必须提供一个函数用于要访问名称/值对中的值，该函数将上下文和名称作为参数输入，并返回一个值。返回的值与名称参数相关，如果提供的名称参数不存在，则返回空值。



<!--REQUIRED parameters:-->

必要参数：

<!--Name the name to return the value for.-->

`Name`：要返回值的名称。

<!--OPTIONAL parameters:-->

可选参数：

<!--Context the context containing the `Baggage` from which to get the baggage entry.-->

`Context`：包含 `Baggage` 的上下文，从中获取 Baggage 条目。

### 设置 baggage

<!--To record the value for a name/value pair, the Baggage API MUST provide a function which takes a context, a name, and a value as input. Returns a new Context which contains a Baggage with the new value.-->

Baggage API 必须提供一个函数用于记录名称/值中的值，该函数接受一个上下文，一个名称和一个值作为参数。返回一个带有新值 `Baggage` 的`上下文`。

<!--REQUIRED parameters:-->

必要参数：

<!--Name The name for which to set the value, of type string.-->

`Name` 设置值的名称，字符串。

<!--Value The value to set, of type string.-->

`Value` 要设置的值，字符串。

<!--OPTIONAL parameters:-->

可选参数：

<!--Metadata Optional metadata associated with the name-value pair. This should be an opaque wrapper for a string with no semantic meaning. Left opaque to allow for future functionality.-->

`Metadaata` 与名称/值对相关联的元数据。这应当是一个不透明的包装一个没有具体语义的字符串。目的是为了便于未来的功能拓展。

<!--Context The context containing the `Baggage` in which to set the baggage entry.-->

`Context`：包含 `Baggage` 的上下文，用于设置 Baggage 条目。

<!--### Delete baggage-->

###  删除 baggage

<!--To delete a name/value pair, the Baggage API MUST provide a function which takes a context and a name as input. Returns a new `Context` which no longer contains the selected name.-->

为了删除名称/值对，Baggage API 必须提供一个函数，接收一个上下文和一个名称作为输入。返回一个新的上下文，该上下文不再包含所选名称。

<!--REQUIRED parameters:-->

必要参数：

<!--Name the name to remove.-->

`Name` 要被移除的名称。

<!--OPTIONAL parameters:-->

可选参数：

<!--Context the context containing the `Baggage` from which to remove the baggage entry.-->

`Context`：包含 `Baggage` 的上下文，用于删除 Baggage 条目。

<!--### Clear-->

### 清理

<!--To avoid sending any name/value pairs to an untrusted process, the Baggage API MUST provide a function to remove all baggage entries from a context. Returns a new `Context` with no `Baggage`.-->

为了避免向不信任的进程发送任何的名称/值对，Baggage API 必须提供一个函数，用于删除一个上下文中所有 Baggage 条目。返回一个没有 `Baggage` 的新上下文。

<!--OPTIONAL parameters:-->

可选参数：

<!--Context the context containing the `Baggage` from which to remove all baggage entries.-->

`Context`：包含 `Baggage` 的上下文，用于清理所有 Baggage 条目。



 <!-- ## Baggage Propagation-->

## Baggage 传递

<!-- `Baggage` MAY be propagated across process boundaries or across any arbitrary boundaries
(process, $OTHER_BOUNDARY1, $OTHER_BOUNDARY2, etc) for various reasons.-->

`Baggage` 可能因为各种原因跨越进程边界，或任意边界（进程，$OTHER_BOUNDARY1, $OTHER_BOUNDARY2, etc）进行传播。

<!--The API layer or an extension package MUST include the following `Propagator`s:-->

API 层或扩展包必须包括以下 传播者。

<!--A `TextMapPropagator` implementing the [W3C Baggage Specification](https://w3c.github.io/baggage).-->

* 一个 `TextMapPropagator` 基于 [W3C TraceContext Specification](https://www.w3.org/TR/trace-context/) 实现。

<!--See [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
for how propagators are to be distributed.-->

查看 [Propagators Distribution](../context/api-propagators.md#propagators-distribution)
了解 Propagators 如何被分发。



<!--Note: The W3C baggage specification does not currently assign semantic meaning to the optional metadata.On `extract`, the propagator should store all metadata as a single metadata instance per entry. On `inject`, the propagator should append the metadata per the W3C specification format.-->

注：W3C baggage 规范目前没有给可选项的元数据赋予语义。当`提取`时，传播者应将所有元信息用一个元数据实例分配给各个条目。当 `注入`时，传播者应当遵循 W3C 规范格式附加元数据。



<!--Notes:If the propagator is unable to parse the `baggage` header, `extract` MUST return a Context with no baggage entries in it. If the `baggage` header is present, but contains no entries, `extract` MUST return a Context with no baggage entries in it.-->

注：如果传播者无法解析 `baggage` 头，`提取` 必须一个空的（没有 Baggage 条目）上下文。如果 `baggage`头存在，但不存在任何条目， `提取` 必须一个空的上下文。

 <!--## Conflict Resolution-->

## 冲突解决

<!--If a new name/value pair is added and its name is the same as an existing name, than the new pair MUST take precedence. The value is replaced with the added value (regardless if it is locally generated or received from a remote peer).-->

如果在添加新的名称/值对时，已经存在同名的名称/值对，新的对必须优先。值将被新添加的值替代（不论是本地生成还是从远处对等处接受）。