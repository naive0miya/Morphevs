# Android 中处理异常的灵活方式

> 原文 (Medium) ：[Flexible way to handle exceptions in Android](https://android.jlelse.eu/flexible-way-to-handle-exceptions-in-android-9de236f304ad)
>
> 作者 ：[Sergey Opivalov](https://android.jlelse.eu/@6hundreds?source=post_header_lockup)

### 引言

当你的项目变得更大时，你将不可避免地面临应用程序不同部分错误处理的问题。 错误处理可能比较棘手，在某些情况下，你只需向用户显示一个消息，而一些错误则需要执行导航到屏幕等。

大多数开发人员在任何项目的表现层使用某种结构模式ー MVP，MVVM，有些人使用 MVI... ... 在大多数情况下，数据层中发生的异常应该提高到表示层，以便适当处理并向用户显示消息。 但是，这并不是每个案例都是如此。

在我们的项目中，我们对 ErrorHandler 有什么期望？

- 我们需要能够处理各种类型的异常
- 我们希望不仅在演示者中使用它
- 代码应该是可重用的

虽然我将在 MVP 方面进行讨论，但是这种方法对任何形式的演示模式都是有效的。 还有一件事: 即使是在 iOS 系统，它也能工作。

#### 肮脏的实现

所以，你正在使用 MVP。 我很肯定你们大多数人也是如此。

1. 使用某种 MVP 库来减少模板代码(Moxy，Mosby，EasyMvp 等)
2. 有一个像这样的 BasePresenter:

```kotlin
abstract class BasePresenter<View : MvpView> : MvpPresenter<View>() {

    protected var disposables = CompositeDisposable()

    override fun detachView(view: View?) {
        super.detachView(view)
        disposables.clear()
    }
}
```

它还可能包含关于附加和分离视图的代码和一些方向更改时存活的代码。 但是在我的情况下，MvpPresenter 会为我做到这一点。

所以我支持你们，伙计们。 =)

BasePresenter 的具体实现是如何实现的？

```kotlin
class SpecificPresenter @Inject constructor(
              private val useCase : UseCase,
              private val router: Router)
: BasePresenter<SpecificView>() {
  
  fun function() {
    useCase.foo()
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .doOnSubscribe { viewState.showLoading(true) }
      .doFinally { viewState.showLoading(false) }
      .subscribe({ viewState.onSuccess(it)}, { viewState.showError(it) })
  }
}
```

很好。 让我们想象一下，在后端的错误响应中没有得到特定的消息，你必须在客户端处理它。 几乎所有的演示者都应该能够处理错误，所以也许你会有这样的想法：

```kotlin
abstract class BasePresenter<View : MvpView> : MvpPresenter<View>() {
  
protected var disposables = CompositeDisposable()
abstract val resourceManager : ResourceManager
  
override fun detachView(view: View?) {
  super.detachView(view)
  disposables.clear()
}
  
protected fun handleError(error : Throwable) {
  when(error) {
    is HttpException -> when (error.code()) {
      401 -> resourceManager.getString(R.string.error_unauthorized)
      500 -> resourceManager.getString(R.string.error_server)
      else -> resourceManager.getString(R.string.error_unknown)
    }
    is IOException -> resourceManager.getString(R.string.error_network)
    else -> resourceManager.getString(R.string.error_unknown)
  }
}
```

附注: ResourceManager 只是应用程序上下文的一个包装器，因为我们不希望在预览中使用 Android 依赖，以确保它们方便单元测试。

在这一点上，你可以想出很多与实现相关的细节。 例如，你可以将 lambda 传递给 handleError 函数，或者创建一个新的抽象类 ErrorHandlingPresenter，该演示者将扩展 BasePresenter 并将 handleError 函数移动到那里。 实际上还有很大的改进空间。 但这种方法有一些严重的缺点:

- 你的代码不遵循 SOLID。 你的演示者负责错误处理。
- 你的代码是不可重用的。 想象一下，当你试图刷新访问令牌时，需要 Retrofit Interceptor 中处理错误？ 你应该怎么做？ 只是复制和粘贴 handleError 功能？

让我们试着解决它。

#### 正确的方法

让我们假设处理任何错误最常见的方法是向用户显示消息。 注意: 对于任何项目来说，这几乎都是正确的，但是你可以随意调整它来适应你的项目需求。

此外，我将向你展示一个完整的类和层次结构的解释。

首先，我们需要一个类的接口，它能够显示错误。 在大多数情况下，它将通过活动 / 片段执行:

```kotlin
interface CanShowError {
  fun showError(error: String)
}
```

错误处理程序的接口:

```kotlin
interface ErrorHandler {
  fun proceed(error: Throwable)
  fun attachView(view: CanShowError)
  fun detachView()
}
```

由于我们的错误处理程序存在于演示者中，而且演示者在方向改变中存活，所以我们不能将 CanShowError 视图的实例传递给错误处理器构造函数。

下面是 ErrorHandler 最常见的实现:

```kotlin
class DefaultErrorHandler @Inject constructor(private val resourceManager: ResourceManager)
: ErrorHandler {

  private lateinit var view: WeakReference<CanShowError>

  override fun proceed(error: Throwable) {
    Timber.e(error)
    val view = view.get()
               ?: throw IllegalStateException(“View must be attached to ErrorHandler”)
    val message = when (error) {
      is HttpException -> when (error.code()) {
        304 -> resourceManager.getString(R.string.not_modified_error)
        400 -> resourceManager.getString(R.string.bad_request_error)
        401 -> resourceManager.getString(R.string.unauthorized_error)
        403 -> resourceManager.getString(R.string.forbidden_error)
        404 -> resourceManager.getString(R.string.not_found_error)
        405 -> resourceManager.getString(R.string.method_not_allowed_error)
        409  -> resourceManager.getString(R.string.conflict_error)
        422 -> resourceManager.getString(R.string.unprocessable_error)
        500 -> resourceManager.getString(R.string.server_error_error)
        else -> resourceManager.getString(R.string.unknown_error)
      }
      is IOException -> resourceManager.getString(R.string.network_error)
      else -> resourceManager.getString(R.string.unknown_error)
    }
  
    view.showError(message)

  }
  override fun attachView(view: CanShowError) {
  this.view = view.weak()
  }
  override fun detachView() {
  view.clear()
  }
}
```

该实现有一个对视图的弱引用，即能够向用户显示错误。 它有一个资源管理器来获取字符串，它有最简单和通用的错误处理逻辑。 DefaultErrorHandler 将是一个全局单例，并且这些视图将根据屏幕呈现给用户的不同而从 ErrorHandler 中附加和分离。

好。 现在我们需要为每个能够处理错误的演示者注入 ErrorHandler 实现，并且不要忘记附加和分离视图。 为此我更喜欢使用继承，所以我有两个基本演示者：最常见的有 CompositeSubscription 和 ErrorHandlingPresenter，它自动附加和分离视图到错误处理程序。

请注意视图类型的泛型约束。

```kotlin
abstract class BasePresenter<View : MvpView> : MvpPresenter<View>() {

    protected var disposables = CompositeDisposable()

    override fun detachView(view: View?) {
        super.detachView(view)
        disposables.clear()
    }
}
```

```kotlin
abstract class ErrorHandlerPresenter<View> : BasePresenter<View>() where View : MvpView,
                                                                         View : CanShowError {
    abstract val errorHandler: ErrorHandler

    override fun detachView(view: View?) {
        super.detachView(view)
        errorHandler.detachView()
    }

    override fun attachView(view: View) {
        super.attachView(view)
        errorHandler.attachView(view)
    }
}
```

现在，我们能够处理错误的具体演示者将如下所示：

```kotlin
class SpecificPresenter @Inject constructor(
          private val useCase1 : UseCase,
          override val errorHandler : ErrorHandler,
          private val router: Router)
  : ErrorHandlingPresenter<SpecificView>() {
    
  fun function() {
    useCase.foo()
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .doOnSubscribe { viewState.showLoading(true) }
      .doFinally { viewState.showLoading(false) }
      .subscribe({ viewState.onSuccess(it)}, { errorHandler.procced(it) })
  }
}
```

在 Dagger2 的帮助下，我们可以将 DefaultErrorHandler 作为 ErrorHandler 接口的实现传递给构造函数。 但是如果你还记得，DefaultErrorHandler 只能显示错误，所以在我的情况下，业务规则是：如果你从后端得到一个404 Not Found 的错误，那么执行导航到另一个屏幕，如果有其他的话 - 向用户显示错误。

因此，我创建了一个 ErrorHandler 的新实现:

```kotlin
class SpecificErrorHandler @Inject constructor(
        private val defaultErrorHandler: DefaultErrorHandler,
        private val router: Router)
    : ErrorHandler {

    override fun proceed(error: Throwable) {
        when (error) {
            is HttpException -> when (error.code()) {
                404 -> router.replaceScreen(AppScreens.USER_UNREGISTERED_SCREEN)
            }
            else -> defaultErrorHandler.proceed(error)
        }
    }

    override fun attachView(view: CanShowError) {
        defaultErrorHandler.attachView(view)
    }

    override fun detachView() {
        defaultErrorHandler.detachView()
    }
}
```

这里我使用了 Composition 并将 DefaultErrorHandler 注入到 SpecificErrorHandler 中。 此外，一个用于执行屏幕导航的路由器。 在继续函数中，我们尝试捕获404错误并导航到另一个屏幕，如果我们不能 - 我们将控件委托给 defaultErrorHandler。 'attach' 和 'detach' 方法一样 - 控制权被委托给 defaultErrorHandler。

由于 DefaultErrorHandler 是一个全局单例，我们应该使用 Qualifier 来注入 SpecificErrorHandler 实现。

```kotlin
class SpecificPresenter @Inject constructor(
              private val useCase : UseCase,
              @LocalErrorHandler override val errorHandler : ErrorHandler,
              private val router: Router)
: ErrorHandlingPresenter<SpecificView>() {
  
  fun function() {
    useCase.foo()
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .doOnSubscribe { viewState.showLoading(true) }
      .doFinally { viewState.showLoading(false) }
      .subscribe({ viewState.onSuccess(it)}, { errorHandler.proceed(it) })
  }
}
```

就是这样。 现在它应该像预期的那样工作。

正如你所看到的，它不依赖于 MVP 的细节，它是灵活的，遵循 SOLID，我们可以在任何可能产生错误的类中使用它。













