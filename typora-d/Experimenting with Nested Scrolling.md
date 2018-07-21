# 嵌套滚动实验

> 原文 (androiddesignpatterns.com)：[Experimenting with Nested Scrolling](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html)
>
> 作者：[Alex Lockwood](https://google.com/+AlexLockwood)

[TOC]

在谷歌工作的三年里，我参与过的最酷的项目之一是 Google Expeditions，这是一个虚拟现实应用程序，允许老师带领学生在全世界进行虚拟实地旅行。 我特别喜欢在应用屏幕上工作的行程选择器，这使得用户可以在不同的虚拟现实体验之间快速切换。 

我已经有一段时间没有写安卓代码了(在过去的一年里，我花了大部分时间来构建 Android 开发工具，比如 [Shape Shifter](https://github.com/alexjlockwood/ShapeShifter) 和 [avocado](https://github.com/alexjlockwood/avocado)) ，所以有一天我挑战自己重写屏幕的一部分作为一种练习。 图1显示了 Google Expeditions 的行程选择器屏幕和我写的样本应用程序(可在 [GitHub](https://github.com/alexjlockwood/adp-nested-scrolling) 和 [Google Play](https://play.google.com/store/apps/details?id=alexjlockwood.nestedscrolling) 上查阅)。 [[VIDEO]](https://www.androiddesignpatterns.com/assets/videos/posts/2018/01/24/expeditions-sample-opt.mp4)

当我在编写代码的时候，我想起了几年前我第一次写这个屏幕的时候，我遇到了一些挑战。 在 API 21中引入了嵌套滚动 API，使一个可滚动的父视图可以包含可滚动的子视图，使我们能够创建滚动手势，材料设计正式化其[滚动技术](https://material.io/guidelines/patterns/scrolling-techniques.html#scrolling-techniques-app-bar-scrollable-regions)模式页面。 图2显示了这些 api 的一个常见用例，其中包括父版本 CoordinatorLayout 和儿童 NestedScrollView。 如果没有嵌套滚动，NestedScrollView 就独立于其周围的环境进行滚动。 然而，一旦启用，CoordinatorLayout 和 NestedScrollView 轮流截取和消费滚动，创建一个折叠的工具栏效果看起来更自然。[[Video]](https://www.androiddesignpatterns.com/assets/videos/posts/2018/01/24/cheesesquare-opt.mp4) 

嵌套的滚动 api 到底是如何工作的？ 对于初学者，你需要一个实现 NestedScrollingChild 的父视图和一个实现 NestedScrollingChild 的子视图。 在下面的图3中，NestedScrollView (NSV)是父节点，而 RecyclerView (RV) 是子版本: 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frosu70wt3j30sg0lcjsb.jpg)

假设用户试图将 RV 滚动到上面。 如果没有嵌套滚动，RV 将立即使用滚动事件，导致我们在图2中看到的不良行为。 我们真正想要的是创造一种错觉，让这两种视图作为一个整体一起滚动。 更具体地说: [1](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#footnote1)

1. 如果 RV 位于其内容的顶部，则向上滚动 RV 应导致 NSV 向上滚动。
2. 如果 NSV 不在其内容的底部，则向下滚动 RV 应导致 NSV 向下滚动。

正如你可能期望的那样，嵌套滚动 api 为 NSV 和 RV 提供了一种在滚动过程中相互交流的方式，这样每个视图都可以自信地确定谁应该使用每个滚动项。 当你考虑到当用户把手指拖到 RV 顶部时发生的事件顺序时，这一点就变得清晰了: 

1. RV 的 onTouchEvent（ACTION_MOVE）方法被调用。
2. RV 调用它自己的 dispatchNestedPreScroll ( ) 方法，通知 NSV 它即将消耗部分滚动。
3. 调用 NSV的 onNestedPreScroll  ( ) 方法，让 NSV 有机会在 RV 消耗它之前对滚动事件作出反应。
4. RV 消耗剩余的滚动（或者，如果 NSV 消耗整个事件，则不执行任何操作）。
5. RV 调用它自己的 dispatchNestedScroll  ( ) 方法，通知 NSV 它已经消耗了滚动的一部分。
6. 调用 NSV 的 onNestedScroll  ( ) 方法，使 NSV 有机会消耗尚未消耗掉的剩余滚动像素。
7. RV 从当前调用返回到 onTouchEvent（ACTION_MOVE），消耗触摸事件。[2](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#footnote2)

不幸的是，仅仅使用 NSV 和 RV 不足以获得我想要的滚动行为。 图4显示了我需要修复的两个问题。 造成这两个问题的原因是 RV 在不应该的情况下消耗滚动和投掷事件。 在左边，RV 不应该开始滚动，直到卡片达到屏幕的顶部。 在右侧，向下投掷 RV 应该能够使卡片在一个光滑的运动中折叠。[[VIDEO]](https://www.androiddesignpatterns.com/assets/videos/posts/2018/01/24/nested-scrolling-bugs1-opt.mp4)

修复这两个问题是相对简单的，因为我们知道嵌套滚动 api 是如何工作的。 我们所需要做的就是创建一个 CustomNestedScrollView 类，通过重写 onNestedPreScroll ( ) 和 onNestedPreFling ( ) 方法来自定义滚动行为: 

```java
/**
 * A NestedScrollView with our custom nested scrolling behavior.
 */
public class CustomNestedScrollView extends NestedScrollView {

  // The NestedScrollView should steal the scroll/fling events away from
  // the RecyclerView if: (1) the user is dragging their finger down and
  // the RecyclerView is scrolled to the top of its content, or (2) the
  // user is dragging their finger up and the NestedScrollView is not
  // scrolled to the bottom of its content.

  @Override
  public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    final RecyclerView rv = (RecyclerView) target;
    if ((dy < 0 && isRvScrolledToTop(rv)) || (dy > 0 && !isNsvScrolledToBottom(this))) {
      // Scroll the NestedScrollView's content and record the number of pixels consumed
      // (so that the RecyclerView will know not to perform the scroll as well).
      scrollBy(0, dy);
      consumed[1] = dy;
      return;
    }
    super.onNestedPreScroll(target, dx, dy, consumed);
  }

  @Override
  public boolean onNestedPreFling(View target, float velX, float velY) {
    final RecyclerView rv = (RecyclerView) target;
    if ((velY < 0 && isRvScrolledToTop(rv)) || (velY > 0 && !isNsvScrolledToBottom(this))) {
      // Fling the NestedScrollView's content and return true (so that the RecyclerView
      // will know not to perform the fling as well).
      fling((int) velY);
      return true;
    }
    return super.onNestedPreFling(target, velX, velY);
  }

  /**
   * Returns true iff the NestedScrollView is scrolled to the bottom of its
   * content (i.e. if the card's inner RecyclerView is completely visible).
   */
  private static boolean isNsvScrolledToBottom(NestedScrollView nsv) {
    return !nsv.canScrollVertically(1);
  }

  /**
   * Returns true iff the RecyclerView is scrolled to the top of its
   * content (i.e. if the RecyclerView's first item is completely visible).
   */
  private static boolean isRvScrolledToTop(RecyclerView rv) {
    final LinearLayoutManager lm = (LinearLayoutManager) rv.getLayoutManager();
    return lm.findFirstVisibleItemPosition() == 0
        && lm.findViewByPosition(0).getTop() == 0;
  }
}
```

我们快结束了！然而，对于那些完美主义者来说，图5显示了一个更好的修复错误。在左侧的视频中，一旦孩子达到其内容的顶部，投掷就会突然停止。我们想要的是让投掷以单一的流畅动作完成，如右侧视频所示。[[VIDEO](https://www.androiddesignpatterns.com/assets/videos/posts/2018/01/24/nested-scrolling-bugs2-opt.mp4)]

问题的症结在于，直到最近，支持库都没有提供一种方法，让嵌套滚动子转移剩下的嵌套传输速度到嵌套滚动父级。 我不会在这里赘述太多细节，因为 Chris Banes 已经写了一篇详细的[博客文章](https://chris.banes.me/2017/06/09/carry-on-scrolling/)来解释这个问题以及它应该如何被修正。[3](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#footnote3) 但是总结一下，我们需要做的就是更新我们的父和子视图，以实现新的和改进的 NestedScrollingParent2 和 NestedScrollingChild2 接口，这些接口是在支持库的 v26中特别添加来解决这个问题的。 

不幸的是，NestedScrollView 仍然实现了旧的 NestedScrollingParent 接口，所以我必须创建自己的 [nestedscrollingview2](https://github.com/alexjlockwood/adp-nested-scrolling/blob/master/app/src/main/java/alexjlockwood/nestedscrolling/NestedScrollView2.java) 类，该类实现了 NestedScrollingParent2 接口来让事情工作。4最后，无错误的 NestedScrollView 实现如下: 

```java
/**
 * A NestedScrollView that implements the new-and-improved NestedScrollingParent2
 * interface and that defines its own customized nested scrolling behavior. View
 * source code for the NestedScrollView2 class here: j.mp/NestedScrollView2
 */
public class CustomNestedScrollView2 extends NestedScrollView2 {

  @Override
  public void onNestedPreScroll(View target, int dx, int dy, int[] consumed, int type) {
    final RecyclerView rv = (RecyclerView) target;
    if ((dy < 0 && isRvScrolledToTop(rv)) || (dy > 0 && !isNsvScrolledToBottom(this))) {
      scrollBy(0, dy);
      consumed[1] = dy;
      return;
    }
    super.onNestedPreScroll(target, dx, dy, consumed, type);
  }

  // Note that we no longer need to override onNestedPreFling() here; the
  // new-and-improved nested scrolling APIs give us the nested flinging
  // behavior we want already by default!

  private static boolean isNsvScrolledToBottom(NestedScrollView nsv) {
    return !nsv.canScrollVertically(1);
  }

  private static boolean isRvScrolledToTop(RecyclerView rv) {
    final LinearLayoutManager lm = (LinearLayoutManager) rv.getLayoutManager();
    return lm.findFirstVisibleItemPosition() == 0
        && lm.findViewByPosition(0).getTop() == 0;
  }
}
```

我现在只有这些了！ 感谢阅读，如果你有任何问题，不要忘记在下面留下评论！ 

1 这篇博文使用了[框架](https://developer.android.com/reference/android/view/View.html#canScrollVertically(int))用来描述滚动方向的相同术语。 也就是说，把你的手指拖到屏幕的底部，导致视图向上滚动，并将你的手指拖向屏幕顶部，导致视图向下滚动。 [↩](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#ref1)

2 值得注意的是，嵌套的衬垫是以非常相似的方式处理的。 孩子在其 onTouchEvent (ACTION_UP)方法中检测到一个插件，并通过调用它自己的 dispatchNestedPreFling ( ) 和 dispatchNestedFling ( ) 方法来通知父方。 这个触发器调用了父方的 onNestedPreFling ( ) 和 onNestedFling ( ) 方法，并给父母一个机会来对孩子消费之前和之后的投掷做出反应。 [↩](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#ref2)

3 我建议观看 Chris Banes 的 [Droidcon 2016](https://www.youtube.com/watch?v=-bhjI_qLPdE) 演讲的下半部分，以获取关于此主题的更多信息。 [↩](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#ref3)

4 如果有足够多的人出现这个 bug，也许这在未来是不必要的 ！ :) [↩](https://www.androiddesignpatterns.com/2018/01/experimenting-with-nested-scrolling.html#ref4)