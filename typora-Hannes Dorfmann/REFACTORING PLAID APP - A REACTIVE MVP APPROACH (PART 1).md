# 重构 plaid (第1部分) - 一种反应性 MVP 方式

> 原文：[REFACTORING PLAID APP - A REACTIVE MVP APPROACH (PART 1)](http://hannesdorfmann.com/android/plaid-refactored-1)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)
>
> 译文：[重构 Plaid -响应式的 MVP 模式(一)](http://hanks.xyz/2015/12/08/refactoring-plaid-1/)
>
> 译者：[hanks-zyh](https://github.com/hanks-zyh)

[TOC]

Nick Butcher 已经在 [github](https://github.com/nickbutcher/plaid) 上开源了一个名为 Plaid 的应用程序。 该应用程序非常酷，并具有出色的 UI / UX。 每当这样的应用程序的源代码可用，开发人员开始复制代码和最佳实践的技巧。 所以我也这样做了，并决定深入到格子的代码。 与往常一样，您会发现您认为可以实现的一些代码部分，或者以其他方式编写代码。 所以，不要只是谈论代码，我已经决定花一些业余时间来重构格子的源代码的一些部分。 这篇博客是我将重构 plaid 应用程序并分享我的想法的系列3篇文章的第一部分。

前言：我开始重构，并坚信我可以重构整个应用程序。 令人惊讶（讽刺），事实证明，我是一个非常天真的男人。 我只是低估了应用程序的复杂性和功能的数量。 我只是在用 Nick Buchter 的代码花了几个小时才发现了一些功能，这些代码对于我来说只是使用这个应用程序而不是 “可见的”。 总之：我意识到我没有时间把所有的想法付诸行动。 因此，我的“重构”主要集中在应用程序的 “主页” 和 “搜索屏幕” 上，而这些主要是我重构的。 不过，我想讨论（理论上）更多可以重构的东西，但是我并没有因为时间的限制。 我的重构代码也可以在 [github](https://github.com/sockeqwe/plaid) 上找到。

## 第一眼

整体用户体验和用户界面非常棒。我无法形容它比这个推文更好：[Twitter](https://twitter.com/lgvalle/status/661509155455410176?ref_src=twsrc%5Etfw&ref_url=http%3A%2F%2Fhannesdorfmann.com%2Fandroid%2Fplaid-refactored-1)

使用该应用程序是一种乐趣。用户界面/用户体验是每个开发人员和设计师的真正灵感。然而，玩了一些应用程序后，我面对一些视觉错误：

- 同时显示加载指示器和错误消息：[plaid app: error and loading at the same time](https://www.youtube.com/watch?v=zCwESjEpNdk)
- 在应用程序中，您可以应用一些 “过滤器” 或换句话说：您可以指定要显示的 Dribbble，Designer News 和 Product Hunt 的哪些 “来源”。 如果在加载当前正在运行的 http 请求时取消选择这些源，则可以运行到应用程序显示项目的状态，但不应该选择源：[plaid app - no filters](https://www.youtube.com/watch?v=nJ3VUjpW0N0)
- 此外，应用程序不能正确处理屏幕方向的变化。它只是重建整个屏幕，以便在每个方向改变后，你再次看到加载指标，并重新执行 http 调用：[plaid - portrait landscape](https://www.youtube.com/watch?v=tuIDrtvL0lg)

通常，这些“问题”是意大利面代码和中等软件架构的第一个指标。所以让我们来看看底层的代码，并查看源代码和用于显示项目列表的组件：[HomeActivity](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/HomeActivity.java)（大约750行代码）处理UI元素的可见性，如显示 RecylcerView 或 ProgressBar。此活动还决定何时显示源过滤器 - 抽屉（在右侧）。此外，在 onActivityResults（）中，它包含了很多东西，包括向设计师新闻发布新帖子。最后但并非最不重要的是，它也通过使用 [DataManager](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/data/DataManager.java) 加载所选过滤器的数据。你看，HomeActivity 有很多责任，可能太多了。 DataManager 基本上使用 Retrofit 和 AsyncTasks 的组合来执行 http 调用来加载数据。棘手的事情是分页。每当到达 RecyclerView 的末尾，就会加载更多（旧的）项目。 DataManager 在内部使用一个 HashMap 来跟踪每个源（当前显示 “Dribbble Popular” 或 “Dribbble Recent” 或 “Designer News Popular” 等后端点）的当前页面。这些项目使用 [FeedAdapter](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/FeedAdapter.java) 显示在 RecyclerView 中。  [SearchActivity](https://github.com/nickbutcher/plaid/blob/master/app/src/main/java/io/plaidapp/ui/SearchActivity.java) 与 HomeActivity 的工作方式非常类似：它也使用 DataManager 和 FeedAdapter。

## 架构

从我的角度来看，目前的实施还没有明确的架构。 HomeActivity 是管理很多东西的某种神对象。 另一个 “问题” 是 HomeActivity 通过调用来自其他方法和不同事件的相同（内部）方法来改变 UI 状态，即从 HomeActivity 源代码中的4个不同的点调用 checkEmptyState（）方法。

我们将通过应用具有被动视图的 Model-View-Presenter 来重构该视图。 被动的视图只会显示演示者所要做的事情。 我是被动观图的 MVP 粉丝。 人们不时问我为什么我建议使用被动视图，而不是监督控制器或其他 MVP 派生。 那么，如果你使用 MVP 没有被动的观图，你基本上是把以前坐在视图中的意大利面代码分成半演示者和半视图。

## HomeActivity 中的 MVP

因此，使用 MVP +被动视图，我们将职责分成两个类：HomeActivity 实现 HomeView 现在被视为 View（被动视图）并实现这个接口：

```java
interface HomeView : MvpView {
    fun showLoading()
    fun showContent()
    fun showError()
    fun showLoadingMore(showing: Boolean)
    fun showLoadingMoreError(t: Throwable)
    fun addOlderItems(items: List<PlaidItem>)
}
```

从现在起，HomeActivity 的工作就是管理 UI 元素（可见性，协调动画等），但是只有在演示者告诉这样做之后。所以视图的状态由 HomePresenter 管理。 HomePresenter 看起来像这样：

```java
class HomePresenter(private val itemsLoader: ItemsLoader<List<PlaidItem>>) : RxPresenter<HomeView, List<PlaidItem>>() {

    fun loadItems() {

        view?.showLoading()
        subscribe(
                itemsLoader.firstPage(),
                { // onError
                    view?.showError()
                },
                { // onNext
                    view?.addOlderItems(it)
                    view?.showContent()
                }
        )

    }

    fun loadMore() {

        view?.showLoadingMore(true)
        subscribe(
                itemsLoader.olderPages(),
                { // onError
                    view?.showLoadingMoreError(it)
                },
                { // onNext
                    view.addOlderItems(it)
                    view.showLoadingMore(false)
                }
        )
    }
}
```

如果你还没有注意到：我们使用 kotlin 作为编程语言，主要是因为我喜欢这种语言，并有机会看到 kotlin 的开发如何在“真实世界”的应用程序中工作。 由于 kotlin 的互操作性，我可以很容易地重用 Nick Butcher 的 java 源代码，主要用于 UI / View 事物。

为了实现 MVP，我们使用了一个 MVP 库 [Mosby](http://hannesdorfmann.com/mosby/)，它也允许我们在屏幕方向改变时保持演示者。 因此，我们不必重新加载数据，并且屏幕方向更改后我们看不到 ProgressBar。 Mosby 允许我们保持观图状态像以前的方向变化。

最后但并非最不重要的，我决定使用 [RxJava](https://github.com/ReactiveX/RxJava) 作为我的 “模型”（因此演示者中的 subscribe（）方法）。所以 ItemsLoader 是我重构和反应的版本 Nick Butcher 的 DataManager。我会在一分钟内解释 ItemsLoader。

## SearchActivity 中的 MVP

如前所述，运行搜索与 HomeActivity 非常相似。 它显示项目的列表（网格），并添加分页加载更多的项目时，到达 RecyclerView 的结束。 所以将 MVP 应用于 SearchActivity 与之前显示的非常相似：

```kotlin
public interface SearchView extends MvpView {
  void showLoading();
  void showContent();
  void showError(Throwable t);
  void showLoadingMore(boolean showing);
  void showLoadingMoreError(Throwable t);
  void addOlderItems(List<PlaidItem> items);
  void showSearchNotStarted();
}
```

```kotlin
class SearchPresenter(private val itemsLoaderFactory: SearchItemsLoaderFactory) : RxPresenter<SearchView, List<PlaidItem>>() {

    private var itemsLoader: ItemsLoader<List<PlaidItem>>? = null

    fun search(query: String) {

        // Create items loader for the given query string
        itemsLoader = itemsLoaderFactory.create(query)

        view?.showLoading()

        subscribe(itemsLoader!!.firstPage(), { // Error handling
            view?.showError(it)
        }, { // onNext
            view?.addOlderItems(it)
        }, {
            view?.showContent()
        })
    }

    fun searchMore(query: String) {

        view?.showLoadingMore(true)
        subscribe(itemsLoader!!.olderPages(), { // Error handling
            view?.showLoadingMore(false)
            view?.showLoadingMoreError(it)
        }, { // onNext
            view?.addOlderItems(it)
        }, { // onComplete
            view?.showLoadingMore(false)
        })

    }

    fun clearSearch() {
        // Unsubscribe any previous search subscriptions
        unsubscribe()

        view.showSearchNotStarted()
    }
}
```

与 HomePresenter 相比，唯一不同的是 SearchPresenter 将 SearchItemsLoaderFactory 作为构造函数参数，并为每个搜索查询动态地创建一个 ItemsLoader。我们将在一分钟内看到这是如何工作的。

## ItemsLoader 和分页

到目前为止，我们已经涵盖了 View 和 Presenter。现在让我们来讨论一下，如果你想和 Bob 的干净架构进行比较，我们可以重构 “模型” 或用例/交互器。

在我们开始之前：有一个名为 PlaidItem 的类（拥有诸如 title 和 image url 之类的属性）。这个类是代表单个项目的基类：

- **Shot extends PlaidItem** 对于从 Dribbble 加载的项目
- **Story extends PlaidItem** 从 Designer News 加载的项目
- **Post extends PlaidItem** 对于从 Product Hunt 加载的项目

现在让我们来讨论一下如何通过使用 RxJava 来更有效地重写 DataManager。 我使用 RxJava，因为我觉得它很酷，现在所有的酷儿都必须使用 RxJava。 你会（希望）看到后来使用 RxJava 的好处（特别是在这个博客系列的第二部分）。

加载项目的难点在于我们支持来自不同后端的分页和加载项目。所以让我们从“自下而上”构建ItemsLoader。在“底部”，我们将找到一个用于执行 http 调用的 Retrofit 界面。现在 plaid 你可以搜索 DesignerNews 和 Dribbble。分页问题：Dribbble 加载100个项目，需要这样一个调用 loadItems（0,100），下一页将是 loadItems（100,200），而 DesignerNews 递增他的页面 1 loadItems（0，1），下一页将 loadItems （1,2）。我们需要一个通用的 API。由于我们使用 kotlin，我们可以传递 “方法引用”（函数 poitners）或 lambda 表达式。所以我们需要的是一个接受这个函数的组件，执行这个函数并返回一个 Observable，在那里我们得到实际的 http 调用的结果。所以基本上我们需要像这样： backendMethodToCall：（Int，Int） - > Observable <T>，其中第一个 int 参数是页面偏移量，第二个 int 参数是限制（每页应该加载多少项）和 T 是结果的泛型（实际上我们总是加载 List <PlaidItem>）。

我们称之为组件 RouteCaller：

```kotlin
class RouteCaller<T>(private val startPage: Int = 0,
                     private val itemsPerPage: Int,
                     private val backendMethodToCall: (Int, Int) -> Observable<T>) {

    /**
     * Offset for loading more
     */
    private val olderPageOffset = AtomicInteger(startPage)

    /**
     * A queue that is used to retry "older"
     * pages if they have failed before continue with even more older
     */
    private val olderFailedButRetryLater: Queue<Int> = LinkedBlockingQueue<Int>()

    /**
     * Get an observable to load older data from backend.
     */
    fun getOlderWithRetry(): Observable<T> {

        val pageOffset = if (
        olderFailedButRetryLater.isEmpty()) {
            olderPageOffset.addAndGet(itemsPerPage)
        } else {
            olderFailedButRetryLater.poll()
        }

        return backendMethodToCall(pageOffset, itemsPerPage)
                .doOnError {
                    olderFailedButRetryLater.add(pageOffset)
                }
    }

    /**
     * Get an observable to load the newest data from backend.
     * This method should be invoked on pull to refresh
     */
    fun getFirst(): Observable<T> {
        return backendMethodToCall(startPage, itemsPerPage)
    }
}
```

RouteCaller 采用这样的方法（Int，Int） - > Observable <T> 作为构造函数参数，并在内部使用正确的参数调用此方法：

- getFirst（）加载第一页
- getOlderWithRetry（）：此方法负责加载较旧的项目。 我们跟踪 oldPageOffset 字段中的当前页面，当我们开始加载更多的项目（换句话说，加载一个较老的页面）时，我们将增加这个页面。 此外，我们使用 .doOnError（）在加载下一页时重试加载失败的页面。

所以 RouteCaller 的职责是填写真实http后端端点调用所需的参数（页面偏移和页面限制）。所以我们有这样的东西：

![](http://hannesdorfmann.com/images/plaid/routing1.png)

为了执行搜索，我们有两个后端来查询 DesignerNewsService 和 DribbleService 。这意味着我们有两个RouteCaller（每个后端搜索方法调用一个）：

![](http://hannesdorfmann.com/images/plaid/routing2.png)

接下来的问题是：我们如何实例化 RouteCaller？我们为每个后端定义了一个 RouteCallerFactory，它基本上提供了一个方法 getAllBackendCallers（），在那里我们得到一个 Observable List <RouteCaller>我们应该执行加载项目。

```kotlin
interface RouteCallerFactory<T> {

    /**
     * Get all available backend route callers
     */
    fun getAllBackendCallers(): Observable<List<RouteCaller<T>>>
}
```

为了执行搜索，我们有了 DesignerNewsSearchCallerFactory 和 DribbbleSearchCallerFactory：DesignerNewsSearchCallerFactory 看起来像这样：

```kotlin
class DesignerNewsSearchCallerFactory(private val searchQuery: String, private val backend: DesignerNewsService) : RouteCallerFactory<List<PlaidItem>> {

    val extractPlaidItemsFromStory = fun(story: StoriesResponse): List<PlaidItem> {
        return story.stories
    }

    // The method to execute from RouteCaller
    val searchCall = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.search(searchQuery, pageOffset).map(extractPlaidItemsFromStory)
    }

    // Create a list with one single RouteCaller() with "searchCall" as method reference
    private val callers = arrayListOf(RouteCaller(0, 100, searchCall))

    override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
        return Observable.just(callers)
    }
}
```

乍一看，DesignerNewsSearchCallerFactory 看起来有点奇怪，因为我们不使用 lambdas，而是创建一个属性 searchCall，它实际上是一个函数 （Int，Int） - > Observable <List <PlaidItem >>。

我们这样做的原因是可测试性：最近我观看了Android开发峰会2015的聊天，他们会问何时会加入 Java 8 支持。 然后，来自 Android 团队的 Reto Meier 回答说，许多开发者主要是在 lambda 表达式中进行操作，并询问观众是否可以为 Android 开发提供 lambda 表达式：几乎所有的观众都举手。 我认为 lambdas 有一个普遍的误解：提供 lambdas 的编程语言的实际功能本身不是 lambdas，是高阶函数和传递方法引用的能力。 Lambdas 只是一种匿名功能。 其实 lambda 是不可测试的，因为它们是硬编码的。 例如，如果我会执行这样的事情：

```kotlin
RouteCaller(0, 100, { pageOffset, limit -> backend.search(searchQuery, pageOffset) })
```

我们如何为 lambda 编写单元测试？ 由于 lambda 是“硬编码”，因此不可能为该 lambda 编写单元测试，而如果我们传递一个函数引用，我们可以很容易地单元测试一个函数。 在 java 8 中，我们可以通过编写这个 `::`  searchCall来传递一个方法引用。 不幸的是，这个语法在 kotlin 中还不被支持（现在 `::` 只支持静态方法）。 因此，这个 “解决方法” 是通过定义一个函数作为属性。 在本系列博客文章的最后部分，更多关于我与 kotlin 的经验。

好，为了执行一个搜索我们有这样的东西：

![](http://hannesdorfmann.com/images/plaid/routing3.png)

请注意，对于搜索 getAllBackendCallers（）返回一个包含一个 RouteCaller 的列表，但是想法是 RouteCallerFactory 将创建所有 RouteCaller 到某个后端的所有可用端点。 正如我们稍后会看到的， HomeDribbbleCallerFactory 为每个 Dribbble 端点返回一个 RouteCallers 列表，以加载热门项目，最近动画等等。所以我们有这样的东西：

![](http://hannesdorfmann.com/images/plaid/routing4.png)

接下来我们介绍一个 Router ，负责将来自不同 RouteCallerFactory 的所有 RouteCaller 组合到一个单一的 Observable 列表中：

![](http://hannesdorfmann.com/images/plaid/routing5.png)

好的，到目前为止，我们已经涵盖了 “路由部分”。 所以最后路由器提供了一个 Observable \<List \<RouteCaller \<T >>> 。 但是，我们什么时候才能最终加载项目，以在 RecyclerView 中显示它们。 这是 ItemsLoader 的责任。 正如名字已经表明这个组件加载项目：

```kotlin
class ItemsLoader<T>(protected val router: Router<T>) {

    fun firstPage(): Observable<T> {
        return FirstPage<T>(router.getAllRoutes()).asObservable()
    }

    fun olderPages(): Observable<T> {
        return OlderPage<T>(router.getAllRoutes()).asObservable()
    }
}
```

ItemsLoader 以 Router 作为构造参数，并提供了两种方法：

- firstPage（）：返回代表第一页的 Observable。
- olderPages（）：返回一个 Observable 加载较旧的页面。

页面代表 RecyclerView 中显示的一个项目页面。如果我们滚动到 RecyclerView 的末尾，我们加载包含较旧项目的下一页。让我们来看看 FirstPage extends Page 和 OlderPage extends Page 类：

```kotlin
abstract class Page<T>(val routeCalls: Observable<List<RouteCaller<T>>>) {

    private var failed = AtomicInteger()
    private var backendCallsCount: Int = 0

    /**
     * Return an observable for this page
     */
    fun asObservable(): Observable<T?> {

        return routeCalls.flatMap { routeCalls ->

            backendCallsCount = routeCalls.size
            val observables = arrayListOf<Observable<T>>()
            routeCalls.forEach { call ->
                    val observable = getRouteCall(call).onErrorResumeNext { error ->
                      // Suppress errors as long as not all fail
                        error.printStackTrace()
                        val fails = failed.incrementAndGet()

                        if (fails == backendCallsCount) {
                            Observable.error(error) // All failed so emmit error
                        } else {
                            Observable.empty() // Not all failed, so ignore this error and emit nothing
                        }
                    }
                    observables.add(observable);
                }

                // return the created Observable
                Observable.merge(observables)
        }
    }

    protected abstract fun getRouteCall(caller: RouteCaller<T>): Observable<T>
}


class FirstPage<T>(routeCalls: Observable<List<RouteCaller<T>>>) : Page<T>(routeCalls) {

    override fun getRouteCall(caller: RouteCaller<T>): Observable<T> {
        return caller.getFirst();
    }
}

class OlderPage<T>(routeCalls: Observable<List<RouteCaller<T>>>) : Page<T>(routeCalls) {

    override fun getRouteCall(caller: RouteCaller<T>): Observable<T> {
        return caller.getOlderWithRetry();
    }
}
```

Page 负责通过调用 RouteCaller 的方法来最终执行 http 调用。 我们也不希望只有一个后台调用失败，整个页面失败。 因此，我们使用 onErrorResumeNext（）来 “拦截” 错误，并且只有在所有 http 调用失败的情况下才返回一个错误。

然后 Presenter 通过 ItemsLoader 订阅 “页面” 被观察者。

## Dependency Injection

您可能已经注意到，到目前为止所描述的几乎所有组件都将其他组件作为构造器参数。这是设计。现在我们用Dagger（我用 Dagger1）来组合这样的元素：

```kotlin
@Module(
    injects = {
        HomePresenter.class
    },
    addsTo = ApplicationModule.class // contains Retrofit interfaces
)
public class HomeModule {

  @Provides @Singleton HomePresenter provideSearchPresenter(SourceDao sourceDao,
      DribbbleService dribbbleBackend,
      DesignerNewsService designerNewsBackend,
      ProductHuntService productHuntBackend ) {

    // Create the router
    List<RouteCallerFactory<List<? extends PlaidItem>>> routeCallerFactories = new ArrayList<>(3);
    routeCallerFactories.add(new HomeDribbbleCallerFactory(dribbbleBackend, sourceDao));
    routeCallerFactories.add(new HomeDesingerNewsCallerFactory(designerNewsBackend, sourceDao));
    routeCallerFactories.add(new HomeProductHuntCallerFactory(productHuntBackend, sourceDao));

    Router<List<? extends PlaidItem>> router = new Router<>(routeCallerFactories);

    ItemsLoader<List<? extends PlaidItem>> itemsLoader = new ItemsLoader<>(router);

    return new HomePresenterImpl(itemsLoader);
  }
}
```

正如你所看到的，我们可以使用 Dagger 来配置我们的 Router 和 ItemsLoader。 对于 SearchPresenter，我们在 SearchModule 中配置一个 ItemsLoader 和 Router。 好处是，如果我们有一天想添加另一个 “源”（如 reddit）来显示来自 reddit 的项目，我们唯一要做的就是定义一个 RedditService（Retrofit），一个 RedditCallerFactoy 并将这个 CallerFactory 添加到 路由器。 我们可以在具体的匕首模块中做到这一点，而不必触摸另一个已经存在的组件的源代码（[Open-Closed 原则](https://en.wikipedia.org/wiki/Open/closed_principle)）。 换句话说，我们已经构建了一个可通过依赖注入进行配置的 “插件系统”。

您可能已经注意到上面显示的代码中的 SourceDao 类。我们将在这个博客系列的第二部分讨论这个问题，当我们要“真正反应”的时候。

## 结束

这是一系列博文的第一部分。 在第一部分中，我们通过应用 Model-View-Presenter 构建了基础，并重构了应用程序如何从后端端点加载数据的方式。 其主要思想是将这个庞大而复杂的任务分解为几个较小的可重用组件，如ItemsLoader，Page，Router 和 RouteCaller，它们更像 Nick Butcher 的 DataManager 实现一样遵循 SOLID 原则。

像往常一样，有更好的方法来实现这样一个应用程序。 尤其是，ItemsLoader 可以完全不同。 我的第一个意图是通过使用 RxJava 的 switchOnNext（）或者合并运算符（如 MatthiasKäppler 所[描述](https://gist.github.com/mttkay/24881a0ce986f6ec4b4d)的）来创建一个无限的 Observable 来加载较旧的页面，但是我得出的结论是，如果可以的话，有关 UI 和错误处理的一些事情稍微容易实现 将单个观测值拆分为两个被观察者（一个第一页面，一个旧页面）。

一如以往，反馈和建议是非常受欢迎的！

在关于重构 plaid 应用程序的这一系列博客文章的下一部分（[第二部分](http://hannesdorfmann.com/android/plaid-refactored-2)）中，我们将通过使用 RxJava 的真正力量来“真正反应”。敬请关注。 

更新：[第二部分在线](http://hannesdorfmann.com/android/plaid-refactored-2)。



