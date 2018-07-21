# 优化 Android 线程和后台任务开发

> 原文：[优化 Android 线程和后台任务开发](https://academy.realm.io/cn/posts/android-threading-background-tasks/)
>
> 作者：[Ari Laceski](https://twitter.com/tensory)

[TOC]

在 Android 开发中，你不应该做任何阻碍主线程的事情。但这究竟意味着什么呢？在这次[海湾 Android 开发者大会](http://www.meetup.com/bayareaandroid/)讲座中，[Ari Lacenski](https://twitter.com/tensory) 认为对于长时间运行或潜在的复杂任务要特别小心。这一次演讲，我们将根据一个真实场景应用的需求，讨论 `AsyncTask`, `Activity`, 和 `Service`，逐步建立一个更易于维护的解决方案。

------

## Android 线程 [(0:46)](javascript:presentz.changeChapter(0,1,true);)

当我们谈论线程，我们知道一个 Android 应用程序至少有一个主线程。这个线程是随着你的 `Application` 类的创建而同时创建的。主线程的关键职责就是绘制用户界面：它是为了处理用户交互，在屏幕上绘制像素，并进行加载 Activity。任何你添加到 `Activity` 里和加载资源有关的代码，都将被放置于主线程处理结束才运行，因此应用程序才能保持对于用户操作的即时响应。

你可能认为这是没有什么大不了的，例如，您可以有一个列表视图（ListView），建立一个列表，并添加一些项目。你可以将列表数据排序，然后把它传给一个列表适配器。有了资源，处理器将尝试为您绘制列表。

然而，你添加到 `Activity` 中的一些代码可能需要比较长时间的计算，它们不应该是在主线程中进行运行的。一个比较常见的例子是：在你的应用程序中尝试做一个网络调用。你可以试着去获得一个 `URLConnection` 的默认实例，并在 `Activity` 中运行它。这在编译的时候是不会出什么问题，但在运行时，你将会得到一个 `NetworkOnMainThreadException`。

让我们再回到一个不太极端的例子，回到列表视图。对于一个非静态列表，你想查询本地数据库，并将结果插入你的列表中。如果你使用 [Realm](http://realm.io/)，那么在 UI 线程（主线程）做这个查询是没有问题的。但是，让我们假设你使用的是 SQLite 数据库。做一个查询请求需要大量的代码，访问数据库引擎，得到的结果，对它们进行排序，并把它们交给你的 `ArrayAdapter`。如果所有的这一切都运行在 `Activity`，你的用户界面交互可能就很不顺畅了。

以前你可能会觉得，在 UI 线程读取 shared preferences，并没有什么不好的事情发生。它能有什么影响？随着你越来越多的任务，去中断或抢占 UI 线程的时间，应用程序将更多地倾向于跳过或滞后处理一些 UI 更新，经常就是导致动画将无法正确更新。用户可能会困惑这个应用是怎么了，什么可能发生最坏的情况就是，如果用户等待了一定长时间，他们会得到的系统发出的错误消息说，“应用程序没有响应。你想继续等待还是杀死这个应用？”这就相当于你走在路上，突然被贴上了一个“干掉我”的标签。就算用户留下来了他们也可能不知道当前应用程序是否是值得信赖的了。

> “每个线程都会分配一个私有内存区域，该区域主要用于在执行过程中存储方法、局部变量和参数。一旦线程终止私有内存区将会被销毁。”
>
> - Anders Göransson, *《高效 Android 线程开发（Efficient Android Threading）》* 一书作者

我真的很喜欢这句话，它来自 [*Efficient Android Threading*](http://shop.oreilly.com/product/0636920029397.do) 这本书，这本书讲了什么是线程和线程做什么事情。它可以帮助我了解什么是后台线程，以及它们是如何在主线程中运行的。这句话也是提醒你注意一个事实，你是你创造的线程的调度者，你应该对他们负责。Android 有很多选项或方式来协助你进行线程调度。但你应该牢记 如果你不是在主线程进行一些操作，那么你要知道你的哪些线程是目前正在运行的。

最后一点我想提出的是，对于多线程，每次执行跳跃从一个线程到另一个，会有一个短暂时间的延迟。这就是所谓的上下文切换。接下来我们将更多地讨论类似这样的链式操作，和优化一些复杂的任务系列。

### `AsyncTask` [(5:51)](javascript:presentz.changeChapter(0,3,true);)

现在，我想谈谈几个 Android SDK 提供的管理线程的工具。首先就是  `AsyncTask`，它能够使你很方便地使用它在 `Activity` 中执行异步任务。但这仅仅是你 *能够* ，而不是你“应该”。

在 AsyncTask 中你首先会看到的两个方法之一就是 `doInBackground`，这是你可以在其中做耗时任务的后台线程调用的方法。第二种方法是 `onPostExecute`，它将运行于主线程。你可以  `doInBackground` 发送信息给 `onPostExecute`，然后继续执行后台任务。

那么 `AsyncTask` 的原理是什么呢？你可以 *按住ctrl + 鼠标单击* 这个类的名字进入到它的源代码文件。我觉得真的值得去阅读 `AsyncTask` 的代码，因为它会告诉你很多关于 Android 的线程模型是什么样的、它们是怎么运行的。

```java
// AsyncTask.java
private static class InternalHandler extends Handler {
		@Override
		public void handleMessage(Message msg) {
			AsyncTaskResult result = (AsyncTaskResult) msg.obj;
			switch (msg.what) {
				case MESSAGE_POST_RESULT:
					// There is only one result
					result.mTask.finish(result.mData[0]);
					break;
				case MESSAGE_POST_PROGRESS:
					result.mTask.onProgressUpdate(result.mData);
					break;
			}
		}
}
```

这个类中最有趣的部分是有一个 handler 实例。handler 的主要工作是提供一种替代方法，以编写一系列方法，并希望它们能够互相调用，完成一系列的任务。当你向他发生事件的时候，在它的 `handleMessage` 中可以使用 `switch` 来执行不同的响应代码。 

`AsyncTask` 通过 `execute` 来启动并执行一个线程。然后，你写在 `doInBackground` 中的代码就会自动运行。你无需关心的一点是，handler 会发送一个 `MESSAGE_POST_RESULT` 信号。这个信号使得 `onPostExecute`  被调用。这听起来很方便，但要记住在使用它的时候，不要随着时间的推移使简单的任务变得越来越复杂，那时 `AsyncTask` 可能就会变得很臃肿和难以控制。

让我们假设我所描述的情况实际上并不足以满足你的需要。那么可以看看接下来这个例子，你有两个任务，需要按顺序运行，这两个都是异步实现的。因为你已经使用 `AsyncTask` 运行一个后台线程，你可能会开始的第一个任务，然后在 `doInBackground` 中启动第二任务。这是 Android 线程模型不允许的事情。因此，可能的做法就是等待第一个任务完全结束再于主线程中启动第二个任务。这在 JavaScript 中也经常会类似这么回调。不过，虽然你可以这么做，但代码上看起来就经常会很糟糕：

```java
new AsyncTask<Void, Void, Boolean>() {
	protected Boolean doInBackground(Void... params) {
		doOneThing();
		return null;
	}
	protected void onPostExecute(Boolean result) {
		AsyncTask doAnotherThing =
			new AsyncTask<Void, Void, Boolean>() {
			protected Boolean doInBackground(Void... params) {
				doYetAnotherThing();
				return null;
			}
			private void doYetAnotherThing() {}
		};
		doAnotherThing.execute();
	}
	private void doOneThing() {}
}.execute();
```

如果你仍然感到困惑，觉得这个代码并不会太难以阅读。但我要告诉你这个代码是很 *糟糕* 的，它显得混乱和嵌套严重。原因是因为在这里，你开启了一个新的线程，请求 `AsyncTask` 的整个生命周期去执行你的第一个任务。你调用 `doOneThing`，然后返回到UI线程。就在那一刻，你又一次的阻塞了用户界面线程，因为没有其他的办法去开始另一个后台任务线程。这点很重要。你是交织地使用 UI 线程去管理你的线程，但实际上没有一个很好的理由这样做。正如我之前提到的，你每次这么做的时候都要引起一个计算延迟或称主线程卡顿。而且，事实上，这是一个不可维护的代码块，基于设计和架构的理由我们不应该这么做。

在这种情况下，我们有一个复杂的任务，有一些其他的事情。Android SDK 提供了许多的选项，因此，对于大多数情况，你不必处理启动和管理自己的线程。

### `IntentService` [(11:57)](javascript:presentz.changeChapter(0,7,true);)

如果你认为我是在为你使用 Services 的话题升温的话，你是对的。我用了一个 `IntentService` 实施以下用例。通过 `IntentService`，可以顺接执行另一个线程。它可以运行在完全独立而不需要 UI 线程做新线程或新任务的交织，它甚至可以在应用程序不在前台的时候进行运行。使用 `IntentService` 的特别好处是你不必自己去启动和关闭线程。

所有 services 都是从 activity 开始的。它们负责更重的责任，并且初始化也比 `AsyncTask` 更难一些，但它们也只是需要更多的设置和在 `Manifest`  文件中声明一下而已。你可以通过 `Intent Bundle` 来传递你的意图。我们会用一个例子来展示如何使用 Activity 和 后台任务 进行交互。

### Activity 和 Service 的通讯[(13:29)](javascript:presentz.changeChapter(0,8,true);)

提供一个关于这个话题的最佳实践：[CodePath guides](https://guides.codepath.com/android/Starting-Background-Services)。现在， `Activity` 启动了一个 `Service` 。`Activity` 还感兴趣的是能够与 `Service` 沟通，这样我们就可以知道什么时候服务结束了。我们可以创建 `ResultReceiver` 对象来给出反馈。通过 Intent 将这个 receiver 传递到 `Service` 。这样以后，`Service` 有一个 `ResultReceiver` 对象，将运行用户界面代码。`Service` 去做它应该做的。在最后的时刻，我们在 `onHandleIntent` 结束之前调用  `rr.send`。我们可以发送一个 `RESULT` 标志和一个数据包。所以，当你调用 `rr.send`，你的 `Activity` 就能接收到它了。

```java
// Activity
ResultReceiver rr = new
ResultReceiver() {
	@Override
	onReceiveResult() {
		// do stuff!
	}
}
// add rr to extras
startService(myServiceIntent)

// Service
onHandleIntent(Intent i) {
	ResultReceiver rr =
	i.getExtras().get(RR_KEY)
	(assigned from bundle)

	rr.send(Activity.RESULT_OK
	successData);
}
```

## 设计任务通信 [(14:57)](javascript:presentz.changeChapter(0,9,true);)

假设现在，我们已经建立起了越来越多的复杂任务。我想分享一个我在“芒果健康”所做的工作内容，当时我在写应用程序登录部分的解决方案。对于我们的应用程序，它是可以无需登录而使用的。然后，我们向用户提供登录的功能，以便他们能够把本地数据同步到远程服务器。如果我们想支持与远程服务器同步数据的话，登录过程的设计就变得比较重。所以，为了帮助我设计登录过程，我给它定了五个原则：

1. 最关键的就是让整个任务在后台线程上运行。我们不想让同步工作在 UI 线程中进行。UI的责任是当他们登录，显示加载旋转和通知用户。
2. 任务应该是可以发出信号，通知是否登录成功或失败。
3. 我们知道这比较复杂，因为这里有多个任务并且我们得按照一定的顺序来执行它们。
4. 一些任务将是异步的，比如数据库同步。但这一步应该不是独立于其他的事情，一个任务接着一个任务。
5. 最后，我们写的代码应该能够让人感到易读。

### 创建一个登录任务 [(16:45)](javascript:presentz.changeChapter(0,10,true);)

建立一个登录任务需要很多步骤，经常令人比较头疼的事要弄清楚要有哪些任务需要运行并且他们顺序是怎么样的。可能有些要同步进行有些要异步进行。我特别列出了以下步骤，这些过程是异步的：

- 获得网络请求的身份验证令牌（auth token）
- 通过网络请求获取用户账户
- 获取远程数据库与新的本地数据库
- 整合新数据库

这四个任务需要在流程中进行拟合。我们也有一些其他的任务，可以同步运行。其中设计的第一原则就是不能阻塞的用户界面线程。所以，对任务进行分解能够让我们更好地去开发好的应用。我还考虑了对于某些任务，要如何给 `Activity`进行反馈。在一个过程中任何一个任务失败，用户都将无法继续启动。最后，我们完成了所有的步骤，并且可以显示登录页面了。

整个任务的清单可以归结为两个主要的原则。首先，试着找出 可以把潜在的复杂的任务分解成更小的任务 的方法。第二，尝试在这些方法中向 `Activity` 发送失败或者成功的反馈。

### 建立登录状态 [(19:03)](javascript:presentz.changeChapter(0,13,true);)

这就是我所说的回调链的例子，如果你可以同时运行两套操作，是因为他们彼此相对独立。但是在一个有多个异步任务的情况下，并且这些异步任务有联系和先后顺序，事情就变得不一样了。比如在我们的情况下，你必须得从服务器上获得一个登录 token，才能拿着这个 token 进行后续操作。如果失败了，那么我们得立即退出。但是，如果它成功，那么我们可以继续进行下一个异步任务。因此，你最终会拥有这一系列的工作，所有的工作都需要以异步运行。

让我们来看一看，我认为可以解决这个问题的一个方法。我试着将这整个事情看作为一个可以从 `Service` 开始的东西，但会以一个模块的形式运行。我有一个类，有许多方法，当我们滚动代码页面，我们只是看到更多的方法，这些方法各自做的任务不一样。这是一个粗糙的方法，这个类的代码会比较多。我写这个类的存在的问题是，它真的，真的很难记住怎么才能让所有的零碎代码一起工作。我什么时候调用某个方法？为什么我要调用这个方法？

如果我是从以前的开发人员手里接手到这份代码，我就很难知道这个类应该做什么。这里有一个登录管理器，但我不知道什么时候登录是真正成功的。原因是，在回调这些异步过程，还有另一个方法调用其他方法。

### 消息处理程序 [(21:58)](javascript:presentz.changeChapter(0,14,true);)

一定程度上来说，以下这个代码是有组织的。它运作得不错，但并不是一个伟大的解决方案。我想谈一谈我们能回到一个更好的解决方案。幸运的是，我们不需要写一个类，我们有其他选择。早些时候，我们看到了一个 handler 被用于协助 `AsyncTask` 。事实证明，使用 handler 同样适用于这种情况。我们需要做的一个程序执行这项工作，再细分成碎片。我们先创建一个 `handleMessage`，它可以处理各种任务。使用它的时候，当某部分代码执行完成时，它需要一个在类中的 handler 发送一个消息告诉它运行下一段代码：

```java
// MyTaskModule.java
private static class LoginHandler extends Handler {
		MyCallback callback;

		@Override
		public void handleMessage(Message msg) {
				switch (msg.what) {
					case QUIT:
						callback.onFailure();
						break;
					case STARTED:
						doStuff();
						send(obtainMessage(NEXT_STEP));
						break;
					case NEXT_STEP:
						callback.onSuccess();
						break;
				}
			}
		}
```

开始时，我们给这个 handler 发送一个开始执行的消息。运行该代码后，它会发送一个 `NEXT_STEP` 消息，使得代码进入到下一步对应的 case 当中。另外，我们还给出了退出的方式，通过发送一个消息称为 `QUIT` 来进行退出。这将使我们能够脱离 handler，并调用一些代码，关闭整个事情。

为了向你展示如何工作的，我举例我的一个叫做 `MangoLoginHandler` 的例子。这实际上是 handler 的一个实例或子类。我们所有的工作都是在`handleMessage` 里面做。如果我们想要的话，我们的代码可以被分割成更多的方法。在我底下的代码，我有一个模块与一个公共的方法，让我们能够从 `Service` 运行代码发送 `LOOPER_STARTED` 消息。这是我们要获得数据必要做的一步。当我们得到一个 handler 的引用的时候，这个 handler 是可以处于任何我们调用它的线程。因为我打算在 `IntentService` 中来启动，这时候我就不是在主线程。所有这一切都封装运行在后台线程中。

就像一个巨大的代码墙，我认为它仍然可以更好地分割成单独的方法。从阅读我的 handler 的标签，我觉得我有一个更好的代码。

### Activity 到 Service 到 Module [(25:57)](javascript:presentz.changeChapter(0,15,true);)

还有一个点，我还没有说呢。我们可以在一个 `Activity` 和 `Service` 之间进行交流。我们知道，我们可以创建一个任务模块，把一个 handler 放到里面，并从 `Service` 启动模块。但是，现在缺少的部分是能够从 handler 回到了 `Service` 和 `Activity` 的一部分。

你所能做的就是在任务模块中定义一个非常简单的接口。这个接口可以是仅限内部访问的，也可以公开。然后，你可以在启动该 handler 的 `Service` 或启动任务的模块来实现这个回调接口。最后你需要响应 `onSuccess` 和 `onFailure` 这两个方法。这些方法意味着任务模块正运行在该服务上。

```java
// MyTaskModule
public interface MyCallback() {
	public void onSuccess();
	public void onFailure();
}

// rest of task implementation
public void start(MyCallback callback) {
	// call onSuccess or
	// onFailure here!
}
```

在你的 `Activity` 中，你启动了你的 `Service` 并且通过 bundle 传递一个 `ResultReceiver`。 在 `Service` 中，你首先实现了回调接口，所以  `Service` 是该接口的一个实例。当你启动你的任务模块，它会获得这个 `Service` 作为回调。在模块中，handler 是贯穿所有的步骤，如果一切顺利完成，它就会调用接口对象的 `onSuccess` 方法。在那时，它会通过 `ResultReceiver` 回调到 `Activity` 中。所以你可以通过这样发送一个 OK 结果到 `Activity` 中。可以停止加载进度圈，并且算是登录进入程序啦。

## 总结 [(28:10)](javascript:presentz.changeChapter(0,16,true);)

我们已经创造了四个对象。你有一个 `Activity`，管理控制用户看到的内容。你有一个 `Service`，照顾所有那些你的任务。它有一条与 `Activity` 的交流路线（那个 `ResultReceiver`）。你还有一个单独的模块，是将从  `Service` 中分离出的，以使其能够重用。在任务模块中，您可以有一个 handler，能够通过全部任务过程，并进行通信，您需要知道所有回到  `Activity` 的方式，以便用户可以响应。

## 进一步阅读 [(28:55)](javascript:presentz.changeChapter(0,17,true);)

当我在研究这个问题时，我发现这些资源真的很有用。 它们是 [CodePath guides](https://guides.codepath.com/android). [*Efficient Android Threading*](http://shop.oreilly.com/product/0636920029397.do) 是一本我读起来很享受的书. 最后还有, Android 官方文档 [processes and threads](http://developer.android.com/guide/components/processes-and-threads.html), 以及 [multiple threads](https://developer.android.com/training/multiple-threads/index.html) 这篇文章也很有帮助.

## 问与答 [(29:30)](javascript:presentz.changeChapter(0,18,true);)

**问: 我可以使用 Android 的帐户管理器（Account Manager）来编写类似的代码吗？**

Ari: 我很高兴你提出这个问题，因为这是从我所展示的代码中不能清楚的。我们实际上也在使用谷歌的帐户管理器。在设备上注册用户帐户，就像一个谷歌帐户，实际上是这个代码所做的许多任务之一。这实际上不是用来替换帐户管理器的使用，但是您可以将帐户数据写入到帐户管理系统中，这是整个过程的一部分。

**问: 我看到了一个谷歌的演示，建议使用一个同步适配器（Sync Adapter）来进行事情排队和处理反馈。这一点你们有考虑过吗？**

Ari: 这是合理的。我认为在使用同步适配器时，你必须建立一个适合于你需要做的其他任务的实现。如果是这样的情况，你不需要执行其他的任务，那么就没有什么好的理由来使用网络库来做同步适配器的工作了。如果调用同步适配器能做你需要的，像同步谷歌帐户，取决于你自己是否直接调用或包装（wrapper）。对于这样的东西，我可能会给它一个非常简单的包装。同步谷歌帐户数据从本地到远程，那肯定是选择最合适的东西。

Stephan: 关于同步适配器（Sync Adapter）有大量的样板代码。你必须创建自己的内容提供者。但是，它应该为你节省一些工作。这是很好的，但它还不够好。

**问: 你能说得更详细一点关于使用信号返回到UI线程吗？特别是，你是否需要锁？你的回调如何结束更新 activity？**

Ari: 我没有用任何锁实现这个。基本上，我不想把它写成这样一种方式：通过 `ResultReceiver` 修改对象会出问题。在我的特殊用途的情况下，我避免锁，我做了两件事情。首先，我停止了进度圈，这实际上不是线程安全的修改。但是，因为执行是传递给用户界面线程的，所以这还是不错的。我会在 `Activity` 中要求一个用户界面线程的方法来改变这个。我正在做的下一件事是完成登录活动，这在用户界面线程上是安全的。我在那个方法中不做太多的事情，避免了这个问题。如果你真的有需要同步更新很多东西，那么我会关注它，然后让我的需求尽可能小。

**问: 你有想过通过一个 EventBus 来分离 Activity 和 Service 吗？**

Ari: 老实说，我没有，不过我也很愿意将来能谈论这一步。

**问: 我们如何能直观地理解，为什么 Android 通过 Service、Sync Adapter、AsyncTask 还有 handler 来设计程序？**

Stephan: 其实是关乎电池的问题，是因为在手机上没有风扇来散热。如果你有很多东西，你把它放在你的口袋里，它会开始燃烧你。而电脑上，内存是如此便宜。你可以做任何你想做的，因为你有很多的内存、一个冷却风扇，和大量的空间。在安卓，你没有那个。很多它归结为电池的问题。

Suyash: 我只是想增加一些有趣的东西。当我第一次进入安卓系统时，你首先尝试在主线程中做所有的事情。如果你做过 JavaScript 编程，你应该不需要使用多线程，直到进入 Android 开发。你的大脑开始学习如何处理新的线程。使用 Android 系统的 `Services`，`AsyncTask`，甚至 handler。所以，这基本上是因为性能的原因，因为一切不应该发生在用户界面线程。我认为这是天才的设计，因为我没有看到它在 iOS。`AsyncTask`很独特。

Ari: 我的经验是，有一个非常直观和手动学习过程中，当你遇到性能问题。我很同情，这要学习大量的信息。Stephan 指的是电池的使用和性能，但我想补充说明。作为一个程序员，如果解决方案是在不同的线程上运行任务，就很容易创建新的线程。然后，你会遇到锁定问题和线程安全问题。如果你忘记管理自己这些过程的生命周期，那么他们仍然运行在 Android 中。所以，当你开始理解这些概念时，很难，但它实际上是最好的，去学习它们然后提升自己吧。

**问: 用户可能需要等待登录，所以你处理的是什么方式，让一个进程运行，同时也响应用户？**

Ari: 在“芒果”应用程序中，我们通过返回按钮来解决这个问题，这是工具栏顶部的后箭头。一个对话的消息会问用户，是否他们可以停留一分钟的时间。另一种选择是让这个直接响应，但要给你的 `Activity` 一个间隙状态。我们可以在这样一种方式，用户将看到，他们已经能够返回到应用程序。他们可以看到以前的屏幕，但他们看不到完整的应用程序。然后，我们必须给他们一些反馈，该过程是整理和等待另一分钟。这样，如果他们按了回到 home 页面，登录过程可以继续工作。

这是非常重要的，因为我们仍然希望在登录过程中工作。使用 Service 的好处就是它能够继续工作而不容易被杀死。但因为这个潜力，我们也必须小心，我们可能要在用户界面中做一些事情，而它们可能已经不见了。你可以微调你的页面让它们若隐可见，这样它们就不会被回收，这样你可以在你的 `ResultReceiver` 正常做某些事了。我觉得这样做有点枯燥，我想真的想不出一个更好的解决方案了。