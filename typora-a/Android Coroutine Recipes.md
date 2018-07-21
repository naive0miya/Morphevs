# Android协程教程

> 原文 (Medium)：[Android Coroutine Recipes](https://proandroiddev.com/android-coroutine-recipes-33467a4302e9)
>
> 作者：[Dmytro Danylyk](https://proandroiddev.com/@dmytrodanylyk)

[TOC]

![](https://ws1.sinaimg.cn/large/006tNc79gy1fron0wkav3j30m80ci409.jpg)

## 如何启动协程

在 kotlinx.coroutines 库中，可以使用 [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html) 或 [async ](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html)函数启动新的协程。

从概念上讲，异步就像启动。 它启动一个单独的协程，它是一个与所有其他协程同时工作的轻量级线程。 不同之处在于启动会返回一个 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html)，而不会带来任何结果值，而异步返回 [Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html) - 一个轻量级的非阻塞 future ，表示稍后提供结果的承诺。 你可以使用延迟值的 .await ( ) 来得到最终的结果，但是 Deferred 也是一个 Job，所以你可以根据需要取消它。

如果启动程序中的代码以异常终止，那么它就像一个线程中的未捕获异常一样处理 Android 应用程序。 异步代码中的未捕获异常存储在生成的 Deferred 内部，并且不会在其他地方交付，除非进行处理，否则它将被静默放置。

## 协程的上下文

在 Android 中我们通常使用两个上下文：

- uiContext 将执行调度到 Android 主 UI 线程（用于父协程）。
- bgContext 在后台线程中调度执行（对于子协程）。

```kotlin
// dispatches execution onto the Android main UI thread
private val uiContext: CoroutineContext = UI
 
// represents a common pool of shared threads as the coroutine dispatcher
private val bgContext: CoroutineContext = CommonPool
```

在以下示例中，我们将 [CommonPool](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-common-pool/index.html) 用在 bgContext，它将并行运行的线程数限制为 Runtime.getRuntime.availableProcessors ( )) - 1的值。 因此，如果协调程序任务被调度，但所有内核都被占用，则它将被排队。

你可能要考虑使用 [newFixedThreadPoolContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-fixed-thread-pool-context.html) 或你自己的缓存线程池的实现。

**注意：**有一个打开的 [IO thread pool for blocking calls #79](https://github.com/Kotlin/kotlinx.coroutines/issues/79)

## 启动+异步（执行任务）

父协程通过 UI 上下文的启动功能启动。

子协程是通过具有 CommonPool 上下文的异步函数启动的。

**注意：**父协程总是等待其所有孩子的完成。

**注意：**如果发生未经检查的异常，应用程序将崩溃。

```kotlin
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
    val result = task.await() // non ui thread, suspend until finished
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/launch/single/MainActivity.kt#L64)

## 启动+异步+异步（顺序执行两个任务）

父协程通过 UI 上下文的启动功能启动。

通过具有 CommonPool 上下文的异步函数启动子协程。

**注意：** task1 和 task2 是按顺序执行的。

**注意：**如果发生未经检查的异常，应用程序将崩溃。

```kotlin
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    // non ui thread, suspend until task is finished
    val result1 = async(bgContext) { dataProvider.loadData("Task 1") }.await()
 
    // non ui thread, suspend until task is finished
    val result2 = async(bgContext) { dataProvider.loadData("Task 2") }.await()
 
    val result = "$result1 $result2" // ui thread
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/launch/sequentially/MainActivity.kt#L64)

## 启动+异步+异步（并行执行两个任务）

父协程通过 UI上下文的启动功能启动。 

通过具有 CommonPool 上下文的异步函数启动子协程。

**注意：** task1 和 task2 是并行执行的。

**注意：**如果发生未经检查的异常，应用程序将崩溃。

```kotlin
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    val task1 = async(bgContext) { dataProvider.loadData("Task 1") }
    val task2 = async(bgContext) { dataProvider.loadData("Task 2") }
 
    val result = "${task1.await()} ${task2.await()}" // non ui thread, suspend until finished
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/launch/parallel/MainActivity.kt#L64)

## 如何启动一个超时协程

如果要为协同作业设置超时，请使用 withTimeoutOrNull 函数来包装暂停的函数，该函数在超时的情况下将返回 null。

```kotlin
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
 
    // non ui thread, suspend until the task is finished or return null in 2 sec
    val result = withTimeoutOrNull(2, TimeUnit.SECONDS) { task.await() }
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/launch/timeout/MainActivity.kt#L63)

## 如何取消协程

loadData 函数返回一个可能被取消的 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html) 对象。当父协程被取消时，它的所有孩子也被递归地取消。

如果在 dataProvider.loadData 仍在进行时调用了 stopPresenting 函数，则 view.showData 函数将不会被调用。

```kotlin
var job: Job? = null
 
fun startPresenting() {
    job = loadData()
}
 
fun stopPresenting() {
    job?.cancel()
}
 
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
    val result = task.await() // non ui thread, suspend until finished
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/cancel/MainActivity.kt#L65)

## 如何处理与协程的异常

**try-catch** 块

父协程通过 UI 上下文的启动功能启动。

通过具有 CommonPool 上下文的异步函数启动子协程。

**注意：**你可以使用 try-catch 块来捕获异常并处理它们。

```kotlin
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    try {
        val task = async(bgContext) { dataProvider.loadData("Task") }
        val result = task.await() // non ui thread, suspend until task is finished
 
        view.showData(result) // ui thread
    } catch (e: RuntimeException) {
        e.printStackTrace()
    }
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/exception/trycatch/MainActivity.kt#L74)

要避免演示者类中的 try-catch，最好处理 dataProvider.loadData 函数内的异常，并使其返回一个通用的 Result 类。

```kotlin
data class Result<out T>(val success: T? = null, val error: Throwable? = null)
 
private fun loadData() = launch(uiContext) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
    val result: Result<String> = task.await() // non ui thread, suspend until the task is finished
 
    if (result.success != null) {
        view.showData(result.success) // ui thread
    } else if (result.error != null) {
        result.error.printStackTrace()
    }
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/exception/result/MainActivity.kt#L80)

## 异步+异步

父协程通过具有 UI 上下文的异步函数启动。

通过具有 CommonPool 上下文的异步函数启动子协程。

注意：要忽略任何异常，请使用异步函数启动父协程。

```kotlin
private fun loadData() = async(uiContext) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
    val result = task.await() // non ui thread, suspend until the task is finished
 
    view.showData(result) // ui thread
}
```

在这种情况下，异常将被存储在一个 Job 对象中。要检索它，你可以使用 invokeOnCompletion 函数。

```kotlin
var job: Job? = null
 
