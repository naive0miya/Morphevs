# 使用 Retrofit、 Rx 和架构组件在 Android 上构建高效的网络

> 原文 (Medium)：[Effective Networking On Android using Retrofit, Rx and Architecture Components](https://medium.com/mindorks/effective-networking-on-android-using-retrofit-rx-and-architecture-components-4554ca5b167d)
>
> 作者：[karntrehan](https://medium.com/@karntrehan?source=post_header_lockup)

[TOC]

对于很多人来说，像我一样，[Retrofit](http://square.github.io/retrofit/) 是 Android 上的库，可以与 REST API 对话。 Retrofit 还支持[适配器](https://github.com/JakeWharton/retrofit2-rxjava2-adapter)将你的 API 响应转换为[反应流](https://mindorks.com/course/learn-rxjava)。除了别的以外，反应流还可帮助你有效地处理[线程](https://blog.gojekengineering.com/multi-threading-like-a-boss-in-android-with-rxjava-2-b8b7cf6eb5e2)。这使得你的 Android 应用程序更健壮，更稳定，更快速。

对于更简单的用例，可以在[这里](https://medium.com/3xplore/handling-api-calls-using-retrofit-2-and-rxjava-2-1871c891b6ae)和[这里](https://android.jlelse.eu/a-simple-android-apps-with-mvp-dagger-rxjava-and-retrofit-4edb214a66d7)阅读 Retrofit + Rx 的集成。这篇高级文章试图进入下一个层次，让你做到以下几点: 

- 集成 Rx + Retrofit + [Android 体系结构组件](https://developer.android.com/topic/libraries/architecture/index.html)（AAC）。
- 使用 MVVM 体系结构的多模块项目。
- 使用 Kotlin 及其密封类来保持状态。
- 有效处理错误。
- 遵循 Android 组件的生命周期。

你可以在这里找到支持此架构的示例应用程序：[[PR]](https://github.com/karntrehan/Posts/)

## 实现

### 架构

我们遵循 MVVM 架构，使用 AAC 的 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 帮助我们的视图与存储库进行通信。所有这些都使用 [Dagger 2](https://google.github.io/dagger/) 粘在一起。我们单一的真相来源是我们使用 AAC 的 [Room](https://developer.android.com/topic/libraries/architecture/room.html) ORM 的本地数据库。

该架构的核心是 Outcome.kt 密封类

```kotlin
sealed class Outcome<T> {
    data class Progress<T>(var loading: Boolean) : Outcome<T>()
    data class Success<T>(var data: T) : Outcome<T>()
    data class Failure<T>(val e: Throwable) : Outcome<T>()

    companion object {
        fun <T> loading(isLoading: Boolean): Outcome<T> = Progress(isLoading)

        fun <T> success(data: T): Outcome<T> = Success(data)

        fun <T> failure(e: Throwable): Outcome<T> = Failure(e)
    }
}
```

outcome 保持了屏幕的状态。它可以

- loading (isLoading) - 其中 isLoading 表示数据查找和可用于显示或隐藏进度条。
- success (data) —  data 是你希望在 UI 上显示的数据。
- failure (exception) —exception 表示可能导致数据查找失败的任何错误 。

### 流程

outcome 对象在不同层次上被观察：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frostwujifj30em0gz751.jpg)

1. 来自你的活动的事件传递到视图模型。

```kotlin
viewModel.getPosts()
```

2. 视图模型将事件传递给存储库。

```kotlin
fun getPosts() {
    if (postsOutcome.value == null)
        repo.fetchPosts(compositeDisposable)
}
```

3. 存储库为 Outcome 类创建 [PublishSubject](https://blog.mindorks.com/understanding-rxjava-subject-publish-replay-behavior-and-async-subject-224d663d452f)。

```kotlin
val outcome: PublishSubject.create<Outcome<List<PostWithUser>>>()
```

4. 存储库开始观察本地数据库的内容，将其推送到 publishSubject.success (data) 并在第一次 ping 服务器以获取远程更新。

```kotlin
fun fetchPosts(compositeDisposable: CompositeDisposable) {
    postFetchOutcome.loading(true)
    //Observe changes to the db
    compositeDisposable.add(postDb.postDao().getAll()
            .performOnBackOutOnMain()
            .subscribe({ retailers ->
                postFetchOutcome.success(retailers)
                if (remoteFetch)
                    refreshPosts(compositeDisposable)
                remoteFetch = false
            }, { error -> handleError(error) })
    )
}
```

5. 远程同步成功，保存到内部数据库。这会触发 publishSubject.success (data) 。出错时，来自服务器的错误将转换为异常并推送到 publishSubject.Error (exception)。

```kotlin
fun refreshPosts(compositeDisposable: CompositeDisposable) {
    postFetchOutcome.loading(true)
    compositeDisposable.add(
            Flowable.zip(
                    postService.getUsers(),
                    postService.getPosts(),
                    BiFunction<List<User>, List<Post>, Unit> { t1, t2 -> saveUsersAndPosts(t1, t2) }
            )
                    .performOnBackOutOnMain()
                    .subscribe({}, { error -> handleError(error) }))
}
```

6. 视图模型有一个 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 对象，用于观察存储库的 publishSubject。

```kotlin
val postsOutcome: LiveData<Outcome<List<Post>>> by lazy {
    //Convert publishSubject to livedata
    repo.postFetchOutcome.toLiveData(compositeDisposable)
}
```

7. 该活动正在观察视图模型的 LiveData 对象并相应地更新屏幕内容。

```kotlin
viewModel.postsOutcome.observe(this, Observer<Outcome<List<Post>>> { outcome ->
        when (outcome) {

            is Outcome.Progress -> srlPosts.isRefreshing = outcome.loading

            is Outcome.Success -> {
                Toast.makeText(context, "Success!", Toast.LENGTH_LONG).show()
                adapter.setData(outcome.data)
            }

            is Outcome.Failure -> {
                if (outcome.e is IOException)
                    Toast.makeText(context, R.string.need_internet_posts, Toast.LENGTH_LONG).show()
                else
                    Toast.makeText(context, R.string.failed_post_try_again, Toast.LENGTH_LONG).show()
            }

        }
    })
```

## 结束

- 加载/数据/错误，所有的状态都会受到活动的影响 。	
- 网络同步的异常情况也可以作为非致命事件推送给 [Crashlytics](http://try.crashlytics.com/) 以便稍后进行调试。
- 由于 ViewModel 和 Dagger 作用域，Android 组件的生命周期得到了遵循，上下文和数据都没有泄漏。
- ViewModel 和 Repository 可以单独进行测试。
- Room 可以用 ObjectBox 替换。

谢谢阅读！







