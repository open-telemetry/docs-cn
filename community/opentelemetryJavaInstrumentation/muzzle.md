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

在编译期间的引用手机和代码生成都是都是通过 ByteBuddy 的插件来完成的（叫做 `MuzzleCodeGenerationPlugin`）。

在每个 instrumentation 模块编译的过程中，ByteBuddy 插件都会通过当前模块的 type instrumentation
（`InstrumentationModule#typeInstrumentations()`）来收集模块所涉及到的内部与第三方库 API 的引用。引用收集的过程从 advice 类开始
（通过 `TypeInstrumentation#transformers()` 方法所返回的 Map 的 value 值）并对 class graph 进行遍历直到遇到一个非 instrumentation 的类
为止（通过 `InstrumentationClassPredicate` 判断）。除引用之外，收集的进程也会构建一个 instrumentation 内部辅助类之间的依赖拓扑 - 这个
依赖拓扑将会在后面被用来构造一个准备注入应用类加载起的辅助类列表（`InstrumentationModule#getMuzzleHelperClassNames()`）。

所有被收集的引用都会用来被构造一个 `ReferenceMatcher` 实例。而每个模块的引用匹配类都可以在方法
`InstrumentationModule#getMuzzleReferenceMatcher()` 中得到并在各个 instrumentation 之间被共享。这个方法的字节码（基本上是一系列
`Reference` 构造器的调用）和 `getMuzzleHelperClassNames()` 方法都将通过 ByteBuddy 的插件靠 ASM 的 code vister 来实现。

这个编译期的插件的源码位于 `javaagent-tooling` 模块的 `io.opentelemetry.javaagent.tooling.muzzle.collector` 包下。

### 运行时的引用匹配

在运行过程中的引用匹配将会通过 `InstrumentationModule` 中的一个 ByteBuddy matcher 来实现。`MuzzleMatcher` 通过在编译阶段生成的
`getMuzzleReferenceMatcher()` 方法来对类加载器过程中被增强的所有符号进行验证（类，方法，字段）。如果 `ReferenceMatcher` 在收集的引用与
应用 classpath 下的类型发现不匹配，那么 instrumentation 的过程都会被终止。

值得一提的是，由于 muzzle 检查的开销是非常大的，所以只有在 `InstrumentationModule#classLoaderMatcher()` 和
`TypeInstrumentation#typeMatcher()` 返回了一个 matcher 之后才会进行。在每个类加载器中都会缓存 muzzle 的匹配结果，所以在每个
instrumentation 模块只会进行一次 muzzle 检查的。

在运行期的 muzzle matcher 的源码位于 `javaagent-tooling` 模块下 `io.opentelemetry.javaagent.tooling.muzzle` 包的
`Instrumenter.Default`类中。

## Muzzle gradle 插件

Muzzle 的 gradle 插件允许在项目的构建中，针对各个第三方库的版本执行运行的引用匹配。

Muzzle 的 gradle 插件只是一个用来在构建时的一个附加检查程序，以便在基础架构发生变动引起没有预料到的第三方库版本发生改动，导致 instrumentation
无法使用的时候及时提出警告。  
**即使没有这个插件运行期的 muzzle 引用匹配 _都会_ 进行**，这个插件并没有可以关闭这个引用匹配的功能。

Gradle 插件定义了两个 task：

- `muzzle` task 将在对各个不同的第三方库之间尽心版本验证：

  ```sh
  ./gradlew :instrumentation:google-http-client-1.19:muzzle
  ```

  如果一个新的并没有被集成的第三方库被使用了将会导致整个构建过程的失败。

- `printMuzzleReferences` task 将会打印所给的模块下的所有 API 引用：
  ```sh
  ./gradlew :instrumentation:google-http-client-1.19:printMuzzleReferences
  ```

Muzzle 插件需要在模块中的 `.gradle` 文件中被配置。  
下面是一个例子：

```groovy
muzzle {
  // it is expected that muzzle fails the runtime check for this component
  fail {
    group = "commons-httpclient"
    module = "commons-httpclient"
    // versions from this range are checked
    versions = "[,4.0)"
    // this version is not checked by muzzle
    skipVersions += '3.1-jenkins-1'
  }
  // it is expected that muzzle passes the runtime check for this component
  pass {
    group = 'org.springframework'
    module = 'spring-webmvc'
    versions = "[3.1.0.RELEASE,]"
    // except these versions
    skipVersions += ['1.2.1', '1.2.2', '1.2.3', '1.2.4']
    skipVersions += '3.2.1.RELEASE'
    // this dependency will be added to the classpath when muzzle check is run
    extraDependency "javax.servlet:javax.servlet-api:3.0.1"
    // verify that all other versions - [,3.1.0.RELEASE) in this case - fail the muzzle runtime check
    assertInverse = true
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

最简单的方式是在 muzzle 的 `pass` 命令中添加 `assertInverse = true` 。这样插件将会自动把被集成的库的所有其他版本列为 `fail` 部分的判断。  
在默认情况下，在 instrumentation 模块中默认采用 `assertInverse = true` 是值得的，尤其是针对库的非常老的版本的时候。 Muzzle 插件将会确保
当我们无法对这些特别老的版本进行有效集成的时候，这些版本不会在运行中被意外增强。一个 `fail` 命令可以强迫 instrumentation 模块的开发者去明确定义
`classLoaderMatcher()` 来保证只有所期望的版本范围会被增强。

在更复杂的场景下，可能需要多个 `pass` 和 `fail` 指令来保证去覆盖尽可能多的版本。

