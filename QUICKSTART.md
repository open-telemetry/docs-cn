# OpenTelemetry 快速入门


- [追踪（Tracing）](#追踪（Tracing）)
  * [创建基础Span](#创建基础Span)
  * [创建嵌套Span](#创建嵌套Span)
  * [Span属性](#Span属性)
  * [创建带事件的Span](#创建带事件的Span)
  * [创建带链接Span](#[创建带链接Span)
  * [上下文传播](#上下文传播)
- [指标（Metrics）](#指标（Metrics）)
- [Tracing SDK配置](#Tracing-SDK配置)
  * [采样器（Sampler）](#采样器（Sampler）)
  * [Span处理器](#span-processor)
  * [导出器（Exporter）](#导出器（Exporter）)

OpenTelemetry可用于测量采集遥测数据的代码。有关更多详细信息，请访问[OpenTelemetry Website]

希望使用OpenTelemetry导出遥测数据的**库**必须依赖`opentelemetry-api`包，并且绝不可以配置或依赖 OpenTelemetry SDK。
SDK配置必须由“应用程序”提供，而该应用程序取决于`opentelemetry-sdk`软件包，或任何其他OpenTelemetry API的实现。这样，
库只有用户应用程序做相应配置的情况下才会获得真正的实现。有关更多详细信息，请查阅[Library Guidelines]。

## 追踪（Tracing）

我们介绍如何使用OpenTelemetry API追踪代码。注意：永远不要调用OpenTelemetry SDK的方法。

首先必须获取一个Tracer，该Tracer负责创建spans和并与[上下文](#上下文传播)交互。Tracer是通过使用OpenTelemetry API来获取的，该API指定了检测要监控的检查库或应用程序的库的名称和版本。可在规范章节 [Obtaining a Tracer]中获得更多信息。

```java
Tracer tracer =
    OpenTelemetry.getTracer("instrumentation-library-name","semver:1.0.0");
```

### 创建基础Span
要创建基础Span，只需指定Span的名称。 Span的开始和结束时间由OpenTelemetry SDK自动设置。
```java
Span span = tracer.spanBuilder("my span").startSpan();
try (Scope scope = tracer.withSpan(span)) {
	// your use case
	...
} catch (Throwable t) {
    Status status = Status.UNKNOWN.withDescription("Change it to your error message");
    span.setStatus(status);
} finally {
    span.end(); // closing the scope does not end the span, this has to be done manually
}
```

### 创建嵌套Span

很多时候我们希望为嵌套操作关联span。OpenTelemetry支持在进程内和跨远程进程进行追踪。更多关于如何在远程进程间共享上下文的详细信息，请查看[上下文传播](#上下文传播)。

对于方法a调用方法b，可以通过以下方式手动链接span：
```java
void a() {
  Span parentSpan = tracer.spanBuilder("a")
        .startSpan();
  b(parentSpan);
  parentSpan.end();
}
void b(Span parentSpan) {
  Span childSpan = tracer.spanBuilder("b")
        .setParent(parentSpan)
        .startSpan();
  // do stuff
  childSpan.end();
}
```

OpenTelemetry API还提供了一种自动方法来传播parentSpan：

```java
void a() {
  Span parentSpan = tracer.spanBuilder("a").startSpan();
  try(Scope scope = tracer.withSpan(parentSpan)){
    b();
  } finally {
    parentSpan.end();
  }
}
void b() {
  Span childSpan = tracer.spanBuilder("b")
     // NOTE: setParent(parentSpan) is not required anymore, 
     // `tracer.getCurrentSpan()` is automatically added as parent
    .startSpan();
  // do stuff
  childSpan.end();
}
``` 

链接来自远程进程的span，只需将[Remote Context](#上下文传播)设置为父级即可。

```java
Span childRemoteParent = tracer.spanBuilder("Child").setParent(remoteContext).startSpan();
```

### Span属性

在OpenTelemetry中，可以自由创建Span，由实现者使用特定于所表示操作的属性对其进行注释。属性在Span上提供有关它追踪的特定操作的附加上下文，比如结果或操作属性。

```java
Span span = tracer.spanBuilder("/resource/path").setSpanKind(Span.Kind.CLIENT).startSpan();
span.setAttribute("http.method", "GET");
span.setAttribute("http.url", url.toString());
```

其中一些操作表示使用众所周知的协议（如HTTP或数据库调用）的调用。
对于这些，OpenTelemetry需要设置特定的属性。完整的属性列表在跨语言规范的[Semantic Conventions]中提供。

### 创建带事件的Span

Span 可以携带零个或多个[Span属性](#Span属性)的命名事件进行注释，每一个事件都是一个key:value键值对并自动携带相应的时间戳。
```java
span.addEvent("Init");
...
span.addEvent("End");
```

```java
Attributes eventAttributes = Attributes.of(
    "key", AttributeValue.stringAttributeValue("value"),
    "result", AttributeValue.longAttributeValue(0L));

span.addEvent("End Computation", eventAttributes);
```

### 创建带链接Span

一个Span可以连接一个或多个因果相关的其他Span。链接可用于表示批处理操作，其中一个Span的初始化由多个Span初始化构成，其中每个Span表示批处理中处理的单个输入项。

```java
Link link1 = SpanData.Link.create(parentSpan1.getContext());
Link link2 = SpanData.Link.create(parentSpan2.getContext());
Span child = tracer.spanBuilder("childWithLink")
        .addLink(link1)
        .addLink(link2)
        .addLink(parentSpan3.getContext())
        .addLink(remoteContext)
    .startSpan();
```

有关如何从远程进程中读取上下文的更多详细信息，请参见[上下文传播](#上下文传播)。

### 上下文传播

进程内传播依靠 [gRPC Context](https://grpc.github.io/grpc-java/javadoc/io/grpc/Context.html)，
一个完善的上下文传播库，包含在一个小构件中，它不依赖于整个gRPC引擎。

OpenTelemetry提供了一种基于文本的方法，可以使用 [W3C Trace Context](https://www.w3.org/TR/trace-context/) HTTP标头。
以下是使用`HttpURLConnection`发出的HTTP请求的示例。
```java
// Tell OpenTelemetry to inject the context in the HTTP headers
HttpTextFormat.Setter<HttpURLConnection> setter =
  new HttpTextFormat.Setter<HttpURLConnection>() {
    @Override
    public void put(HttpURLConnection carrier, String key, String value) {
        // Insert the context as Header
        carrier.setRequestProperty(key, value);
    }
};

URL url = new URL("http://127.0.0.1:8080/resource");
Span outGoing = tracer.spanBuilder("/resource").setSpanKind(Span.Kind.CLIENT).startSpan();
try (Scope scope = tracer.withSpan(outGoing)) {
  // Semantic Convention.
  // (Observe that to set these, Span does not *need* to be the current instance.)
  outGoing.setAttribute("http.method", "GET");
  outGoing.setAttribute("http.url", url.toString());
  HttpURLConnection transportLayer = (HttpURLConnection) url.openConnection();
  // Inject the request with the *current*  Context, which contains our current Span.
  OpenTelemetry.getPropagators().getHttpTextFormat().inject(Context.current(), transportLayer, setter);
  // Make outgoing call
} finally {
  outGoing.end();
}
...
```

类似的基于文本的方法可用于从传入请求中读取W3C追踪上下文。
下面提供了使用以下命令处理传入HTTP请求的示例 [HttpExchange](https://docs.oracle.com/javase/8/docs/jre/api/net/httpserver/spec/com/sun/net/httpserver/HttpExchange.html)。

```java
HttpTextFormat.Getter<HttpExchange> getter =
  new HttpTextFormat.Getter<HttpExchange>() {
    @Override
    public String get(HttpExchange carrier, String key) {
      if (carrier.getRequestHeaders().containsKey(key)) {
        return carrier.getRequestHeaders().get(key).get(0);
      }
      return null;
    }
};
...
public void handle(HttpExchange httpExchange) {
  // Extract the SpanContext and other elements from the request.
  Context extractedContext = OpenTelemetry.getPropagators().getHttpTextFormat()
        .extract(Context.current(), httpExchange, getter);
  Span serverSpan = null;
  try (Scope scope = ContextUtils.withScopedContext(extractedContext)) {
    // Automatically use the extracted SpanContext as parent.
    serverSpan = tracer.spanBuilder("/resource").setSpanKind(Span.Kind.SERVER)
        .startSpan();
    // Add the attributes defined in the Semantic Conventions
    serverSpan.setAttribute("http.method", "GET");
    serverSpan.setAttribute("http.scheme", "http");
    serverSpan.setAttribute("http.host", "localhost:8080");
    serverSpan.setAttribute("http.target", "/resource");
    // Serve the request
    ...
  } finally {
    if (serverSpan != null) {
      serverSpan.end();
    }
  }
}
```

## 指标（Metrics）

Span是获取应用程序正在执行的操作的详细信息的好方法，但是，对于更多聚合的场景哪？OpenTelemetry为指标提供支持，
一个时序数据可能表示诸如CPU利用率，HTTP服务器的请求计数或业务指标，例如交易。

所有指标都可以用标签进行注释：附加的限定有助于描述度量指标的细分。

以下是计数器用法的示例：
```java
// Gets or creates a named meter instance
Meter meter = OpenTelemetry.getMeter("instrumentation-library-name","semver:1.0.0");

// Build counter e.g. LongCounter 
LongCounter counter = meter
        .longCounterBuilder("processed_jobs")
        .setDescription("Processed jobs")
        .setUnit("1")
        .build();

// It is recommended that the API user keep a reference to a Bound Counter for the entire time or 
// call unbind when no-longer needed.
BoundLongCounter someWorkCounter = counter.bind("Key", "SomeWork");

// Record data
someWorkCounter.add(123);

// Alternatively, the user can use the unbounded counter and explicitly
// specify the labels set at call-time:
counter.add(123, "Key", "SomeWork");
```


`Observer`是支持异步API并按需收集度量数据的附加工具，按间隔收集数据。

以下是观察者用法的示例：
```java
// Build observer e.g. LongObserver
LongObserver observer = meter
        .observerLongBuilder("cpu_usage")
        .setDescription("CPU Usage")
        .setUnit("ms")
        .build();

observer.setCallback(
        new LongObserver.Callback<LongObserver.ResultLongObserver>() {
          @Override
          public void update(ResultLongObserver result) {
            // long getCpuUsage()
            result.observe(getCpuUsage(), "Key", "SomeWork");
          }
        });
```

## Tracing SDK配置

本文档中的配置示例仅适用于`opentelemetry-sdk`提供的SDK。API的其他实现可能提供不同的配置机制。

该应用程序必须安装带有输出器的span处理器，并且可以自定义OpenTelemetry SDK的行为。

比如一个基本配置实例化了SDK追踪器注册表，并设置为将追踪数据导出到日志记录流。

```java
// Get the tracer
TracerSdkProvider tracerProvider = OpenTelemetrySdk.getTracerProvider();

// Set to export the traces to a logging stream
tracerProvider.addSpanProcessor(
    SimpleSpanProcessor.newBuilder(
        new LoggingSpanExporter()
    ).build());
```

### 采样器（Sampler）
追踪和导出应用程序中的每个用户请求并不总是可行的。
为了在可观察性和占用资源之间取得平衡，可以对追踪数据进行采样。
OpenTelemetry SDK提供了三个开箱即用的采样器：
 - [AlwaysOnSampler]不管上游采样决定如何，都对每个追踪进行采样。
 - [AlwaysOffSampler]不会对任何追踪进行采样，无论上游采样决定如何。
 - [Probability] 对上游的任何采样的追踪数据，根据配置的百分比进行采样。

可以通过实现io.opentelemetry.sdk.trace.Sampler接口来提供其他采样器。

```java
TraceConfig alwaysOn = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.alwaysOn()
).build();
TraceConfig alwaysOff = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.alwaysOff()
).build();
TraceConfig half = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.probability(0.5)
).build();
// Configure the sampler to use
tracerProvider.updateActiveTraceConfig(
    half
);
```

### Span处理器

OpenTelemetry提供了不同的Span处理器。`SimpleSpanProcessor`立即将结束的Span转发到导出器，而`BatchSpanProcessor`对它们进行批处理并批量发送它们。`MultiSpanProcessor`将多个Span处理器配置为同时处于活动状态。

```java
tracerProvider.addSpanProcessor(
    SimpleSpanProcessor.newBuilder(new LoggingSpanExporter()).build()
);
tracerProvider.addSpanProcessor(
    BatchSpanProcessor.newBuilder(new LoggingSpanExporter()).build()
);
tracerProvider.addSpanProcessor(MultiSpanProcessor.create(Arrays.asList(
            SimpleSpanProcessor.newBuilder(new LoggingSpanExporter()).build(),
            BatchSpanProcessor.newBuilder(new LoggingSpanExporter()).build()
)));
```

### 导出器（Exporter）

Span处理器由导出器初始化，该导出器负责将遥测数据发送到特定的后端。OpenTelemetry提供了四种开箱即用的导出器：

- In-Memory: 数据保存在内存中，在debug时使用。
- Jaeger Exporter: 准备收集的遥测数据并将其通过gRPC发送到Jaeger后端。
- Zipkin Exporter: 准备收集的遥测数据，并通过Zipkin API将其发送到Zipkin后端。
- Logging Exporter: 将遥测数据保存到日志流中。
- OpenTelemetry Exporter: 将数据发送到[OpenTelemetry Collector] (还未实现)。

其他导出器可以在[OpenTelemetry Registry]中找到。

```java
tracerProvider.addSpanProcessor(
    SimpleSpanProcessor.newBuilder(InMemorySpanExporter.create()).build());
tracerProvider.addSpanProcessor(
    SimpleSpanProcessor.newBuilder(new LoggingSpanExporter()).build());

ManagedChannel jaegerChannel =
    ManagedChannelBuilder.forAddress([ip:String], [port:int]).usePlaintext().build();
JaegerGrpcSpanExporter jaegerExporter = JaegerGrpcSpanExporter.newBuilder()
    .setServiceName("example").setChannel(jaegerChannel).setDeadline(30000)
    .build();
tracerProvider.addSpanProcessor(BatchSpanProcessor.newBuilder(
    jaegerExporter
).build());
```
> 英文原文：https://github.com/open-telemetry/opentelemetry-java/blob/master/QUICKSTART.md

[AlwaysOnSampler]: https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L82--L105
[AlwaysOffSampler]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L108--L131
[Probability]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L142--L203
[Library Guidelines]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md
[OpenTelemetry Collector]: https://github.com/open-telemetry/opentelemetry-collector
[OpenTelemetry Registry]: https://opentelemetry.io/registry/?s=exporter
[OpenTelemetry Website]: https://opentelemetry.io/
[Obtaining a Tracer]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#obtaining-a-tracer
[Semantic Conventions]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/data-semantic-conventions.md
