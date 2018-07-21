# 用 Kotlin Coroutines 和 Android 架构组件创建一个简洁代码应用程序— 第3部分

> 原文 (Medium)：[Create a Clean-Code App with Kotlin Coroutines and Android Architecture Components — Part 3](https://blog.elpassion.com/create-a-clean-code-app-with-kotlin-coroutines-and-android-architecture-components-part-3-f3f3850acbe6)
>
> 作者：[Marek Langiewicz](https://blog.elpassion.com/@marek.langiewicz?source=post_header_lockup)

[TOC]

我们已经有一个工作的应用程序实现为 [MainModel](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt) 类。现在我们需要为它创建 UI。这部分将是 Android 特定的。

> 不幸的是，[MainModel](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt) 已经有点特定于 Android 了，因为它使用 [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) 的 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html)（和 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html)）

## 06 View

MainActivity 类表示一个观察我们模型的视图

![](https://ws3.sinaimg.cn/large/006tNc79gy1froutyu49yj30jg0qqdjy.jpg)

> [https://github.com/elpassion/crweather/…../MainActivity.kt](https://github.com/elpassion/crweather/…../MainActivity.kt)

感谢 [Android 架构组件](https://developer.android.com/topic/libraries/architecture/index.html)，我们的活动非常小。我们不需要重写像 [onStart](https://developer.android.com/reference/android/app/Activity.html#onStart%28%29), [onResume](https://developer.android.com/reference/android/app/Activity.html#onResume%28%29), [onPause](https://developer.android.com/reference/android/app/Activity.html#onPause%28%29), [onStop](https://developer.android.com/reference/android/app/Activity.html#onStop%28%29) 等等的生命周期方法...我们只需要 [onCreate](https://developer.android.com/reference/android/app/Activity.html#onCreate%28android.os.Bundle%29) 来设置。

在 onCreate 方法中，我们设置了内容视图，操作栏，选择抽屉和回收视图的适配器。是一般的安卓系统的东西。 除此之外，我们还做了两件事：

- 我们在城市导航视图中设置了一个点击侦听器，因此它调用我们模型的 [SelectCity](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/DataTypes.kt#L23) 操作（第25行）
-  我们初始化模型本身（第28行）

让我们仔细看看模型初始化：

第一行（32）从 [ViewModelProvider](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProvider.html) 获取 MainModel 实例，ViewModelProvider 又由 [ViewModelProviders.of (Activity)](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProviders.html#of%28android.support.v4.app.FragmentActivity%29) 工厂方法提供。 这里的 Android Architecture Components 库有点像 [Dagger](https://google.github.io/dagger/)。 它为我们管理任何 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 实例，并确保在屏幕旋转（或任何其他设备配置更改）后获得相同的模型实例。 这样，新的 MainActivity 实例可以访问相同的模型实例，“订阅”（或者说“[观察](https://developer.android.com/reference/android/arch/lifecycle/LiveData.html#observe%28android.arch.lifecycle.LifecycleOwner,%20android.arch.lifecycle.Observer%3CT%3E%29)”）相同的应用程序状态 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 对象，并在状态改变时显示适当的UI。

> 在 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 类 doc 的开头有一个很好的关于这个 “依赖注入” 机制的详细描述。

现在，当我们有我们的模型实例时，我们可以观察它。 [LiveData.observe](https://developer.android.com/reference/android/arch/lifecycle/LiveData.html#observe%28android.arch.lifecycle.LifecycleOwner,%20android.arch.lifecycle.Observer%3CT%3E%29) 方法将 LifecycleOwner 作为第一个参数，所以 LiveData 知道我们的活动何时准备好接收状态更新以及何时销毁。 LiveData 类有一些保证，可以简化我们观察它的方式：

- 所有的状态变化都在主线程上调度。 这意味着我们可以立即访问 android 视图并更新显示的数据。
- 仅当观察者处于活动状态时才会分派变更：其[生命周期](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.html)处于 [STARTED](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.State.html#STARTED) 或 [RESUMED](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.State.html#RESUMED) 状态。 这意味着当我们的活动不可见时，我们不会收到任何不必要的更新。
- 新的观察者一旦变为活动状态就立即收到当前值。 这意味着只要我们的活动变得可见，我们总会有东西显示。
- 当观察者 [DESTROYED](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.State.html#DESTROYED) 时，LiveData 将其从观察者列表中移除。 这意味着我们不必记住 “取消订阅”，因为它会自动完成。
- 所有这些与生命周期相关的事件的顺序都经过精心设计，所以我们永远不会得到臭名昭着的碎片事务异常 如果你想了解更多的细节，请观看这个 [YouTube 演示（从16:30开始）](https://youtu.be/bEKNi1JOrNs?t=16m30s)的几分钟。

在 MainActivity 中，我们只需设置 LiveData 观察者来调用我们的 display_ 函数：

displayLoading 显示/隐藏进度条。
displayCity 在导航视图中显示给定的城市。
displayCharts 在回收站视图中显示给出的图表列表（带有一些动画） - 稍后我们将对其进行分析。
displayMessage 显示给定的（通常是错误）消息（为简单起见，使用一个 toast）。

不幸的是，LiveData（以及 ViewModel）是特定于 Android 的，所以它使测试变得复杂。 但是，通过使用 [RxJava](https://github.com/ReactiveX/RxJava) 实现 LiveData 的 “生命周期感知” 功能，我们可以实现更好的问题分离。 这需要：一些特殊的 Observable \<LifecycleEvent> 来表示生命周期的变化; 一些 [BehaviorSubject](https://github.com/ReactiveX/RxJava/wiki/Subject)（用于保存数据并返回订阅上的最后一个已知值）; switchMap 操作符（等待观察者处于活动状态）; takeUntil 操作符（在适当的 LifecycleEvent 中自动取消订阅）; 或者更多。 我们也可以使用一个库，比如 [RxLifecycle](https://github.com/trello/RxLifecycle)。 在 [El Passion](https://www.elpassion.com/) 中，我们已经在一个 hackathon 项目（[Teamcity Android Client](https://github.com/elpassion/teamcity-android-client)）中用 RxLifecycle 尝试了这个解决方案，并且效果很好。

> 我想，实现 LiveData 给我们的所有保证（比如：没有片段事务异常）可能会非常棘手。

要显示图表列表，我们使用非常基本的适配器设置的常规回收站视图。 有趣的部分是我们如何呈现一个单一的项目（[图表](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/DataTypes.kt#L6)）。 我们实现了自定义视图类：基类 [ChartView](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/ChartView.kt)：[CrView](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/CrView.kt)。 这两个类使用可挂起的函数和 actor 来实现图表动画。 让我们仔细看看它。

## 07 CrView

在 Android 中重绘用户界面（与其他 UI 框架一样）是异步的。 我们调用 [View.invalidate ( )](https://developer.android.com/reference/android/view/View.html#invalidate%28%29)让系统知道我们想重画这个视图，然后系统决定何时调用 [View.onDraw (Canvas)](https://developer.android.com/reference/android/view/View.html#onDraw%28android.graphics.Canvas%29) 函数来重绘视图。 但是如果我们能假装它是同步的呢？ 然后，然后我们可以使用普通的控制流构造， 比如循环， 然后重新绘制循环中每个动画帧的视图(在这两者之间有一些延迟也假装是同步的)。 我们已经知道如何实现这样的函数(假装是同步的) : 暂停函数。 

![](https://ws3.sinaimg.cn/large/006tNc79gy1frouu6lwyyj30jg0fawgc.jpg)

> [https://github.com/elpassion/crweather/…../CrView.kt](https://github.com/elpassion/crweather/…../CrView.kt)

我们在这里所做的第一件事（第8,11,28行）是允许用户在每次系统调用 [onDraw](https://developer.android.com/reference/android/view/View.html#onDraw%28android.graphics.Canvas%29) 时设置一些在 [Canvas](https://developer.android.com/reference/android/graphics/Canvas.html) 上执行的代码块。

接下来，我们创建 suspend fun redraw ( )  实现其实很简单。 我们所要做的就是让系统知道我们想要重绘（第17行），然后挂起并保存在某个地方（第18行）。稍后，当系统调用 onDraw 时，我们会检查是否有一些重绘调用暂停，如果是这样的话恢复它(第29行)。 我们以异步方式重新开始，所以 draw 总是快速结束。 

基本上就是这样。 我还添加了第二个可挂起的函数: 绘制，以显示使用这个类的另一种可能的方法，但是现在我并没有使用它。 

现在，让我们使用 CrView 作为基类来实现可以显示和动画天气图的自定义视图。

## 08 ChartView

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouuayy1oj30jg0gojtl.jpg)

> [https://github.com/elpassion/crweather/…../ChartView.kt](https://github.com/elpassion/crweather/…../ChartView.kt)

我们希望有一个公共可变属性 var chart：带有自定义 setter 的 Chart，它将导致新的图表出现（但不是立即，它应该平滑地从当前图表变成新的图表 ）。 动画所需的计算和绘图本身由 actor 处理，因此图表制定者只将所有新图表发送到 actor 的邮箱。 我们使用不可挂起的操作：提供（第11行）向 actor 发送新的图表。 这将覆盖 actor 的邮箱中的任何未经处理的图表使用新的。

现在，我们来看一下 actor 的实现。

## 09 Second Actor

我之前在博客中已经看到过一位演员：

这个也是与 actor 协同构建器实现的，它使用相同的协程调度器：UI，以及相同的通道类型：Channel.CONFLATED。

这个想法是保持 currentChart 在一个局部变量（和表示每个图表点动画的速度和方向的 currentVelocities）; 然后使用一个普通的 while 循环，在这个循环中我们将 currentChart 点向最终位置（destinationChart）移动一点点，重绘视图，然后延迟几毫秒，然后重复。 while 循环负责将 currentChart 动画到用户所需的 destinationChart。 这个动画循环必须放置在另一个 for 循环中，该循环遍历 actor 的邮箱，并检索用户提供的下一个destinationCharts。

我们在这里有三个暂停点：

从邮箱接收下一个图表（第21行）。 

在每个动画帧的内部 while 循环中重绘视图（第30行）。 

在每个动画帧之后暂停16毫秒（第31行）。

我将跳过最枯燥的实现细节（可以在 [GitHub / CrWeather](https://github.com/elpassion/crweather) 上查看），除了几个：

- 在开始的时候（第19行），我们设置了 CrView，以便在每次系统要重绘视图时都绘制 currentChart。
- 外循环迭代（第21行）几乎不会挂起，因为只有在邮箱中已有新图表时，内循环才能完成。
- 我们使用相同的数据类：Chart 来记住 currentChart 点的速度 - 这里的每个 Point 代表一个速度矢量。 为了保持代码简短，我重复使用了 Chart 类 - 为图表点速度提供单独的数据结构会更好。
- 获得新的目标图表后，我们复制并重新设置 currentChart（第23行），使其包含与新目标图表相同的点数。
- 目前的情况也是如此（第25行）
- moveABitTo 扩展函数实际上将 currentChart 移向 destinationChart，也会稍微改变速度，所以动画会在一段时间后保持稳定。 检查 [ChartsUtils.kt](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/ChartsUtils.kt) 文件的实现细节。

但是，如果系统放弃特定的 ChartView 实例会发生什么？actor 是否会无限期地计算新的动画帧？ 它不会，这是为什么：

重绘函数调用 View.invalidate ( ) 并挂起，但是系统不会再对这个 View 对象调用 onDraw。这意味着我们的继续将不会恢复。因此，如果系统没有对 ChartView 的实时引用，那么对可挂起点（Continuation）的引用也会丢失，所以它们都可以与 actor 本身一起被垃圾收集。一般来说，当暂停的协程被遗忘时（所以没有人会继续提到），它会被正确的垃圾收集。

感谢 Kotlin 协同程序，我们可以用非常自然的方式表达动画的逻辑和接收新的“变形目标”的逻辑。代码显示了我们如何看待这个问题 - 我们认为：“对于每个新图表而言...（而不是新图表）：将图表变形一点，重绘当前图表，等待一点点，重复”，这正是实现看起来像。在特定角色的完全控制之下，我们可以把更多的状态保持为局部变量（比如 currentChart 和 currentVelocities），这是很好的 - 每个协程都按顺序运行。

> 请注意，我们从 CrView.onDraw 外部访问的 actor 代码中改变状态。一般来说，并发计算的 actor 模型禁止任何共享状态，以支持通过 actor 的邮箱实现所有的通道。

我认为这种风格比传统的面向对象设计更简单和更多的组合 - 将它与 Android [Property Animation](https://developer.android.com/guide/topics/graphics/prop-animation.html) 系统中的所有类进行比较。 尽管所有这些 [ValueAnimators](https://developer.android.com/reference/android/animation/ValueAnimator.html)，[ObjectAnimators](https://developer.android.com/reference/android/animation/ObjectAnimator.html)，[AnimatorSets](https://developer.android.com/reference/android/animation/AnimatorSet.html)，不同的[评估者](https://developer.android.com/reference/android/animation/TypeEvaluator.html)和[插值器](https://developer.android.com/reference/android/animation/TimeInterpolator.html)都允许创建非常复杂的动画; 实现一个类似的可挂起函数系统会更加自然，不需要太多的样板代码。

## 10 Conclusions

Kotlin Coroutines 可以和 RxJava 成功竞争，但是与 RxJava 的配合也非常好。

在我看来，我们不应该把一个可挂起的函数看作是一个具有一些附加功能的函数。 这个概念有可能极大地改变我们在这个越来越异步的世界中创建我们的软件的方式。 

我们在这里测试的第二项技术-- Android 体系结构组件 -虽然没有 Kotlin Coroutines 那么重要，但它仍然非常有用。 如果你想要一些(易于使用的)可观察流，LiveData 类可以提供帮助(如果你不想包含 RxJava) ; 如果你想要一个简单的"依赖注入"(如果你想停止担心活动生命周期和设备配置的更改) ，那么 LiveData 类将会很有帮助。 

Android 上的软件开发变得越来越精彩，这要感谢令人惊叹的库，工具和（最重要的）Kotlin :-)
如果你对未来感到疑惑，我认为 JetBrains 公司的 [Svetlana Isakova](https://github.com/svtk) 对你有两个正确的答案：[Kotlin 的未来:-](https://youtu.be/XEgibiHdJtQ?t=46m11s))

> 你也可以去看看  Andrey Breslav 的帖子：[Kotlin Future Features 调查结果](https://blog.jetbrains.com/kotlin/2017/06/kotlin-future-features-survey-results/)





