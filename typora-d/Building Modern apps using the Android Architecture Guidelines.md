# 使用 Android 架构构建现代化应用的指南

> 原文 (Medium)：[Building Modern apps using the Android Architecture Guidelines](https://proandroiddev.com/building-modern-apps-using-the-android-architecture-guidelines-3238fff96f14)
>
> 作者：[Akshay Chordiya](https://proandroiddev.com/@aky?source=post_header_lockup)

[TOC]

## 目录

- [Introduction to Android Architecture Components](https://medium.com/@aky/introduction-to-android-architecture-components-22b8c84f0b9d)
- [Exploring ViewModel Architecture component](https://medium.com/@aky/exploring-viewmodel-architecture-component-5d60828172f9)
- [Exploring LiveData Architecture component](https://medium.com/@aky/exploring-livedata-architecture-component-f9375d3644ee)
- [Exploring Room Architecture component](https://medium.com/@aky/exploring-room-architecture-component-6db807094241)
- [Exploring Room Architecture component — The Extras](https://medium.com/@aky/exploring-room-the-extras-cf3f0259ceed)
- [Building Modern apps using the Android Architecture Guidelines](https://medium.com/@aky/building-modern-apps-using-the-android-architecture-guidelines-3238fff96f14)

## 前言

在本文中，您将了解什么是 [Android 架构指南](https://developer.android.com/topic/libraries/architecture/guide.html)，以及如何利用它们在架构组件的帮助下构建现代的 Android 应用程序。 

如果你更喜欢视频，请前往：[[VIDEO]](https://caster.io/courses/android-architecture-components-deep-dive)

或者

如果您直接需要代码，请前往：[[PR]](https://github.com/AkshayChordiya/android-arch-news-sample)

> 如果您不了解体系结构组件，请查看我以前的一系列文章，从 Android 体系结构组件入门开始介绍。

## 架构指南

安卓架构指南是安卓团队的一套最佳实践和推荐的架构模式; 如何构建可测试和生产质量的应用程序。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frostcs0wsj30k00f0gm7.jpg)

这是安卓团队在架构指南中推荐的架构图。 

- 它建议保持我们的活动和片段精益求精，只需要维护用户界面相关的代码，比如点击监听等等 
- 并且 ViewModel 提供了 UI 控制器需要的数据，如活动和片段，配置变化时有助于生存
- 管 ViewModel 从 Repository 中获取数据，Repository 是数据的唯一真实来源，这意味着每当你的应用需要数据时，它总是来自存储库 
- 现在存储库决定是使用 Room 从本地数据库获取数据还是使用 Retrofit 从 Web 服务获取数据。

简而言之，

| Component                              | Action                            |
| -------------------------------------- | --------------------------------- |
| UI Controllers (Activity and Fragment) | Only UI related logic             |
| ViewModel                              | Container for data required by UI |
| Repository                             | Single source of truth for data   |
| Room                                   | Local Database                    |
| Retrofit                               | Web Service                       |

建议使用依赖注入（[Dagger](https://github.com/google/dagger)）或服务定位器模式来管理组件之间的依赖关系。

## 公开数据和状态

在使用 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 时，默认情况下，您只获得数据，而不是像[指南附录部分](https://developer.android.com/topic/libraries/architecture/guide.html#addendum)所述的加载，错误和成功等状态。

Resource 类保存状态。 即 SUCCESS, ERROR 或 LOADING，数据和错误消息。

Resource.kt 

```kotlin
/**
 * A generic class that holds a value with its loading status.
 * @param <T>
</T> */
data class Resource<ResultType>(var status: Status, var data: ResultType? = null, var message: String? = null) {

    companion object {
        /**
         * Creates [Resource] object with `SUCCESS` status and [data].
         */
        fun <ResultType> success(data: ResultType): Resource<ResultType> = Resource(Status.SUCCESS, data)

        /**
         * Creates [Resource] object with `LOADING` status to notify
         * the UI to showing loading.
         */
        fun <ResultType> loading(): Resource<ResultType> = Resource(Status.LOADING)

        /**
         * Creates [Resource] object with `ERROR` status and [message].
         */
        fun <ResultType> error(message: String?): Resource<ResultType> = Resource(Status.ERROR, message = message)
    }
}
```

Status.kt 

```kotlin
/**
 * Status of a resource that is provided to the UI.
 *
 *
 * These are usually created by the Repository classes where they return
 * `LiveData<Resource<T>>` to pass back the latest data to the UI with its fetch status.
 */
enum class Status {
    SUCCESS,
    ERROR,
    LOADING;

    /**
     * Returns `true` if the [Status] is loading else `false`.
     */
    fun isLoading() = this == LOADING
}
```

有了这个，我们的 LiveData 将能够在下面的例子中给我们的 Activity 或 Fragments 数据和状态指示：

```kotlin
// Activity or Fragment
viewModel.getFeeds().observe(this,Observer<Resource<List<Feed>?>?> {
    it?.let { resource ->
       when (resource.status) {
            Status.SUCCESS -> { 
                // Update the UI
            }
            Status.ERROR -> {
                // Show error toast or error indicator
            }
            Status.LOADING -> {
                // Show progress bar
            }
       }
   }
})
```

我们的 ViewModel 很可能看起来像这样：

```kotlin
class FeedViewModel : ViewModel() {
    /**
     * Complete list of feeds
     */
    private var feeds: LiveData<Resource<List<Feed>?>>
    init {
        feeds = /** Get the feeds using Repository */     
    }
    /**
     * Get list of feeds
     */
    fun getFeeds() = feeds
```

LiveData 的这一小小包装对于根据状态更新用户界面非常有用。

> PS：你很快会看到它在 DB 和 Web 之间的同步工作效果如何 

## 持久数据

大多数现代的 Android 应用程序在本地数据库和网络服务之间保持着同步，这对于那些即使设备离线也能帮助使用这个应用的用户来说是非常有用的。 这是用户开始在应用程序中开始一天一次的期望。 

[“Building for Billion”](https://developer.android.com/distribute/best-practices/develop/build-for-the-next-billion.html) 指南也指出了同样的问题 ，指南在下图中说明了数据应如何在网络和当地数据库之间流动: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frostkbudnj30eu0cx74p.jpg)

它声明首先从本地数据库中获取数据; 如果数据不存在，则根据"应取"条件进行所需的网络调用，然后将取得的数据保存到本地的响应数据库中。 

> 请记住，用户界面应该总是基于数据库中的数据进行反映 

这是一个非常常见的用例，因此 Android 团队已经创建了一个名为 [NetworkBoundResource](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/repository/NetworkBoundResource.java) 的 helper 类。 

## 揭秘 NetworkBoundResource

Networkboundresource 是一个简单的类，它使用 MediatorLiveData 和 Resource 类的能力在本地数据库和网络服务之间进行数据流的作业，它将状态和数据上传到活动或片段的上游。 

这个类非常好。它的工作原理与上面的数据流图完全相同（如果图太多，请参阅下面的流程）

1. 起初它会发送事件进行加载 
2. 从本地数据库中获取数据(注意: 也可以是空的) 
3. 检查 "shouldFetch" 中的条件 
4. 如果为 false，则从本地数据库向上游发送数据 
5. 如果为 true，它执行所需的网络调用并获取数据 
6. 在响应时，将数据保存到本地数据库中，然后将数据和事件发送到上游 

现在，为了简单起见，我减少了几件事情，只关注流程解释中的重要部分。 

```kotlin
/**
 * A generic class that can provide a resource backed by both the sqlite database and the network.
 *
 *
 * You can read more about it in the [Architecture
 * Guide](https://developer.android.com/arch).
 *
 * @param <ResultType>
 * @param <RequestType>
 * </RequestType></ResultType> */
abstract class NetworkBoundResource<ResultType, RequestType> @MainThread constructor(
        private val appExecutors: AppExecutors
) {

    /
      The final result LiveData
     /
    private val result = MediatorLiveData<Resource<ResultType?>>()

    init {
        // Send loading state to UI
        result.value = Resource.loading()
        val dbSource = this.loadFromDb()
        result.addSource(dbSource) { data ->
            result.removeSource(dbSource)
            if (shouldFetch(data)) {
                fetchFromNetwork(dbSource)
            } else {
                result.addSource(dbSource) { newData -> setValue(Resource.success(newData)) }
            }
        }
    }

    /
      Fetch the data from network and persist into DB and then
      send it back to UI.
     /
    private fun fetchFromNetwork(dbSource: LiveData<ResultType>) {
        val apiResponse = createCall()
        // we re-attach dbSource as a new source, it will dispatch its latest value quickly
        result.addSource(dbSource) { result.setValue(Resource.loading()) }
        result.addSource(apiResponse) { response ->
            result.removeSource(apiResponse)
            result.removeSource(dbSource)

            response?.apply {
                if (isSuccessful) {
                    appExecutors.diskIO().execute {
                        processResponse(this)?.let { requestType -> saveCallResult(requestType) }
                        appExecutors.mainThread().execute {
                            // we specially request a new live data,
                            // otherwise we will get immediately last cached value,
                            // which may not be updated with latest results received from network.
                            result.addSource(loadFromDb()) { newData -> setValue(Resource.success(newData)) }
                        }
                    }
                } else {
                    onFetchFailed()
                    result.addSource(dbSource) { result.setValue(Resource.error(errorMessage)) }
                }
            }
        }
    }

    @MainThread
    private fun setValue(newValue: Resource<ResultType?>) {
        if (result.value != newValue) result.value = newValue
    }

    protected fun onFetchFailed() {}

    fun asLiveData(): LiveData<Resource<ResultType?>> {
        return result
    }

    @WorkerThread
    private fun processResponse(response: ApiResponse<RequestType>): RequestType? {
        return response.body
    }

    @WorkerThread
    protected abstract fun saveCallResult(item: RequestType)

    @MainThread
    protected abstract fun shouldFetch(data: ResultType?): Boolean

    @MainThread
    protected abstract fun loadFromDb(): LiveData<ResultType>

    @MainThread
    protected abstract fun createCall(): LiveData<ApiResponse<RequestType>>
}
```

> 注意：我正在使用并显示 NetworkBoundResouce 的 Kotlin 版本，但您可以在此处找到原始类。[[PR](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/repository/NetworkBoundResource.java)]

如果你想知道 AppExecutors 类，请从这里获取。[[PR](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/AppExecutors.java)]

## 使用方法

让我们来看一下这个类的示例用法，以准确理解这个类是如何工作的。

NetworkBoundResource 是一个抽象类，因此当你创建它的对象时，需要重写下面的方法。

- loadFromDb ( ) - 包含从数据库中获取数据的逻辑（比如 Room）。
- shouldFetch ( ) -返回布尔值，指示是否从网络获取数据，true 表示从网络获取数据。
- createCall  ( ) - 包含从网络服务获取数据的逻辑。
- saveCallResult ( ) - 这里我们保存从网络服务获取的数据。

```kotlin
class FeedRepository @Inject constructor(
        private val feedService: FeedService,
        private val feedDao: FeedDao,
        private val appExecutors: AppExecutors = AppExecutors()
) {

    fun getFeeds(): LiveData<Resource<List<Feed>?>> = object : NetworkBoundResource<List<Feed>, List<Feed>>(appExecutors) {
        override fun saveCallResult(item: List<Feed>) {
            feedDao.insertAll(item)
        }

        override fun shouldFetch(data: List<Feed>?): Boolean = data == null || data.isEmpty()

        override fun loadFromDb(): LiveData<List<Feed>> = feedDao.getFeeds()

        override fun createCall(): LiveData<ApiResponse<List<Feed>>> = feedService.getFeeds()
    }.asLiveData()
}    
```

## 测试

体系结构指南有助于保持所有组件的隔离，这是优秀的测试 

| Component  | Test         | Mock               |
| ---------- | ------------ | ------------------ |
| UI         | Espresso     | ViewModel          |
| ViewModel  | JUnit        | Repository         |
| Repository | JUnit        | DAO and WebService |
| DAO        | Instrumented | -                  |
| WebService | Instrumented | MockWebServer      |

提示：我将很快发布一些关于测试架构组件的文章。

好了，理论上说够了。 所以，我已经建立了一个非常简单的新闻应用程序，使用架构指南来展示应用程序。 检查一下，看看一些具体的代码: [[PR]](https://github.com/AkshayChordiya/android-arch-news-sample/)

## 结束

安卓体系结构指南令人惊叹，我个人很高兴安卓团队最终就如何构建现代化和高质量的 Android 应用提出了自己的意见。 

如果您想了解有关此主题的更多信息，请随时 ping 我，并且可以在 Twitter 上查询问题。[[Twitter]](https://twitter.com/Akshay_Chordiya)

