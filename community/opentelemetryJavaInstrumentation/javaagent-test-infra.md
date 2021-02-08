### 了解 javaagent instrumentation 的测试组件

Javaagent instrumentation 的测试在运行的时候是依赖一个经过 shaded 的 `-javaagent` 来确保模拟当 agent 运行在一个普通应用的时候的相同字节码增强
场景。

下面介绍让上述场景实现的关键组件。

### gradle/instrumentation.gradle

- 对进行 instrumentation 的类进行 shaded
- 根据测试配置添加相应的 jvm 启动参数
  - -javaagent:[agent for testing]
  - -Dotel.initializer.jar=[shaded instrumentation jar]

`otel.initializer.jar` 属性是用来加载经过 shaded 的 instrumentation jar 包到 `AgentClassLoader`， 这样的话 javaagent 的 jar 包不需要在
每次测试执行的时候重新构建。

### :testing:agent-exporter

这里包含了所使用的 span 与 metric 的 exporter。

这些都是内存中的 exporter，这样的话可以通过这测试用例来验证 span 和 metric 是否如预料中那样被 export。

这些 exporter 与内存中的数据都存在于 `AgentClassLoader`当中，所以测试用例必须通过反射来获取这些数据进行验证。简而言之，就是在内存中的数据都通过
OLTP 协议的对象来实现，这样就可以在 `AgentClassLoader` 中被序列化成字节数据并传递到测试用例中的类加载器中被反序列化并验证。`:testing-common`
模块（将在下面被介绍）为测试用例开发者封装了上述复杂的逻辑。

### :agent-for-testing

这是一个包含 `:testing:agent-exporter` 的定制测试 javaagent 的成果物模块。

### :testing-common

这个模块提供了方法去验证 instrumentation 中所产生的 span 和 metric 的数据，并封装了从 `AgentClassLoader` 中获取内存 exporter 数据的复杂逻辑。

