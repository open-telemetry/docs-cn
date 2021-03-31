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