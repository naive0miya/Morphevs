# RxJava 介绍

> 原文：[RxJava 介绍](https://academy.realm.io/cn/posts/360andev-christina-lee-intro-rxjava-java-android/)
>
> 作者：[Christina Lee](https://twitter.com/RunChristinaRun)

[TOC]

RxJava 是一个了不起的 Android 工具，它使得代码的编写和维护更加容易，而且更少出错，但是作为一个新的安卓开发者，却不容易理解 RxJava 到底做了些什么事情使得编写代码更加容易了。在这篇 [360|AnDev](https://360andev.com/) 演讲中，我们会讨论传统的安卓开发方法，这些方法容易出错的地方和 RxJava 是如何解决这些问题的。除了提及 Rxjava 的初衷，这篇演讲也提供了一些常见的 RxJava 的使用实例，和一些第一次使用 RxJava 的常见陷阱。这篇演讲不需要 Rx 基础。

------

## 介绍 [(0:00)](javascript:presentz.changeChapter(0,0,true);)

我的名字叫 Christina，我来向大家介绍 RX。在我们深入介绍之前，先做个警告，我觉得许多介绍的演讲都流于形式，是这样，这很简单。使用这么简单的一两行代码，你的世界就变容易很多。我真的很想做这样的一个演讲。我也想这样说。

但是不幸的是，RX 可不是这么简单。RX 是复杂的，有趣的和有意思的。在许多方面都有意义，可不是简单。所以，请容忍我，这里有许多内容。我将尽我所能使得大家容易理解，但是可能不是那么容易。它也不是一行代码，我希望这不会太有关系。

## 背景 [(0:50)](javascript:presentz.changeChapter(0,2,true);)

在我深入谈论 RX 之前，让我们看看目前的情况，因为如果目前的情况没有问题的话，你也不需要解决方案。今天有两件事情让我觉得有趣，为什么我们需要异步操作？安卓对异步操作的支持是怎么样的？

第一个问题很容易回答。因为用户是我们的朋友，而且我们关心它，我希望他们能有好的体验。如果你使用异步操作的话，你可以利用服务器的能力，那些因为移动设备的限制而不能实现的，需要真实的后台进程才能处理的事情，就可以实现了。你可以给你的用户提供美妙的体验，而不会拖慢他们的速度，冻屏或者跳屏都不会出现了。

关于第二个问题，你有许多的工具可以使用。这里没有办法给你一个全集。我们没有办法讨论所有的安卓中异步相关的东西，但是我们会涉及其中流行的部分。我可以肯定的是，你们之前一定听过 Async Task，而且很可能使用过它，当然还有 Futures、Event busses和 Observables，这是我们今天要讨论的内容。这当然不全面，所以我提供了一个更多信息的博客，以便你在演讲后详细地阅读它。它讨论了 Swift 和安卓的所有事情，基本上是所有的事情。

我想强调的是当我说 Futures 的时候，我会使用规范的形式来谈及 Futures，我不是指的内嵌的 Futures。我指的是 `ListenableFuture`。 对于内嵌版本的 Futures，有着许多错误的地方会迷惑你。这里，我将使用 Futures 来代表 `ListenableFuture`，当然它的底层是 Futures，但是添加了许多有用的函数。

我们有许多工具，而且虽然我们有许多的工具，可最后只会使用一到两个，这样我们就需要评估这些工具。我不会站在这说我有最佳评估矩阵的衡量标准，但是我会提出我的观点。我考虑的三个方面是：__何时运行，怎么运行和影响谁。__当我从这三个方面考虑问题的时候，它就会告诉我那些工具是合适的。

如果我们把 Async 作为例子，你可以注意到 AsyncTask 需要被显式执行。这很棒，因为这意味着我们能够控制它什么时候开始。这个工作在 AsyncTask 里面就不容易实现了，所以关于如何运行，是你激活了它，如果你想构建什么东西的话，你可以在一些回调函数里面实现它们。这不理想，但是把它们聚合在一起不容易。关于输出，许多数据是一个 “副作用”。

如果你不熟悉 *副作用*，首先，这可是个很棒的感念。我建议你们去了解了解，其次，在医学上，它有着必然的因果关系。如果我吃感冒药的话，目标就是治愈我的感冒。如果它让我变绿了，就是副作用。这不是目标，我不想变绿。

同样的例子，如果你想做些事情，然后它就开始从网络中读取数据，或者它改变状态，而且这都是不确定的，这就是副作用，因为你不能通过你的函数来知道到底会发生什么。

```java
private class SomeTask extends
		AsyncTask<ParamsType, ProgressType, ResultType> {

	//some implementation here
	protected ResultType doInBackground(ParamsType params)

	protected void onPostExecute(ResultType result)
}

new SomeTask().execute(params)
```

如果你观察这段代码，你能看到最后一行是我们显示执行的。你可以看到一个 `doInBackground` 函数，这里我们有些实现，然后当它执行完毕的时候，`onPostExecute` 会执行，但是你不确定你会如何从这里开始执行事情，或者是一旦你开始了，它们是如何改变事情的。

如果你有评估准则，你会有一些理想情况的假设和目标，否则的话，评估准则的意义是什么呢？我打算提供一些理想的情况，这是我们的目标。

首先，显式执行。如果你能开始一些事情的话，你就能控制它。对于 Future 来说，这不总是正确的，当你创建它们的时候，它们就运行了。你希望尽可能的控制它们。在这个例子里面，它能帮你，因为你能创建它，在它运行之前控制它。

你还希望简化线程管理。本次演讲是关于异步行为的。线程管理是关键，如果你想简单的话。你希望它们是能被转化的，而且副作用越少越好。

我将会深入谈到每一点，但是基础是一样的。显示执行意味着你能控制这些可观察者。你能创建它们，存储它们，修改它们，然后在它们完美的时候，在它们得到它们需要的数据的时候，它们和你期望的一样的时候，你可以马上执行它，而不是过一会。强调一遍，关于线程管理，这是异步操作。如果你不能容易地切换线程，或者理解你在哪个线程，这就会很糟，因为你会因此而出错。

我们也想容易点，因为它易读，这对写代码的时候很有用。关于简单组合，异步工作能独立完成就好了。当我们执行的时候，网络操作不依赖任何东西，我们不需要在网络返回时做任何事情，我们做了，然后忘记它，这样很棒。但是现实不是这样，而且事情常常是交互的，因为它们是互相依赖的，所以它们希望是协同工作的。如果你使用缓存，然后网络，你不希望它们之间还有中间步骤。你希望它们是串起来且具有意义的。如果你的异步库是可组合的，它就不容易出错，这是件好事情，这样我们就胜利了。

最后，最小化副作用。副作用越小，你的代码就越经得起质疑，你的代码就越易读，其他人也容易阅读你的代码，以为它不依赖于状态，它是确定的，而且解释起来也很容易。

## Rx [(6:46)](javascript:presentz.changeChapter(0,17,true);)

我设置了这些标准。你可能知道事情该如何发展了。我个人觉得 RX 能满足这些条件，让我来告诉你原因。RX 是显式执行的。你如果想让一些事情发生的话，你需要注册。在那之前，你都是在构建或者创造或者存储，事情没有被启动。

使用 RX 非常容易理解你是在哪个线程上执行的，因为那里有单行操作符 `subscribeOn` 和 `observeOn`，这里你可以配置线程。如果你看看代码，你可以看到 `observeOn` IO 线程，这是发生在 IO 上的。它们很容易改变，过滤，对应和其他各种各样的操作。这是它真正闪光的地方。消除副作用，这也很棒。这是异步的工作，所以它们从来都没有摆脱他们。你在使用网络，你在改变定义的状态，但是它做了非常好的尝试来减少它们的影响。

关于 RX 的快速的事实。首先，RX 是为异步事件创建的。也是专门为聚合构建的。其次， RX 是 **Reactive Extensions**。再次，RX 支持许多平台。我今天的演讲是关于安卓__RxJava__，你也可以在 JavaScript 里使用它。也可以在 Swift 里面使用。你能在不同的语言里面使用它。

RX 的三个关键点是现在在屏幕上的三点。RX 有可观察者，有 LINQ 查询，有调度器。LINQ 查询是基于这些概念的，虽然它不是显式使用的。在可观察者、查询和调度者之间，我们有了 RX 的世界。

深入理解下，可观察者代表着异步数据流。这对于刚刚接触 RX 的人来说常常非常难以理解。所以我会晚点深入谈谈。但是现在，就先假设它们是流。查询允许你能在这些流上面执行操作，调度，而且不出意外地，你能控制这些流的一致性。

回顾一下：这些是 RX 的三个核心部分。我们能表示异步数据流，我们能查询，我们能用操作合并这些流，而且我们能用调度来协同这些流。

## Observable [(9:12)](javascript:presentz.changeChapter(0,25,true);)

回到我们的第一部分：Observable。Observable，如我所说，是数据流。它们是拉数据的方式，需要注意的是你可以采用 RX 里面的高级功能来实现推送数据模式，但是一般情况下，它都是拉数据的。你能够创建它，存储它，然后传递它们，等等。因为它们是显式执行的。最后一点是我特别想强调。

Observable 能帮助我们隐藏所有线程和同步的复杂细节。这是说复杂性不存在了，只是库帮你解决了这些问题，而且我们希望它们处理的很好，因为它们是全职在做这些事情。RX 常常非常困难，就是因为这些流和 Observable 的概念。

我昨天晚上和一些人聊天。我问他们如何理解流，这对他们来说意味着什么，有人说 *类比工厂* 是它真正有意义的地方。我把这个观点抛出来。希望这能帮助到一些人。如果不行，就把他们当作是流，希望这样就能理解了。

类比工厂意味着你开始有一些原材料。比如你在生产汽车，你的工厂来了钢铁。这是原材料，这就是我们的网络请求。我们在某处开始，然后一旦你有了原材料，你把它们送给传送带。你可能会把它压成金属板，或者压成汽车的形状，或者做些类似的事情。你可以也需要引入其他工厂的一些资源。也许你需要一些橡胶来做轮胎。当你在这些传送带上执行的时候，你构建它，增加东西，最终当汽车出现在门外的时候，你有了最终的产品，在我们的例子里面，就是从流里面获取了数据。

所以我们放进一些东西，一些原材料，我们用不同的方法构建它，然后我们得到我们想要的产品。

两个关键的可观察者的生存周期阶段，我这么叫它，就是你想往里放数据和你想往外获取数据的时候。你还有其他想要流的原因吗？就是数据。看看第一部分，放入数据。你有许多不同的方法能够实现，我会展示一些。最简单的就是 `observable.just`。

```java
Observable.just("Hello World!")
```

这是一个串，我只想要这个，所以我放了进去。

```java
val names: Array<String> =
	arrayOf("Christina", "Nicole", "Alison")

//Will output Christina --> Nicole --> Alison --> X
Observable.from(names)
```

第二部分，你能在 Observable 上使用迭代器。这里我有一个我的一些朋友名字的数组：Christina， Nicole，和 Alison。我可以说 `Observable.from` 这个可迭代对象，然后从这个可观察者中返回的就是 Christina， Nicole， Alison，然后就结束了。

```java
String[] names = {"Christina", "Nicole", "Alison"

//Will output: -> Christina -> Nicole -> Alison -> x
Observable.from(names)
```

这个创建过程最重要的部分就是使用 `.create`。

```java
Observable.create<String> { s ->
	s.onNext("I created an observable!")
	s.onCompleted()
}
```

这里你可以发现发生了些什么，我使用注册 `S`，我特别地管理了设置的东西。在这个例子里面，我提到了注册，在 `onNext` 里发送我在 Observable 里面的字符串。我不想在这里放置更多的字符串，所以我让它告诉你完成了。然后我说 *注册，你结束了*。这是一个 Observable。任何事情都没有发生，但是当我创建一个的时候，这个的行为是，它发出字符串，然后结束。

```java
Observable.create(new OnSubscribe<String>() {
	@Override
	public void call(Subscriber<? super String> subscriber)
		subscriber.onNext("I created an Observable!"
		subscriber.onCompleted();
	}
})
```

你可能注意到这和 Java 不一样。这是使用__Kotlin__完成的代码。我将使用 Kotlin，这样我能保持代码的一致性，但是我将尽可能地解释它。

还有一个 `onComplete`，这有一点疯狂，因为我们实现了所有的这些 `onNext`，我们使用的是数组，然后突然一下，就有了 `onComplete`。*结束* 意味着什么？这甚至不是放入数据，那是拿出数据，你是对的。这就是一个从可观察者里拿出数据的机会，如果你不理解在一个流的结束的位置，你是如何接收数据的，那么你也不会理解是如何放入数据的。

我们从流里还需要什么？主要是数据。我们想有一个方法说 *嘿，下部分数据是什么？* 我使用了网络，我能得到什么？我们也想知道，还有剩下的数据吗？因为如果没有了，我们就结束了，我将会继续，我将会回收资源。我们真的想知道：我还需要关心这个流吗？或者我能继续吗？

最后，因为现实世界不是完美的 (我们存在的痛苦)，有错误发生吗，我需要知道吗？代码里面看起来是怎样的？我想要下一个数值的时候，有 `onNext`。我知道当有事情结束的时候，有 `onComplete`，而且我知道当有事情出错的时候，有 `onError`。

希望这些名字是有意义的。它们是可读的。这是 RX 让我我最喜欢的原因之一。这里没有什么令人吃惊的地方。下一个值，`onNext`。结束 `onComplete`。错误，`onError`。

```kotlin
Observable.create<String> { s ->
	val expensiveThing = doExpensiveComputationHere()
	s.onNext(expensiveThing)

	val otherExpensiveThing = doOtherExpensiveComputation()
	s.onNext(otherExpensiveThing)

	//all done
	s.onComplete()
}
```

这些在实际工作中会是如何？重申一次，我们有顶层的函数 `create`，所以我们说 `observable.create`。在这个例子里面，我用 `string` 类型，所以这是一个可观察的对象，而且生产一堆不同的字符串。你可以，你知道，在这里放上一个不同的类型。

而且，读这些 Kotlin 代码就像在读伪代码，你有一些注册，然后我们有一个开销很大的函数。也许它会从磁盘读取内容或者其它事情。让我继续，然后在有数值返回的时候再处理。一旦我有数值了，让我通过调用 `onNext` 把它放到那个流里面。我们调用这个函数，得到数值，然后我们说，`s.onNext` 那个值。也许我们还没完成，也许现在我想访问网络，或者我想做另外一个磁盘读取，或者谁知道要发生什么。这里有些其他开销严重的函数。我会调用它，然后我会获取值，我能使用 `onNext` 把它推送到流上。

也许我只有那两个开销大的事情，也许我有更多的。在这个例子里面，我们在这两个调用后就结束了，所以我会告诉流，结束了，而且我会说 `onComplete`。再详细点解释每个步骤，你可以如你所愿多点或少点调用 `onNext`。现在，我的屏幕上有一个星号，因为它不是一个完整的真实的语句。而且在那种情况下，我下面会谈到一些性能问题，如果你太频繁的这样使用的话，你可能会遇到这些问题。作为一个 RX 的介绍，你可能不会遇到它，但是这是要注意的地方，如果你真的有繁重的处理的话，你可能会遇到性能问题，而且你想小心地面对它，然后正确地处理它。

一个关于 `onNext` 的小测验。我真的想确保每个人都理解了。有人能告诉我这个流看起来是什么样子吗？这不是一个困难的问题。它真的是和你想的一样简单。两个和三个，准确地说，就是这！它放入了两个，放入三个，然后结束了！类似的，输出也一样。这开起来像 *苹果，香蕉，橘子，菠萝*，然后结束了。回家后尝试着这样做，你也能做到的。你可以实现一些二进制数。它将会看起来一样。你会一个个地输出，然后流就会关闭。我感觉关于 `onNext` 有许多的迷思。它真的就是这么简单。

谈到 `onError`。这真的是关键了，这是我将会详细讲到的一点。如果你需要了解一件事的话，我希望就是这件，因为它会省去你许多的麻烦。`onError` 和 `onComplete` 都结束流。它们都是流的结束事件。唯一的不同就是当你调用 `onError`，流正在结束因为有些事情出错了；当你调用 `onComplete`，流正在结束因为你结束了你的工作，这时不需要做任何事情。但是两者都会结束流，不同的地方就是流结束的原因。

一个 RX 的 `onError` 伟大的地方是所有的错误都在一个地方处理。你可以绑定一组不同的可观察者，出现的任何错误都会出现在你的 `onError` 里面。如果它发生在上游，如果它发生在下游，它都会来到你的 `onError` 里面。这和 Futures 里面的东西不同，Futures 有独立的成功和失败的回调，然后也许你需要告诉另外一个 Future 这个 Futures 的错误，那么你该如何传播这个错误呢，然后巴拉，巴拉，巴拉。这个错误会传播回到最底层，所有的错误都在那里。如果你想找错误，它就在那里。看这里，如果你想要错误的话。这是它的要点。

```kotlin
Observable.create<String> { s ->
	try {
		val result = functionThatMightError()
		s.onNext(result)
	} catch (e: Error) {
		s.onError(e)
	}
	s.onCompleted()
}
```

回到终止事件的地方，这里有个例子，我想逐步地通读一遍来看看发生了什么。我有一个 Observable，我们自己创建的。我们在这里放了许多的 try-catch，因为这里有个函数会出错。我不知道它做了些什么，也许是访问网络，但是它会出错。所以，在这个例子中，我们假设他工作正常，我们调用它，然后我们调用 `onNext`，传入参数，然后我们执行 `onComplete`。但是这不是函数 *errors* 的情况。

```kotlin
Observable.create<String> { s ->
	try {
		val result = functionThatMightError()
		s.onNext(result)
	} catch (e: Error) {
		s.onError(e)
	}
	s.onCompleted() //<--NOPE
}
```

这和前面的 Observable 创建一样，但是这次让我们假设那个函数出错 *一定出错*。你可能希望它会马上调用 `onNext`，然后 `onError`，然后 `onComplete`。但是记住，`onError` 是流的终止事件，所以你会得到 `onError`。这是我们不期望的，所以停一停，让我们想想。在 `onError` 之后，流结束了，没有任何地方可以调用 `onComplete`，因为那是流成功的结束。我们已经是不成功的结束了。`onComplete` 和你想的一样。它告诉你流结束了。继续执行而且为你自己清理现场吧，回收你持有的任何东西然后继续。

回顾：我们有我们的 `onNext`，我们有我们的 `onComplete`，我们有 `onError`。每一个函数都完成你希望它们完成的功能：传播下个数据片，告诉你流已经结束了，或者告诉你流的一部分有一个错误，就是这样。

如果你记得我在 LINQ 查询讨论的东西的话，关键点是：Observable 和许多异步方法的显著不同是，它们在代码中的组合的时候和各种转换的时候都是非常有用的。

## 操作符 [(21:40)](javascript:presentz.changeChapter(0,62,true);)

现在我们来谈谈操作符。我们有许多的操作符。我不可能向你们介绍所有的操作符。你有过滤的操作符，采用一些元素然后跳过一些元素的操作符，映射的操作符，水平映射的操作符，你能想到的任何东西，都有操作符的存在。它们在实战中看起来如何？

```java
Observable.from([1,2,3,4]).map { num ->
	num + 1
}

Output: 2 --> 3 --> 4 --> 5
```

如果你看看这个，数组的 `observable.from` 和我们在之前的演讲里面看到的一样。如果你也有这样的 Observable，它可以发射数字 1,2,3,4 然后它会停止。

```java
Integer[] numbers = {1, 2, 3, 4};
Observable<Integer> obs = Observable.from(numbers).map(
	new Func1<Integer, Integer>() {
		@Override
		public Integer call(Integer integer) {
			return integer + 1;
		}
	}
);

//Outputs: 2 --> 3 --> 4 --> 5 --> x
```

这里调用它的 `onComplete`。虽然这里我们增加了额外的异步，就是 *map*。这个 map 接收一个变量，所以函数有一个给定的参数叫做 number。我们想做的事情就是我们想说的事情，返回数字加一。一进来，我们有一了，我们说，但是我们不想要一了，我们想要一加一，所以二被输出了。然后二进入流，我们有了二加一，所以是三。然后突然地，一个 Observable 可以输出 1,2,3,4，现在输出2,3,4,5。

```java
Observable.from([1,2,3,4]).filter { num ->
	num % 2 == 0
}

Output: 2 --> 4
```

我们也可以用 `filter`. 这里，我们有一个数据，然后返回一个布尔表达式，如果数字模二为零的话返回真。如果不是的话，就不给流返回。这里模二是一，所以不能继续。二模二是零，所以可以继续。这样我们有输出 1,2,3,4；然后通过过滤，变成 2,4。

```java
Integer[] numbers = {1, 2, 3, 4};
Observable<Integer> obs = Observable.from(numbers).filter(
	new Func1<Integer, Boolean>() {
		@Override
		public Boolean call(Integer integer) {
			return integer % 2 == 0;
		}
	}
);

//Output: 2 --> 4 --> x
```

这只是一个 “给我事件” 的数字函数。我们也能做些事情，比如组合和合并 Observable。

```kotlin
val schoolFriendsObs =
	Observable.from(arrayOf("Mo", "Dave"))
val workFriendsObs =
	Observable.from(arrayOf("Nicole", "Alison"))

val allFriendsObs = Observable.merge(
	schoolFriendsObs,
	workFriendsObs
)
```

我没有那么多朋友。他们都在这里了。你可以看到第一个数组是我学校的朋友。我有 Mo 和 Dave。太棒了。然后我有些工作的朋友。我有 Nicole 和 Alison。也有一个关于他们的 Observable。然后在底部，我组合了我所有的朋友。让我从合并过后的学校和工作的朋友中创建一个 Observable。这个输出看起来像这样。

```shell
//Note: Output is only in the same
//order because input was synchronous
I/RxKotlinHelper: onNext(Mo)
I/RxKotlinHelper: onNext(Dave)
I/RxKotlinHelper: onNext(Nicole)
I/RxKotlinHelper: onNext(Alison)
I/RxKotlinHelper: onComplete()
```

我们有 Mo，我们有 Dave，我们有 Nicile， 我们有 Alision，然后结束了。

## Scheduler [(24:42)](javascript:presentz.changeChapter(0,71,true);)

所有 Observable 的元素都在这个输出里面了，然后我们结束了，因为没有剩下任何东西了。现在，我给你们留下一个笔记。这里的数据是顺序的，是因为它们是字符串的名字，所以它们是 *同步的*。如果你在处理网络，那么顺序可能会是 Nicole, Mo, Dave, 然后 Alison。它们不一定像原来的顺序。

继续最后一部分，Scheduler。这也是有意思的地方，因为我们谈到异步工作。如果你不注册，什么都不会方式。重申一次，你能看到星号是因为我骗了你。这里有个警告。高级性能问题。你现在不需要理解它，但是当你真正需要考虑这个问题的时候，是有许多值得思考的地方的。

但在我们的上下文中，在 Java 结束的上下文中，如果你不注册，什么都不会发生。我之前提到 `observable.create` 的地方，然后我放置了输出，那不是完全正确的。那是这些可观察者能够输出的内容，但是它们没有真正地输出，因为我从来都没有说过在它们身上 *注册*。我创建了一个对象，然后如果你在它上面注册的话，它会输出那些东西。

关于注册者的一些东西。它们能接受不同数量的函数。它也可以不接受函数，如果你只想启动它，而不关心返回的是什么的话。它们能接受三个函数，`onNext`， `onComplete`，和 `onError`。最大三个，最少零个。

我将停在这里，然后强烈建议你总是传递 *最少* 一个 `onError`，因为如果你不这样做，你的堆栈将完全不可读，你会恨你自己的。请，最少，传入 `onError`。这很重要，如果有什么事情出错了，你能理解哪里出了问题。

然后，关于工作该在哪里执行，我们有两个不同的操作符，它们是 `subscribeOn` 和 `observeOn`，我将使用这两个操作符。请知晓，这是我们管理合适开始的方式。

代码看起来像什么样子呢？

```kotlin
val names = arrayOf("Christina", "Nicole", "Alison"

Observable.from(names).subscribe(
	{ next ->
		Log.i(TAG, "onNext($next)")
	}
)
```

我有一个同样名字的数组。调用 `observable.from` 这个数组，然后最后一行是新的代码。我说 `.subscribe`，然后在这个例子里面，我只传入了一个函数。我传入了 `onNext` 函数，然后那行代码说，下一个我能得到的数据片，打印它。就是这样。

输出看起来像这样：

```shell
I/RxKotlinHelper: onNext(Christina)
I/RxKotlinHelper: onNext(Nicole)
I/RxKotlinHelper: onNext(Alison)
```

它打印了名字，然后什么都没有发生。现在，如果我们想更严格，我们可以传入三个函数。

```kotlin
val names = arrayOf("Christina", "Nicole", "Alison"

Observable.from(names)
	.subscribe(
		{ next ->
			Log.i(TAG, "onNext($next)")
		},
		{ error ->
			Log.i(TAG, "onError($error)")
		},
		{
			Log.i(TAG, "onCompleted()")
```

这是个完整的函数集合。我们传入 `onNext`，这是第一个函数。我们传入 `onError`，打印错误，然后最后一个函数是 `onComplete`，我们将打印 onComplete。这次输出看起来很熟悉。

```shell
I/RxKotlinHelper: onNext(Christina)
I/RxKotlinHelper: onNext(Nicole)
I/RxKotlinHelper: onNext(Alison)
I/RxKotlinHelper: onCompleted()
```

虽然我们增加了一行，就是 `onNext`，所以和以前一样，但是现在我们增加了 `onComplete`，所以当结束事件发生的时候，我们有一个输出。这就是最底下的 `onCompleted`。

我目前为止忽略的是 `.subscribe` 会返回一个注册。你可能不需要用到这个注册，但是它应该是有一个注册的引用的，这不奇怪。你可以在它上面调用 `unsubscribe`，然后就会有这样的东西：

```kotlin
val sub: Subscription = SomeObs.subscribe{ next ->
	Log.i(TAG, "onNext($next)")
}
sub.unsubscribe()
```

我在一些 Observable 上面调用 `subscribe`，这是我上一个幻灯片上面创建的东西，然后我打印了 `onNext` 里面发生的事情。它会再一次打印名字或者其它东西，然后如果你不关心它是怎么工作的话，我也不愿意监听它了，我能够调用 `unsubscribe`，因为我有这个注册的引用。你不需要引用，但是在你自己不需要的时候释放你曾经 `subscribeOn` 的东西，是个很好的习惯。

这就是 Observable 和所有注册过程的内容。那么线程呢？我的工作在哪里运行呢？

回忆一下，我们有这两个操作符，我们能使用它们来告诉 Observable 它们应该从哪里开始工作。`subscribeOn` 只应该使用一次。你可以不止一次地加它，但是不会起作用。`subscribeOn` 会在你创建可观察者的最近的位置起作用。如果你在主线程的第一行调用 `subscribeOn`，那么 10 行代码后，你调用了 IO 线程，它会在主线程上注册。它会忽略接下来发生的事情。你只能使用一次，就这。如果你不显式说明它应该从何注册的话，它就会发生在线程上。而且默认的是你创建可观察者的线程。这常常是主线程，但是如果你在计算线程或者其它线程上创建可观察者的话，你需要把 `subscribeOn` 放到你确定能运行的地方，因为它默认会在计算线程，这是它运行的地方。

关于 `subscribeOn` 的关键点是你代码的第一部分总是在你的 `subscribeOn` 上执行的。在你注册的那点上，代码会开始在那个线程里执行。也许这行代码之下，你在做其他的事情，然后你先有一个 `observeOn` 来改变线程，这没问题。但是 `subscribeOn` 告诉你你的代码从哪里开始执行。第一块运行的代码总会执行 `subscribeOn` 直到你有机会改变它，如果你这样选择的话。

这会把我们导向 `observeOn`。也许你已经在主线程上注册了，也许它是一个计算线程，但是现在你想显示些东西，放在 UI 上，所以你想切换回主线程。这是 `observeOn` 能帮上忙的地方。你可以无限次地这样使用。

这里有个可怕的警告。这会有些压力的问题。重申一次，如果你不是真的在处理性能要求严格的事情，你可能不会这样做，但是如果你是，这样做可以。

理解 `observeOn` 影响接下来的所有事情很重要。如果我在一个 10 行 Observable 链条上的第 2 行放上一个 Observable 的话，它会从第 3 行开始影响起。除非我在第 6 行再放一个 Observable，或者 `observeOn` 在第 6 行，在这个例子中，Observable 现在会作用于剩下的所有事情了。每一个你的 `observeOn` 下面的东西都会在你告诉 `observeOn` 的任何线程里发生。我们看看代码。

## 注册，监听 [(30:33)](javascript:presentz.changeChapter(0,89,true);)

我将会逐行解释。

```kotlin
Observable.from(arrayOf("Red", "Orange", "Blue"))
	.doOnNext { color ->
		Log.i(TAG, "Color $color pushed through
				on ${Thread.currentThread()}")
	}.observeOn(Schedulers.io()).map { color ->
		color.length
	}.subscribe { length ->
		Log.i(TAG, "Length $length being recieved
			on ${Thread.currentThread()}")
	}
```

我们开始于一个 Observable。你已经看过千百万次了。他将会输出一组颜色。就是这样。现在我将要增加一个 `doOnNext`。你还没有看到它，但是 `doOnNext` 是可以自解释的。当 `onNext` 发生的时候，我将执行我提供的这个函数。在这个例子中, 我提供的函数就是打印日志。我想把它在执行的线程打印出来。这就是这里发生的内容。在 `onNext` 发生的时候打印线程。

现在，也许其他的事情会发生了，我不想再在这个线程里了，所以我放置了一个 `observeOn`，然后我说，让我们切换到 IO 线程吧。我会做些事情比如映射它。我有一个颜色的字符串。也许我只关心字符串的长度。这很重要。我会说：接收这个串，然后给我这个串的长度而不是串本身。然后你就完成所有的产品了。我增加了 `subscribe` 然后它打印出我最后得到我的输出的线程。输出看起来是怎么样的？

```shell
I/Rx: Color Red pushed through on Thread[main,5,main]
I/Rx: Color Orange pushed through on Thread[main,
I/Rx: Color Blue pushed through on Thread[main,5

I/Rx: Length 3 being
	recieved on Thread[RxIoScheduler-2,5,main]
I/Rx: Length 6 being
	recieved on Thread[RxIoScheduler-2,5,main]
I/Rx: Length 4 being
	recieved on Thread[RxIoScheduler-2,5,main]
```

它从主线程开始，因为我没有给它一个 `subscribeOn`，我在我的例子应用的主线程里面编写了这个 Observable。在我切换到 IO 线程的时候，不出意外的，它开始在 IO 线程上输出内容。你可以看到最下面的三行代码，它说我们在 IO 调度器上。现在，让我来使用一个显示的 `subscribeOn`。

```kotlin
Observable.from(arrayOf("Red", "Orange", "Blue"))
	.doOnNext { color ->
		Log.i(TAG, "Color $color pushed through
			on ${Thread.currentThread()}")
	}.observeOn(Schedulers.io())
	.map { color -> color.length }
	.subscribeOn(Schedulers.computation())
	.subscribe { length ->
		Log.i(TAG, "Length $length being recieved
			on ${Thread.currentThread()}")
}
```

我不想在主线程上运行这个事情。这还是那个我们一行行建立起来的 Observable，但是有一个 `subscribeOn` 调用，我把它放到计算线程中去。它会怎么改变事情的运行呢？

```shell
//Output trimmed to fit

I/Rx: Red pushed on Thread[RxComputationScheduler]
I/Rx: Length 3 recieved on Thread[RxIoScheduler]

I/Rx: Orange pushed on Thread[RxComputationScheduler]
I/Rx: Length 6 recieved on Thread[RxIoScheduler]

I/Rx: Blue pushed on Thread[RxComputationScheduler]
I/Rx: Length 4 recieved on Thread[RxIoScheduler]
```

现在你可以看到第一部分的工作做完了，它们是在计算调度器上做完的。在那之后，一旦我们切换到 IO 线程上，我们把它们映射到链接上，这会在 IO 调度器上完成。这是一个你能看到的例子，我们能在一个地方开始工作，我们能切换到不同的调度器上，而且它能在其他的地方完成。我还能增加一个不同的 `observeOn` 来把它放回主线程，然后你就能在输出里面看到交互运行了。

## Rx 和 Android [(32:46)](javascript:presentz.changeChapter(0,100,true);)

RX 和 Android。我还没有开始讨论过 Android。我们讲到了可观察者和一般意义上的 RxJava。让我们谈谈这怎么影响安卓开发者的生活。重申一次，这和运算符类似。我不可能告诉你们所有的 RxJava 在你们应用里面的应用方式，因为篇幅是有限的。你是不是应该使用它又是另外一个话题，但是你可���在不同的地方使用它。

这里有些有点麻烦的例子。你可以绑定点击，然后过滤某种模式，或者你通过你的可观察者的聚合器来判断人们是不是在双击。你可以 FlatMap 网络缓存命中。你可以在一个单一流里面处理鉴权的问题，所以你可以得到用户输入然后你发送给网络，然后你得到返回值。也许第二次需要网络，获取用户详细信息。所有这些事情都发生在一个单一流里。

我将串讲一些例子。在我这样做之前，我想说，它们只是帮助你们看看在安卓应用里面是如何实现的。如果你不能理解它们，没关系。这里只是给你们一些感觉，看看什么是可能的。

```kotlin
fun Button.debounce(length: Long, unit: TimeUnit) {
	setEnabled(false)

Observable.timer(length, unit)
	.observeOn(AndroidSchedulers.mainThread())
	.subscribe {
		setEnabled(true)
	}
}
```

这里有个去抖按钮的例子。我不确定你为什么要这样做，但是你可以这样做！这里发生的事情是，我给一个按钮类增加了一个叫做 `debounce` 的东西。我可以给它一些时间单位，然后我开始 `setEnabled` 为 `false`。我会设置一个时钟观察者。它会超时。当它超时的时候我设置 `setEnabled` 为 `true`。

在 `setEnabled` 为 false 的时间里面，用户不能点击这个按钮。然后在我的时钟超时后，`onNext` 被调用， `Enabled(false)`变为 true，然后用户可以再次点击了。

```kotlin
fun size(): Observable<Int> = Database.with(ctx)
	.load(DBType.photoDetails)
	.orderByTs(Database.SORT_ORDER.DESC)
	.limit(24)
	.map { it.size() }
```

我在我的照片分享应用里面使用过这个，我们在数据库里面有许多照片。我们使用的一个场景是 - 这是个少见的例子 - 我会使用 context 来得到我的数据库，然后我说：嘿，加载些照片的详细信息吧。我真的只关心最近的照片。你能帮我按照时间顺序排序吗？然后，顺便提一句，我不能一次处理许多照片，所以给我最后 24 张就好了。这是我真的关心的东西。然后也许我只关心照片的大小。我想在 UI 上做些东西，谁知道呢。然后最后一行说，好吧，不是照片的细节，给我照片的大小。映射成为大小。

这看起来很紧凑。如果没有 RX，这会是 5 页代码。它会非常冗长。我不得不在代码间跳来跳去。你能在 6 行代码里完成这件事情是件非常了不起的事情。这是真的很复杂而且非常容易阅读。

```kotlin
return Observable.create<Unit> { s ->
	val outputFile = writeOutputFile(mediaFile)

	when (type) {
		Type.Photo -> addPicToGallery(ctx, outputFile)
		Type.Video -> addVideoToGallery(ctx, outputFile)
		else -> {
			s.onError(Error("Unexpected download type!"))
	}
}
s.onNext(Unit)
s.onCompleted()
```

你可以对 IO 做这样的规范示例。你可能下载一些东西。你有一些函数来写文件。我知道那儿发生了什么，但是我得到了一个文件。也许我们需要检查类型。如果是图片，就放到另外的地方。如果是视频，放到另一个地方。如果都不是，那发生什么呢？抛出一个错误。然后，在这个例子里面，我们不会通过流输出数据，因为我们不关心。

我们常常使用这个可观察者的地方，就是 toast 说 “下载完成” 或者 “下载失败” 我们不需要从那个流里发送文件，因为没有用。我们关心的是，这个过程完成了吗，或者失败了？

你会发现我们有 `onError`。如果有错误，如果我们不知道文件类型，我们不知道怎么处理它，而且如果不知道，我们就只能调用 `onNext` 而没有任何参数。`Unit` 这里是 null 类型，所以它说调用 `onNext`，但是我不会给你的 `onNext` 函数任何参数。它将会退出。然后，我们说，我们结束了。封装它。你可以看到有一个 `subscribeOn` IO，所以你可以放置一个 `subscribeOn`。

```kotlin
fun codeObservable(): Observable<String?> {
	val filter = IntentFilter(SmsUtility.INTENT_VERIF_CODE)
	return ContentObservable.fromLocalBroadcast(this
		.map { intent ->
			intent.getStringExtra(SmsUtility.KEY_VERIF_CODE)
	}
}
```

还有些方法可以从本地广播里面提取信息。所有那些给你验证码的应用，它们给你短信验证码，然后你需要把它们输入到你的应用中去，这是你认证的方式。安卓很伟大，是因为如果你读他们的短信的话，你会吓到用户。这是个真实发生的例子。

有人给你发送一个验证码。你自动读取短消息，抽出验证码然后把它放到 `LocalBroadcast` 中。但是 `LocalBroadcast` 会有点笨拙，所以让它继续，然后把它变成一个可观察者。这里发生的是，我们有 `IntentFilter` 来获取验证码，而且我们把它传给 `ContentObservable.fromLocalBroadcast`。

我们不需要 intent。这不是我们感兴趣的地方。我们感兴趣的就是验证码，所以我们肯定这是个正确的。最后一行说：你给我了一个 intent，但是我真正关心的就是验证码。把它抽取出来，然后你可以看到返回类型是 `String?`。这意味着 `String` 会存在而且也可能是 null。如果那个关键值不在我的 intent 里面，它会返回一个 null。如果在，就会返回验证码，然后我可以做些进一步的验证。

```kotlin
timedAuthObservable
	.observeOn(Schedulers.io())
	.flatMap { code ->
		userModel.sendVerifyResponse(code)
	}.flatMap {
		userModel.getSuggUsername().onErrorReturn {
	}.observeOn(AndroidSchedulers.mainThread())
	.subscribe({ suggestedUsername -->
		//update UI with suggested username
})
```

你可以构建些真的很复杂的首次流程。这是伪代码，但是 `timedAuthObservable` 是这样的情况，你进入输入密码的环节，看看它们是不是有效，所以你看到进度圈。但是也许你的网络很差，所以它会出现连接失败，你感觉很讨厌。这是一个`timedAuthObservable`。

你可以在 IO 线程实现它，因为你需要网络。然后也许你接下来想做些事情。什么事情不重要。我增加了一些伪代码，正如你想验证你的返回值一样，现在你登录了，你想获取一个建议的用户名，这样你就能使用它们建议的名字完成首次登录。

与这里发生的事情相比， *如何* 发生更重要。这是你在 8 行代码里完成的所有鉴权的过程，而且你能很好地管理这些代码在哪里运行，仅仅通过 `observeOn` 和 `subscribeOn`。

```kotlin
//Verify with backend, then prepare data for UI
override fun getVerifiedData(code: String):
Observable<Unit>

	return UserService.noAuthClient
		.verifyUser(authToken, code)
		.flatMap {
			UserService.authClient.fetchUserDetails()
		}.map{ data ->
			loadableUserState.loadFromData(data)
		}.observeOn(AndroidSchedulers.mainThread())
}
```

你也可以完成这样的事情。如果你不鉴权，启动一个 IO 事件来认证，而且一旦你认证了，你可以调用一个函数，现在你已经认证了，然后获得了认证通过的回复。没有可观察者，实现这些不太容易，因为你有些东西需要经过验证，然后你需要存储它们，而且你需要另外启动一个任务来验证这些结果。

现在你开始没有验证已经不重要了，因为你会开始使用 FlatMap 用户验证客户端服务，你被认证了，因为这是流工作的方式。你从流的顶部到底部，所以在你的函数被调用的时候，你就已经被认证了。或者抛出了一个错误，你会知道抛出的是那个错误的。这很强大，而且看起来也没有很多代码，但是你可以从一个没有登录的用户开始，你能在你的 IO 之后开始你的登陆后的相关流。

## 结论 [(39:56)](javascript:presentz.changeChapter(0,108,true);)

让我们总结一下。我不是来告诉你们， RX 是唯一的解决问题的方式，而且总是正确的方法。在这个演讲中，你们能发现有许多不平常的问题，而且容易引起混淆。这不是我想说的。我想说的是 RX 是目前最强大的工具。当然它可能不适用于所有的情况，但是如果你是在做高性能的东西，如果你做的东西和网络相关，或者你从磁盘经常读数据，或者你正在做许多异步工作，它的确是个正确的工具。

它的开销不小，但是带来的好处也不少，它能做许多事情。这里有一个学习曲线。我知道。这很难受。这好像是在玻璃上行走，正如我们昨晚讨论的一样。这不舒服。但是一旦你掌握了它，它会帮你减少错误。

一旦你学会这样思考，代码就会变得更加易读因为它们被封装在了一个逻辑的时间链条中。你可以看到有些东西访问网络，被转换，然后放到 UI 线程来被你的 UI 使用。这是个链条。这个链条代表了整个动作而且你不需要引入额外的机会产生错误，比如引入状态存储。

最后，我觉得流会带来巨大的范式转换。我们不这样思考，是因为这不是眼下大多数应用构建的方式，但是如果你能把数据想象成为流，你整个应用的数据流就会浮出水面。紧接着，当你使用其他的异步原语的时候，发生的事情就是你给一个线程发送一些东西，然后这个线程就在那里了，然后你的 activity 做了一些事情，你就可以在不同的地方获得异步数据了。

如果你使用的线程强制联系在一起，然后你能轻松地写一个 GUI，会发生什么。这些东西原来就有，当你启动一个网络事件，你可以看到它贯穿你的整个应用，你所有的问题就是你的 UI 被限制住了，或者放到别处等晚点使用。第一个网络调用，你带入其他数据的转换，或者过滤数据，或者减少数据，所有的事情能够在 GUI 里面出现。这很强大，因为它给了你的应用一个方式，这个方式每个人都能理解。

如果你有一个路线图，你知道你在什么地方，那就真的很有用了，不仅仅是开发者，对产品的人也有用。告诉它们数据流是怎么工作的，哪里获得数据，它们如何改变，在哪里结束。

最后，这是我可以给 RX 的最好的总结。如果你有一辆汽车，你有一个老旧的汽车，你将要做的事情就是去商店，这也不错。马路的限速是多少，25 MPH？你可以这么说。也许座位不是很舒服，但是你可以调整它。也许不能发动了，你可以扭扭钥匙，这都没问题。它能工作，就很好，就很棒。

然而，如果你对旅行还有好奇，或者你想从高速公路出去，我保证如果你有一个 Tesla 和一辆老汽车，你会意识到两者的不同，因为马力不一样，安全性也不一样。它学习了过去 20 年的驾驶经验，所以 Tesla， 有一个更好的安全记录，并且会世界闻名。这是因为它们看到了过去发生的事情，它们改正了这些错误，然后在新的技术上重新构建。

如果你想开着 Tesla 去商店，你可能不会注意到一个不同，因为 25 MPH 不是这个汽车建造的目标。你不需要加速从 0 到 60 去杂货铺。这可能不是你最好的选择。但是如果你开车穿过全国，如果你想做更大的事情，更复杂的东西，更持久的东西，更能纪念的东西，Tesla 是你想要的。它适合这种场合。它会让你的生活更愉悦，更安全。而且坦率地说，有更多的乐趣。

谢谢！