# ViewModels 和 LiveData : 模式 + 反模式

> 原文 (Medium)：[ViewModels and LiveData: Patterns + AntiPatterns](https://medium.com/google-developers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54)
>
> 作者：[Jose Alcérreca](https://medium.com/@JoseAlcerreca?source=post_header_lockup)

[TOC]

## 分配责任

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovce2kgij30m806dzkr.jpg)

> 使用体系结构组件构建的应用程序中实体的典型交互

理想情况下，ViewModels 不应该知道任何有关 Android 的信息。 这提高了可测试性，泄漏安全性和模块性。 一般的经验法则是确保在 ViewModels 中没有 android 框架。（导入 android.arch 之类的例外）。这同样适用于演示者。 

> ❌ 不要让 ViewModel（和演示者）知道 Android 框架类

条件语句，循环和一般决策应在 ViewModel 或应用程序的其他层面进行，而不是在 Activities 或 Fragments 中。 视图通常不是单元测试（除非你使用 [Robolectric](http://robolectric.org/)），所以代码行越少越好。 视图应该只知道如何显示数据并将用户事件发送到 ViewModel（或 Presenter）。 这被称为[被动视图](https://martinfowler.com/eaaDev/PassiveScreen.html)模式。

> ✅ 将活动和片段中的逻辑保持在最低限度

## 在 ViewModels 中的视图引用

[ViewModels](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 与活动或片段有不同的作用域。虽然 ViewModel 处于运行状态时，活动可以处于任何[生命周期状态](https://developer.android.com/guide/components/activities/activity-lifecycle.html)。活动和片段可以在当 ViewModel 不知情情况下被销毁和重新创建。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovch3cdcj30m80got9k.jpg)

> 配置更改时 ViewModels 会保持

将视图（活动或片段）的引用传递给 ViewModel 是一个严重的风险。 让我们假设 ViewModel 从网络请求数据，数据会在一段时间后返回。此时，视图引用可能会被销毁，或者可能是不再可见的旧活动，从而产生内存泄漏，并可能导致崩溃。

> ❌ 避免在 ViewModels 中引用视图。

在 ViewModels 和 Views 之间建议的通信方式是使用 LiveData 或其他库的 observable  进行观察者模式通信。 

## 观察者模式

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovck3jj8j30bg02vwee.jpg)

在 Android 中设计演示层的一个非常方便的方法是使 View (活动或片段)观察(订阅更改) ViewModel。 因为 ViewModel 并不了解 Android，它不知道 Android 喜欢如何频繁地杀死视图。 这有一些好处: 

1. 对于配置更改，ViewModels 持久化，所以当发生旋转时，不需要重新查询外部数据源(如数据库或网络) 。
2. 当长时间运行的操作结束时，ViewModel 中的观察值被更新。 这些数据是否被观察并不重要。 当尝试更新不存在的视图时，没有空指针异常发生 。
3. ViewModels 不会引用视图，因此内存泄漏的风险就会降低 

```java
private void subscribeToModel() {
  // Observe product data
  viewModel.getObservableProduct().observe(this, new Observer<Product>() {
      @Override
      public void onChanged(@Nullable Product product) {
        mTitle.setText(product.title);
      }
  });
}
```

> 来自活动或片段的典型订阅。

> ✅ 不要将数据推送到用户界面，而是让用户界面观察它的变化 。

## 臃肿的 ViewModels

任何能让你分离关注的事情都是一个好主意。 如果你的 ViewModel 持有太多的代码或者有太多的责任，考虑: 

- 将一些逻辑移动到一个演示者，与 ViewModel 相同的范围。 它将与你的应用程序的其他部分进行通信，并在 ViewModel 中更新 LiveData 持有者。 
- 添加一个域层并采用 [Clean Architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)。这导致一个可测试和可维护的架构。这也有利于快速下线。[架构蓝图](https://github.com/googlesamples/android-architecture)中有一个简洁架构示例。

> ✅ 分配职责，如果需要添加一个域层。

## 使用数据存储库

从[应用程序体系结构指南](https://developer.android.com/topic/libraries/architecture/guide.html)中所看到的，大多数应用程序都有多个数据源，例如：

- Remote: network or the Cloud
- Local: database or file
- In-memory cache

在你的应用程序中有一个数据层是个好主意，完全没有意识到你的演示层。 保持缓存和数据库与网络同步的算法并非微不足道。 建议使用单独的存储库类作为处理这种复杂性的单一入口点。 

如果你有多个和非常不同的数据模型，考虑添加多个存储库。 

> ✅ 将数据存储库添加为数据的单点条目。

## 处理数据状态

考虑这个场景: 你正在观察一个由 ViewModel 公开的 LiveData，该模型包含要显示的项目列表。 视图如何在加载数据、网络错误和空列表之间进行转换？ 

- 你可以从 ViewModel 公开 `LiveData <MyDataState>` 。例如，MyDataState 可能包含关于数据是否正在加载，已成功加载或失败的信息。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovcnqlxkj30e204kwei.jpg)

你可以将数据包含在具有状态和其他元数据的类中，比如错误消息。 请参见样例中的 [Resource](https://developer.android.com/topic/libraries/architecture/guide.html#addendum) 类。 

> 使用包装器或另一个 LiveData 来公开你的数据状态 。

## 保存活动状态

活动状态是在活动消失时重新创建屏幕所需的信息，这意味着活动被破坏或者进程被杀死。 旋转是最明显的情况，我们已经覆盖了 ViewModels。 如果保存在 ViewModel 中，那么状态是安全的。 

然而，你可能需要在 ViewModels 也不复存在的其他场景中恢复状态: 例如，当操作系统资源不足并且杀死你的进程时，你可能需要恢复状态。 

为了有效地保存和恢复用户界面状态，使用 onSaveInstanceState ( ) 和 ViewModels 的组合。 

有关示例，请参阅：[ViewModels: Persistence, onSaveInstanceState( ), Restoring UI State and Loaders](https://medium.com/google-developers/viewmodels-persistence-onsaveinstancestate-restoring-ui-state-and-loaders-fc7cc4a6c090)

## 事件

事件就是发生过一次的事情。 模型会暴露数据，但事件又如何呢？ 例如，导航事件或显示 Snackbar 消息是应该只执行一次的操作。 

事件的概念与 LiveData 如何存储和恢复数据不完全吻合。 考虑一个具有以下字段的 ViewModel: 

```java
LiveData<String> snackbarMessage = new MutableLiveData<>();
```

一个 activity 开始观察这个，并且 ViewModel 完成了一个操作，所以它需要更新消息: 

```java
snackbarMessage.setValue("Item saved!");
```

该活动接收值，并显示 Snackbar。 很显然，这是有效的。 

但是，如果用户旋转手机，则新的活动被创建并开始观察。当 LiveData 观测开始时，活动立即接收到旧值，从而导致消息再次显示！ 

我们扩展了 LiveData 并创建了一个名为 [SingleLiveEvent](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java) 的类，作为我们示例中的解决方案。它只发送订阅后发生的更新。请注意，它只支持一个观察者。

> ✅ 使用像 SingleLiveEvent 这样的可观察事件来处理导航或 Snackbar 消息等事件。

## 泄漏 ViewModels

反应模式在安卓系统中运行良好，因为它允许用户界面和应用程序的其他层之间建立一个方便的连接。 LiveData 是这个结构的关键组成部分，所以通常你的活动和片段会观察 LiveData 实例。 

如何与其他组件进行通信取决于你，但要小心漏洞和边缘情况。 考虑这个图表，演示层使用观察者模式，数据层使用回调: 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovcqpho2j30m406ldg4.jpg)

> 用户界面的观察者模式和数据层中的回调 。

如果用户退出应用程序，视图将会消失，所以 ViewModel 不再被观察。 如果存储库是一个单独的或对应用程序有其他范围的，则存储库将不会被销毁，直到该进程被杀死。 只有当系统需要资源或者用户手动杀死应用程序时，这种情况才会发生。 如果存储库在 ViewModel 中持有一个回调引用，ViewModel 将会被暂时泄露 。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovct4mx0j30m806a0t8.jpg)

> 活动已经结束，但 ViewModel 仍然存在 。

如果 ViewModel 是轻量级的，或者操作保证能够快速完成，这个漏洞就不是什么大问题了。 然而，情况并非总是如此。 理想情况下，ViewModels 应该可以在没有任何观点观察的情况下自由使用: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovcvez51j30m606e3yt.jpg)

你有很多选择来实现这个目标: 

-  通过 ViewModel.onCleared ( ) ，你可以告诉存储库将回调放到 ViewModel 。
- 在存储库中，你可以使用弱引用或者你可以使用事件总线(容易被滥用，甚至被认为是有害的) 。
- 使用 LiveData 在存储库和 ViewModel 之间以类似的方式在视图和视图模型之间使用 LiveData 。

> ✅  考虑边缘情况、泄漏以及运行时间长的操作是如何影响架构中的实例的 。
>
> ❌ 不要把逻辑放在 ViewModel 中，它对于保存简洁状态或与数据相关是至关重要的。 任何来自 ViewModel 的调用都可能是最后一个 。

## 存储库中的 LiveData

为了避免视图模型的泄漏和回调地狱，可以这样观察存储库: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovcyrl30j30m805uwf2.jpg)

当 ViewModel 被清除或视图的生命周期结束时，订阅被清除: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovd1hr39j30m606e3yt.jpg)

如果你尝试这种方法有一个问题：如果你无法访问 LifecycleOwner，你如何从 ViewModel 订阅存储库？ 使用 [Transformations](https://developer.android.com/topic/libraries/architecture/livedata.html#transformations_of_livedata) 是解决这个问题的一个非常方便的方法。 Transformations.switchMap 允许你创建一个新的 LiveData，以响应其他 LiveData 实例的更改。 它还允许贯穿链上的观察者生命周期信息：

```java
LiveData<Repo> repo = Transformations.switchMap(repoIdLiveData, repoId -> {
        if (repoId.isEmpty()) {
            return AbsentLiveData.create();
        }
        return repository.loadRepo(repoId);
    }
);
```

> Transformations 示例[[来源]](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/ui/repo/RepoViewModel.java) 。

在这个例子中，当触发器得到一个更新时，该函数被应用并且结果被下游调度。一个活动将观察到 repo，并且相同的 LifecycleOwner 将用于 repository.loadRepo（id）调用。

> ✅ 每当你认为在 [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html) 中需要[生命周期](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.html)对象，[转换](https://developer.android.com/topic/libraries/architecture/livedata.html#transformations_of_livedata)可能就是解决方案。

## 扩展 LiveData

最常用的 LiveData 用例是在 ViewModels 中使用 MutableLiveData，并将它们作为 LiveData 显示出来，使其不受观察者的影响。 

如果你需要更多的功能，扩展 LiveData 会让你知道什么时候有活跃的观察者。 例如，当你想要开始监听一个位置或者传感器服务时，这是非常有用的。 

```java
public class MyLiveData extends LiveData<MyData> {

    public MyLiveData(Context context) {
        // Initialize service
    }

    @Override
    protected void onActive() {
        // Start listening
    }

    @Override
    protected void onInactive() {
        // Stop listening
    }
}
```

## 何时不扩展 LiveData

你也可以使用 onActive ( ) 启动一些加载数据的服务，但除非你有充分的理由，否则你不需要等待 LiveData 被观察到。 一些常见的模式: 

- 将一个 start ( ) 方法添加到 ViewModel 并尽快调用[请参阅[蓝图示例](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/addedittask/AddEditTaskFragment.java#L64)] 。
- 设置一个启动负载的属性 [见 [GithubBrowserExample](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/ui/repo/RepoFragment.java#L81)] 。

> ❌ 你通常不会扩展 LiveData。当开始加载数据时，让你的活动或片段告诉 ViewModel 。 



对于与体系结构组件相关的任何其他主题，你是否需要指导(或意见)？ 请在评论中告诉我们。 

