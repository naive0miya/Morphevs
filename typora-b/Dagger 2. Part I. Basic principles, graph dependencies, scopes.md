# 第一部分. 基本原理、依赖关系图、范围

> 原文 (Medium)：[Part I. Basic principles, graph dependencies, scopes.](https://android.jlelse.eu/dagger-2-part-i-basic-principles-graph-dependencies-scopes-3dfd032ccd82)
>
> 作者：[Eugene Matsyuk](https://android.jlelse.eu/@eugenematsyuk?source=post_header_lockup)

[TOC]

Dagger 2 文章系列： 

1. Dagger 2. Part I. Basic principles, graph dependencies, scopes.
2. [Dagger 2. Part II. Custom scopes, Component dependencies, Subcomponents](https://proandroiddev.com/dagger-2-part-ii-custom-scopes-component-dependencies-subcomponents-697c1fa1cfc).
3. [Dagger 2. Part three. New possibilities](https://proandroiddev.com/dagger-2-part-three-new-possibilities-3daff12f7ebf).

大家好！目前我们已经有很多工具，使得 android 编码更容易的库。你应该追踪并尝试使用。Dagger 2 库就是这样的工具之一。

网上有很多关于这个库的教程。但当我开始研究 Dagger 2，阅读文章，观看评论时，我发现了一件坏事。 对于那些不使用 Spring 和其他类似框架或库的人来说，要理解依赖图的创建和提供的方式，对我来说是相当困难的。听众和读者通常有很多带有新注解的代码。不管怎样，这种方法应该奏效。 最后，在任何评论或文章之后，我的脑海中没有一个完整清晰的理解。 

现在我明白了，我没有好的图片，图形和方案，可以显示"什么，从哪里，去哪里"。 这就是为什么在我的文章中我试图感受这个缺口(加上这一点)。 我希望它能帮助初学者和其他相关人员更好地理解 Dagger 2，并选择它作为自己的项目。 我现在可以说，这是值得做的。 

起初我想写一篇文章，但是我有太多的教程和图片，并且决定把信息分成很小的部分。 在这种情况下，读者将能够逐步接近这个主题。 

## 理论

首先，我想简短地描述一些理论方面。dagger 2是一个帮助开发人员实现依赖注入模式(一种特殊形式的控制反转)的库。 

### 控制反转（IoC）原则

1. 顶层模块不应该依赖于较低层次的模块。所有层次的模块都应该依赖于抽象。 
2. 抽象不应该依赖于细节。 细节应该取决于抽象 。

### 用IoC消除设计缺陷

1. 刚性。如果我们改变一个模块，其他模块也会被更改。
2. 脆弱性。如果我们改变程序的一部分，其他部分就会出现无法控制的错误 。
3. 不动性。很难将单个模块与应用程序的其余部分分开使用 。

## 依赖注入（DI）

这是一个为任何程序组件提供外部依赖的过程。 当用于依赖项控制时，它是 IoC 的一种特殊形式。 根据单一责任原则，该对象允许外部专门设计的机制全面照顾构建所需的依赖关系。因此，Dagger 2也确实在制造这样一个通用的机制 

虽然我期待关于 IoC，DI 及其交互的问题和讨论，我想补充一点，这些定义来自维基百科，而且他们的详细讨论不是本文的主题。 

现在我想介绍一下这个依赖注入库的主要优势。

## Dagger 2的优势

1. 简单的共享实现。 
2. 复杂依赖关系的简单设置。 大的应用程序通常有很多依赖性。 Dagger 2允许你轻松控制所有的依赖 。
3. 简单的单元测试和集成测试。 我们将在关于使用 Dagger 2测试的文章中讨论这个问题。 
4. 支持局部单例。
5. 代码生成。收到的代码很清晰，可用于调试。
6. 没有混淆的问题。 第5和第6点都是 Dagger 2的优势属性，与 Dagger 1相比较。 Dagger 1使用反射工作。 这就是为什么运行时存在性能问题、混淆、奇怪错误的原因 
7. 小型库。

作为简单访问 “共享” 实现的示例，我使用了一个代码: 

```java
public class MainActivity extends AppCompatActivity {

    @Inject RxUtilsAbs rxUtilsAbs;
    @Inject NetworkUtils networkUtils;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        App.getComponent().inject(this);
    }

}
```

@Inject 注解被添加到字段且 App.getComponent ( ).inject（this）被添加到 onCreate 方法。现在，MainActivity 类可以访问 RxUtilsAbs 和 NetworkUtils 的现成实现。

这些优势使得 Dagger 2 成为现在在 Android 中实现 DI 的最佳库。这个库当然有它的缺点。在我的文章系列结束时我们会谈论它们。 现在我的任务是激发你的兴趣，激励你尝试 Dagger 2。 

Dagger 2 的基本元素（注解）

1. @Inject 注射 - “请求依赖关系”。
2. @Module 模块 -  “提供依赖关系”。
3. @Provide 提供者 - @Module 中的方法，“告诉 Dagger 我们如何构建和呈现依赖关系”。
4. @Component 组件 - @Inject 和 @Module 之间的桥梁。
5. @Scope 范围 - 创建全局和局部单例。
6. @Qualifier - 如果同一类型的不同物体是必要的。

现在，请阅读这些注解以获得大致的想法。我们将讨论它们中的每一个。所以理论完成了。在我的文章结尾处可以通过参考文献更仔细地研究它。我们的主要目标是了解如何通过 Dagger 2 构建整个依赖关系图。

## 练习

现在我们从更有趣的事情开始。我们来研究一个特定的案例。从每个应用程序都有的单例开始。在 Android 中，我们使用单例因为活动和片段的生命周期。我希望将使用的单例分成两类：

1. 应用程序任何部分都可能需要的 “全局” 单例。这些是 Context ，utility 类和影响应用程序工作的实用程序类和其他类 。
2. 仅在一个或多个模块中需要 “局部” 单例。由于可能的屏幕旋转和其他原因，逻辑和数据的一部分应该经常被移除到与生命周期无关的地方。我将在下一篇文章中详细描述"局部"单例。 

我想从“全局”单例开始。我们通常如何使用它们？我想以下代码应该是使用最多的 ：

```java
SomeSingleton.getInstance().method();
```

这是常见的做法。但是如果我们想要使用 DI 模式，这个代码不能满足，原因如下：

1. SomeSingleton 在使用此类时可能存在其他的依赖关系。这不是一个明确的依赖关系，因为它不清楚（在构造函数中或在字段中或在方法中 ）。这就是为什么我们只能通过观察特定方法的代码才能看到这种依赖关系。但是看看类接口，我们很难假设这里使用的 SomeSingleton。
2. SomeSingleton 正在执行初始化过程。如果应用了延迟初始化，则使用 SomeSingleton 的类将启动初始化进程（首先要求进行初始化）。因此，除了它们的工作之外，其他类也负责启动单例初始化。
3. 如果我们有很多这样的单例，系统就充满了隐式的依赖。有些单例可以依靠其他类，也不能使他们的进一步支持变得容易。此外，单例在系统中交错，并且可以在不同的包裹中。这可能也会带来不便。

虽然可以忍受。但是当你想在你的代码中使用单元测试时，所有的事情都发生了很大的变化。你必须对这些隐式的依赖做一些事情，正确地替换它们。现在你愿不愿意将代码转换为任何"可测试代码"。如果你有隐式的依赖，那是绝对不可能的。 

而 Dagger 2 到来。现在我们可以看到我们如何通过 Dagger 2 来实现带有单例的 DI 模式 。我们还可以看到依赖图构建的整个周期。 

我想从“全局”单例开始。

## 创建单例

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovx12advj31hc0u0gnw.jpg)

正如我们前面提及 @Module 是一个注解，它标志着类提供的依赖性。我将把这些类称为 Modules。 我将把提供依赖的方法称为提供方法。例如 ReceiversModule 中有一个 provideNetworkChannel 方法。该方法提供了 NetworkChannel 类型的对象。这个方法可以有任何名称。最重要的是 @Provides 在方法和返回类型（NetworkChannel）之前提供注解。当返回类型是接口或抽象类（RxUtilsAbs），我们最终初始化并返回需要的实现（RxUtils）。下面我将描述 @Singleton 注解，这就是为什么我们现在不注意它的原因。可以将必要的对象发送给模块和构造函数。示例 - AppModule。UtilsModule 更有趣。该模块需要具有 Context 和 NetworkChannel 类型的对象才能提供依赖关系。这意味着我们应该告诉 Dagger 我们需要 Context 和 NetworkChannel 来创建 RxUtilsAbs 和 NetworkUtils 对象。为此，我们为方法 provideRxUtils 提供参数 Context context，为 provideNetworkUtils 方法添加参数 Context context，NetworkChannel networkChannel 。

在这种情况下，参数可以有任何名称（context 或 contextSuper）。参数的类型非常重要。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovx6octjj31130ai74m.jpg)

创建一个接口 AppComponent，然后我们使用注解 @Component（modules = {AppModule.class，UtilsModule.class，ReceiversModule.class}）。

为了方便起见，我们将此接口命名为 Component。如上所述 @Component 是 @Module 和 @Inject 之间的桥梁。换句话说，Component 是已完成的依赖关系图。这是什么意思？我们稍后会理解它。

使用这个注解，我们对告诉 Dagger ，AppComponent 包含三个模块：AppModule，UtilsModule，ReceiversModule。每个这样的模块提供的依赖关系可用于由 AppComponent 组合的所有其他模块。下面的图片很好地说明了这一点。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovxa9rfwj31hc0u0400.jpg)

我认为这张图片很好地展示了  Dagger 2 从哪里获取 Context 和 NetworkChannel 对象来构建 RxUtilsAbs 和 NetworkUtils。例如，如果我们从组件注释中移除 AppModule 而不是在编译时，Dagger 2 将发生冲突并询问它可以在哪里获取对象上下文。我们也在接口中声明方法 void inject（MainActivity mainActivity）。使用这种方法，我们告诉 Dagger 我们想注入哪个类。

 我应该补充一点 ，如果你想将依赖关系注入除 MainActivity 之外的另一个类（例如 SecondActivity），则必须在接口中声明它。例如，

```java
@Component(modules = {AppModule.class, UtilsModule.class, ReceiversModule.class})
@Singleton
public interface AppComponent {
    void inject(MainActivity mainActivity);
    void inject(SecondActivity secondActivity);
}
```

参数可以有任何名称（mainActivity 可以通过活动等来替换）。要注入的对象的类型具有重要意义！我们不能为所有提供依赖关系的类使用任何通用类型，如下所示：

```java
@Component(modules = {AppModule.class, UtilsModule.class, ReceiversModule.class})
@Singleton
public interface AppComponent {
    void inject(Object object);
}
```

由于 Dagger 2是用代码生成操作的， 而不是使用反射，所以您必须设置具体的类型（不能使用抽象或泛型）！ 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovxeyy9tj31hc0u078q.jpg)

