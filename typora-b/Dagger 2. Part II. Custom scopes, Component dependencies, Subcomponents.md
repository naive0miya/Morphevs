# 第二部分. 自定义范围、组件依赖、子组件

> 原文 (Medium)：[Dagger 2. Part II. Custom scopes, Component dependencies, Subcomponents](https://proandroiddev.com/dagger-2-part-ii-custom-scopes-component-dependencies-subcomponents-697c1fa1cfc).
>
> 作者：[Eugene Matsyuk](https://android.jlelse.eu/@eugenematsyuk?source=post_header_lockup)

[TOC]

Dagger 2 文章系列：

1. [Dagger 2. Part I. Basic principles, graph dependencies, scopes](https://android.jlelse.eu/dagger-2-part-i-basic-principles-graph-dependencies-scopes-3dfd032ccd82).
2. Dagger 2. Part II. Custom scopes, Component dependencies, Subcomponents.
3. [Dagger 2. Part three. New possibilities](https://proandroiddev.com/dagger-2-part-three-new-possibilities-3daff12f7ebf).

大家好！ 

继续我们的 Dagger 2 文章系列。 如果你还没有读过第一部分，你现在应该先阅读它。非常感谢大家的评论。这里我们将讨论自定义范围，组件通过组件依赖和子组件连接起来。并将触及诸如移动应用架构等重要主题，以及 Dagger 2如何帮助我们构建更干净、模块解耦的架构。 

欢迎感兴趣的大家！

## 架构和自定义范围

我们从架构开始吧。 近年来这一课题受到了广泛关注，许多文章和演讲都专门讨论这个问题。毫无疑问，主题是重要的，因为我们在俄罗斯的说法是:"当你给船起名字的时候，它就会漂浮起来。"。 因此，我强烈建议先阅读这些文章: 

1. [Clean Architecture from uncle Bob](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html)
2. [Clean Architecture in Android](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)

我非常喜欢简洁架构的方法 。它允许执行清洁的垂直和水平模块组合， 在这些模块中，每个实体都能完全做到它应该做的。 例如，片段只负责显示用户界面，而不是制作网络调用、数据库调用、业务逻辑等等，否则就会使片段变成一堆令人讨厌的复杂的代码串。 我觉得你们很多人都很熟悉。 

考虑下面的例子: 有一个应用程序，它有一些模块，其中一个是聊天模块。 聊天模块包含三个屏幕: 单一的聊天屏幕，群聊屏幕和设置。 

记住简洁架构，我们可以区分三个水平层次: 

1. 全局应用程序级。这里我们保留了整个应用程序生命周期所需的对象， 即全局单例。即 Context（应用程序上下文），NetworkUtils（实用程序类）和 IDataRepository（执行网络调用的类）。
2. 聊天级。三个聊天屏幕中每一个所需的对象：IChatInteractor（实现特定聊天业务案例的类）和 IChatStateController（负责聊天状态的类）。
3. 每个屏幕级。每个屏幕都有自己的 Presenter，可以重新定向（意味着它的生命周期比 Activity 的长）。

面你可以看到这些生命周期在示意图时间线: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovy4abc2j30p00b2q48.jpg)

你还记得我们在上一篇文章中提到的 “局部单例” 吗？现在，聊天级实体和每个聊天屏幕实体代表 “局部单例”，即生命周期大于 Activity / Fragment 生命周期的对象，但小于整个应用程序生命周期。

现在 Dagger 2 进入演练。它有个神奇的机制称为 Scopes 。此机制负责在存在相应范围的同时创建并保留所需类的单个实例。我敢打赌，“当相应的范围存在” 这个短语混淆了一下，并产生了问题。不要惊慌，下面的一切都会被清除。

上一篇文章，我们用 @Singleton 标记了 “全局单例” 范围。此范围始终存在于应用程序中。但是我们可以创建自己的自定义范围注解。例如：

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface ChatScope {
}
```

而 Dagger 2 的 @Singleton 注解声明如下所示：

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface Singleton {
}
```

那就是 @Singleton 与 @ChatScope 没有什么不同，第一个恰好是由 Dagger 默认提供的。这些注解的唯一目的是指出 Dagger 提供的是有范围还是无范围的对象。但同样，我们负责"作用域"对象的生命周期。 

回到我们的例子，因此对于我们当前的体系结构，我们有三个对象组，每个对象组都有自己的生命周期。 因此，我们需要三个范围的注解: 

1. @Singleton - 全局单例。
2. @ChatScope - 用于聊天对象。
3. @ChatScreenScope - 用于特定的聊天屏幕对象。

同时，我们注意到，@ChatScope 对象必须可以访问 @Singleton 对象，@ChatScreenScope 对象可以访问 @Singleton 和 @ChatScope 对象。

示意图如下：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovy94b2mj30p00avt9w.jpg)

还有相应的 Dagger 组件创建：

1. AppComponent 提供 “全局单例” 。
2. ChatComponent 为所有聊天屏幕提供 “局部单例” 。
3. SCComponent 为每个特定聊天屏幕提供 “局部单例”（SingleChatFragment，即单一聊天屏幕）。

再次想象上述实体: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovyc8qkvj30k00jpq46.jpg)

因此，我们得到了三个组件，它们有三个不同的范围注解，它们之间相互依赖。  SCComponent 依赖于ChatComponent , ChatComponent 依赖于 AppComponent 。

现在又出现了另一个重要的问题: 如何正确地将这些组成部分联系起来？ 有两种选择。 

## 组件依赖

该选项从 Dagger 1 迁移而来。

让我们马上表示组件依赖性特征：

1. 两个相互依赖的组件不能有相同的范围。更多细节在这里
2. 父组件必须显式声明可用于子组件的对象。
3. 组件可以依赖于其他组件。

对于我们的情况，依赖关系图看起来是这样的：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovyhim29j30p00e2mz0.jpg)

