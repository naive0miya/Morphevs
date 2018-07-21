# MVP＆生命周期＆调度器

> 原文 (Medium)： [MVP & Lifecycles & Dispatchers Oh My! ](https://medium.com/s23nyc-tech/mvp-lifecycles-dispatchers-oh-my-19eda37a1a52)
> 作者： [Mike Nakhimovich](https://medium.com/@theMikhail) 

[TOC]

最近 Brian Plummer 和我开始在 Nike 工作。在有机会安顿下来之后，我们的任务就是创建一个可用于各种 Nike Android 应用的库，特别是我最喜欢的 Nike 应用 SNKRS。

在高层次上，任务是创建一个用户可以浏览的流程。我们选择使用由 MVP 支持的单个 Activity 体系结构，并使用生命周期感知 presenters 和状态更改的被动调度程序。我们完全用 Kotlin 写了我们的库。在这篇文章中，我们将解释我们是如何做到的。

像所有的架构一样，这个架构仍然是一个正在进行的工作，但是我们想分享我们的思考过程。这种体系结构使我们能够建立一个扩增实境流，这个流程封装在一个单独的活动中，有6个屏幕和预览。在过去，我们总是以一个紧密耦合的混乱状态结束，这种混乱是通过互相注入或者以不可伸缩的方式嵌套的视图。这一次，我们能够建立一个被动的架构，在这里，每个演示者和视图从一个集中的调度器中对新状态做出反应。 因此，我们通过使我们的视图和演示者脱钩来实现我们的目标。 

我们一直是 MVP 的铁杆粉丝。对于一个新的开发者来说，这个架构似乎总是最容易解释的：

- 当一个视图 inflates 时，它会创建一个 presenter 的实例并附加到该视图上 。
- 当视图需要与除了它自己的布局之外的任何事物进行交互时，它就会在演示者上调用一个函数，然后通过视图接口等待响应 。
- presenter 没有任何 Android 框架代码，可以使用视图接口在 JVM 中轻松进行测试。

以下是非生命周期感知 view 和 presenter 的示例：

view 接口

```java
interface NoNetworkMVPView : MvpView {
    fun show()
    fun hide()
    fun showSnackBar()
}
```
view 实现我们的接口，并附加到我们的 presenter

```java
class NoNetworkView @JvmOverloads constructor(context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = -1) :
        ConstraintLayout(context, attrs, defStyleAttr), NoNetworkMVPView {
    
    @Inject lateinit var noNetworkPresenter: NoNetworkPresenter

    init {
        Injector.obtain(getContext()).inject(this)
    }

    override fun onAttachToWindow() {
        super.onAttachToWindow()
        noNetworkPresenter.attachView(this)
        noNetworkClose.setOnClickListener { noNetworkPresenter.goBack() }
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        noNetworkPresenter.detachView()
    }
    
    //callbacks from presenter
    override fun show() { visibility = View.VISIBLE }

    override fun hide() { visibility = View.INVISIBLE }
}
```

Presenter

```java
@ActivityScoped
class NoNetworkPresenter @Inject constructor() :
        BasePresenter<NoNetworkMVPView>() {
    override fun attachView(mvpView: NoNetworkMVPView) {
        super.attachView(mvpView)
       if(ViewState.hiding)mvpView?.hide() else mvpView?.show()
    }

    override fun detachView() {
       ///
    }

    fun goBack() {
       //go back
    }
}
```
因为 presenters 不和 Android 直接连接，我们需要调用 presenter.attachView 和 presenter.detachView ，于此同时 presenter 不知道 view 何时绑定或解绑 。如果我们不调用 presenter.detachView，那么当  presenter 必须进行长时间的操作时，我们就有内存泄漏的风险。

我们的 mvpView 可以是 activity , fragment 或者 view。其主要目的是实现一个接口，presenter 将使用它来告诉视图应该渲染什么。 如果我[一年前写这篇文章](https://hackernoon.com/presenters-are-not-for-persisting-f537a2cc7962)，我会说：“我们在这里有一个不错的 presenter！”但这是2017年（差不多2018年！），我们现在生活在一个生命周期感知组件的世界里， 最重要的是 SupportActivity 实现 LifeCycleOwner 。由于Activities 是一个 LifecycleOwner，并且视图能够获得关于其包含 activity 的引用，所以我们决定让我们的 presenter.attachView 函数进入生命周期并注册。

```java
override fun attachView(mvpView: NoNetworkMVPView, lifecycle: Lifecycle){
lifecycle.addObserver(this)
...
}
```
这是由于我们的演示者实现了生命周期观察者。 

现在，我们的 Presenter 正在监听 activty 的生命周期，我们可以让我们的 detachView 监听我们 activity 的 ON_PAUSE 生命周期，这意味着我们永远不必明确地调用它。有了自动分离，我们忘记从一个视图调用 detachView 方法也不会有影响了，从这个角度来看是值得高兴的。

```java
@OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
override fun detachView() {
    disposables.clear()
    mvpView = null
}
```
现在，views 不再需要在实现类中调用 onDetachFromWindow，因为我们的 presenter 可以自动从他们自己的 view 分离。我们的用户界面的一般流程将是：
- Activity 创建
- Activity 附加 views
- 每个 view 创建 一个 presenter，同时 presenter 本身绑定 view
- presenter 请求任何必要的数据并将其传递给 view
- 当我们的 Activity 暂停时，presenter  将从本身分离 view，防止内存泄漏。

我们现在有一个 NoNetworkView - 你猜对了！ - 当我们有网络连接的问题时我们想要显示一些信息。当网络失去了连接的时候，我们将显示 NoNetworkView , 不管当前在哪个屏幕上。我们不想做的是注入 NoNetworkPresenter 到任何其他 view/presenter 才能显示 noNetwork , 因为我们认为这将最终导致循环依赖, 例如, 如果 SuccessPresenter 需要能够调用 NoNetworkPresenter , 这反过来需要能够调用 SuccessPresenter 的东西。相反，我们希望利用 RxJava，让每个 presenters 对状态变化做出反应，告诉他们是否渲染或隐藏他们的 UI。 我们所做的就是让我们的 presenters 监听一个被动的 dispatcher 。 以下是我们 presenters 的 API：
```java
override fun attachView(mvpView: NoNetworkMVPView, lifecycle: Lifecycle) {
    super.attachView(mvpView, lifecycle)
    dispatcher.noNetwork()
            .observeOn(scheduler)
            .subscribe { mvpView.show() }
dispatcher
        .showing()
        .filter { it !is Showing.NoNetwork }
        .subscribe { mvpView.hide() }
```

在上面的例子中，我们的演示者正在监听一个名为 noNetwork 的流。当一个事件被发出时，它告诉它的视图呈现自己。

同样，presenter 监听我们的事件的反面，这其实是其他 showing 事件，并隐藏其 view 。

下面我们来看看我们的 dispatcher 。
```java
@Singleton
class Dispatcher @Inject
constructor(val stateSubject: PublishSubject<State>) {

    val showEvents: Stack<State> = Stack()

    fun dispatch(state: State) = stateSubject.onNext(state)

    fun dispatchShow(state: State) {
        showEvents.push(state)
        Timber.d("pushing " + showEvents.peek())
        stateSubject.onNext(state)
    }
//showing events
    fun noNetwork() = stateSubject.ofType(Showing.NoNetwork::class.java)
fun showingFailure() = stateSubject.ofType(Showing.Failure::class.java)

fun showingSuccess() = stateSubject.ofType(Showing.Success::class.java)
fun showingArView() = stateSubject.ofType(Showing.ArView::class.java)
    //non showing events
    fun arTargetFound() = stateSubject.ofType(State.ArTargetFound::class.java)
    
 
}
```
我们的 dispatcher 有几个功能：
- 我们可以用它来发送一个新的状态，这个状态告诉屏幕去做某些事。例如，dispatcher.dispatch（State.ArTargetFound） 会告诉我们的 AR 演示者显示一个找到的按钮。
- 我们可以用它来发送一个新的状态，需要显示一个新的屏幕。 dispatcher.dispatchShowing（Showing.NoNetwork） 给任何观察调度员的人发送一个新的显示状态  这个状态变化也被添加到我们的 showEvents 堆栈中。 （我们一会儿再谈这个问题 ）。
- 然后，presenter 可以监听特定的状态变化（带有相关的有效负载）并作出相应的反应 。例如，无网络时视图可以监听 dispatcher.noNetwork ( )。

登陆视图可以监听 dispatcher.showingLanding.subscribe ( )。

我们使用 Kotlin 密封类来表示我们可以调度的不同状态。 使用密封类可以使我们将某些状态表示为对象，而其他类可以是携带有效载荷的数据类。 

```java
sealed class State {
    data class ResolvingTarget(var resolveRequestData: ResolveRequestData) : State()
    object BackStackEmpty : State()
    object BeginCameraReveal: State()
}

sealed class Showing : State() {
    object Landing : Showing()
    object ArView : Showing()
    data class NoNetwork(var resolveRequestData: ResolveRequestData) : Showing()
    data class Success(var successMetaData: SuccessMetaData) : Showing()
    data class Failure(var failureActionMetaData: FailureActionMetaData) : Showing()
}
```
使用一个状态调度器，我们可以使 presenter 彼此分离，这意味着我们不需要将 presenter 彼此嵌套，或者有复杂的逻辑来定义何时应该显示或隐藏视图。 每个 presenter 将会监听他们需要的事件，并且监听他们应该关注的事件。 看看我们的 ARCamera 的 presenter 本身 attachView  会发生什么：

```java
override fun attachView(mvpView: AR, lifecycle: Lifecycle) {
    super.attachView(mvpView, lifecycle)
    disposables.add(dispatcher.showingArView()
            .subscribeOn(scheduler)
            .subscribe({ mvpView.showArView() }, { Timber.e(it) }))

    disposables.add(dispatcher
            .showing()
            .filter{(it !is Showing.ArView)}
            .observeOn(scheduler)
            .subscribe{mvpView.hide()})
}
```
与此同时，ARView 是可见的， 我们可能想显示一个 overlay  或一些额外的控制。 我们可以用一个 OverlayPresenter 来完成这个工作，它可以监听类似的事件: 

```java
disposables.add(dispatcher.showingArView()
        .subscribeOn(scheduler)
        .observeOn(scheduler)
        .subscribe({ mvpView.show() }, { Timber.e(it) }))


disposables.add(dispatcher
        .showing()
        .filter{(it !is Showing.ArView)}
        .observeOn(scheduler)
        .subscribe{mvpView.hide()})
```
 OverlayPresenter 可能也会关心其他状态，所以它也可以对这些状态做出反应：
```java 
disposables.add(dispatcher.arTargetFound()
        .subscribeOn(Schedulers.io())
        .observeOn(scheduler)
        .subscribe({ mvpView.targetAcquired() }, { Timber.e(it) }))

disposables.add(dispatcher.arTargetLost()
        .subscribeOn(Schedulers.io())
        .observeOn(scheduler)
        .subscribe({ mvpView.targetLost() }, { Timber.e(it) }))
```
我们架构的最后一部分是将我们所有的不同视图添加到一个具有可见性的单一布局:  不可见，这样当我们的活动开始时， 它们都可以被附加。 这反过来又会调用每个 presenter 的 attachView 函数，该函数将订阅视图需要响应的任何事件。 当用户浏览我们的 AR Activity 时，我们会根据发送的状态变化来显示和隐藏视图。 当我们的一个幸运用户在野外找到一个 AR 目标时，我们可以解决他们的位置，然后发送一个状态变化，这个状态变化将驱动下一个屏幕: 

```java
API.submit(request) }
        .subscribeOn(Schedulers.io())
        .observeOn(Schedulers.io())
        .subscribe({
            Timber.e("Submit:Success:" + it.success)
            if (it.success) {
                dispatcher.dispatchShow(Showing.Success(resultData))
            } else {
                dispatcher.dispatchShow(Showing.Failure(resultData))
            }
        )
```

看看我们的成功和失败 presenter 是多么简单：
```java
@ActivityScoped
class SuccessPresenter @Inject constructor(val dispatcher: Dispatcher) :
        BasePresenter<SuccessMVPView>() {
    override fun attachView(mvpView: SuccessMVPView, lifecycle: Lifecycle) {
        super.attachView(mvpView, lifecycle)
        dispatcher.showingSuccess()
                .subscribe({ bindData((it as Showing.Success).successMetaData) },
                        { Timber.e(it) })
        dispatcher
                .showing()
                .filter { it !is Showing.Success }
                .subscribe { mvpView.hideView() }

    }


    fun bindData(successMetaData: SuccessMetaData) {
        mvpView?.bindData(successMetaData)
    }
}


@ActivityScoped
class FailurePresenter @Inject constructor(val dispatcher: Dispatcher,
                                           @Android val scheduler: Scheduler) :
        BasePresenter<FailureBinder>() {
    override fun attachView(mvpView: FailureBinder, lifecycle: Lifecycle) {
        super.attachView(mvpView, lifecycle)
        dispatcher.showingFailure()
                .observeOn(scheduler)
                .subscribe({ bindData((it as Showing.Failure).failureActionMetaData) },
                        { Timber.e(it) })

        dispatcher
                .showing()
                .filter { it !is Showing.Success }
                .subscribe { mvpView.hide() }
    }


    fun bindData(failureActionMetaData: FailureActionMetaData) {
        mvpView?.bindData(failureActionMetaData)
    }
}
```
唯一剩下的部分是处理后台堆栈。 任何 view/activity/presenter 都可以调用 dispatcher.goBack ( )，它将在当前的状态之前发送最后一个显示状态，这样你的视图就会消失，另一个视图也会出现。   如果在我们的后台堆栈没有更多的显示状态，我们就会发送一个状态  State.BackStackEmpty，我们的活动监听：

```java
fun Dispatcher.goBack() {
    popLastShowingState()//current state is what we are currently showing
    dispatchShow(popLastShowingState())
}

fun popLastShowingState(): State {
    if(!showEvents.empty()) Timber.d("popping "+showEvents.peek())
    return if (showEvents.isEmpty()) State.BackStackEmpty else showEvents.pop()
}
```

ContainingActivity:
```java
dispatcher.backStackEmpty().subscribe { finish() }
...
override fun onBackPressed() {
   dispatcher.goBack()
}
```
就像常规的 Android 后台堆栈，我们的调度器后台是从我们的视图中抽象出来的。 任何视图都不应该知道当它完成时会发生什么，它只知道它被显示或隐藏。 这样做的好处在于，我们将每个状态变化和它的有效载荷一起记录到一个叫做 showEvents 的单一堆栈结构中。 视图不需要知道它是第一次显示，还是通过反向堆栈遍历来恢复。 顺便提一下，我们的状态变化是带有有效负载的 Kotlin 数据类，当您回退时，这些有效负载将随每个状态一起推送。 

这就是我们的新 MVP！ 它对我们有效，我们也希望它对你也有用。 下一篇文章我们将探讨在 JavaScript 库的帮助下，我们如何在不到一个月的时间里，使用目标和三维模型来实现扩增实境 

下面是一些基础类的大意，可以让你开始学习。 为更加彻底解决问题，我们希望能够把它变成一个库。

Code

Dispatcher.kt
```java
sealed class State {


    object BackStackEmpty : State()
    object SampleNonScreenChangeEvent: State()
}

sealed class Showing : State() {
    data class NoNetwork(var data: Data) : Showing()
    data class Success(var successData: SuccessMetaData) : Showing()
    data class Failure(var failureData: FailureMetaData) : Showing()
}

@Singleton
open class Dispatcher @Inject
constructor(val stateSubject: PublishSubject<State>) {

    val showEvents: Stack<State> = Stack()

    fun dispatch(state: State) = stateSubject.onNext(state)

    fun dispatchShow(state: State) {
        showEvents.push(state)
        Timber.d("pushing "+showEvents.peek())
        stateSubject.onNext(state)
    }

    fun showing() = stateSubject.filter { it is Showing }


    fun showingFailure() = stateSubject.ofType(Showing.Failure::class.java)

    fun showingSuccess() = stateSubject.ofType(Showing.Success::class.java)

    fun noNetwork() = stateSubject.ofType(Showing.NoNetwork::class.java)

    fun backStackEmpty() = stateSubject.ofType(State.BackStackEmpty::class.java)

    fun goBack() {
        popLastShowingState()//current state is what we are currently showing
        dispatchShow(popLastShowingState())
    }

    private fun popLastShowingState(): State {
        if(!showEvents.empty()) Timber.d("popping "+showEvents.peek())
        return if (showEvents.isEmpty()) State.BackStackEmpty else showEvents.pop()
    }
}
```
Presenter.kt 
```java
interface Presenter<in V : MvpView> {

    fun attachView(mvpView: V,lifecycle: Lifecycle)

    fun detachView()
}

open class BasePresenter<T : MvpView> : Presenter<T>, LifecycleObserver {

    protected val disposables = CompositeDisposable()

    var mvpView: T? = null
        private set

    val isViewAttached: Boolean
        get() = mvpView != null

    override fun attachView(mvpView: T,lifecycle: Lifecycle) {
        this.mvpView = mvpView
        lifecycle.addObserver(this)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    override fun detachView() {
        disposables.clear()
        mvpView = null
    }

    fun checkViewAttached() {
        if (!isViewAttached) throw MvpViewNotAttachedException()
    }

    class MvpViewNotAttachedException : RuntimeException("Please save Presenter.attachView(MvpView) before" + " requesting data to the Presenter")
}

interface MvpView
```

所有代码可以在  [[GIST]](https://gist.github.com/digitalbuddha) 查看。示例应用程序。[[PR]](https://github.com/digitalbuddha/Dispatcher)



