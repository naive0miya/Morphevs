# 使用 MVI 的反应性应用 - 第7部分 - TIMING (SINGLELIVEEVENT PROBLEM)

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART7 - TIMING (SINGLELIVEEVENT PROBLEM)](http://hannesdorfmann.com/android/mosby3-mvi-7)
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




在我以前的博客文章中，我们讨论了适当的状态管理的重要性，以及为什么我认为引入 [Google GitHub repo  体系结构组件中讨论的 SingleLiveEvent](https://github.com/googlesamples/android-architecture-components/issues/63)，不是一个好主意，因为它隐藏了真正的潜在问题：状态管理。 在这篇博文中，我想讨论 SingleLiveEvent 声称要解决的问题如何通过 Model-View-Intent 和适当的状态管理来解决。

说明这个问题的常见情况是如果发生错误，则显示一个 Snackbar。 Snackbar 并不意味着是持久的，而应该只是显示错误信息几秒钟，然后消失。 问题是我们如何建模这个错误状态和“消失”？

看看这个视频，以更好地理解我在说什么：[mvi timing snackbar reappears](https://www.youtube.com/watch?v=8r7UgJg6d2s)

此示例应用程序显示从 CountriesRepository 加载的国家列表。 如果我们点击一个国家，我们开始第二个活动，只显示 “细节”（只是国家的名字）。 当我们回到国家列表时，我们希望在点击一个国家之前，在屏幕上看到相同的 “状态”。 到目前为止这么好，但是如果我们触发 pull-to-refresh 和在加载数据时发生错误，最终会在屏幕上显示一个显示错误消息的 Snackbar，会发生什么情况。 正如你在上面的视频中看到的那样，每当我们回到国家列表中时，Snackbar 会再次显示，但这通常不是用户期望的行为，对吧？

问题是这个屏幕处于 “显示错误状态”。 基于 ViewModel 和 LiveData 的 Google 体系结构组件示例使用 SingleLiveEvent 来解决此问题。 这个想法是：每当 View 重新订阅（从 “细节” 屏幕返回后）到他的 ViewModel，SingleLiveEvent 确保 “错误状态” 不会再次发射。 虽然这可以防止重新出现 Snackbar，它真的解决了这个问题？

## Timing is Everything (for Snackbars)

我仍然认为这样的 “解决方法” 不是正确的方法。 我们可以做得更好吗？ 我认为适当的状态管理和单向数据流是更好的解决方案。 Model-View-Intent 是遵循这些原则的架构模式。 那么我们如何解决 MVI 中的 “ Snackbar 问题” 呢？ 首先，让我们模型状态：

```java
public class CountriesViewState {

  // True if progressbar should be displayed
  boolean loading;

  // List of countries (country names)  if loaded
  List<String> countries;

  // true if pull to refresh indicator should be displayed
  boolean pullToRefresh;

  // true if an error has occurred while pull to refresh -> Show Snackbar.
  boolean pullToRefreshError;
}
```

MVI 中的想法是 View 层获得（不可变的）CountriesViewState 并只显示那个。所以如果 pullToRefreshError 为 true，那么显示一个 Snackbar，否则不要。

```java
public class CountriesActivity extends MviActivity<CountriesView, CountriesPresenter>
    implements CountriesView {

  private Snackbar snackbar;
  private ArrayAdapter<String> adapter;

  @BindView(R.id.refreshLayout) SwipeRefreshLayout refreshLayout;
  @BindView(R.id.listView) ListView listView;
  @BindView(R.id.progressBar) ProgressBar progressBar;

   ...

  @Override public void render(CountriesViewState viewState) {
    if (viewState.isLoading()) {
      progressBar.setVisibility(View.VISIBLE);
      refreshLayout.setVisibility(View.GONE);
    } else {
      // show countries
      progressBar.setVisibility(View.GONE);
      refreshLayout.setVisibility(View.VISIBLE);
      adapter.setCountries(viewState.getCountries());
      refreshLayout.setRefreshing(viewState.isPullToRefresh());

      if (viewState.isPullToRefreshError()) {
        showSnackbar();
      } else {
        dismissSnackbar();
      }
    }
  }

  private void dismissSnackbar() {
    if (snackbar != null)
      snackbar.dismiss();
  }

  private void showSnackbar() {
    snackbar = Snackbar.make(refreshLayout, "An Error has occurred", Snackbar.LENGTH_INDEFINITE);
    snackbar.show();
  }
}
```

这里的重要一点是 Snackbar.LENGTH_INDEFINITE，这意味着 Snackbar 保持不动，直到我们解雇它为止。 所以我们不要让 android 动画 Snackbar 进出。 而且，我们不要让 android 搞乱状态，也不要让 android 为业务逻辑的状态引入一个 UI 状态。 而不是使用  Snackbar.LENGTH_SHORT 来显示 Snackbar 两秒钟，而是让业务逻辑负责将 CountriesViewState.pullToRefreshError 设置为 true 两秒钟，然后将其设置为 false。

我们如何在 RxJava 中做到这一点？我们可以使用 Observable.timer（）和 startWith（）运算符。

```java
public class CountriesPresenter extends MviBasePresenter<CountriesView, CountriesViewState> {

  private final CountriesRepositroy repositroy = new CountriesRepositroy();

  @Override protected void bindIntents() {

    Observable<RepositoryState> loadingData =
        intent(CountriesView::loadCountriesIntent).switchMap(ignored -> repositroy.loadCountries());

    Observable<RepositoryState> pullToRefreshData =
        intent(CountriesView::pullToRefreshIntent).switchMap(
            ignored -> repositroy.reload().switchMap(repoState -> {
              if (repoState instanceof PullToRefreshError) {
                // Let's show Snackbar for 2 seconds and then dismiss it
                return Observable.timer(2, TimeUnit.SECONDS)
                    .map(ignoredTime -> new ShowCountries()) // Show just the list
                    .startWith(repoState); // repoState == PullToRefreshError
              } else {
                return Observable.just(repoState);
              }
            }));

    // Show Loading as inital state
    CountriesViewState initialState = CountriesViewState.showLoadingState();

    Observable<CountriesViewState> viewState = Observable.merge(loadingData, pullToRefreshData)
        .scan(initialState, (oldState, repoState) -> repoState.reduce(oldState))

    subscribeViewState(viewState, CountriesView::render);
  }
}
```

CountriesRepositroy 有一个方法 reload（），它返回一个 Observable <RepoState>。 RepoState（在这个 MVI 系列的前一篇文章中被命名为 PartialViewState）仅仅是一个 POJO 类，它指示该存储库是否正在提取数据，已成功加载数据或发生错误（[源代码](https://github.com/sockeqwe/mvi-timing/blob/41095bdecf32c149c1d81b3d773937e7c08d4bdf/app/src/main/java/com/hannesdorfmann/mvisnackbar/RepositoryState.java)）。 然后我们使用 [State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3) 来计算视图状态（scan（）运算符）。 如果您已经阅读了我的 MVI 系列的以前的博客文章，这应该是熟悉的。 “新” 事物是：

```java
repositroy.reload().switchMap(repoState -> {
  if (repoState instanceof PullToRefreshError) {
    // Let's show Snackbar for 2 seconds and then dismiss it
    return Observable.timer(2, TimeUnit.SECONDS)
        .map(ignoredTime -> new ShowCountries()) // Show just the list
        .startWith(repoState); // repoState == PullToRefreshError
  } else {
    return Observable.just(repoState);
  }
```

这段代码执行以下操作：如果遇到错误（repoState instanceof PullToRefreshError），则会发出此错误状态（PullToRefreshError），然后导致状态缩减器设置 CountriesViewState.pullToRefreshError = true。 2秒后 Observable.timer（）发出 ShowCountries 状态，导致状态缩减器设置 CountriesViewState.pullToRefreshError = false。

这就是我们如何在 MVI 中展示和隐藏 Snackbar。[mvi timing correct state](https://www.youtube.com/watch?v=ykDnwZYY9tk)

请注意，这不是像 SingleLiveEvent 这样的解决方法。 这是合适的状态管理，视图只是显示或“渲染”给定的状态。 因此，一旦我们的应用程序的用户从细节屏幕回到国家列表，他不会看到 Snackbar 了，因为状态已经改变了 CountriesViewState.pullToRefreshError = false。 因此 Snackbar 不显示。

## User dismisses Snackbar

如果我们想允许用户用轻扫手势解雇 Snackbar，该怎么办？ 那么，这非常简单。 解雇 Snackbar 是另一个改变状态的意图。 为了把这个放到现有的代码中，我们只需要确保定时器或者滑动来解散 intent 就设置 CountriesViewState.pullToRefreshError = false。 我们唯一要记住的是，如果在定时器之前轻扫以消除意图触发器，我们必须取消定时器。 这听起来很复杂，但实际上这并不是, 感谢 RxJava 伟大的 API 和操作符：

```java
Observable<Long> dismissPullToRefreshErrorIntent = intent(CountriesView::dismissPullToRefreshErrorIntent)

...

repositroy.reload().switchMap(repoState -> {
  if (repoState instanceof PullToRefreshError) {
    // Let's show Snackbar for 2 seconds and then dismiss it
    return Observable.timer(2, TimeUnit.SECONDS)
        .mergeWith(dismissPullToRefreshErrorIntent) // merge timer and dismiss intent
        .take(1) // Only take the one who triggers first (dismiss intent or timer)
        .map(ignoredTime -> new ShowCountries()) // Show just the list
        .startWith(repoState); // repoState == PullToRefreshError
  } else {
    return Observable.just(repoState);
  }
```

[mvi timing swipe dismiss](https://www.youtube.com/watch?v=NYEZTXirGuA)

与 mergeWith（）我们结合计时器和解散意图到一个观察，然后采取（1）我们采取第一个发射。 如果在定时器之前刷卡消除触发器，则取（1）取消定时器，反之亦然：如果定时器先触发，则不对解除意图作出反应。

## 结束

所以让我们试着搞砸我们的用户界面。让我们拉一下刷新，关闭 snackbar ，也让计时器做的事情：[mvi timing trying messup state](https://www.youtube.com/watch?v=UAiT2LSl6ik)

正如你在上面的视频中看到的：无论我们多么努力，View都会正确显示UI小部件，因为单向的数据流和由业务逻辑驱动的状态（View 层无状态，View 从底层获取状态， 显示它）。 例如：我们从来没有看到拉同时刷新指示器和 Snackbar（除了一个小的重叠，在 Snackbar 退出动画时）。

当然，这个 Snackbar 例子是非常简单的，但是我认为它证明了像 Model-View-Intent 这样的体系结构模式的威力，这些模式使得状态管理非常重要。 不难想象，这种模式对于更复杂的屏幕和用例的效果如何。

这个演示程序的源代码可以在 [on Github](https://github.com/sockeqwe/mvi-timing) 上找到。



