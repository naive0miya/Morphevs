# 一个简洁架构代码库的指南

> 原文 (Medium)：[A Guided Tour inside a clean architecture code base.](https://proandroiddev.com/a-guided-tour-inside-a-clean-architecture-code-base-48bb5cc9fc97)
>
> 作者：[Yossi Segev](https://proandroiddev.com/@yossisegev?source=post_header_lockup)

[TOC]

最近，我发布了一个名为 [MovieNight](https://github.com/mrsegev/MovieNight) 的开源示例项目。 在过去的几周里，我对应用程序架构有很多疑问， 所以我决定写这篇博文。我会描述不同的组件和它们之间的关系，以及我一路做出的一些架构决策。 

我会尽力突出代码库中更有趣的部分，但是我可能在一篇文章中写出很多东西。 如果你觉得这个话题很有趣，我建议你克隆这个项目，自己去探索代码库。 

你可以在这里找到 MovieNight 的源代码：[[PR]](https://github.com/mrsegev/MovieNight)

## 在我们开始＃1之前

这篇博客文章使用了以下主题的知识：Kotlin，RxJava，依赖注入和测试。我尽最大努力在整篇文章中添加链接，以便进一步阅读。如果你遇到了一个你不熟悉的主题，我鼓励你停下来仔细阅读一下。所以我们可以保持在同一进度上。

## 在我们开始＃2之前

从本质上来说，架构是动态和不断演变的。 没有什么"完美的架构"这样的东西，每个问题都有几种解决方案。 请记住，每一个架构决策都是一种权衡。 

## 简洁的方式

MovieNight 架构深受  [Uncle Bob’s clean architecture](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html) 影响。了解简洁的方式背后的原则是我们指南的关键。

虽然初看起来简洁架构可能有点难理解，但一旦你明白了它的全部意义，理解起来就相当简单。

简洁方式的核心原则可概括如下：

-  应用程序代码被分隔为层次。这些层定义了代码库中的[关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns) 
- 图层遵循严格的依赖规则，每一层只能与下面的图层进行交互 
- 当我们进入底层时，代码变得更加通用。底层决定了策略和规则，而上层决定了实现细节，比如数据库、网络管理器和用户界面 

考虑下面的抽象例子：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frool8rsxuj30b406kglo.jpg)

- C 层可以访问 B 层和 A 层。
- B 层可以访问 A 层内的所有内容，但不知道 C 层内的任何内容。实际上，B 层甚至不知道 C 层的存在。
- A 层位于最底层，她不知道她范围之外的任何事情，而且我们很快就会看到，她的无知确实是一种幸福。

就是这样。 其他任何架构细节都只是为了满足这些核心原则。

## 为什么使用简洁的方式？

你是否曾经有过用一个异步、完全成熟的数据库替换内存中的同步数据存储？  😰

需求将会改变。 设计将会改变。 基本上，所有的实现细节都很容易改变。 通过将代码分层，我们可以将这些细节"推"到上层，通过遵循依赖规则，我们能够将它们从应用程序的核心功能中隔离出来。 

这种隔离允许我们编写更加可测试和独立于任何外部因素的代码，因此当一个改变"命中"时，我们可以快速有效地对它做出反应，而不会破坏太多的墙壁。 

既然我们达成了共识，我们可以看看 MovieNight 实际的架构层。 

## MovieNight 体系结构图层

该应用程序由三个层组成： 领域层，数据层和表示层。

查看 MovieNight 的高级项目结构，你会看到每个图层都由项目中的模块表示。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froolv52kwj30aq05sq31.jpg)

像本文中的所有内容一样，这个项目结构只是一种做事方式。我喜欢它，因为它可以帮助我避免层之间的意外泄露，因为我必须在每个模块 build.gradle 文件中指定它依赖的其他模块（即图层）。

以下是这些图层的高级图表以及它们的含义：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froon2i8lnj30ci08mdg7.jpg)

如果这个图表现在看起来有些困难 - 不要冒汗！ 😄 我们将研究每个层的实现细节，并且很快，所有内容（希望）都会有意义。

让我们深入每一层，从最底层开始 - 领域层。

## 领域层

域层是应用程序的基线。 它的目的是描述应用程序是什么以及它可以做什么(很快我们就会看到这个陈述的具体例子)。 

 还记得我们讨论过的第三个简洁方法原则吗？ 当我们向底层移动时  - 代码变得通用。

这里的每一段代码都像代码一样普通。 具体的实现和细节属于上层。 处于底层时，域层不知道应用程序中的其他任何东西，以至于这里的代码与 Android 框架没有任何关系，它只是"纯粹的" Kotlin。 为什么？ 因为它与域层的目的无关。 

该层主要包含域实体，接口和称为用例的特殊类。

## 域实体

这些是我们应用程序的基本组成部分，也是当外部变化时最不可能改变的。 对于 MovieNight 来说，你会在这里找到一些类似 MovieEntity 和 VideoEntity 的类，它们描述了我们所使用的基本数据结构。 下面是实体包: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froonfgzj0j309g06e3ys.jpg)

