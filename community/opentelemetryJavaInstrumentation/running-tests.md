### 运行测试用例

#### Java 版本

Open Telemetry 的自动集成最低支持的 Java 版本为 java 8。除非特定声明，我们的 jar 包成果物都是基于 java 8 环境编译。
我们的测试工具套件针对 java 8，所有的 LTS 版本和最新的非 LTS 版本来执行。

一些我们自动集成的库可能对最低的运行版本有更高的要求。在这个情况下我们将在库的要求下以更高的 java 版本来进行编译与测试。
结果的 class 文件将会有更高的字节码级别，但是由于它和库的 java 版本相匹配而不会在运行时导致问题产生。

#### Instrumentation 测试

执行命令 `./gradlew instrumentation:test` 将会为每个集成库在构建时的 java 版本下执行单元测试。这些测试往往采用其对应的库支持的最低 java 版本。

#### 在特定的 java 版本下执行测试

默认情况下我们在 Java 11 下运行测试，同时也支持在 Java 8 和 15 下执行。为了运行后面的 Java 版本，可以通过配置 gradle 的 `testJavaVersion`
参数来指定 java 版本，比如 `./gradlew test -PtestJavaVersion=8` 和 `./gradlew test -PtestJavaVersion=15`。如果你本地没有
安装对应的指定 JDK 版本， Gradle 将会自动为你下载。

#### 在集成的库最新版本下执行测试

这作为一部分在 nightly build 中执行，这是为了保证当一个库的最新的版发布的时候导致我们的测试失败的时候可以及时意识到。

可以在 `gradlew` 命令后加上 `-PtestLatestDeps=true` 来在本地执行这一测试。

#### 执行单个测试

执行 `./gradlew :instrumentation:<INSTRUMENTATION_NAME>:test --tests <GROOVY TEST FILE NAME>` 命令将会只
执行所选择的单个测试。

## 冒烟测试

冒烟测试不作为全局`test`任务运行的一部分运行，因为它们需要很长时间并且与大多数贡献无关。明确指定`:smoke-tests:test`运行它们。

如果您需要运行特定的冒烟测试套件：

```
./gradlew :smoke-tests:test -PsmokeTestSuite=payara
```

如果您使用的是 Windows 并且想要使用 linux 容器运行测试：

```
USE_LINUX_CONTAINERS=1 ./gradlew :smoke-tests:test -PsmokeTestSuite=payara
```


