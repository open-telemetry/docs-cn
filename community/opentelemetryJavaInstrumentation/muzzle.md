# Muzzle



Muzzle 是一个 Java agent 的安全特性，当被增强的应用版本与 instrumentation 中的代码所支持的版本不匹配的时候，它可以防止 instrumentation 被加载。
这确保了应用类加载路径上符号（类，方法，字段）与 instrumentation 中 advice 的方法中所增强的符号之间的 API 兼容性。换句话说，muzzle 确保了 agent 与应用
类路径上的兼容性。

Muzzle 将会在遇到不匹配与冲突的时候防止 instrumentation 的加载。

## 如何生效

Muzzle 存在两个阶段：

- 在编译阶段将会收集所使用到的第三方库的引用与被用到的辅助类
- 在执行阶段比较将上一步收集到的信息与 classpath 上的实际使用的 API 符号进行比较

### 编译阶段的引用收集

编译时引用收集和代码生成的过程是使用Gradle 插件 ( [`io.opentelemetry.instrumentation.muzzle-generation`](https://plugins.gradle.org/plugin/io.opentelemetry.instrumentation.muzzle-generation)) 实现的。

对于每个检测模块，代码生成插件首先将`InstrumentationModuleMuzzle`接口应用于它，然后通过生成所需的字节码继续实现来自该接口的所有方法。它收集引用当前处理模块的类型检测 ( `InstrumentationModule#typeInstrumentations()`)使用的内部和第三方 API 的符号。引用收集过程从通知类（通过调用`TypeInstrumentation#transform(TypeTransformer)`方法收集 ）开始，遍历类图，直到遇到对非检测类的引用（由`InstrumentationClassPredicate` 和`InstrumentationModule#isHelperClass(String)`谓词）。除了引用之外，收集过程还构建了内部检测助手类之间的依赖拓扑图 - 此依赖拓扑图稍后用于构造将注入应用程序类加载器 ( `InstrumentationModuleMuzzle#getMuzzleHelperClassNames()`)的助手类列表。Muzzle 还会自动生成该`InstrumentationModuleMuzzle#registerMuzzleVirtualFields()` 方法。然后使用所有收集的引用来生成`InstrumentationModuleMuzzle#getMuzzleReferences`方法。

如果您的`InstrumentationModule`子类定义了一个与来自 的方法具有完全相同签名的方法`InstrumentationModuleMuzzle`，那么muzzle编译插件将不会覆盖您的代码：muzzle只会生成那些没有自定义实现的方法。

编译时插件的源代码位于`muzzle`模块 package 中`io.opentelemetry.javaagent.tooling.muzzle.generation`。



### 运行时的引用匹配

在运行过程中的引用匹配将会通过 `InstrumentationModule` 中的一个 ByteBuddy matcher 来实现。`MuzzleMatcher` 通过在编译阶段生成的
`InstrumentationModuleMuzzle`方法来对类加载器过程中被增强的所有符号进行验证（类，方法，字段）。如果 matcher 在收集的引用与
应用 classpath 下的类型发现不匹配，那么 instrumentation 的过程都会被终止。

值得一提的是，由于 muzzle 检查的开销是非常大的，所以只有在 `InstrumentationModule#classLoaderMatcher()` 和
`TypeInstrumentation#typeMatcher()` 返回了一个 matcher 之后才会进行。在每个类加载器中都会缓存 muzzle 的匹配结果，所以在每个
instrumentation 模块只会进行一次 muzzle 检查的。

在运行期的 muzzle matcher 的源码位于`muzzle`模块中。

## Muzzle-check gradle 插件

[`muzzle-check`](https://plugins.gradle.org/plugin/io.opentelemetry.instrumentation.muzzle-check) 的 gradle 插件允许在项目的构建中，针对各个第三方库的版本执行运行的引用匹配。

 `muzzle-check`的 gradle 插件只是一个用来在构建时的一个附加检查程序，以便在基础架构发生变动引起没有预料到的第三方库版本发生改动，导致 instrumentation
无法使用的时候及时提出警告。  
**即使没有这个插件运行期的 muzzle 引用匹配 _都会_ 进行**，它不是可选功能。

Gradle 插件定义了两个 task：

- `muzzle` task 在运行时使用muzzle来验证不同的三方类库版本：
  
  ```sh
  ./gradlew :instrumentation:google-http-client-1.19:javaagent:muzzle
  ```
  
  如果发布了新的、不兼容的检测库版本，则构建失败。

- `printMuzzleReferences` task 将会打印所给的模块下的所有 API 引用：
  
  ```sh
  ./gradlew :instrumentation:google-http-client-1.19:javaagent:printMuzzleReferences
  ```

Muzzle 插件需要在模块中的 `.gradle` 文件中被配置。  
下面是一个例子：

```groovy
muzzle {
  // it is expected that muzzle fails the runtime check for this component
  fail {
    group.set("commons-httpclient")
    module.set("commons-httpclient")
    // versions from this range are checked
    versions.set("[,4.0)")
    // this version is not checked by muzzle
    skip("3.1-jenkins-1")
  }
  // it is expected that muzzle passes the runtime check for this component
  pass {
    group.set("org.springframework")
    module.set("spring-webmvc")
    versions.set("[3.1.0.RELEASE,]")
    // except these versions
    skip("1.2.1", "1.2.2", "1.2.3", "1.2.4")
    skip("3.2.1.RELEASE")
    // this dependency will be added to the classpath when muzzle check is run
    extraDependency("javax.servlet:javax.servlet-api:3.0.1")
    // verify that all other versions - [,3.1.0.RELEASE) in this case - fail the muzzle runtime check
    assertInverse.set(true)
  }
}
```

- 使用 `pass` 或者 `fail` 可以直接声明 muzzle 检查过程中对于各个第三方库的校验规则。
- `versions` 是一个版本范围， `[]` 是包含而 `()` 则是排除。在这里可以直接声明确切的起止版本号，其中 `[1.0.0,4)` 和 `[1.0.0,4.0.0-Alpha)`
  可以达到一样的效果。
- `assertInverse` 是一个可以声明 `versions` 范围外的版本库相反指令的简短方式。
- `extraDependency` 允许在编译期对 classpath 上额外的库添加检查。这适用于那些并没有在被集成的库上被绑定，但是在运行时常常依赖的 jar 包。

这个 gradle 插件的源码位于 `buildSrc` 目录下。

### 覆盖所有版本与 `assertInverse`

理想情况下当使用 muzzle gradle 插件的时候，我们的检查应该覆盖到我们支持的第三方库的所有版本。对于一些版本的 muzzle 检查失败是为了确保在运行时不会加载
对应的 instrumentation 的方式，并且不会对被增强的应用产生任何影响。

最简单的方式是在 muzzle 的 `pass` 命令中添加 `assertInverse.set(true)` 。这样插件将会自动把被集成的库的所有其他版本列为 `fail` 部分的判断。  
在默认情况下，在 instrumentation 模块中默认采用 `assertInverse.set(true)` 是值得的，尤其是针对库的非常老的版本的时候。 Muzzle 插件将会确保
当我们无法对这些特别老的版本进行有效集成的时候，这些版本不会在运行中被意外增强。一个 `fail` 命令可以强迫 instrumentation 模块的开发者去明确定义
`classLoaderMatcher()` 来保证只有所期望的版本范围会被增强。

在更复杂的场景下，可能需要多个 `pass` 和 `fail` 指令来保证去覆盖尽可能多的版本。
