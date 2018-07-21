# 内容过渡进阶(第二部分)

> 原文 (androiddesignpatterns.com)：[Content Transitions In-Depth (part 2)](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)
>
> 作者 ：[Alex Lockwood](https://plus.google.com/+AlexLockwood?prsrc=5)

本文将对内容过渡及其在活动和片段过渡 API 中的角色进行深入分析。 这是我将就这个主题撰写的一系列文章中的第二篇:

- **Part 1:** [Getting Started with Activity & Fragment Transitions](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
- **Part 2:** [Content Transitions In-Depth](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)
- **Part 3a:** [Shared Element Transitions In-Depth](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html)
- **Part 3b:** [Postponed Shared Element Transitions](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html)
- **Part 3c:** Implementing Shared Element Callbacks (coming soon!)
- **Part 4:** Activity & Fragment Transition Examples (coming soon!)

在写第4部分之前，这里有一个[示例应用程序](https://github.com/alexjlockwood/activity-transitions)演示一些高级活动过渡。

首先，我们总结了我们在第1部分中学到的内容过渡，并说明如何利用它们在 Android Lollipop 中实现平滑、无缝的动画。

### 什么是内容过渡？

内容转换决定了非共享视图ーー所谓的过渡视图ーー在活动或片段过渡过程中如何进入或退出场景。由于谷歌新的 Material Design 语言，内容过渡允许我们协调每个活动 / 片段的入口和出口，使得屏幕之间的切换变得顺利和轻松。 从 Android Lollipop 开始，内容过渡可以通过调用下面的[窗口](http://developer.android.com/reference/android/view/Window.html)和[片段](http://developer.android.com/reference/android/app/Fragment.html)方法来设置内容过渡:

- setExitTransition ( ) - 当 A 开始 B 时，A 的退出过渡动画 A 的过渡视图退出场景。
- setEnterTransition ( ) - 当 A 开始 B 时，B 的进入过渡动画 B 的过渡视图进入场景。
- setReturnTransition ( ) - 当 B 返回到 A 时，B 的返回过渡动画 B 的过渡视图返回场景。
- setReenterTransition ( ) - 当 B 返回到 A 时，A 的重新进入过渡动画 A 的过渡视图重新进入场景。

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2014/12/15/games-opt.mp4" frameborder="0" allowfullscreen></iframe>

例如，视频2.1展示了内容过渡在 Google Play 游戏应用程序中如何用来实现活动间的平滑动画。当第二个活动开始时，它的进入内容过渡会从屏幕的底部边缘缓慢地将用户的头像视图移动到场景中。 当按下后退按钮时，第二个活动的返回内容过渡将视图层次划分为两个，并在屏幕的顶部和底部的每半部分进行动画。

到目前为止，我们对于内容过渡的分析只触及了表面，还有几个重要的问题没有解决。 内容过渡是如何在幕后触发的？ 可以使用哪些类型的过渡对象？ 框架如何确定过渡视图集？ 在内容过渡过程中，ViewGroup 及其子项是否可以作为单个实体一起动画？ 在接下来的几节中，我们将逐一解决这些问题。

### 内容过渡的幕后

从上一篇文章中回顾过渡有两个主要责任: 捕捉目标视图的开始和结束状态，并创建一个 Animator，它将激活两个状态之间的视图。 内容过渡没有什么不同: 在创建内容过渡的动画之前，框架必须通过改变每个过渡视图的可见性来提供它所需要的状态信息。 更具体地说，当活动 a 开始活动 b 时，会发生以下事件序列: [1](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#footnote1)

1. 活动 A 调用 startActivity ( )。

   a. 该框架遍历 A 的视图层次结构，在 A 的退出过渡运行时，确定将要退出场景的过渡视图集。

   b. A 的退出过渡捕获了 A 中过渡视图的开始状态。

   c. 该框架将 A 中的所有过渡视图设置为 INVISIBLE。

   d. 在下一个显示帧，A 的退出过渡捕捉 A 中过渡视图的结束状态。

   e. A 的退出过渡会比较每个过渡视图的开始和结束状态，并根据这些差异创建一个Animator。 Animator 运行并且过渡视图退出场景。

2. 活动 B 启动。

   a. 该框架遍历 B 的视图层次结构，在 B 的进入过渡运行时，确定将要进入场景的过渡视图集。 过渡视图最初设置为 INVISIBLE。

   b. B 的进入过渡捕捉 B 中过渡视图的开始状态。

   c. 该框架将 B 中的所有过渡视图设置为 VISIBLE。

   d. 在下一个显示帧，B 的进入过渡捕捉 B 中过渡视图的结束状态。

   e. B 的进入过渡比较每个过渡视图的开始和结束状态，并根据差异创建一个 Animator。 Animator 运行并且过渡视图进入场景。

通过切换 INVISIBLE 和 VISIBLE 之间的每个过渡视图的可见性，框架可确保为内容过渡提供创建所需动画所需的状态信息。 显然，所有内容的过渡对象都必须至少能够捕获并记录每个过渡视图在其开始和结束状态下的可见性。 幸运的是，抽象的 Visibility 类已经为你完成了这项工作：Visibility 的子类只需要实现 onAppear ( ) 和onDisappear ( ) 工厂方法，在这两个方法中，他们必须创建并返回一个 Animator，这个 Animator 可以让视图动画进出场景。 从 API 21开始，有三个具体的可见性实现 - 淡入淡出，幻灯片和爆炸 - 所有这些都可以用来创建活动和片段内容过渡。 如果有必要，自定义可见性类也可以实现; 这样做将在未来的博客文章中涵盖。

### 过渡视图和过渡组

到目前为止，我们假设内容过渡是在一组称为过渡视图的非共享视图上运行的。 在本节中，我们将讨论框架如何确定这一组视图，以及如何使用过渡组进一步定制。

在转换开始之前，框架通过对活动窗口(或片段的)整个视图层次结构执行递归搜索来构造一组过渡视图。 这个搜索通过调用重写的递归 ViewGroup#captureTransitioningViews 方法来搜索层次结构的根视图，该方法的[源代码](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/java/android/view/ViewGroup.java#L6243-L6258)如下:

```Java
/** @hide */
@Override
public void captureTransitioningViews(List<View> transitioningViews) {
    if (getVisibility() != View.VISIBLE) {
        return;
    }
    if (isTransitionGroup()) {
        transitioningViews.add(this);
    } else {
        int count = getChildCount();
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            child.captureTransitioningViews(transitioningViews);
        }
    }
}
```

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2014/12/15/webview-opt.mp4" frameborder="0" allowfullscreen></iframe>

递归是相对简单的: 框架遍历树的每个层次，直到它找到一个可见的[叶子视图](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/java/android/view/View.java#L19351-L19362)或者一个过渡组。 在活动 / 片段过渡过程中，过渡组基本上允许我们将整个 ViewGroups 作为单个实体动画。 如果 ViewGroup 的 [isTransitionGroup ( )](https://developer.android.com/reference/android/view/ViewGroup.html#isTransitionGroup()) [2](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#footnote2) 方法返回 true，那么它和它的所有子视图将作为一个整体进行动画。 否则，递归将继续，并且 ViewGroup 的过渡子视图将在动画期间独立处理。搜索的最终结果是将由内容过渡动画的完整过渡视图集。[3](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#footnote3)

上面的视频2.1中可以看到一个说明正在进行的过渡组的实例。在进入过渡过程中，用户的头像会独立于其他人滑动到屏幕，而在返回过渡期间，包含用户头像的父视图组被设置为一个动画。Google Play 游戏应用程序可能会使用一个过渡组来达到这个效果，使得当用户返回到之前的活动时，看起来当前场景会分裂为一半。

有时也需要使用过渡组来修复活动 / 片段过渡中的神秘错误。 例如，考虑视频2.2中的示例应用程序: 调用活动显示 Radiohead 唱片封面的网格，被调用的活动显示背景标题图像，共享元素唱片封面和 WebView。 这个应用程序使用了一个类似于 Google Play 游戏应用的返回过渡，将顶部背景图片和底部的 WebView 滑动到屏幕的顶部和底部。 然而，正如你在视频中看到的那样，出现故障并且 WebView 无法平滑地滑出屏幕。

那么到底出了什么问题呢？ 这个问题的根源在于 WebView 是一个 ViewGroup，因此默认情况下未被选择为过渡视图。因此，当返回内容过渡运行时，WebView 将被完全忽略，并在过渡结束时突然删除之前将继续绘制在屏幕上。 幸运的是，在返回过渡开始之前，我们可以通过调用 webView.setTransitionGroup 	(true)  来轻松地解决这个问题。

### 结论

总的来说，这篇文章提出了三个重要的观点:

1. 一个内容过渡决定了一个活动或片段的非共享视图ーー所谓的过渡视图ーー在活动或片段过渡过程中进入或退出场景。
2. 内容过渡是通过对过渡视图的可见性所做的更改触发的，因此应该几乎总是扩展抽象可见性类。
3. 过渡组使我们能够在内容过渡期间将整个 ViewGroup 作为单个实体进行动画。

一如既往，感谢阅读！ 如果你有任何问题，请随时留下评论，如果你觉得有帮助的话，不要忘记 + 1和 / 或分享这篇博客文章！

------

1 活动和片段的返回 / 重新进入过渡过程中也发生了类似的事件。[↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#ref1)

2 注意，如果 ViewGroup 默认具有非空的可绘制背景和 / 或非空过渡名称(如方法[文档](https://developer.android.com/reference/android/view/ViewGroup.html#isTransitionGroup())中所述) ，则 isTransitionGroup ( ) 将返回 true。 [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#ref2)

3 注意，在运行过渡时，还将考虑到在内容过渡对象中显式[添加](https://developer.android.com/reference/android/transition/Transition.html#addTarget(android.view.View))或[排除](https://developer.android.com/reference/android/transition/Transition.html#excludeTarget(android.view.View,%20boolean))的任何视图。 [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html#ref3)

