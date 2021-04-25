# Trace 语义规约

使用OpenTelemetry，实现者可以自由创建Span并使用特定于所表示操作的属性对其进行注释。 Span表示系统内部或系统之间的特定操作。 一些操作代表众所周知的协议，例如HTTP和数据库调用。 监视系统中需要什么附加信息才能正确表示和分析跨度，取决于协议和操作类型。 统一如何用不同的语言创建属性也很重要。 它允许工作人员比较和交叉分析多粒度微服务环境，而无需学习多种语言或遥测技术。

Span定义以下语义约定：

* [General](./span-general.md)：描述不同类型的操作时使用的通用语义属性。
* [HTTP](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/http.md)：HTTP客户端/服务器中的Span。
* [Database](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/database.md)：SQL或NoSQL客户端调用的Span。
* [RPC/RMI](rpc.md)：远程调用的Span （例如，gRPC）。
* [Messaging](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/messaging.md)：交互式消息系统的Span （例如消息队列, 发布/订阅, 等等..）。
* [FaaS](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/faas.md)：以函数作为服务的Span （例如， AWS Lambda）。
* [Exceptions](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/exceptions.md)：用于记录与Span相关联的异常属性。

除了trace和[metrics]（../../metrics/semantic_conventions/README.md）的语义约定外，OpenTelemetry还具有[Resources资源](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/resource/sdk.md)的综合概念。 也通过[资源语义约定](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/resource/semantic_conventions/README.md)进行定义。
