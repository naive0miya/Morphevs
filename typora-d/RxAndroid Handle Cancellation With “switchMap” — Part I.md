# RxJava：使用 “switchMap” 处理中断

> 原文 (Medium)：[RxJava: Handle Cancellation With “switchMap”](https://tech.pic-collage.com/rxandroid-handle-interrupt-with-switchmap-3a650393299f)
>
> 作者：[Boy Wang](https://tech.pic-collage.com/@boyw165?source=post_header_lockup)

[TOC]

> 2018年2月19日更新：用 takeUntil 添加 “尝试＃3”。

安卓应用有太多的中断。例如：当一个用户正在浏览一个新闻源，一个来电是一个中断，用户点击系统后退按钮是一个中断，登录时切换活动是一个中断，等等。

中断怎么会对应用程序造成伤害？ 想想这个案例: 

当中断发生时，活动被推送到后台，一个异步任务刚刚完成其工作，您的代码响应结果通过访问用户界面。 砰！ 由于活动不再活跃，引发一个 IllegalStateException  或 NullPointerException。 通过 Don’t Keep Activity 模式，这一点是显而易见的。 

通常，您一定要在 onDestroy 中的取消注册访问活动的  UI 的异步任务的所有回调。当需要处理很多异步任务时，代码变得更加复杂。	

[RxJava](https://github.com/ReactiveX/RxJava)  非常擅长用户界面管理应用程序的状态。 它所做的最好的事情就是分离关注，并且有一个链条回收过程。 如果你曾经使用过 RxJava，你会注意到可观察者和命令回调之间的一个很大的区别是有效负载中的数字参数。 不像回调函数和你想要的参数一样多，可观察者到只接受一个下游的东西。 这个设计困扰了我很多，现在我发现它隐含地迫使你做好分离关注通过给每个可观察者的最少的责任。 如果每个可观察到的都承担起最小的责任，一个有效负载就足够了！ 

## 案例分析

在 PicCollage 应用程序中，有一个共享到社交网络功能。当用户点击共享按钮时，需要处理一系列异步任务：

1. 显示进度条并生成一个用于共享的高清位图。 当任务完成时，隐藏进度栏 
2. 显示一个对话框，要求用户填写邮件 
3. 如果前面的对话被确认，显示进度栏和共享目标社交网站。 当共享任务完成时，隐藏进度栏 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotgt26nuj30m80543z1.jpg)

问题是: 用户可以在三个步骤的任何步骤中取消，所以设计系统来处理取消？ 

## 使用 Observable

我喜欢 Observable 的一个原因是链式回收处理。 Rxjava 基本上是一个构建双链表的 lambda 函数(代码块)和每个可观察者(链表中的一个节点)的双链表的框架。 因此，下游(又称观察者)将能够通过调用给定的 dispose ( ) 来要求源代码停止，然后处理过程从底到顶，一直到最上游。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotgxkkdzj30hm0d4js5.jpg)

## 尝试＃1，天真的方法：

我必须生成一个高清位图，一个弹出对话框和一个共享对话框可观察者 ，并将它们与 switchMap 操作符连接在一起(请记住，switchMap 是这样做的... ... 我们用它来。 我还持有返回的 sharingDisposable，以便稍后终止流。 例如: 

```kotlin
// Hold the disposable in order to cancel the operation later.
val sharingDisposable = mView
    .onClickStart()
    .switchMap { _ ->
        // 1. Generate BMP
        getGenBmpObservable()
            // 2. Show dialog
            .switchMap { _ -> 
                getDialogSingle().toObservable()
            }
            // 3. Share
            .switchMap { dialogPayload -> 
                if (dialogPayload.result == RESULT_OK) {
                    // If the user clicks OK, share to FB.
                    getShareToFacebookObservable(dialogPayload.data)
                } else {
                    // The EMPTY observable calls observer's onComplete()
                    // to indicate the stream is over.
                    Observable.empty()
                }
            }
    }
    .subscribe { _ ->
        Log.d("xyz", "all finished!")
    })

// When the cancel signal is received, dispose the observable chain.
onClickCancel()
    .subscribe { _ ->
        sharingDisposable.dispose()
    }
```

图如下所示：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1froth21ahrj30m80ghta5.jpg)

我持有 sharingDisposable，以便我可以调用 sharingDisposable.dispose ( ) 来取消未完成的作业。

然而，这个天真的实现有一个关键问题：调用 dispose ( ) 会终止整个流，这意味着您不能再接收按钮点击 。为了使按钮点击再次工作，您必须在取消之后重新连接流。 

另一个问题是我们必须在系统后退按钮，进度对话框取消和自定义取消按钮的回调中共享 sharingDisposable。

无论如何，这些代码还不够简洁，而且还没有响应式编程。 

## 尝试＃2，使用 “switchMap” 的反应式方法：

反应式编程的精神之一就是事件系统的单向前进方向。组件是一组函数，它们采取输入和计算输出。 因此，你的业务逻辑中几乎没有状态变量。 

在前面的例子中，sharingDisposable 是一种在不同函数之间共享的状态。 是否有可能不需要一遍又一遍地重新连接共享流？ 

是的，我们可以！ 我们可以将取消(或中断)一个输入到流中，它除了替换正在进行的任务之外什么也不做。 通过这样做，点击共享流就会保持活力。 

我们将共享按钮的输出和取消合并为一个可观察者，然后用一个 switchMap 将合并的可观察者连线。switchMap 产生 share-to-Facebook 动作和取消动作。图如下所示：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frothcpx56j30m80ghmyk.jpg)

