# 推迟的共享元素过渡(第 3b 部分)

> 原文 (androiddesignpatterns.com)：[Postponed Shared Element Transitions (part 3b)](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html)
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

我们首先讨论了由于一个常见问题而推迟某些共享元素过渡的必要性。

### 理解问题

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2015/03/09/postpone-bug-opt.mp4" frameborder="0" allowfullscreen></iframe>

在处理共享元素过渡时，一个常见的问题来源于这样一个事实，即它们是在活动生命周期的早期由框架开始的。 从第一部分回顾，转场必须捕捉目标视图的开始和结束状态，以便构建一个正常运行的动画。 因此，如果框架在其共享元素在所谓活动中给出最终大小、位置和大小之前，框架就开始了共享元素的过渡，那么这个过渡将会捕获其共享元素的不正确的最终值，并且由此产生的动画将会完全失败(见视频3.3，一个失败的进入过渡可能看起来的例子)。

是否在过渡开始之前计算共享元素的最终值主要取决于两个因素: (1)所谓活动布局的复杂性和深度; (2)所谓的活动加载所需数据所需的时间。 布局越复杂，在屏幕上确定共享元素的位置和大小所需的时间就越长。 同样，如果共享元素在活动中的最终外观取决于异步加载的数据，那么框架可能会在将数据传递回主线程之前自动启动共享元素转换。以下列出了你可能遇到这些问题的一些常见情况:

- 共享元素存在于一个被调用活动托管的片段中。 [事务在提交后不会立即执行](https://developer.android.com/reference/android/app/FragmentTransaction.html#commit()); 它们将作为主线程的工作安排在稍后时间完成。 因此，如果共享元素存在于片段的视图层次结构中，而 FragmentTransaction 执行得不够快，则框架可能会在共享元素正确测量并在屏幕上显示之前启动共享元素的过渡。[1](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#footnote1)
- 共享元素是一个高分辨率的图像。 设置一个高分辨率图像，超过 ImageView 的初始边界可能会触发视图层次结构上的[额外布局传递](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/java/android/widget/ImageView.java#L453-L455)，从而增加了在共享元素准备好之前开始过渡的可能性。 像 [Volley](https://android.googlesource.com/platform/frameworks/volley) 和 [Picasso](http://square.github.io/picasso/) 这样的流行位图加载 / 缩放库的异步特性不能可靠地解决这个问题: 该框架之前并不知道图像正在从后台线程上下载、缩放和 / 或从磁盘中获取图像，不管图像是否仍在处理中，都将开始共享元素的过渡。
- 共享元素依赖于异步加载的数据。 如果共享元素需要一个 AsyncTask，一个 AsyncQueryHandler，一个 Loader ，或者类似的东西，在他们最后出现在所谓的活动之前，框架可能会开始过渡，然后再将数据传送回主线程。

### postponeEnterTransition ( ) 和 startPostponedEnterTransition ( )

在这一点上，你可能会想,"如果有一种暂时延迟过渡的方法，直到我们确切地知道共享元素已被正确测量和布局。" 好吧，你很幸运，因为活动过渡 API [2](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#footnote2) 给了我们这样做的方法！

为了暂时阻止共享元素过渡，请在被调用的 activity 的 onCreate ( ) 方法中调用 [postponeEnterTransition ( )](https://developer.android.com/reference/android/app/Activity.html#postponeEnterTransition()) 。 稍后，当你确定所有共享元素已经被正确地定位和调整大小时，调用 startPostponedEnterTransition ( ) 来恢复过渡。 你会发现一个常见的模式是在 OnPreDrawListener 中启动延迟的过渡，它将在共享元素被测量和显示之后被调用: [3](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#footnote3)

```Java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // Postpone the shared element enter transition.
    postponeEnterTransition();

    // TODO: Call the "scheduleStartPostponedTransition()" method
    // below when you know for certain that the shared element is
    // ready for the transition to begin.
}

/**
 * Schedules the shared element transition to be started immediately
 * after the shared element has been measured and laid out within the
 * activity's view hierarchy. Some common places where it might make
 * sense to call this method are:
 * 
 * (1) Inside a Fragment's onCreateView() method (if the shared element
 *     lives inside a Fragment hosted by the called Activity).
 *
 * (2) Inside a Picasso Callback object (if you need to wait for Picasso to
 *     asynchronously load/scale a bitmap before the transition can begin).
 *
 * (3) Inside a LoaderCallback's onLoadFinished() method (if the shared
 *     element depends on data queried by a Loader).
 */
private void scheduleStartPostponedTransition(final View sharedElement) {
    sharedElement.getViewTreeObserver().addOnPreDrawListener(
        new ViewTreeObserver.OnPreDrawListener() {
            @Override
            public boolean onPreDraw() {
                sharedElement.getViewTreeObserver().removeOnPreDrawListener(this);
                startPostponedEnterTransition();
                return true;
            }
        });
}
```

尽管这两个方法的名字不同，它们也可以用来推迟共享元素的返回过渡。 简单地推迟调用 [onActivityReenter ( )](https://developer.android.com/reference/android/app/Activity.html#onActivityReenter(int,%20android.content.Intent))  方法中的返回过渡: [4](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#footnote4)

```Java
/**
 * Don't forget to call setResult(Activity.RESULT_OK) in the returning
 * activity or else this method won't be called!
 */
@Override
public void onActivityReenter(int resultCode, Intent data) {
    super.onActivityReenter(resultCode, data);

    // Postpone the shared element return transition.
    postponeEnterTransition();

    // TODO: Call the "scheduleStartPostponedTransition()" method
    // above when you know for certain that the shared element is
    // ready for the transition to begin.
}
```

尽管让你的共享元素过渡更加顺畅和可靠，但同样重要的是要意识到在应用程序中引入延迟的共享元素过渡也可能有一些潜在的有害副作用:

- 永远不要忘记调用 startPostponedEnterTransition ( )。 忘记这样做将使你的应用程序处于死锁状态，使用户永远无法访问下一个 Activity 屏幕。
- 永远不要推迟过渡时间超过几分之一秒。 推迟一小段时间的过渡可能会在应用程序中引入不必要的延迟，扰乱用户，减缓用户体验。

一如既往，感谢阅读！ 如果你有任何问题，请随时留下评论，如果你觉得有帮助的话，不要忘记 + 1和 / 或分享这篇博客文章！

------

1 当然，大多数应用程序通常可以通过调用 [FragmentManager#executePendingTransactions ( )](https://developer.android.com/reference/android/app/FragmentManager.html#executePendingTransactions()) 来解决这个问题，这将迫使任何待处理的 FragmentTransactions 立即执行而不是异步执行。 [↩](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#ref1)

2 请注意，postponeEnterTransition ( ) 和 startPostponedEnterTransition ( ) 方法只适用于 Activity Transitions 而不是 Fragment Transitions。 有关解释和可能的解决方案，请参阅 StackOverflow 的答案和这个 Google + 文章。 [this StackOverflow answer](http://stackoverflow.com/q/26977303/844882) 和 [this Google+ post](https://plus.google.com/+AlexLockwood/posts/3DxHT42rmmY). [↩](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#ref2)

3 Pro tip: 你可以通过调用 [View#isLayoutRequested ( )](http://developer.android.com/reference/android/view/View.html#isLayoutRequested())  来验证是否需要分配 OnPreDrawListener。 [View#isLaidOut ( )](http://developer.android.com/reference/android/view/View.html#isLaidOut()) 在某些情况下也可能派上用场。[↩](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#ref3)

4 测试共享元素返回 / 重新进入过渡行为的一个好方法是进入开发者选项，并启用"不保留活动"设置。 这将有助于测试在返回过渡开始之前，调用活动需要重新创建其布局、请求任何必要的数据等。 [↩](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html#ref4)

