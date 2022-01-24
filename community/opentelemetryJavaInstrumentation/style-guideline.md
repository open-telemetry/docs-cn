## 代码风格

我们遵循 [Google Java 代码风格](https://google.github.io/styleguide/javaguide.html).

### 自动格式化

如果源码没有按照谷歌的 java 代码风格进行格式化，将会在构建失败。

这里主要的目标是为了防止在不同的 IDE 之间对于代码格式的不同选择而导致大量的代码格式化。

运行

```bash
./gradlew spotlessApply
```

重新格式化所有需要重新格式化的文件。

运行

```bash
./gradlew spotlessCheck
```

仅对需要验证的task进行格式化。

#### Pre-commit hook

如果希望将代码格式完全交给机器，那么可以在提交前配置一个 pre-commit hook 来对代码格式进行验证。  
可以通过下面的命令进行激活：

```bash
git config core.hooksPath .githooks
```

#### Editorconfig

对于 IntelliJ 用户，我们提供了 `.editorconfig` 文件来为代码格式配置提供了额外的便利。当然这个文件并不会支持所有的格式要求，所以还需要时常运行
`spotlessApply` 指令。

### 额外检查

在构建中将会通过 checkstyle 将会对一些 Google Java 代码风格自动格式化而无法处理的场景。

在本地执行这些检查：

```
./gradlew checkstyleMain checkstyleTest
```

### 静态导入

我们常常利用静态导入一些常见的操作类型，但是并不是所有的静态方法或对象都适合被静态导入。下面粗略的展示了我们推荐的静态导入场景：

- Test assertions (JUnit and AssertJ)
- 测试中的 Mocking/stubbing (with Mockito)
- 容器类的辅助 Collections helpers (比如 `singletonList()` 和 `Collectors.toList()`)
- ByteBuddy `ElementMatchers` (用来构建 instrumentation modules)
- Immutable constants (被明确声明的)
- 单例 (尤其是被明确声明是不可变的对象)
- `tracer()` 等会暴露单例 tracer 对象的方法
  
  

## 类内容排序

以下顺序是首选：

- 静态字段（放在非静态字段之前）
- 实例字段（放在非静态实例字段之前）
- 构造函数
- 方法
- 嵌套类

如果方法相互调用，最好是将调用的方法放在调用它的方法上面。举个例子，私有方法放在使用它的非私有方法下面。

在静态实用程序类（所有成员都是静态的）中，私有构造函数（用于防止构造）应该在方法之后而不是在方法之前。

## `final` 关键字用法

public类尽可能声明成`final`。

只能在非final修饰的public类中使用final申明方法。

字段尽可能的什么成`final`。

方法参数永远都不要申明成`final`。

局部变量只有在`final`没有内联初始化的情况下才应该被声明（声明这些变量`final`可以帮助防止意外的双重初始化）。

## `@Nullable` 注释用法

[注意：本节是理想化的，并不能反应当前的代码库]

所有可以为空的参数和字段都可以使用`@Nullable`注释（特别是javax.annotation.Nullable，它作为compileOnly依赖项包含在otel.java-gradle插件中）。



那些有默认值的参数或者字段没有必要使用`@NonNull`，应该在`package-info.java`每个模块的根包上的文件中声明 ，例如

```java
@DefaultQualifier(
    value = NonNull.class,
    locations = {TypeUseLocation.FIELD, TypeUseLocation.PARAMETER, TypeUseLocation.RETURN})
package io.opentelemetry.instrumentation.api;

import org.checkerframework.checker.nullness.qual.NonNull;
import org.checkerframework.framework.qual.DefaultQualifier;
```

公共API需要对空值进行防御性检测，即使参数未使用`@Nullable`. 内部 API 不需要防御性地检查`null`参数。

为了帮助强制`@Nullable`使用注解，`otel.nullaway-conventions`应该在所有模块中使用 gradle 插件来执行基本的可为空使用验证：

```kotlin
plugins {
  id("otel.nullaway-conventions")
}
```


