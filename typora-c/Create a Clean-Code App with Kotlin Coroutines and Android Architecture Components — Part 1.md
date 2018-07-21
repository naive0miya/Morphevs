# 用 Kotlin Coroutines 和 Android 架构组件创建一个简洁代码应用程序— 第1部分

> 原文 (Medium)：[Create a Clean-Code App with Kotlin Coroutines and Android Architecture Components](https://blog.elpassion.com/create-a-clean-code-app-with-kotlin-coroutines-and-android-architecture-components-f533b04b5431)
>
> 作者：[Marek Langiewicz](https://blog.elpassion.com/@marek.langiewicz?source=user_popover)

[TOC]

> 包括完整演示天气应用程序

安卓系统正在快速发展。 许多开发者和公司都在努力解决一些常见的问题，并创建一些能够完全改变我们应用结构的伟大工具和库。 

我们对新的可能性感到兴奋，但是很难找到时间重写我们的应用程序，从一种新的编程风格中真正受益。 但是如果我们真的开始一个新的项目呢？ 那些突破性的想法中，哪一个可以运用？ 哪些解决方案足够稳定？ 我们是否应该广泛地使用 RxJava，并以反应优先的思维模式来构建我们的应用？ 

>Cycle.js 库 ( by [André Staltz](https://medium.com/@andrestaltz) ) 包含反应优先思维的一个很好的解释: [Cycle.js — Streams](https://cycle.js.org/streams.html).

Rx 是高度可组合的，具有很大的潜力，但与常规的面向对象编程风格截然不同，如果没有 RxJava体验，任何开发人员都将难以理解。

在开始一个新项目之前，还有更多的问题要问。例如：

- 我们应该使用 Kotlin 而不是 Java 吗？ （其实答案很简单：是）
- 我们应该使用实验性的 Kotlin Coroutines 吗？ （同样，这也促进了全新的编程风格）
- 我们是否应该使用 Google 的新实验库： Android 架构组件？

有必要先在一个小应用程序中尝试一下，以便做出明智的决定。[这正是我所做的](https://github.com/elpassion/crweather)，在这个过程中得到了一些有用的见解。如果你想知道我学到了什么，请继续阅读！

## About [The App](https://github.com/elpassion/crweather)

实验的目的是创建一个应用程序，下载用户选择的城市的天气数据，然后用图表（和一些花哨的动画）显示预报 。 这很简单，但它包含了 Android 项目的大部分典型功能。

事实证明，协程和架构组件在一起发挥得非常好，给我们清晰的应用程序架构和良好的分离关注点。 协程允许以自然和简明的方式表达想法。 即使你需要在两者之间进行一些异步调用，如果你要逐行编写你想要的确切逻辑，可执行的函数也是非常好的。

另外：在回调之间没有更多的跳跃。 在这个示例应用程序中，协程也完全消除了使用 RxJava 的需要。 带有可挂起点的函数比一些 RxJava 运算符链更容易阅读和理解 - 那些链可以很快变得过于功能化。 

> 话虽如此，我不认为 RxJava 在每个用例中都可以用协程来代替。 观察者给了我们不同的表现力，无法一对一地映射为可挂起函数。 特别是一旦构造的可观察的运算符链允许许多事件流过它，而每个调用仅可恢复一次可挂起点。

回到我们的天气应用程序：你可以在下面的动作中看到它 - 但要小心，我不是一个设计师。:-)
图表动画显示你可以轻松地通过简单的协程轻松实现它们 - 无需任何 ObjectAnimators，Interpolator，Evaluators，PropertyValuesHolders 等。

[Cr Weather demo](https://www.youtube.com/watch?v=1WUjdyGetkg)

下面显示了最重要的源代码片段。但是，如果你希望看到完整的项目，则可以在 [GitHub](https://github.com/elpassion/crweather) 上找到。

没有太多的代码，应该很容易理解。

我将介绍从网络层开始的应用程序结构。然后，我将转到（几乎）不是 Android 特定的业务逻辑（在 [MainModel.kt](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt) 文件中）。并完成 UI 部分（这显然是 Android 特定的）。

以下是为方便你而添加文本参考编号的一般体系结构图。我将特别关注绿色元素 - suspendable functions 和 actors（一个演员是一个非常有用的协程生成器）。

> 演员模型一般是并发计算的数学模型 - 更多关于我的下一篇博客文章。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froury3nhxj30m80lc0y4.jpg)

## 01 Weather Service

此服务从[开放天气地图](http://openweathermap.org/api) REST API 下载给定城市的天气预报。

我使用来自 [Square](https://github.com/square) 的简单而强大的库，称为 [Retrofit](http://square.github.io/retrofit/)。 我想现在每个 Android 开发人员都知道这一点，但是如果你从未使用它，它是 Android 上最受欢迎的 HTTP 客户端。 它进行网络调用并解析 对 [POJO](https://en.wikipedia.org/wiki/Plain_old_Java_object) 的响应。 这里没什么特别的 - 只是一个典型的 Retrofit 配置。 我插入 [Moshi](https://github.com/square/retrofit/tree/master/retrofit-converters/moshi) 转换器将 JSON 响应转换为数据类。

![](https://ws4.sinaimg.cn/large/006tNc79gy1frous3nug8j30jg0lvq61.jpg)

> [https://github.com/elpassion/crweather/…/OpenWeatherMapApi.kt](https://github.com/elpassion/crweather/…/OpenWeatherMapApi.kt)

这里需要注意的一点是，我将 Retrofit 生成的函数的返回类型设置为：[Call](https://github.com/square/retrofit/blob/master/retrofit/src/main/java/retrofit2/Call.java)。

我使用 Call.enqueue（Callback）来实际调用 Open Weather Map 。我不使用 Retrofit 提供的任何[调用适配器](https://github.com/square/retrofit/tree/master/retrofit-adapters)，我们想要创建一个通用的可挂起函数包裹一个 Call 对象。 

## 02 Utils

这是我们进入（[brave new](https://www.youtube.com/watch?v=_Lvf7Zu4XJU)）协同程序世界的地方：我们想要创建一个包装 [Call](https://github.com/square/retrofit/blob/master/retrofit/src/main/java/retrofit2/Call.java) 对象的通用可挂起函数。

> 我假设你至少知道协程的基本知识。如果不是，请阅读“[协程指南](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md)”的第一章（由 Roman Elizarov 撰写）

它将是一个扩展函数：[suspend fun Call \<T> .await ( )](https://github.com/elpassion/crweather/blob/9c3e3cb803b7e4fffbb010ff085ac56645c9774d/app/src/main/java/com/elpassion/crweather/CommonUtils.kt#L24) 调用 [Call.enqueue ( )](https://github.com/square/retrofit/blob/b3ea768567e9e1fb1ba987bea021dbc0ead4acd4/retrofit/src/main/java/retrofit2/Call.java#L48)（实际上进行网络调用）的，然后暂停并稍后恢复（当响应返回时）。

![](https://ws3.sinaimg.cn/large/006tNc79gy1frous9er0ij30jg0k1acg.jpg)

> [https://github.com/elpassion/crweather/…/CommonUtils.kt](https://github.com/elpassion/crweather/…/CommonUtils.kt)

要将任何异步计算转换为可挂起函数，我们使用 Kotlin 标准库中的 [suspendCoroutine](https://github.com/JetBrains/kotlin/blob/8f452ed0467e1239a7639b7ead3fb7bc5c1c4a52/libraries/stdlib/src/kotlin/coroutines/experimental/CoroutinesLibrary.kt#L89) 函数。它给了我们一个 [Continuation](https://github.com/JetBrains/kotlin/blob/8fa8ba70558cfd610d91b1c6ba55c37967ac35c5/libraries/stdlib/src/kotlin/coroutines/experimental/Coroutines.kt#L23) 对象，它是一种通用的回调。 只要我们希望我们的新的可挂起的函数恢复（通常是抛出一个异常），我们只需要调用它的 [resume](https://github.com/JetBrains/kotlin/blob/8fa8ba70558cfd610d91b1c6ba55c37967ac35c5/libraries/stdlib/src/kotlin/coroutines/experimental/Coroutines.kt#L32) 方法（或 [resumeWithException](https://github.com/JetBrains/kotlin/blob/8fa8ba70558cfd610d91b1c6ba55c37967ac35c5/libraries/stdlib/src/kotlin/coroutines/experimental/Coroutines.kt#L38)）。

下一步将是使用我们新的可挂起的 Call \<T> .await ( ) 函数将 Retrofit 生成的异步函数转换为方便的可挂起函数。

## 03 Repository

存储库对象是我们的应用程序中显示的数据（[图表](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/DataTypes.kt)）的来源。

![](https://ws3.sinaimg.cn/large/006tNc79gy1frousm5r4ej30jg0kidiu.jpg)

> [https://github.com/elpassion/crweather/…/Repository.kt](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/Repository.kt)

在这里，我们有一些私人挂起功能，通过应用我们 suspend fun Call \<T>.await( ) 扩展到天气服务功能。 通过这种方式，所有人都可以使用 Forecast  数据，而不是用 Call \<Forecast> 。 然后我们在我们的一个公共的可挂起的函数中使用它： suspend fun getCityCharts（city：String）：List \<Chart>。 它将来自 api 的数据转换成可以显示图表列表。 我使用 List \<DailyForecast> 上的一些自定义扩展属性将数据实际转换为 List \<Chart> 。 重要说明：只有可挂起的函数才能调用其他可挂起的函数

这里的硬编码是为了简单。如果你想测试应用程序，请在这里生成新的 [appid](http://openweathermap.org/appid) - 如果太多的人太频繁地使用这个硬编码的应用程序将被自动阻塞24小时。

在下一步中，我们将创建主应用程序模型（实现 Android [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 体系结构组件），该模型使用 actor（协程构建器）来实现应用程序逻辑。

## 04 Model

在这个应用程序中，我们只有一个简单的模型： [MainModel](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt) : [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html)，我们的一个活动（[MainActivity](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainActivity.kt)）使用的模型。

![](https://ws1.sinaimg.cn/large/006tNc79gy1frousrft28j30jg0mdq6o.jpg)

> [https://github.com/elpassion/crweather/…/MainModel.kt](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt)

这个类代表了应用程序本身。 它会通过我们的活动（实际上是由 Android 系统 [ViewModelProvider](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProvider.html)）实例化，但是它将在配置变化（比如屏幕旋转）后存活 - 新的活动实例将获得相同的模型实例。 我们不必担心这里的活动生命周期。们没有实现所有这些活动生命周期相关的方法(onCreate，onDestroy，...) ，我们只有一个 onCleared ( ) 方法，当用户退出应用程序时调用。 

> 在活动结束时调用清理的方法。

尽管我们不再与活动生命周期紧密耦合，但我们仍然需要以某种方式发布我们的应用程序模型的当前状态，以在某处（在活动中）显示它。这是 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 的作用。

LiveData 就像 [RxJava](https://github.com/ReactiveX/RxJava) [BehaviorSubject](https://github.com/ReactiveX/RxJava/wiki/Subject) 再次被重新创建...它拥有一个可观察的可变值。最重要的区别是我们如何订阅它，我们稍后会在 [MainActivity](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainActivity.kt) 中看到它。

>另外 LiveData 没有所有那些强大的可组合的操作符 Observable。 只有一些简单的[转换](https://developer.android.com/reference/android/arch/lifecycle/Transformations.html)。
>
>另一个区别是 LiveData 是特定于 Android 的，而 RxJava 的 Sunject 不是，因此可以使用常规的非 Android JUnit 测试轻松进行测试。
>
>还有一个区别是，LiveData 是 “生命周期感知” - 更多关于我的下一篇文章，我介绍 [MainActivity](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainActivity.kt) 类。

在这里，我们实际上使用了 [MutableLiveData](https://developer.android.com/reference/android/arch/lifecycle/MutableLiveData.html) : [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 对象，它允许自由地将新值推入。 应用程序状态由四个 LiveData 对象表示：城市，图表，加载和消息。 其中最重要的是图表： LiveData \<List \<Chart >> 对象，表示当前要显示的图表列表。

所有更改应用程序状态和对用户操作作出反应的工作都由 ACTOR 执行。 actor 很棒，将在我的下一篇博客文章解释:-)

## Summary

我们已经为我们的主要 actor 准备好了一切。 如果你看 actor 代码本身 - 你可以（看）知道它是如何工作的，即使不知道协程或 actor 理论。 即使它只有几行，它实际上包含了这个应用程序的所有重要的业务逻辑。 魔法是我们称之为可挂起函数的地方(标记为灰色箭头和绿线)。 一个可挂起点是对用户操作的迭代，其次是网络调用。 感谢协程，它看起来像同步阻塞代码，但它根本不会阻塞线程。

