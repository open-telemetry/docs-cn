# 使用 `Instrumenter` API

`Instrumenter`封装用于收集遥测，从收集的数据，以起始和结束跨度，使用度量仪器记录的值的整个逻辑。`Instrumenter` 公共API只包含三个方法：`shouldStart()`，`start()`和`end()`。该类旨在装饰检测库代码的实际调用；`shouldStart()` 和`start()`方法在请求处理开始的时候调用。`end()`方法必须在请求处理结束和响应到达或者因错误而失败时调用。该`Instrumenter`是参数化的泛型类`REQUEST`和`RESPONSE`类型。它们代表着操作的输入和输出。`Instrumenter`可以配置各种提取器，可以增强或修改遥测数据。

## 使用 `shouldStart()`来检查当前操作是否生成了任何遥测数据。

第一个方法，需要在任何其他`Instrumenter`方法之前调用`shouldStart()`.它确定是否应该对操作进行遥测检测。该`Instrumenter`框架实现了几个抑制规则，以防止生成重复的遥测数据；例如，相同的 HTTP 服务器请求总是产生单个 HTTP`SERVER`跨度。

该`shouldStart()`方法接受当前的 OpenTelemetry`Context`和检测的库`REQUEST`类型。参考以下示例：

```java
Response decoratedMethod(Request request) {
  Context parentContext = Context.current();
  if (!instrumenter.shouldStart(parentContext, request)) {
    return actualMethod(request);
  }

  // ...
}
```

如果该`shouldStart()`方法返回`false`，则剩余的`Instrumenter`方法不应该被调用。

## 开始使用`start()`启动检测操作

当`shouldStart()`返回时`true`，您可以使用`start()`来启动检测操作。
该`start()`方法开始收集有关正在调用的检测库函数的遥测数据。它启动`Span`并开始记录指标（如果在使用的`Instrumenter`实例中注册了任何指标）。

该`start()`方法接受当前的 OpenTelemetry`Context`和检测的库`REQUEST`类型，并返回新的 OpenTelemetry `Context`，它应该在检测操作结束之前变为当前状态。参考以下示例：

```java
Response decoratedMethod(Request request) {
  // ...

  Context context = instrumenter.start(parentContext, request);
  try (Scope scope = context.makeCurrent()) {
    // ...
  }
}
```

新启动一个`context`作为当前的上下文，并在其内部`scope`调用实际的库方法。

## 使用 `end()`结束检测操作

`end()`当检测操作完成时调用该方法。始终在 之后调用此方法非常重要`start()`。`start()`不稍后调用`end()` 将导致不准确或错误的遥测和上下文泄漏。该`end()`方法结束当前跨度并完成记录指标（如果在`Instrumenter`实例中注册了任何指标）。

该`end()`方法接受几个参数：

- `start()`方法返回一个OpenTelemetry的`Context`。
- `REQUEST`开始处理的实例。
- 可选地，`RESPONSE`结束处理的实例 - 可能是`null`在未收到或发生错误的情况下。
- （可选）使用`Throwable`操作引发的错误。

参考以下示例：

```java
Response decoratedMethod(Request request) {
  Context parentContext = Context.current();
    if (!instrumenter.shouldStart(parentContext, request)) {
    return actualMethod(request);
  }

  Context context = instrumenter.start(parentContext, request);
  try (Scope scope = context.makeCurrent()) {
    Response response = actualMethod(request);
    instrumenter.end(context, request, response, null);
    return response;
  } catch (Throwable error) {
    instrumenter.end(context, request, null, error);
    throw error;
  }
}
```

在代码示例中`start()`方法返回的值`context`被传递给`end()` 方法。`end()`无论装饰的结果如何，无论`actualMethod()`是有效响应还是错误，始终调用该方法。

## 使用`InstrumenterBuilder`创建一个新的 `Instrumenter`

一个`Instrumenter`可以通过调用它的静态获得`builder()`方法和使用返回的`InstrumenterBuilder`配置捕获遥测和应用自定义设置。该`builder()`方法接受三个参数：

