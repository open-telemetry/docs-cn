# 链路传递 API

**状态**: [稳定版本, 短期内不会改动](../document-status.md)

<details>
<summary>目录大纲</summary>

<!-- toc -->

- [综述](#overview)
- [链路传递 特征属性](#propagator-types)
  * [Carrier](#carrier)
  * [基本操作](#operations)
    + [Inject 注入](#inject)
    + [Extract 提取](#extract)
- [TextMap的传递](#textmap-propagator)
  * [字段](#fields)
  * [TextMap Inject](#textmap-inject)
    + [Setter 参数](#setter-argument)
      - [Set](#set)
  * [TextMap Extract](#textmap-extract)
    + [Getter 参数](#getter-argument)
      - [Keys](#keys)
      - [Get](#get)
- [自定义注入和提取接口](#injectors-and-extractors-as-separate-interfaces)
- [复合传播](#composite-propagator)
  * [Create a Composite Propagator](#create-a-composite-propagator)
  * [Composite Extract](#composite-extract)
  * [Composite Inject](#composite-inject)
- [全局传播](#global-propagators)
  * [Get 全局传播](#get-global-propagator)
  * [Set 全局传播](#set-global-propagator)
- [分布式链路传递协议汇总](#propagators-distribution)
  * [B3 必要条件](#b3-requirements)
    + [B3 提取](#b3-extract)
    + [B3 注入](#b3-inject)
    + [字段](#fields-1)
    + [配置](#configuration)

<!-- tocstop -->

</details>

## 综述

### 基本知识补充
关注点：如果系统按功能划分，关注点就是系统的核心功能。
横切关注点：从整个系统横向维度看到的核心功能。比如日志功能就是横切关注点的一个典型案例。日志功能往往横跨系统中的每个业务模块。


横切关注点(系统基本功能)通过Propagator可以传递它的状态给下一个进程，Propagator 是通过应用程序读写上下文数据进行数据的交换。
每个关注点都包含了一组Propagator
Propagator把每一个横切关注点通过上下文去注入和提取，链路和Baggage关系就是如此

链路传递一般通过特定的请求拦截器库来实现，拦截器通过拦截输入和输出请求，利用对Propagator的注入和提取操作来实现传递

Propagator API
用户可以通过检测库来实现Propagator操作的API


## Propagator 类型

`Propagator`的类型会被一个具体传输做严格定义，绑定到一个数据类型，这样才能做到跨进程间做上下文数据的传播


The Propagators API 目标定义了这样一些类型:

- `TextMapPropagator` 从Carrier注入和提取是一组 key/value 键值对

二进制`Propagator`类型将来也会支持，参考[#437](https://github.com/open-telemetry/opentelemetry-specification/issues/437).


### Carrier 搬运工具

Carrier 通过Propagator作为媒介去读写数据，Propagator定义了不同Carrier数据类型，比如字符map或者字节数组

通过Inject，可以改动Carriers


### 基本操作

`Propagator` 包含 `Inject` 、 `Extract` 基本操作, 分别对应从Carrier去写入和读取信息，每个`Propagator`必须定义具体Carrier数据类型和相关的额外参数。


#### Inject

将值注入到Carrier里. 比如说，写到HTTP请求的Headers里。

必须的参数:

- 上下文 `Context`. Propagator 首先从上下文 `Context`取回合适的值，这样上下文对象有：`SpanContext`, `Baggage`或者其它第三方的横切关注点  
- Carrier 具备保存传播信息的能力.比如一个输出消息或者一个HTTP请求.

#### Extract

Extracts 提取输入请求的值. 比如从HTTP请求的Headers里获取相应的值。

如果从Carrier中有一个值无法解析，为了保证得到横切关注点，禁止去抛出异常。同时，为了保留之前有效的值，不能在上下文`Context`里存储新值。



必须的参数:

- `Context`上下文
- Carrier 具备保存传播信息的能力.比如一个输入消息或者一个HTTP请求.

返回一个新的上下文`Context`对象，它派生于传给Extract方法的一个参数，这个参数就是一个上下文`Context`对象
这样上下文对象有：`SpanContext`, `Baggage`或者其它第三方的横切关注点

## TextMap Propagator

`TextMapPropagator` 通过字符串的key/values 键值对执行一个横切关注点的注入和抽取，Carrier通过它完成跨进程的传递.

Carrier在client和server间传递的数据往往是一个HTTP request.

为了提高兼容性，key/values 键值对只包含US-ASCII 字符，这样保证每次请求HTTP header的有效性，参考[RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.2).


`Getter` 和 `Setter` 分别帮助组件实现注入和提取，而且来自Carrier中独立的对象，避免运行时分配。
同时在访问上下文时，也不用单独实现Carrier 额外接口.


`Getter` 和 `Setter` 必须无状态和允许保存的是常量，这也是为了有效避免运行时分配资源。

### 字段

The predefined propagation fields. If your carrier is reused, you should delete the fields here
before calling [inject](#inject).

在Carrier 中字段定义成string键来识别特定格式的组件.

For example, if the carrier is a single-use or immutable request object, you don't need to
clear fields as they couldn't have been set before. If it is a mutable, retryable object,
successive calls should clear these fields first.
比如，Carrier 是一个单例或者不变的Request对象，你在做Set之前不需要去清除老的字段。
如果它是一个易变，回收的对象，要成功调用先要清除这些字段。

这里有些使用的案例

- 允许预先分配字段, 特别系统中的 gRPC Metadata
- allow a single-pass over an iterator

Returns list of fields that will be used by the `TextMapPropagator`.

Observe that some `Propagator`s may define, besides the returned values, additional fields with
variable names. To get a full list of fields for a specific carrier object, use the
[Keys](#keys) operation.

### TextMap 注入

注入一个值到Carrier中.  必须的参数和基本的[Inject](#inject) 操作一样.

可选的参数:

- 一个`Setter` 设置一个 传播键值对. Propagators 可以多次调用生成多对键值对.
  它可以帮助编程语言自由地注入数据到Carrier过程中

#### Setter 的参数

Setter  是注入 `Inject` 方法一个参数： 它把相应的值写入字段中.

`Setter` 允许一个 `TextMapPropagator` 对象把所有传递字段写入到Carrier.

具体一种实现：带有`Set`方法的一个`Setter`类。类似下面描述

##### Set

等价一个传递的Propagator

必要的参数:

- the carrier holding the propagation fields. For example, an outgoing message or an HTTP request.
- key 类型.
- value 字段.

方法实现应该保持大小写敏感，(比如，`Content-Type`不应该转化成`content-type`)，除非使用协议大小写不敏感，否则必须保持大小写敏感

### TextMap 提取

从一个输入请求中提取值。 它必须参数和基本的 [Extract](#extract) 操作一样.

可选参数:

- A `Getter` invoked for each propagation key to get. This is an additional
  argument that languages are free to define to help extract data from the carrier.

Returns a new `Context` derived from the `Context` passed as argument.

#### Getter 参数

Getter is an argument in `Extract` that get value from given field

`Getter` allows a `TextMapPropagator` to read propagated fields from a carrier.

One of the ways to implement it is `Getter` class with `Get` and `Keys` methods
as described below. Languages may decide on alternative implementations and
expose corresponding methods as delegates or other ways.

##### 关键字

The `Keys` function MUST return the list of all the keys in the carrier.

Required arguments:

- The carrier of the propagation fields, such as an HTTP request.

The `Keys` function can be called by `Propagator`s which are using variable key names in order to
iterate over all the keys in the specified carrier.

For example, it can be used to detect all keys following the `uberctx-{user-defined-key}` pattern, as defined by the
[Jaeger Propagation Format](https://www.jaegertracing.io/docs/1.18/client-libraries/#baggage).

##### Get

The Get function MUST return the first value of the given propagation key or return null if the key doesn't exist.

Required arguments:

- the carrier of propagation fields, such as an HTTP request.
- the key of the field.

The Get function is responsible for handling case sensitivity. If the getter is intended to work with a HTTP request object, the getter MUST be case insensitive.

## 独立接口来实现注入和提取

Languages can choose to implement a `Propagator` type as a single object
exposing `Inject` and `Extract` methods, or they can opt to divide the
responsibilities further into individual `Injector`s and `Extractor`s. A
`Propagator` can be implemented by composing individual `Injector`s and
`Extractors`.

## 复合Propagator

提供了把不同横切关注点的一组`Propagator`聚合成一个单体实例的实现。

A composite propagator can be built from a list of propagators, or a list of
injectors and extractors. The resulting composite `Propagator` will invoke the `Propagator`s, `Injector`s, or `Extractor`s, in the order they were specified.

Each composite `Propagator` will implement a specific `Propagator` type, such
as `TextMapPropagator`, as different `Propagator` types will likely operate on different
data types.

There MUST be functions to accomplish the following operations.

- Create a composite propagator
- Extract from a composite propagator
- Inject into a composite propagator

### Create a Composite Propagator

Required arguments:

- A list of `Propagator`s or a list of `Injector`s and `Extractor`s.

Returns a new composite `Propagator` with the specified `Propagator`s.

### Composite Extract

Required arguments:

- A `Context`.
- The carrier that holds propagation fields.

If the `TextMapPropagator`'s `Extract` implementation accepts the optional `Getter` argument, the following arguments are REQUIRED, otherwise they are OPTIONAL:

- The instance of `Getter` invoked for each propagation key to get.

### Composite Inject

Required arguments:

- A `Context`.
- The carrier that holds propagation fields.

If the `TextMapPropagator`'s `Inject` implementation accepts the optional `Setter` argument, the following arguments are REQUIRED, otherwise they are OPTIONAL:

- The `Setter` to set a propagation key/value pair. Propagators MAY invoke it multiple times in order to set multiple pairs.

## 全局 Propagators

凡是支持 `Propagator` 类型组件，OpenTelemetry API 要提供一种方式可以获取到`Propagator`.
检测库可以支持所有远程调用操作上下文的注入和提取.
通过各种依赖技术或者一个全局访问对象，Propagators 在各种语言平台上就可以被创建。

**Note:** 这样做法不值得提倡：在某些组件库使用专有的上下文传播协议或者硬编码的形式去使用一个具体的全局Propagator.
在某些场景下，组件库可能并不是选择API，而是直接用硬编码做上下文的注入和提取.


The OpenTelemetry API MUST use no-op propagators unless explicitly configured
otherwise. 上下文传播 propagation may be used for various telemetry signals -
traces, metrics, logging and more. Therefore, context propagation MAY be enabled
for any of them independently. For instance, a span exporter may be left
unconfigured, although the trace context propagation was configured to enrich logs or metrics.

Platforms such as ASP.NET may pre-configure out-of-the-box
propagators. If pre-configured, `Propagator`s SHOULD default to a composite
`Propagator` containing the W3C Trace Context Propagator and the Baggage
`Propagator` specified in the [Baggage API](../baggage/api.md#propagation).
These platforms MUST also allow pre-configured propagators to be disabled or overridden.

### Get 全局 Propagator

只要有`Propagator` 类型的地方这个方法一定要有的.

返回一个全局的 `Propagator`. 它通常是一个复合实例。

### Set 全局 Propagator

只要有`Propagator` 类型的地方这个方法一定要有的.

设置成一个全局 `Propagator` 实例.

必要的 :

- 一个 `Propagator`对象. 它通常是一个复合实例.

## 分发传递协议

下面所列的是Opentelemetry 官方支持或者扩展包能够支持的分发传递协议：



* [W3C TraceContext](https://www.w3.org/TR/trace-context/).  OpenTelemetry API可选的一种HTTP分发方案
* [W3C Baggage](https://w3c.github.io/baggage). OpenTelemetry API可选的一种分发方案
* [B3](https://github.com/openzipkin/b3-propagation).
* [Jaeger](https://www.jaegertracing.io/docs/latest/client-libraries/#propagation-format).

这里还有其它Propagators 在Opentelemetry 扩展包维护的分发方案：


* [OT Trace](https://github.com/opentracing?q=basic&type=&language=). 传递数据格式基于`OpenTracing` 链路格式，
但是不能直接使用`OpenTracing`，因为它生态里面数据格式并没有得到广泛的采纳。

其它Propagators还有 AWS X-Ray 链路Header 协议，Opentelemetry 核心库也支持维护


### B3 必要条件

B3 has both single and multi-header encodings. It also has semantics that do not
map directly to OpenTelemetry such as a debug trace flag, and allowing spans
from both sides of request to share the same id. To maximize compatibility
between OpenTelemetry and Zipkin implementations, the following guidelines have
been established for B3 context propagation.

#### B3 抽取

B3抽取数据时，Propagators:

* MUST attempt to extract B3 encoded using single and multi-header
  formats. The single-header variant takes precedence over
  the multi-header version.
* MUST preserve a debug trace flag, if received, and propagate
  it with subsequent requests. Additionally, an OpenTelemetry implementation
  MUST set the sampled trace flag when the debug flag is set.
* MUST NOT reuse `X-B3-SpanId` as the id for the server-side span.

#### B3 注入

注入数据到B3, Propagators:

* B3 使用单header格式默认注入数据
* 如果想使用多header模式，B3要修改默认的配置
* 不能重复定义 `X-B3-ParentSpanId` ： OpenTelemetry 不支持在Request 两端使用同样的一个Span值.

#### 字段

字段返回和对应数据格式相关的header的名字。header 就是用来做注入操作。

#### 配置

| Option    | 抽取 顺序 | 注入 格式 | 详细说明     |
|-----------|---------------|---------------| ------------------|
| B3 单header模式 | 单header, 多header  | 单header        | [Link][b3-single] |
| B3 多header模式  | 单header, 多header  | 多header        | [Link][b3-multi]  |

[b3-single]: https://github.com/openzipkin/b3-propagation#single-header
[b3-multi]: https://github.com/openzipkin/b3-propagation#multiple-headers

举个例子, 下面就是多Header的编码链路字段:
```
X-B3-TraceId: 80f198ee56343ba864fe8b2a57d3eff7
X-B3-ParentSpanId: 05e3ac9a4f6e3b90
X-B3-SpanId: e457b5a2e4d86bd1
X-B3-Sampled: 1
```
下面就是单Header模式，其实就是一个分割字符串:
```
b3: 80f198ee56343ba864fe8b2a57d3eff7-e457b5a2e4d86bd1-1-05e3ac9a4f6e3b90
```
