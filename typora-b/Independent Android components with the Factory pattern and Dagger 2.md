# 使用工厂模式和 Dagger 2 来独立 Android 组件

> 原文 (Medium)：[Independent Android components with the Factory pattern and Dagger 2](https://medium.com/@remcomokveld/independent-android-components-with-the-factory-pattern-and-dagger-2-f1e631fdc869)
>
> 作者：[Remco Mokveld](https://medium.com/@remcomokveld?source=post_header_lockup)

[TOC]

面向对象编程的一个重要概念是，类应该有低耦合。 我在大学里学到了这个概念，但当时我并没有完全理解它的含义。 这是一个我可以看到理论的东西，但是为了完全理解它，我需要看到它在实际生产代码中的作用。在本文中，我将试图给出一个很好的例子，说明为什么它对于编写可读且可维护的 Android 应用程序确实很重要。 

自[1979](https://www.win.tue.nl/~wstomv/quotes/structured-design.html#6)年以来，低耦合已经在编程领域得到讨论，但仍然非常有效。 为了确定两个类之间的耦合程度，必须回答以下问题: 

> 为了理解另一个模块，必须知道多少模块？ 

这里的模块可以有很多不同的尺寸，但是一个普遍接受的最佳实践是尽可能小的模块。如果一个模块变得太大，它应该成为其他小模块的组成。我喜欢在 Kotlin 中看到一个包作为一个模块，这个模块由一组较小的模块组成，这些模块就是包中的类。 

现在，在某些情况下，两个类之间知道很多关于彼此的知识并且彼此紧密结合是有益的，但总的来说，我尽量避免这种情况发生，不仅仅是因为它被认为是一个糟糕的设计，而是因为我发现测试紧密耦合的类非常困难。 

一个典型的紧耦合例子可以在下面的类中看到。 这是一个带有新闻文章列表的屏幕的部分代码。 

```kotlin
class NewsListAdapter @Inject constructor(
  val htmlParser: MyHtmlParser,
  val picasso: Picasso
) : RecyclerView.Adapter() {
  var rows = emptyList<NewsRowViewModel>()
  fun showNewsArticles(articles: List<NewsArticle>) {
    rows = articles.map { 
      NewsRowViewModel(htlmParser, picasso, it)
    }
    notifyDataSetChanged()
  }
}
class NewsRowViewModel(
  val htmlParser: MyHtmlParser,
  val picasso: Picasso,
  val artice: NewsArticle
) {
  fun getTitle() = htmlParser.parse(article.titleHtml)
  fun getThumbnailRequest() = picasso.load(article.thumbnailUrl)
}
```

在这种情况下，NewsListAdapter 知道 NewsRowViewModel 所具有的每个依赖关系，它知道它需要一个 HtmlParser 来显示标题，并且知道它需要一个 Picasso 实例来加载缩略图。 所以适配器和 NewsRowViewModel 之间的耦合非常紧密。

如果在适配器中只有一种类型的行，这可能还不是一个很大的问题，但随着适配器复杂性的增长，你经常会看到不同种类的 ViewModel，它们具有不同的依赖关系，这会迅速地混乱你的代码，并使读取适配器做什么变得更加困难，因为 ViewModel 使用什么样的依赖性并不清楚。另外，如果出于某种原因，你想将图像加载库从 Picasso 更改为 Glide，你需要在 NewsRowViewModel 和 NewsListAdapter 中改变这一点。 

## 向Dagger 2呼叫救援

在很多情况下，Dagger 2 可以帮助很多减少耦合的程度，不让客户阶层担心它的依赖关系是如何构建的。在继续前面的例子之前，让我们来看看如何在 Dagger 中使用 NewsListAdapter 本身。 构造函数有一个 @Inject 注解，所以它可以被注入到代码的任何地方。  在一个典型的场景中， 适配器将被注入到 NewsListActivity 中，如下面的代码所示。

```kotlin
class NewsListActivity : AppCompatActivity() {
  @Inject
  lateinit var adapter: NewsListAdapter
  override fun onCreate(savedInstanceState: Bundle?) {
    AndroidInjection.inject(this)
    super.onCreate(savedInstanceState)
  }
}

```

现在有了 NewsRowViewModel ，这是不可能的，因为它在其构造函数中采用了不在 Dagger 对象图中的 NewsArticle 实例。 

这就是[工厂模式](http://www.oodesign.com/factory-pattern.html)发挥作用的地方。 工厂模式的意图是: 

- 创建对象，而不向客户层暴露实例逻辑。 
- 通过通用接口引用新创建的对象。 

在这种情况下，为了减少耦合，我不一定要寻找创建具有通用接口的对象的东西。 这里没有不同实现的接口。 然而，它希望隐藏 NewsListAdapter 中 NewsRowViewModel 的实例化逻辑。 

当创建实例化单个类的工厂时，一个常见的模式是将该工厂定义为它所实例化的类的静态内部类。 

通过添加构造函数注入，可以注入所有与基于文章实例化新的 NewsRowViewModel 的类无关的所有依赖项。 然后，该文章可以作为参数传递给工厂方法。 

这是一个带有 Factory 内部类的 NewsRowViewModel 的例子。

```kotlin
class NewsRowViewModel(
  val htmlParser: MyHtmlParser,
  val picasso: Picasso,
  val artice: NewsArticle
) {
  fun getTitle() = htmlParser.parse(article.titleHtml)
  fun getThumbnailRequest() = picasso.load(article.thumbnailUrl)

  class Factory @Inject constructor(
    val htmlParser: HtmlParser, 
    val picasso: Picasso) {
    fun create(article: NewsArticle) {
      return NewsRowViewModel(htmlParser, picasso, article)
    }
  }
}
```

这就是 NewsListAdapter 如何使用它。

```kotlin
class NewsListAdapter @Inject constructor(
  val rowFactory: NewsRowViewModel.Factory
) : RecyclerView.Adapter() {
var rows = emptyList<NewsRowViewModel>()
fun showNewsArticles(articles: List<NewsArticle>) {
    rows = articles.map { rowFactory.create(it) }
    notifyDataSetChanged()
  }
}
```

这样所有的适配器都知道，它有一个成员变量，它可以从新闻报道中创建一个 NewsRowViewModel，这就是它所需要知道的。 它还使 Adapter 类更容易阅读，从构造函数中可以很清楚地看到它很可能是创建 NewsRowViewModels，并且没有其他的依赖来混乱这个意图。 

这种方法的一大好处是，如果 NewsRowViewModel 获得额外的依赖关系，你不需要在 NewsListAdapter 中更改任何内容。 例如，如果 NewsRowViewModel 中需要一个  NewsListViewModel，它可以直接注入  NewsRowViewModel.Factory ，然后传递给 NewsRowViewModel 的构造函数。 

如果一个新的要求是在列表的第5和第10位置应该有一个广告， 而第一个应该是预先提取适配器中的所有可能发生的情况，那就是它将得到一个 AdViewModel.Factory 被注入，在新闻文章映射功能中视图模型应该是如下这样的 : 

```kotlin
fun showNewsArticles(articles: List<NewsArticle>) {
    rows = articles.map { rowFactory.create(it) }
    rows.add(4, adFactory.create(prefetch = true)
    rows.add(9, adFactory.create(prefetch = false)
    notifyDataSetChanged()
  }
}
```

同样，适配器现在知道创建一个新的 AdViewModel 所需要知道的所有信息，而不需要了解 NewsRowViewModel 或 AdViewModel 内部发生的事情。 如果其中一个的依赖关系不能满足，Dagger 会给你一个清晰的编译错误。

请注意，这种模式不仅可以应用于适配器中，而且在许多其他场景中，你的依赖关系中的一些，并不是全部是可注入的。 我希望你发现这篇文章有助于写出可读且可维护的代码。 