我们讨论更深点。正如您记得的，我们将 MainActivity 类设置为 AppComponent 中的注入目标。在这个类中，我们可以使用由模块 AppModule，UtilsModule，ReceiversModule 提供的依赖关系。为此，我们可以在类中添加适当的字段，使用注解 @Inject 标记它们并使它们可用（至少包可见性）（如果该字段被设置为私有， Dagger 不能提供正确的实现）。

应该指出的是，应该将类 RxUtils（RxUtils 作为 RxUtilsAbs 的子项）添加到我们在模块 UtilsModule 中设置的字段 RxUtilsAbs rxUtilsAbs 中。

在 onCreate 方法中，我们添加一行 。App.getComponent( ).inject(this);

由于我们讨论了创建单例，我们的 AppComponent 的组件应该更好地保存在类 Application 中。在我们的例子中，AppComponent 可以通过 App.getComponent（）来访问。

通过方法 inject（MainActivity mainActivity），我们完全连接了我们的依赖关系图。因此，可以在 MainActivity 中访问模块 AppComponent（Context，NetworkChannel，RxUtilsAbs，NetworkUtils）提供的所有依赖关系。

我们来讨论类 App 的 buildComponent ( ) 方法。

在编译之前不能访问 DaggerAppComponent 组件。这就是为什么我们一开始不应该关注 IDE，因为 IDE 可以说没有类 DaggerAppComponent 组件。 IDE 也无法帮助构建器创建。所以我们必须首次使用 DaggerAppComponent “盲目” 编写初始化 AppComponent。