让我们分别看看每个组件的模块。

AppComponent

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovymxrm7j31e00s444d.jpg)

请注意，在 AppComponent 接口中 ，我们显式声明了可用于子组件的对象（但不是为其子组件 ，所以稍后我们将讨论这种情况）。例如，如果一个子组件需要 NetworkUtils，那么 Dagger 会给出相应的错误。

我们仍然可以在接口中声明注入目标/主体。你不应该被组件是父类的事实所误导，它仍然可以将依赖性本身注入必需的类(活动、片段等)。 

ChatComponent

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovysark6j31e00l4jv8.jpg)

我们显式声明了 ChatComponent 注解中的依赖关系（它依赖于 AppComponent）。顺便说一下，正如上面提到的，组件可以有多个父母（只需在其注解中添加新组件）。但是组件的范围注解必须是不同的。 此外，我们还声明了可以访问的对象，这些对象可以访问到接口中的子组件。 

注意上图中的绿色箭头（组件依赖关系的一般方案）？如前所述，ChatComponent 可以使用 AppComponent 的依赖项，这些依赖项被显式声明。 但是 ChatComponent 的孩子不能使用 AppComponent 的依赖项，除了那些在聊天组件中明确声明的依赖(我们在上下文中所做的)。 。 

SCComponent

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovyx57maj31e00ewach.jpg)

SCComponent 依赖于 ChatComponent 并将依赖注入到 SingleChatFragment 中。因此，它可以在对应于 SingleChatFragment 的对应接口中明确声明 SCPresenter 和其他父组件的对象。 

剩下的最后一步是初始化组件：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovz0hfz7j30p0098756.jpg)

与通用组件不同，我们在使用构建器初始化相关组件时有一个附加方法：appComponent（...）（用于 DaggerChatComponent）和 chatComponent（...）（用于 DaggerSCComponent），我们在这里传递一个初始化的父组件作为参数。顺便说一下，如果一个组件有两个父母，那么构建者有两种方法。三个父母就有三种方法，以次类推。

由于每个组件的生命周期都不同于 Activity / Fragment 生命周期，我们在 Application 类中存储组件实例。 最后将研究 Application  类的例子。 

## 子组件

这个是在 Dagger 2 中引入的。

特征 :

1. 父组件必须在其接口中声明子组件 getter 。
2. 子组件可以访问所有父母对象。
3. 子组件只能有一个父组件。

当然，子组件与组件依赖有所不同。 让我们研究一下方案和代码来清楚这一点。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovz3foo7j30p00e2dhf.jpg)

