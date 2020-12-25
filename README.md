# OpenTelemetry 标准规范

<p align="center">
  <strong>
    <a href="https://github.com/open-telemetry/opentelemetry-specification">English Version<a/>
    &nbsp;&nbsp;&bull;&nbsp;&nbsp;
    <a href="https://github.com/open-telemetry/docs-cn">中文文档使用指南<a/>
    &nbsp;&nbsp;&bull;&nbsp;&nbsp;
    <a href="https://gitter.im/open-telemetry/docs-cn">参与中文文档贡献<a/>
  </strong>
</p>

![OpenTelemetry Logo](https://opentelemetry.io/img/logos/opentelemetry-horizontal-color.png)


对 OpenTelemetry 充满好奇? 去 [官网](https://opentelemetry.io) 一探究竟吧！

OpenTelemetry 规范描述了所有 OpenTelemetry 协议实现的跨语言要求和期望。对规范的实质性修改必须使用 [OpenTelemetry 增强提案(Enhancement Proposal)](https://github.com/open-telemetry/oteps) 流程。小的更改，如措辞优化、拼写/语法更正等，可以直接通过 Pull request 进行。

需要额外讨论的问题可以在协议例会中提出。对欧盟和美国时区友好的会议在每周二太平洋时间上午 8 点举行。会议记录在 [Google doc](https://docs.google.com/document/d/1-bCYkN-DWJq4jw1ybaDZYYmx-WAe6HnwfWbkm8d57v8/edit?usp=sharing) 中。对亚太时区友好的会议在太平洋时间每周二下午 4 点举行。参见 [OpenTelemetry calendar](https://github.com/open-telemetry/community#calendar)。

## 目录

- [Overview](specification/overview.md)
- [Glossary](specification/glossary.md) [100%]
- [Library Guidelines](specification/library-guidelines.md)
  - [Package/Library Layout](specification/library-layout.md)
  - [General error handling guidelines](specification/error-handling.md)
- API Specification
  - [Baggage](specification/baggage/api.md)
    - [Propagators](specification/context/api-propagators.md)
  - [Tracing](specification/trace/api.md) [50%]
  - [Metrics](specification/metrics/api.md)
- SDK Specification
  - [Tracing](specification/trace/sdk.md)
  - [Metrics](specification/metrics/sdk.md)
  - [Resource](specification/resource/sdk.md)
  - [Configuration](specification/sdk-configuration.md)
- Data Specification
  - [Semantic Conventions](specification/overview.md#semantic-conventions)
  - [Protocol](specification/protocol/README.md)
- About the Project
  - [Timeline](#project-timeline)
  - [Notation Conventions and Compliance](#notation-conventions-and-compliance)
  - [Versioning](#versioning)
  - [Acronym](#acronym)
  - [Contributions](#contributions)
  - [License](#license)

## Project Timeline

The current project status as well as information on notable past releases is found at
[the OpenTelemetry project page](https://opentelemetry.io/project-status/).

Information about current work and future development plans is found at the
[specification development milestones](https://github.com/open-telemetry/opentelemetry-specification/milestones).

## Notation Conventions and Compliance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in the [specification](./specification/overview.md) are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

An implementation of the [specification](./specification/overview.md) is not compliant if it fails to satisfy one or more of the "MUST", "MUST NOT", "REQUIRED", "SHALL", or "SHALL NOT" requirements defined in the [specification](./specification/overview.md).
Conversely, an implementation of the [specification](./specification/overview.md) is compliant if it satisfies all the "MUST", "MUST NOT", "REQUIRED", "SHALL", and "SHALL NOT" requirements defined in the [specification](./specification/overview.md).

## Versioning

Changes to the [specification](./specification/overview.md) are versioned according to [Semantic Versioning 2.0](https://semver.org/spec/v2.0.0.html) and described in [CHANGELOG.md](CHANGELOG.md). Layout changes are not versioned. Specific implementations of the specification should specify which version they implement.

Changes to the change process itself are not currently versioned but may be independently versioned in the future.

## Acronym

The official acronym used by the OpenTelemetry project is "OTel".

Please refrain from using "OT" in order to avoid confusion with the now deprecated "OpenTracing" project.

## Contributions

See [CONTRIBUTING.md](CONTRIBUTING.md) for details on contribution process.

## License

By contributing to OpenTelemetry Specification repository, you agree that your contributions will be licensed under its [Apache 2.0 License](https://github.com/open-telemetry/specification/blob/master/LICENSE).