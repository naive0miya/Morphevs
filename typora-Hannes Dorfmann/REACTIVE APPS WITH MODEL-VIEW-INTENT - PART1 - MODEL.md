# 使用 MVI 的反应性应用 - 第1部分 - MODEL

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART1 - MODEL](http://hannesdorfmann.com/android/mosby3-mvi-1)
>
> 作者：[Hannes Dorfmann ](http://hannesdorfmann.com/about/)

[TOC]

本文是博客文章系列“带模型 - 视图 - 意图的反应式应用程序”的一部分。 这里是目录：

- [Part 1: Model](http://hannesdorfmann.com/android/mosby3-mvi-1)
- [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)
- [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)
- [Part 4: Independent UI Components](http://hannesdorfmann.com/android/mosby3-mvi-4)
- [Part 5: Debugging with ease](http://hannesdorfmann.com/android/mosby3-mvi-5)
- [Part 6: Restoring State](http://hannesdorfmann.com/android/mosby3-mvi-6)
- [Part 7: Timing (SingleLiveEvent problem)](http://hannesdorfmann.com/android/mosby3-mvi-7)


一旦我发现我一直在模拟我的 Model 类错误，我以前在 Android 平台相关的一些主题上遇到的很多问题和头痛都没有了。 此外，我终于可以使用 RxJava 和 Model-View-Intent（MVI）构建反应式应用程序，尽管我迄今为止构建的应用程序也是反应性的，但与我无关 在这篇博客文章系列将要描述。 在第一部分，我想谈谈模型和为什么模型很重要。

那么我的意思是“以错误的方式模拟模型”？ 那么，有很多架构模式将“视图”从“模型”中分离出来。 至少在 Android 开发中，最受欢迎的是模型 - 视图 - 控制器（MVC），模型 - 视图 - 演示者（MVP）和模型 - 视图 - 视图模型（MVVM）。 你看到这些模式的名字，你注意到了什么吗？ 他们都谈论一个“模型”。 我意识到大部分时间我都没有一个模型。

例如：只需从后端加载一个人员列表。一个“传统”的 MVP 实现可能如下所示：

```java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().showLoading(true); // Displays a ProgressBar on the screen

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().showPersons(persons); // Displays a list of Persons on the screen
      }

      public void onError(Throwable error){
        getView().showError(error); // Displays a error message on the screen
      }
    });
  }
}
```

但是“模型”在哪里或什么？它是后端吗？不，那是商业逻辑。这是结果列表吗？不，这只是我们 View 中显示的一件事情，像加载指示符或错误消息。那么，“模型”究竟是什么？

从我的角度来看，应该有一个像这样的“模型”类：

```java
class PersonsModel {
  // In a real application fields would be private
  // and we would have getters to access them
  final boolean loading;
  final List<Person> persons;
  final Throwable error;

  public(boolean loading, List<Person> persons, Throwable error){
    this.loading = loading;
    this.persons = persons;
    this.error = error;
  }
}
```

然后演示者可以这样实现：

```java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().render( new PersonsModel(true, null, null) ); // Displays a ProgressBar

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().render( new PersonsModel(false, persons, null) ); // Displays a list of Persons
      }

      public void onError(Throwable error){
          getView().render( new PersonsModel(false, null, error) ); // Displays a error message
      }
    });
  }
}
```

现在视图有一个模型，然后将被“渲染”在屏幕上。 这个概念并不是什么新东西。 Trygve Reenskaug 从1979年开始的最初的 MVC 定义有一个相当类似的概念：View 观察模型的变化。 不幸的是，术语 MVC 被用来描述太多不同的模式，这与 Reenskaug 在1979年制定的模式并不相同。例如，后端开发者使用 MVC 框架，iOS 有ViewController，Android 上的 MVC 究竟是什么意思？ 活动是控制器？ 什么是 ClickListener 呢？ 如今，MVC 这个词只是一个很大的错误，对于 Reenskaug 原来所制定的错误和曲解是错误的。 但是让我们停止在这里讨论MVC，这可能会失控。

让我们回到我刚才所说的话。拥有“模型”解决了我们在 Android 开发中经常遇到的很多问题：

1. 状态问题
2. 屏幕方向更改
3. 后台堆栈的导航
4. 进程死亡
5. 不变性和单向数据流
6. 可调试和可重现的状态
7. 可测性

让我们来讨论这些问题，让我们看看 MVP 和 MVVM 的“传统”实现是如何处理这些问题的，以及“模型”如何帮助防止常见的陷阱。

## 1.状态问题

反应式应用程序 - 这是一个流行词，不是吗？我的意思是具有UI的应用程序会对状态更改做出反应。啊，这里还有一个很好的词：“状态”。什么是“状态”？那么大多数情况下，我们将“状态”描述为屏幕上显示的内容，例如View显示 ProgressBar 时的“加载状态”。其中关键在于：我们前端的开发人员倾向于关注 UI。这不一定是坏事，因为在一天结束时，一个好的用户界面会决定用户是否会使用我们的应用程序，因此应用程序的成功程度如何。但是看一下上面非常基本的 MVP 代码示例（不是使用 PersonsModel 的示例）。这里，UI 的状态由 Presenter 协调，因为 Presenter 告诉 View 显示什么。 MVVM也是如此。在这篇博文中，我想区分两种 MVVM 实现：第一种是Android 数据绑定，第二种是使用 RxJava。在具有数据绑定的 MVVM 中，状态直接位于 ViewModel 中：

```java
class PersonsViewModel {
  ObservableBoolean loading;
  // ... Other fields left out for better readability

  public void load(){

    loading.set(true);

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
      loading.set(false);
      // ... other stuff like set list of persons
      }

      public void onError(Throwable error){
        loading.set(false);
        // ... other stuff like set error message
      }
    });
  }
}
```

在使用 RxJava 的 MVVM 中，我们不使用数据绑定引擎，而是将 Observable 绑定到 View 中的 UI Widget，例如：

```java
class RxPersonsViewModel {
  private PublishSubject<Boolean> loading;
  private PublishSubject<List<Person> persons;
  private PublishSubject loadPersonsCommand;

  public RxPersonsViewModel(){
    loadPersonsCommand.flatMap(ignored -> backend.loadPersons())
      .doOnSubscribe(ignored -> loading.onNext(true))
      .doOnTerminate(ignored -> loading.onNext(false))
      .subscribe(persons)
      // Could also be implemented entirely different
  }

  // Subscribed to in View (i.e. Activity / Fragment)
  public Observable<Boolean> loading(){
    return loading;
  }

  // Subscribed to in View (i.e. Activity / Fragment)
  public Observable<List<Person>> persons(){
    return persons;
  }

  // Whenever this action is triggered (calling onNext() ) we load persons
  public PublishSubject loadPersonsCommand(){
    return loadPersonsCommand;
  }
}
```

当然，这些代码片段并不完美，你的实现可能看起来完全不同。重点是通常在 MVP 和 MVVM 中，状态由Presenter 或 ViewModel 驱动。

这导致了以下观察：

- 业务逻辑有它自己的状态，Presenter（或 ViewModel）有它自己的状态（并且你试图同步业务逻辑和 Presenter 的状态，使它们具有相同的状态），View 也可以有它自己的状态。 您直接在视图中设置可见性，或者 Android 本身在 recreation 期间从 bundle 中恢复状态）。
- Presenter（或 ViewModel）有任意多的输入（View 触发一个由 Presenter 处理的动作），但是 Presenter 也有许多输出（或者输出通道，比如 MVP 中的 view.showLoading（） 或 view.showError（） ViewModel 提供了多个 Observable），最终导致 View，Presenter 和业务逻辑的冲突状态，特别是在处理多个线程时。

在最好的情况下，这只会导致可见的错误，例如同时显示一个加载指示器（“加载状态”）和错误指示器（“错误状态”）：[plaid app: error and loading at the same time](https://www.youtube.com/watch?v=zCwESjEpNdk)

在最糟糕的情况下，您会从 Crashlytics 等崩溃报告工具向您报告一个严重的错误，您无法重现，因此几乎无法修复。

如果我们只有一个从底层（业务逻辑）到顶层（视图）传递的状态的单一来源的真理。实际上，在我们谈到“模型”时，在这篇博客的开头就已经看到了类似的概念。

```java
class PersonsModel {
  // In a real world application those fields would be private
  // and we would have getters to access them
  final boolean loading;
  final List<Person> persons;
  final Throwable error;

  public(boolean loading, List<Person> persons, Throwable error){
    this.loading = loading;
    this.persons = persons;
    this.error = error;
  }
}
```

你猜怎么了？ 模型反映了状态。 一旦我理解了这一点，许多状态相关的问题就解决了（并且从一开始就被阻止了），突然间我的Presenter只有一个输出：getView（）。render（PersonsModel）。 这反映了一个简单的数学函数，如 f（x）= y（对于多个输入也是可能的，即 f（a，b，c），正好是一个输出）。 数学可能不是每个人的一杯茶，但一个数学家不知道什么是一个错误。 软件工程师呢。

了解一个“模型”是什么以及如何正确地建模是非常重要的，因为最终模型可以解决“状态问题”。

## 2.屏幕方向更改

在 Android 屏幕放向变化是一个具有挑战性的问题。处理这个问题最简单的方法是忽略它。只需重新加载每个屏幕方向的变化。这是一个完全有效的解决方案。大多数情况下，您的应用程序也可以脱机工作，以便数据来自本地数据库或其他本地缓存。因此，在屏幕方向改变之后，加载数据是超快的。不过，我个人不喜欢看到一个加载指标，即使只是几毫秒，因为在我看来，这不是一个无缝的用户体验。因此，人们（包括我自己）开始使用 MVP 和“保留演示者”，以便在屏幕方向更改期间视图可以被分离（并且被销毁），而演示者在内存中存活，然后视图被重新连接到演示者。 MVVM 与 RxJava 可能有相同的概念，但是我们必须记住，一旦 View 从他的 ViewModel 取消订阅，可观察流将被销毁。例如，你可以用 Subjects 来解决这个问题。在具有数据绑定的 MVVM 中，ViewModel 通过数据绑定引擎自己直接绑定到 View 上。为了避免内存泄漏，我们必须在屏幕方向更改上销毁 ViewModel。

但是保留 Presenter（或 ViewModel）的问题是：我们如何将 View 的状态恢复到屏幕方向更改之前的状态，以便 View 和 Presenter 再次处于同一状态？ 我已经写了一个叫做 [Mosby](https://github.com/sockeqwe/mosby) 的 MVP 库，它具有一个名为 ViewState 的功能，它基本上同步了您的业务逻辑与 View 的状态。 另一个 MVP 库 [Moxy](https://github.com/Arello-Mobile/Moxy) 通过使用“命令”在屏幕方向更改后重现 View 的状态，实现了一个非常有趣的解决方案：

![](http://hannesdorfmann.com/images/mvi-mosby3/moxy.gif)

我非常肯定，还有其他解决方案的视图的状态问题。让我们回过头来总结这些库试图解决的问题：他们试图解决我们已经讨论过的状态问题。

因此，再一次，有一个反映当前 “状态” 的 “模型” 和一个方法来 “渲染” “模型” 解决这个问题就像调用getView().render(PersonsModel) 一样简单（用最新的模型当重新附加演示者的视图时）。

## 3.后台堆栈的导航

当视图不再被使用时，是否需要保留 Presenter（或 ViewModel）？ 例如，如果由于用户导航到另一个屏幕而导致碎片（视图）已被替换为另一个碎片，则没有视图附加到演示者。 如果没有附加视图，则演示者显然不能使用来自业务逻辑的最新数据来更新视图。 如果用户回来了（即按下后退按钮弹出后台堆栈）？ 重新加载数据或重新使用现有的 Presenter？ 这更多的是一个哲学问题。 通常一旦用户回到上一个屏幕（弹出堆栈），他会希望继续他离开的地方。 这基本上就是在2中讨论的“恢复视图的状态问题”。所以解决方案很简单：用一个代表状态的“模型”，我们只需 getView().render(PersonsModel) 来渲染视图。

## 4.进程死亡

我认为这是 Android 开发中常见的误解，认为程序死亡是一件坏事，我们需要库来帮助我们在死亡之后恢复状态（以及 Presenter 或 ViewModels）。首先，一个进程死亡只会有一个很好的原因：Android 操作系统需要更多的资源用于其他应用程序或节省电池。但是，当你的应用程序处于前台并且被应用程序用户积极使用时，这绝不会发生。所以，做一个好的公民，不要与平台作斗争。如果你真的有一些长期的工作要做，在后台使用一个服务，因为这是唯一的方式来通知操作系统，你的应用程序仍然是“积极使用”。如果发生进程死亡，Android 会提供一些回调，如 onSaveInstanceState（）来保存状态。再次，状态。我们应该保存我们的视图信息到 Bundle？演示者是否有自己的状态，我们也必须保存到 bundle 中？什么是业务逻辑状态？我们已经有了这个舞蹈：如1.2和3.中所描述的，我们只需要一个表示整个状态的模型类。然后很容易将这个模型保存到一个包中并在之后恢复。不过，我个人认为大多数情况下最好不要保存状态，而是重新加载整个屏幕，就像我们在第一次应用程序启动时所做的那样。想想 NewsReader 应用程序显示新闻文章的列表。当我们的应用程序被杀害，我们保存状态，6小时后，用户重新打开我们的应用程序，状态恢复，我们的应用程序可能会显示过时的内容。在这种情况下可能不存储模型/状态并简单地重新加载数据。

## 5.不变性和单向数据流

我不会谈论不可变性的优点，因为这个主题有很多可用的资源。我们想要一个不可变的“模型”（代表状态）。为什么？因为我们只需要一个单一的真理来源。我们不希望我们的应用程序中的其他组件可以在我们传递 Model 对象时操作模型/状态。让我们想象一下，我们将编写一个简单的 “计数器” Android 应用程序，它具有递增和递减按钮，并在 TextView 中显示当前的计数器值。如果我们的模型（在这种情况下只是计数器的值 - 一个整数）是不变的，我们如何改变计数器？我很高兴你问。我们不是直接在每个按钮点击操纵 TextView。一些观察：首先，我们的视图应该只有一个 view.render（...）。其次，我们的模型是不变的，所以模型的直接变化是不可能的。第三，真相只有一个来源：商业逻辑。我们让点击事件 “下沉” 到业务逻辑。业务逻辑知道当前模型（即具有当前模型的私有字段），并将根据旧模型创建具有递增/递减值的新模型。

![](http://hannesdorfmann.com/images/mvi-mosby3/counter.png)

通过这样做，我们建立了一个单向数据流，业务逻辑作为创建不可变模型实例的单一事实源。 但是，这看起来好像只是一个简单的柜台而已，不是吗？ 是的，一个计数器只是一个简单的应用程序。 大多数应用程序从一个简单的应用程序开始，但复杂性增长迅速。 即使对于简单的应用程序来说，单向数据流和不可变模型也是非常必要的，以确保在复杂性增长时它们保持简单（从开发人员的角度来看）。

## 6.可调试和可重现的状态

而且，单向数据流确保了我们的应用程序易于调试。 下次我们从 Crashlytics 那里得到一个崩溃报告，我们可以很容易地重现和修复这个崩溃，这不是很好吗，因为所有必需的信息都附加到崩溃报告中。 什么是“所需的信息”？ 那么，我们需要的所有信息就是当前模型和用户在发生崩溃时想要执行的操作（即单击递减按钮）。 这就是我们需要重现这次崩溃的信息，而且这些信息非常容易记录并附加到崩溃报告中。 如果没有单向的数据流（即有人误用 EventBus 并将 CounterModel 激发出来）或者没有不变性（这样我们就不确定谁实际上已经改变模型），这将不那么容易。

## 7.可测性

“传统” MVP 或 MVVM 提高了应用程序的可测试性。 MVC 也是可测试的：从来没有人说过，我们必须把所有的业务逻辑放在活动中。 使用表示状态的模型，我们可以简化单元测试的代码，因为我们可以简单地检查 assertEquals（expectedModel，model）。 这消除了许多我们必须模拟的对象。 此外，这还会删除许多特定方法的验证测试，即 Mockito.verify（view，times（1））。showFoo（）。 最终，这使得我们的单元测试的代码更具可读性，可理解性并且最终可维护，因为我们不必处理实际代码的实现细节。

## 结束

在这个博客系列的第一部分，我们谈了很多关于理论的东西。 我们是否真的需要一个关于模型的专门博客文章？ 我认为这是一个基本的理解，一个模型是重要的，有助于防止一些问题，否则我们会疑惑。 模型并不意味着业务逻辑。 这是生成模型的业务逻辑（即 Interactor，Usecase，Repositor 或其他任何您在应用程序中调用的逻辑）。 在第二部分中，当我们最终用 Model-View-Intent 构建一个反应式应用程序时，我们将会看到这个理论模型。 我们要构建的演示应用程序是一个虚构的在线商店的应用程序。 这里只是一个简短的预览演示，你可以期望在第二部分。 敬请关注。[Model-View-Intent: shopping cart example](https://www.youtube.com/watch?v=rmR9mV1Dsqk)