顺便说一句，代码 buildComponent ( ) 可以减少：

```java
protected AppComponent buildComponent() {
        return DaggerAppComponent.builder()
                .appModule(new AppModule(this))
                .build();
    }
```

正如我们之前提到的，Dagger 2 负责创建整个依赖关系图。如果出了什么问题，  Dagger 2 将通过编译器报告。而且我们在运行时没有像在 Dagger 1那样出现意外和不明确的崩溃。 请看下面的图。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovxifyjdj31hc0u0wg1.jpg)

好吧，我们现在可以放松一下！信息量最多的部分已经结束。我认为该图清楚地表明：

1. 模块提供依赖关系。 这意味着我们应该在模块中声明要提供的对象。 
2. 组件是一个依赖图。 它组合模块并为需要的类提供依赖。 

如果有不清楚或不明显的地方，请添加评论。我会纠正和解释一切。

最后，我将讨论注解 @Singleton。它是由 Dagger 提供的一个范围注解。如果将 @Singleton 放置在提供依赖项的方法之前，Dagger 将在组件初始化期间创建一个标记依赖(或单例)的单个版本。 它将在每次调用 dependencies 时提供该单个版本。 

少说话，多展示图片！

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovxlujt0j30ze0quq47.jpg)

每个依赖项都提供了注解 @Singleton。 这意味着每当 Dagger 不得不使用这种依赖时，它只会使用它的单一版本。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovxpb67fj31h60jtq3w.jpg)

现在进行比较，我们从方法 provideNetworkChannel 中移除注解 @singleton (依赖项将被解除)。 这意味着每当 Dagger 不得不使用这种依赖时，它每次都会使用新的版本。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovxthhuoj30yv0qzmyf.jpg)

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovxwnwtgj31hc0iudgt.jpg)

我们也可以创建自定义范围注解（更多详细信息在我的下一篇文章中）。 您可以在下面看到范围注解的一些特性：

1. 通常为组件设置范围注解并提供方法。 
2. 如果至少有一个提供方法有范围注解，组件应该具有相同的范围注解。
3. 只有在所有模块中的所有提供的方法都没有作用的时候，才能对组件进行解析。
4. 一个组件的所有范围注解(对于所有提供方法的模块，都应该是组件本身的一部分)都应该是相同的。 

关于范围注解的更多细节将在我的下一篇文章中描述 。

这是第一篇，在本文中，我们研究了 IoC，DI，Dagger 2 的理论方面，通过 Dagger 2对依赖图的创建进行了详细的研究，对范围注解和 @singleton 的实际实现进行了研究。 

