# 属性与标签命名
对应版本时间 18 Nov 2020

<details>
<summary>
目录
</summary>

- [命名多元化原则](#name-pluralization-guidelines)
- [给OpenTelemetry开发者的建议](#recommendations-for-opentelemetry-authors)
- [给应用开发者的建议](#recommendations-for-application-developers)

</details>

_本节描述的命名规范适用于 资源，Span，日志属性名称（也被称为"attribute keys"），Metric标签（labels）的 key。为简洁起见，
在本节中，当我们使用没有形容词的术语“name”时，它意味着“属性名（attribute name）或Metric标签的key（metric label key）”。_

每个名称都必须是有效的Unicode序列。

_注意：我们只要求名称以Unicode序列表示。本规范没有定义Unicode序列如何精确编码。 
编码格式可以因为不同的编程语言而产生变化。 使用习惯性的编程语言或者编码格式来表示Unicode。_

命名多元化原则：

- 使用命名空间来避免命名的冲突。使用字符"."来分割命名空间。例如使用`service.version`来表示服务版本，
  而`service`是命名空间，`version`是命名空间内的属性名称。

- 命名空间是可以进行嵌套的。例如，telemetry.sdk顶级命名空间内是一个命名telemetry空间，
  而命名空间内telemetry.sdk.name是一个属性telemetry.sdk。
  注意: 一个实体位于另一个实体中这一情况，通常不足以成为形成嵌套名称空间的充分理由。
  命名空间的目的是避免名称冲突，而不是描述实体层次结构。 这一目的应该是形成嵌套的名称空间的主要理由。

- 请为每一个属性名为多单词点组成的组件，用下划线分隔单词（遇到`snake case`即使用`snake_case`）。
  例如`http.status_code`表示http名称空间中的状态代码。

- 名称不应该与名称空间重合。例如，如果 `service.instance.id`是属性名称，则名称为`service.instance`的属性就不会再生效，
  因为`service.instance`已经是名称空间。
  由于该规则，选择名称时要小心：每个现有的属性或者标签键在未来都不能再作为命名空间使用，
  反之亦然：任何现有名称空间都禁止存在同名属性或标签键。
  
## 命名多元化原则

- 当一个属性表示单个实体时，属性名称应为单数。例如：`host.name`, `db.user`, `container.id`。

- 当属性可以表示多个实体时，属性名称应为复数形式，值类型应为数组。
  例如，`process.command_args`可能包含多个值：可执行文件名称和命令参数。

- 当属性代表度量时， 应遵循
  [Metric Name Pluralization Guidelines(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/semantic_conventions/README.md#pluralization)
  作为属性名称。

## 给OpenTelemetry开发者的建议

- 属于OpenTelemetry语义约定的所有名称都应属于命名空间。

- 提出新的语义约定时，请确保检查
  [Resources(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/semantic_conventions/README.md)，
  [Spans(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/README.md)，
  和
  [Metrics(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/semantic_conventions/README.md)，
  以查看新名称是否适合。

- 当需要新的名称空间时，请考虑它应该是顶级命名空间（例如`service`）还是嵌套命名空间（例如`service.instance`）。

- 存在四个区域的语义约定：`Resource`，`Span`，日志属性名称，以及 `Metric`的标签keys。
  除此以外，对于`span`我们还有另外两个区域: 事件和链接属性名称。所有这些区域中相同的名称空间或名称必须具有相同的含义。
  例如，`http.method``span`属性名称表示与`http.method``metric`标签完全相同的概念，
  具有相同的数据类型和相同的可能值集（在这两种情况下，它都将HTTP协议的请求方法的值记录为字符串）。

- 语义约定必须将名称限制为可打印的基本拉丁字符（更确切地说， 为Unicode代码点的
  [U+0021 .. U+007E](https://en.wikipedia.org/wiki/Basic_Latin_(Unicode_block)#Table_of_characters)
  子集）。建议进一步将名称限制为以下Unicode代码点：拉丁字母，数字，下划线，点（作为名称空间分隔符）。

## 给应用开发者的建议

作为应用程序开发人员，当您需要记录属性或标签时，请先咨询有关
[Resources(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/semantic_conventions/README.md)，
[Spans(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/README.md)，
和
[Metrics(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/semantic_conventions/README.md)
的现有语义约定 。如果不存在合适的名称，则需要重新命名。为此，请考虑以下几种选择：

- 该名称特定于您的公司，也可能在公司外部使用。为避免与其他公司引入的名称冲突（在使用多个供应商的应用程序的分布式系统中），
  建议在新名称前加上公司的反向域名，例如 `com.acme.shopname`。

- 该名称特定于您的应用程序，仅在内部使用。如果您已经拥有一个内部公司流程来帮助您确保不会发生名称冲突，请随时关注。
  否则，如果应用程序名称在组织内合理唯一（例如`myuniquemapapp.longitude`可能很好），则建议在属性名称或标签键之前添加应用程序名称 。
  确保应用程序名称与现有的语义约定名称空间不冲突。

- 该名称通常可适用于行业中的应用程序。在这种情况下，请考虑向该规范提交提案，以在语义约定中添加新名称，并在必要时还添加新名称空间。

建议将名称限制为可打印的基本拉丁字符（更确切地说， 为Unicode代码点的
[U+0021 .. U+007E](https://en.wikipedia.org/wiki/Basic_Latin_(Unicode_block)#Table_of_characters)
子集）。