这是我写的一部分，当我写到域层描述了什么是应用程序和它能做什么时，我的意思的一部分: 看看实体包，很清楚 MovieNight 在处理什么样的数据ーー电影、电影评论等等。 

```kotlin
data class MovieEntity(

        var id: Int = 0,
        var title: String,
        var overview: String? = null,
        var voteCount: Int = 0
        // ...
)
```

在我们的应用程序例子中，这里没有什么花哨的东西，这些类只是作为数据容器， 所以我选择将它们创建为 [Kotlin 数据类](https://kotlinlang.org/docs/reference/data-classes.html)。

## 用例

也被称为交互者，一个用例封装了一个可以执行的特定的任务。这些用例稍后将被上层使用。看看用例包：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1froonomkk0j30d208edgi.jpg)

我们再次描述应用程序。仅仅通过查看文件名，应用程序可以做的就变得很明显了: 我们可以浏览流行电影，我们可以管理喜欢的电影列表，我们可以搜索电影。 

所有的用例都扩展了抽象的 UseCase 类：

```kotlin
abstract class UseCase<T>(private val transformer: Transformer<T>) {

    abstract fun createObservable(data: Map<String, Any>? = null): Observable<T>

    fun observable(withData: Map<String, Any>? = null): Observable<T> {
        return createObservable(withData).compose(transformer)
    }
}
```

这是一个更具体的用例，它提取了流行的电影：

```kotlin
class GetPopularMovies(transformer: Transformer<List<MovieEntity>>,
                       private val moviesRepository: MoviesRepository) : UseCase<List<MovieEntity>>(transformer) {
  
    override fun createObservable(data: Map<String, Any>?): Observable<List<MovieEntity>> {
        return moviesRepository.getMovies()
    }
}
```

有几件事情值得一提：