- 一个`OpenTelemetry`实例，用于获取`Tracer`和`Meter`对象。
- 检测名称，指示*检测*库名称，而不是 *检测*库名称。此处传递的值应唯一标识检测库，以便在故障排除期间可以确定遥测数据的来源。在[OpenTelemetry 规范中](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#instrumentation-libraries)阅读有关检测库的更多信息。
- `SpanNameExtractor`确定span名称。

一个`Instrumenter`可以由几个较小的组件构建。以下小节描述了可用于自定义`Instrumenter`.

### 使用 `SpanNameExtractor`命名span

`SpanNameExtractor`是一个简单的函数式接口，它接受`REQUEST`类型并返回跨度名称。有关跨度命名的更详细指南，请查看[`Span`规范](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#span) 和跟踪[语义约定](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/README.md)。

参考以下示例：

```java
class MySpanNameExtractor implements SpanNameExtractor<Request> {

  @Override
  public String extract(Request request) {
    return request.getOperationName();
  }
}
```

示例`SpanNameExtractor`实现采用请求类型提供的拟合值并将其用作span名称。请注意，这`SpanNameExtractor`是一个`@FunctionalInterface`: 而不是将它作为一个单独的类来实现，您可以只传递`Request::getOperationName`给该`builder()` 方法。

### 使用`AttributesExtractor`给span和metric数据添加属性

`AttributesExtractor`负责在处理开始和结束时提取span和metric属性。它包含两种方法：

- `onStart()`当检测操作开始时调用该方法。它接受两个参数：一个`AttributesBuilder`实例和传入的`REQUEST`实例。
- `onEnd()`当检测操作结束时调用该方法。它接受与相同的两个参数`onStart()`以及一个可选`RESPONSE`和一个可选的`Throwable`错误。

这两种方法的目的都是从接收到的请求（以及响应或错误）中提取感兴趣的属性并将它们设置到构建器中。一般来说，最好在`onStart()`填充attributes，因为这些attributes可用于`Sampler`.

参考以下示例：

```java
class MyAttributesExtractor implements AttributesExtractor<Request, Response> {

  private static final AttributeKey<String> NAME = stringKey("mylib.name");
  private static final AttributeKey<Long> COUNT = longKey("mylib.count");

  @Override
  public void onStart(AttributesBuilder attributes, Request request) {
    set(attributes, NAME, request.getName());
  }

  @Override
  public void onEnd(
      AttributesBuilder attributes,
      Request request,
      @Nullable Response response,
      @Nullable Throwable error) {
    if (response != null) {
      set(attributes, COUNT, response.getCount());
    }
  }
}
```

上面的`AttributesExtractor`示例实现设置了两个属性：一个从请求中提取，一个从响应中提取。建议将`AttributeKey`实例保留为静态final常量并重用它们。每次设置属性时创建一个新键都有引入不必要开销的风险。

您可以使用或方法`AttributesExtractor`向中添加。`InstrumenterBuilder``addAttributesExtractor()``addAttributesExtractors()`

您也可以使用`addAttributesExtractor()`或者`addAttributesExtractors()`向`InstrumenterBuilder`添加一个`AttributesExtractor`。

### 使用 `SpanStatusExtractor` 设置span状态

默认情况下，当`Throwable`发生错误的时候设置span状态为`StatusCode.ERROR`，其他情况使用`StatusCode.UNSET`。设置自定义`SpanStatusExtractor`允许自定义此行为。

该`SpanStatusExtractor`接口只有一个方法`extract()`接受`REQUEST`，一个可选`RESPONSE`和一个可选的`Throwable`错误。它应该返回一个`StatusCode`将在span包装检测操作结束时设置的。

参考以下示例：

```java
class MySpanStatusExtractor implements SpanStatusExtractor<Request, Response> {

  @Override
  public StatusCode extract(
      Request request,
      @Nullable Response response,
      @Nullable Throwable error) {
    if (response != null) {
      return response.isSuccessful() ? StatusCode.OK : StatusCode.ERROR;
    }
    return SpanStatusExtractor.getDefault().extract(request, response, error);
  }
}
```

上面`SpanStatusExtractor`的示例根据操作结果返回一个自定义`StatusCode`，编码在response类中。如果响应不存在（例如，由于错误），使用`SpanStatusExtractor.getDefault()`方法返回默认的行为。

您可以使用`setSpanStatusExtractor()`方法在`InstrumenterBuilder` 设置`SpanStatusExtractor`。

### 使用 `SpanLinksExtractor`添加span 关联

该`SpanLinksExtractor`接口可用于在检测操作开始时添加到其他span的链接。它有一个`extract()`接收以下参数的方法：

- 一个`SpanLinkBuilder`可以用来添加的链接。
- 通过`Instrumenter.start()`传递父`Context`。
- 通过`Instrumenter.start()`传递`REQUEST`。

您可以[在此处](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#links-between-spans)阅读有关跨度链接及其用例的更多信息。

参考以下示例：

```java
class MySpanLinksExtractor implements SpanLinksExtractor<Request> {

  @Override
  public void extract(SpanLinksBuilder spanLinks, Context parentContext, Request request) {
    for (RelatedOperation op : request.getRelatedOperations()) {
      spanLinks.addLink(op.getSpanContext());
    }
  }
}
```

在`SpanLinksExtractor`示例中，请求对象很方便的使用`getRelatedOperations()`方法查找所有的span并链接到一起。当这样的函数不存在时，您需要从请求头或者元数据中提取需要构造的信息来构造一个`SpanContext`。

您可以使用`addSpanLinksExtractor()`方法在`InstrumenterBuilder` 设置`SpanLinksExtractor`。

### 使用`ErrorCauseExtractor`丢弃包装异常类型

发生错误时，根本原因可能隐藏在几个“包装器”异常类型后面，例如来自 Java 标准库的`ExecutionException`或  `CompletionException`。默认情况下，来自 JDK 的已知包装器异常类型将从捕获的错误中删除。要删除其他包装器异常，例如检测库提供的异常，您可以实现`ErrorCauseExtractor`，它具有以下功能：

- 它只有`extractCause()`一种方法负责剥离不必要的异常层并提取导致操作失败的实际错误。
- 它接受一个`Throwable`并返回一个`Throwable`。

参考以下示例：

```java
class MyErrorCauseExtractor implements ErrorCauseExtractor {

  @Override
  public Throwable extractCause(Throwable error) {
    if (error instanceof MyLibWrapperException && error.getCause() != null) {
      error = error.getCause();
    }
    return ErrorCauseExtractor.jdk().extractCause(error);
  }
}
```

在`ErrorCauseExtractor`示例中，实现检查错误是否是`MyLibWrapperException`实例并获取原因，在这种情况下它会解开它。`error.getCause() != null`检查非常重要：如果提取器没有验证这两个条件，就可能意外地删除整个异常，使检测丢失一个异常错误，从而从根本上改变了捕获的遥测。接下来，提取器返回到`jdk()`删除已知 JDK 包装器异常类型的默认实现。

您可以使用`setErrorCauseExtractor()`方法在`InstrumenterBuilder` 设置`ErrorCauseExtractor`。

### 使用`TimeExtractor`自定义操作开始时间和结束时间

在某些情况下，检测库提供了一种方法来检索操作开始和结束的准确时间戳。`TimeExtractor`接口可用于将此信息提供给 OpenTelemetry  trace和metric数据。

`extractStartTime()`只能从请求中提取时间戳。`extractEndTime()` 接受请求、可选响应和可选`Throwable`错误。

参考以下示例：

```java
class MyTimeExtractor implements TimeExtractor<Request, Response> {

  @Override
  public Instant extractStartTime(Request request) {
    return request.startTimestamp();
  }

  @Override
  public Instant extractEndTime(Request request, @Nullable Response response, @Nullable Throwable error) {
    if (response != null) {
      return response.endTimestamp();
    }
    return request.clock().now();
  }
}
```

上面的示例实现使用请求来检索开始时间戳。如果可用，则响应用于计算结束时间；如果它丢失（例如，发生错误时），则使用相同的时间源来计算当前时间戳。

您可以在`InstrumenterBuilder`使用`setTimeExtractor()` 方法中设置时间提取器。

### 通过实现`RequestMetrics` 和`RequestListener`注册 metrics

如果您需要向`Instrumenter`中添加指标，您可以实现`RequestMetrics` 和`RequestListener`接口。`RequestMetrics`只是一个工厂接口`RequestListener`- 它接收一个 OpenTelemetry`Meter`并返回一个新的监听器。

在`RequestListener`包含两个方法：

- `start()`在检测操作开始时执行。它返回一个`Context`-如果有需要的话，它可以使用内部存储指标状态并传递到`end()`。
- `end()` 在检测操作结束时执行。

这两种方法都接收一个`Context`，它的一个实例`Attributes`包含在检测操作开始或结束时计算的属性，以及可用于准确计算持续时间的开始和结束纳秒时间戳。

参考以下示例：

```java
class MyRequestMetrics implements RequestListener {

  static RequestMetrics get() {
    return MyRequestMetrics::new;
  }

  private final LongUpDownCounter activeRequests;

  MyRequestMetrics(Meter meter) {
    activeRequests = meter
        .upDownCounterBuilder("mylib.active_requests")
        .setUnit("requests")
        .build();
  }

  @Override
  public Context start(Context context, Attributes startAttributes, long startNanos) {
    activeRequests.add(1, startAttributes);
    return context.with(new MyMetricsState(startAttributes));
  }

  @Override
  public void end(Context context, Attributes endAttributes, long endNanos) {
    MyMetricsState state = MyMetricsState.get(context);
    activeRequests.add(1, state.startAttributes());
  }
}
```

上面实例列出了通过静态方法`get()`实现一个`RequestMetrics`工厂接口。监听器使用计数器实现了当前正在请求的数量。请注意，`start()`和`end()`方法之间的状态是使用`MyMetricsState`类（一个非常简单的数据类，未在上面的示例中列出）共享的，使用`Context`.

上面列出的示例类`RequestMetrics`在静态`get()`方法中实现了工厂接口。监听器实现使用计数器来测量当前正在传输的请求数。请注意，`start()`和`end()`方法之间的状态是使用`MyMetricsState`类（一个非常简单的数据类，未在上面的示例中列出）共享的，使用`Context`进行传递。

您可以添加`RequestMetrics`到`InstrumenterBuilder`using`addRequestMetrics()`方法。

您可以在`InstrumenterBuilder`使用`addRequestMetrics()` 方法来添加`RequestMetrics`。

### 使用`ContextCustomizer`对`Context`提供丰富的操作

在一些罕见的情况下，需要`Context`在从`Instrumenter#start()`方法返回之前填充它。该`ContextCustomizer`接口可用于实现这一点。它暴露了一个`start()`方法并`Context`、 `REQUEST`和`Attributes`参数，在操作开始时提取的方法，并返回一个修改后的`Context`.

参考以下示例：

```java
class MyContextCustomizer implements ContextCustomizer<Request> {

  @Override
  public Context start(Context context, Request request, Attributes startAttributes) {
    return context.with(new InProcessingAttributesHolder());
  }
}
```

示例`ContextCustomizer`上方插入一个额外的`InProcessingAttributesHolder`到`Context`并在`Instrumenter#start()`方法之前将其从返回。在`InProcessingAttributesHolder`类，正如它的名字所暗示的，可以用来跟踪不可用的要求开始或结束的属性-例如，如果用仪器库只在加工过程中暴露的重要信息。可以从当前查找持有者类，`Context`并在检测操作开始和结束之间填充该信息。它可以稍后作为`RESPONSE`类型（或其一部分）传递给`Instrumenter#end()`方法，以便配置`AttributesExtractor`可以处理收集的信息并将其转换为遥测属性。

您可以在`InstrumenterBuilder`使用`addContextCustomizer()` 方法来添加`ContextCustomizer`。



### 禁用检测

在极少数情况下，完全禁用构造`Instrumenter`可能很有用，例如，基于配置属性。`InstrumenterBuilder`暴露了`setDisabled()` 方法：通过`true`将转向新创建`Instrumenter`成无操作实例。



### 最后，通过 `SpanKindExtractor` 设置span类型并获取一个新的 `Instrumenter`!

使用`InstrumenterBuilder`以下的一种方法来使`Instrumenter`创建结束：

- `newInstrumenter()`：返回的`Instrumenter`将始终以 kind 开头`INTERNAL`。
- `newInstrumenter(SpanKindExtractor)`：返回的`Instrumenter`将始终以传递的`SpanKindExtractor`.
- `newClientInstrumenter(TextMapSetter)`：返回的`Instrumenter`将始终启动`CLIENT` 跨度并将操作上下文传播到传出请求中。
- `newServerInstrumenter(TextMapGetter)`：返回的`Instrumenter`将始终启动`SERVER` 跨度，并将从传入请求中提取父跨度上下文。
- `newProducerInstrumenter(TextMapSetter)`：返回的`Instrumenter`将始终启动`PRODUCER` 跨度并将操作上下文传播到传出请求中。
- `newConsumerInstrumenter(TextMapGetter)`：返回的`Instrumenter`将始终启动`SERVER` 跨度，并将从传入请求中提取父跨度上下文。



最后4个变种用来创建非`INTERNAL` span，它接收`TextMapSetter` 或`TextMapGetter`实现作为参数。这些是正确实现服务之间的上下文传播所必需的。如果您想了解如何使用和实现这些接口，请阅读[OpenTelemetry Java 文档](https://opentelemetry.io/docs/java/manual_instrumentation/#context-propagation)。

`SpanKindExtractor`由上面列表中的第二个变种接收的接口是一个简单的接口，它接收一个`REQUEST`并返回 `SpanKind`，该接口应在启动此操作的span时使用。

参考以下示例：

```java
class MySpanKindExtractor implements SpanKindExtractor<Request> {

  @Override
  public SpanKind extract(Request request) {
    return request.shouldSynchronouslyWaitForResponse() ? SpanKind.CLIENT : SpanKind.PRODUCER;
  }
}
```

实例`SpanKindExtractor`根据请求处理决定是否使用`PRODUCER`或`CLIENT`。此示例反映了一个真实场景：您可能会在消息传递库检测中找到类似的代码，因为根据[OpenTelemetry 消息传递语义约定](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/semantic_conventions/messaging.md#span-kind) ，如果发送消息是完全同步的并等待响应，则span 类型设置为`CLIENT`。
