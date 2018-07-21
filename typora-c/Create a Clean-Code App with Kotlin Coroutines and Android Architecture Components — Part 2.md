# 用 Kotlin Coroutines 和 Android 架构组件创建一个简洁代码应用程序 — 第2部分

> 原文 (Medium)：[Create a Clean-Code App with Kotlin Coroutines and Android Architecture Components — Part 2](https://blog.elpassion.com/create-a-clean-code-app-with-kotlin-coroutines-and-android-architecture-components-part-2-4f585050d7d7)
>
> 作者：[Marek Langiewicz](https://blog.elpassion.com/@marek.langiewicz?source=post_header_lockup)

[TOC]

这是关于在一个简单的天气应用程序中使用 Kotlin Coroutines 和 Android 架构组件的第二篇博客文章。

我们的应用程序中最重要的部分是 [MainModel](https://github.com/elpassion/crweather/blob/master/app/src/main/java/com/elpassion/crweather/MainModel.kt)。在这里，我们有一个 actor 决定何时从网络请求一些数据，何时使用缓存数据，何时推送新状态（我们的活动就可以观察到 ）。

为了理解它是如何工作的，首先让我们深入一下 actor 的想法。

## 05 Actor

关于 actor 并发计算作为通用原语有一个完整的理论。 你可以在 [Wikipedia](https://en.wikipedia.org/wiki/Actor_model) 上阅读更多关于它的信息。

基本上，actor 是一种能够根据他所等待的信息顺序地做一些工作。这些信息通过"邮箱"发送，在 Kotlin 将其作为一个通道实现。 

那么让我们先来看看通道。 

### 05.1 Channels Basics

通道的概念类似于一个[阻塞队列](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html)，但是它具有挂起操作而不是阻塞操作，并且它可以被关闭。 我们可以发送消息给它并接收来自它的消息。 这两种操作都可能导致暂停。 当通道满时，发送操作可以暂停(当某人在此通道上调用接收时将恢复该操作)。 当通道为空时，接收操作可以暂停(当某人在此通道上调用发送时将恢复该操作)。 

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouszc4kzj30dh0asjru.jpg)

在 Kotlin 中，Channel \<E> 接口扩展了两个：

- SendChannel \<E> 基本接口 它定义了 suspend fun send (element：E)方法 
- ReceiveChannel \<E> 基本接口 它定义了 suspend fun receive ( )：E 方法

这些基本接口还包含其他不那么重要的（而不是可挂起的）方法：

- SendChannel.offer（element：E）：Boolean

如果可能，向通道添加元素，如果通道已满，则返回 false

- SendChannel.close（cause：Throwable？= null）

关闭通道

- ReceiveChannel.poll（）：E？

如果可用，则返回一个元素；如果通道为空，则返回 null

- 还有一些其它（不太重要的）方法

### Channels’ Types

[RendezvousChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/RendezvousChannel.kt) - 它不包含任何内部缓冲区

![](https://ws3.sinaimg.cn/large/006tNc79gy1frout2kqczj305d043wea.jpg)

- 每个发送调用都被暂停，直到有人调用接收（除非有一些接收操作已经被挂起）
- 每个接收调用都被暂停，直到其他人调用发送（除非有一些发送操作已经挂起）

[ArrayChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/ArrayChannel.kt)  - 它包含一个固定大小的缓冲区

![](https://ws2.sinaimg.cn/large/006tNc79gy1frout59t6lj30dp04f3yc.jpg)

- 只有在缓冲区已满的情况下才会暂停发送 
- 只有在缓冲区为空的情况下才会暂停接收

[LinkedListChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/LinkedListChannel.kt) —它包含无限容量的链接列表缓冲区

![](https://ws1.sinaimg.cn/large/006tNc79gy1frout7nhy0j30it04ea9x.jpg)

- 发送从不被暂停 - 但它可能会抛出 [OutOfMemoryError](https://docs.oracle.com/javase/7/docs/api/java/lang/OutOfMemoryError.html) （好吧，如果内存不足，所有东西都可以扔掉） 
- 缓冲区为空时接收暂停

[ConflatedChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/ConflatedChannel.kt)  — 它最多可以缓冲一个元素，并将所有后续的发送调用合并在一起

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouta0n8ij3078049mwz.jpg)

- 发送永远不会被挂起，但新元素会覆盖等待接收的任何旧元素 
- 缓冲区为空时接收暂停

还有其他的类型，但我们会跳过它们。如果你想了解更多信息，请查看“[协程指南](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md#channels)”。

我们将使用 [ConflatedChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/ConflatedChannel.kt) 作为我们的 actor 的通道。

### 05.2 Actors Basics

在很多情况下，有一个带有附加通道的协同程序可以向系统的其他部分发送或接收一些元素。 通常，这个协程只发送元素到通道，这些元素被系统的其他部分接收（所以我们的协程是一个 “生产者”）。 或者相反：我们的协同程序只接收来自一个通道的元素，而其他人则通过这个通道向我们发送元素（所以我们的协程是消费者，或者说是一个“actor”）。

为了实现这样的场景，我们可以创建一个可挂起的函数，将这个通道作为参数，并在代码中使用它。 然后，我们将实例化适当的通道，并使用一些协程生成器，通过创建的通道调用我们的可挂起的函数。如果这听起来有点太多了，[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 库中有两个特殊的协程生成器，用于两种最常见的情况（当我们将通道用作 “邮箱” 时）：

- [produce](https://github.com/Kotlin/kotlinx.coroutines/blob/ecda27fa01602e42922eadc16ef34b920b8006bb/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Produce.kt#L89) 创建一个与协程本身用来发送元素的附加通道的协程。从这个协程返回的对象实现了 ReceiveChannel 接口，所以用户可以使用这个协程生成的元素。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froutd8ofwj30b306sglm.jpg)

- [actor](https://github.com/Kotlin/kotlinx.coroutines/blob/ecda27fa01602e42922eadc16ef34b920b8006bb/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Actor.kt#L82) — 用协程本身用来接收元素的连接通道创建协程。从这个协程返回的对象实现了 SendChannel 接口，所以用户可以发送元素到这个协程。

![](https://ws2.sinaimg.cn/large/006tNc79gy1froutfllsxj30an06vaa1.jpg)

[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 中的 [actor](https://github.com/Kotlin/kotlinx.coroutines/blob/ecda27fa01602e42922eadc16ef34b920b8006bb/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Actor.kt#L82) 协程生成器实现如下所示：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frouti8g7oj30e1075gm6.jpg)

> [https://github.com/Kotlin/kotlinx.coroutines/...../Actor.kt#L82](https://github.com/Kotlin/kotlinx.coroutines/...../Actor.kt#L82)

我们来分析一下它的参数：

context - 创建协程的上下文
（和所有其他协同工作者一样）。

capacity  - 这个数字定义了什么类型的通道
将被创建为 actor 的“邮箱”：

- RendezvousChannel

容量 = 0

- LinkedListChannel

容量 = Channel.UNLIMITED

- ConflatedChannel

容量 = Channel.CONFLATED

- ArrayChannel

容量 = required (fixed) buffer size
start - 协程启动选项
（和所有其他协同工作者一样）。

block - 协程代码。
这里我们提供了实际的 actor 的代码。

最重要的参数是最后一个参数。 这是我们定义我们的 actor 实际上会做什么的地方。 正如你可以看到它是一个接收器类型的挂起函数：[ActorScope](https://github.com/Kotlin/kotlinx.coroutines/blob/7b10c942fd769b64d8c33fbbe5a667ab028230b3/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Actor.kt#L26)。 这个范围（除了扩展通常的 [CoroutineScope](https://github.com/Kotlin/kotlinx.coroutines/blob/7b10c942fd769b64d8c33fbbe5a667ab028230b3/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/CoroutineScope.kt#L26)）扩展了 [ReceiveChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Channel.kt#L104)。 由于这个原因，我们可以方便地从我们的协程代码中访问我们的 “邮箱”。 我们甚至可以使用普通的 for 循环遍历发送给我们的元素。 此循环将调用接收操作，所以如果接收操作挂起，它将挂起协程。

不要担心这听起来很复杂 - 一旦我们看看我们的应用程序中的 actor 的实际代码，这一切都将清楚。 但是，在此之前，我们来看看最后一件事情 - 从协程生成器返回的类型：ActorJob。 ActorJob 接口扩展了 Job 接口 - 从协同构建器返回的所有对象，但它也扩展了 SendChannel 接口。 这样用户可以使用发送操作轻松地将元素发送给我们的 actor。 用户还可以尝试使用其他不可挂起的操作（offer）将元素发送到频道。 它只是返回 false 而不是挂起，如果它不能发送新的元素到这个通道。 这是我们将在我们的应用程序中做的。

### 05.3 Our First Actor

现在，在处理了一些理论之后，我们准备分析我们应用程序中的主角。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froutn4u00j30jg0mdq6o.jpg)

> [https://github.com/elpassion/crweather/…../MainModel.kt](https://github.com/elpassion/crweather/…../MainModel.kt)

有三件重要的事情可以在 actor 协同程序构建器调用中看到（第27行）：

- 我们的  actor 的通道的类型是 [Action](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/DataTypes.kt#L21) 。这意味着用户可以通过通道向我们的 actor 发送动作。目前，只有一个可能的操作类型：[SelectCity](https://github.com/elpassion/crweather/blob/0d2489b26d9e3d50ec84e87306f71610bbc12b2e/app/src/main/java/com/elpassion/crweather/DataTypes.kt#L23)，但是这个设置展示了如何将它扩展到其他用户操作。
- 我们使用 android 特定的 [UI](https://github.com/Kotlin/kotlinx.coroutines/blob/30dd5c129c493ca7b1a3b2b80437e8a7ec68f3ec/ui/kotlinx-coroutines-android/src/main/kotlin/kotlinx/coroutines/experimental/android/HandlerContext.kt#L28) 协程调度器作为协同程序上下文 - 这确保了这个 actor 的所有代码都将在 android 主线程上执行。它允许我们通过调用 actor 的代码中的 [MutableLiveData.setValue](https://developer.android.com/reference/android/arch/lifecycle/MutableLiveData.html#setValue%28T%29) 函数来轻松地更改应用程序状态。
- 我们也可以使用通常的 [CommonPool](https://github.com/Kotlin/kotlinx.coroutines/blob/ecda27fa01602e42922eadc16ef34b920b8006bb/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/CommonPool.kt#L31) 协同程序调度程序，但是我们必须使用 [MutableLiveData.postValue](https://developer.android.com/reference/android/arch/lifecycle/MutableLiveData.html#postValue%28T%29) 来更改应用程序状态。
- 我们使用 [Channel.CONFLATED](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/ConflatedChannel.kt) 作为 “邮箱” 通道类型，所以新的用户操作取代了所有未处理的操作。在我们的情况下，这正是我们想要的，因为如果我们还没有选择用户请求的城市，用户现在想要选择另一个城市 - 我们应该忘记旧的行动，并尽快处理新的行动。

最后，我们来看一下 actor 的代码：

这只是一个 for 循环遍历 “this”。 正如我们所记得的，actor 的代码的 “this”（接收者）是扩展了 [ReceiveChannel](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Channel.kt#L104) 的 [ActorScope](https://github.com/Kotlin/kotlinx.coroutines/blob/7b10c942fd769b64d8c33fbbe5a667ab028230b3/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Actor.kt#L26)。 ReceiveChannel 类型定义了[迭代器](https://github.com/Kotlin/kotlinx.coroutines/blob/a74eb5f8b08e84f904e75aa71da4547c9178da8d/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/channels/Channel.kt#L177)，所以我们可以在 for 循环中使用它。 每次迭代都会调用接收操作，如果邮箱通道为空，可能会挂起。 所以这是我们的第一个悬挂点（IDE 用绿线穿过灰色箭头标记每个悬挂点）。 通道关闭时，迭代结束。 在 for 循环中，我们只是检查接收到的动作的类型（目前总是 SelectCity）并做适当的工作。

现在，我们必须实现 SelectCity 用户操作。 首先，我们立即更改应用程序状态的两个部分：我们将城市 LiveData 值设置为用户请求的城市名称（因此我们的视图 MainActivity 将显示新城市作为选定城市），并将 LiveData 值加载到 true（所以我们 视图显示某种进度指示器）。

最后，我们尝试收集新的选定城市的数据。 我想这条线（nr 21/20）是应用程序中最重要的线:-)首先，我们寻找缓存中给定城市的（新鲜）图表。 缓存上的助手扩展函数：getFreshCharts 返回 null，如果没有新的图表被找到，如果是这样的话：我们通过调用 getNewCharts 挂起函数进行网络调用。 这是我们在演员代码中的第二个也是最后一个暂停点。 恢复之后，我们设置状态：将 LiveData 值图表（也在第21/20行），并将加载的 LiveData 值设置为 false（第16行）。 如果 getNewCharts 暂停点恢复异常，我们会捕获它并通过消息 LiveData 显示错误消息。

> 一个小问题与这些信息有关ーー你能猜到吗？ 提示: 想想设备的旋转 

suspend fun getNewCharts  - 除了调用我们精心准备的来自 Repository 的 suspend fun getCityCharts  - 还保存缓存中特定城市的新图表。

这几乎是我们应用程序的所有业务逻辑，都以简洁清晰的方式实现，这要感谢归功于 actor（以及一般的协程）。

> 注意我们在 actor 的代码中改变外部状态。 一般来说，并发计算的 [actor 模型](https://en.wikipedia.org/wiki/Actor_model)禁止任何共享状态，以支持通过 actor 的邮箱实现所有的通信。
>
> 如果你想了解更多 actor，我真的很喜欢看这个视频：[actor 模型](https://youtu.be/7erJ1DV_Tlo)，出自 [Erik Meijer](https://en.wikipedia.org/wiki/Erik_Meijer_%28computer_scientist%29), [Carl Hewitt](https://en.wikipedia.org/wiki/Carl_Hewitt), Clemens Szyperski

应用程序的最后一部分只是一个视图。它负责显示当前的应用程序状态，并将用户操作转发给主应用程序模型（我们刚刚讨论过）。你可以在我的下一篇博文中阅读关于视图层的实现：

