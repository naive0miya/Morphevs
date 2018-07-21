# Fragment 过渡 和窗口 Insets

> 原文 (Medium)：[Windows Insets + Fragment Transitions](https://medium.com/google-developers/windows-insets-fragment-transitions-9024b239a436)
>
> 作者：[Chris Banes](https://medium.com/@chrisbanes?source=post_header_lockup)

[TOC]

这篇文章是我正在写的关于片段过渡的小系列文章中的第二篇 。第一篇是下面提供的 ，它设置了如何使片段过渡工作。 

[Fragment Transitions](https://medium.com/google-developers/fragment-transitions-ea2726c3f36f)

> 在我进一步探讨之前，我会假设你知道 window insets 是什么以及它们是如何被调度的。如果你不了解，我建议你看这个演讲（是的，它来自我）。[Becoming a master window fitter](https://chris.banes.me/talks/2017/becoming-a-master-window-fitter-lon/)

我要坦白一件事。 当我在这个系列的第一篇博客中工作的时候，我在视频中作弊了。 事实上，我用 window insets 来解决一个问题，这意味着我实际上最终得到了以下结果: 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frow3bfqicg30a00hsu0y.gif)

完全不是我在第一篇文章中展示的内容。我不想让第一篇文章过于复杂化，所以决定单独写这篇文章。无论如何，你可以看到当添加过渡时我们突然失去了所有状态栏的处理，视图被推到状态栏的后面。

## 问题在于

这两个片段背后都大量使用了 window insets 在系统栏后面绘制。片段 A 使用 [CoordinatorLayout](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.html) 和 [AppBarLayout](https://developer.android.com/reference/android/support/design/widget/AppBarLayout.html)，而片段 B 使用自定义 window inset 处理（使用一个 [OnApplyWindowInsetsListener](https://developer.android.com/reference/android/support/v4/view/OnApplyWindowInsetsListener.html)）。无论执行情况如何，过渡都会对两者产生影响 。

那么，为什么会发生这种情况？当你使用片段的过渡时，退出（片段 A）和进入（片段 B）内容视图的实际情况如下：

1. 过渡 commit( )。
2. 因为我们使用的是片段 A 的退出过渡，所以视图 A 保持原位，并在它上运行过渡 。
3. 视图 B 被添加到容器视图，并立即设置为不可见。
4. 片段 B 的进入和'共享元素进入'过渡开始。
5. 视图 B 设置为可见。
6. 当片段 A 的退出过渡完成后，视图 A 将从容器视图中移除。

这些听起来都不错，那么为什么它会突然影响到 window insets 处理呢？ 这是因为在过渡过程中，两个片段的视图都出现在容器中。

这听起来完全没问题，不是吗？在我的场景中，两个片段的视图都想要处理和使用 window insets，因为他们都期望在屏幕上是唯一的"主要"视图。不过，只有一个视图会接收 window insets：第一个孩子。这涉及 ViewGroup 如何[分发 window insets](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/view/ViewGroup.java#6928)，通过迭代它的孩子，直到它们中的一个消耗 insets。如果第一个孩子（这里是片段 A）消耗 insets，任何后续的孩子（这里是片段 B）都不会得到它们，我们最终会陷入这种情况。

让我们再次介绍一下，但是这一次添加了分发 window insets 时间：

1. 过渡 commit( )。
2. 因为我们使用的是一个退出过渡，视图 A 保持在原位并且在它上运行过渡。
3. 视图 B 被添加到容器视图，并立即设置为不可见。
4. window insets 分发。我们希望 View B（子级1）获得它们，而不是 View A（子级0）再次获取它们。
5. 片段 B 的进入和'共享元素进入'过渡开始。
6. 视图 B 设置为可见。
7. 当片段 A 的退出过渡完成后，视图 A 将从容器视图中移除。

## 解决方法

修复实际上是相对简单的 ：我们只需确保两个视图都接收 window insets。

我的方法是在容器视图中添加一个 [OnApplyWindowInsetsListener](https://developer.android.com/reference/android/support/v4/view/OnApplyWindowInsetsListener.html)（在此例中的宿主活动），它将手动调度任何 insets 到所有的孩子，而不仅仅是直到一个消耗这些 insets 。

```kotlin
fragment_container.setOnApplyWindowInsetsListener { view, insets ->
  var consumed = false

  (view as ViewGroup).forEach { child ->
    // Dispatch the insets to the child
    val childResult = child.dispatchApplyWindowInsets(insets)
    // If the child consumed the insets, record it
    if (childResult.isConsumed) {
      consumed = true
    }
  }

  // If any of the children consumed the insets, return
  // an appropriate value
  if (consumed) insets.consumeSystemWindowInsets() else insets
}
```

在我们应用这个方法之后，两个片段都接收到了 window insets ，我们得到了我在第一个帖子中所显示的结果: 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frow5z36ccg30a00hsnpd.gif)



## 确保请求

有一件小事，我差点忘了写。如果你正在处理片段中的 window insets ，隐式（通过使用 AppBarLayout 等）或显式地处理，你需要确保你请求一些 insets。使用 [requestApplyInsets ( )](https://developer.android.com/reference/android/support/v4/view/ViewCompat.html#requestApplyInsets%28android.view.View%29) 很容易做到这一点：

```kotlin
override fun onViewCreated(view: View, icicle: Bundle) {
  super.onViewCreated(view, savedInstanceState)
  // yadda, yadda
  ViewCompat.requestApplyInsets(view)
}
```

您必须这样做， 因为如果聚合系统用于整个视图层次结构的可见性值发生变化时，窗口才会自动发送嵌套。因为有时候你的两个片段提供了完全相同的值，所以聚合值不会改变，因此系统将忽略"更改"。 

