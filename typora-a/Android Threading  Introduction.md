# Android 线程：简介

> 原文 (Medium)：[Android Threading : Introduction](https://medium.com/@maheswaranapk/android-threading-introduction-90e91e196600)
>
> 作者：[Mahes Sivakumar](https://medium.com/@maheswaranapk)

[TOC]

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron6eho3jj30m80ettbd.jpg)

当用户启动应用程序时，Android 为核心任务如 UI 绘图创建一个进程和主线程( UI 线程)。 这个主线程处理来自应用程序的所有事件: 应用回调，输入事件回调，服务等等..。 . 这些回调可以触发其他运行在主线程上的工作，如用户界面、计算等..。 

我们要运行的任何代码块都将直接在主线程上运行。 当应用程序在用户交互的基础上执行密集的工作时，这个单一的线程就会给你带来糟糕的性能。 如果 Main Thread 被一些代码块阻止，那么 App 就挂起来了。 屏蔽几秒钟的 UI 线程将启动 [ANR](https://developer.android.com/topic/performance/vitals/anr.html) [应用程序不响应]对话框。 

因此，将更长的工作移动到其他线程上不会干扰 UI 渲染是很有意义的。 这就是在安卓系统中线程的重点。 

**注意：** Android UI 工具包不是线程安全的。 所以，避免使用工作线程中的 UI 组件。 

提供了很多方法来卸载这些工作。 知道什么样的原始环境可以减轻头痛。 

- **HandlerThread** - 线程回调。
- **AsyncTask** - 保持开启/关闭 UI 线程的工作。
- **ThreadPoolExecutor** - 用于并行工作。
- **IntentService** - 保持 UI 线程的任务

## HandlerThread :

通常 Java 线程是一次性可用的。它在执行运行方法后死亡。

如果一个线程可以坚持一段时间，那么我们可以为它做一些工作。为此，我们需要小包装。

1. 循环机制保持线程活着。
2. 队列在哪里可以拉工作和执行
3. 我们需要一个创建工作并将其推入队列的线程。

Android Framework 已经内置了类来做这些事情。

[**Looper**](https://developer.android.com/reference/android/os/Looper.html) - 类用于为线程运行消息循环并保持线程活着。

[**MessageQueue**](https://developer.android.com/reference/android/os/MessageQueue.html) - 持有由 Looper 发送的消息列表的类。

[**Handler**](https://developer.android.com/reference/android/os/Handler.html) - 允许你发送和处理与线程的 MessageQueue 关联的 Message 对象。

所有这些东西结合在一起就是 HandlerThread。

HandlerThread 是一个长时间运行的线程，从队列中取出工作并对其进行操作。

你可以将多个工作发布到 HandlerThread 中。工作将按顺序执行。所以，长时间运行的阻塞任务将使所有短暂运行的任务等待。

一个更好的方法是将长时间运行的任务分离到单独的 HandlerThread 或使用其他线程技术。

**何时使用 HandlerThreads： **

HandlerThreads 是不需要 UI 更新的完美解决方案。

## AsyncTask :

AsyncTask 允许你执行后台操作并在 UI 线程上发布结果，而无需操纵线程和/或处理程序。

使用 AsyncTask，你可以执行后台任务，然后将结果发布到 UI 线程以更新 UI。

最常用的方法是：

1. **onPreExecute( )** - 在线程开始运行之前调用UI线程。通常用于设置任务。
2. **doInBackground(Params…)** - 在后台线程上运行。将在 onPreExecute ( ) 之后立即被调用。完成后，它将结果发送到 onPostExecute ( )。
3. **onProgressUpdate( )** - 调用在 UI 线程中调用的 doInBackground ( ) 中的 publishProgress ( ) 来进行 UI 更新时调用。
4. **onPostExecute(Result)** - 在后台线程完成后调用 UI 线程。

当我们为正在运行的任务调用 cancel ( ) 时，任务将不会被终止。我们需要通过检查来处理逻辑是取消任务。

**执行顺序：**

默认情况下，所有创建的 AsyncTask 将共享相同的线程，并以单个消息队列的串行方式执行。

我们执行 20 个工作，如果第三个工作需要更多的时间，那么将阻止其他 17 个工作。

这很危险

有一种方法可以使用 THREAD_POOL_EXECUTOR 以线程池方式执行 AsyncTask

**何时使用 AsyncTask：**

AsyncTask 是短期工作的完美解决方案，可以快速结束，需要经常进行 UI 更新。

## ThreadPoolExecutor :

当使用线程到最大值时，它是管理所有任务的艰巨任务。

相反，我们可以使用 ThreadPoolExecutor 将大量工作分解为小任务。

考虑我们需要解码 20 个图像，每个图像需要 2 秒钟解码。把所有的工作放在一个单一的线程中，需要 40 秒。这是一个坏主意。

相反，我们创建了 5 个线程，每个线程解码 4 个图像，然后在 8 秒内完成任务。

然后我们会遇到如何绕过线程，安排工作，管理线程生命周期，负载平衡等问题。

这些是 ThreadPoolExecutor 的工作。只要告诉线程数量并发布任务，ThreadPoolExecutor 就会处理所有事情。

基本上，ThreadPoolExecutor 在其中一个池中执行给定的任务。

在创建 ThreadPool 时，可以设置活动线程的数量和池中的最大线程数。使用我们可以避免 CPU 超载。

**何时使用 ThreadPoolExecutor：**

ThreadPoolExecutor 可以用来当我们有很多很多任务要做的时候。否则，我们可以使用 HandlerThreads 或 AsyncTask。

## IntentService :

在 Android 中，Service 和 Intents 在 UI Thread 中被执行。而且它不适合任何渲染或后台工作。

考虑你想回应一个意图，这项任务将需要更多的时间来完成。一般的解决方法是，我们将在主线程中收到请求，并将工作推送到其他线程。

andlerThreads 可能是一个选项。但是当工作不到的时候，这个线程就要坐下来，拿一些资源。

IntentService 是 Service 和 HandlerThread 的混合体。它扩展了 Service 类并创建了一个 HandlerThread 来处理意图。

它创建线程并将任务移动到它。

它使用 “ 工作队列处理器 ” 模式。如果一个任务需要很长时间，那么队列中的其他任务将被阻塞。

从 IntentService 传递结果的最好方法之一是使用 LocalBroadcastManager。

**何时使用 IntentService：**

这对于意图性的工作是非常有用的。例如：IntentService 可以与 AlarmManager 一起使用，以执行计划工作等等。

我们可以单独使用 HandlerThreads 来做任何事情。 IntentService 提供了一个服务的好处。





Android 平台充斥着线程。线程和内存总是不能正常工作。所以，请始终保持线程安全。

谢谢阅读！ ❤️。请让我知道你对此的看法。