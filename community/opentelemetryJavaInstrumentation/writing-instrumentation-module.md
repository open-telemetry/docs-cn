# 循序渐进开发一个 `InstrumentationModule`

`InstrumentationModule` 是 OpenTelemetry javaagent instrumentation 的核心部分。
我们的 javaagent 使用了许多约束，在实现模块时必须按照许多不明显的规范。  

本文档试图描述如何实现你的 javaagent instrumentation ，并记录可能影响增强的所有问题。在阅读这篇文档之前，我们建议你先行阅读 
`InstrumentationModule` 和 `TypeInstrumentation` 的 Javadoc ，因为它们经常提供关于如何使用某个特定方法的更详细的解释
（以及为什么它是这样工作的）。      

## 每个javaagent检测的核心：`InstrumentationModule`

一个 `InstrumentationModule` 描述了一组独立的 `TypeInstrumentation` ，它们需要组合在一起才能为一个库准确进行增强。 
Type instrumentations 被组合在一个模块当中并共享辅助类和[运行时 muzzle 校验](muzzle.md)，并处于相同的类加载器之中，它们将会一起
被启用禁用。      

OpenTelemetry javaagent 将会使用 Java `ServiceLoader` API 来寻找所有的模块。为了保证你的 instrumentation 能够被访问到你需要保证一个
正确的 `META-INF/services/` 处于 javaagent 的 jar 包之中。当然最简单的方式是使用 `@AutoService` 注解：       

```java
@AutoService(InstrumentationModule.class)
class MyLibraryInstrumentationModule extends InstrumentationModule {
  // ...
}
```

一个 `InstrumentationModule` 至少需要一个名称。 Javaagent 的使用者可以通过选择一个其中的名字来
[使一个 instrumentation 不生效](../suppressing-instrumentation.md)。 Instrumentation module 的名称应该遵循短横线规则。
核心的 instrumentation 名称 （module 中的第一个）应该与其 gradle 模块名相同（不包含其版本后缀）。      

```java
public MyLibraryInstrumentationModule() {
  super("my-library", "my-library-1.0");
}
```

关于 `InstrumentationModule` 名称更详细的信息请参阅 
`InstrumentationModule#InstrumentationModule(String, String...)` 的 Javadoc 。   

### 使用`order()`进行排序

如果你需要你的 instrumentation 按照特定的顺序生效（比如你的自定义 instrumentation 增强了内置的 servlet 因此需要在其之后运行）你可以覆盖
`order()` 方法去定义顺序：      

```java
@Override
public int order() {
  return 1;
}
```

越高的 `order()` 意味着 instrumentation 模块将会越迟被生效。默认值都是0。      

### 重写`isHelperClass()`方法

OpenTelemetry javaagent 将会收集在 instrumentation/advice 类中使用的辅助类并将其注入到应用类路径之上。这些类可以自动被
找到，但是也可以通过实现 `isHelperClass(String)` 来
显示声明哪些包和方法应该被视为辅助类。      

```java
@Override
public boolean isHelperClass(String className) {
  return className.startsWith("org.my.library.opentelemetry");
}
```

