# OpenTelemetry 标准规范

<p align="center">
  <strong>
    <a href="https://github.com/open-telemetry/opentelemetry-specification">English Version<a/>
    &nbsp;&nbsp;&bull;&nbsp;&nbsp;
    <a href="https://github.com/open-telemetry/docs-cn">中文文档使用指南<a/>
    &nbsp;&nbsp;&bull;&nbsp;&nbsp;
    <a href="https://github.com/open-telemetry/docs-cn/blob/main/community/Readme.md">参与中文文档贡献<a/>
  </strong>
</p>

![OpenTelemetry Logo](https://opentelemetry.io/img/logos/opentelemetry-horizontal-color.png)


对 OpenTelemetry 充满好奇? 去 [官网](https://opentelemetry.io) 一探究竟吧！

OpenTelemetry 规范描述了所有 OpenTelemetry 协议实现的跨语言要求和期望。对规范的实质性修改必须使用 [OpenTelemetry 增强提案(Enhancement Proposal)](https://github.com/open-telemetry/oteps) 流程。小的更改，如措辞优化、拼写/语法更正等，可以直接通过 Pull request 进行。

需要额外讨论的问题可以在协议例会中提出。对欧盟和美国时区友好的会议在每周二太平洋时间上午 8 点举行。会议记录在 [Google doc](https://docs.google.com/document/d/1-bCYkN-DWJq4jw1ybaDZYYmx-WAe6HnwfWbkm8d57v8/edit?usp=sharing) 中。对亚太时区友好的会议在太平洋时间每周二下午 4 点举行。参见 [OpenTelemetry calendar](https://github.com/open-telemetry/community#calendar)。

## 目录

- [Overview](overview.md)  ✅
- [Glossary](glossary.md)  ✅
- [Library Guidelines](library-guidelines.md)
  - [Package/Library Layout](library-layout.md)
  - [General error handling guidelines](error-handling.md)
- API Specification
  - [Propagators](context/api-propagators.md)
    - [Context](context/context.md)
  - [Baggage](baggage/api.md)
  - [Tracing](trace/api.md)  🚧
  - [Metrics](metrics/api.md)
- SDK Specification
  - [Tracing](trace/sdk.md)
  - [Metrics](metrics/sdk.md)
  - [Resource](resource/sdk.md)
  - [Configuration](sdk-configuration.md)
- Data Specification
  - [Semantic Conventions](overview.md#semantic-conventions)
  - [Protocol](protocol/README.md)
- About the Project
  - [项目 Timeline](#项目-Timeline)
  - [符号约定和合规性](#符号约定和合规性)
  - [版本控制](#版本控制)
  - [缩略语](#缩略语)
  - [参与贡献](#参与贡献)
  - [License](#license)

## 项目 Timeline

当前项目状态与过去各重要版本的信息可以从以下网址中获取。[The OpenTelemetry project page](https://opentelemetry.io/project-status/).

当前项目工作和未来的发展计划信息可以在 [Specification development milestones](https://github.com/open-telemetry/opentelemetry-specification/milestones) 中查阅。

## 符号约定和合规性

规范中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"NOT RECOMMENDED"、"MAY "和 "OPTIONAL"，当且仅当它们以全大写字母出现时，应按 [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] 中所述进行解释。

如果一个规范的实现未能满足规范中定义的 MUST", "MUST NOT", "REQUIRED", "SHALL", 或 "SHALL NOT" 中的一项或多项要求，则该实现是不合规。反之，如果一个规范的实现满足规范中定义的所有 MUST", "MUST NOT", "REQUIRED", "SHALL", 或 "SHALL NOT" 要求，则该规范的实现是合规的。

> 中文文档额外提示: 对本章节中的用词翻译有具体说明。请参考 中文文档使用指南。

## 版本控制

版本号的变更将遵守 Semantic Versioning 2.0，并将会在 [CHANGELOG.md](CHANGELOG.md) 中描述。布局变更将不进行版本控制。本规范的具体实现应指定它们所实现的规范版本。

变更过程本身的变更目前还没有版本化，但将来可能会独立版本化。

## 缩略语

OpenTelemetry 项目的正式官方缩写为 "OTel"。

请不要使用 “OT”，以避免和现有已废弃的 “OpenTracing” 项目相混淆。

## 参与贡献

关于如何参与 OpenTelemetry 标准制定项目，参见 [CONTRIBUTING.md](CONTRIBUTING.md)

> 中文文档额外提示：对中文文档的贡献，请参考 参与中文文档贡献

## License

您需要同意您对 OpenTelemetry Specification 项目的贡献，将在 [Apache 2.0 License ](https://github.com/open-telemetry/specification/blob/master/LICENSE)许可证下。

> 您对 OpenTelemetry doc-cn 项目的贡献，也将在  [Apache 2.0 License ](https://github.com/open-telemetry/specification/blob/master/LICENSE)许可证下。