代码片段如下所示：

```kotlin
private val mDisposablesOnCreate = CompositeDisposable()

private val INTENT_OF_DO_SOMETHING      = 0
private val INTENT_OF_CANCEL_EVERYTHING = 1

// Activity onCreate().
override fun onCreate() {
    // Share and cancel input:
    //
    //   start +
    //          \
    //           +----> something in between ----> end.
    //          /
    //  cancel +
    mDisposablesOnCreate.add(
        Observable
            .merge(
                // Share intent.
                onClickShare()
                     .map { _ -> INTENT_OF_DO_SOMETHING },
                // Cancel intent.
                onClickCancel()
                     .map { _ -> INTENT_OF_CANCEL_EVERYTHING })
            // Create a share action or a cancel action.
            .switchMap { intent ->
                when (intent) {
                    INTENT_OF_DO_SOMETHING -> toShareAction()
                    INTENT_OF_CANCEL_EVERYTHING -> toCancelAction()
                    else -> toCancelAction()
                }
            }
            .subscribe { _ ->
                Log.d("xyz", "all finished!")
            })
}

// Activity onDestroy()
override fun onDestroy() {
    mDisposablesOnCreate.clear()
}

// "Share" button observable.
fun onClickShare(): Observable<Any> { ... }
// "Cancel" button observable.
fun onClickCancel(): Observable<Any> { ... }
```

行动代码如下所示：

```kotlin
private fun toShareAction(): Observable<ShareProgressState> {
    // 1. Generate BMP
    return getGenBmpObservable()
        // 2. Show dialog
        .switchMap { _ -> 
            getDialogSingle()
                .toObservable()
        }
        // 3. Share
        .switchMap { dialogPayload -> 
            if (dialogPayload.result == RESULT_OK) {
                // If the user clicks OK, share to FB.
                getShareToFacebookObservable(dialogPayload.data)
            } else {
                // The EMPTY observable calls observer's onComplete()
                // to indicate the stream is over.
                Observable.empty()
            }
        }
}

/**
 * Returns a CANCEL action.
 */
private fun toCancelAction(): Observable<ShareProgressState> {
    return Observable.just(ShareProgressState(justStop = true))
}
```

它之所以有效，是因为取消操作会产生一个取消操作，它取代了之前的操作。 通过替换，旧的操作流在附加新的操作流之前被处理掉。 魔术是由 [switchMap](http://rxmarbles.com/#switchMap) 操作符完成的。 

## 尝试＃3，使用 “switchMap” 和 “takeUntil” 更好的反应式方法：

到目前为止一切顺利，可观察的链条在取消信号的作用下仍然存活。 然而，前面的方法将两个操作结合在一起，共享和取消操作都会产生 ShareProgressState。 为什么取消操作会产生一个 ShareProgressState？ 这完全没有意义，而且当有更多的操作时，代码也不是很灵活。 

我们可以使用 takeUntil 操作符简化代码。当第二个 observable（作为参数给出）发出一个 item 时，takeUntil 会产生一个特殊的 observable，它会自行终止。

> takeUntil 操作符实质上是这样说的：“我对这个流/可观察源感兴趣，直到在这个流/可观察者上发生了什么”。 - Jaime Cham

```kotlin
private val mDisposablesOnCreate = CompositeDisposable()

// Activity onCreate().
override fun onCreate() {    
    // Long computation operation.
    mDisposablesOnCreate.add(
        // Share button.
        onClickShare()
            .switchMap { _ ->
                // First, switchMap convert the click to an action observable.
                // Whenever a new click is received, switchMap interrupt and
                // kills the existing ongoing observable and replace it with
                // the new one.
                // Second, subscribe to the action observable with a cancel
                // throttle where the takeUntil self terminates and also kill
                // the action observable when a cancel signal is received.
                toShareAction()
                    .takeUntil(mCancelSrc)
            }
            .subscribe { _ ->
                Log.d("xyz", "all finished!")
            })

    // Cancel signal.
    mDisposablesOnCreate.add(
        onClickCancel()
            .subscribe { _ ->
                mCancelSrc.onNext(0)
            })
}

// Activity onDestroy()
override fun onDestroy() {
    mDisposablesOnCreate.clear()

// "Share" button observable.
fun onClickShare(): Observable<Any> { ... }
// "Cancel" button observable.
fun onClickCancel(): Observable<Any> { ... }
```

它看起来像：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frothhag5aj30m808i0tj.jpg)