查看 [muzzle 文档](muzzle.md#%E7%BC%96%E8%AF%91%E9%98%B6%E6%AE%B5%E7%9A%84%E5%BC%95%E7%94%A8%E6%94%B6%E9%9B%86)来查看更多信息

### 使用`helperResourceNames（）`方法注入其他资源

有些库可以通过其 SPI 接口轻松地让你实现指标收集的能力。 OpenTelemetry javaagent 可以将 `ServiceLoader` 的文件注入，
但是需要如下声明：       

```java
@Override
public String[] helperResourceNames() {
  return new String[] {"META-INF/services/org.my.library.SpiClass"};
}
```

所有 `helperResourceNames()` 方法中定义的服务提供文件中所涉及到的方法都将会被视为辅助类： 它们将会被检查无效引用并自动
注入到应用类加载当中。      

### `classLoaderMatcher()`

同一个库的不同版本常常需要不同的 instrumentations ：打个比方， servlet 3 引入了几个新的异步类，这些类被初始化的时候需要被增强来收集
指标数据。一个 `InstrumentationModule` 可以定义是否进行增强的附加条件：    

```java
@Override
public ElementMatcher.Junction<ClassLoader> classLoaderMatcher() {
  return hasClassesNamed("org.my.library.Version2Class");
}
```

上面的例子当对应的库版本不包含所引入的类的时候时候，将会跳过应用中不包含对应类的代码增强。       

### `typeInstrumentations()`

最后，一个 `InstrumentationModule` 的实现至少需要一个 `TypeInstrumentation` 的实现：      

```java
@Override
public List<TypeInstrumentation> typeInstrumentations() {
  return Collections.singletonList(new MyTypeInstrumentation());
}
```

一个不包含 type instrumentations 的模块将什么都不会做。      

## `TypeInstrumentation`

一个 `TypeInstrumentation` 介绍了将会针对一个类型将会进行的改变。根据被插件增强的库的不同，type instrumentations 需要被组合在一起使用
（组合在一个模块中）。       

```java
class MyTypeInstrumentation implements TypeInstrumentation {
  // ...
}
```

### `typeMatcher()`

一个 type instrumentation 需要去定义什么样的类（单个或多个）将会被增强：    

```java
@Override
public ElementMatcher<TypeDescription> typeMatcher() {
  return named("org.my.library.SomeClass");
}
```

### `classLoaderOptimization()`

当你需要去增强实现了某个接口的所有类，或者增强所有实现了某个特定注解的类的话你需要实现 `classLoaderOptimization()` 方法。通过类名去匹配
的确很快，但是当你需要根据字节码（比如实现接口，是否实现注解，是否包含方法）去决定的时候是一个非常大的开销。 `classLoaderOptimization()` 
所返回的 matcher 将会使 `TypeInstrumentation` 对不包含这个库的应用进行增强的时候更加高效。        

```java
@Override
public ElementMatcher<ClassLoader> classLoaderOptimization() {
  return hasClassesNamed("org.my.library.SomeInterface");
}

@Override
public ElementMatcher<? super TypeDescription> typeMatcher() {
  return implementsInterface(named("org.my.library.SomeInterface"));
}
```

### `transformers(TypeTransformer)`

最后的 `TypeInstrumentation` 方法描述了对于所匹配的类型将会进行怎样的增强。 `TypeTransformer` （agent 内部所实现的接口） 定义
了一系列你可以使用的增强操作的集合：

* 调用 `applyAdviceToMethod(ElementMatcher<? super MethodDescription>, String)` 方法允许你使用一个 advice 类（第二个参数）对
  所有符合条件的方法（第一个参数）进行增强。这里建议尽可能的使方法 matcher 更加严格 - type instrumentation 应该只去增强那些应该被增强的类，也仅仅是这些。

* `applyTransformer(AgentBuilder.Transformer)` 允许你去使用一个任意的 ByteBuddy 转换器。这是一个激进，并不推荐的做法，这会导致其并不会
  被 muzzle 和 helper 类的检测 - 使用前请保持谨慎。 

```java
@Override
public void transform(TypeTransformer transformer) {
  transformer.applyAdviceToMethod(
    isPublic()
        .and(named("someMethod"))
        .and(takesArguments(2))
        .and(takesArgument(0, String.class))
        .and(takesArgument(1, named("org.my.library.MyLibraryClass"))),
    this.getClass().getName() + "$MethodAdvice");
}
```

当对 Java 类型进行匹配的时候你可以使用 `takesArgument(0, String.class)` 的格式。被增强库中的类需要通过 `named()`
 matcher 来进行匹配。    

`TypeInstrumentation` 的实现常常会定义 advice 类作为其静态内部类。通常在 `transform()` 方法中，
这些类在从方法名与 advice class 的映射中被通过名称引用。     

你可能已经注意到在例子当中 advice 类被一种有点奇怪的方式被引用：      

```java
this.getClass().getName() + "$MethodAdvice"
```

简单的引用内部类并调用 `getName()` 方法的方式相对于上述这种混合的方式便于阅读也更便于理解，但是请注意，这是**故意的**并应该保持。      

Instrumentation 模块被 agent 的类加载器所加载，这种字符串连接也是一种防止实际的 advice 类被加载到 agent 类加载器而做的优化。       

## Advice classes

Advice 类并不是实际的"类"，它们是将会直接拷贝增强到被增强库文件的零碎的代码碎片。你不应该把它们视为标准的 Java 类 - 许多标准并不适用于它：     

* 如果它们是内部类，那么它们必须是静态的
* 它们必须只包含静态方法；      
* 它们不能拥有任何状态（字段）- 静态常量也不行！只有 advice 方法的内容将会被拷贝到被增强的代码中，常量则不会；   
* 在 `InstrumentationModule` 或者 `TypeInstrumentation` 中定义的内部 advice 类禁止使用其他类的成员（日志，常量等等）；
* 通过提取通用方法或者父类来进行代码复用可能会无法正常工作：除非你可以创建一个额外的辅助类来存储被复用的代码；       
* 它们不应该包含任何不被 `@Advice` 所修饰的方法。       

```java
public static class MethodAdvice {
  @Advice.OnMethodEnter(suppress = Throwable.class)
  public static void onEnter(/* ... */) {
    // ...
  }

  @Advice.OnMethodExit(suppress = Throwable.class, onThrowable = Throwable.class)
  public static void onExit(/* ... */) {
    // ...
  }
}
```

在被 `@Advice` 上的注解声明 `suppress = Throwable.class` 选项非常的重要。 Advice 方法所抛出的异常将会被捕获并通过一个 
OpenTelemetry javaagent 所定义的特殊 `ExceptionHandler` 。这个 handler 将会确保合适地将所有未预料到的异常记录在日志当中。        

`OnMethodEnter` 和 `OnMethodExit` advice 方法常常会共享一部分信息。我们常常会使用 `otel` 前缀的变量来把上下文，作用域
（还有一些别的）在两个方法之间传递。       

```java
@Advice.OnMethodEnter(suppress = Throwable.class)
public static void onEnter(@Advice.Argument(1) Object request,
                           @Advice.Local("otelContext") Context context,
                           @Advice.Local("otelScope") Scope scope) {
  // ...
}
```

在产生观测数据的 instrumentations 当中这两个方法常常如下所示：     

```java
@Advice.OnMethodEnter(suppress = Throwable.class)
public static void onEnter(@Advice.Argument(1) Object request,
                           @Advice.Local("otelContext") Context context,
                           @Advice.Local("otelScope") Scope scope) {
  Context parentContext = Java8BytecodeBridge.currentContext();

  if (!instrumenter().shouldStart(parentContext, request)) {
    return;
  }

  context = instrumenter().start(parentContext, request);
  scope = context.makeCurrent();
}

@Advice.OnMethodExit(suppress = Throwable.class, onThrowable = Throwable.class)
public static void onExit(@Advice.Argument(1) Object request,
                          @Advice.Return Object response,
                          @Advice.Thrown Throwable exception,
                          @Advice.Local("otelContext") Context context,
                          @Advice.Local("otelScope") Scope scope) {
  if (scope == null) {
    return;
  }
  scope.close();
  instrumenter().end(context, request, response, exception);
}
```

你也许会意识到上面的这个例子没有使用 `Context.current()` ，而是一个 `Java8BytecodeBridge` 方法。这是故意的：如果你在增强一个
 Java 8 之前的库的时候，如果在这个库内插桩了 Java 8 才支持的接口 default 方法调用 （或者接口中的静态方法）的时候将会导致一个 
 `java.lang.VerifyError` 运行时异常，因为在 Java 7 （及之前）是不支持这部分在 Java 8 才加入的语法的。      
因为 OpenTelemetry API 拥有许多默认或者静态的通用接口方法（比如 `Span.current()`）， `javaagent-api` 模块中的方法
 `Java8BytecodeBridge` 提供了静态方法在 advice 中调用这些默认方法。      
 实际上，我们建议在 advice 类中避免使用 Java 8 的语法特性 - 有时候你无法预料到被增强的库使用什么样的 Java 版本来编译。        

有时候需要将某些上下文的类与被增强库中的类所关联，并且这个库并没有提供实现这点的途径。 OpenTelemetry javaagent 提供了 `ContextStore` 
来达到这个目的。       

```java
ContextStore<Runnable, Context> contextStore =
    InstrumentationContext.get(Runnable.class, Context.class);
```

一个 `ContextStore` 与 map 概念上非常类似。但这并不是一个简单的 map ：javaagent 使用了大量字节码修改技巧来实现这种优化。    
正是因为如此，检索一个` ContextStore` 实例受到了一定限制： `InstrumentationCon text#get()` 方法只能被 advice 方法所调用，
并且其只能接受类的引用作为参数 - 它不能配合变量即方法参数正常工作。     
作为 key 的类与上下文类必须在编译期间可以访问到。        