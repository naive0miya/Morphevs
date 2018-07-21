# 现代 Android Kotlin 开发 - 第3部分

> 原文 (Medium )：[Modern Android development with Kotlin Part 3](https://proandroiddev.com/modern-android-development-with-kotlin-part-3-8721fb843d1b)
>
> 作者：[Mladen Rakonjac](https://proandroiddev.com/@mladenrakonjac?source=post_header_lockup)

[TOC]

首先，我想道歉等待下一篇博客文章。我在过去三个月里有很多义务。最后两部分有近 60k 的观看！哇！谢谢大家！

> 我们很难找到一个覆盖 Android 开发中所有新功能的项目，所以我决定写一个。

在这篇文章中，我们将使用以下内容：

0. Android Studio 3 [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
1. Kotlin language [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
2. Build Variants [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
3. ConstraintLayout [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
4. Data binding library [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
5. MVVM architecture + repository pattern + Android Manager Wrappers [Part 2](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-2-17444fcdbe86)
6. RxJava2 and how it helps us in architecture 
7. Dagger 2.11, what is Dependency Injection, why you should use it. 
8. Retrofit (with Rx Java2)
9. Room (with Rx Java2)

现在我们有了新的依赖版本：

- Gradle wrapper 4.1。现在是稳定的
- 最后的 Kotlin 版本是：1.2.20
- Android 体系结构组件现在是稳定的1.0.0，现在已弃用 LifecycleActivity。我们现在应该使用 AppCompatActivity。
- 现在我们使用 targetSdkVersion 27，buildToolsVersion 27.0.2

所有这些变化你可以在[这个提交](https://github.com/mladenrakonjac/ModernAndroidApp/commit/9fed304ac022370e69965b07483fabc1494f6f0e)中找到。

在写关于 RxJava 之前，我对你有好消息：官方 Android 文档现在在 Kotlin 和 Java 上展示了样本！ :)

## 什么是 RxJava？

这不是关于 RxJava，这个概念很广泛。RxJava是一个用于与可观测流、 activex 进行异步编程的 API 的 Java 实现。 实际上，它混合了三个概念: 观察者模式、迭代器模式和函数式编程。 还有其他编程语言的库: RxSwift，RxNet，RxJs。 

我觉得这很难开始。 有时候它真的很令人困惑，如果它没有得到很好的实现，它可能会给你带来一些问题。 尽管如此，花时间去学习是值得的。 我会尝试通过简单的步骤来解释 RxJava。 

首先，让我们回答一些简单的问题，当你开始阅读 RxJava 时，你可能会问自己: 

真的需要吗？

没有。 只是你可以在 Android 开发中使用的另一个库。 但是也不需要 Kotlin，Databinding 库也不需要。 我希望你明白我想说什么。 它只是像其他所有的库一样帮助你。 

我应该在 RxJava2 之前学习 RxJava1 吗？

你可以从 RxJava 2开始。 尽管如此，作为一个 Android 开发者，知道这两个原因对你是有好处的，因为你可以参与维护别人的 RxJava 1代码。 

我看到有 RxAndroid，我应该使用 RxAndroid 还是 RxJava？

可以在任何 Java 开发中使用 RxJava，而不仅仅是 Android。 例如，RxJava 可以与 backend 开发的 Spring 框架一起使用。 RxAndroid 是一个包含了在 Android 中使用 RxJava 所需的内容的库。 所以，如果你想在 Android 开发中使用 RxJava，你必须再添加一个库，RxAndroid。 稍后，我将解释 RxAndroid 在 RxJava 顶部添加的内容。 

我们在做 Kotlin，为什么我们不使用 RxKotlin？

无需再编写一个Rx 库，因此所有 Java 代码在 Kotlin 世界都得到支持。有 [RxKotlin](https://github.com/ReactiveX/RxKotlin) 库，但是这个库被写在RxJava之上。它只是将 Kotlin 功能添加到 RxJava。你可以使用 RxJava 和 Kotlin，而不使用 RxKotlin 库。为了简单起见，我不会在这部分使用 RxKotlin。

## 如何将 RxJava2 添加到项目中？

要添加它，你只需要在 build.gradle 文件中插入这两行：

```groovy
dependencies {
    ... 
    implementation "io.reactivex.rxjava2:rxjava:2.1.8"
    implementation "io.reactivex.rxjava2:rxandroid:2.0.1"
    ...
}
```

并使用 gradle 文件同步项目。

## RxJava 包含什么？

我想将 RxJava 分成三部分：

1. 观察者模式和数据流的类：可观察者和观察者
2. 调度程序（并发）
3. 数据流的操作符

## 可观察者和观察者

我们已经解释了这个模式。你可以将 Observable 视为数据源，Observer 是获取数据的那个。

有不同的方法来创造可观测的事物。 最简单的方法是使用 Observable.just ( ) ，它将一个项目创建并创建可观察的项目，发出该项目。 

让我们去我们的 GitRepoRemoteDataSource，并改变我们的 getRepositories 方法返回 Observable：

```kotlin
class GitRepoRemoteDataSource {

    fun getRepositories() : Observable<ArrayList<Repository>> {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First from remote", "Owner 1", 100, false))
        arrayList.add(Repository("Second from remote", "Owner 2", 30, true))
        arrayList.add(Repository("Third from remote", "Owner 3", 430, false))

        return Observable.just(arrayList).delay(2,TimeUnit.SECONDS)
    }
}
```

 Observable \<ArrayList \<Repository >> 表示 Observable 发出 Repository 对象的数组列表。如果你想创建发布 Repository 对象的 Observable \<Repository> ，你应该使用 Observable.from（arrayList）。

代码中的 .delay（2，TimeUnit.SECONDS） 使我们的 Observables 在开始发送数据之前等待2秒钟。

但是等等，我们没有告诉 Observable 什么时候开始发送数据。观察者通常在观察者订阅时开始发送数据。

请注意，我们不再需要以下接口：

```kotlin
interface OnRepoRemoteReadyCallback {
    fun onRemoteDataReady(data: ArrayList<Repository>)
}
```

让我们为 GitRepoLocalDataSource 做同样的事情：

```kotlin
class GitRepoLocalDataSource {

    fun getRepositories() : Observable<ArrayList<Repository>> {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First From Local", "Owner 1", 100, false))
        arrayList.add(Repository("Second From Local", "Owner 2", 30, true))
        arrayList.add(Repository("Third From Local", "Owner 3", 430, false))

        return Observable.just(arrayList).delay(2, TimeUnit.SECONDS)
    }

    fun saveRepositories(arrayList: ArrayList<Repository>) {
        //todo save repositories in DB
    }
}
```

请注意，我们也不需要这个接口：

```kotlin
interface OnRepoLocalReadyCallback {
    fun onLocalDataReady(data: ArrayList<Repository>)
}
```

现在我们需要改变我们的存储库来返回 Observable：

```kotlin
class GitRepoRepository(private val netManager: NetManager) {

    private val localDataSource = GitRepoLocalDataSource()
    private val remoteDataSource = GitRepoRemoteDataSource()

    fun getRepositories(): Observable<ArrayList<Repository>> {

        netManager.isConnectedToInternet?.let {
            if (it) {
                //todo save those data to local data store
                return remoteDataSource.getRepositories()
            }
        }

        return localDataSource.getRepositories()
    }
}
```

如果有互联网连接，我们从远程数据源返回 Observable，否则我们从本地数据源返回它。而且，当然，也不需要 OnRepositoryReadyCallback 接口。

正如你所猜测的，我们需要改变在 MainViewModel 中获取数据的方式。现在我们应该从 gitRepoRepository 获取 Observable 并订阅它。一旦我们订阅 ，Observable 将开始发射数据：

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
    ...

    fun loadRepositories() {
        isLoading.set(true)
        gitRepoRepository.getRepositories().subscribe(object: Observer<ArrayList<Repository>>{
            override fun onSubscribe(d: Disposable) {
                //todo
            }

            override fun onError(e: Throwable) {
               //todo
            }

            override fun onNext(data: ArrayList<Repository>) {
                repositories.value = data
            }

            override fun onComplete() {
                isLoading.set(false)
            }
        })
    }
}
```

一旦一个观察者订阅了 Observable， onSubscribe 方法就会被调用。请注意，我们有一个 Disposable 作为 onSubscribe 方法的参数。我一会儿再写。 

每当 Observable 发出 onNext ( ) 方法的数据都会被调用。一旦 Observable 完成发射 onComplete ( ) 将被调用。之后，Observable 终止。

如果发生错误，则会调用 Error ( )， 并在此之后 Observable 终止。这意味着 Observable 不会再发射数据， onNext ( ) 将不会被调用，onComplete  ( ) 也不会被调用。

另外，在这里注意。如果你尝试订阅已经终止的 Observable，你会得到 IllegalStateException。

## 那么，RxJava 如何帮助我们？

- 首先，我们摆脱所有这些接口。为每个存储库或数据源创建接口是一种样板
- 如果我们使用接口，并且如果我们的数据层发生错误，我们的应用程序可能会崩溃。使用 RxJava 错误将在onError ( ) 方法中返回，以便我们可以向用户显示相应的错误消息。
- 它是非常清洁的，因为我们总是使用 RxJava 来处理数据层 
- 我以前没有告诉过你：以前的做法可能导致内存泄漏。

## 如何避免使用 RxJava2 和 ViewModel 的内存泄漏？

让我们再次看到 ViewModel 的生命周期：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovab7q3wj30ei0f3wev.jpg)

一旦 Activity 被销毁，ViewModel 的 onCleared ( ) 方法将被调用。在 onCleared ( ) 方法中，我们应该处理所有的 Disposables。那么让我们来做：

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
    ...
    
    lateinit var disposable: Disposable

    fun loadRepositories() {
        isLoading.set(true)
        gitRepoRepository.getRepositories().subscribe(object: Observer<ArrayList<Repository>>{
            override fun onSubscribe(d: Disposable) {
                disposable = d
            }

            override fun onError(e: Throwable) {
                //if some error happens in our data layer our app will not crash, we will
                // get error here
            }

            override fun onNext(data: ArrayList<Repository>) {
                repositories.value = data
            }

            override fun onComplete() {
                isLoading.set(false)
            }
        })
    }

    override fun onCleared() {
        super.onCleared()
        if(!disposable.isDisposed){
            disposable.dispose()
        }
    }
}
```

我们可以进一步改进这个代码: 

首先，我们可以使用实现 Disposable 的 DisposableObserver 而不是 Observer，并使用 dispose ( ) 方法。所以我们不需要 onSubscribe ( ) 方法，因为我们可以直接在 DisposableObserver 实例上处理。

其次，不是返回 void 的 .subscribe ( ) 方法，我们可以使用返回给定观察者的 .subscribeWith ( ) 方法。

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
    ...

    lateinit var disposable: Disposable

    fun loadRepositories() {
        isLoading.set(true)
        disposable = gitRepoRepository.getRepositories().subscribeWith(object: DisposableObserver<ArrayList<Repository>>() {

            override fun onError(e: Throwable) {
               // todo
            }

            override fun onNext(data: ArrayList<Repository>) {
                repositories.value = data
            }

            override fun onComplete() {
                isLoading.set(false)
            }
        })
    }

    override fun onCleared() {
        super.onCleared()
        if(!disposable.isDisposed){
            disposable.dispose()
        }
    }
}
```

我们仍然可以改进这个代码: 

我们保存了 Disposable 实例，所以当 onCleared ( ) 方法被调用时，处理它。但是等等，我们是不是应该对每次调用数据层都这样做呢？ 如果我们有10个不同的调用数据层怎么办？ 对我们来说， 重要的是要区分这些 disposables 吗？

没有。有一个更聪明的方法来做到这一点。当 onCleared ( ) 方法被调用的时候，我们应该把它们全部放在一个桶中，并把它们全部丢弃。我们可以使用 CompositeDisposable。

> CompositeDisposable，可容纳多个其他一次性使用的一次性容器

所以，每当我们创建 Disposable  的时候，我们应该把它保存到 CompositeDisposable 中：

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
    ...
  
    private val compositeDisposable = CompositeDisposable()

    fun loadRepositories() {
        isLoading.set(true)
        compositeDisposable.add(gitRepoRepository.getRepositories().subscribeWith(object: DisposableObserver<ArrayList<Repository>>() {

            override fun onError(e: Throwable) {
                //if some error happens in our data layer our app will not crash, we will
                // get error here
            }

            override fun onNext(data: ArrayList<Repository>) {
                repositories.value = data
            }

            override fun onComplete() {
                isLoading.set(false)
            }
        }))
    }

    override fun onCleared() {
        super.onCleared()
        if(!compositeDisposable.isDisposed){
            compositeDisposable.dispose()
        }
    }
}
```

感谢 Kotlin 扩展功能，我们可以进一步改进: 

> Kotlin 类似于 C＃ 和 Gosu，Kotlin 提供了在不需要从类继承的情况下使用新功能扩展类的能力 

我们创建一个名为 extensions 的新包，并添加新的 RxExtensions.kt 文件：

```kotlin
import io.reactivex.disposables.CompositeDisposable
import io.reactivex.disposables.Disposable

operator fun CompositeDisposable.plusAssign(disposable: Disposable) {
    add(disposable)
}
```

现在我们可以使用 + = sign 将 Disposable 对象添加到 CompositeDisposable 实例中：

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
    ...

    private val compositeDisposable = CompositeDisposable()

    fun loadRepositories() {
        isLoading.set(true)
        compositeDisposable += gitRepoRepository.getRepositories().subscribeWith(object : DisposableObserver<ArrayList<Repository>>() {

            override fun onError(e: Throwable) {
                //if some error happens in our data layer our app will not crash, we will
                // get error here
            }

            override fun onNext(data: ArrayList<Repository>) {
                repositories.value = data
            }

            override fun onComplete() {
                isLoading.set(false)
            }
        })
    }

    override fun onCleared() {
        super.onCleared()
        if (!compositeDisposable.isDisposed) {
            compositeDisposable.dispose()
        }
    }
}
```

现在我们来尝试运行该应用程序。一旦你点击加载数据按钮，应用程序将在两秒钟内崩溃。然后，如果你去日志，你会看到，这个错误发生在 onNext 方法和异常的原因是：

> java.lang.IllegalStateException：不能在后台线程上调用 setValue

为什么会发生？

## 调度器（并发）

RxJava 带有调度器，可以让我们选择执行哪个线程。更准确地说，我们可以使用 subscribeOn ( ) 方法选择哪个线程将观察操作，哪个观察者线程将使用 observeOn ( ) 方法。 通常，所有来自数据层的代码都应该在后台线程上运行。 例如，如果我们使用 Schedulers.newThread ( )，调度程序将在每次调用它时提供新的线程。 为了简单起见，还有其他来自调度程序的方法，我不会在这个博客文章中介绍。

可能你已经知道所有的 UI 代码都是在 Android 主线程上完成的。 RxJava 是 Java 库，它不知道 Android 主线程。 这就是我们使用 RxAndroid 的原因。 RxAndroid 给了我们选择 Android 主线程作为我们的代码将被执行的线程的可能性。 显然，我们的 Observer 应该在 Android 主线程上运行。

我们开始做吧：

```kotlin
...
fun loadRepositories() {
        isLoading.set(true)
        compositeDisposable += gitRepoRepository
                .getRepositories()
                .subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribeWith(object : DisposableObserver<ArrayList<Repository>>() {
              ...
        })
    }
...
```

让我们再次运行应用程序。现在一切正常:)良好的工作！

## 更多的可观察类型

我们有其他可观察的类型：

Single\<T> ,可观察的，只发出一个项目或错误

Maybe\<T> , 可观察的，不发射物体，一个物体或错误

Completable , 发出 onSuccess 事件或错误。

Flowable\<T>, 像 Observable\<T> 发出 n 个项目，没有项目或错误。虽然 Observable 不支持背压，但 Flowable 支持它。

## 什么是背压？

为了记住一些概念，我喜欢将现实生活与之并行: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovahkvf5j30iq0tcdkd.jpg)

我认为这是一个漏斗。 如果你填充的更多，那么你的瓶颈可以接受它，你会填满它，你会弄得一团糟。 同样的事情在这里。 有时你的观察者不能处理它正在接收的事件的数量，所以需要放慢速度。

你可以在 RxJava 的[文档](https://github.com/ReactiveX/RxJava/wiki/Backpressure-%282.0%29)中找到更多关于它的信息。

## 操作符

关于 RxJava 的很酷的事情是它的操作符。在 RxJava 中只需要一行代码就可以解决一些需要10行以上的常见问题。有操作符可以帮助我们：

- 结合可观察者
- 过滤
- 条件
- 将可观察者转换为其他类型

我将举一个例子，让我们完成在我们的 GitRepoLocalDataSource 中保存数据。因为我们正在保存数据，所以我们需要 Completable 来模拟它。假设我们也想模拟1秒的延迟。真正天真的做法是：

```kotlin
fun saveRepositories(arrayList: ArrayList<Repository>): Completable {
    return Completable.complete().delay(1,TimeUnit.SECONDS)
}
```

> Completable.complete ( ) 返回一个在订阅时立即完成的 Completable 实例。

一旦你的 Completable 完成，它将被终止。所以，任何操作符（延迟是其中的一个操作符）将不会被执行。在这种情况下，我们的 Completable 将不会有任何延迟。让我们找到办法：

```kotlin
fun saveRepositories(arrayList: ArrayList<Repository>): Completable {
    return Single.just(1).delay(1,TimeUnit.SECONDS).toCompletable()
}
```

为什么这样？

Single.just（1）将创建单个延迟发射1秒。

> toCompletable ( ) 返回一个 Completable，放弃 Single 的结果，并在此源 Single 调用 onSuccess 时调用 onComplete

所以，这段代码将返回 Completable，它会在1秒后调用 onComplete。

现在，我们应该改变我们的 GitRepoRepository。 让我们回顾一下逻辑。 我们检查互联网连接。 如果有互联网连接，我们必须从远程数据源获取数据，将其保存在本地数据源中并返回数据。 否则，我们只从本地数据源获取数据。 看看：

```kotlin
fun getRepositories(): Observable<ArrayList<Repository>> {

    netManager.isConnectedToInternet?.let {
        if (it) {
            return remoteDataSource.getRepositories().flatMap {
                return@flatMap localDataSource.saveRepositories(it)
                        .toSingleDefault(it)
                        .toObservable()
            }
        }
    }

    return localDataSource.getRepositories()
}
```

使用 .flatMap，一旦 remoteDataSource.getRepositories ( ) 发出物品，该物品将被映射到发出相同物品的新 Observable。 我们从 Completable 创建的新的 Observable 将本地数据存储中保存的相同发射项目转换为 Single，发射相同的发射项目。 因为我们需要返回 Observable，所以我们必须将 Single 转换为 Observable。

很疯狂吧？ 想象一下我们还能用 RxJava 做些什么。 RxJava 是一个强大的工具！探索它。 我相信你会喜欢的。 

