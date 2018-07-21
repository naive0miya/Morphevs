# 使用 MVI 的反应性应用 - 第5部分 - DEBUGGING WITH EASE

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART5 - DEBUGGING WITH EASE](http://hannesdorfmann.com/android/mosby3-mvi-5)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

本文是博客文章系列“带模型 - 视图 - 意图的反应式应用程序”的一部分。 这里是目录：

- [Part 1: Model](http://hannesdorfmann.com/android/mosby3-mvi-1)
- [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)
- [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)
- [Part 4: Independent UI Components](http://hannesdorfmann.com/android/mosby3-mvi-4)
- [Part 5: Debugging with ease](http://hannesdorfmann.com/android/mosby3-mvi-5)
- [Part 6: Restoring State](http://hannesdorfmann.com/android/mosby3-mvi-6)
- [Part 7: Timing (SingleLiveEvent problem)](http://hannesdorfmann.com/android/mosby3-mvi-7)


在之前的博客文章中，我们讨论了模型 - 视图 - 意图（Model-View-Intent，MVI）模式及其特征。 在第1部分中，我们谈到了由“业务逻辑”驱动的单向数据流和应用程序状态的重要性。 在这个博客文章中，我们将看到如何在调试时取得如此的成果，以简化开发人员的生活。

你有没有得到一个崩溃报告，你不能重现错误？ 听起来很熟悉？ 对我也是！ 花了好几个小时看了堆栈跟踪和我们的源代码之后，我放弃了这个问题，并在“问题跟踪器”中用“不能重现它”或“必须是奇怪的设备/制造商特定的错误”等一些评论来解决这些问题。

我们在本博客文章系列中开发的购物车应用程序的具体示例：在主屏幕上，我们的应用程序的用户可以执行 “拉到刷新”，并以某种方式进行刷新，正如崩溃报告所述，加载时出现 NullPointerException 异常 通过拉到刷新触发的最新数据。

所以你作为开发者启动了应用程序，并在主屏幕上进行了刷新，但该应用程序没有崩溃。 它按预期工作。 所以你仔细看看你的代码，但你不能看到如何抛出一个 NullPointerException。 你附加一个调试器，一步一步通过所涉及的组件的代码，但仍然：它工作正常。 这个应用程序如何能够在拉到刷新上崩溃？

问题是你不能在崩溃发生之前重现状态。 如果发生崩溃的用户可以在崩溃报告中提供他的应用程序状态（发生崩溃之前）以及跟踪堆栈跟踪，这不是很好吗？ 通过单向数据流和 Model-View-Intent，这非常容易。 我们只记录用户触发的所有意图以及在视图上呈现的模型（模型反映了应用程序状态，即 a。视图状态）。 我们通过在 HomePresenter 中添加日志来实现主屏幕的功能（有关此类的更多详细信息，请参阅第3部分，我们已经讨论了状态缩减器的优点）。 在以下代码片段中，我们使用了 [Crashlytics](https://fabric.io/kits/ios/crashlytics)，但应该可以对其他任何崩溃报告工具进行相同的操作。

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeViewState initialState; // Show loading indicator

  public HomePresenter(HomeViewState initialState){
    this.initialState = initialState;
  }

  @Override protected void bindIntents() {

    Observable<PartialState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: load first page"))
          .flatmap(...); // business logic calls to load data

    Observable<PartialState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: pull-to-refresh"))
          .flatmap(...); // business logic calls to load data

    Observable<PartialState> nextPage = intent(HomeView::loadNextPageIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: load next page"))
          .flatmap(...); // business logic calls to load data

    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh, nextPage);
    Observable<HomeViewState> stateObservable = allIntents
          .scan(initialState, this::viewStateReducer) // call the state reducer
          .doOnNext(newViewState -> Crashlytics.log( "State: "+gson.toJson(newViewState) ));

    subscribeViewState(stateObservable, HomeView::render); // display new state
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    ...
  }
}
```

我们只需使用 RxJava 的 .doOnNext（）运算符为每个意图添加日志记录，然后为每个意图的 “结果”（视图状态）添加日志记录，然后将在视图上呈现该视图状态。我们将视图状态序列化为 json（我们将在一分钟内讨论）。

现在我们的崩溃报告如下所示：

![](http://hannesdorfmann.com/images/mvi-mosby3/crashlytics-mvi-logs.png)

看一下日志：不仅我们看到崩溃发生之前的最新状态，但我们可以看到用户如何到达这个状态的完整历史。为了更好的可读性，我已经强调了状态转换，并用[...]替换了“数据”字段（在回收视图中显示的项目）。因此，用户启动了应用程序 - 加载首页意图。然后加载指示器显示 “loadingFirstPage”：true，然后加载数据（data [...]）。接下来，用户滚动浏览项目列表，并到达 RecyclerView 的末尾，触发加载下一页页面意图加载更多的数据（分页），导致状态转移到 “loadingNextPage”：true。一旦下一页加载，数据（数据）已被更新，并且 “loadedNextPage”：false 已被正确设置。用户也做了同样的事情（向下滚动 RecyclerView，并第二次触发加载下一页的意图）。然后他开始了一个拉到刷新的意图，状态转换到 “loadingPullToRefresh”：true。突然，应用程序崩溃（之后没有更多的日志）。

那么这些信息如何帮助我们修复这个 bug 呢？ 显然，我们知道用户触发了哪些意图，所以我们可以手动来重现错误。 而且，我们将应用程序状态（随着时间的推移）作为json快照。 我们可以简单地把最后一个状态，反序列化 JSON，并把这个状态作为我们的初始状态来修复这个错误：

```java
String json ="  {\"data\":[...],\"loadingFirstPage\":false,\"loadingNextPage\":false,\"loadingPullToRefresh\":false} ";
HomeViewState stateBeforeCrash = gson.fromJson(json, HomeViewState.class);
HomePresenter homePresenter = new HomePresenter(stateBeforeCrash);
```

然后，我们附上一个调试器，并触发拉到刷新的意图。 事实证明，如果用户向下滚动了页面两次，没有更多的数据可用，我们的应用程序没有正确处理这个事实，所以下面的拉动刷新导致崩溃。

**综述**

使应用程序的状态“快照”，使我们的开发人员的生活更容易。 不仅如此，我们可以很容易地重现崩溃，此外，我们可以将序列化状态写入几乎为零的额外成本的[回归测试](https://en.wikipedia.org/wiki/Regression_testing)。 请记住，只有当应用程序的状态遵循单向数据流（由业务逻辑驱动），不可变性和纯功能的原则时才可能这样做。 Model-View-Intent 将我们推向了这个方向，所以构建“快照”应用程序是这种架构的一个很好的和有用的副作用。

“快照”应用有什么缺点？显然，我们正在序列化应用程序状态（即与 Gson）。这增加了一些额外的计算时间。在我的平均大小的应用程序中，这首次需要大约30毫秒的时间才能与 Gson 进行序列化，因为 Gson 必须使用反射来扫描类，以确定必须序列化的字段。连续的状态序列化在 Nexus 4 上平均需要大约6毫秒的时间。由于序列化在 .doOnNext（）中运行，通常在后台线程上运行，但是，我的应用程序的用户必须等待6毫秒以上，屏幕上的新状态。从我的角度来看，应用程序用户并不注意。但是，快照状态的一个问题是，在崩溃时，崩溃报告工具从用户设备上传到他的服务器的数据量要大得多。没有什么大不了的，如果用户通过 WiFi 连接，但可能是用户移动数据计划的问题。最后但并非最不重要的一点是，将状态附加到崩溃报告时，可能会泄漏敏感的用户数据。要么不要将可能导致附加到崩溃报告的状态不完整（因此几乎无用）或敏感数据（可能需要一些额外的 cpu 时间）的敏感数据序列化。

总结一下：我个人认为在快照我的应用方面有很多好处，但是，您可能需要做一些权衡。也许你开始启用快照您的应用程序的内部版本或测试版本，看看它是如何运行你的应用程序。

**奖励：时间旅行**

在开发过程中有一些“时间旅行”的选择不是很好吗？也许嵌入在一个像 Jake Wharton 的 u2020 演示程序那样的调试抽屉中：

![](http://hannesdorfmann.com/images/mvi-mosby3/u2020-debug-drawer.gif)

我们在这样一个调试抽屉里需要的是两个按钮 “上一个状态” 和 “下一个状态”，以便我们可以一步一步地从一个状态回到上一个（或下一个）状态。 显然，这需要一些额外的设置，然后快照我们的应用程序的状态。 例如：如果我们将 HTTP 请求作为状态转换的一部分，我们当然不希望在时间旅行中再次发出真正的 HTTP 请求，因为后端数据可能同时发生了变化。

时间旅行需要在应用程序的边界处使用某种额外的图层，例如 http 请求（对于 sqlite 数据库等），我们可以 “记录” 和 “重播” 这些内容。 对这样的事情感兴趣？ 似乎我的朋友 Felipe 正在为 OkHttp 做这样的事情。 跟随他，以获得他目前正在工作的库的更多细节。[Felipe Lima](https://twitter.com/felipecsl)