fun startPresenting() {
    job = loadData()
    job?.invokeOnCompletion { it: Throwable? ->
        it?.printStackTrace() // (1)
        // or
        job?.getCompletionException()?.printStackTrace() // (2)
 
 
        // difference between (1) and (2) is that (1) will NOT contain CancellationException
        // in case if job was cancelled
    }
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/exception/ignore/MainActivity.kt#L72)

## 启动+协程异常处理程序

父协程通过 UI 上下文的启动功能启动。

通过具有 CommonPool 上下文的异步函数启动子协程。

**注意：**你可以将 CoroutineExceptionHandler 添加到父协同程序上下文来捕获异常并处理它们。

```kotlin
val exceptionHandler: CoroutineContext = CoroutineExceptionHandler { _, throwable -> throwable.printStackTrace() }
 
private fun loadData() = launch(uiContext + exceptionHandler) {
    view.showLoading() // ui thread
 
    val task = async(bgContext) { dataProvider.loadData("Task") }
    val result = task.await() // non ui thread, suspend until the task is finished
 
    view.showData(result) // ui thread
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/main/java/com/dmytrodanylyk/exception/handler/MainActivity.kt#L71)

## 如何测试协程

要启动协程，你需要指定一个 [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines.experimental/-coroutine-context/index.html)。

```kotlin
class MainPresenter(private val view: MainView,
                    private val dataProvider: DataProviderAPI) {
 
    private fun loadData() = launch(UI) { // UI - dispatches execution onto Android main UI thread
        view.showLoading()
 
        // CommonPool - represents common pool of shared threads as coroutine dispatcher
        val task = async(CommonPool) { dataProvider.loadData("Task") }
        val result = task.await()
 
        view.showData(result)
    }
 
}
```

如果要为 MainPresenter 类编写单元测试，则需要为UI和后台执行指定一个协程上下文。

最简单的方法是向 MainPresenter 构造函数中添加两个参数：UI 默认值为 uiContext，默认值为 CommonPool 的 ioContext。

```kotlin
class MainPresenter(private val view: MainView,
                    private val dataProvider: DataProviderAPI
                    private val uiContext: CoroutineContext = UI,
                    private val ioContext: CoroutineContext = CommonPool) {
 
    private fun loadData() = launch(uiContext) { // use the provided uiContext (UI)
        view.showLoading()
 
        // use the provided ioContext (CommonPool)
        val task = async(bgContext) { dataProvider.loadData("Task") }
        val result = task.await()
 
        view.showData(result)
    }
}
```

现在你可以通过提供一个 [Unconfined](https://github.com/Kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/src/main/kotlin/kotlinx/coroutines/experimental/CoroutineContext.kt#L55) 来轻松地测试你的 MainPresenter 类，它将只在当前线程上执行代码。

```kotlin
@Test
fun test() {
    val dataProvider = Mockito.mock(DataProviderAPI::class.java)
    val mockView = Mockito.mock(MainView::class.java)
 
    val presenter = MainPresenter(mockView, dataProvider, Unconfined, Unconfined)
    presenter.startPresenting()
    ...
}
```

[See full source code.](https://github.com/dmytrodanylyk/coroutine-recipes/blob/master/app/src/test/java/com/dmytrodanylyk/MainPresenterTest.kt)

相关的文章和特别的感谢： [Guide to coroutines by example](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md), [Roman Elizarov](https://stackoverflow.com/questions/46226518/what-is-the-difference-between-launch-join-and-async-await-in-kotlin-coroutines),[Jake Wharton](https://www.reddit.com/r/androiddev/comments/7570dv/android_coroutine_recipes/), Andrey Mischenko

