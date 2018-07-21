# 实用的 ProGuard 规则使用例子

> 原文 (Medium)：[Practical ProGuard rules examples](https://medium.com/google-developers/practical-proguard-rules-examples-5640a3907dc9)
>
> 作者：[Wojtek Kaliciński](https://medium.com/@wkalicinski?source=post_header_lockup)

[TOC]

在我之前的文章中，我解释了[为什么每个人都应该在他们的 Android 应用程序中使用 ProGuard](https://medium.com/google-developers/troubleshooting-proguard-issues-on-android-bce9de4f8a74)，如何启用它，以及在这样做时你可能会遇到什么样的错误。 这里涉及到很多理论，因为我认为了解潜在的原则是很重要的，这样才能准备好应对任何潜在的问题。 

我还在一篇单独的文章中谈到了[为一个即时应用构建配置 ProGuard](https://medium.com/google-developers/enabling-proguard-in-an-android-instant-app-fbd4fc014518) 的具体问题。 

在这一部分中，我想谈谈 ProGuard 规则在一个中型规模的示例应用程序的实例：[Nick Butcher](https://medium.com/@crafty) 的 [Plaid](https://github.com/nickbutcher/plaid)。

## 从 plaid 学到的经验

实际上，plaid 是研究 ProGuard 问题的一个很好的主题，因为它包含了一个混合的第三方库，它们使用诸如注释处理和代码生成、反射、 java 资源加载和本地代码(JNI)。 我提取并记录了一些实用的建议，这些建议应该适用于其他应用程序: 

## 数据类

```java
public class User {
  String name;
  int age;
  ...
}
```

可能每个应用程序都有某种数据类(也称为 DMOs，模型等等，这取决于上下文以及它们在应用架构中的位置)。 关于数据对象的问题是，通常在某一点上，它们将被加载或保存(序列化)到其他媒介，例如网络(HTTP 请求)、数据库(通过 ORM)、磁盘上的 JSON 文件或者 Firebase 数据存储。 

许多简化这些字段的序列化和反序列化的工具依赖于反射。 Gson，Retrofit，Firebase ー他们都在数据类中检查字段名称，并将其转化为另一个表示形式(例如: {"name":"Sue","age": 28}) ，用于传输或存储。 当他们将数据读入 Java 对象时，也会发生同样的情况ーー他们看到一个键值对"name":"John"，并试图通过查找字符串名字字段将其应用到 Java 对象上。 

**结论**：我们不能让 ProGuard 重命名或删除这些数据类中的任何字段，因为它们必须与序列化格式相匹配。 在整个类中添加 @keep 注释或者在所有模型上添加一个通配符规则: 

```groovy
-keep class io.plaidapp.data.api.dribbble.model.** { *; }
```

> **警告**：当测试你的应用是否容易受到这个问题的影响时，你可能会犯错误。 例如，如果你将一个对象序列化到 JSON，并将它保存到应用程序的版本 n 中，而没有适当的保留规则，保存的数据可能会是这样的: {“a”：“Sue”，“b”：28} 。由于 ProGuard 将你的字段更名为 a 和 b，所有内容似乎都能正常工作，数据将被保存并正确加载。
>
> 但是，当你再次构建应用并发布应用版本 N + 1 时，ProGuard 可能会决定将你的字段重命名为不同的内容，例如 c 和 d。因此，以前保存的数据将无法加载。
>
> 你必须确保你有正确的保持规则。

## 从本地调用的 Java 代码（JNI）

安卓[默认的 ProGuard 文件](https://developer.android.com/studio/build/shrink-code.html#shrink-code)(你应该总是包含它们，它们有一些非常有用的规则)已经包含了一个在 native 端实现的方法规则(- keepclasswithmembernames 类  { native methods; })。 不幸的是，没有一种全面的方法可以将代码保持在相反的方向: 从 JNI 到 Java。 

使用 JNI，完全有可能构造一个 JVM 对象，或者从 c / c + + 代码中找到并调用一个 JVM 句柄，事实上，[Plaid 中使用的其中一个库就是这样做的](https://github.com/Uncodin/bypass/blob/master/platform/android/library/jni/bypass.cpp#L61)。 

**结论**：由于 ProGuard 只能检查 Java 类，它将不知道在 native 代码中发生的任何用法。 我们必须通过 @keep  注解或保留规则来明确保留类和成员的这些用法。 

```groovy
-keep, includedescriptorclasses 
            class in.uncod.android.bypass.Document { *; }
-keep, includedescriptorclasses 
            class in.uncod.android.bypass.Element { *; }
```

## 从 JAR / APK 打开资源

安卓有自己的资源和资产系统，通常不会成为 ProGuard 的问题。 然而，在普通 Java 中，还有另外一种[直接从 JAR 文件加载资源的机制](https://docs.oracle.com/javase/8/docs/technotes/guides/lang/resources.html)，一些第三方库即使在安卓应用程序中编译时也可能使用它(在这种情况下，他们会尝试从 APK 加载)。 

问题在于，这些类通常会在自己的包名下寻找资源(这个名称在 JAR 或 APK 中转换为文件路径)。 在混淆时，ProGuard 可以重命名包名，因此在编译之后，类和资源文件可能不再在最后的 APK 中处于同一个包中。 

通过这种方式来识别加载资源，你可以在代码中以及在你所依赖的任何第三方库中寻找调用到 Class.getResourceAsStream / getResource / getResource。 

**结论**：我们应该使用该机制保留 APK 资源的任何类的名称。 

在 Plaid 中，OkHttp 库中有两个，Jsoup 中有一个：

```groovy
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase
-keepnames class org.jsoup.nodes.Entities
```

## 如何提出第三方库的规则

在理想的世界中，你使用的每个依赖项都将在 AAR 中提供必需的 ProGuard 规则。有时他们会忘记这么做，或者只发布 JAR，这些 JAR 没有标准化的方式来提供 ProGuard 规则。

在这种情况下，在开始调试应用程序并制定规则之前，请记住检查文档。一些库的作者提供了推荐的 ProGuard 规则（例如在 plaid 中使用的 Retrofit），这可以为你节省大量时间和挫败感。不幸的是，许多库没有这样做（例如本文提到的 Jsoup 和 Bypass）。另外请注意，在某些情况下，随库提供的配置只能在禁用优化的情况下工作，所以如果打开它们，你可能会处于未知的领域。

那么如何在库不提供这些规则时提出规则？ 我只能给你一些建议：

1. 阅读构建输出和 logcat！


2. 构建警告会告诉你哪些 -dontwarn 规则要添加。
3. ClassNotFoundException，MethodNotFoundException 和 FieldNotFoundException 将会告诉你哪些 -keep 规则需要添加。

> 当你的应用程序因为 ProGuard 启用而崩溃时，你应该感到高兴——你将有一个地方可以开始你的调查:) 
>
> 调试中最糟糕的问题是你的应用程序工作时，但是没有显示屏幕或者没有从网络中加载数据 
>
> 这就是你需要考虑我在这篇文章中描述的一些场景，把你的手弄脏，甚至潜入第三方代码，理解为什么它可能失败，比如当它使用 reflection， introspection  或者 JNI 时 

## 调试和堆栈跟踪

默认情况下，ProGuard 将删除程序执行不需要的许多代码属性和隐藏的元数据。 其中一些对开发人员实际上是有用的ーー例如，你可能希望保留源文件名和堆栈跟踪的行号，以便更容易地进行调试: 

```groovy
-keepattributes SourceFile, LineNumberTable
```

> 你也应该记住[当你构建一个发布版本并上传到 Play 时，保存 ProGuard 映射文件](https://developer.android.com/studio/build/shrink-code.html#decode-stack-trace) 从你的用户所经历的任何崩溃中获取去混淆的堆栈痕迹 。

如果要在应用程序的 proguarguard 构建中附加一个调试器来通过方法代码，你还应该保留以下属性以保留关于局部变量的一些调试信息(你只需要在调试生成类型中使用此行) : 

```groovy
-keepattributes LocalVariableTable, LocalVariableTypeTable
```

## 小型调试构建类型

默认的构建类型被配置，这样调试不会运行 ProGuard。 这是有道理的，因为我们想在开发时快速迭代和编译，但是仍然希望使用 ProGuard 的版本尽可能小并且优化。 

但为了全面测试和调试任何 ProGuard 问题，像这样建立一个独立的、小型化的调试版本是很好的 : 

```groovy
buildTypes {
  debugMini {
    initWith debug
    minifyEnabled true
    shrinkResources true
    proguardFiles getDefaultProguardFile('proguard-android.txt'), 
                  'proguard-rules.pro'
    matchingFallbacks = ['debug']
  }
}
```

使用这种构建类型，你可以[连接调试器](https://developer.android.com/studio/debug/index.html)，[运行 UI 测试](https://developer.android.com/training/testing/ui-testing/espresso-testing.html)（也在 CI 服务器上）或 [monkey  测试](https://developer.android.com/studio/test/monkey.html)你的应用程序，以查找尽可能接近你的发布版本的构建中的可能问题。

**结论**：结论: 当你使用 ProGuard 时，你应该始终保证你的发布完全建立起来，要么通过端到端测试，要么手动浏览应用程序中的所有屏幕，看看是否有什么东西丢失或者崩溃。 

## 运行时注解，类型自我检查

ProGuard 默认会从你的代码中删除所有注解，甚至是一些多余的类型信息。对于一些不存在问题的库来说，  - 那些在编译时处理注解并生成代码的库（如 Dagger 2 或 Glide 等）可能在程序运行后不需要这些注解。

还有另外一类工具，它们实际检查注释或查看运行时参数和异常的类型信息。 例如，通过使用代理对象拦截方法调用，然后查看注释和键入信息，以决定从 HTTP 请求中输入或读取什么内容。 

**结论**：有时需要保留在运行时读取的类型信息和注解，而不是编译时间。你可以[在 ProGuard 手册中查看属性列表](https://www.guardsquare.com/en/proguard/manual/attributes)。

```groovy
-keepattributes Annotation, Signature, Exception
```

> 如果你使用默认的 Android ProGuard 配置文件（getDefaultProguardFile ('proguard-android.txt')），则会为你指定前两个选项 - 注解和签名。如果你不使用默认值，你必须确保自己添加它们（如果你知道它们是你的应用程序的需求，那么只需复制它们也不会有什么坏处）。

## 将所有信息转移到默认包中

ProGuard 配置中默认不添加 -repackageclasses 选项。如果你已经在混淆你的代码，并且已经修正了任何问题与适当的保留规则， 你可以添加此选项以进一步减少 DEX 大小。它通过将所有类移动到默认（根）包来实现，实质上释放了像 “com.example.myapp.somepackage” 这样的字符串占用的空间。

```groovy
-repackageclasses
```

## ProGuard 优化

正如我之前提到的，ProGuard 可以为你做三件事：

1. 它可以删除未使用的代码， 
2. 重命名标识符使代码更小， 
3. 执行整个程序优化 。

我看到它的方式，每个人都应该尝试配置他们的版本，以获得1和2的工作。

要解锁3.（附加优化），你必须使用不同的默认 ProGuard 配置文件。在 build.gradle 中将 proguard-android.txt 参数更改为 proguard-android-optimize.txt：

```groovy
release {
  minifyEnabled true
  proguardFiles 
      getDefaultProguardFile('proguard-android-optimize.txt'),
      'proguard-rules.pro'
}
```

这将使你的发布速度更慢，但是可能会使你的应用程序运行更快，并且进一步缩小代码大小，这要归功于内联、类合并和更积极的代码删除等优化。 但是，要做好准备，它可能会引入新的和难以诊断的错误，所以谨慎使用它，如果有什么问题，一定要禁用某些优化或者完全禁用优化配置。 

在 Plaid 中，ProGuard 优化干扰了 Retrofit 如何在没有具体实现的情况下使用代理对象，并且去掉了一些实际需要的方法参数。 我不得不把这句话加到我的配置里: 

```groovy
-optimizations !method/removal/parameter
```

你可以[在 ProGuard 手册中找到可能的优化列表以及如何禁用它们](https://www.guardsquare.com/en/proguard/manual/optimizations)。

## 什么时候使用 @Keep 和 -keep

@Keep 在默认的 Android ProGuard 规则文件中，@Keep 实际上是一堆 -keep 规则，因此它们基本上是等价的。 因此它们基本上是等效的。指定 -keep 规则更加灵活，因为它提供了通配符，你也可以使用不同的变体，这些变体可以做一些略微不同的事情 （-keepnames，-keepclasseswithmembers 等）。

每当需要一个简单的"保持这个类"或者"保持这个方法"规则，我实际上更喜欢简单地在类或成员上添加 @keep 注解，因为它与代码保持一致，就像文档一样。 

如果一些其他开发人员想要重构代码，他们会立即知道标有 @Keep 的类/成员需要特殊处理，而不需要记得查看 ProGuard 配置，并冒着破坏某些东西的风险。此外，IDE 中的大多数代码重构都应该自动保留带类的 @Keep 注释。

## Plaid 统计

以下是来自 Plaid 的一些统计数据，显示了使用 ProGuard 可以删除多少代码。在一个更复杂的应用程序上，有更多的依赖性和更大的 DEX，节省可能更大。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotgkrpktj30hi06paac.jpg)

