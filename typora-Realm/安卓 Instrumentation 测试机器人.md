# 安卓 Instrumentation 测试机器人

> 原文：[安卓 Instrumentation 测试机器人](https://academy.realm.io/cn/posts/kau-jake-wharton-testing-robots/)
>
> 作者：[Jake Wharton](https://twitter.com/jakewharton)

[TOC]

类似 Espresso 的库可以使 UI 测试能够和应用有着稳定的交互了，但开发人员往往随心所欲，结果却会使得这些测试复杂得难以管理，更加难以维护。在这篇演讲里，Jake 会告诉大家那些所谓的机器人模式，在 Kotlin 语言特性的帮助下，是如何让你创建稳定的，可读的和可维护的测试的。

------

## 绿色盖伊先生 [(0:00)](javascript:presentz.changeChapter(0,0,true);)

这篇演讲的内容运用得很广泛；不论你是一个安卓开发者，或者你是一个 Java UI 的桌面开发者，网页开发者，服务器 APIs 的开发者 – 和测试相关的人都能从这个模式中受益匪浅。

安卓会是我们这次示范的例子，但是它其实可以用在任何地方。这个模式就叫 *机器人模式*。

在我开始介绍机器人模式之前，我想快速谈谈我们的测试工作总体上是怎么进行的。这就是，如果你有一个绿色的 QA 组，你的绿色 QA 组只使用计算机。它们和计算机交互，不管测试的对象是一个 web app，桌面 app，或者一台手机。

当你有这个 QA 小组以后，它们就会为你执行你的服务，应用的测试，他们只和一个东西打交道。那就是 *视图*。这是你的应用表述自己的方法；他们执行高层任务来验证它们工作。

然而，在这些视图背后，还有其他架构方面的东西。我选择的架构是 model view presenter。选择这些架构的背后都是有原因的，但是那些绿色盖伊先生们并不关心，因为他们只和 view 打交道。

## 行为架构 [(2:43)](javascript:presentz.changeChapter(0,5,true);)

我们选择这些架构的原因是为了节省我们的工作量。也许是我们的后台改变了。也许是我们的数据库改变了 – 非常典型的 – 从 SQLite 变成 Realm。下层的 model 将会被替换，而且我们只需要更新 presenter，同时我们的视图不需要发生变化。

或者应用的需求发生了变化，你需要更换视图。现在，你只需要更新部分 presenter，而且你的 model 不需要关心 view。我们对这些干系者们有着非常好的隔离。

这给我们的测试带来了另外的便利。我们能够在 presenter 后面粘上一个假的 model，而且在它的前面还添加一个单元测试，现在你可以测试 presenter 里面的内部业务逻辑了。

我们也可以反过来，我们给 view 添加一个单元测试，然后我们可以验证，不论 presenter 何时调用 view 的某个方法，view 的反应和期望一样。这个架构可以给我们提供这些便利。*没有一个架构*，不论是测试还是替换应用里这些神奇盒子的部件，你都不可能实现这个级别的颗粒度和灵活性。

但是我需要重申一次，我们的绿色盖伊先生们，那些事实上执行着你的应用的验收测试或者功能测试的人们，只关心 view。

## 测试期望 [(3:38)](javascript:presentz.changeChapter(0,9,true);)

本次演讲中，我们将使用一个我精心设计的应用，这代表着 [Square Cash](https://cash.me/)，这是我工作过的一个产品。应用的测试像某种验收测试。比如，一个高层级的功能测试会是：有 42 美元，把它发送给特定的 email 地址，然后验证转账是否成功。

这是我们想实现的顶层功能。这也是我们希望测试验证的地方。但是这里也有 *如何测试* 的问题。*如何测试* 的问题呢，由我们的朋友-绿色盖伊先生们来解释执行。

他会在数字盘上输入数字 42，点击收件箱，然后在弹出的键盘上输入 email 地址，点击 send 按钮，然后等到屏幕切换，他会看到一个 checkbox。

但是这个 *如何测试*，这个实现方式是我们的绿色盖伊先生实时操作的。真的，他想确定的目标就是 *我们想测试的东西*。他想确定我们把钱成功地转出了。

## 传统的测试 [(4:42)](javascript:presentz.changeChapter(0,12,true);)

我们都是程序员，我们都能把这些事情自动化。这也是我们在程序里面做的事情。我将编写一个自动的测试工具来完成这个工作，而不是每次发布的时候都需要那个绿色盖伊先生帮助我们执行这些无趣的任务，这样我们也不需要给他付工资了。

你可能会写出如下的测试：

```
PayScreen pay = (PaymentScreen) obtainScreen();

pay.amountView.setValue(42_00);
pay.recipientView.setText("foo@bar.com");
pay.sendView.click();

Thread.sleep(1000);

assertThat(obtainScreen()).isInstanceOf(SuccessScreen.class);

```

你说，给我一个显示在屏幕上的视图。抓取它上面的组件。填入一些数据。找到屏幕的按钮。点击它。等一会儿，因为这是异步的。然后给我那个可以确定成功的屏幕，屏幕显示是我们成功付款的验证点。

你这样做的问题是，你把需要验证的事情推到了视图上，而且这两件事情还紧密地耦合在了一起。

如果我们想更换些东西，比如我们的视图因为一个重要的风格改变了，我们需要修改我们的测试了。你不可能丢掉它，但是你需要重构它。你的测试现在就变成了关心测试是怎么完成的了，而不是最终的功能是怎么验证的了。

这样一来，无论何时你改变你的应用，你都需要重新解释你测试的一些部分来保证另一边的行为是一样的。

但是，如果我们回到我们的绿色盖伊先生，在替代他的工作上，你做的很不好。你现在紧密地耦合了代码。这样，不可避免的，业务逻辑，技术，所有这些东西如果改变了，你就不得不改变你的测试。而且如果你替换你的绿色盖伊先生的视图，他不关心，他每次都能重新解释执行，然后运行它。他关心的是他要测试的目标，而不是如何测试。

我们该做些什么才能用我们的测试来代替他的工作呢？

## 改进的测试 [(7:15)](javascript:presentz.changeChapter(0,16,true);)

一个实现的方法是从外部进入，像一个鼠标或者一个手指，而不是深入应用的代码来了解它们的 views。

```
findViewWithText("4").click();
findViewWithText("2").click();
findViewWithHint("Recipient").setText("foo@bar.com");
findViewWithText("Send").click();

Thread.sleep(1000);

findViewWithText("Success!");

```

“我将找到这个 view 然后点击它。” 最终，你把这些事情打包在了一起，实现了同样的测试。同样 sleeping。我们找到我们的 view，然后验证它。这是同样的方法。在安卓世界里，[Espresso](https://developer.android.com/training/testing/ui-testing/espresso-testing.html) 帮助你这样做：

```
onView(withText("4")).perform(click());
onView(withText("4")).perform(click());
onView(withHint("Recipient")).perform(typeText("foo@bar.com"));
onView(withText("4")).perform(click());

onView(withText("Success!")).check(visible());

```

它通过视图层次结构来和你的应用交互。但是，不是通过你自己写的类，它工作起来，你的应用就好像一组不透明的视图。你指引它找到特定的视图，然后执行一些操作，好像是用户正在点击屏幕或者在键盘上输入一样。

当这个方法出现的时候，每个人都欣喜若狂。这真的简化了他们的测试。如果你注意到，`Thread.sleep()` 消失了，Espresso 为我们处理了异步操作的等待问题。

到目前为止，你可能都还感到开心。但是我要说我们一直期望着测试我们真正要测的 *功能*。然而这个方法依旧是 *如何测试*；你依旧是在打包所有测试的执行步骤并运行它。如果你的视图变化得特别厉害，你将需要遍历每一个测试然后更新测试需找的视图，还有这些事情的执行顺序也需要更新，这将是件非常乏味的事情。这不是我们想要的，因为我们这些程序员 – 是非常懒惰的；我们需要找到最有效的方法来最小化我们的工作量。

我们采用 view-model-presenter 架构的原因就是我们想隔离 *测试对象* 和 *测试方法*。model 就是测试对象 – 网络、数据库、系统和任何地方来的原始数据。我们的 presenter 负责把 molds 解析成 *如何* 在视图上显示的形式。

另一边，我们已经有了使用 Espresso 的测试用例，或者你给任何平台写的任何测试，我们把两个概念混淆在一起了 – 测试对象和测试方法。这是个根本的问题，也是需要我们解决的问题。

所以，我提出机器人模式，我将给你们展示这个拥有隔离概念的神奇方法。最终，Kotlin 的语言特性使得变化下的这些测试容易表达，非常整洁而且灵活性十足。

## 机器人：隔离开测试对象和测试方法 [(9:50)](javascript:presentz.changeChapter(0,19,true);)

机器人是什么？最终，这是个设计模式和一个开放的解释。我会给你们展示两个例子，来演示我运用这个模式的方法。重申一次，虽然例子是基于安卓的，但是你真的可以把这个模式运用到任何平台上，只要你记住隔离和思考下实现的方法就可以了。

机器人是一个我们打算在一个视图或者别的东西上进行的高层级动作的类。这很像是一个通过接入点展示功能的服务器。你和这些接入点的交互会是非常高层级的和描述性的。

然后你和这些接入点交互的细节问题，它使用的序列化格式和它要求的参数。这些都是怎么实现的细节。测试方法是我们封装在机器人里面的细节。

## 简单的 PaymentRobot [(10:40)](javascript:presentz.changeChapter(0,20,true);)

关于我们的美丽小应用，我们在付款界面上能做一些事情。这个类的名字叫做 `PaymentRobot`：

```
class PaymentRobot {
    PaymentRobot amount(long amount) { ... }
    PaymentRobot recipient(String recipient) { .. }
    ResultRobot send() { ... }
}

class ResultRobot { 
    ResultRobot isSuccess() { ... }
}

```

我们可以输入数量。但是我们不知道输入的是多少。我们所知道的事情就是如果我们要在屏幕上输入一些数值的话，它需要一些值。

我们可以输入一个接收者。我们不知道接收者是不是在屏幕上，我们只知道屏幕有一个允许我们输入接收者的地方。

然后我们可以做一些操作。我们可以在屏幕上做些操作，比如点击发送按钮。在我们应用里面，当你点击发送按钮的时候，应用的上下文会发生变化，我们会看到一个屏幕显示操作是成功还是失败。这样我们的发送操作可以返回一个不同的机器人，这个机器人知道该怎么和接下来的屏幕交互。

我们可以验证这个结果屏幕是不是显示成功。最终，如果付款失败，一些假设断言会发生而且会中断你的测试。

那些各种各样的细节，我们可以把它们推到这里，这些都是怎么实现的细节。它们是什么不重要，它们是你的应用，你的框架，你的平台特有的逻辑，和你怎么测试无关。机器人的实现就是 *测试方法*。我们采纳了和你的应用交互的高层级的意图，我们把它们抽象出来，然后在这些机器人后面封装那些和你的应用具体相关的细节。

现在，我们就有这些甜蜜的，轻量级的 API 可以用了：

```
PaymentRobot payment = new PaymentRobot();

ResultRobot result = payment
    .amount(42_00)
    .recipient("foo@bar.com")
    .send();
    
result.isSuccess();

```

仅仅说，”嘿，当我启动应用的时候，我知道我在付款界面，给我付款机器人吧。” 我们将发送 $42 付款给我们的 Foo Bar 朋友。这返回给了我下一个界面的机器人，然后我将判定是否成功。

这里，我们通过我们的 builder 封装了 *测试对象* 到我们测试中，而不是 *测试方法*。这个测试描述了我们原始的问题是什么，一个我们的 QA 同事能够看懂的脚本。

脚本没有说明要按下哪个按钮。它只说，开一张 $42 付款给 Foo Bar。点击发送。验证成功与否。这些现在都是 *测试对象* 了。实现一大堆 *测试方法* 的是机器人。

你的测试是描述性的，简洁的而且你的机器人有一点疯狂。但是至少这个数据需要封装一次。机器人只需要写一次，测试可能会写很多次。

## 一个机器人，一百个测试用例 [(14:15)](javascript:presentz.changeChapter(0,25,true);)

这是一个成功的用例。这个用例用来看看应用是否工作。整个过程如下：

```
@Test public void singleFundingSourceSuccess {
    PaymentRobot payment = new PaymentRobot();
    
    ResultRobot result = payment
        .amount(42_00)
        .recipient("foo@bar.com")
        .send();
        
    result.isSuccess();
}

```

我们也需要测试当一个人想发送百万美元的时候，我们不能让你转账成功。

```
@Test public void singleFundingSourceTooMuch {
    PaymentRobot payment = new PaymentRobot();
    
    ResultRobot result = payment
        .amount(1_000_000)
        .recipient("foo@bar.com")
        .send();
        
    result.isFailed();
}

```

另外一个例子是测试你的账户余额不足的情况。也许你只剩下 150 块，而你打算发送 1000 块，这不会成功的。我们需要验证，

```
@Test public void singleFundingSourceInsufficientFunds {
    PaymentRobot payment = new PaymentRobot();
    
    ResultRobot result = payment
        .amount(1_000_00)
        .recipient("foo@bar.com")
        .send();
        
    result.isFailed();
}

```

这本质上是边界测试用例的组合爆炸 – 这些测试用例。而且这样的测试用例很多。我们希望这些用例能够代表 *测试对象*– 被测试的对象。这就是应用的业务逻辑，而不是我们如何实现这些业务逻辑的过程，也不是支付过程中的流程。

所以我们只需要编写一次机器人。它就拥有了所有的交互逻辑，然后我们就有了这些测试用例，这些用例很多，因为你可以设计许多测试场景。它们非常整洁，易于阅读，而且非常抽象。

当这些东西变化的时候，如果只是视图改变了，比如布局，或者你输入数字的方式，那么只有你的机器人需要改变。但是如果改变的是业务逻辑功能，整个应用的架构 – 你的应用的价值主张 – 如果你重新排列的视图，那么只有你的测试用例会发生变化，而不是机器人。

希望你开始看到了这样做的优势，这就是为什么一开始我谈及架构的原因。你可以看到这些架构是有益处的。如果我们从这些架构产生的源头获得启发，而且在我们的测试领域运用它们，我们的测试不仅仅变得更易阅读，而且最终会变得更加稳健，更加灵活，更加适应不可避免的测试变化。

在我们的架构里，我们对每一个屏幕都有一个机器人， 而我们的测试用例的数量惊人。我们已经拥有了良好的解耦和重点。

## Kotlin 机器人 [(16:30)](javascript:presentz.changeChapter(0,31,true);)

我们的测试很不错。它非常漂亮，易读和整洁。但是 Kotlin 可以帮助我们做的更好。所以，如果我们想把这些机器人变成 Kotlin 机器人，我们该怎么做呢？

我们想试着使用 Kotlin 提供的语言特性，同时想保持它的安全性。我们能做些简单的事情，Command+Shift+Alt+K 开始直接导入 Kotlin。但是这不能给我们带来任何好处，我们能走的更远一点。

一般而言，如果你看到一个编译器，脑海里立马能想到的事情就是你如何调教 Kotlin 的编译器。让我们开始这样做。

首先，让我们用更高级的语法特性来代替传统编译器。让我们摆脱编译器的返回值 – 黑掉它。在我们处理对象产生之前。我们就创建了这个机器人和调用了它所有的方法。我们将转向工厂方法模式。

```
fun payment(func: PaymentRobot.() -> Unit) = PaymentRobot.apply { func() }

class PaymentRobot {
    fun amount(amount: Long) { ... }
    fun recipient(recipient: String) { .. }
    fun send(): ResultRobot { ... }
}

class ResultRobot { 
    fun isSuccess() { ... }
}

```

通过在 apply 作用块里调用 `func`，这个函数返回机器人自己，而不是 void。这使得我们能够把方法非常优雅的连接在一起。

现在，这会给我们的调用代码带来什么呢？

```
val result = payment {
    amount(4200)
    recipient("foo@bar.com")
}.send()

result.isSuccessful()

```

好吧，我们不需要调用构造函数，我们能直接调用静态方法。Kotlin 使得我们能在没有括号和其他辅助的情况下，传递一个混合块，所以我们能像上面那样做。这些机器人里面的高层级方法，我们现在能在不做任何修饰的前提下调用它，因为它是函数的扩展。

在上面的代码块中，我们好像是在机器人类内部操作一样。我们能好像友元函数一样调用它们。当我们调用它们的时候，它们会调用到机器人合适的函数上。接收方也一样。接下来，我会在 payment 方法上的返回值上调用 `send()`。

这就是为什么我们做这些小技巧的原因。我们仍然希望返回的是机器人，所以当我们用这个代码块调用 payment 的时候，我们能够得到我们原始的机器人。然后我们调用 `send` 方法，它返回给我们可以交互的下一个机器人，因为这时会有屏幕的转移而且我们想移到下个新的机器人那儿。

我们实际上做的事情就是创造了一个漂亮的视觉上的函数链，这样我们就不需要不断的引用不同的机器人里面的本地变量了。我们基本上采用了和 payment 一样的模式。让一段代码块实现机器人。我们将它运用到返回一个新的机器人的 `send()` 方法里。

```
fun send(func ResultRobot.() -> Unit): ResultRobot {
    // ...
    return ResultRobot().apply{ func() }
}

```

我们基本上使用了我们的扩展函数，然后运用到机器人上，否则我们就需要返回它。我们改变了 send 方法的行为。现在它不再返回一个值，而是接受了一个 lambda，这样我们可以把我们的转账成功的检查放到 lambda 里面，与此同时，我们不再需要本地变量了。

```
payment {
    amount(4200)
    recipient("foo@bar.com")
}.send() {
    isSuccessful()
}

```

我们调用 payment。我们在这个转向另一个机器人的 payment 机器人上可以做任何事情。我们调用 send，它也会返回给我们一个机器人。我们直接调用 send 方法然后我们给他传入我们在下一屏上想做的所有事情。

如果我们愿意，我们能把函数链一直这样延续下去。但是我们想做的更好一点。这里有一个五个字符的疯狂语言特征叫做 `infix`，它能使得我们在发送函数之前推送，这本质上可以转换成为一个二进制运算符了。这样，这就变成了接收两个值得函数，而且返回另一个值。

我们将使用这个功能来实现一个小改进：去掉括号和发送方法之间的点。

```
payment {
    amount(4200)
    recipient("foo@bar.com")
} send() {
    isSuccessful()
}

```

现在我们有了这个漂亮的整洁的而且非常易读的代码块来描述 *测试对象* – 而不是 *测试方法*。这个测试实际运行时候的视图和测试本身毫无关联。这个测试只关心需要测试的业务逻辑，而不是你的应用实现是怎么执行测试的。

## 一般的机器人策略 [(23:37)](javascript:presentz.changeChapter(0,45,true);)

机器人模式开始于进入点，也就是能看到的第一屏，然后你有在那一屏上的任何动作或者断言。我们推荐不要太多的断言，我们想把这些断言推到单元测试中。

通过遍历整个应用，我们隐式地断言了许多事情，而且保证了应用如我们期望一样反应，而且我们希望交互的东西都在屏幕上。`isSuccess()` 就好似一个显示断言 – 我们断定屏幕上会显示一些东西。

然后我们的 `send()` 方法基本上是屏幕切换点，也是机器人切换点。我们的屏幕变化了，我们需要另外一个机器人，我们青睐的简洁函数给我们返回了这个机器人。

## 描述性的堆栈跟踪 [(24:30)](javascript:presentz.changeChapter(0,46,true);)

另外一个很酷的事情是 – 除了非常易读以外，我们能得到非常漂亮的堆栈跟踪信息：

```
Exception in thread "main" java.Lang.AssertionError:
    Expected <Success!> but found <Failure!>
    
    at ResultRobot.isSuccess(ResultRobot.kt:18)
    at PaymentTest.singleFundingSourceSuccess.2.invoke(PaymentTest.kt:27)
    at PaymentRobot.send(PaymentRobot.kt:13)
    at PaymentTest.singleFundingSourceSuccess(PaymentTest.kt:8)


```

你不需要点击测试来找到到底是哪一行出了错，然后弄清楚发生了什么。这个堆栈信息本质上就是非常易读了。我们知道我们在 `singleFundingSourceSuccess()` 测试里面。我们的 payment 机器人调用了 `send()`。所以这就是点击 payment 屏幕上的发送按钮。

然后我们知道机器人的结果是断言成功。所以你得到了堆栈信息，这最小化了我们深入了解测试失败的步骤。你甚至不需要去看测试的代码，除非你需要知道输入的数字是多少。你的堆栈信息复制了你得到失败屏幕的步骤，这是个很有帮助的副作用。

## 一个可替代的机器人 [(25:25)](javascript:presentz.changeChapter(0,47,true);)

我想谈及的另外一个事情是关于 `infix` 的所有困难，通过这样你才有了漂亮的链式表达。在测试的逻辑只有少数分支的时候，这样是工作的。但是如果你有一个应用，它的每一个屏幕都能够在不同的状态和输入的情况下，迁移到另外六个屏幕上去的时候，每一个迁移都使用一个单独的机器人就会有一点可怕了。

这是这个模式的另外一种实现方法。你基本上可以移除你的测试中显示屏幕迁移，把它们变得更加隐式。让我们假设当你发送付款的时候，你打算发送很多的钱，我们想验证你的身份来确定你不是一个黑客。

```
payment {
    amount(4200)
    recipient("foo@bar.com")
    send()
} 
birthday {
    date(1970,1,1)
    next()
}
ssn {
    value("123-45-6789")
    next()
}
result {
    isSuccessful()
}

```

现在我们会询问你的生日和你的社会保险号。我们基本上是隐式地实现它，因为这些事情是接连发生在屏幕上的，同时也需要在代码里面顺序执行，而不是在每个单独的机器人里面嵌入这些屏幕转移的代码。

你仍然能获得迁移保证的方法是：你基本上把这些函数按照出现的顺序变成了断言：

```
fun payment(func PaymentRobot.() -> Unit) {
    onView(withText("0")).check(visible())
    return PaymentRobot().apply { func() }
}

```

如果你在支付界面上做些操作，在我给你这个机器人和真正运行之前，我会验证一些支付界面上面东西是否存在。

我们会验证一些测试期望的先决条件，来保证和应用交互的正确性。在这个例子里面，我们会需找屏幕的标签，做点小小的验证。有时候这也许是不必要的。

## 机器人模式 [(27:55)](javascript:presentz.changeChapter(0,51,true);)

最后，我想强调一下我们应该把这个模式看成一个更宽泛的模式。用你的方式解析它，并且最大化它的价值。没有一个特定的库。这是一个纯粹的简单的模式。

我们都同意一个应用内的好架构会使得代码具有长期可维护性。然而，我们从来不讨论另外一边，你的测试架构。通常它都是亡羊补牢，你常常在后面灭火。

然后，如果你花点时间，也不需要那么多，想想你的测试架构，在你的测试里面保持和应用一样的解耦和隔离，你的测试就会有更高的质量了，更具有维护性了，长远看，这会节省你写代码的工作量。