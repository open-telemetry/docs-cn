# OpenTelemetry客户端设计原则

本文档定义了一些通用的原则，这些原则将帮助设计人员创建易用的OpenTelemetry客户端。这些原则在所有的语言中的都是一样的，但是在不同的语言的表达方式 上会有足够的灵活性

OpenTelemetry客户端希望的是所有的功能开箱即用，而且允许通过扩展的方式进行创新和试验

请先阅读[概述](overview.md)来了解下OpenTelemetry的基本架构

本文档并不会描述OpenTelemetry客户端API的详细信息和功能。对于API规范请参阅[API规范](../README.md)。

_给OpenTelemetry客户端作者们的注释：_ OpenTelemetry规范、API和SDK实现指南都还在研究的过程中，如果您发现信息不完整或缺失，内容矛盾，样式不一
致或其他任何缺陷，请通过创建issue或在[Gitter](https://gitter.im/open-telemetry/OpenTelemetry-specification)留言来告知规范的作者。
作为规范的实现者，你会对如何改进规范有一些独到的宝贵见解，技术成员委员会和规范SIG会高度重视您的意见，并欢迎您提供反馈。

## 要求

1. OpenTelemetry API必须定义明确，而且必须清楚地和具体的实现分离开。这可以让用户仅需要使用API而无需使用具体的实现（第二点和第三点将解释为什么这样设计很重要）。

2. 那些需要添加探针到它们代码的三方库或框架，仅仅需要依赖OpenTelemetry的API即可。三方库或框架的开发者并不需要关心（也没法知道）最终应用到程序的是OpenTelemetry的何种实现。

3. 最终应用程序开发者通常会决定如何配置OpenTelemetry SDk和使用哪种扩展方式。他们可以不选择任何OpenTelemetry的实现，即使应用程序或库已经增加了探针。
   因为在那些不想使用OpenTelemetry的应用程序中，增加了探针的三方库或框架必须仍然是完全可用的（这让框架开发者无需提供"已探针"和"未探针"两个版本了）。

4. 必须将SDK清楚地分离成与线路无关的部分，这些部分要实现通用逻辑（例如：批处理，通过处理信息来丰富标签）和依赖于协议的导出器exporter。Telemetry导出器exporter必须
   包含最小化的功能，这样才可以让提供商轻松地添加对其特定协议的支持

5. SDK的实现需要包含以下导出器exporter
    - OTLP
    - Jaeger
    - Zipkin
    - Prometheus
    - 标准输出（或日志），用于调试、测试、以及各种日志代理工具的输入

   注意：其中的一些是支持多种协议的（例如gRPC，Thrift等）。所以在exporter导出器中协议的确切实现列表是待定的。

   其他特定于供应商的导出器exporter（实现自供应商协议的exporter导出器）不应包含在OpenTelemetry客户端中，要将其放在其他的位置（未来将会定义存储和维护特定于供应商的导出器exporter的确切方法）

## OpenTelemetry客户端通用设计

这是OpenTelemetry客户端的通用设计（箭头表示调用）：

![OpenTelemetry客户端设计图](../internal/img/library-design.png)

### 期望的用途

OpenTelemetry客户端由4种类型的[程序包](glossary.md#packages)组成：API包，SDK包，语义预定包，插件包。只要API组件和SDK组件是分离的，
根据信号类型将API和SDK被分拆到多个包中也可以认为是一种具体的实现方式

被OpenTelemetry探针增强的库，框架，应用仅需要依赖API包。这些三方库的开发者通过调用API来产生telemetry数据

那些使用被OpenTelemetry API探针增强的三方库的应用可以控制是否装配OpenTelemetry SDK和生成telemetry数据。如果没有装配SDK，API调用的时候是 不会有实际操作的，以此保证最小的开销

为了使用telemetry，应用需要依赖OpenTelemetry SDK。应用还需要配置导出器exporter和其他插件，这样telemetry数据才能正确生成并发送到他们选择的分析工具。 至于插件如何开启和配置，不同的语言是不一样的。

### API与其最小化实现

API包是一个自给自足的包，因为仅依赖于API而不装配任何SDK的终端程序或者三方库依然是可以正常编译和运行的，虽然不会有任何数据发送到telemetry后端。

自给自足通过下面的这种方式来达成的。

依赖API的时候回包含一个最小化的API实现。当没有其他明确的API实现指定时，telemetry数据不会被收集。下图是这种情况下组件的工作方式

![最小化运作图](../internal/img/library-minimal.png)

从最小API实现中返回的数据都是有效的而且不需要调用者进行额外的检查是很重要的（例如createSpan()方法不能失败而且要返回一个非null的有效Span对象）。
调用者不需要知道或是担心最小化实现的具体生效原理。这可以最小化在探针代码中的样例和错误处理

同样重要的是，最小化实现会尽可能带来最小的性能损耗，所以被OpenTelemetry增强了的三方库和框架对那些并不想使用OpenTelemetry的用户来说，产生的开销是可以忽略不计的

### SDK实现

SDK实现是一个单独的(可选)依赖项。当被安装上后，它将替换最小实现被包含在API包中（确切的机制取决于语言）。

SDK实现了必须的核心功能，将API调用转换为可导出的telemetry数据。下图是SDK启用时各组件的工作情况：

![完整运作图](../internal/img/library-full.png)

SDK定义了一个[exporter接口](trace/sdk.md#span-exporter)。负责发送telemetry数据的特定于协议的导出器exporter必须实现此接口。

SDK还可以包含其他可选的辅助性导出器exporter，可以根据需要组装进SDK以实现额外的功能。

库的设计者需要基于[通用规范](trace/sdk.md#span-exporter)来定义各自语言的`Exporter`接口

#### 协议化的导出器Exporter

Telemetry后端提供者应实现[Exporter接口](trace/sdk.md#span-exporter)。通过Export()方法接收的数据需要进行序列化然后通过提供者指定的方式发送到后端。

鼓励提供者们将特定协议的导出器exporter设计得尽可能的简单，并实现想要的额外功能，例如排队和重试使用的是SDK中的帮助程序。

终端用户应被给与足够的灵活性，能够从队列、重试、打标、批处理方面选择出最适合应用程序的方案。例如，如果一个应用程序的Telemetry数据要发送到一个无法保证可用性
的后端程序，那用终端用户会选择持久化队列和一个支持失败重试的`导出器Exporter`。相反地，一个应用程序只是发送telemetry数据到本地的运行中的守护agent进程，那 用户会选择不带重试和排队的简单导出配置。

如果将SDK的其他导出导出器exporter作为单独的库提供，则库名前缀应根据相关技术的命名约定使用术语"OpenTelemetry"和"Exporter"。

例如：

- Python和Java：OpenTelemetry-exporter-jaeger
- JavaScript：@OpenTelemetry/exporter-jeager

#### 资源检测

鼓励云供应商们提供软件包以检测环境中的资源信息。这些功能必须在SDK包的外部进行实现。
详情请看[资源SDK](./resource/sdk.md#detecting-resource-information-from-the-environment)。

### 可选的实现

终端用户可能会选择直接依赖一个可选的实现。

SDK提供了许多在实现上可能会使用到的灵活性和扩展性。在开发一个其他的实现时，请查看OpenTelemetry提供的可扩展点。

一个可选实现的使用例子就是自动化测试。在自动化测试的时候可以将mock实现直接添加进来。例如，它可以将所有生成的telemetry数据存储到内存中且提供检查这些
存储数据的能力。这可以测试用例验证生成的telemetry数据是否正确生成。鼓励OpenTelemetry客户端开发者提供这样的一个mock实现。

注意：通过SDK和Mock`导出器Exporter`也可以进行mock，而无需替换出整个SDK。

mock方式的选择是基于测试目标和测试过程中真正想要拦截获取数据的点

### 版本标签

API包和SDK包必须使用语义版本编号。API包版本号和SDK包版本号是互相解耦的，可以不一样(它们也可以和自己所以实现的规范版本号不一样)。API包和SDK包必须标明它们各自的版本号

版本号的分离让OpenTelemetry客户端开发者们可以独立发布API和SDK，而无需协调、匹配规范的版本号

由于API和SDK包的版本是独立的，所以每一个API和SDK的release包必须明确指明它们所实现的规范版本号。此外，如果某一个SDK的版本只能与特定版本的API包兼容
的话，那么OpenTelemetry客户端开发者必须发布这些兼容信息。比如某个SDK包的发布信息中会这样写："SDK 0.3.4，配合API 0.1.0使用，实现自OpenTelemetry规范 0.1.0"

_TODO: 使用OpenTelemetry作为探针的三方库作者该如何指引他们的终端用户选择正确的SDK包

### 性能与阻塞

请参阅[性能与阻塞](performance.md)规范指南，以了解API的性能期望、达到这些期望的策略、以及描述在负载下是如何记录它们的行为的

### 并发与线程安全

请参阅各自相关的API规范指南。以了解API实现需要提供怎样的线程安全以及它们是如何被记录的

* [指标API](./metrics/api.md#concurrency)
* [链路追踪API](./trace/api.md#concurrency)
=======
<!-- # OpenTelemetry Client Design Principles -->
# OpenTelemetry客户端设计规范

<!-- This document defines common principles that will help designers create OpenTelemetry clients that are easy to use, are uniform across all supported languages, yet allow enough flexibility for language-specific expressiveness. -->
本文档定义了 OpenTelemetry 客户端设计的通用规范。目的是所有支持的语言的客户端保持统一，更容易使用，并且能允许保留对特定语言的特性支持的弹性。

<!-- OpenTelemetry clients are expected to provide full features out of the box and allow for innovation and experimentation through extensibility. -->
OpenTelemetry 客户端的目标是让所有的功能都一个开箱即用，并且在创新型的功能和实验性功能上保持的扩展性。

<!-- Please read the [overview](overview.md) first, to understand the fundamental architecture of OpenTelemetry. -->
请先阅读 [overview](overview.md) 以了解 OpenTelemetry 架构的基本原理。

<!-- This document does not attempt to describe the details or functionality of the OpenTelemetry client API. For API specs see the [API specifications](../README.md). -->
本文档不描述 OpenTelemetry 客户端 API 的功能和细节。此部分可以参阅 [API specifications](../README.md) 。

<!-- _Note to OpenTelemetry client Authors:_ OpenTelemetry specification, API and SDK implementation guidelines are work in progress. If you notice incomplete or missing information, contradictions, inconsistent styling and other defects please let specification writers know by creating an issue in this repository or posting in [Gitter](https://gitter.im/open-telemetry/opentelemetry-specification). As implementors of the specification you will often have valuable insights into how the specification can be improved. The Specification SIG and members of Technical Committee highly value your opinion and welcome your feedback. -->
_OpenTelemetry 客户端作者请注意:_ OpenTelemetry 的规范、 API 和 SDK 的实现指引还在修订中。如果您发现了信息缺失、冲突、样式不一致或者其他缺陷，请在本仓库创建 issue 或者在 [Gitter](https://gitter.im/open-telemetry/opentelemetry-specification) 上留言，以便文档的作者跟进问题。作为设计规范的实现者，通常您会对如何提升文档质量有更深入的理解，因此规范 SIG 和技术委员会成员会高度重视您的每一条意见和反馈。

<!-- ## Requirements -->
## 要求

<!-- 1. The OpenTelemetry API must be well-defined and clearly decoupled from the implementation. This allows end users to consume API only without also consuming the implementation (see points 2 and 3 for why it is important). -->
1. OpenTelemetry API 必须是清晰明确的并且和实现解耦。这样最终用户可以仅仅理解 API 而不需要关系实现。(第2和第3点解释了原因)。

<!-- 2. Third party libraries and frameworks that add instrumentation to their code will have a dependency only on the API of OpenTelemetry client. The developers of third party libraries and frameworks do not care (and cannot know) what specific implementation of OpenTelemetry is used in the final application. -->
2. 第三方库和框架的仅仅需要集成和依赖 OpenTelemetry 客户端 API。第三方库和框架的作者通常不关心（也无法知晓）OpenTelemetry 的最终使用场景和规范。

<!-- 3. The developers of the final application normally decide how to configure OpenTelemetry SDK and what extensions to use. They should be also free to choose to not use any OpenTelemetry implementation at all, even though the application and/or its libraries are already instrumented.  The rationale is that third-party libraries and frameworks which are instrumented with OpenTelemetry must still be fully usable in the applications which do not want to use OpenTelemetry (so this removes the need for framework developers to have "instrumented" and "non-instrumented" versions of their framework). -->
3. 通常由最终的软件开发者来决定如何配置 OpenTelemetry SDK 和使用哪些插件。允许他们接入的程序或者库时可以选择不开启任何 OpenTelemetry 的功能。其最终目的是为了让使用者在即便不想使用 OpenTelemetry 的时候也能直接使用其内部集成了 OpenTelemetry 的第三方库和框架(所以，这就不需要框架开发者分开提供集成和未集成 OpenTelemetry 的版本)。

<!-- 4. The SDK must be clearly separated into wire protocol-independent parts that implement common logic (e.g. batching, tag enrichment by process information, etc.) and protocol-dependent telemetry exporters. Telemetry exporters must contain minimal functionality, thus enabling vendors to easily add support for their specific protocol. -->
4. SDK 必须把与协议无关的公共逻辑（比如：批处理、按进程信息打标签）和与协议相关的遥测 Exporter 隔离开。遥测 Exporter 必须仅包含最小功能块，以便厂商可以更容易地增加私有协议的支持。

<!-- 5. The SDK implementation should include the following exporters:
    - OTLP.
    - Jaeger.
    - Zipkin.
    - Prometheus.
    - Standard output (or logging) to use for debugging and testing as well as an input for the various log proxy tools.
    - In-memory (mock) exporter that accumulates telemetry data in the local memory and allows to inspect it (useful for e.g. unit tests).

    Note: some of these support multiple protocols (e.g. gRPC, Thrift, etc). The exact list of protocols to implement in the exporters is TBD.

    Other vendor-specific exporters (exporters that implement vendor protocols) should not be included in OpenTelemetry clients and should be placed elsewhere (the exact approach for storing and maintaining vendor-specific exporters will be defined in the future). -->
5. SDK 实现应该包含以下的 Exporter:
    - OTLP.
    - Jaeger.
    - Zipkin.
    - Prometheus.
    - 标准输出 (或日志) 以用于调试和测试，也可以用于集成日志代理工具。
    - 内存 (Mock) Exporter ，可以用于在本地内存中收集和检查遥测数据。(比如用于单元测试)。

    注: 上面有一些 Exporter 包含多种协议 (比如 gRPC , Thrift , 等等)。具体 Exporter 需要实现哪些协议还处于待定状态。

    其他厂商定制化的 Exporter (实现了厂商私有协议) 不被包含在 OpenTelemetry 客户端列表中，其文档位于其他位置 (未来会确定保存和维护厂商定制化 Exporter 位置)。

<!-- ## OpenTelemetry Client Generic Design -->
## OpenTelemetry 客户端通用设计模型

<!-- Here is a generic design for an OpenTelemetry client (arrows indicate calls): -->
以下是 OpenTelemetry 客户端的通用设计模型 (调用关系):

<!-- ![OpenTelemetry client Design Diagram](../internal/img/library-design.png) -->
![OpenTelemetry 客户端通用设计图](internal/img/library-design.png)

### Expected Usage

<!-- The OpenTelemetry client is composed of 4 types of [packages](glossary.md#packages): API packages, SDK packages, a Semantic Conventions package, and plugin packages.
The API and the SDK are split into multiple packages, based on signal type (e.g. one for api-trace, one for api-metric, one for sdk-trace, one for sdk-metric) is considered an implementation detail as long as the API artifact(s) stay separate from the SDK artifact(s). -->
OpenTelemetry 客户端有4种 [包](glossary.md#packages) 组成: API 包, SDK 包, 语义转换包, 以及插件包。
API 和 SDK 被拆分成了多个包, 每个包都基于单个类型 (比如: 一个用于 api-trace, 一个用于 api-metric, 一个用于 sdk-trace, 一个用于 sdk-metric) ，并且把 API 包和 the SDK 分开实现。

<!-- Libraries, frameworks, and applications that want to be instrumented with OpenTelemetry take a dependency only on the API packages. The developers of these third-party libraries will make calls to the API to produce telemetry data. -->
集成 OpenTelemetry 的库、框架、和应用仅需要依赖 API 包。第三方库的开发者通过调用API来产生遥测数据。

<!-- Applications that use third-party libraries that are instrumented with OpenTelemetry API control whether or not to install the SDK and generate telemetry data. When the SDK is not installed, the API calls should be no-ops which generate minimal overhead. -->
如果应用使用的第三方库集成了 OpenTelemetry API，它也能控制是否安装 SDK 来生成遥测数据。如果 SDK 为安转，API 调用应该不做任何事情且保持最小化开销。

<!-- In order to enable telemetry the application must take a dependency on the OpenTelemetry SDK. The application must also configure exporters and other plugins so that telemetry can be correctly generated and delivered to their analysis tool(s) of choice. The details of how plugins are enabled and configured are language specific. -->
为了保证在应用开启遥测时必须依赖 OpenTelemetry SDK 。应用必须配置 Exporter 和其他的插件以确保遥测模块可以生成正确的数据并且传递给其选择的分析工具。如果开启和配置插件的细节由语言规范约束。

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
SDK 定义了[导出接口](#span-exporter)。协议专用的 Exporter 负责吧遥测数据发送到实现了改接口的后端。

<!-- SDK also includes optional helper exporters that can be composed for additional functionality if needed. -->
如果需要的话，SDK 也可以包含一些辅助的 Exporter 用于组装一些附加的功能。

<!-- Library designers need to define the language-specific `Exporter` interface based on [this generic specification](trace/sdk.md#span-exporter). -->
库的设计者需要基于 [这个通用规范](trace/sdk.md#span-exporter) 定义语言专用的 `Exporter` 接口。

<!-- #### Protocol Exporters-->
#### 协议 Exporter

<!-- Telemetry backend vendors are expected to implement [Exporter interface](trace/sdk.md#span-exporter). Data received via Export() function should be serialized and sent to the backend in a vendor-specific way. -->
我们期望遥测服务后端的供应商来实现 [导出接口](trace/sdk.md#span-exporter)。接收到的数据可以通过 ```Export()``` 函数来通过供应商专有的方式序列化并发送到后端。

<!-- Vendors are encouraged to keep protocol-specific exporters as simple as possible and achieve desirable additional functionality such as queuing and retrying using helpers provided by SDK. -->
我们鼓励供应商保持 Exporter 的协议规范尽可能简单，并且实现适当的扩展功能，比如排队机制和使用 SDK 提供的工具来做失败重试。

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
一个替代实现的例子是自动化测试。可以使用一个 Mock 的实现加到自动化测试中去。比如，在内存里保存所有的遥测数据并提供来复杂这些保存的数据的能力。这样可以让测试用例来检查遥测数据是否正确。我们鼓励 OpenTelemetry 客户端的作者来提供类似这样的 Mock 实现。

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
_TODO: 使用 OpenTelemetry 的第三方库的作者如何指引最终用户找到正确的 SDK 包？

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
