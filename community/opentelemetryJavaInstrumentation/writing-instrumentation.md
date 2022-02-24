# 开发 instrumentation 集成库

**Warning**: OpenTelemetry Java Instrumentation 仓库仍就处于集成这里所描述的能力的过程中。

每当我们希望在 OpenTelemetry 中添加新的 Java 库的支持，比如说添加链路的能力，我们都需要为这个库开发一个新的集成库。
先让我们开始复习一些术语。

**instrumentation 库**: 这里的主要逻辑是创建 span 并通过数据丰富特定库的监控 API。打个比方，当在集成一个
RPC 库的时候， instrumentation 库将会通过一些特定该库的方法来监听请求的开始与结束，并在相应的两个地方通过代码来创建与结束相应
的 span 。许多库都会提供拦截类型的 API 比如 gRPC 所提供的 `ClientInterceptor` 和 servlet 的 `Filter`。
还有一些库将会提供一个 Java 接口与请求相对应，并且 instrumentation 库可以通过实现该接口来实现对方法进行包装，对 span 进行逻辑管理。
用户常常会添加 instrumentation 库所提供的初始化类到他们自己的应用中，这样 instrumentation 库将会在用户的应用中开始其生效。

有些库可能只会暴露静态 API 出去并没有任何拦截的扩展点导致无法对与其请求进行拦截。这样的库将无法创建创建 instrumentation 库。

**Java agent instrumentation**: 此处的逻辑与 instrumentation 库类似，但是区别于用户自己将相关类自行初始化，Java agent instrumentation
将会自动初始化这些类通过字节码增强。这允许用户不需要在考虑与 OpenTelemetry 集成的前提下来开发自己的应用，并获取其集成在不付出任何代价的前提下。
agent instrumentation 生成的字节码常常会和用户在自己应用里所开发的代码或多或少会有相同。

除了自动化还在 instrumentation 库之外， agent instrumentation 可以在 instrumentation 库无法生效的场景下进行增强，比如`URLConnection`，
agent instrumentation 甚至可以拦截 JDK 的类。这样的库将无法通过 instrumentation 库的形式来进行增强，但是 agent instrumentation 可以做到
这一点。

## 目录架构

请借鉴一些当前已经存在的集成库例子来作为目录架构的例子，比如[aws-sdk-2.2](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/aws-sdk/aws-sdk-2.2) 。

当开发新的集成库的时候，先在 `instrumentation` 目录下建立一个新的子目录，其名称对应你所要开发的被集成的库名与支持其的最低的版本号。理想情况
下，基于老版本的库开发的集成库将会支持后续一个大范围的版本，但是这里受到库所提供的 API 的严格制约。

在这个子目录下，创建 `library` （当 instrumentation 库无法实现的时候跳过这个步骤），`javaagent` 和 `testing` 目录。

打个比方，当我们要为一个版本为 `1.0` 名为 `yarpc` 的 RPC 框架开发集成库的时候，我们将会创建的目录结构的树形结构如下所示

```
instrumentation ->
    ...
    yarpc-1.0 ->
        javaagent
            yarpc-1.0-javaagent.gradle
        library
            yarpc-1.0-library.gradle
        testing
            yarpc-1.0-testing.gradle
```

并在最顶层的 `settings.gradle` 进行如下配置

```groovy
include 'instrumentation:yarpc-1.0:javaagent'
include 'instrumentation:yarpc-1.0:library'
include 'instrumentation:yarpc-1.0:testing'
```

## 开发 instrumentation 库

以在 `library` 开发 instrumentation 库开始。这里通常需要使用我们在 `instrumentation-common` 库中所定义的 `Tracer` 来创建并
注释 span 作为库中拦截器实现的一部分。这个 module 通常只依赖 OpenTelemetry API，`instrumentation-common` 和被集成的第三方库本身。
[instrumentation-library.gradle](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/gradle/instrumentation-library.gradle)
需要在 library 中被配置到 build 工具中。

## 开发 instrumentation 的单元测试

