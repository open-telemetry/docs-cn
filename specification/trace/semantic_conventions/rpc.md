# RPC Span的语义规约

本文档定义了如何在Span中描述远程过程调用（也称为远程方法调用/ RMI）。

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [通用远程过程调用约定](#通用远程过程调用约定)
  * [Span名称](#Span名称)
  * [属性](#属性)
    + [服务名称](#服务名称)
  * [与HTTP Span的区别](#与http-span的区别)
- [gRPC](#grpc)
  * [属性](#grpc属性)
  * [状态](#grpc状态)
  * [事件](#事件)

<!-- tocstop -->

## 通用远程过程调用约定

一个远程调用分别用一个客户端的Span和一个服务端的Span来描述。

对于对外部发出的请求`SpanKind`必须设置为`CLIENT`，外部对内的请求设置为`SERVER`。

当被调用的服务和方法的名称已知并且可用时，远程过程调用只能用这些语义约定表示。

### Span名称

Span名称必须是完整的远程方法名称，格式如下：

```
$package.$service/$method
```

(`$service`不能包含点，而`$method`不能包含斜线)

如果没有包名或未知，则省略`$package`部分（包括`.`）。

Span名称的一些示例:

- `grpc.test.EchoService/Echo`
- `com.example.ExampleRmiService/exampleMethod`
- `MyCalcService.Calculator/Add` 由服务端上报  
  `MyServiceReference.ICalculator/Add` 由.NET WCF客户端调用上报
- `MyServiceWithNoPackage/theMethod`

### 属性

| 属性名 | 类型 | 描述 |  示例  | 是否必须? |
| --- | --- | --- | --- | --- |
| `rpc.system`   | string | 标识远程系统服务的字符串 | `"grpc"`, `"java_rmi"` or `"wcf"`. | 是 |
| `rpc.service`  | string | 所调用服务的完整名称包括包名（如果可用） | `myservice.EchoService` | 否，但推荐设置 |
| `rpc.method`   | string | 调用方法的名称 | `exampleMethod` | 否，但推荐设置 |
| `net.peer.ip`   | string | 远程端点ip地址，对于IPv4为点分十进制，对于IPv6为[RFC5952]（https://tools.ietf.org/html/rfc5952）| `127.0.0.1` | 见下文 |
| `net.peer.name` | string | 远程主机名或类似的，请参见下面的说明 | `example.com` | 见下文 |
| `net.peer.port` | number | 远程端口 | `80`; `8080`; `443` | 见下文 |
| `net.transport` | string | 使用的传输协议 | `IP.TCP` | 见下文 |

[网络属性][]至少需要`net.peer.name`或`net.peer.ip`之一。

对于客户端Spans，`net.peer.port`是必需的，并且前提是该连接是基于IP的并且该端口可用（它描述了它们所连接的服务器端口）。
对于服务端Spans，`net.peer.port` 可选的 (它描述了客户端连接的端口)。

此外，对于非IP连接（如命名管道绑定），需要设置[net.transport][]。

#### 服务名称

服务器进程在接收和处理远程调用时，`rpc.service`中提供的服务名称不一定必须与[`service.name`] []资源属性匹配。
一个进程可以暴露多个RPC端点，因此具有多个RPC服务名称。从部署角度来看，如`service.*`资源属性所表示的，将被视为具有一个`service.name`的一个已部署服务。
同样，在向服务器发送RPC请求的k客户端上，`rpc.service`提供的服务名称不要求一定与[`peer.service`][]span属性匹配。

例如，给定一个部署为`QuoteService`的进程，该名称将成为应用于整个流程的`service.name`资源属性的名称。
该进程可以暴露两个RPC端点，一个叫`CurrencyQuotes` (= `rpc.service`) 含有一个方法叫`getMeanRate` (= `rpc.method`) ，另一个端点叫 `StockQuotes`  (= `rpc.service`) 含有`getCurrentBid` and `getLastClose` (= `rpc.method`)两个方法。
在这个示例中，代表客户端请求的Spans应该设置它们 `peer.service`属性为 `QuoteService`，并与服务端的`service.name`资源属性相匹配。
通常，用户不应将`peer.service`设置为标准的RPC服务名称。

### 与HTTP Span的区别

通常只能使用[HTTP spans](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/http.md)来表示HTTP调用。

如果它们针对调用者已知的特定远程服务和方法，例如，当它是通过HTTP传输的远程过程调用时，`rpc.*`属性可以在该Span上额外添加，或在单独的RPC Span(传输HTTP调用的父RPC Span)中添加。

请注意，在这种情况下方法是关于被调用的远程方法，不是HTTP调用方法（如：GET，POST，等等）。

## gRPC

这部分是描述关于[gRPC][]的远程调用以及附加的一些约定。

`rpc.system` 属性必须设置为 `"grpc"`.

### gRPC属性

<!-- semconv rpc.grpc -->
| 属性  | 类型 | 描述  | 示例  | 是否必须 |
|---|---|---|---|---|
| `rpc.grpc.status_code` | number | gRPC请求的[状态码](https://github.com/grpc/grpc/blob/v1.33.2/doc/statuscodes.md) | `0`; `1`; `16` | 是 |

`rpc.grpc.status_code` 一定是下面列表之一:

| 值  | 描述 |
|---|---|
| `0` | OK |
| `1` | CANCELLED |
| `2` | UNKNOWN |
| `3` | INVALID_ARGUMENT |
| `4` | DEADLINE_EXCEEDED |
| `5` | NOT_FOUND |
| `6` | ALREADY_EXISTS |
| `7` | PERMISSION_DENIED |
| `8` | RESOURCE_EXHAUSTED |
| `9` | FAILED_PRECONDITION |
| `10` | ABORTED |
| `11` | OUT_OF_RANGE |
| `12` | UNIMPLEMENTED |
| `13` | INTERNAL |
| `14` | UNAVAILABLE |
| `15` | DATA_LOSS |
| `16` | UNAUTHENTICATED |
<!-- endsemconv -->

[gRPC]: https://grpc.io/

### gRPC状态

[Span 状态](../api.md#设置状态) 必须使用unset作为gRPC `OK` 状态码，其他的都是设置为`Error`。

### 事件

在一个gRPC的调用周期中应使用以下属性创建在客户端和服务器Span上发送和接收的每个消息事件。

```
-> [time],
    "name" = "message",
    "message.type" = "SENT",
    "message.id" = id
    "message.compressed_size" = <压缩后的容量（以字节为单位）>,
    "message.uncompressed_size" = <未压缩的容量，以字节为单位>
```

```
-> [time],
    "name" = "message",
    "message.type" = "RECEIVED",
    "message.id" = id
    "message.compressed_size" = <压缩后的容量（以字节为单位）>,
    "message.uncompressed_size" = <未压缩的容量，以字节为单位>
```

`message.id`必须以`1`开头的两个不同的计数器来计算。 此方法确保值在不同的实现中保持一致。 如果是单次调用，则客户端和服务器Span将仅记录已发送和已接收消息之一。
