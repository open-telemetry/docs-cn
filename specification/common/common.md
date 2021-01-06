<!--Common specification concepts-->

# 通用规范概念

`状态`: [功能冻结](../document-status.md)。


<details>
<summary>
目录
</summary>

- [属性](#属性)

</details>

<!--## Attributes-->

## 属性

<!--Attributes are a list of zero or more key-value pairs. An `Attribute` MUST have the following properties:-->

属性是一个由零或多个键值对组成的列表。一个`属性`必须具有以下属性。

<!--The attribute key, which MUST be a non-`null` and non-empty string.-->

- 属性的键，必须是一个非 `null` 且非空字符串。

  <!--The attribute value, which is either:-->

- 属性的值，可以是：

  <!--A primitive type: string, boolean, double precision floating point (IEEE 754-1985) or signed 64 bit integer.-->

  - 一个原始类型：字符串，布尔类型，双精度浮点数（IEEE 754-1985）或有符号的 64 位整数。

  <!--An array of primitive type values. The array MUST be homogeneous, i.e. it MUST NOT contain values of different types. For protocols that do not natively support array values such values SHOULD be represented as JSON strings.-->
  
  - 一个原始类型值的数组。数组必须是同类型的，也就是说，它不能包含不同类型的值。对于原生不支持数组值的协议，这些值应该用 JSON 字符串表示。

<!--Attribute values expressing a numerical value of zero, an empty string, or an empty array are considered meaningful and MUST be stored and passed on to processors/exporters.-->

值为零，空字符串或空数组的属性值是有意义的，必须被存储并传递给处理器/导出器。

<!--Attribute values of `null` are not valid and attempting to set a `null` value is undefined behavior.-->

值为 `null` 的属性无效，试图设置 `null` 值是未定义的行为。

<!--`null` values SHOULD NOT be allowed in arrays. However, if it is impossible to make sure that no `null` values are accepted (e.g. in languages that do not have appropriate compile-time type checking), `null` values within arrays MUST be preserved as-is (i.e., passed on to span processors / exporters as `null`). If exporters do not support exporting `null` values, they MAY replace those values by 0, `false`, or empty strings.-->

<!--This is required for map/dictionary structures represented as two arrays with indices that are kept in sync (e.g., two attributes `header_keys` and `header_values`, both containing an array of strings to represent a mapping`header_keys[i] -> header_values[i]`).-->

数组中不应该有 `null` 值。但是，如果无法确保拒绝 `null` 值（例如，没有适当的编译时类型检查的语言中），数组中的 `null` 值必须按原样保留（即作为 `null` 值传递给 span 处理器/导出器）。如果出口商不支持导出 `null `值，他们可以用 0、`false` 或空字符串替换这些值。这对于一对具有同步索引的数组的map/dictionary 结构而言是必须的（例如，两个属性 `header_keys` 和 `header_values`，都包含一个字符串数组来表示映射 `header_keys[i] -> header_values[i]`）。

<!--See [Attribute and Label Naming](attribute-and-label-naming.md) for naming guidelines.-->

命名指南请参见[属性和标签命名](attribute-and-label-naming.md)。

