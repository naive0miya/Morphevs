# 使用 MVI 的反应性应用 - 第4部分 - INDEPENDENT UI COMPONENTS

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART4 - INDEPENDENT UI COMPONENTS](http://hannesdorfmann.com/android/mosby3-mvi-4)
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


在这篇博文中，我们将讨论如何构建独立的UI组件，并阐明为什么亲子关系在我看来是一种代码异味。 此外，我们将讨论为什么我认为这种关系是不必要的。

演示者（或视图模型）如何与建模设计模式（例如模型 - 视图 - 意图，模型 - 视图 - 演示者或模型 - 视图 - 视图模型）不时地产生一个问题？ 甚至更具体：“小孩演示者” 如何与其 “父母演示者” 沟通？

从我的观点来看，这种亲子关系是一种代码异味，因为它们引入了父代和子代之间的直接耦合，这导致代码难以阅读，难以维护，在需求变化影响很多组件的情况下 因此在大型系统中几乎是不可能完成的任务），最后但并非最不重要的是引入了很难预测甚至更难复制和调试的共享状态。

到目前为止还好，但不知何故信息必须从演示者 A 传递给演示者 B：演示者如何与其他演示者进行通信？ 他们不是！ 演示者要告诉其他演示者什么？ 事件X发生了？ 演示者不必彼此交谈，他们只是观察相同的模型（或者精确的业务逻辑的相同部分）。 这就是他们如何得到有关更改的通知：从底层。

![](http://hannesdorfmann.com/images/mvi-mosby3/mvp-business-logic.png)

无论何时发生事件X（即用户单击 View 1 中的按钮），Presenter 都会让信息陷入业务逻辑。 由于其他演示者正在观察相同的业务逻辑，所以业务逻辑通知他们某些事情已经发生变化（模型已经更新）。

![](http://hannesdorfmann.com/images/mvi-mosby3/mvp-business-logic2.png)

我们已经在第一部分讨论了这个原理（单向数据流）的重要性。

让我们来实现一个真实世界的例子：在我们的购物应用程序中，我们可以把物品放入购物篮。此外，还有一个屏幕，我们可以看到我们的篮子的内容，我们可以一次选择和删除多个项目。

[Delete Shopping Cart](https://www.youtube.com/watch?v=ZvnceMj8NoY)

如果我们能够将这个大屏幕分割成多个更小，独立和可重用的 UI 组件，那不是很酷吗？ 让我们说一个工具栏，显示选择的项目数，和一个 RecyclerView 实际上显示购物篮中的项目列表。

```xml
<LinearLayout>
  <com.hannesdorfmann.SelectedCountToolbar
      android:id="@+id/selectedCountToolbar"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      />

  <com.hannesdorfmann.ShoppingBasketRecyclerView
      android:id="@+id/shoppingBasketRecyclerView"
      android:layout_width="match_parent"
      android:layout_height="0dp"
      android:layout_weight="1"
      />
</LinearLayout>
```

但是这些组件如何相互沟通呢？ 很显然，每个组件都有自己的 Presenter：SelectedCountPresenter 和 ShoppingBasketPresenter。 这是一个亲子关系吗？ 不，两者都只是观察相同的模型（从相同的业务逻辑更新）：

![](http://hannesdorfmann.com/images/mvi-mosby3/shoppingcart-businesslogic.png)

```java
public class SelectedCountPresenter
    extends MviBasePresenter<SelectedCountView, Integer> {

  private ShoppingCart shoppingCart;

  public SelectedCountPresenter(ShoppingCart shoppingCart) {
    this.shoppingCart = shoppingCart;
  }

  @Override protected void bindIntents() {
    subscribeViewState(shoppingCart.getSelectedItemsObservable(), SelectedCountView::render);
  }
}


class SelectedCountToolbar extends Toolbar implements SelectedCountView {

  ...

  @Override public void render(int selectedCount) {
   if (selectedCount == 0) {
     setVisibility(View.VISIBLE);
   } else {
       setVisibility(View.INVISIBLE);
   }
 }
}
```

ShoppingBasketRecyclerView 的代码看起来几乎相同，因此我们跳过这里。 但是，如果我们仔细查看 SelectedCountPresenter，则会注意到此 Presenter 与 ShoppingCart 耦合。 我们也想在我们的应用程序的其他屏幕上使用 UI 组件。 为了使该组件可重用，我们必须移除这个依赖关系，这实际上是一个简单的重构：演示者通过构造函数获取 Observable <Integer> as Model 而不是 ShoppingCart：

```java
public class SelectedCountPresenter
    extends MviBasePresenter<SelectedCountView, Integer> {

  private Observable<Integer> selectedCountObservable;

  public SelectedCountPresenter(Observable<Integer> selectedCountObservable) {
    this.selectedCountObservable = selectedCountObservable;
  }

  @Override protected void bindIntents() {
    subscribeViewState(selectedCountObservable, SelectedCountToolbarView::render);
  }
}
```

等等，我们能够使用 SelectedCountToolbar 组件，每当我们必须显示当前选择的项目的数量。 这可以是 ShoppingCart 中的项目数量，但是此UI组件也可以在应用程序中使用完全不同的上下文和屏幕。 而且，这个 UI 组件可以放在一个独立的库中，并在另一个应用程序中使用，比如照片应用程序来显示所选照片的数量。

```java
Observable<Integer> selectedCount = photoManager.getPhotos()
    .map(photos -> {
       int selected = 0;
       for (Photo item : photos) {
         if (item.isSelected()) selected++;
       }
       return selected;
    });

return new SelectedCountToolbarPresnter(selectedCount);
```

**综述**

这篇博客文章的目的是为了说明通常不需要亲子关系，只要观察业务逻辑的相同部分即可避免。 没有事件总线，没有来自父 Activity / Fragment 的 findViewById（），不需要 presenter.getParentPresenter（）或其他解决方法。 只是观察者模式。 在基本实现了观察者模式的 RxJava 的帮助下，我们能够轻松地构建这样的反应式 UI 组件。

**额外的想法**

MVI 与 MVP 或 MVVM 相比，我们强制（以积极的方式）业务逻辑驱动某个组件的状态。因此，具有更多 MVI 经验的开发者可以得出以下结论：

> 如果这种视图状态是另一个组件的模型呢？如果一个组件的视图状态更改是另一个组件的意图呢？

例：

```java
Observable<Integer> selectedItemCountObservable =
        shoppingBasketPresenter
           .getViewStateObservable()
           .map(items -> {
              int selected = 0;
              for (ShoppingCartItem item : items) {
                if (item.isSelected()) selected++;
              }
              return selected;
            });

Observable<Boolean> doSomethingBecauseOtherComponentReadyIntent =
        shoppingBasketPresenter
          .getViewStateObservable()
          .filter(state -> state.isShowingData())
          .map(state -> true);

return new SelectedCountToolbarPresenter(
              selectedItemCountObservable,
              doSomethingBecauseOtherComponentReadyIntent);
```

乍一看这似乎是一个有效的方法，但不是一个亲子关系的变种？ 当然，这不是一种传统的分层亲子关系，它更像是一个洋葱（内在的一个给外部的一个状态）似乎更好，但是仍然是一个紧密耦合的关系，不是吗？ 我还没有下定决心，但我认为现在避免这种类似洋葱的关系更好。 如果您有不同的意见，请在下面留言。 我很想听听你的想法。