## “switchMap” 工作原理

这部分是可选的，我会深入研究 switchMap 并解释这个魔术。  ✨

switchMap 将接收到的信号转换为 observable，并等待 observable 发出的结果并将结果绕到其下游。无论何时接收到新信号，switchMap 都会中断并杀死现有的可观测信号，并将其替换为新信号。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frothksrk3j30m80c5gnp.jpg)

switchMap 非常像 flatMap，唯一的区别是 switchMap 中只有一个活动的内部观察者，而不是在 flatMap 中有一个活动的内部观察者列表。

switchMap 有两个内部观察者，其中＃1是订阅外部上游，＃2是订阅映射器返回的 Observable。它持有内置的 Disposable，并在新的 Observable 附加之前进行处理。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frothnso0aj30b40ckt9f.jpg)

当内部流被处置时，链式回收过程在 switchMap 内部发生，如下所述。两个内部观察者是 SwitchMapObserver（no. 1）和 SwitchMapInnerObserver（no. 2）。代码片段向您展示了 [SwitchMapObserver](https://github.com/ReactiveX/RxJava/blob/2.x/src/main/java/io/reactivex/internal/operators/observable/ObservableSwitchMap.java#L99-L130) 中发生了什么：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frothruluyj30m80fytbu.jpg)

```kotlin
// This is the snippet of SwitchMapObserver code.

// 1. The inner observer #1 's onNext, this function is called when it 
// receives the data from upstream.
@Override
public void onNext(T t) {
    long c = unique + 1;
    unique = c;

    // 2. Cancel (dispose) the given Observable from your mapper funtion.
    SwitchMapInnerObserver<T, R> inner = active.get();
    if (inner != null) {
        inner.cancel();
    }

    ObservableSource<? extends R> p;
    try {
        p = ObjectHelper.requireNonNull(mapper.apply(t), "The ObservableSource returned is null");
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        s.dispose();
        onError(e);
        return;
    }

    // 3. Create a new observer #2 and subscribes it to the given Observable.
    SwitchMapInnerObserver<T, R> nextInner = new SwitchMapInnerObserver<T, R>(this, c, bufferSize);

    for (;;) {
        inner = active.get();
        if (inner == CANCELLED) {
            break;
        }
        if (active.compareAndSet(inner, nextInner)) {
            p.subscribe(nextInner);
            break;
        }
    }
}
```

### Disposable

那么当 Disposable 的 dispose ( ) 被调用时会发生什么？首先，Disposable 由 Observable 提供给观察者，以便观察者能够请求其源停止。它本质上是完成从下到上的链接的桥梁。

您有责任终止或回收 dispose ( ) 或 onDispose ( ) 中的任何未完成的任务。

### 示例代码

看看我的 Github 项目。[PR](https://github.com/boyw165/my-rx-exp/blob/master/app/src/main/java/com/my/exp/rx/rxCancel/RxCancelPresenter.kt)

## 结束

1. RxJava 本质上是一个构建 lambda 函数的双链表的框架。它具有一个状态流的概念，而不是许多只有起始状态和结束状态的延期库。
2. 将您想要立即取消的 Observable 链作为一个大 Observable 进行封装。
3. 用 takeUntil 调节可观察者的动作。
4. 使用 switchMap 替换未完成的操作流。这个操作符会在将当前 Observable 连接到它的内部观察者之前处理它。
5. 确保在 switchMap 中返回一个 COLD Observable。返回像 Subject 这样的 HOT Observable 会产生大量的链条泄漏 。

最后，我强烈建议您研究 [RxAndroid](https://github.com/ReactiveX/RxAndroid) 或 [RxBinding](https://github.com/JakeWharton/RxBinding) 的代码以获得更好的理解。通过在 [ObservableSwitchMap](https://github.com/ReactiveX/RxJava/blob/2.x/src/main/java/io/reactivex/internal/operators/observable/ObservableSwitchMap.java) 或 [BasicFuseableObserver](https://github.com/ReactiveX/RxJava/blob/2.x/src/main/java/io/reactivex/internal/observers/BasicFuseableObserver.java) 中设置断点，使得代码跟踪变得更加容易。 

谢谢阅读。如果你认为这篇文章有帮助，请不要犹豫，给予鼓掌。或者，如果您发现了我犯的一些错误，请随时在这里评论。 ✨

## 阅读

- Jaime Cham 撰写的博客文章 [“通过 RxJS 进行反应式编程的简洁介绍”](https://tech.pic-collage.com/a-gentle-introduction-to-reactive-programming-via-rxjs-52d801228763)。
- Sachin Chandil 的[博客文章](https://android.jlelse.eu/rxjava2-dispose-an-observable-chain-684e6ca2790)提出了定制 disposable  Observable 的想法。
- Uber 已经发布了一个有趣的 [RxJava 库]( https://uber.github.io/AutoDispose/)，可以自动终止链条