我们可以看到，子组件可以访问所有父物件，这在整个依赖树中都是正确的。例如，SCComponent 可以访问 NetworkUtils。

AppComponent

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovz84yt5j31e00s4jx0.jpg)

接下来，看看子组件的差异。让我们在 AppComponent 接口中创建 ChatComponent 初始化方法。主要部分是方法的签名：返回类型 ChatComponent 和参数（ChatModule）。

请注意，在我们的 ChatModule 的构造函数中，不需要任何对象（缺省构造函数），因此在方法 plusChatComponent 中可以省略此参数。但是，为了更清楚地了解依赖和教育性目的，我们将尽可能详细地说明。

ChatComponent

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovzpr9apj31e00l777z.jpg)

同时， ChatComponent 是一个子组件，也是父组件。事实上，这是一个东西的父母是由 SCComponent 创建方法在接口中声明的。 事实上，它是一个子组件，它的 @Subcomponent  注解指出。 

SCComponent

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovzv9g3uj31e00ettaw.jpg)

正如我们前面提到的，由于每个组件都有自己的生命周期，由于每个组件都有不同于活动 / 片段生命周期的自己的生命周期，我们在 Application  类中执行组件实例存储和初始化: 

```java
public class MyApp extends Application {

    protected static MyApp instance;

    public static MyApp get() {
        return instance;
    }

    // Dagger 2 components
    private AppComponent appComponent;
    private ChatComponent chatComponent;
    private SCComponent scComponent;

    @Override
    public void onCreate() {
        super.onCreate();
        instance = this;
        // init AppComponent on start of the Application
        appComponent = DaggerAppComponent.builder()
                .appModule(new AppModule(instance))
                .build();
    }

    public ChatComponent plusChatComponent() {
        // always get only one instance
        if (chatComponent == null) {
            // start lifecycle of chatComponent
            chatComponent = appComponent.plusChatComponent(new ChatModule());
        }
        return chatComponent;
    }

    public void clearChatComponent() {
        // end lifecycle of chatComponent
        chatComponent = null;
    }

    public SCComponent plusSCComponent() {
        // always get only one instance
        if (scComponent == null) {
            // start lifecycle of scComponent
            scComponent = chatComponent.plusSComponent(new SCModule());
        }
        return scComponent;
    }

    public void clearSCComponent() {
        // end lifecycle of scComponent
        scComponent = null;
    }

}
```

现在我们终于可以在代码中看到应用程序生命周期, 所以 AppComponent 是清晰的，我们在应用程序启动时初始化它，不再触摸它。不再触摸它。 但是当需要使用 plusChatComponent ( ) 和 plusSCComponent ( ) 方法时，ChatComponent 和 SCComponent 就被初始化了。 所以当你反复调用。这些方法还负责在每个调用中给出相同的实例。 

```java
scComponent = chatComponent.plusSComponent(new SCModule());
```

新的 SCCComponent 实例使用它自己的依赖关系图进行构建。

使用 clearChatComponent ( ) 和 clearSCComponent ( ) 方法，我们可以通过它们停止现有组件的生命周期。是的，只需将引用归零。如果我们再次需要 ChatComponent 和 SCComponent，我们只需调用创建新实例的 plusChatComponent ( ) 和 plusSCComponent ( ) 方法。

以防万一，我澄清一下，在这个特殊的例子中，当 ChatComponent 未初始化时，我们不能初始化 SCComponent ーー我们将得到 NullPointerException。 但是如果你有很多组件，我建议将这段代码从 MyApp 移动到一个特殊的单例（例如 Injector），它将负责创建，销毁和提供 Dagger 组件和子组件。

正如你所看到的，自定义范围，组件依赖和子组件是极其重要的 Dagger 2 元素，它使我们能够创建更加结构化和更简洁的架构。

阅读

1. [Good article about Dagger 2 in general](http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/)
2. [About custom scopes from Miroslaw](http://frogermcs.github.io/dependency-injection-with-dagger-2-custom-scopes/)
3. [Component dependencies and subcomponents distinctions](http://jellybeanssir.blogspot.ru/2015/05/component-dependency-vs-submodules-in.html)

下一篇文章，我们将研究 Dagger 2 在测试中的用法，以及其他重要且有用的功能。

