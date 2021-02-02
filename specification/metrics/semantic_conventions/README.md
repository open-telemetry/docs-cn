# Metrics 语义约定

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [一般准则](#general-guidelines)
  * [单位](#units)
  * [多元化](#pluralization)
- [一般 Metric 语义约定](#general-metric-semantic-conventions)
  * [instrument 命名](#instrument-naming)
  * [instrument 单位](#instrument-units)

<!-- tocstop -->

The following semantic conventions surrounding metrics are defined:

* [HTTP Metrics](http-metrics.md): 语义约定和`instruments`的`HTTP``metrics`.
* [系统 Metrics](system-metrics.md): 语义约定和`instruments`的标准系统`metrics`.
* [过程 Metrics](process-metrics.md): 语义约定和`instruments`的标准过程`metrics`.
* [运行时环境 Metrics](runtime-environment-metrics.md): 语义约定和`instruments`的运行时环境`metrics`.

除了`metrics`和
[traces](../../trace/semantic_conventions/README.md)
的语义约定之外，OpenTelemetry还定义了使用其自己的
[资源语义约定(待翻译)](../../resource/semantic_conventions/README.md)对
[资源(待翻译)](../../resource/sdk.md)
进行总体化的概念

## 一般准则

`Metric`的名称和标签存在于一个独立的领域和一个独立的层次结构中. 
`Metric`的名称和标签必须要考虑现有的领域内的名称。
在新增`Metric`的名称与标签时，必须考虑现有技术的框架/仓库。

关联`metrics`应根据其用法嵌套在层次结构中。为常见的`metric`类别定义顶层层次结构：
用于系统`metrics`，例如CPU和网络；适用于应用运行时，例如GC内部。

`metrics`定义了具有层次结构的命名空间。支持OpenTelemetry工件可以为某些类别的指标定义指标结构和层次结构，这些可以帮助您在创建将来的指标时做出决策。
`OpenTelemetry`组件支持定义某些`metric`的结构和层次结构，这些可以帮助您在后续再次创建`metric`时做出更好的决策。

通用标签应统一命名。这有助于发现并消除相似标签与`metrics`名称的歧义。

正如`Prometheus`所建议的那样，在对所有的`metric`命名时都是需要具有含义的。
["METRIC 与标签命名"](https://prometheus.io/docs/practices/naming/#metric-names) as

应当避免语义歧义。使用带有前缀的度量名称时，类似的度量标准在现有的度量标准中具有显著的不同。
例如，每个垃圾回收器在运行时都会存在略微不同的策略和执行方式，对于GC使用一个单一的`metric`名称集合，
不按照时间划分，会导致用户在使用时对于理解其`metric`含义时的混乱（例如，优先使用`runtime.java.gc*`而不是`runtime.gc.*`）。
某些操作系统的指标也是同样的模凌两可。

### 单位

常规`metrics`或将其单位包含在OpenTelemetry元数据`metric.WithUnit`中的`metrics`（例如go语言），不应该在度量名称中包含这些单位。
当单位为`metrics`名称提供其他含义时，可以包含单位。最重要的是，`metrics`必须是可以理解和使用的。

使用OpenMetrics展示格式构建在OpenTelemetry和系统之间可互相操作的组件时，请使用
[《 OpenMetrics指南》](./openmetrics-guidelines.md)。

### 多元化

度量名称不应被复数，除非所记录的值表示
[可计数](https://en.wikipedia.org/wiki/Count_noun).
的离散实例 。通常，仅当所讨论指标的单位为非单位（如`faults`或`operations`）时，名称才应该复数。

例子：
* `system.filesystem.utilization`，，`http.server.duration`和`system.cpu.time` ，即使记录了很多数据点也不应复数。
* `system.paging.faults`，`system.disk.operations`和`system.network.packets` 应该被复数，即使仅记录一个数据点也是如此。

## 一般 Metric 语义约定

以下语义约定旨在保持命名的一致性。它们为本规范中的大多数情况提供了指南，对于本文档中未明确定义的其他工具，应遵循这些指南。

### instrument 命名

- **极限量** - 一种测量常量极限的`instrument`，已知总量的`instrument``entity.limit`。例如，`system.memory.limit` 用于系统上的内存总量。

- **已使用量** - 一种用来衡量已知总数的（**极限量**) 数量中所已经使用数量的工具`entity.usage`。
  例如， system.memory.usage带有标签state = used | cached | free | ...的每种状态下的内存量。
  在适当的情况下 ，所有标签值的使用总和应等于**极限量**。

  测量无限制消耗的资源量与**已使用量**有所区别。

- **使用率** - `instrument``entity.utilization`用于测量**已使用量**的除以**极限量**的使用率。
  例如，`system.memory.utilization`对于正在使用的内存部分的使用率，值在`[0,1]`的范围内。

- **时间** - 一种测量时间流逝的工具 `entity.time`。例如，`system.cpu.time`带有label `state = idle | user | system | `...。
  时间测量不一定是使用时间，并且可以小于或大于两次测量之间的实际使用时间。
  
  **时间**工具是使用指标的一种特殊情况，其中 **极限量**通常可以计算为所有标签值的**时间**总和。
  可以使用`metric`事件时间戳自动导出时间工具的**使用率**。
  例如，`system.cpu.utilization`定义为`system.cpu.time`测量值之差除以经过的时间。

- **io** - 一种用于测量双向数据流的仪器应被调用`entity.io`并带有方向标签。例如，`system.network.io`。

- 不符合以上描述的其他工具可以更自由地命名。例如system.paging.faults和system.network.packets。
  无需在名称中指定单位，因为它们是在创建`instrument`时包含的，但是如果有歧义，可以添加单位。

### instrument 单位

单位应遵循
[UCUM（计量单位统一规范）](http://unitsofmeasure.org/ucum.html) 
(更多说明位于文档
[#705(暂未翻译)](https://github.com/open-telemetry/opentelemetry-specification/issues/705)).

- 对于`instruments`的**使用率**测量，应当使用默认的`1`（`the unity`）。
- 测量某物的整数的`instruments`应使用默认单位1（`the unity`）和 带花括号的注释来赋予其他含义。
  例如{packets}，{errors}，{faults}等。
