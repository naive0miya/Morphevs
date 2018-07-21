# Fragment 过渡

> 原文 (Medium)：[Fragment Transitions](https://medium.com/google-developers/fragment-transitions-ea2726c3f36f)
>
> 作者：[Chris Banes](https://medium.com/@chrisbanes?source=post_header_lockup)

[TOC]

这是我在这个小系列文章中的第一篇文章，我在这里探索如何使 transitions 与 fragments 很好地协同工作。这篇文章是关于让他们运行的。

几个月前，我在一个名为 Tivi 的应用中展示了一个网格到网格的转换。 

然而，在我设法达到这个目的之前，我一直努力让任何过渡动画工作。在应用程序中，当扩展数据集和底部导航事件发生时， fragments 更改是非边界变化的。只有数据类型发生重大变化时（例如，进入某个项目的细节页面），我才会开始一项 activity。

你在上面看到的变化是一个 fragments 替换。片段 A 是概览屏幕，然后当用户点击“更多”按钮时，它会被片段 B 替换，它包含了所有的网格项目。 

我现有的片段事务代码大致如下：

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.home_content, fragment)
    .addToBackStack(tag)
    .commit()
```

于是我查找了 transition   API 并添加了一些共享元素。在这种情况下，共享的元素是包含海报的方形图像 : 

```kotlin
supportFragmentManager.beginTransaction() 
    .replace(R.id.home_content, fragment)
    .addToBackStack(tag)
    .apply {
      for (view in sharedElementViews) {
        addSharedElement(view)
      }
    }
    .commit()
```

然后我在进入的片段（片段 B）上设置一个共享元素过渡。在这种情况下，我使用了一个  [ChangedBounds](https://developer.android.com/reference/android/support/transition/ChangeBounds.html) 启动，因为视图只是从 A➡️B 点移动并改变大小。

```kotlin
class GridFragment : Fragment() {
  override fun onCreate(savedInstanceState: Bundle?) { 
    super.onCreate(savedInstanceState)
    sharedElementEnterTransition = ChangeBounds()
  }
}
```

在这一点上，我乐观地期望它能够正常工作，但我实际上得到了这个：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frow23syuxg30a00hs4qp.gif)

所以它在进入时根本不起作用，但退出的过渡工作得很好。这只是一个开始。

## 推迟

在这一点上，我想起了我的同事 [Nick Butcher](https://twitter.com/crafty) 过去在为 [plaid](https://github.com/nickbutcher/plaid) 过渡时向我提及的一些事情。特别是在你的视图准备好之前不得不推迟过渡（布局，数据加载等）。

加入推迟并开始我的片段：

```kotlin
override fun onViewCreated(view: View, icicle: Bundle?) {
  // View is created so postpone the transition for now
  postponeEnterTransition()

  viewModel.liveList.observe(this) {
    controller.setList(it)
    // Data is loaded so we’re ready to start our transition
    startPostponedEnterTransition()
  }
}
```

我们必须为进入和退出片段执行此操作，以便按预期的方式进入（点击）和退出（后退按钮）。

但仍然没有进入的过渡。

## 重新排序

我在这里作弊了，联系了安卓团队的 [George Mount](https://twitter.com/georgemount1) 先生并询问 Activity-Fragment-Transitions。他指出了我需要做的事情：重新排序。

事实证明，你必须启用重新排序的片段事务，才能使推迟片段的过渡生效。幸运的是，这很容易做到（但很容易忘记）：

```kotlin
supportFragmentManager.beginTransaction() 
  .setReorderingAllowed(true)
  .replace(R.id.home_content, fragment)
  .addToBackStack(tag)
  .apply { 
    for (view in sharedElementViews) {
      addSharedElement(view)
    }
  }
  .commit()
```

在这之后，进入的过渡偶尔会起作用，但大多数时候我只是得到一个交叉淡出。不过现在至少有一部分时间是在运行过渡。

## 等待你的父母

由于我的过渡至少有时会运行 ，我开始调试。我推断过渡运行的时间，视图已经布置（isLaidOut == true）并且已经绘制完毕。

所以这里的最后一个难题是... 等待。

如果你回顾一下我们的推迟调用，我们实际上是在数据加载后立即开始推迟过渡。在我们开始过渡之前，我们需要给这些视图一个更新的机会，更重要的是，过渡应该在布局和绘制完成之后进行。

所以我们的新推迟调用变成：

```kotlin
override fun onViewCreated(view: View, icicle: Bundle?) {
  // View is created so postpone the transition 
  postponeEnterTransition()
  viewModel.liveList.observe(this) {
    controller.setList(it)
    // Data is loaded so lets wait for our parent to be drawn 
    (view?.parent as? ViewGroup)?.doOnPreDraw {
      // Parent has been drawn. Start transitioning! 
      startPostponedEnterTransition()
    }
  }
}
```

你可能想知道为什么我们在父视图上设置 OnPreDrawListener 而不是视图本身。这是因为你的视图实际上可能不会被绘制，因此侦听器将永远不会启动，事务将永远被延迟。为了解决这个问题，我们将侦听器设置在父级，这可能会被绘制出来。 

瞧，我们有一个工作的过渡：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frow1l4pemg30a00hsnpd.gif)

您可能想知道为什么这看起来不像我上面提到的推文。我会在后面的文章中介绍。 ✨⚡。下一篇文章将着眼于获取片段过渡使用的 window insets 。

如果你想知道 doOnPreDraw 方法来自哪里，它来自 [android-ktx](https://github.com/android/android-ktx)。如果您在应用程序中使用 Kotlin，请务必检查一下。