当我们开发的 instrumentation 已经被完成，我们需要添加单元测试到 `testing` module 中。测试用例将会对 library 和 agent 都生效，两者的区别主要在于
客户端或者服务端是的初始化方式。在 library 的测试中，将会有直接调用集成库的 API 的代码，但是在 agent 的测试中，将不需要显示地声明使用 library 的 API。
所创建的测试用例通常是一个 abstract 类和一个 abstract 方法来返回一个被集成的对象比如一个客户端。这个类需要集成自 `InstrumentationSpecification` 来
被 Spock 来识别并引入一些辅助方法来方便进行校验。

在开发了几个单元测试之后，需要回到 `library` 包下，我们需要保证其拥有关于 `testing` 子 module 的 test 引用，并添加一个测试类继承自测试 module 中的 abstract 类。
你需要实现方法类来初始化客户端，并根据 library 的机制来添加拦截器， 也许是一个名为 `registerInterceptor` 的方法或者对 library 的工厂结果进行包装。这里的测试需要
实现 `InstrumentationTestRunner` 的 trait 来实现一下通用的启动测试逻辑。如果测试能够通过，说明 instrumentation 库也能够生效。

## 开发 Java agent instrumentation

既然我们已经在进行集成库的开发，那么我们也需要开发 agent instrumentation，这样用户不需要修改他们的应用就可以使用。需要确认 `javaagent` 引用了 `library` 并
对 `testing` 持有 test 引用。Agent instrumentation 定义了一系列针对类的匹配规则去进行字节码增强。你将会常常对 library 的测试中使用的类进行增强，比如
说客户端的构造器。或者是创建构造器的方法，比如说构造器的构造方法。Agent instrumentation 可以在构造方法返回的时候进行字节码注入，比如说进行 `registerInterceptor`
方法的调用并初始化集成部分。你在字节码增强部分所开发的代码常常会与你在之前写的测试代码类似，agent 承担了 instrumentation 库初始化的职责，因此用户不需要再去
做类似的操作。     
你可以在[这里](writing-instrumentation-module.md)看到一个详细的 javaagent instrumentation 指引。       

在完成上述的开发之后，需要为 agent instrumentation 添加单元测试。我们需要保证在用户不知情的前提下能够集成成功。添加一个测试类集成自刚才你在 test 部分开发的
基本测试类，但与之前不同的是，这里客户端的初始化需要不使用我们项目中的任何一个 API，包括 library 部分所提供的。并为你的测试类实现 `AgentTestRunner` trait 来保证
通用的测试启动逻辑，并尝试运行。所有的测试都应该通过。

值得一提的是所有在 `javaagent` module 下的单元测试将会在 `-javaagent` 命令下运行来模拟当 agent 在一个普通应用上运行时，会发生的相同的字节码修改。
这代表 javaagent instrumentation 将会在 `AgentClassLoader` 中并无法直接访问你的测试代码，下一部分你将会开发单元测试来直接访问 javaagent instrumentation。

## 开发 Java agent 单元测试

如上文所说， `javaagent` 中的测试并不能直接访问 javaagent 的集成类。

理想情况下，javaagent 的集成部分只是一个对于 instrumentation 库的简单封装，所以并不需要专门开发针对 javaagent instrumentation 中类的逻辑。

但是如果你还是希望针对 javaagent instrumentation 开发单元测试，添加另一个叫做 `javaagent-unit-tests` 的 module。例子如下：

```
instrumentation ->
    ...
    yarpc-1.0 ->
        javaagent
            yarpc-1.0-javaagent.gradle
        javaagent-unit-test
            yarpc-1.0-javaagent-unit-test.gradle
        ...
```

## 一些需要格外注意的地方

### 如果被集成的对象是无法访问的 maven 依赖

如果被集成的服务端或库的 jar 包无法从 maven 仓库进行访问，你可以创建一个 module，其中的 stub 类只声明几个你需要进行集成的方法。在 stub 类中的方法可以只
抛出 `throw new UnsupportedOperationException()`， 这些类将会只在 advice 类的编译过程中被使用，并不会直接被打包到 agent 中去。在运行过程
中，被集成的服务端或库的真实类将会被使用到。

创建一个 module 名为 `compile-stub` 并在 `compile-stub.gradle` 文件中添加下列内容

```
apply from: "$rootDir/gradle/java.gradle"
```

在 javaagent module 中添加只编译的依赖如下

```
compileOnly project(':instrumentation:xxx:compile-stub')
```
