# 使用 MVI 的反应性应用 - 第6部分 - RESTORING STATE

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART6 - RESTORING STATE](http://hannesdorfmann.com/android/mosby3-mvi-6)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

本文是博客文章系列“带模型 - 视图 - 意图的反应式应用程序”的一部分。 这里是目录：

- [Part 1: Model](http://hannesdorfmann.com/android/mosby3-mvi-1)
- [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)
- [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)
- [Part 4: Independent UI Components](http://hannesdorfmann.com/android/mosby3-mvi-4)
- [Part 5: Debugging with ease](http://hannesdorfmann.com/android/mosby3-mvi-5)
- [Part 6: Restoring State](http://hannesdorfmann.com/android/mosby3-mvi-6)
- [Part 7: Timing (SingleLiveEvent problem)](http://hannesdorfmann.com/android/mosby3-mvi-7)



[TOC]

在之前的博文中，我们讨论了模型 - 视图 - 意图（MVI）以及单向数据流的重要性。 这简化了状态的恢复。 如何和为什么？ 我们将在这个博客文章中讨论这个问题。

在本篇博文中，我们将重点讨论两种情况：在内存中恢复状态（例如在屏幕方向更改期间）和恢复“持久状态”（例如，先前保存在 Activity.onSaveInstanceState（）中的 Bundle）。

## 在内存

这是简单的情况。我们只需要让我们的 RxJava 流随着时间的推移发布 android 组件生命周期（即 Activity，Fragment 甚至 ViewGroups）。例如，Mosby 的 MviBasePresenter 通过使用 PublishSubjectfor View intents 和 BehaviorSubject 在内部建立这样一个 RxJava 流来渲染视图上的状态。我已经在第2部分的末尾描述了这些实现细节。主要思想是 MviBasePresenter 是一个组件，它位于 View 的 lifecylce 之外，以便可以将视图附加到这个 Presenter 并将其分离。在 Mosby，当视图被永久销毁时，Presenter 会被“摧毁”（被垃圾收集）。再次，这只是 Mosby 的实现细节。你的 MVI 实现可能完全不同。重要的是像 Presenter 这样的组件生活在 View 的生命周期之外，因为那么处理 View 附加事件和分离事件是很容易的：只要 View 被重新连接到 Presenter，我们只需调用 view.render（previousState）（因此 Mosby 在内部使用 BehaviorSubject）。这只是如何处理屏幕方向更改的一个解决方案。它也适用于后退堆栈导航，即后退堆栈中的片段：如果我们从堆栈回来，我们只需再次调用view.render（previousState），并且视图显示正确的状态。实际上，即使没有添加视图，状态仍然可以被更新，因为 Presenter 不在该生命周期中，并且保持 RxJava 状态流处于活动状态。想象一下，接收一个推送通知，改变数据（状态的一部分），而没有附加视图。同样，只要视图重新连接，最新状态（包含来自推送通知的更新数据）将切换到视图进行渲染。

## 持久的状态

对于像MVI这样的单向数据流模式，这种情况也简单得多。 比方说，我们希望我们的视图（即活动）的状态不仅在内存中存活，而且还通过进程程死亡。 通常在 Android 中，会使用 Activity.onSaveInstanceState（Bundle）来保存该状态。 与 MVP 或 MVVM 相反，在 MVI 中，您不一定具有表示状态的模型（请参阅第1部分），您的视图具有渲染（状态）方法，可以轻松跟踪最新状态。 所以最明显的解决方案是使状态 Parcelable 并将其存储到包中，然后像这样恢复它：

```java
class MyActivity extends Activity implements MyView {

  private final static KEY_STATE = "MyStateKey";
  private MyViewState lastState;

  @Override
  public void render(MyState state) {
    lastState = state;
    ... // update UI widgets
  }

  @Override
  public void onSaveInstanceState(Bundle out){
    out.putParcelable(KEY_STATE, lastState);
  }

  @Override
  public void onCreate(Bundle saved){
    super.onCreate(saved);
    MyViewState initialState = null;
    if (saved != null){
      initialState = saved.getParcelable(KEY_STATE);
    }

    presenter = new MyPresenter( new MyStateReducer(initialState) ); // With dagger: new MyDaggerModule(initialState)
  }
  ...
}
```

我认为你知道怎么做了。 请注意，在 onCreate（）中，我们并不直接调用 view.render（initialState），而是让初始状态下沉到状态管理发生的地方：状态简化器（参见第3部分），我们将它与 .scan(initialState，reducerFunction）一起使用 。

## 结束

使用单向数据流和代表状态的模型，与其他模式相比，许多状态相关的事情实现起来要简单得多。 但是，通常我不会将状态保存到应用程序中，原因有两个：首先，Bundle 具有大小限制，因此不能将任意大的状态放入一个包中（或者可以将状态另存为文件或 像 Realm 这样的对象存储）。 其次，我们只讨论了如何序列化和反序列化状态，但这不一定与恢复状态一样。

例如：假设我们有一个 LCE（加载内容错误）视图，在加载数据和项目列表之后，加载数据（项目）时显示加载指示器。 所以状态就像 MyViewState.LOADING。 假设加载需要一些时间，并且加载时 Activity 处理被终止（也就是因为另一个应用程序由于来电而进入手机应用程序的前台）。 如果我们只是序列化 MyViewState.LOADING 并在如上所述重新创建 Activity 之后反序列化它，那么我们的状态简化器将会调用迄今为止是正确的 view.render（MyViewState.LOADING），但实际上我们不会再调用加载数据 http 请求）只是盲目地使用反序列化的状态。

正如您所看到的，序列化和反序列化状态与状态恢复不同，状态恢复可能需要一些额外的步骤来增加复杂性（与使用迄今为止使用的任何其他架构模式相比，使用 MVI 仍然更简单）。 包含某些数据的反序列化状态在 View 重新创建时可能会过时，因此您可能必须刷新（加载数据）。 在我工作的大多数应用程序中，我发现只有在内存中保持状态并且在进程死亡之后才开始使用空初始状态，就好像应用程序第一次启动那样简单和更友好。 理想情况下，应用程序具有缓存和脱机支持，以便在死亡后加载数据的速度很快。

这最终导致了一个共同的信念，我已经与其他 Android 开发人员进行了一些艰苦的争论：如果我使用缓存或存储，我已经有这样一个组件生活在 android 组件生命周期之外，我不必全部 保留组件的东西和 MVI 无关的，对吧？ 大多数时候，这些 Android 开发人员指的是 Mike Nakhimovich 发表的 [Presenters are not for persisting](https://hackernoon.com/presenters-are-not-for-persisting-f537a2cc7962) 介绍 [NyTimes Store](https://github.com/NYTimes/Store)，一个数据加载和缓存库。 不幸的是，那些开发人员不明白，加载数据和缓存不是状态管理。 例如，如果我必须从2个商店或缓存加载数据呢？

最后，像 NyTimes Store这样的缓存库能帮助我们处理进程死亡吗？ 显然不是因为程序死亡可能随时发生。 处理它。 我们唯一可以做的就是要求android操作系统不要杀死我们的应用程序进程，因为我们仍然有一些工作要做，通过使用 android 服务（也是这样一个组件，生活在其他 android 组件生命周期以外）或不 我们现在需要使用 RxJava 的 android 服务了吗？ 我们将在下一部分讨论 Android 服务，RxJava 和 MVI。 敬请关注。

