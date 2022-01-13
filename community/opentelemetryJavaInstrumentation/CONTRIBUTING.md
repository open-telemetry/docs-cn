## Contributing

我们很欢迎修复 bug 的 pr，但是如果你的 pr 涉及到了新的功能或者改变了当前的功能，那么在提交之前请先
[创建一个 issue](https://github.com/open-telemetry/opentelemetry-java-instrumentation/issues/new) 并将你的想法或者你想要进行的改动
与我们讨论一番。当一系列的共识达成时，就可以提交一个 pr 来进行 review 。

为了去对项目进行构建和测试你需要 11 版本以上的 JDK 。一些 instrumentations 和测试用例可能会对 java 版本加以严格的要求。查看
[执行测试用例](running-tests.md) 来查看这部分的更多细节。

### 构建

#### Snapshot 版本构建

对于开发者而言，在一个 release 版本发布之前发生了代码改动就需要进行验证，那么就需要在 `main` 分之支上进行 snapshot 构建。它们可以从
Sonatype OSS 快照存储库中获得，其地址位于 https://oss.sonatype.org/content/repositories/snapshots/
([浏览器查看](https://oss.sonatype.org/content/repositories/snapshots/io/opentelemetry/)) 。

#### 通过源码进行构建

使用 Java 11 版本进行构建

```bash
java -version
```

```bash
./gradlew assemble
```

并且你将会发现一个 java agent 成果物位于

`javaagent/build/lib/opentelemetry-javaagent-<version>-all.jar`.

### IntelliJ 配置

查看 [IntelliJ 配置](intellij-setup.md)

### 代码风格

查看 [代码风格](style-guideline.md)

### 执行测试用例

查看 [执行测试用例](running-tests.md)

### 开发 instrumentation

查看 [开发 instrumentation](writing-instrumentation.md)

### 了解 javaagent 的组件

查看 [了解 javaagent 的组件](javaagent-jar-components.md)

### 了解 javaagent instrumentation 的测试组件

查看 [了解 javaagent instrumentation 的测试组件](javaagent-test-infra.md)

### 调试 Debug 指引

查看 [Debug 指引](debugging.md)

### 了解 Muzzle

查看 [了解 Muzzle](muzzle.md)
