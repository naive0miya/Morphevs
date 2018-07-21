# 重构 plaid (第2部分) - 一种反应性 MVP 方式

> 原文：[REFACTORING PLAID APP - A REACTIVE MVP APPROACH (PART 2)](http://hannesdorfmann.com/android/plaid-refactored-2)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

这是我们如何重构由 Nick Butcher 开源的 [plaid](https://github.com/nickbutcher/plaid) 应用程序的第二部分。 在这一部分，我们将要加强[第一部分](http://hannesdorfmann.com/android/plaid-refactored-1)中描述的 MVP 架构，使其成为真正的反应。

前言：我开始重构，并坚信我可以重构整个应用程序。 令人惊讶（讽刺），事实证明，我是一个非常天真的男人。 我只是低估了应用程序的复杂性和功能的数量。 因此，即使这里描述的不是所有的东西都被实现（重构），我也开始写博客文章。 我只想提出一些如何实施的想法。 我的重构代码也可以在 [github](https://github.com/sockeqwe/plaid) 上找到。

## 从第一部分回顾

这是关于重构格子应用程序的博客系列的第二部分。 在第一部分中，我们应用了 Model-View-Presenter，并通过调用 RouteCallers 引入了一个用于从不同后端端点加载项目的 ItemsLoader。 请阅读第一部分的更多细节。

如果有需要另一个 MVP 博客文章的话，人们也会问我，因为已经有很多这样的博客文章了。 那么，我的意图不是写 MVP。 相反，我想讨论如何建立一个 “真正的反应” 的应用程序（MVP顶部）。 让我们继续我们离开的地方。

## Source - Filter

在应用程序的主屏幕中，您可以在屏幕右侧打开抽屉（或通过单击工具栏中的过滤器图标）来启用/禁用我们从中加载项目的不同后端端点。 我们称之为后端端点 Source。 所以我们有 “Dribbble Popular”，“Dribbble Recent”，“Designer News Popular” 等来源。

![](http://hannesdorfmann.com/images/plaid/source-filters.png)

如果您使用 SharedPreferences 来保存大量数据，而不仅仅是 “用户首选项”，请举手。 我绝对有罪，使用 SharedPreferences 这种东西，Nick Butcher 也用于存储来源。 问题是用户可以添加（我们将在一分钟内讨论这个），更新（启用/禁用）和删除源。 听起来，一个数据库更适合，对吗？

## Database

让我们重构一下，用 SQLiteDatabase 替换 SharedPreferences。数据库很难。所以我们可以写纯 sql 语句或选择合适的库来处理数据库。那里有很多库，有些库比其他库要好。由于我们使用 RxJava，所以我们当然希望使用具有一流 RxJava 支持的库。在我看来，ORM不是正确的路要走。为什么？因为通常 ORM 必须被设计成在平均用例上运行良好。但几乎每个应用程序都有一个特殊的用例。例如，编写一个解析与其他对象（SQL连接？）的关系的高效查询在原生 SQL 中已经足够困难了。但是如何在使用某个 ORM 库时做到这一点？您必须花费大量时间来了解 ORM 库如何编写有效的查询。每当您更改 ORM 库时，您都必须反复学习新的 ORM 库。因此，我宁愿在没有 ORM 库的情况下工作，编写原始的 SQL 查询，我唯一需要了解的就是 SQL。

因此，我们将使用 Square 中的 [SQLBrite](https://github.com/square/sqlbrite)，它是一个支持 RxJava 的 SQLiteOpenHelper 轻量级包装器。 我们不想处理游标和 ContentValues。 因此，我们使用 [SQLBrite-DAO](https://github.com/sockeqwe/sqlbrite-dao)，它提供了一些诸如 DAO（数据访问对象）的抽象和基于微型对象映射器（光标到对象，而不是 ORM）和 ContentValues 生成器的注释处理器。 如果您正在寻找具有类似功能的更高级别的解决方案，您应该查看一下 [StorIO](https://github.com/pushtorefresh/storio)，但是我更加低级，所以我们将使用 SQLBrite + SQLBrite-DAO。

我们这样定义一个类的来源：

```kotlin
@ObjectMappable
class Source() { // Unfortunately data class are not supported yet by sqlbrite-dao

    object ID { // ID's for predefined Sources
        const val UNKNOWN_ID = -1L
        const val DESIGNER_NEWS_POPULAR = -2L
        const val DESIGNER_NEWS_RECENT = -3L
        const val DRIBBBLE_POPULAR = -4L
        const val DRIBBBLE_FOLLOWING = -5L
        const val DRIBBLE_MY_SHOTS = -6L
        const val DRIBBLE_MY_LIKES = -7L
        const val DRIBBLE_RECENT = -8L
        const val DRIBBLE_DEBUTS = -9L
        const val DRIBBLE_ANIMATED = -10L
        const val DRIBBLE_MATERIAL = -11L
        const val PRODUCT_HUNT = -12L
    }

    @Column(SourceDao.COL.ID)
    var id: Long = ID.UNKNOWN_ID

    @Column(SourceDao.COL.ORDER)
    var order: Int = 0

    @Column(SourceDao.COL.ENABLED)
    var enabled: Boolean = false

    @Column(SourceDao.COL.AUTH_REQUIRED)
    var authenticationRequired = false

    @Column(SourceDao.COL.BACKEND_ID)
    var backendId: Int = -1

    @Column(SourceDao.COL.NAME)
    var name: String? = null

    @Column(SourceDao.COL.NAME_RES)
    var nameRes: Int = -1
  }
```

@ObjectMappable 和 @Column 是来自 SQLBrite-DAO 注释处理器的注释。接下来我们定义一个 DAO 来处理和从数据库中查询 Source，如下所示：

```kotlin
class SourceDao : Dao() {

    object COL {
        const val ID = "id"
        const val ORDER = "orderPosition"
        const val ENABLED = "enabled"
        const val AUTH_REQUIRED = "authRequired"
        const val BACKEND_ID = "backendId"
        const val NAME = "name"
        const val NAME_RES = "nameRes"
    }

    private val TABLE = "Source"

    override fun createTable(database: SQLiteDatabase) {
        CREATE_TABLE(
                TABLE,
                "${COL.ID} INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL",
                "${COL.ENABLED} BOOLEAN",
                "${COL.AUTH_REQUIRED} BOOLEAN",
                "${COL.ORDER} INTEGER",
                "${COL.BACKEND_ID} INTEGER NOT NULL",
                "${COL.NAME} TEXT",
                "${COL.NAME_RES} INTEGER")
                .execute(database)
    }

    fun getAllSources(): Observable<List<Source>> {
        return defer {
            query(
                    SELECT(COL.ID, COL.ENABLED, COL.ORDER, COL.BACKEND_ID, COL.NAME, COL.NAME_RES)
                      .FROM(TABLE)
                      .ORDER_BY(COL.ORDER)
            ).run()
            .mapToList(SourceMapper.MAPPER) // annotation processor generates that
        }
    }

    fun getSourcesForBackend(backendId: Int): Observable<List<Source>> {
        return defer {
            query(
                    SELECT(COL.ID, COL.ENABLED, COL.ORDER, COL.BACKEND_ID, COL.NAME, COL.NAME_RES)
                      .FROM(TABLE)
                      .WHERE("${COL.BACKEND_ID} = ?")
                 ).args(backendId.toString())
                 .run()
                 .mapToList(SourceMapper.MAPPER)
        }
    }

    fun insert(source: Source): Observable<Long> {

      val builder = SourceMapper.contentValues().
                      .id(source.id)
                      .enabled(source.enabled)
                      .name(source.name)
                      .nameRes(source.nameRes)
                      .order(source.order)
                      .authenticationRequired(source.authenticationRequired)
                      .backendId(source.backendId)
                      .build()

        return defer { insert(TABLE, cv)
    }

    fun enableSource(sourceId: Long, enabled: Boolean): Observable<Int> {

        val cv = SourceMapper.contentValues()
                .enabled(enabled)
                .build()

        return defer { update(TABLE, cv, "${COL.ID} = ?", sourceId.toString()) }
    }
}
```

SQLBrite-DAO 提供了一个简单的 SQL 语法，因此 IDE 中的自动完成工作，注释处理器生成一个类 SourceMapper 处理光标和 ContetnValues 和 SQLBrite 包装到 RxJava 中，所以我们可以使用 Observable。

## SourceFilterFragment 中的 MVP

在当前的实施中，具有源列表的右侧抽屉被集成在 HomeActivity 中。 我们将重构这个，并在这里应用 MVP。 我们实现 SourceFilterFragment implmenets SourceFilterView 和 SourceFilterPresenter，这样订阅 SourceDao：

```kotlin
class SourceFilterPresenter(val sourceDao: SourceDao, val presentationModelMapper: (List<Source>) -> List<SourceFilterPresentationModel>) : RxPresenter<SourceFilterView, List<SourceFilterPresentationModel>>() {

    fun loadSources() {

        view?.showLoading(false)

        subscribe(
                sourceDao.getAllSources().map(presentationModelMapper),
                // onError
                {
                    view?.showError(it, false)
                },
                // onNext
                {
                    view?.setData(it)
                    view?.showContent()
                }
        )
    }

    fun changeEnabled(source: SourceFilterPresentationModel) {

        val observable = sourceDao.enableSource(source.sourceId, !source.enabled)

        subscribe(observable,
                { // On Error
                    view?.showError(it, true)
                }) // OnNext not needed

    }
}
```

MVP 的初衷是 Presenter 将模型转换成将在视图中显示的 PresentationModel。我们不在 SourceFilerView 中显示 List，而是在 SourceFilterPresentationModel 中显示。此演示者模型针对视图进行了优化。我们总是需要 MVP 中的 PresentationModel 吗？这取决于。如果你的模型只是一个 POJO，那么直接在视图中显示模型应该没问题，但是请不要将应用层模型“泄漏”到视图中。不久或将来，您将从视图中调用应用程序层模型的方法。通过使用 PresentationModel，视图完全不了解应用程序层模型。然而，我们使用 SourceFilterPresentationModel，因为我们的问题是我们的 Source 根本没有准备好显示在 RecyclerView 中。 Source 类可以有一个 int nameRes（R.string.something），如果它是一个预定义的 Source 或一个 String 名称，如果 Source 已经被应用程序的用户添加了（稍后更多）。此外，Source 类只包含一个 backendId，但我们想要显示一个图标和标题的单元格，如下所示：

![](http://hannesdorfmann.com/images/plaid/source-filter-item.png)

要显示这样一个项目，我们必须将后端标识符映射到一个图标上，并将标题 “map” nameRes 映射到一个字符串或使用名称作为标题。当然，你可以在 RecyclerView 的 onBindViewHolder（）适配器中这样做，但是适配器的复杂性会增加。而且，如果你在 BindViewHolder（）方法的适配器中这样做，这个 “映射” 将在 android 主要 UI 线程上完成，并在滚动期间执行。这在我们的用例中可能不是问题，但是如果你必须做一些更复杂的事情，比如排序元素或者计算一些要显示的属性，那么你可能会遇到麻烦。因此，我们传递一个函数 presentationModelMapper：（List <Source>） - > List <SourceFilterPresentationModel> 作为 SourceFilterPresenter 的构造函数参数，然后使用 RxJava 的 map（）运算符将 List <Source> 映射到 List <SourceFilterPresentationModel>。 RxJava 的另一个好处是它是线程模型。实际上，我们在查询数据库的后台线程上进行异步映射，然后在主 UI 线程上给出视图的结果 PresentationModel。这个映射函数看起来像这样：

```kotlin
class BackendManager {

    object ID {
        const val DRIBBBLE = 0
        const val DESIGNER_NEWS = 1
        const val PRODUCT_HUNT = 2
    }

    val getBackendIconRes = fun(backendId: Int) = when (backendId) {
        ID.DRIBBBLE -> R.drawable.ic_dribbble
        ID.DESIGNER_NEWS -> R.drawable.ic_designer_news
        ID.PRODUCT_HUNT -> R.drawable.ic_product_hunt
        else -> throw IllegalArgumentException("Unknown Backend for id = ${backendId}")
    }
}

// We pass BackendManager.getBackendIconRes() as backendToIconMap function
class SourceToPresentationModelMapper(private val context: Context, private val backendToIconMap: (Int) -> Int) : (List<Source>) -> List<SourceFilterPresentationModel> {

    override fun invoke(sources: List<Source>): List<SourceFilterPresentationModel> {

        val presentationModels = ArrayList<SourceFilterPresentationModel>()
        sources.forEach { source ->

            val name =
                    if (source.name == null) {
                        context.resources.getString(source.nameRes)
                    } else {
                        source.name!!
                    }

            presentationModels.add(SourceFilterPresentationModel(source.id, backendToIconMap(source.backendId), name, source.enabled))
        }
        return presentationModels
    }
}
```

这是另一种在 kotlin 中定义函数（作为类）的方法。我只是想和 kotlin 玩一下。

## 让我们 "反应"

许多开发人员对 Rx 编程感到兴奋，因为Rx编程提供像 flatMap（）等功能类似的操作符。他们中的许多人不明白，通过使用 RxJava 他们正在观察数据。 因此，他们根本不了解 Rx 编程。 这不仅是关于数据转换。 Rx 实现观察者模式。 您正在订阅观察者以获取更新。 这可能是一个像 http 响应一样的可消耗数据流，但是观察者模式和 RxJava 的真正威力可以通过使数据源可观察到，可以不止一次地发射项目来看到和使用。

## Observable Database

我们还没有讨论太多关于 SQLBrite 的问题，但是 SQLBrite 的最大优点是每当我们改变一个数据库表的数据集（插入，更新或删除行）时，我们都会通过 SQLBrite 和 RxJava 得到通知。 SourceFilterPresenter.loadSources（） 订阅 sourceDao.getAllSources（），它返回一个 Observable \<List \<Source>。 但是，这不仅仅是一个 “单一的” 可观察的运行查询，并完成查询结果一旦发射。 不，这个 Observable 将保持订阅（直到取消订阅），SQLBrite 将识别表更改，并简单地重新运行 SQL 查询，并用新的查询结果集调用 onNext（）。

在订阅 SourceDao 时，我们还没有讨论 SourceFilterPresenter 是如何工作的：

1. 当 SourceFilterFragment 启动 SourceFilterPresenter 的实例化并且 SourceFilterPresenter.loadSources（） 被调用。 这将调用 SourceDao.getAllSources（），它基本上运行 SELECT  FROM Source 并返回一个 Observable。 Presenter 在这个 observable 上订阅自己，onNext（）被 sql 查询结果调用。 演示者然后告诉视图显示源（PresentationModel），即在 RecyclerView 中。
2. 当用户单击 RecyclerView 中的 “源项目” 来启用或禁用此源时，视图将调用 presenter.changeEnabled（）。这将基本上做像更新数据库 UPDATE Source SET enabled = ? WHERE id = ?
3. SQLBrite 将认识到表 “Source” 的数据集已被更新（步骤2）。 因此，SQLBrite 会检查在同一个表上是否存在与 Subscribers 相关的 Observable，并检测到步骤1（SELECT  FROM Source）中的 Observable 仍然“活着”。 因此，SQLBrite 将重新运行此 SQL 查询并发出新的查询结果（使用更新的源）。 演示者在订户onNext（）中接收到新的结果，然后更新视图。

需要注意的一个重要的事情是，通过单击项目上的 RecyclerView 来启用/禁用 Source，适配器中的 Source 对象本身将不会被更新。 如上所述，查询将重新运行并发出一个全新的 List \<Source> ，然后将被映射到一个全新的 List \<SourceFilterPresentationModel>，然后将显示在 RecyclerView 中（取代之前的 List）。 这是设计！ 通过这样做，这是确保拥有不可变对象的第一步。

不可变性防止别人触摸你的对象。你可以把它想象成在体育场看棒球比赛。你坐在论坛中间的某个地方。随着游戏的进行，你饿了。幸运的是附近有一家卖热狗的卖主，但是他离你一些座位。不过，你点了一个热狗。供应商会把你的热狗给你正在坐的那一排的第一个人，他会把它传递给他的邻居，那个人会把你的热狗传给他的下一个邻居，等等，直到你终于得到你的热狗。我敢肯定，你不希望别人在从供应商到你的途中吃了一块热狗。你想要一个不变的热狗！对我们的数据对象同样有效。我们不希望其他组件（特别是其他线程）可以更改我们的数据对象。你可能已经注意到这意味着我们的 Source 类不应该有 setter，但实际上已经有了。这是我的错误与 SQLBrite-DAO 的组合，尚不完全支持 kotlin，并需要一个公开的 setter 方法为他的注释处理器。如果你在一个真正的应用程序工作，我建议使用谷歌的 [auto-value](https://github.com/google/auto/tree/master/value) 来创建不可变的对象。

## Adding a Source

如果你打开搜索，你可以 “保存” 这个搜索（点击 fab）。这意味着我们使用搜索字符串作为名称创建一个 “Source”（因此 Source 可以具有 nameRes = R.string.foo 或 name =“Hello”）：[plaid app - adding source](https://www.youtube.com/watch?v=PPwNV45L3T0)

SQLBrite + RxJava下实际发生的事情与之前描述的相同（启用/禁用一个 Source）：

![](http://hannesdorfmann.com/images/plaid/inserting-source.png)

## Reactive Routing

正如已经说过的：这是关于重构格子应用的博客系列的第二部分。 在第一部分中，我们应用了 Model-View-Presenter，并通过调用 RouteCallers 引入了一个用于从不同后端端点加载项目的 ItemsLoader。 如果你还没有更多的细节，请阅读第一部分。 总结一下，主屏幕是这样构建的：

![](http://hannesdorfmann.com/images/plaid/part1-recap.png)

请注意，我们使用 RxJava 建立了一个可观察的单向自底向上数据流。 我们已经讨论过将数据源存储在数据库中，用户可以动态地启用和禁用数据源。 只要用户在 SourceFilterView 中启用/禁用 Source，我们必须重新加载 HomeView 中的项目，对吧？ 我们已经看到 HomePresenter 负责通过使用 ItemsLoader 加载数据项。 那么我们如何通知 HomePresenter 已经启用/禁用了一个源？ 在这种情况下使用 EventBus 是一个常见的解决方案。

但是有一个更好的方法，一个真正被动的方法：我们不必告诉 HomePresenter 变化。怎么样？那么， HomePresenter 已经在观察 ItemsLoader 了。所以 HomePresenter 并不关心 “源变化”。所有 HomePresenter 感兴趣的是从他的 onNext（）订阅者回调接收项目，并通过 HomeView 显示它们。所以当一个源被更改时，将显示新的项目以显示。容易（理论上），对吗？但是，实际上需要做些什么呢？好消息：我们已经拥有几乎所有我们需要的东西。 RouteCallerFatory.getAllBackendCallers（）已经返回一个 Observable \<List \<RouteCaller >>（请注意这是一个 Observable）。路由器将这个 Observable 传递给 ItemsLoader 并且 HomePresenter 在其上订阅。因此，如前所述，要将新项目带到 HomePresenter 的 onNext（）回调函数中，我们必须发出新项目。哪种产品？ New List \<RouteCaller> 因为每个 RouteCaller 将由 ItemsLoader（Page）执行以从后端端点加载项目，并最终将加载的项目发送到 HomePresenter。换句话说，为了使路由 “被动”，我们必须通过观察数据库（SQLBrite）并在启用或禁用源时发出更新的 RouteCaller 来使 RouteCallerFatory “被动”

```kotlin
class HomeDribbbleCallerFactory(private val backend: DribbbleService, sourceDao: SourceDao) : RouteCallerFactory<List<PlaidItem>> {

    private val backendCalls = ArrayMap<Long, RouteCaller<List<PlaidItem>>>()
    private val sources: Observable<List<Source>>

    init {
        // Observable for the Database
        sources = defer {
            sourceDao.getSourcesForBackend(BackendManager.ID.DRIBBBLE).share()
        }
    }

    private fun createCaller(source: Source): RouteCaller<List<PlaidItem>> {
        return RouteCaller(0, ITEMS_PER_PAGE, getBackendMethodToInvoke(source))
    }

    // Find the method to invoke (retrofit backend endpoint call) for a given Source
    private fun getBackendMethodToInvoke(source: Source):
            (pageOffset: Int, itemsPerPage: Int) -> Observable<List<PlaidItem>> = when (source.id) {
        // Predefined Sources
        Source.ID.DRIBBBLE_POPULAR -> getPopular
        Source.ID.DRIBBBLE_FOLLOWING -> getFollowing
        Source.ID.DRIBBLE_ANIMATED -> getAnimated
        Source.ID.DRIBBLE_DEBUTS -> getDebuts
        Source.ID.DRIBBLE_RECENT -> getRecent
        Source.ID.DRIBBLE_MY_LIKES -> getMyLikes
        Source.ID.DRIBBLE_MY_SHOTS -> getUserShots

        // Custom Source created by user by searching.
        else -> SearchFunc(source.name!!)
    }

    override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
        return sources.map(mapSourcesToBackendCalls)
    }

    // Map Sources from Database to corresponding RouteCaller
    val mapSourcesToBackendCalls = fun(sources: List<Source>): List<RouteCaller<List<PlaidItem>>> {
        val calls = ArrayList<RouteCaller<List<PlaidItem>>>()
        sources.forEach { source ->

            val call = backendCalls[source.id]

            if (call == null) {
                // New source added
                if (source.enabled) {
                    val newCall = createCaller(source)
                    backendCalls.put(source.id, newCall)
                    calls.add(newCall)
                }

            } else {
                // Already existing source

                if (!source.enabled) {
                    // Source has been disabled
                    backendCalls.remove(source.id)
                } else {
                    calls.add(call)
                }
            }
        }

        return calls
    }


    //
    // Backend endpoint calls
    //

    val getPopular = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getPopular(pageOffset, itemsPerPage)
    }

    val getFollowing = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getFollowing(pageOffset, itemsPerPage)
    }

    val getAnimated = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getAnimated(pageOffset, itemsPerPage)
    }

    val getDebuts = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getDebuts(pageOffset, itemsPerPage)
    }

    val getRecent = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getRecent(pageOffset, itemsPerPage)
    }

    val getMyLikes = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getUserLikes(pageOffset, itemsPerPage)
    }

    val getUserShots = fun(pageOffset: Int, itemsPerPage: Int): Observable<List<PlaidItem>> {
        return backend.getUserShots(pageOffset, itemsPerPage)
    }

    private class SearchFunc(val queryString: String) : (Int, Int) -> Observable<List<PlaidItem>> {

        override fun invoke(pageOffset: Int, pageLimit: Int): Observable<List<PlaidItem>> {

            return Observable.defer<List<PlaidItem>> {
                try {
                    // Dribbble API doesn't provide a search endpoint.
                    // Therefore, parse HTML search result manually
                    Observable.just(DribbbleSearch.search(queryString, DribbbleSearch.SORT_RECENT, pageOffset))
                } catch(e: Exception) {
                    Observable.error(e)
                }
            }
        }
    }

}
```

正如你所看到的，我们的 HomeDribbbleCallerFactory 是观察我们的数据库（sourceDao.getSourcesForBackend（）），每当数据库已经被更改，因为用户启用/禁用一个源或添加了一个新的（自定义搜索）sources.map（mapSourcesToBackendCalls）将 被再次调用，并发布一个新的 List <RouteCaller> 与更新的路由来调用。 接下来，ItemsLoader（通过路由器和 FirstPage）将对新发布的 List <RouteCaller> 作出反应，并加载新的项目，最终触发 HomePresenter 的 onNext（）回调与新的加载项目。 这听起来比实际上更复杂。 看看下面的图形表示：[reactive plaid](https://www.youtube.com/watch?v=pmWjwrLVDdA)

黄色圆圈表示数据流。 您会看到 SQLBrite 将重新运行所有查询并通知订阅者。 由于 HomeDribbbleCallerFactory 是数据库的订户，因此在启用/关闭源时会通知此组件，并自动调整路由器。 感谢 RxJava 和 SQLBrite，我们可以构建一个真正的被动应用程序，让应用程序对变化做出反应，并在最后通过仍然有单向数据流（不太可能使用EventBus）“神奇地” 更新 UI。

## Posting items to DesignerNews

如果您还没有注意到：您可以直接从格子应用程序（主屏幕上的浮动动作按钮）向 DesignerNews 站点提交新闻。 Nick Butcher 使用 IntentService 执行 http 调用，并使用 LocalBroadcastReceiver（EventBus 类型）来通知 HomeActivity 故事是否已成功提交：

![](http://hannesdorfmann.com/images/plaid/posting-old.png)

使用 android 服务绝对是确保故事将从活动生命周期独立发布的途径。但是，当前的实现有一个小问题：如果用户提交 PostStoryService 启动，但是这个服务只会在成功上传之后通知 HomeActivity，然后 HomeActivity 会在 RecyclerView 中显示已上传的项目，而另一个则上传。问题是，如果你有一个缓慢的互联网连接（即执行 http 调用提交到设计器新闻需要10秒），用户没有线索或视觉反馈发生了什么事情。一个好主意是在项目布局（ViewHolder）中用 ProgressBar 在 HomeActivity 的 RecyclerView 中立即显示这个项目，并且只要这个 ProgressBar 被成功提交（http 调用成功）就隐藏 ProgressBar。如果出现错误，如果 HomeActivity 的 RecyclerView 中的失败项目将突出显示一个错误图标和一个重试按钮，将会很好。更好的办法是添加脱机支持（无网络连接）。听起来很复杂，不是吗？你如何实现？使用 EventBus？

别担心，我们可以以 “真正的被动” 的方式实现，而不需要额外的工作来将其整合到我们已经重构的代码中。 让我们一步一步来做。 首先，我们必须实现离线支持，所以我们必须在本地设备上以本地方式保存故事：我们使用SQLite 数据库和 SQLBrite + SQLBrite-DAO。 我们来定义一个简单的数据类 NewDesignerNewsStory，它代表了我们将在以后发布在 Designer News 上的一个故事：

```kotlin
@ObjectMappable
class NewDesignerNewsStory : PlaidItem {

    object State { // ID's for predefined Sources
        const val NOT_SUBMITTED = 0
        const val IN_PROGRESS = 1
        const val FAILED = 2
    }

    @Column(StoryDao.COL.ID)
    var id : Long

    @Column(StoryDao.COL.TITLE)
    var title: String

    @Column(StoryDao.COL.COMMENT)
    var comment: String

    @Column(StoryDao.COL.URL)
    var url: String

    @Column(StoryDao.COL.STATE)
    var state = State.NOT_SUBMITTED
  }
```

此外，我们实现了 StoryDao，它负责插入一个 NewDesignerNewsStory，更新一个 NewDesignerNewsStory 的状态。 我不打算显示代码，因为我猜你知道这个 SQL 语句是怎么样的。 接下来，我们将重构 PostStoryService 以使用 StoryDao 查询未提交的帖子的本地数据库并发布它们：

```kotlin
class PostStoryService : IntentService ("PostStoryService") {

   @Inject lateinit var storyDao : StoryDao
   @Inject lateinit var backend : DesignerNewsService

   fun onHandleIntent(i : Intent) {
      val toSubmit  = storyDao.getAllNotInProgress() // List<StoryDao> of NOT_SUBMITTED and FAILED
      toSubmit.forEach(submit)
   }

   private val submit = fun(story : NewDesignerNewsStory){
     try {
       storyDao.setUploadInProgress(story.id) // Mark Story as in progress
       backend.postNewStory(story)  // Blocking retrofit call
       storyDao.remove(story.id) // successfully so remove this from database
     } catch (e : IOException){
       storyDao.setFailed(story.id)
     }
   }
}
```

Nick Butcher 已经实现了一个 PostNewDesignerNewsStoryActivity 来让用户界面提交一个故事。 我们也会重构这个，并且像以前那样应用 MVP。 所以最后会有一个 PostNewDesignerNewsStoryActivity（View），一个 PostNewDesignerNewsStoryPresenter，它将通过使用 StoryDao 将 NewDesignerNewsStory 保存到本地数据库中。 这个想法是将格子应用程序用户想要提交到数据库中的每个新故事存储起来，而不是直接执行 http 调用，然后启动 PostStoryService 来执行 http 调用。 正如你在代码中所看到的那样，我们将在尝试提交故事的同时更新状态（IN_PROGRESS / FAILED）并将其永久保存到数据库中。 简而言之：

![](http://hannesdorfmann.com/images/plaid/offline-support.png)

好吧，但现在你可能会问自己：我们在 HomeActivity 中如何从数据库中显示这些项目？ 那么，这比你想象的要容易得多。 我们已经实现了一个我们已经在 HomeActivity 中使用的非常灵活的路由机制。 所以我们只需要添加另一条路由到我们的路由器：但是，然后把 http 调用到后端，我们添加一个路由到我们的本地（离线）数据库。 所需要的是定义一个 RouteCallerFactory：

```kotlin
class OfflineStoryCallerFactory (private val storyDao : StoryDao) : RouteCallerFactory<List<PlaidItem>>{

  private val callers = arrayListOf(RouteCaller(0, 100, callFunc))

  private val callFunc = fun (pageOffset : Int, limit : Int) : Observable<List<PlaidItem>> {
    return storyDao.getAll()
  }

  override fun getAllBackendCallers(): Observable<List<RouteCaller<List<PlaidItem>>>> {
    return callers
  }

}
```

容易吗？让我们将 OfflineStoryCallerFactory 添加到用于主屏幕的路由器。在第1部分中，我们已经讨论过，我们可以在相应的匕首模块中进行配置：

```kotlin
@Module(
    injects = {
        HomePresenter.class
    },
    addsTo = ApplicationModule.class // contains Retrofit interfaces
)
public class HomeModule {

  @Provides @Singleton HomePresenter provideSearchPresenter(SourceDao sourceDao,
      StoryDao storyDao,
      DribbbleService dribbbleBackend,
      DesignerNewsService designerNewsBackend,
      ProductHuntService productHuntBackend ) {

    // Create the router
    List<RouteCallerFactory<List<? extends PlaidItem>>> routeCallerFactories = new ArrayList<>(3);
    routeCallerFactories.add(new HomeDribbbleCallerFactory(dribbbleBackend, sourceDao));
    routeCallerFactories.add(new HomeDesingerNewsCallerFactory(designerNewsBackend, sourceDao));
    routeCallerFactories.add(new HomeProductHuntCallerFactory(productHuntBackend, sourceDao));

    // Add the new route to local database
    routeCallerFactories.add(new OfflineStoryCallerFactory(storyDao));

    Router<List<? extends PlaidItem>> router = new Router<>(routeCallerFactories);

    ItemsLoader<List<? extends PlaidItem>> itemsLoader = new ItemsLoader<>(router);

    return new HomePresenterImpl(itemsLoader);
  }
}
```

所以最后我们有一个 HomePresenter，它从一个 ItemsLoader 获得项目，这使得他可以通过 Dribbble（HomeDribbbleCallerFactory），Designer News（HomeDesingerNewsCallerFactory），Product Hunt（HomeProductHuntCallerFactory） 以及用户本地（离线）数据库加载和显示项目 OfflineStoryCallerFactory。 由于 PostStoryService，PostNewDesignerNewsStoryPresenter 和 OfflineStoryCallerFactory 使用建立在 SQLBrite + RxJava 之上的 StoryDao，因此整个事件是被动的。 这意味着，只要使用 StoryDao 将新的 Story 添加到本地数据库，HomePresenter 也会自动得到通知。 启用/禁用源时，与之前所述的原理相同。

![](http://hannesdorfmann.com/images/plaid/posting-new.png)

另请注意，PostStoryService 正在改变 NewDesignerNewsStory 的状态，这将导致更新 HomeView 的 UI（显示/隐藏将被上传的项目上的 ProgressBar）。 我们也得到了免费的错误处理：即当一个 NewDesignerNewsStory 不能被提交给 Designer News 后端时，我们可以在 HomeActivity 的 RecyclerView 中显示的这个项目上显示一个重试按钮。 通过点击重试按钮，我们可以再次启动 PostStoryService，它将尝试再次上传该项目。 请注意，我们仍然有一个单向的数据流。 我们可以通过在网络连接不好的情况下以指数形式启动 PostStoryService，或者使用 JobSchedulers 或 GcmNetworkManager 仅在有意义的情况下启动 PostStoryService（节省电池）来改善这一点。

## 结束

在这一部分中，我们已经展示了 RxJava（或一般的 Rx 编程）不仅仅是连接 Retrofit http 调用和转换数据。观察者模式并不是什么新鲜事，但这就是 RxJava 如此美丽的原因。建立一个单向的数据流是清除和方便调试和维护体系结构的关键。这可能是使用 RxJava 基本 Observable（再一次：OBSERVER PATTERN）和一个 EventBus 之间的最大区别，它可以完成这项工作，但是您可能没有单向数据流。我们也可以将反应模式应用于我根本没有涉及的认证。

请注意，源代码远非完美，我敢肯定，你会发现错误。所以，请不要盲目地将这些代码复制并粘贴到您的应用程序中。我根本没有时间来覆盖所有的情况并重构所有的东西。我不得不发布那个博客帖子来阻止我花费更多的时间和日子来重构这个应用程序。你可以看到在这个博客中发布的想法和代码，作为一种“真正被动”的应用程序的“概念验证”。我想再次指出不变对象和[纯函数](https://en.wikipedia.org/wiki/Pure_function)的重要性。不幸的是我的源代码并不像应该那样不可变。如前所述，重点是写一个“反应”的应用程序。

不过，我愿意清理我的代码，并向 Nick Butcher 的原始存储库提出请求。但是，我不认为这个 “反应性 MVP 方法” 将被合并到格子应用程序源代码中。我明白，我的方法是为高级开发人员。此外，我使用一些第三方库和 kotlin，这可能不是每个人都知道的。因此，我的代码并不适合所有类型的开发人员，毫无疑问，作为 Google 员工的 Nick Butcher 希望通过他的应用程序来实现。有这样一个开放源代码的应用程序，这对于 Android 初学者和专家来说都是可以理解的鼓舞人心的 UI / UX 是很重要的。

我也想抓住这个机会，谈谈我脑海里想了将近两年的想法：将 Android 应用程序的不同来源绑定到一个应用程序中，将会非常好。例如，应用程序可以显示一些推特和 Google+ 帖子，如 Jesse Wilson, Jake Wharton, Matthias Käppler, Piwai, Dan Lew, Corey Latislaw, Cyril Mottier, Lisa Wray, Artem Zinnatullin 和其他人（原谅我，如果我有忘记在这里提到你的名字），但也有一些像 [/r/androiddev](https://www.reddit.com/r/androiddev/)，Stackoverflow，YouTube 频道，播客等其他资源捆绑成一个很好的应用程序与优秀的 UI / UX 格式。我不认为这是一个单一的开发人员可以在业余时间开发的东西（重构格子应用程序的经验给我的权利）。但是如果我们能够找到一小部分开发者参与社区为社区开发的这样一个应用程序，那么我肯定也愿意为之作出贡献。

在关于重构格子应用程序的这一系列博客文章的下一部分（第三部分和最后一部分）中，我将在重构期间与 kotlin 分享我的经验，并且我们将讨论如何测试这样的应用程序，因为不仅 [#perfmatters](https://twitter.com/hashtag/perfmatters?src=hash)，而且 [#qualitymatters](http://artemzin.com/blog/android-development-culture-the-document-qualitymatters/)

