### 调试

调试 java agent 中的代码将是一个不小的挑战，因为 instrumentation 部分的代码将会直接内联到目标类中。

#### Advice 方法

Advice 方法中的断点将不会生效，因为里面的代码都已经由 ByteBuddy 内联在了目标类中。需要尽可能去保证这部分的代码足够的轻量。  
Advice 方法需要被添加以下注解：

```java
@net.bytebuddy.asm.Advice.OnMethodEnter
@net.bytebuddy.asm.Advice.OnMethodExit
```

调试 agent 的 advice 方法与 agent 的初始化逻辑最好的方式是使用以下的方法：

```java
System.out.println()
Thread.dumpStack()
```

#### Agent 初始化代码

如果你想调试 agent 的初始化代码（比如 `OpenTelemetryAgent`，`AgentInitializer`，`AgentInstaller`, `OpenTelemetryInstaller`等等），那么你必须在 JVM 参数
`-javaagent:` 之前声明 `-agentlib:` 参数并使用 `suspend=y` （下面有完成的例子）。

#### 开启调试

下面是进行远程调试的配置。断点应该在 advice 方法之外的任意位置。

```bash
java -agentlib:jdwp="transport=dt_socket,server=y,suspend=y,address=5000" -javaagent:opentelemetry-javaagent-<version>.jar -jar app.jar
```
