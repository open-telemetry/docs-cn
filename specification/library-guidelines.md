<!-- # OpenTelemetry Client Design Principles -->
# OpenTelemetry客户端设计规范

<!-- This document defines common principles that will help designers create OpenTelemetry clients that are easy to use, are uniform across all supported languages, yet allow enough flexibility for language-specific expressiveness. -->
本文档定义了 OpenTelemetry 客户端设计的通用规范。目的是所有支持的语言的客户端保持统一，更容易使用，并且能允许保留对特定语言的特性支持的弹性。

<!-- OpenTelemetry clients are expected to provide full features out of the box and allow for innovation and experimentation through extensibility. -->
OpenTelemetry客户端的目标是让所有的功能都一个开箱即用，并且在创新型的功能和实验性功能上保持的扩展性。

<!-- Please read the [overview](overview.md) first, to understand the fundamental architecture of OpenTelemetry. -->
请先阅读 [overview](overview.md) 以了解 OpenTelemetry 架构的基本原理。

<!-- This document does not attempt to describe the details or functionality of the OpenTelemetry client API. For API specs see the [API specifications](../README.md). -->
本文档不描述 OpenTelemetry 客户端 API 的功能和细节。此部分可以参阅 [API specifications](../README.md) 。

<!-- _Note to OpenTelemetry client Authors:_ OpenTelemetry specification, API and SDK implementation guidelines are work in progress. If you notice incomplete or missing information, contradictions, inconsistent styling and other defects please let specification writers know by creating an issue in this repository or posting in [Gitter](https://gitter.im/open-telemetry/opentelemetry-specification). As implementors of the specification you will often have valuable insights into how the specification can be improved. The Specification SIG and members of Technical Committee highly value your opinion and welcome your feedback. -->
_OpenTelemetry 客户端作者请注意:_ OpenTelemetry 的规范、 API 和 SDK 的实现指引还在修订中。如果您发现了信息缺失、冲突、样式不一致或者其他缺陷，请在本仓库创建 issue 或者在 [Gitter](https://gitter.im/open-telemetry/opentelemetry-specification) 上留言，以便文档的作者跟进问题。作为设计规范的实现者，通常您会对如何提升文档质量有更深入的理解，因此规范 SIG 和技术委员会成员会高度重视您的每一条意见和反馈。

<!-- ## Requirements -->
## 要求

<!-- 1. The OpenTelemetry API must be well-defined and clearly decoupled from the implementation. This allows end users to consume API only without also consuming the implementation (see points 2 and 3 for why it is important). -->
1. OpenTelemetry API 必须是清晰明确的并且和实现解耦。这样最终用户可以仅仅理解API而不需要关系实现。(第2和第3点解释了原因).

<!-- 2. Third party libraries and frameworks that add instrumentation to their code will have a dependency only on the API of OpenTelemetry client. The developers of third party libraries and frameworks do not care (and cannot know) what specific implementation of OpenTelemetry is used in the final application. -->
2. 第三方库和框架的仅仅需要集成和依赖OpenTelemetry客户端 API。第三方库和框架的作者通常不关心（也无法知晓）OpenTelemetry的最终使用场景和规范。

<!-- 3. The developers of the final application normally decide how to configure OpenTelemetry SDK and what extensions to use. They should be also free to choose to not use any OpenTelemetry implementation at all, even though the application and/or its libraries are already instrumented.  The rationale is that third-party libraries and frameworks which are instrumented with OpenTelemetry must still be fully usable in the applications which do not want to use OpenTelemetry (so this removes the need for framework developers to have "instrumented" and "non-instrumented" versions of their framework). -->
3. 通常由最终的软件开发者来决定如何配置 OpenTelemetry SDK 和使用哪些插件。允许他们接入的程序或者库时可以选择不开启任何 OpenTelemetry 的功能。其最终目的是为了让使用者在即便不想使用 OpenTelemetry 的时候也能直接使用其内部集成了 OpenTelemetry 的第三方库和框架(所以，这就不需要框架开发者分开提供集成和未集成 OpenTelemetry 的版本)。

<!-- 4. The SDK must be clearly separated into wire protocol-independent parts that implement common logic (e.g. batching, tag enrichment by process information, etc.) and protocol-dependent telemetry exporters. Telemetry exporters must contain minimal functionality, thus enabling vendors to easily add support for their specific protocol. -->
4. SDK 必须把与协议无关的公共逻辑（比如：批处理、按进程信息打标签）和与协议相关的遥测 exporter 隔离开。遥测 exporter 必须仅包含最小功能块，以便厂商可以更容易地增加私有协议的支持。

<!-- 5. The SDK implementation should include the following exporters:
    - OTLP.
    - Jaeger.
    - Zipkin.
    - Prometheus.
    - Standard output (or logging) to use for debugging and testing as well as an input for the various log proxy tools.
    - In-memory (mock) exporter that accumulates telemetry data in the local memory and allows to inspect it (useful for e.g. unit tests).

    Note: some of these support multiple protocols (e.g. gRPC, Thrift, etc). The exact list of protocols to implement in the exporters is TBD.

    Other vendor-specific exporters (exporters that implement vendor protocols) should not be included in OpenTelemetry clients and should be placed elsewhere (the exact approach for storing and maintaining vendor-specific exporters will be defined in the future). -->
5. SDK 实现应该包含以下的 exporter:
    - OTLP.
    - Jaeger.
    - Zipkin.
    - Prometheus.
    - 标准输出 (或日志) 以用于调试和测试，也可以用于集成日志代理工具。
    - 内存 (mock) exporter，可以用于在本地内存中收集和检查遥测数据。(比如用于单元测试).

    注: 上面有一些 exporter 包含多种协议 (比如 gRPC, Thrift, 等等)。具体 exporter 需要实现哪些协议还处于待定状态。

    其他厂商定制化的 exporter (实现了厂商私有协议) 不被包含在 OpenTelemetry 客户端列表中，其文档位于其他位置 (未来会确定保存和维护厂商定制化 exporter 位置)。

<!-- ## OpenTelemetry Client Generic Design -->
## OpenTelemetry 客户端通用设计模型

<!-- Here is a generic design for an OpenTelemetry client (arrows indicate calls): -->
以下是 OpenTelemetry 客户端的通用设计模型 (调用关系):

<!-- ![OpenTelemetry client Design Diagram](../internal/img/library-design.png) -->
![OpenTelemetry 客户端通用设计图](internal/img/library-design.png)

### Expected Usage

<!-- The OpenTelemetry client is composed of 4 types of [packages](glossary.md#packages): API packages, SDK packages, a Semantic Conventions package, and plugin packages.
The API and the SDK are split into multiple packages, based on signal type (e.g. one for api-trace, one for api-metric, one for sdk-trace, one for sdk-metric) is considered an implementation detail as long as the API artifact(s) stay separate from the SDK artifact(s). -->
OpenTelemetry 客户端有4种 [包](glossary.md#packages) 组成: API 包, SDK 包, 语义转换包, 以及插件包.
API 和 SDK 被拆分成了多个包, 每个包都基于单个类型 (比如: 一个用于 api-trace, 一个用于 api-metric, 一个用于 sdk-trace, 一个用于 sdk-metric) ，并且把 API 包和 the SDK 分开实现。

<!-- Libraries, frameworks, and applications that want to be instrumented with OpenTelemetry take a dependency only on the API packages. The developers of these third-party libraries will make calls to the API to produce telemetry data. -->
集成 OpenTelemetry 的库、框架、和应用仅需要依赖 API 包。第三方库的开发者通过调用API来产生遥测数据。

<!-- Applications that use third-party libraries that are instrumented with OpenTelemetry API control whether or not to install the SDK and generate telemetry data. When the SDK is not installed, the API calls should be no-ops which generate minimal overhead. -->
如果应用使用的第三方库集成了 OpenTelemetry API，它也能控制是否安装 SDK 来生成遥测数据。如果 SDK 为安转，API 调用应该不做任何事情且保持最小化开销。

<!-- In order to enable telemetry the application must take a dependency on the OpenTelemetry SDK. The application must also configure exporters and other plugins so that telemetry can be correctly generated and delivered to their analysis tool(s) of choice. The details of how plugins are enabled and configured are language specific. -->
为了保证在应用开启遥测时必须依赖 OpenTelemetry SDK 。应用必须配置 exporter 和其他的插件以确保遥测模块可以生成正确的数据并且传递给其选择的分析工具。如果开启和配置插件的细节由语言规范约束。

<!-- ### API and Minimal Implementation -->
### API 和最小实现

<!-- The API package is a self-sufficient dependency, in the sense that if the end-user application or a third-party library depends only on it and does not plug a full SDK implementation then the application will still build and run without failing, although no telemetry data will be actually delivered to a telemetry backend. -->
API 包是自依赖的。即如果最终用户或者第三方库依赖依赖 API包，它可以在不加载SDK实现的情况下编译通过且正常运行，即便这种情况实际上不会生成和传递遥测数据。

<!-- This self-sufficiency is achieved the following way. -->
自依赖的通过以下方式实现。

<!-- The API dependency contains a minimal implementation of the API. When no other implementation is explicitly included in the application no telemetry data will be collected. Here is what active components look like in this case: -->
API 依赖包含API的最小实现。如果没有其他的实现被显式包含进应用，那么就不会手机任何数据。以下是刚才提及的组件的关系：

![Minimal Operation Diagram](internal/img/library-minimal.png)

<!-- It is important that values returned from this minimal implementation of API are valid and do not require the caller to perform extra checks (e.g. createSpan() method should not fail and should return a valid non-null Span object). The caller should not need to know and worry about the fact that minimal implementation is in effect. This minimizes the boilerplate and error handling in the instrumented code. -->
有一个非常重要的点是，这个API的最小实现返回的数据应该是有效的，并且不需要调用者做额外的检查（比如： ```createSpan()``` 方法不会失败并且总是返回一个有效的非空 ```Span``` 对象）。调用者不需要知道也无需单向这个最小实现的实现细节。这样能在集成过程中最小化实验代码和错误处理。

<!-- It is also important that minimal implementation incurs as little performance penalty as possible, so that third-party frameworks and libraries that are instrumented with OpenTelemetry impose negligible overheads to users of such libraries that do not want to use OpenTelemetry too. -->
还有一个要点是最小化实现要尽可能减少开销。这样集成了 OpenTelemetry 的第三方库和框架在提供给用户时，即便用户不想开启 OpenTelemetry 功能，其开销也可以忽略不计。

<!-- ### SDK Implementation -->
### SDK 实现

<!-- SDK implementation is a separate (optional) dependency. When it is plugged in it substitutes the minimal implementation that is included in the API package (exact substitution mechanism is language dependent). -->
SDK 实现是一个独立（可选）的依赖项。当它被接入后，他将会替代掉同样的最小实现的 API 包（具体的替代机制因语言而异）。

<!-- SDK implements core functionality that is required for translating API calls into telemetry data that is ready for exporting. Here is how OpenTelemetry components look like when SDK is enabled: -->
SDK 实现是生成用于导出的遥测数据的 API 调用的核心功能。以下是当 SDK 功能开启后， OpenTelemetry 组件的结构：

![Full Operation Diagram](internal/img/library-full.png)

<!-- SDK defines an [Exporter interface](trace/sdk.md#span-exporter). Protocol-specific exporters that are responsible for sending telemetry data to backends must implement this interface. -->
SDK 定义了[导出接口](#span-exporter)。协议专用的 exporter 负责吧遥测数据发送到实现了改接口的后端。

<!-- SDK also includes optional helper exporters that can be composed for additional functionality if needed. -->
如果需要的话，SDK 也可以包含一些辅助的 exporter 用于组装一些附加的功能。

<!-- Library designers need to define the language-specific `Exporter` interface based on [this generic specification](trace/sdk.md#span-exporter). -->
库的设计者需要基于 [这个通用规范](trace/sdk.md#span-exporter) 定义语言专用的 `Exporter` 接口。

<!-- #### Protocol Exporters-->
#### 协议 Exporter

<!-- Telemetry backend vendors are expected to implement [Exporter interface](trace/sdk.md#span-exporter). Data received via Export() function should be serialized and sent to the backend in a vendor-specific way. -->
我们期望遥测服务后端的供应商来实现 [导出接口](trace/sdk.md#span-exporter)。接收到的数据可以通过 ```Export()``` 函数来通过供应商专有的方式序列化并发送到后端。

<!-- Vendors are encouraged to keep protocol-specific exporters as simple as possible and achieve desirable additional functionality such as queuing and retrying using helpers provided by SDK. -->
我们鼓励供应商保持 exporter 的协议规范尽可能简单，并且实现适当的扩展功能，比如排队机制和使用 SDK 提供的工具来做失败重试。

<!-- End users should be given the flexibility of making many of the decisions regarding the queuing, retrying, tagging, batching functionality that make the most sense for their application. For example, if an application's telemetry data must be delivered to a remote backend that has no guaranteed availability the end user may choose to use a persistent local queue and an `Exporter` to retry sending on failures. As opposed to that for an application that sends telemetry to a locally running Agent daemon, the end user may prefer to have a simpler exporting configuration without retrying or queueing. -->
最合理的情况是让应用程序的最终用户可以在排队、重试、标签、批处理等功能中做出选择。比如，如果一个应用的遥测数据必须被发送到不保证可用性的远程的后端。用户可以选择使用一个本地持久化队列和一个可以在失败后重新发送数据的 `Exporter` 。或者另一种方案是应用可以把数据发送到一个本地的代理服务，这样最终用户就可以使用一个更简单的不包含重试和排队的导出配置。

<!-- If additional exporters for the sdk are provided as separate libraries, the
name of the library should be prefixed with the terms "OpenTelemetry" and "Exporter" in accordance with the naming conventions of the respective technology. -->
SDK 应当被拆分成多个不同的库，除此以外，库的名字应当按相关技术的命名规范，把术语 "OpenTelemetry" 和 "Exporter" 包含在前缀中。

<!-- For example:-->
比如:

- Python 和 Java: opentelemetry-exporter-jaeger
- Javascript: @opentelemetry/exporter-jeager

<!-- #### Resource Detection -->
#### 资源检测

<!-- Cloud vendors are encouraged to provide packages to detect resource information from the environment. These MUST be implemented outside of the SDK. See [Resource SDK](./resource/sdk.md#detecting-resource-information-from-the-environment) for more details. -->
我们鼓励云服务提供商提供包来从环境中检测资源信息。这些包必须在 SDK 外实现。更多细节参见 [资源 SDK](./resource/sdk.md#detecting-resource-information-from-the-environment)

<!-- ### Alternative Implementations -->
### 替代实现

<!-- The end-user application may decide to take a dependency on alternative implementation. -->
最终用户的应用程序可能会想要依赖替代实现。

<!-- SDK provides flexibility and extensibility that may be used by many implementations. Before developing an alternative implementation, please, review extensibility points provided by OpenTelemetry. -->
SDK 提供了一定的弹性和扩展功能并且被很多的实现使用了。在开发一个新的替代实现前，请再检视一下 OpenTelemetry 提供的扩展点。

<!-- An example use-case for alternate implementations is automated testing. A mock implementation can be plugged in during automated tests. For example, it can store all generated telemetry data in memory and provide a capability to inspect this stored data. This will allow the tests to verify that the telemetry is generated correctly. OpenTelemetry client authors are encouraged to provide such a mock implementation. -->
一个替代实现的例子是自动化测试。可以使用一个 Mock 的实现加到自动化测试中去。比如，在内存里保存所有的遥测数据并提供来复杂这些保存的数据的能力。这样可以让测试用例来检查遥测数据是否正确。我们鼓励 OpenTelemetry 客户端的作者来提供类似这样的 mock 实现。

<!-- Note that mocking is also possible by using SDK and a Mock `Exporter` without needing to swap out the entire SDK. -->
注意: 需要 Mock 功能也可以通过使用 SDK 中的 Mock `Exporter` 而不需要换掉整个 SDK。

<!-- The mocking approach chosen will depend on the testing goals and at which point exactly it is desirable to intercept the telemetry data path during the test. -->
Mock 方案的选择取决于在测试目标和测试期间希望拦截哪些遥测数据。

<!-- ### Version Labeling -->
### 版本标签

<!-- API and SDK packages must use semantic version numbering. API package version number and SDK package version number are decoupled and can be different (and they both can be also different from the Specification version number that they implement). API and SDK packages MUST be labeled with their own version number. -->
API 和 SDK 包必须采用语义化版本控制规范。API 包的版本号和 SDK 包的版本号是解耦的，所以可以不同（并且他们都可以和他们实现的规范的版本号不同）。API 和 SDK 包必须各自都有自己的版本标签。

<!-- This decoupling of version numbers allows OpenTelemetry client authors to make API and SDK package releases independently without the need to coordinate and match version numbers with the Specification. -->
解耦版本号管理可以让 OpenTelemetry 客户端的作者创建独立的 API 和 SDK 包，且不需要与规范的版本号相协调和匹配。

<!-- Because API and SDK package version numbers are not coupled, every API and SDK package release MUST clearly mention the Specification version number that they implement. In addition, if a particular version of SDK package is only compatible with a specific version of API package, then this compatibility information must be also published by OpenTelemetry client authors. OpenTelemetry client authors MUST include this information in the release notes. For example, the SDK package release notes may say: "SDK 0.3.4, use with API 0.1.0, implements OpenTelemetry Specification 0.1.0". -->
因为 API 和 SDK 包的版本号并不耦合，所以每个 API 和 SDK 发布包必须显式声明他们实现的规范版本。并且如果特定版本的 SDK 包仅兼容某个特定版本的 API 包，这些兼容性信息也必须由 OpenTelemetry 客户端的作者公布出来。OpenTelemetry 客户端的作者必须把这些兼容性信息公布在发行说明里。比如，SDK 包的发行说明里可以说“SDK 0.3.4 匹配 API 0.1.0，实现了 OpenTelemetry 规范 0.1.0”。

<!-- _TODO: How should third-party library authors who use OpenTelemetry for instrumentation guide their end users to find the correct SDK package?_ -->
_TODO: 使用 OpenTelemetry 的第三方库的作者如何指引最终用户找到正确的SDK包？

<!-- ### Performance and Blocking -->
### 性能和阻塞

<!-- See the [Performance and Blocking](performance.md) specification for
guidelines on the performance expectations that API implementations should meet, strategies for meeting these expectations, and a description of how implementations should document their behavior under load. -->
API 实现中对性能的预估、实现策略和对低负载的时的行为描述见 [性能和阻塞](performance.md) 的规格说明。

<!-- ### Concurrency and Thread-Safety -->
### 并发和线程安全

<!-- Please refer to individual API specification for guidelines on what concurrency
safeties should API implementations provide and how they should be documented: -->
关于API 实现对并发安全的包保证，请参考各个组件特定的 API 规范指引和相关章节：

* [Metrics API](./metrics/api.md#concurrency)
* [Tracing API](./trace/api.md#concurrency)