- 所有用例的输出都是 [Observable](https://github.com/ReactiveX/RxJava/wiki/Observable)。 这里没多少东西要说。可观察者允许我们以一种更加功能性和反应性的方式编写代码，这是我喜欢的 
- 所有用例必须在其构造函数中接收一个 Transformer 对象。Transformer 类只是一个简单的 [ObservableTransformer](http://reactivex.io/RxJava/javadoc/io/reactivex/ObservableTransformer.html)。使用 Transformer 允许我们动态地控制用例 “运行” 的线程。这使得它在编写测试时特别有用，例如：当我们获取流行电影的列表时，我们希望它运行一个工作线程，而不是阻塞主线程，但是在测试时我们希望代码同步运行。值得注意的是，除了测试之外，我想不出在主线程上运行用例的好理由。
- 我们可以将可选数据传递给用例。有时，我们需要将一些数据传递给用例，Kotlin 可选值和默认值在这里很有用，因为我们不需要在不需要数据时指定空值。

正如你所看到的，GetPopularMovies 用例在创建时使用收到的 MovieRepository 对象，并将我们带到领域层的一种类型的居民  - 接口。

## 接口

领域层接口规定了上层必须遵循的合同。这些抽象确保应用程序核心功能将保持正确，而不管任何实施细节的改变。让我们再次看看 GetPopularMovies 用例：当被调用时，用例返回一个可以从 MoviesRepository 获取数据的可观察数据。 

这里是 MovieRepository 实现：

```kotlin
interface MoviesRepository {
    fun getMovies(): Observable<List<MovieEntity>>
    fun search(query: String): Observable<List<MovieEntity>>
    fun getMovie(movieId: Int): Observable<Optional<MovieEntity>>
}
```

这只是一个接口。为什么？因为存储库的实现细节与 GetPopularMovies 用例的功能无关。

看看第5行的 GetPopularMovies 代码：如果 moviesRepository.getMovies ( ) 从远程 API 或本地数据库返回数据是否重要 ？没有，只要实际的实现实现了 MoviesRepoitory 接口 - GetPopularMovies 就会运行良好！

在我们继续我们的指南之前，让我们花点时间谈谈测试。

## 单元测试用例

领域层的 “无知” 使我们能够轻松地测试我们的用例。 这里是 GetPopularMovies 单元测试：

```kotlin
@Test
fun testGetPopularMovies() {
    val movieRepository = Mockito.mock(MoviesRepository::class.java)
    Mockito.`when`(movieRepository.getMovies()).thenReturn(Observable.just(generateMovieEntityList()))
    val getPopularMovies = GetPopularMovies(TestTransformer(), movieRepository)
    getPopularMovies.observable().test()
            .assertValue { results -> results.size == 5 }
            .assertComplete()
}
```

看第4行：因为 MoviesRepository 只是一个接口，所以当调用 getMovies ( ) 时，我可以轻松地使用 [Mockito](http://site.mockito.org/) 来模拟它的行为，并使用存根数据返回 Observable。

请记住，用例在创建时会收到一个 Transformer？在第5行，我使用 TestTransformer 来确保用例的同步执行，以便我可以更轻松地进行测试。

在第7,8行，我通过评估用例发出的数据并确保它与我在第4行上模拟的数据相同来完成测试。

## 总结领域层

领域层位于我们代码库的最底部。在这里，我们定义我们的领域实体，接口和用例以供上层使用。我们尽可能保留一些通用的东西来保护核心功能不受变化的影响，让处理实现细节的麻烦留给上层。 

好。让我们进入到数据层。 🆙

## 数据层

就在领域层之上，我们可以找到数据层。其目的是提供应用程序需要运行的所有数据。

我们不再处于领域层的通用仙境。在这里，我们将找到 MovieNight 使用的不同数据提供者的具体实现。我们还可以在这里看到一种新的特殊类型 - Mappers，但我们会在稍后讨论它们。

请记住，表示层位于数据层之上，所以数据提供者不知道如何以及何时被调用，这是一件好事。 	

## 数据提供者实现细节

数据提供者的实现细节（例如缓存机制，数据库和网络管理器）都在这里，因此你可以找到像 Retrofit 和 Room 等库的引用。当然，这些库是由类包装的，这些类与域层定义的接口相对应，以隐藏它们的存在。 

以下是一个示例：来自域层的 MoviesRepository 接口在数据层中变为 MoviesRepositoryImpl（这是一个可怕的名字！）。

我们来看看 MoviesRepositoryImpl 类：

```kotlin
class MoviesRepositoryImpl(api: Api,
                           private val cache: MoviesCache,
                           movieDataMapper: Mapper<MovieData, MovieEntity>,
                           detailedDataMapper: Mapper<DetailsData, MovieEntity>) : MoviesRepository {

    private val memoryDataStore: MoviesDataStore
    private val remoteDataStore: MoviesDataStore

    init {
        memoryDataStore = CachedMoviesDataStore(cache)
        remoteDataStore = RemoteMoviesDataStore(api, movieDataMapper, detailedDataMapper)
    }

    override fun getMovies(): Observable<List<MovieEntity>> {
        return if (!cache.isEmpty()) {
            return memoryDataStore.getMovies()
        } else {
            remoteDataStore.getMovies().doOnNext { cache.saveAll(it) }
        }
    }

    override fun search(query: String): Observable<List<MovieEntity>> {
        return remoteDataStore.search(query)
    }

    override fun getMovie(movieId: Int): Observable<Optional<MovieEntity>> {
        return remoteDataStore.getMovieById(movieId)
    }
}
```

这里值得一提的是：

- MoviesRepositoryImpl 实现 MoviesRepository 接口。 所以它可以被域层用例使用。
- 该类充当一个工厂，可以在远程数据存储和本地数据存储之间 “玩转”。getMovies ( ) 实现在访问 API 之前检查是否存在缓存数据。其他方法，如 search ( ) 直接调用 API。
- 数据源的实现是抽象的。即使它们属于与 MoviesRepositoryImpl 相同的图层，并且技术上我们不会通过揭示它们的实现来破坏依赖关系规则，有关数据源的任何细节的知识与 MoviesRepositoryImpl 的功能无关，所以我们将它们抽象到 MoviesDataStore 接口后面。 （当然，MoviesDataStore 是在域层中定义的）
- 映射器正在使用。我在本节开头提到了映射器，现在是时候谈论它们了。

## 映射器

正如他们的名字所暗示的，映射器是 “知道” 如何映射类 A 到类 B 的类。所有 MovieNight 中的映射器都扩展了 Mapper 抽象类：

```kotlin
abstract class Mapper<in E, T> {
    
    abstract fun mapFrom(from: E): T

    fun mapOptional(from: Optional<E>): Optional<T> {
        from.value?.let {
            return Optional.of(mapFrom(it))
        } ?: return Optional.empty()
    }

    fun observable(from: E): Observable<T> {
        return Observable.fromCallable { mapFrom(from) }
    }

    fun observable(from: List<E>): Observable<List<T>> {
        return Observable.fromCallable { from.map { mapFrom(it) } }
    }
}
```

这里有一大堆方便的方法，但核心功能非常简单（第3行）：从一边插入 A 类，从另一边获得 B 类。

## 我们为什么需要映射器？

域实体是我们应用程序的基础数据结构，在域层中定义。 他们不应该知道"外面的世界"或他们的"纯洁"会受到损害(我希望你明白为什么现在这是一件坏事)。 

问题在于数据层包含特定的实现，特定的实现往往具有特定的需求。 Retrofit 是一个明显的例子：为了允许 Retrofit 解析网络响应，我们经常使用 GSON 等库。为了解析工作，GSON 有一组注解，我们可以用它来指导解析器。没有注解表示没有解析，但我们不可能使用 GSON 注解注释域实体。域层不知道 GSON 是什么。 	

我们可以做什么？我们可以创建一组新的实体，可以注解的实体并在 Retrofit 使用 。这些实体将位于数据层内部，远离域实体。一个这样的类是 MovieData：

```kotlin
@Entity(tableName = "movies")
data class MovieData(

        @SerializedName("id")
        @PrimaryKey
        var id: Int = -1,

        @SerializedName("vote_count")
        var voteCount: Int = 0,

        @SerializedName("vote_average")
        var voteAverage: Double = 0.0
        
        // ...
}
```

虽然 MovieData 与域实体 MovieEntity 非常相似，但它们之间的唯一区别是 MovieData 包含特定的实现代码（Retrofit 和 Room 库注解）。

因此，如果 Retrofit 响应包含电影列表，则它们现在由 MovieData 对象代替了 MovieEntity 对象组成。问题解决了？ No。 

还有一个问题：用例（域层的居民）不知道 MovieData 是什么，他们只熟悉 MovieEntity！我们可以做什么？每当我们跨越数据层到域层的边界时，我们就可以将 MovieData  映射到 MovieEntity ーー输入映射器。 

## 测试数据层

这里的大部分测试都是单元测试（Room 依赖于 Android 框架，所以我使用仪器进行测试），而且因为我们抽象了大部分的实现，即使是在数据层的居民之间，也很容易测试核心功能。 

让我们来看看 MoviesRepositoryImpl 测试套装的一小部分：

```kotlin
@Before
fun before() {
    api = mock(Api::class.java)
    movieCache = TestMoviesCache()
    movieRepository = MoviesRepositoryImpl(api, movieCache, movieDataMapper, detailsDataMapper)
}

@Test
fun testWhenCacheIsNotEmptyGetMoviesReturnsCachedMovies() {

    movieCache.saveAll(generateMovieEntityList())
    movieRepository.getMovies().test()
            .assertComplete()
            .assertValue { movies -> movies.size == 5 }

    verifyZeroInteractions(api)
}

@Test
fun testWhenCacheIsEmptyGetMoviesReturnsMoviesFromApi() {
   val movieListResult = MovieListResult()
   movieListResult.movies = TestsUtils.generateMovieDataList()
   `when`(api.getPopularMovies()).thenReturn(Observable.just(movieListResult))
    movieRepository.getMovies().test()
            .assertComplete()
            .assertValue { movies -> movies.size == 5 }
}
```

我们可以看到，MoviesRepositoryImpl 使用的所有数据源都被接口抽象出来，因此我们可以使用 Mockito 轻松地对其进行模拟，或者创建一个用于测试目的的简单实现来测试缓存机制，而不会有太多麻烦。

## 总结数据层

数据层位于域层和表示层之间。在这里，我们定义数据提供者的实现细节。我们仍然尝试抽象不同的组件，并将其实现从一个隐藏到另一个，以支持更改并轻松测试核心功能。

接下来是表示层。

## 表示层

我们成功登顶了！表示层将所有不同的片段连接成一个单独的、运行的单元，即应用程序。 在这里，我们将找到活动和片段，UI 和 MVP 体系结构下的演示者，映射器和依赖注入框架，这些框架连接应用程序中的所有内容。

## 表示层架构

注意大标题？是的，那是因为这部分可以很容易地成为自己的博客文章，但我假设（希望？），你已经有了 MovieNight 源代码。我会尽量保持这一部分的简短和重点，让你自己弄清楚所有不同的细节。

表示层由类似于 MVP 体系结构的东西组织。它实际上是我目前正在试验的几种架构理念之间的混合体。

## 视图和 ViewStates

与 MVP 不同的是，视图并未实现由演示者调用的接口。相反，该视图正在观察 ViewState 对象中的更改。这些 ViewState 由 LiveData 对象提供。 正在更新的 LiveData 对象，并且是演示者的一部分。值得一提的是，演示者可以保存多个 LiveData 对象，这个视图将注册到所有这些对象。

ViewState 只是一个数据容器，它包含视图呈现自己所需的所有信息。我们来看一个简单的例子; 这是流行电影屏幕的 ViewState：

```kotlin
data class PopularMoviesViewState(
        var showLoading: Boolean = true,
        var movies: List<Movie>? = null
)
```

很简单不是吗？通过 “阅读” ViewState，视图知道应该显示加载指示符还是要显示哪些 Movie 对象。 （是的，这里有一个新的 Movie 对象）。

以下是 ViewMoviesFragment 视图处理 ViewState 的方式：

```kotlin
private lateinit var progressBar: ProgressBar
private lateinit var popularMoviesAdapter: PopularMoviesAdapter

// ...

private fun handleViewState(state: PopularMoviesViewState) {
        progressBar.visibility = if (state.showLoading) View.VISIBLE else View.GONE
        state.movies?.let { popularMoviesAdapter.addMovies(it) }
}
```

正如我已经提到的，ViewState 对象正在由演示者进行更新。

## 演示者

MovieNight 中的演示者是 Android 的 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 对象。 我知道，这有点令人困惑。如果有人在 “烦人的命名约定” 类别下保持 Google 得分+1。 但是，让我们继续咆哮吧。 值得注意的是，表示模型实际上是域层用例。 

## 为什么使用 ViewModels？

有些人会正确地指出演示者应该远离 Android 框架。通过使用 ViewModels，我正在 “告诉” 演示者关于 Android 生命周期事件的信息，这并不理想。那么，为什么我选择这样做呢？

因为一切都是权衡。我选择牺牲一定程度的抽象并获得无缝的 “经过测试的” 生命周期事件处理。 事情就这么简单。 

让我们来看看演示者的例子。这里是 PopularMoviesViewModel：

```kotlin
class PopularMoviesViewModel(private val getPopularMovies: GetPopularMovies,
                             private val movieEntityMovieMapper: Mapper<MovieEntity, Movie>):
        BaseViewModel() {

    var viewState: MutableLiveData<PopularMoviesViewState> = MutableLiveData()
    var errorState: SingleLiveEvent<Throwable?> = SingleLiveEvent()

    init {
        viewState.value = PopularMoviesViewState()
    }

    fun getPopularMovies() {
        addDisposable(getPopularMovies.observable()
                .flatMap { movieEntityMovieMapper.observable(it) }
                .subscribe({ movies ->
                    viewState.value?.let {
                        val newState = this.viewState.value?.copy(showLoading = false, movies = movies)
                        this.viewState.value = newState
                        this.errorState.value = null
                    }
                }, {
                    viewState.value = viewState.value?.copy(showLoading = false)
                    errorState.value = it
                }))
    }
}
```

这里发生了许多有趣的事情， 让我们按照 “外观顺序” 来解决它们：

第5,6行：这些是视图将观察的 LiveData 对象。其中一个携带 ViewState 对象，另一个携带可选的 Throwable。请注意，errorState 是一种称为 [SingleLiveEvent](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java) 的特殊类型的 LiveData，它的目的是只发送一次更新事件，这在配置更改期间非常有用。

第13行：addDisposable ( ) 是 BaseViewModel 的方法，它将 RxJava 订阅注册到 CompositeSubscription 。当调用 ViewModel onCleared ( ) 时，BaseViewModel 调用 compositeDisposable.clear ( )。

第13-19行：在这里，我们可以看到演示者如何使用用例作为模型：

- 第13行：演示者订阅可观察用例。
- 第14行：使用映射器将域层实体映射到表示层实体。
- 第16-19行：LiveData 值通过一个新的 ViewState 对象进行更新。注意 copy ( ) 方法的用法，该方法将对象参数复制到一个全新的对象中，使我们能够保持 ViewState 不变。我们通过使用 ViewState 的 Kotlin 数据类来 “免费” 获得 copy ( ) 方法。

这包括演示者部分。是时候谈论表示层的另一部分 - 依赖关系注入。

## 依赖注入

正如我前面提到的那样，依赖注入（我从现在开始将它称为 DI）将应用程序中的所有内容连接起来。 DI 负责在抽象规则的代码库中提供具体的实现。

MovieNight DI 基于 Dagger 2 库。 如果你熟悉 Dagger2，值得一提的是我使用具有自定义 Scope 的 SubComponents 来控制不同注入的范围。这允许特定于屏幕的注入对象（如用例和 [ViewModel factories](https://developer.android.com/reference/android/arch/lifecycle/ViewModelProvider.Factory.html)）在不再需要时从内存中释放。

结束。一般而言，DI 和 Dagger2 是非常值得写博客（至少两篇）的巨大主题，所以我将它留在那里。 

是时候再谈谈测试的事了。 

## 测试演示者

测试演示者与调用用例一样简单，并且断言 ViewState 更新符合我们的期望。 

这里是 PopularMoviesViewModel 测试套装的一小部分：

```kotlin
@Before
@UiThreadTest
fun before() {
    moviesRepository = Mockito.mock(MoviesRepository::class.java)
    val getPopularMoviesUseCase = GetPopularMovies(TestTransformer(), moviesRepository)
    popularMoviesViewModel = PopularMoviesViewModel(getPopularMoviesUseCase, movieEntityMovieMapper)
    viewObserver = mock(Observer::class.java) as Observer<PopularMoviesViewState>
    errorObserver = mock(Observer::class.java) as Observer<Throwable?>
    popularMoviesViewModel.viewState.observeForever(viewObserver)
    popularMoviesViewModel.errorState.observeForever(errorObserver)
}

@Test
@UiThreadTest
fun testShowingMoviesAsExpectedAndStopsLoading() {
    val movieEntities = DomainTestUtils.generateMovieEntityList()
    `when`(moviesRepository.getMovies()).thenReturn(Observable.just(movieEntities))
    popularMoviesViewModel.getPopularMovies()
    val movies = movieEntities.map { movieEntityMovieMapper.mapFrom(it) }

    verify(viewObserver).onChanged(PopularMoviesViewState(showLoading = false, movies = movies))
    verify(errorObserver).onChanged(null)
}
```

在上面的例子中，我使用 Mockito 来模拟所有依赖项，调用 getPopularMovies ( ) 方法，最后 - 验证我的模拟观察者被具有特定参数的更新后的 ViewState 调用。

## 结束

这是一个很长的帖子，但我很确定，到目前为止，我已经涵盖了我想写的所有内容。这篇博文旨在强调 MovieNight 代码库的一些有趣部分以及我所做的一些决定。

我希望你觉得这里提到的主题和想法有趣和有意义。

谢谢你的阅读。 😄

