# 共享元素过渡进阶(第 3a 部分)

> 原文 (androiddesignpatterns.com)：[Shared Element Transitions In-Depth (part 3a)](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html)
>
> 作者 ：[Alex Lockwood](https://plus.google.com/+AlexLockwood?prsrc=5)

这篇文章将深入分析共享元素的过渡及其在活动和片段过渡 API 中的角色。 这是我将就这个主题撰写的一系列文章中的第三个:

- **Part 1:** [Getting Started with Activity & Fragment Transitions](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
- **Part 2:** [Content Transitions In-Depth](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)
- **Part 3a:** [Shared Element Transitions In-Depth](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html)
- **Part 3b:** [Postponed Shared Element Transitions](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html)
- **Part 3c:** Implementing Shared Element Callbacks (coming soon!)
- **Part 4:** Activity & Fragment Transition Examples (coming soon!)

在写第4部分之前，这里有一个[示例应用程序](https://github.com/alexjlockwood/activity-transitions)演示一些高级活动过渡。

本系列的第3部分将分为三个部分: 第 3a 部分将侧重于共享元素如何在幕后运行，第 3b 部分和第 3c 部分将更多地侧重于 API 的具体实现细节，例如推迟某些共享元素过渡和实现 SharedElementCallbacks 的重要性。

首先，我们总结了我们在第1部分中所学到的共享元素过渡，并说明如何利用它们在 Android Lollipop 中实现平滑、无缝的动画。

### 什么是共享元素过渡？

共享元素过渡决定了共享元素视图(也称为英雄视图)是如何在场景过渡期间从一个活动 / 片段动画到另一个活动 / 片段。 共享元素由所谓的活动 / 片段的进入和返回共享元素过渡来激活共享元素 [1](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#footnote1)，每个元素都可以使用下面的窗口和片段方法来指定:

- setSharedElementEnterTransition ( ) - B 的进入共享元素过渡将动画共享元素视图从其在 A 中的起始位置移动到其在 B 中的最终位置。
- setSharedElementReturnTransition ( ) - B 的返回共享元素过渡将动画共享元素视图从他们在 B 中的起始位置移动到他们在 A 中的最终位置。

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2015/01/12/music-opt.mp4" frameborder="0" allowfullscreen></iframe>

视频3.1展示了在 Google Play Music 应用程序中共享元素过渡是如何使用的。 这个过渡包括两个共享的元素: ImageView 和它的父 CardView。 在过渡期间，ImageView 无缝地在两个活动之间进行动画，而 CardView 逐渐扩展 / 合同到位。

尽管第一部分只是简单介绍了这个主题，这篇博文的目的是对共享元素过渡进行更深入的分析。 共享元素过渡是如何在幕后触发的？ 可以使用哪些类型的过渡对象？ 在过渡期间，共享元素视图是如何以及在何处出现的？ 在接下来的几节中，我们将逐一解决这些问题。

### 共享元素过渡的幕后

回顾过去的两个帖子，过渡有两个主要责任: 捕捉目标视图的开始和结束状态，并创建一个 Animator，它将动画两个状态之间的视图。 共享元素过渡并没有什么不同: 在共享元素过渡创建动画之前，它必须首先捕获每个共享元素的开始和结束状态ー即它在调用和被调用活动 / 片段中的位置、大小和外观。 有了这些信息，过渡可以决定每个共享元素视图应该如何生成动画。

类似于[内容过渡如何在幕后运行](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)，框架通过在运行时直接修改每个共享元素的视图属性来实现共享元素的过渡。 更具体地说，当活动 a 启动活动 b 时，会发生以下事件序列: [2](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#footnote2)

1. 活动 A 调用 startActivity ( )，并且活动 B 被创建，测量并且布局为初始半透明窗口和透明窗口背景颜色。
2. 框架重新定位 B 中的每个共享元素视图以匹配它在 A 中的确切大小和位置。不久之后，B 的进入过渡捕获 B 中所有共享元素的开始状态。
3. 框架重新定位 B 中的每个共享元素视图以匹配它在 B 中的最终大小和位置。紧接着，B 的进入过渡捕获 B 中所有共享元素的结束状态。
4. B 的进入过渡比较其共享元素视图的开始和结束状态，并根据差异创建 Animator。
5. 该框架指示 A 隐藏它的共享元素视图，并且运行生成的 Animator。 由于 B 的共享元素将动画放置到位，因此 B 的窗口背景逐渐淡入 A，直到 B 完全不透明并完成过渡。

尽管内容过渡是由每个可视化视图的变化所决定的，而共享元素的过渡是由每个共享元素视图的位置、大小和外观的变化来决定。 至于 API 21，该框架提供了几种不同的过渡实现，可以用来自定义在场景更改过程中共享元素是如何动画的:

- [ChangeBounds](https://developer.android.com/reference/android/transition/ChangeBounds.html) - 捕获共享元素视图的布局边界并为差异添加动画。 ChangeBounds 常用于共享元素转换，因为大多数共享元素的大小和/或位置在两个活动/片段中都不相同。
- [ChangeTransform](https://developer.android.com/reference/android/transition/ChangeTransform.html) - 捕捉共享元素视图的缩放和旋转，并为差异设置动画。[3](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#footnote3)
- [ChangeClipBounds](https://developer.android.com/reference/android/transition/ChangeClipBounds.html) - 捕获共享元素视图的 [clip bounds](https://developer.android.com/reference/android/view/View.html#getClipBounds()) ，并为差异设置动画。
- [ChangeImageTransform](https://developer.android.com/reference/android/transition/ChangeImageTransform.html) - 捕获共享元素 ImageView 的变换矩阵，并为差异设置动画。 与 ChangeBounds 结合使用时，此过渡允许 ImageView 更改大小，形状和/或 [ImageView.ScaleType](https://developer.android.com/reference/android/widget/ImageView.ScaleType.html) 以平滑高效地进行动画。
- [@android:transition/move](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/res/res/transition/move.xml) - 并行播放上述所有四种转换类型的 TransitionSet。 正如第1部分所讨论的，如果未明确指定进入/返回共享元素过渡，则框架将默认运行此过渡。

在上面的例子中，我们还可以看到，在活动 / 片段中，共享元素视图实例实际上并不是"共享"的。 事实上，用户在进入和返回的共享元素过渡过程中所看到的几乎所有内容都是直接绘制在 b 的内容视图中的。 框架不是将共享元素视图实例从 a 转移到 b，框架使用了不同的方法来实现相同的视觉效果。 当 a 启动 b 时，该框架将收集 a 中共享元素的所有相关状态信息并将其传递给 b，然后使用这些信息初始化其共享元素视图的开始状态，每个元素的初始状态都与它们在 a 中的确切位置、大小和外观相匹配。 当过渡开始时，b 中的所有内容，除了共享元素之外，最初用户是看不到的。 然而，随着过渡的进展，框架在 b 的活动窗口逐渐淡化，直到 b 中的共享元素完成动画，而 b 的窗口背景是不透明的。

### 使用共享元素 Overlay[4](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#footnote4)

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2015/01/12/overlay-opt.mp4" frameborder="0" allowfullscreen></iframe>

最后，在我们能够完全理解共享元素过渡如何被框架所绘制之前，我们必须讨论共享元素覆盖。 尽管不是很明显，但是默认情况下，窗口的 [ViewOverlay](https://developer.android.com/reference/android/view/ViewOverlay.html) 中的共享元素被绘制在整个视图层次结构之上。 如果你以前没有听说过它，那么在 API 18中引入了 ViewOverlay 类，以便于在视图的顶部绘制。 添加到视图的 ViewOverlay 中的 Drawables 和视图将保证在其他所有视图之上绘制，即使是 ViewGroup 的子视图。 考虑到这一点，框架为什么会选择在视图层次结构中默认的视图层次结构中的所有其他元素之外，选择在窗口的 ViewOverlay 中绘制共享元素。 共享元素视图应该是整个过渡过程中的焦点; 可能过渡视图意外地在共享元素之上绘制会立即破坏效果。[5](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#footnote5)

尽管默认情况下，共享元素会在共享元素 ViewOverlay 中绘制，但框架确实给了我们通过调用窗口 [Window#setSharedElementsUseOverlay(false)](https://developer.android.com/reference/android/view/Window.html#setSharedElementsUseOverlay(boolean))  方法来禁用覆盖，以防你觉得有必要。 如果你曾经选择禁用覆盖，要警惕它可能造成的不理想的副作用。 例如，视频3.2运行一个简单的共享元素过渡两次，分别启用和不启用共享元素覆盖。 第一次运行过渡时，共享元素 ImageView 会在共享元素叠加层中按预期进行动画，并在层次结构中的所有其他视图之上进行动画。 然而，第二次运行过渡时，我们可以清楚地看到，禁用覆盖已经引入了一个问题。 当底部过渡视图向上滑入所谓活动的内容视图时，共享元素 ImageView 被部分覆盖，并在过渡的前半部分绘制在过渡视图的下方。 虽然有可能通过改变在布局中绘制视图的顺序和 / 或通过在共享元素的父元素上设置 setClipChildren (false) 来修正这个问题，但是这种 "hacky" 修改很容易变得不可管理，而且比它们的价值更麻烦。 简而言之，试着不要禁用共享元素叠加，除非你发现它是绝对必要的，并且你可能会受益于更简单和更戏剧化的共享元素转换。

### 结论

总的来说，这篇文章提出了三个重要的观点:

1. 共享元素过渡决定了共享元素视图(也称为英雄视图)是如何在场景过渡期间从一个活动 / 片段动画到另一个活动 / 片段。
2. 共享元素的过渡受到每个共享元素视图的位置、大小和外观的变化控制。
3. 共享元素被默认绘制在窗口的整个视图层次结构之上。

一如既往，感谢阅读！ 如果你有任何问题，请随时留下评论，如果你觉得有帮助的话，不要忘记 + 1和 / 或分享这篇博客文章！

------

1 请注意，活动过渡 API 使你可以使用 setSharedElementExitTransition ( ) 和setSharedElementReenterTransition ( ) 方法指定退出并重新进入共享元素过渡，尽管这样做通常不是必需的。 有关说明一个可能用例的示例，请查看此博客文章。 [this blog post.](https://halfthought.wordpress.com/2014/12/08/what-are-all-these-dang-transitions/) 有关退出和重新进入共享元素过渡不能用于片段过渡的解释，请参阅 George Mount 的答案和此 StackOverflow 后的评论。[this StackOverflow post](http://stackoverflow.com/q/27346020/844882). [↩](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#ref1)

2 在活动和片段的退出 / 返回 / 重新进入过渡期间发生了类似的事件。  [↩](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#ref2)

3 ChangeTransform 的另一个微妙特征是，它可以检测和处理在过渡期间对共享元素视图的父级所做的更改。 例如，当共享元素的父元素具有不透明的背景并且默认在场景更改过程中被选择为一个过渡视图时，这个方法就会派上用场。 在这种情况下，ChangeTransform 将会检测到共享元素的父元素正在被内容过渡积极修改，从其父元素中抽出共享元素，并将共享元素分别动画。 更多信息请参阅 George Mount 的 StackOverflow。[StackOverflow answer.](http://stackoverflow.com/q/26899779/844882)  [↩](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#ref3)

4 注意这一部分只涉及活动过渡。 与活动过渡不同，在片段过渡期间，共享元素不会默认地在 ViewOverlay 中绘制。 也就是说，你可以通过应用一个 ChangeTransform 转换来达到类似的效果，如果它发现其父级已经改变，它将在 ViewOverlay 中将共享元素绘制在层次结构之上。 更多信息请参见 StackOverflow 文章。[this StackOverflow post.](http://stackoverflow.com/q/27892033/844882)  [↩](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#ref4)

5 注意到在整个视图层次结构之上绘制共享元素的一个负面副作用是，这意味着共享元素可以在系统 UI (如状态栏、导航栏和操作栏)的顶部绘制共享元素。 有关如何防止这种情况发生的更多信息，请参阅这个 Google + 文章。 [this Google+ post](https://plus.google.com/+AlexLockwood/posts/RPtwZ5nNebb). [↩](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html#ref5)

