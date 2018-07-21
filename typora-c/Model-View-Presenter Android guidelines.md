# 模型-视图-演示者: Android 指南

> 原文 (Medium)：[Model-View-Presenter: Android guidelines](https://medium.com/@cervonefrancesco/model-view-presenter-android-guidelines-94970b430ddf)
>
> 作者：[Francesco Cervone](https://medium.com/@cervonefrancesco?source=post_header_lockup)

[TOC]

有很多关于 MVP 架构的文章和例子，有很多不同的实现。 开发者社区一直在努力以最好的方式将这种模式应用到 Android 上。 

如果你决定采用这种模式，那么你就是在做一个架构的选择，你必须知道你的代码库将会改变，以及你接近新特性的方法(为了更好的目标)。 你必须知道你需要面对一些常见的 Android 问题，比如 Activity 的生命周期，你可能会问自己一些问题，比如: 

- 我应该保存演示者的状态吗？
- 我应该保持演示者吗？
- 演示者应该有一个生命周期？

在本文中，我将列出一系列指南或最佳实践，以便：

- 使用这种模式来解决最常见的问题(或者至少是我个人经历中的问题) 。 
- 最大化这种模式的好处 。

首先，让我们来描述一下示例：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frov7rbxsmj30f901yjra.jpg)

- 模型：它是一个负责管理数据的接口。 模型的职责包括使用 API，缓存数据，管理数据库等。该模型还可以是一个与负责这些责任的其他模块进行通信的接口。 例如，如果您使用的[存储库模式](https://martinfowler.com/eaaCatalog/repository.html)，模型可以是一个存储库。如果您使用的是 [简洁架构](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)，则该模型可能是交互器。
- 演示者：演示者是模型和视图之间的中间人。所有的演示逻辑属于它。 演示者负责查询模型并更新视图，对用户交互做出反应，更新模型。
- 视图：它只负责以演示者决定的方式呈现数据。 该视图可以通过活动、片段、任何安卓小部件或任何可以执行操作的任何东西来实现，如显示 ProgressBar，更新 TextView，填充 RecyclerView 等等。

## 1. 让视图变得愚蠢和被动

安卓系统最大的问题之一是视图(Activities，Fragments，...)不容易测试，因为框架的复杂性。 为了解决这个问题，您应该实现被动[视图模式](https://martinfowler.com/eaaDev/PassiveScreen.html)。 这种模式的实现通过使用控制器将视图的行为减少到绝对最小值，在我们的例子中，就是演示者。这种选择极大地提高了可测性。 

例如，如果你有一个用户名 / 密码表单和一个"提交"按钮，你不会在视图中写入验证逻辑，而是在演示者内部。 您的视图只需要收集用户名和密码，然后发送给演示者。 

## 2. 使演示者独立于安卓框架

为了使前一个原则真正有效(提高可测性) ，请确保演示者不依赖于 Android 类。 仅使用 Java 依赖关系编写演示者有两个原因: 首先，你从实现细节(Android 框架)中抽象演示者，因此，您可以为演示者编写非工具化测试(即使没有机器密码) ，在本地 JVM 上运行测试速度更快，而且不需要模拟器。 

> 如果我需要上下文怎么办？ 

那么，摆脱它。 在这样的情况下，你应该问自己为什么需要上下文。 例如，您可能需要上下文来访问共享首选项或资源。 但是，您不应该在演示者中这样做：你应该在模型中访问视图中的资源和首选项。 这只是两个简单的例子，但我敢打赌，大多数情况下，这只是一个错误责任的问题。 

顺便说一下，在这种情况下，当你需要对一个对象进行解耦时，这个依赖倒置原则会帮助很多。 

## 3. 写一份合同来描述视图和演示者之间的交互

你要写一个新的特性时，在第一步就写一份合同是一种很好的做法。 合同描述了视图和演示者之间的交流，它帮助您以一种更简洁的方式设计交互。 

我喜欢使用谷歌在 [Android 架构库](https://github.com/googlesamples/android-architecture)中提出的解决方案: 它由两个内部接口组成，一个用于视图，另一个用于演示者。 

我们来举个例子。 

```java
public interface SearchRepositoriesContract {
  interface View {
    void addResults(List<Repository> repos);
    void clearResults();
    void showContentLoading();
    void hideContentLoading();
    void showListLoading();
    void hideListLoading();
    void showContentError();
    void hideContentError();
    void showListError();
    void showEmptyResultsView();
    void hideEmptyResultsView();
  }
  interface Presenter extends BasePresenter<View> {
    void load();
    void loadMore();
    void queryChanged(String query);
    void repositoryClick(Repository repo);
  }
}
```

>  一个用于搜索和显示 GitHub 存储库的 MVP 合同 

仅仅阅读方法名称，你应该能够理解我在这个合同中描述的用例。 

如果你做不到，我就惨败了。 

从示例中可以看出，视图方法非常简单，表明除了 UI 之外没有任何逻辑。 

#### 视图合同

就像我之前说过的那样，视图是由一个 Activity（或者一个 Fragment）来实现的。 演示者必须依赖于View接口，而不是直接依赖于 Activity：通过这种方式，您将演示者从视图实现(然后从 Android 平台)中解耦，同时遵循 SOLID 原则，"依赖于抽象。 不要依赖于具体实现"。 

我们可以在不改变演示者代码行的情况下替换具体视图。 此外，我们可以通过创建一个模拟视图来轻松地对演示者进行单元测试。 

#### 演示者合同

> 等等。我们是否真的需要一个演示者接口？

[实际上没有](http://blog.karumi.com/interfaces-for-presenters-in-mvp-are-a-waste-of-time/)，但我会说是的。 

关于这个话题有两种不同的看法。 有些人认为你应该编写演示者接口，因为你将具体视图从具体的演示者中分离出来。 

然而，一些开发人员认为你正在抽象一些已经是抽象的(视图) ，你不需要编写一个接口。 此外，你可能永远不会写一个替代的演示者，那么这将是浪费时间和代码行。 

无论如何，拥有一个接口可以帮助你编写一个模拟的演示者，但是如果你使用 [Mockito](http://site.mockito.org/) 这样的工具，你就不需要任何接口了。 

就我个人而言，我更喜欢编写演示者接口有两个简单的原因（除了之前列出的）：

- 我不是为演示者写一个接口。我正在撰写描述 View 和 Presenter 之间交互的合同。在同一个文件中同时拥有“合同当事人”的规则对我来说听起来非常干净。 
- 这不是一个真正的努力。

我写这样的合同已经有一年了，我从来没有把这看成是一个问题。

## 4. 定义命名规则来分离责任

演示者通常可以有两类方法：

- Actions （例如 load ( )）：它们描述了演示者的功能
- User events（例如 queryChanged ( )）：它们是由用户触发的操作，如 “在搜索视图中输入” 或 “点击列表项” 。

你有更多的行动，更多的逻辑将在视图中。 相反，用户活动建议他们把做什么的决定留给演示者。 例如，只有当用户键入至少一个固定数量的字符时，才能启动搜索。 在这种情况下，视图只是调用 queryChanged (...)方法，演示者将决定何时应用此逻辑启动新的搜索。 

相反，当用户滚动到列表的末尾时，就会调用 loadMore ( ) 方法，然后再加载一页结果。 这个选择意味着当用户滚动到最后，视图知道一个新的页面必须加载。 为了"逆转"这个逻辑，我可以在 onScrolledToEnd ( ) 中命名这个方法让具体的演示者决定做什么。 

我的意思是，在"合同设计"阶段，您必须为每个用户事件决定相应的操作和逻辑应该属于谁。 

## 5. 不要在"演示者"接口中创建活动生命周期样式回调

这个标题，我的意思是演示者不应该有 onCreate ( )，onStart ( )，onResume ( ) 以及他们的双重方法，原因有几个: 

-  通过这种方式，演示者将特别与活动生命周期相耦合。 如果我想用一个片段替换活动呢？ 我应该在什么时候调用 presenter.onCreate (state ) 方法？ 在片段的 onCreate (...) ，onCreateView (...) 或 onViewCreated (...)？ 如果我使用自定义视图会怎么样？ 
- 演示者不应该有这么复杂的生命周期。 事实上，主要的安卓组件是这样设计的，并不意味着你必须在任何地方反映这种行为。 如果你有机会简化，那就去做吧 。

在活动生命周期回调中，您可以调用演示者的动作，而不是调用同名的方法。例如，你可以在 Activity.onCreate (...) 结束时调用 load ( )。 

## 6. 演示者与视图有1对1的关系

如果没有视图，演讲者就没有意义。 它伴随着视图，当视图被破坏时，它就会消失。 它一次只管理一个视图。 

您可以以多种方式处理演示者中的视图依赖性。 一个解决方案是提供一些方法，如在演示者接口中 attach(View view) 和 detach ( ) ，就像之前的示例一样。 这个实现的问题是视图是可以为空的，那么您必须在每次演示者需要它时添加一个 null-check。 这可能有点无聊。 

我说，视图和演示者之间有一个1对1的关系。 我们可以利用这一点。 实际上，具体的演示器可以将视图实例作为构造函数参数。 顺便说一下，你可能需要一种方法来订阅某些事件的演示者。 因此，我建议定义一个方法 start ( )  (或类似的东西)来运行演示器的业务。 

> 什么是 detach ( )？

如果你有一个方法 start  ( )，你可能至少需要一个方法来释放依赖关系。既然我们调用了让演示者订阅一些事件的方法 start  ( )，我将称之为 stop  ( )。

```java
// BasePresenterAttach.java
public interface BasePresenter<V> {
  void attach(V view);
  void detach();
}
```

```java
// BasePresenterStart.java 
public interface BasePresesnter {
  void start();
  void stop();
}
```

## 7. 不要在演示者内部保存状态

我的意思是使用一个 Bundle。如果您想尊重第2点，则不能这样做。不能将数据序列化到 Bundle 中，因为演示者将与 Android 类耦合。 

我不是说演示者应该是无状态的，因为我在说谎。 在我之前描述的用例中，例如，演示者至少应该有某处的页码/偏移量。

> 所以，你必须保留演示者，对吧？

## 8. 不. 不要保留演示者

我不喜欢这个解决方案，主要是因为我认为这个演示者不是我们应该保持的东西，它不是一个数据类。 

一些提案提供了一种在使用保留的片段或[加载器](https://medium.com/@czyrux/presenter-surviving-orientation-changes-with-loaders-6da6d86ffbbf#.ii7px6adf)进行配置更改期间保留演示者的方式。 除了个人的考虑，我不认为这是最好的解决方案。有了这个技巧，演示者就能够在配置更改中存活下拉，但是当安卓杀死了这个过程并摧毁了活动时，后者将与一个新的演示者一起重新创建。 因此，这个解决方案只能解决问题的一半。 

## 9. 为模型提供缓存以恢复视图状态

在我看来，解决"还原状态"问题需要对应用架构进行一些调整。 [这篇文章](https://medium.com/@theMikhail/presenters-are-not-for-persisting-f537a2cc7962#.ssl022wg7)提出了一个符合这种想法的伟大解决方案。 基本上，作者建议使用类似于存储库或任何目的来管理数据的接口来缓存网络结果，从而对应用程序而不是活动进行处理(以便它能够在定向变化中存活)。 

这个接口只是一个更聪明的模型。 后者应该至少提供一个磁盘缓存策略和可能的内存缓存。 因此，即使进程被销毁，演示者也可以使用磁盘缓存还原视图状态。 

视图只应该涉及恢复状态的任何必要的请求参数。 例如，在我们的例子中，我们只需要保存查询。 

现在，你有两个选择：

- 您在模型层中抽象出这种行为，以便在演示者调用 repository.get（params）时，如果页面已经在缓存中，则数据源只返回它，否则调用 API
- 您在演示者内部管理这个内容，在合同中添加另一个方法来恢复视图状态。 reset（params），loadFromCache（params）或 overload（params）是不同的名称，描述相同的行为，你选择。

## 结论

我想我们结束了。 就是这样。 这是我对模型视图演示者应用于 Android 的知识。 我试着从我的错误、别人的经历和经历中吸取教训。 我一直在寻找最好的方法。 

我希望你喜欢这篇文章。 如果你有任何反馈，请在下面评论: 我非常乐意得到建议和看到任何不同的解决方案。 

