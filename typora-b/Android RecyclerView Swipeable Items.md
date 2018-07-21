# Android RecyclerView : 可滑动项目

> 原文 (Medium)：[Android RecyclerView: Swipeable Items](https://android.jlelse.eu/android-recyclerview-swipeable-items-46a3c763498d)
>
> 作者：[Mark O'Sullivan](https://android.jlelse.eu/@markosullivan?source=post_header_lockup)

[TOC]

![](https://cdn-images-1.medium.com/max/800/1*R_lBa5M1_h1PtcH4hYK2hA.gif)

## 背景

最近我一直很难找到一个很好的解决方案来实现 RecyclerView 列表项目的 swipeable 布局，当项目被滑动时，会在下面显示一个次要的布局。在我的例子中，我希望将常用的操作按钮包含在项目下面。 

这与 Android 应用中另外两个更常见的功能不同：

- 滑动移除项 
- 拖动以重新排序列表 

如果您想要完成上面两项，我强烈建议您查看下面的博客。

[Drag and Swipe with RecyclerView Part One: Basic ItemTouchHelper Example medium.com](https://medium.com/@ipaulpro/drag-and-swipe-with-recyclerview-b9456d2b1aaf)

起初我以为这个功能可以使用 ItemTouchHelper 来实现，这个功能是 Paul Burke 在我刚刚链接的博客中使用的 。我可以让滑动工作正常，并让它停留，这样它不会从屏幕上消失，但我发现无法触发顶部布局下次要布局的点击事件。此外，每当我试图点击时，顶层都会重置它的位置。

在那次令人沮丧的经历之后，我开始寻找 GitHub 寻找一个快速简单的解决方案。

### 1. Swipe-Controller-Demo

[FanFataL/swipe-controller-demo swipe-controller-demo - This is source code from the article about creating swipe menu with RecyclerView without any… github.com](https://github.com/FanFataL/swipe-controller-demo)

这是我在测试之后找到的第一个解决方案，我努力想要添加添加 Buttons / ImageButton 等原生 Android 组件，但是在我看来它并没有那么灵活。

### 2. SwipeRevealLayout

[chthai64/SwipeRevealLayout SwipeRevealLayout - Easy, flexible and powerful Swipe Layout for Android github.com](https://github.com/chthai64/SwipeRevealLayout)

这个解决方案是最好的，但是觉得包含了太多我不需要或者不想要的功能，例如：从项目的顶部或底部滑动，恢复项目的开启/关闭状态等。它有我需要的功能，我只是不想要所有的额外功能。 如果您需要这些功能的话，我强烈建议您查看该库！

我的解决方案是从 swiperevallayout 中删除所有内容 ，只保留实现 RecyclerView 项目 swipeable 布局的相关类，在下一节中，我将列出需要采取的步骤，以便将此功能添加到您的项目中。 

## 如何实施

1. 在你的项目中添加下面的类：[SwipeRevealLayout.java](https://github.com/MarkOSullivan94/SwipeRevealLayoutExample/blob/master/app/src/main/java/me/markosullivan/swiperevealactionbuttons/SwipeRevealLayout.java)
2. 调整您的 RecyclerView ViewHolder 的布局代码，并将 SwipeRevealLayout 组件作为 RecyclerView 项目顶层和底层的容器。关于如何设置它的一个示例：[list_item_main.xml](https://github.com/MarkOSullivan94/SwipeRevealLayoutExample/blob/master/app/src/main/res/layout/list_item_main.xml) 。
3. 确保底层是 SwipeRevealLayout 作为第一个布局组件。
4. 确保底层使用 'wrap_content' 或预定义的宽度。我使用 'match_parent' 进行测试，不会出现效果。
5. 如果要在底层添加 clickable 的功能，请确保顶层有 android：clickable =“true”，否则，当你点击顶层时，对底层组件的点击仍然会触发 。
6. 选项：您可以定义要从哪边拖动。默认情况下，它将从左侧拖动，在我定义的示例项目中从右侧拖动。在指定 SwipeRevealLayout 组件的属性时，使用 'app：dragFromEdge =' 指定拖动的边缘 。

就是这样！你已经得到了一个 RecyclerView，可以滑动项目来显示次要布局，下一步是什么？那么这是你自己决定的。

如果你想测试一个演示项目，可以随意 clone 我已经上传到 GitHub 的示例：[[PR](https://github.com/MarkOSullivan94/SwipeRevealLayoutExample)]

我希望你发现这篇文章是有用的，最终能用我今天分享的东西来制作一些很酷的东西。 感谢阅读！ 

