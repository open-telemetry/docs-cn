## 代码风格

我们遵循 [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html).

### 格式自动修改

如果源码没有按照谷歌的 java 风格配置将会在构建失败。

这里主要的目标是为了防止在不同的 IDE 之间对于代码格式的不同选择而导致的代码格式化。

运行

```bash
./gradlew spotlessApply
```

将会对所有需要重新调整代码格式的文件进行格式调整。

运行

```bash
./gradlew spotlessCheck
```

将会对代码合适进行调整。

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

在构建中将会通过 checkstyle 将会对一些 Google Java Style Guide 自动格式化而无法处理的场景。

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

