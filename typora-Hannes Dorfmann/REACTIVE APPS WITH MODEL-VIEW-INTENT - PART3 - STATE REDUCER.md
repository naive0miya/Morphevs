# 使用 MVI 的反应性应用 - 第3部分 - STATE REDUCER

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART3 - STATE REDUCER](http://hannesdorfmann.com/android/mosby3-mvi-3)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

本文是博客文章系列“带模型 - 视图 - 意图的反应式应用程序”的一部分。 这里是目录：

- [Part 1: Model](http://hannesdorfmann.com/android/mosby3-mvi-1)
- [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)
- [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)
- [Part 4: Independent UI Components](http://hannesdorfmann.com/android/mosby3-mvi-4)
- [Part 5: Debugging with ease](http://hannesdorfmann.com/android/mosby3-mvi-5)
- [Part 6: Restoring State](http://hannesdorfmann.com/android/mosby3-mvi-6)
- [Part 7: Timing (SingleLiveEvent problem)](http://hannesdorfmann.com/android/mosby3-mvi-7)


在前面的部分中，我们讨论了如何使用单向数据流实现带有 Model-View-Intent 模式的简单屏幕。 在这篇博客文章中，我们将借助状态减速器，用 MVI 构建一个更复杂的屏幕。

如果您还没有阅读第2部分，那么在继续阅读本博客文章之前，您应该先阅读这篇文章，因为介绍了如何通过 Presenter 连接业务逻辑和数据如何单向流动。

现在让我们来构建一个更复杂的屏幕：[mvi complex home screen](https://www.youtube.com/watch?v=WWeRn0tMoXM)

正如您在上面的视频中所看到的，此屏幕显示按类别分组的项目（产品）列表。 该应用程序只显示每个类别的3个项目，用户可以点击 “加载更多按钮” 加载该类别的所有项目（http 请求）。 另外，用户可以进行拉到刷新，并且一旦用户向下滚动到列表的末尾，更多类别被加载（分页）。 当然，所有这些行为都可以同时执行，并且每个行为也可能失败（即没有互联网连接）。

让我们一步一步来实现。首先，我们来定义 View 界面。

```java
public interface HomeView {

  /**
   * The intent to load the first page
   *
   * @return The value of the emitted item (boolean) can be ignored. true or false has no different meaning.
   */
  public Observable<Boolean> loadFirstPageIntent();

  /**
   * The intent to load the next page (pagination)
   *
   * @return The value of the emitted item (boolean) can be ignored. true or false has no different meaning.
   */
  public Observable<Boolean> loadNextPageIntent();

  /**
   * The intent to react on pull-to-refresh
   *
   * @return The value of the emitted item (boolean) can be ignored. true or false has no different meaning.
   */
  public Observable<Boolean> pullToRefreshIntent();

  /**
   * The intent to load more items from a given category
   *
   * @return Observable with the name of the category
   */
  public Observable<String> loadAllProductsFromCategoryIntent();

  /**
   * Renders the viewState
   */
  public void render(HomeViewState viewState);
}
```

具体的 View 实现非常简单，因此我不会在这里显示代码（可以在 [github](https://github.com/sockeqwe/mosby/blob/master/sample-mvi/src/main/java/com/hannesdorfmann/mosby3/sample/mvi/view/home/HomeFragment.java) 上找到）。 接下来让我们关注一下这个模型。 如以前的文章所述，模型应反映状态。 那么让我们来介绍一个名为 HomeViewState 的模型：

```java
public final class HomeViewState {

  private final boolean loadingFirstPage; // Show the loading indicator instead of recyclerView
  private final Throwable firstPageError; // Show an error view if != null
  private final List<FeedItem> data;   // The items displayed in the recyclerview
  private final boolean loadingNextPage; // Shows the loading indicator for pagination
  private final Throwable nextPageError; // if != null, shows error toast that pagination failed
  private final boolean loadingPullToRefresh; // Shows the pull-to-refresh indicator
  private final Throwable pullToRefreshError; // if != null, shows error toast that pull-to-refresh failed

   // ... constructor ...
   // ... getters  ...
}
```

请注意，FeedItem 只是每个项目必须实现的界面，可以通过 RecyclerView 显示。 例如，产品实现了 FeedItem。 此外，在回收站 SectionHeader 中显示的类别标题实现了 FeedItem。 指示该类别的更多项目可以加载的 UI 元素是一个 FeedItem，并在内部保存它自己的小状态，以指示我们是否正在加载更多的某个类别的项目：

```java
public class AdditionalItemsLoadable implements FeedItem {
  private final int moreItemsAvailableCount;
  private final String categoryName;
  private final boolean loading; // if true then loading items is in progress
  private final Throwable loadingError; // indicates an error has occurred while loading

   // ... constructor ...
   // ... getters  ...
```

最后但并非最不重要的是有一个业务逻辑组件 HomeFeedLoader 负责加载 FeedItems：

```java
public class HomeFeedLoader {

  // Typically triggered by pull-to-refresh
  public Observable<List<FeedItem>> loadNewestPage() { ... }

  //Loads the first page
  public Observable<List<FeedItem>> loadFirstPage() { ... }

  // loads the next page (pagination)
  public Observable<List<FeedItem>> loadNextPage() { ... }

  // loads additional products of a certain category
  public Observable<List<Product>> loadProductsOfCategory(String categoryName) { ... }
}
```

现在让我们在演示者中逐步连接点。 请注意，这里作为 Presenter 的一部分显示的一些代码应该移到真实世界应用程序中的 Interactor（为了更好的可读性，我没有这样做）。 首先，让我们开始加载初始数据：

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {
    //
    // In a real app some code here should rather be moved into an Interactor
    //
    Observable<HomeViewState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
        .flatMap(ignored -> feedLoader.loadFirstPage()
            .map(items -> new HomeViewState(items, false, null) )
            .startWith(new HomeViewState(emptyList, true, null) )
            .onErrorReturn(error -> new HomeViewState(emptyList, false, error))

    subscribeViewState(loadFirstPage, HomeView::render);
  }
}
```

到目前为止，对于如何实现第2部分中所述的“搜索屏幕”，没有太大区别。现在，我们尝试添加对 “下拉刷新” 的支持。

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {
    //
    // In a real app some code here should rather be moved into an Interactor
    //
    Observable<HomeViewState> loadFirstPage = ... ;

    Observable<HomeViewState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
        .flatMap(ignored -> feedLoader.loadNewestPage()
            .map( items -> new HomeViewState(...))
            .startWith(new HomeViewState(...))
            .onErrorReturn(error -> new HomeViewState(...)));

    Observable<HomeViewState> allIntents = Observable.merge(loadFirstPage, pullToRefresh);

    subscribeViewState(allIntents, HomeView::render);
  }
}
```

但是等一下：feedLoader.loadNewestPage（）只返回 “较新” 的项目，但是我们已经加载的项目呢？ 在 “传统” MVP 中，人们会称之为 view.addNewItems（newItems），但是我们已经在第一部分中讨论了为什么这是一个坏主意（“状态问题”）。 我们现在面临的问题是，拉到刷新取决于以前的 HomeViewState，因为我们想要“合并”以前的项目与从下拉刷新返回的项目。

女士们，先生们，请热烈欢迎 STATE REDUCER

State Reducer 是一个来自函数式编程的概念，它将前一个状态作为输入，并从之前的状态计算一个新状态，如下所示：

```java
public State reduce( State previous, Foo foo ){
  State newState;
  // ... compute the new State by taking previous state and foo into account ...
  return newState;
}
```

这个想法是这样一个 reduce（）函数结合了前一个状态和 foo 来计算一个新的状态。 Foo 通常代表我们想要应用于之前状态的变化。 在我们的例子中，我们希望通过 pull-to-refresh 来 “减少” 以前的 HomeViewState（最初由 loadFirstPageIntent 计算）。 猜猜看，RxJava 有一个名为 scan（）的操作符。 让我们重构一下我们的代码。 我们必须引入另一个类来表示局部变化（我们在之前的代码中称为 Foo 的事物被剪切掉了），它将被用来计算新的状态。

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {
    //
    // In a real app some code here should rather be moved into an Interactor
    //
    Observable<PartialState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
        .flatMap(ignored -> feedLoader.loadFirstPage()
            .map(items -> new PartialState.FirstPageData(items) )
            .startWith(new PartialState.FirstPageLoading(true) )
            .onErrorReturn(error -> new PartialState.FirstPageError(error))

    Observable<PartialState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
        .flatMap(ignored -> feedLoader.loadNewestPage()
            .map( items -> new PartialState.PullToRefreshData(items)
            .startWith(new PartialState.PullToRefreshLoading(true)))
            .onErrorReturn(error -> new PartialState.PullToRefreshError(error)));

    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh);
    HomeViewState initialState = ... ; // Show loading first page
    Observable<HomeViewState> stateObservable = allIntents.scan(initialState, this::viewStateReducer)

    subscribeViewState(stateObservable, HomeView::render);
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    ...
  }
}
```

所以我们在这里做的是，每个 Intent 现在返回一个 Observable <PartialState> 而不是直接 Observable <HomeViewState>。 然后我们用 Observable.merge（）将它们全部合并成一个可观察的流，最后应用 reducer 函数（Observable.scan（））。 基本上这意味着什么，无论何时用户启动一个意图，这个意图将产生 PartialState 对象，然后将其 “减少” 到 HomeViewState，然后最终将显示在视图（HomeView.render（HomeViewState））中。 唯一缺少的部分是状态减速器功能本身。 HomeViewState 类本身没有改变（向上滚动以查看类定义），但是我们添加了一个 Builder（Builder 模式），以便我们可以方便地创建新的 HomeViewState 对象。 所以让我们来实现状态减速器功能：

```java
private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    if (changes instanceof PartialState.FirstPageLoading)
        return previousState.toBuilder() // creates a new copy by taking the internal values of previousState
        .firstPageLoading(true) // show ProgressBar
        .firstPageError(null) // don't show error view
        .build()

    if (changes instanceof PartialState.FirstPageError)
     return previousState.builder()
         .firstPageLoading(false) // hide ProgressBar
         .firstPageError(((PartialState.FirstPageError) changes).getError()) // Show error view
         .build();

     if (changes instanceof PartialState.FirstPageLoaded)
       return previousState.builder()
           .firstPageLoading(false)
           .firstPageError(null)
           .data(((PartialState.FirstPageLoaded) changes).getData())
           .build();

     if (changes instanceof PartialState.PullToRefreshLoading)
      return previousState.builder()
            .pullToRefreshLoading(true) // Show pull to refresh indicator
            .nextPageError(null)
            .build();

    if (changes instanceof PartialState.PullToRefreshError)
      return previousState.builder()
          .pullToRefreshLoading(false) // Hide pull to refresh indicator
          .pullToRefreshError(((PartialState.PullToRefreshError) changes).getError())
          .build();

    if (changes instanceof PartialState.PullToRefreshData) {
      List<FeedItem> data = new ArrayList<>();
      data.addAll(((PullToRefreshData) changes).getData()); // insert new data on top of the list
      data.addAll(previousState.getData());
      return previousState.builder()
        .pullToRefreshLoading(false)
        .pullToRefreshError(null)
        .data(data)
        .build();
    }


   throw new IllegalStateException("Don't know how to reduce the partial state " + changes);
}
```

我知道，所有这些 instanceof 检查都不是超级漂亮，但这不是这篇博文的重点。 为什么技术博客写上面那样的 “难看” 的代码呢？ 这是因为我们想要在某个话题上提出一个观点，而不要求读者对源代码（即我们的购物车应用）有一个完整的心理模型，也不需要事先知道某些设计模式。 因此，我认为最好避免在博客文章中设计模式，这会产生更好的代码，但可能导致更难读的博客帖子。 这篇博文的重点是设置在状态减速器上。 通过使用 instanceof 检查来查看状态减速器，每个人都可以理解减速器的功能。 你应该在你的应用中使用 instanceof 检查吗？ 不，请使用设计模式或其他解决方案，如使用公共 HomeViewState computeNewState（previousState） 等方法将 PartialState 定义为接口。 一般来说，当使用 MVI 构建应用程序时，您可能会发现 Paco Estevez 的[RxSealedUnions](https://github.com/pakoito/RxSealedUnions) 很有用。

好吧，我想你知道减速器是如何工作的。让我们来实现其余的功能：分页和加载某个类别的更多项目的能力：

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {

    //
    // In a real app some code here should rather be moved to an Interactor
    //

    Observable<PartialState> loadFirstPage = ... ;
    Observable<PartialState> pullToRefresh = ... ;

    Observable<PartialState> nextPage =
      intent(HomeView::loadNextPageIntent)
          .flatMap(ignored -> feedLoader.loadNextPage()
              .map(items -> new PartialState.NextPageLoaded(items))
              .startWith(new PartialState.NextPageLoading())
              .onErrorReturn(PartialState.NexPageLoadingError::new));

      Observable<PartialState> loadMoreFromCategory =
          intent(HomeView::loadAllProductsFromCategoryIntent)
              .flatMap(categoryName -> feedLoader.loadProductsOfCategory(categoryName)
                  .map( products -> new PartialState.ProductsOfCategoryLoaded(categoryName, products))
                  .startWith(new PartialState.ProductsOfCategoryLoading(categoryName))
                  .onErrorReturn(error -> new PartialState.ProductsOfCategoryError(categoryName, error)));


    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh, nextPage, loadMoreFromCategory);
    HomeViewState initialState = ... ; // Show loading first page
    Observable<HomeViewState> stateObservable = allIntents.scan(initialState, this::viewStateReducer)

    subscribeViewState(stateObservable, HomeView::render);
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    // ... PartialState handling for First Page and pull-to-refresh as shown in previous code snipped ...

      if (changes instanceof PartialState.NextPageLoading) {
       return previousState.builder().nextPageLoading(true).nextPageError(null).build();
     }

     if (changes instanceof PartialState.NexPageLoadingError)
       return previousState.builder()
           .nextPageLoading(false)
           .nextPageError(((PartialState.NexPageLoadingError) changes).getError())
           .build();


     if (changes instanceof PartialState.NextPageLoaded) {
       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
        // Add new data add the end of the list
       data.addAll(((PartialState.NextPageLoaded) changes).getData());

       return previousState.builder().nextPageLoading(false).nextPageError(null).data(data).build();
     }

     if (changes instanceof PartialState.ProductsOfCategoryLoading) {
         int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());

         AdditionalItemsLoadable ail = (AdditionalItemsLoadable) previousState.getData().get(indexLoadMoreItem);

         AdditionalItemsLoadable itemsThatIndicatesError = ail.builder() // creates a copy of the ail item
         .loading(true).error(null).build();

         List<FeedItem> data = new ArrayList<>();
         data.addAll(previousState.getData());
         data.set(indexLoadMoreItem, itemsThatIndicatesError); // Will display a loading indicator

         return previousState.builder().data(data).build();
      }

     if (changes instanceof PartialState.ProductsOfCategoryLoadingError) {
       int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());

       AdditionalItemsLoadable ail = (AdditionalItemsLoadable) previousState.getData().get(indexLoadMoreItem);

       AdditionalItemsLoadable itemsThatIndicatesError = ail.builder().loading(false).error( ((ProductsOfCategoryLoadingError)changes).getError()).build();

       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
       data.set(indexLoadMoreItem, itemsThatIndicatesError); // Will display an error / retry button

       return previousState.builder().data(data).build();
     }

     if (changes instanceof PartialState.ProductsOfCategoryLoaded) {
       String categoryName = (ProductsOfCategoryLoaded) changes.getCategoryName();
       int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());
       int indexOfSectionHeader = findSectionHeader(categoryName, previousState.getData());

       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
       removeItems(data, indexOfSectionHeader, indexLoadMoreItem); // Removes all items of the given category

       // Adds all items of the category (includes the items previously removed)
       data.addAll(indexOfSectionHeader + 1,((ProductsOfCategoryLoaded) changes).getData());

       return previousState.builder().data(data).build();
     }

     throw new IllegalStateException("Don't know how to reduce the partial state " + changes);
  }
}
```

实现分页（加载项目的下一个“页面”）与 “拉到刷新” 非常相似，除了我们将加载的项目添加到列表的末尾而不是列表的顶部，刷新。 更有趣的是我们如何处理加载某个类别的更多项目。 那么，为了显示给定类别的加载指示器和错误/重试按钮，我们只需要在所有的 FeedItem 列表中搜索相应的 AdditionalItemsLoadable 对象。 然后我们更改该项目，以显示加载指示器或错误/重试按钮。 如果我们成功加载了某个类别的所有项目，我们搜索SectionHeader和AdditionalItemsLoadable，并用新加载的项目替换它们之间的所有项目。 而已。

**综述**

这篇博客文章的目的是向你展示一个状态减速器如何帮助我们用很少和可理解的代码来构建复杂的屏幕。 回想一下，如果没有状态减速器的话，你将如何实现 “传统” MVP 或 MVVM？ 能够使用状态减速器的关键显然是我们有一个反映状态的模型类。 因此，了解模型的实际内容非常重要，正如本博文系列的第一部分所述。 另外，只有在确定状态（或模型是精确的）来自单一的事实来源时，才能使用状态减速器。 因此，单向数据流是非常重要的。 我希望现在更有意义的是，为什么我们将第一部分和第二部分花在这些话题上，而现在你们有了这个 “阿哈” 的时刻，所有的点连在一起。 如果不是，不用担心，我也花了相当长的一段时间（还有很多练习和很多错误和重试）。

