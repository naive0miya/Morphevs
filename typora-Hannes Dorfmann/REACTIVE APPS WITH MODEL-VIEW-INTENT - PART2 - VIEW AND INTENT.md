# 使用 MVI 的反应性应用 - 第2部分 - VIEW 和 INTENT

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART2 - VIEW AND INTENT](http://hannesdorfmann.com/android/mosby3-mvi-2)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about)

[TOC]

本文是博客文章系列“带模型 - 视图 - 意图的反应式应用程序”的一部分。 这里是目录：

- [Part 1: Model](http://hannesdorfmann.com/android/mosby3-mvi-1)
- [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)
- [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)
- [Part 4: Independent UI Components](http://hannesdorfmann.com/android/mosby3-mvi-4)
- [Part 5: Debugging with ease](http://hannesdorfmann.com/android/mosby3-mvi-5)
- [Part 6: Restoring State](http://hannesdorfmann.com/android/mosby3-mvi-6)
- [Part 7: Timing (SingleLiveEvent problem)](http://hannesdorfmann.com/android/mosby3-mvi-7)


在第一部分中，我们讨论了一个模型实际是什么，与状态的关系以及一个明确定义的模型如何解决Android开发中的一些常见问题。 在这篇博客文章中，我们通过引入 Model-View-Intent 模式来构建反应式应用程序，继续走向“反应式应用程序”。

如果你还没有阅读第一部分，那么在继续阅读这篇博客文章之前，你应该先阅读一下。总结：而不是像这样编写代码（例如在 “传统” MVP 中）。

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

我们应该创建一个反映 “状态” 的 “模型”：

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

现在视图有一个模型，然后通过调用 render（personsModel），在屏幕上 “渲染” 模型。 在第一部分中，我们还谈到了单向数据流的重要性，您的业务逻辑应该推动这种模式。 在我们开始连接之前，让我们快速讨论 MVI 的主要思想。

## Model-View-Intent (MVI)

[André Medeiros (Staltz)](https://twitter.com/andrestaltz) 针对他编写的称为 [cycle.js](https://cycle.js.org/) 的 JavaScript 框架指定了这种模式。从理论（和数学）的角度来看，我们可以如下描述 Model-View-Intent：

![](http://hannesdorfmann.com/images/mvi/mvi-func2.png)

- intent( ) : 这个函数接受来自用户的输入（即 UI 事件，比如点击事件），并将其转换为 “something”，作为参数传递给 model（）函数。 这可能是一个简单的字符串，可以将模型的值设置为或更复杂的数据结构，如 Object。 我们可以说我们有意图改变模型。
- model( ) : model（）函数将 intent（） 的输出作为输入来操作 Model。 这个函数的输出是一个新的模型（状态改变）。 所以它不应该更新一个已经存在的模型。 我们要不变性！ 在第一部分中，我给出了一个“反应用程序”的具体例子。 同样，我们不改变已经存在的模型对象实例。 我们根据意图描述的变化创建一个新的模型。 请注意，model（）函数是您的代码中唯一允许创建新 Model 对象的代码片段。 那么这个新的不可变模型就是这个函数的输出。 基本上，model（）函数调用我们的应用业务逻辑（可以是 Interactor，Usecase，Repository ...您在应用中使用的任何模式/术语），并提供一个新的 Model 对象作为结果。
- view( ) : 这个方法使用从 model（）函数返回的模型，并将其作为 view（）函数的输入。然后视图以某种方式简单地显示这个模型。 view（）与 view.render（model）基本相同。

## 用 RxJava 连接点

我们希望我们的数据流向单向。 RxJava 发挥了作用。 我们是否需要 RxJava 来构建具有单向数据流或基于 MVI 的应用程序的反应式应用程序？ 不，我们也可以编写命令和程序代码。 但是，RxJava 对于基于事件的编程非常有用。 既然用户界面是基于事件的，使用 RxJava 也是很有意义的。

在这篇博文中，我们将为一家虚构的在线商店构建一个简单的应用程序。 我们向后端发出 http 请求，以加载我们在应用中显示的产品。 我们可以搜索某些产品，并将产品添加到购物篮中。 总的来说，最终的应用程序如下所示。[Model-View-Intent: shopping cart example](https://www.youtube.com/watch?v=rmR9mV1Dsqk)

源代码可以在 [github](https://github.com/sockeqwe/mosby/tree/master/sample-mvi)上找到。 让我们开始实现一个简单的屏幕：让我们执行搜索。 首先，我们定义一个模型，最后在本博客文章系列的第一部分描述的 View 中显示。 在这个博客文章系列中，我们给所有的 Model 类添加了一个 “ViewState” 后缀，也就是说我们的 Model 类用于在我们的在线商店应用程序中搜索产品，因为Model反映了这个状态。 像 SearchModel 这样的替代名称听起来有点奇怪，或者 SearchViewModel 可能会导致与 MVVM 混淆。 命名很难。

```java
public interface SearchViewState {

  /**
   * The search has not been stared yet
   */
  final class SearchNotStartedYet implements SearchViewState {
  }

  /**
   * Loading: Currently waiting for search result
   */
  final class Loading implements SearchViewState {
  }

  /**
   * Indicates that the search has delivered an empty result set
   */
  final class EmptyResult implements SearchViewState {
    private final String searchQueryText;

    public EmptyResult(String searchQueryText) {
      this.searchQueryText = searchQueryText;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }
  }

  /**
   * A valid search result. Contains a list of items that have matched the searching criteria.
   */
  final class SearchResult implements SearchViewState {
    private final String searchQueryText;
    private final List<Product> result;

    public SearchResult(String searchQueryText, List<Product> result) {
      this.searchQueryText = searchQueryText;
      this.result = result;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }

    public List<Product> getResult() {
      return result;
    }
  }

  /**
   * Indicates that an error has occurred while searching
   */
  final class Error implements SearchViewState {
    private final String searchQueryText;
    private final Throwable error;

    public Error(String searchQueryText, Throwable error) {
      this.searchQueryText = searchQueryText;
      this.error = error;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }

    public Throwable getError() {
      return error;
    }
  }
}
```

由于 Java 是一种强类型语言，因此我们通过在自己的类中分割每个 “子状态” 来为 Model 类选择一种类型安全的方法。 我们的业务逻辑返回一个 SearchViewState 类型的对象，它可能是一个 SearchViewState.Error 的实例等。这只是个人喜好。 我们也可以将这个完全不同的模型，例如：

```java
class SearchViewState {
  Throwable error; // if not null, an error has occurred
  boolean loading; // if true loading data is in progress
  List<Product> result; // if not null this is the result of the search
  boolean SearchNotStartedYet; // if true, we have the search not started yet
}
```

同样，你如何建模你的模型只是个人喜好的问题。如果您使用 kotlin 编程语言，那么密封类是一个很好的选择。

接下来，让我们关注业务逻辑。我们来介绍一个负责执行搜索的 SearchInteractor。如前所述，“输出” 是一个 SearchViewState 对象。

```java
public class SearchInteractor {
  final SearchEngine searchEngine; // Makes http calls

  public Observable<SearchViewState> search(String searchString) {
    // Empty String, so no search
    if (searchString.isEmpty()) {
      return Observable.just(new SearchViewState.SearchNotStartedYet());
    }

    // search for products
    return searchEngine.searchFor(searchString) // Observable<List<Product>>
        .map(products -> {
          if (products.isEmpty()) {
            return new SearchViewState.EmptyResult(searchString);
          } else {
            return new SearchViewState.SearchResult(searchString, products);
          }
        })
        .startWith(new SearchViewState.Loading())
        .onErrorReturn(error -> new SearchViewState.Error(searchString, error));
  }
}
```

我们来看看 SearchInteractor.search（）的方法签名：我们有 String searchString 作为输入参数，Observable <SearchViewState> 作为输出。 这已经暗示我们期望随着时间的推移在这个可观察的流上发射任意多个 SearchViewState 实例。 startWith（）表示在我们实际开始搜索查询（通过执行http请求的 SearchEngine）之前，我们发出 SearchViewState.Loading。 最后，这将强制View在执行搜索时显示一个 ProgressBar。

onErrorReturn（）捕获执行搜索时可能发生的任何异常，并发出 SearchViewState.Error。当我们订阅这个Observable 时，我们不能只使用 onError 回调吗？这是 RxJava 常见的误解：当整个可观察流遇到不可恢复的错误，因此可观察流终止时，错误回调被使用。在我们的情况下，像没有活跃的互联网连接错误不是一个不可恢复的错误。它还只是我们模式所代表的另一个状态。此外，我们可以在之后移动到另一个状态，即一旦有效的互联网连接可用，我们可以移动到由 SearchViewState.Loading 表示的“加载状态”。因此，我们从业务逻辑到每当“状态”发生变化时发布一个变化的模型，我们建立一个可观察的流。我们当然不希望在因特网连接错误中终止这个可观察的流。因此，这样的错误被作为一个状态来处理（而不是终止流的致命错误），当模型发生错误时，这个错误被模型反射并发送到可观察流。通常在 MVI 中，Model Observable 永远不会终止（永远不会到达用户的onComplete（）或 onError（））。

综上所述：SearchInteractor（业务逻辑）提供了一个可观察的流 Observable <SearchViewState>，并在每次状态改变时发出一个新的 SearchViewState。

接下来我们来讨论我们的 View 层是怎样的。 视图应该做什么？ 那么显然这个视图应该显示这个模型。 我们已经同意 View 应该有一个像 render（model）这样的函数。 另外，视图应该为其他图层提供一种对用户输入事件作出反应的方式。 这些在 MVI 中被称为意图。 在我们的例子中，只有一个意图：用户可以通过在输入字段中输入字符串来搜索产品。 我们以与MVP类似的方式实施 MVI。 在 MVP 中为 View 层定义一个接口是一个好习惯，所以我们也在 MVI 中做这个。

```java
public interface SearchView {

  /**
   * The search intent
   *
   * @return An observable emitting the search query text
   */
  Observable<String> searchIntent();

  /**
   * Renders the View
   *
   * @param viewState The current viewState state that should be displayed
   */
  void render(SearchViewState viewState);
}
```

在这种情况下，我们的视图只提供一个意图，但是一般来说视图可以提供多种意图。 在第1部分中，我们讨论了为什么单个 render（）函数是一个好方法，如果不清楚为什么我们应该更喜欢单个渲染（），则应该再次阅读第1部分（或者在下面留下注释;另请参见部分注释部分1）。 在开始查看图层的具体实现之前，让我们看看最终的结果应该如何：[mvi search example](https://www.youtube.com/watch?v=07pkkbFQhtk)

```java
public class SearchFragment extends Fragment implements SearchView {

  @BindView(R.id.searchView) android.widget.SearchView searchView;
  @BindView(R.id.container) ViewGroup container;
  @BindView(R.id.loadingView) View loadingView;
  @BindView(R.id.errorView) TextView errorView;
  @BindView(R.id.recyclerView) RecyclerView recyclerView;
  @BindView(R.id.emptyView) View emptyView;
  private SearchAdapter adapter;

  @Override public Observable<String> searchIntent() {
    return RxSearchView.queryTextChanges(searchView) // Thanks Jake Wharton :)
        .filter(queryString -> queryString.length() > 3 || queryString.length() == 0)
        .debounce(500, TimeUnit.MILLISECONDS);
  }

  @Override public void render(SearchViewState viewState) {
    if (viewState instanceof SearchViewState.SearchNotStartedYet) {
      renderSearchNotStarted();
    } else if (viewState instanceof SearchViewState.Loading) {
      renderLoading();
    } else if (viewState instanceof SearchViewState.SearchResult) {
      renderResult(((SearchViewState.SearchResult) viewState).getResult());
    } else if (viewState instanceof SearchViewState.EmptyResult) {
      renderEmptyResult();
    } else if (viewState instanceof SearchViewState.Error) {
      renderError();
    } else {
      throw new IllegalArgumentException("Don't know how to render viewState " + viewState);
    }
  }

  private void renderResult(List<Product> result) {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.VISIBLE);
    loadingView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    adapter.setProducts(result);
    adapter.notifyDataSetChanged();
  }

  private void renderSearchNotStarted() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderLoading() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.VISIBLE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderError() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.VISIBLE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderEmptyResult() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.VISIBLE);
  }
}
```

**render(SearchViewState)**  方法应该是自我解释。 在 searchIntent（）中，我们使用了 Jake Wharton 的 [RxBindings](https://github.com/JakeWharton/RxBinding) 库，它提供了像 Observable for Android UI 小部件这样的 RxJava 绑定。 RxSearchView.queryText（）创建一个 Observable <String>，每次用户在 EditText UI 小部件中键入内容时，都会发出搜索字符串。 如果用户键入的字符数超过3个字符，我们使用 filter（）来仅启动一个搜索查询，并且我们不希望每次用户输入新字符时都要点击后端，而是等待用户完成 键入（debounce（）等待500毫秒以确定用户是否已完成键入）。

所以我们知道这个屏幕的 “输入” 是 searchIntent（），render（）是 “输出”。 我们如何从 “输入” 到 “输出”？ 以下视频形象地显示：[mvi search explained](https://www.youtube.com/watch?v=oo0SBtqKhMw)

剩下的问题是我们如何将 View 的意图与业务逻辑联系起来？ 如果仔细看看上面的视频，你会发现中间有一个 flatMap（）运算符。 这已经暗示我们还有一个额外的组件，我们还没有谈到：演示者。 演示者负责连接类似于我们在 MVP 中使用演示者的点。

```java
public class SearchPresenter extends MviBasePresenter<SearchView, SearchViewState> {
  private final SearchInteractor searchInteractor;

  @Override protected void bindIntents() {
    Observable<SearchViewState> search =
        intent(SearchView::searchIntent)
            .switchMap(searchInteractor::search) // I have used flatMap() in the video above, but switchMap() makes more sense here
            .observeOn(AndroidSchedulers.mainThread());

    subscribeViewState(search, SearchView::render);
  }
}
```

注意，SearchView :: searchIntent 只是一个 Java 8 的简写 searchView.searchIntent（）

什么是 MviBasePresenter 什么是 intent（）和 subscribeViewState（）？ 这个类是我写的一个叫 [Mosby](https://github.com/sockeqwe/mosby) 库的一部分（Mosby 3.0 有一个 MVI 模块）。 这个博客文章不是关于 Mosby 的，但是我想简要地谈一下 MviBasePresenter 是如何工作来说服你，虽然我不得不承认第一眼看起来就是这样。 让我们从生命周期开始吧： MviBasePresenter 并没有真正的生命周期。 有一个 bindIntent（）方法将视图的意图绑定到业务逻辑。 通常，使用 flatMap（）或 switchMap（）或 concatMap（）将意图“转发”到业务逻辑。 仅在首次将视图附加到Presenter时调用此方法。 当视图被重新附加时（即在屏幕方向改变之后）它不会被再次调用。

这可能听起来有些奇怪，有人会问：“ MviBasePresenter 是否能够保持屏幕方向的变化，如果是的话，Mosby 如何确保可观察的流仍然没有泄漏内存？”这就是 intent（）和 subscribeViewState（）的意图。 intent（）在内部创建一个 PublishSubject，并将它用作业务逻辑的“网关”。 所以实际上这个 PublishSubject 正在订阅 View 的intent Observable。 调用 intent（o1）实际上返回一个订阅了 o1 的 PublishSubject。

在方向更改上 Mosby 将 View 从 Presenter 分离出来，但只从视图中暂时取消订阅此内部 PublishSubject，并在视图重新连接到演示者时将 PublishSubject 重新订阅到 View 的意图。

subscribeViewState（）做相同的，但相反（Presenter to View 通信）。 它在内部创建一个 BehaviorSubject 作为从业务逻辑到 View 的“网关”。 既然它是一个 BehaviorSubject，我们可以从业务逻辑中获得“模型更新”，即使此时没有视图被添加（即视图在后面的栈上）。 BehaviorSubjects 始终保持它收到的最新值，并在 View 重新连接后重播。

规则很简单：使用 intent（）来“包装”视图的任何意图。使用 subscribeViewState（）而不是 Observable.subscribe（...）。

![](http://hannesdorfmann.com/images/mvi-mosby3/MviBasePresenter.png)

bindIntent（）的计数器部分是 unbindIntents（），它永远被调用一次。 比如把一个片段放在后面的堆栈上不会永久的破坏 View，但是完成一个 Activity。 由于 intent（）和 subscribeViewState（）已经负责订阅管理，所以你几乎不需要实现 unbindIntents（）。

那么像 onPause（）或 onResume（）等其他生命周期事件呢？ 我仍然认为[演示者不需要生命周期事件](http://hannesdorfmann.com/android/presenters-dont-need-lifecycle)。 但是，如果你真的认为你需要它们，你可以简单地看到像 onPause（）这样的生命周期事件。 你的视图可以提供一个由 android 生命周期触发的 pauseIntent（），而不是像点击按钮那样的用户交互意图。 但两者都是有效的意图。

## 结束

在第二部分中，我们讨论了 Model-View-Intent 的基础知识，并通过使用 MVI 来实现一个非常简单的屏幕。 也许这个例子太简单了，以至于你还没有完全看到 MVI 模式的好处，MVI 模式是一个表示状态的模型，与 “传统” MVP 或 MVVM 相比是单向数据流。 MVP 或 MVVM 没有错，我不是说 MVI 比其他架构模式更好。 但是，我认为 MVI 可以帮助我们为复杂的问题编写优雅的代码，正如我们将在本系列的下一部分（[第3部分](http://hannesdorfmann.com/android/mosby3-mvi-3)）中看到的那样，当我们要讨论状态约束的时候。

