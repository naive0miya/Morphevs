# 活动和片段过渡开始(第一部分)

> 原文 ：[Getting Started with Activity & Fragment Transitions (part 1)](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
>
> 作者 ：[Alex Lockwood](https://plus.google.com/+AlexLockwood?prsrc=5)

这篇文章简要介绍了过渡，并介绍了新的活动 & 片段过渡 api，这些 api 被添加到了安卓5.0的棒棒糖中。 这是我将就这个主题撰写的一系列文章中的第一个:

- **Part 1:** [Getting Started with Activity & Fragment Transitions](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
- **Part 2:** [Content Transitions In-Depth](https://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)
- **Part 3a:** [Shared Element Transitions In-Depth](https://www.androiddesignpatterns.com/2015/01/activity-fragment-shared-element-transitions-in-depth-part3a.html)
- **Part 3b:** [Postponed Shared Element Transitions](https://www.androiddesignpatterns.com/2015/03/activity-postponed-shared-element-transitions-part3b.html)
- **Part 3c:** Implementing Shared Element Callbacks (coming soon!)
- **Part 4:** Activity & Fragment Transition Examples (coming soon!)

在写第4部分之前，这里有一个[示例应用程序](https://github.com/alexjlockwood/activity-transitions)演示一些高级活动过渡。

我们首先回答以下问题: 什么是过渡？

### 什么是过渡？

棒棒糖的活动和片段过渡是基于安卓系统中一个相对较新的特性，即 Transitions。 在 KitKat 中引入的过渡框架为应用程序中不同 UI 状态之间的动画化提供了一个方便的 API。 框架是围绕两个关键概念建立的: 场景和过渡。 场景定义了应用程序 UI 的给定状态，而过渡定义了两个场景之间的动画变化。

当一个场景发生变化时，过渡有两个主要的责任:

1. 捕捉开始和结束场景中每个视图的状态，以及
2. 创建一个基于这些差异，将使视图从一个场景动画到另一个场景

举个例子，当用户点击屏幕时，一个活动会淡出它的视图。 如下面的代码1所示，我们只需要使用 Android 的过渡框架，只用几行就能达到这一效果，如下面的代码[1](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#footnote1)所示:

```Java
public class ExampleActivity extends Activity implements View.OnClickListener {
    private ViewGroup mRootView;
    private View mRedBox, mGreenBox, mBlueBox, mBlackBox;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mRootView = (ViewGroup) findViewById(R.id.layout_root_view);
        mRootView.setOnClickListener(this);

        mRedBox = findViewById(R.id.red_box);
        mGreenBox = findViewById(R.id.green_box);
        mBlueBox = findViewById(R.id.blue_box);
        mBlackBox = findViewById(R.id.black_box);
    }

    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(mRootView, new Fade());
        toggleVisibility(mRedBox, mGreenBox, mBlueBox, mBlackBox);
    }

    private static void toggleVisibility(View... views) {
        for (View view : views) {
            boolean isVisible = view.getVisibility() == View.VISIBLE;
            view.setVisibility(isVisible ? View.INVISIBLE : View.VISIBLE);
        }
    }
}
```

为了更好地理解在这个示例中发生了什么，让我们逐步分析这个过程，假设每个视图最初都可以在屏幕上看到:

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2014/12/04/trivial-opt.mp4" frameborder="0" allowfullscreen></iframe>

1. 检测到点击后，开发人员调用 [beginDelayedTransition ( )](https://developer.android.com/reference/android/transition/TransitionManager.html#beginDelayedTransition(android.view.ViewGroup,%20android.transition.Transition))，将场景根和 Fade 过渡作为参数传递。 框架立即为场景中的每个视图调用过渡的 [captureStartValues ( )](https://developer.android.com/reference/android/transition/Transition.html#captureStartValues(android.transition.TransitionValues)) 方法，并且过渡记录每个视图的可见性。
2. 当调用返回时，开发人员将场景中的每个视图设置为 INVISIBLE。
3. 在下一个显示帧，框架为场景中的每个视图调用过渡的 [captureEndValues ( )](https://developer.android.com/reference/android/transition/Transition.html#captureEndValues(android.transition.TransitionValues)) 方法，并且过渡记录每个视图的（最近更新的）可见性。
4. 该框架调用过渡的 [createAnimator ( )](https://developer.android.com/reference/android/transition/Transition.html#createAnimator(android.view.ViewGroup,%20android.transition.TransitionValues,%20android.transition.TransitionValues)) 方法。 过渡分析每个视图的开始和结束值，并注意到不同之处：视图在开始场景中可见，但在结束场景中为 INVISIBLE。 淡入过渡使用这些信息来创建并返回一个 AnimatorSet，它将淡入每个视图的 alpha 属性为 0f。
5. 该框架运行返回的 Animator，导致所有视图逐渐淡出屏幕。

这个简单的例子突出了过渡框架所提供的两个主要优点。 首先，过渡从开发者那里抽象出动画的想法。 因此，过渡可以显著减少你在活动和片段中编写的代码的数量: 开发者必须做的就是设置视图的开始和结束值，并且 Transition 将自动构建一个基于差异的动画。 其次，使用不同的过渡对象可以很容易地改变场景之间的动画。 例如，视频1.1，通过用 Slide 或 Explode 来取代 Fade 过渡，我们可以产生截然不同的效果。 正如我们将看到的那样，这些优势将允许我们用相对较少的代码构建复杂的活动和片段过渡动画。 在接下来的几节中，我们将自己看看如何使用棒棒糖的新活动和片段过渡 api 来实现。

### Android Lollipop 的活动和片段过渡

至于 Android 5.0，在不同的活动或者片段之间切换时，过渡可以用来执行精心设计的动画。 虽然活动和片段动画可以在之前的平台版本中使用 [Activity#overridePendingTransition ( )](http://developer.android.com/reference/android/app/Activity.html#overridePendingTransition(int,%20int))  和 [FragmentTransaction#setCustomAnimation ( )](http://developer.android.com/reference/android/app/FragmentTransaction.html#setCustomAnimations(int,%20int,%20int,%20int)) 来指定活动和片段动画，但它们是有限的，因为它们只能对整个活动 / 片段容器进行动画处理。 新的棒棒糖 api 更进一步，让个人视图动画化成为可能，当他们进入或离开他们的容器，甚至允许我们从一个活动 / 片段容器到另一个活动 / 片段容器中进行动画化的共享视图。

让我们开始讨论这一系列文章中将使用的术语。 请注意，尽管以下术语的定义是活动过渡，但也将使用完全相同的术语来进行片段过渡:

> 让 a 和 b 作为活动，并假定活动 a 开始活动 b。 我们把 a 称为"调用活动"(调用 startActivity ( )) ，b 称为"被调用活动"。

活动过渡 api 是围绕退出、进入、返回和重新进入过渡的想法构建的。 在上面定义的活动 a 和活动 b 中，我们可以把每个活动描述如下:

> 活动 a 的退出过渡决定了 a 中的视图在启动 b 时是如何动画的。
>
> 活动 b 的进入过渡决定了 b 中的视图在开始 b 时是如何动画的。
>
> 活动 b 的返回过渡决定了 b 返回到 a 时 b 中的视图是如何动画的。
>
> 活动 a 的重新进入过渡决定了当 b 返回到 a 时，a 中的视图是如何动画的。

最后，该框架为两种活动过渡提供了 api ——内容过渡和共享元素过渡——每一种都允许我们以独特的方式自定义活动之间的动画:

> 内容过渡决定了活动的非共享视图(也称为过渡视图)如何进入或退出活动场景。
>
> 共享元素的过渡决定了一个活动的共享元素(也称为英雄视图)是如何在两个活动之间动画的。

<iframe width="560" height="315" src="https://www.androiddesignpatterns.com/assets/videos/posts/2014/12/04/news-opt.mp4" frameborder="0" allowfullscreen></iframe>

视频1.2提供了一个很好的插图，内容过渡和共享元素过渡使用在 Google Play Newsstand 应用程序。 虽然我们不能看到 Newsstand 源代码，我最好的猜测是使用下面的过渡:

1. 活动 A（调用活动）的退出和重新进入内容过渡均为空。 我们可以这样说，因为当用户退出并重新进入活动时， A 中的非共享视图不会生成动画。[2](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#footnote2)

2. 活动 B（被调用活动）的进入内容过渡使用自定义幻灯片过渡，将列表项目从屏幕底部移动到适当位置。
3. 活动 B 的返回内容过渡是一个 TransitionSet，它并行地播放两个子过渡：一个 Slide (Gravity.TOP) 过渡，用于定位活动上半部分的视图以及一个 Slide (Gravity.BOTTOM) 过渡，用于定位活动的下半部分。 其结果是，当用户点击后退按钮并返回到活动 A 时，活动似乎“从中间断开”。
4. 进入和返回共享元素转换都使用 ChangeImageTransform，导致 ImageView 在两个活动之间无缝地进行动画。

你可能也注意到了在过渡过程中在共享元素下播放的很酷的揭露动画。 我们将在未来的博客文章中介绍如何做到这一点。 现在，让我们把事情简单化，熟悉活动和片段过渡 api。

### 介绍活动过渡 API

使用新的棒棒糖 api，创建一个基本的 Activity 过渡是相对容易的。 以下是为了在应用程序中实现一个步骤而必须采取的步骤。 在接下来的文章中，我们将会经历更先进的用例和例子，但是现在接下来的两个章节将作为一个很好的介绍:

- 通过以编程方式或在主题的 XML [3](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#footnote3) 中请求被调用和调用活动中的 [Window.FEATURE_ACTIVITY_TRANSITIONS](http://developer.android.com/reference/android/view/Window.html#FEATURE_ACTIVITY_TRANSITIONS) 窗口功能来启用新的过渡 API。默认情况下，材质主题应用程序启用此标志。

- 为你的调用和调用活动设置[退出](https://developer.android.com/reference/android/view/Window.html#setExitTransition(android.transition.Transition))和[进入](https://developer.android.com/reference/android/view/Window.html#setEnterTransition(android.transition.Transition))内容过渡。 材质为主题的应用程序的默认退出和进入内容过渡分别设置为 null 和 [Fade](https://developer.android.com/reference/android/transition/Fade.html)。 如果没有明确设置[重新进入](https://developer.android.com/reference/android/view/Window.html#setReenterTransition(android.transition.Transition))或[返回](https://developer.android.com/reference/android/view/Window.html#setReturnTransition(android.transition.Transition))过渡，则分别使用活动的退出和进入内容过渡。

- 为你的调用和调用活动设置[退出](https://developer.android.com/reference/android/view/Window.html#setExitTransition(android.transition.Transition))和[进入](https://developer.android.com/reference/android/view/Window.html#setEnterTransition(android.transition.Transition))共享元素过渡。 材质为主题的应用程序的默认退出和进入共享元素过渡设置为 [@android:transition/move](https://github.com/android/platform_frameworks_base/blob/lollipop-release/core/res/res/transition/move.xml)。 如果没有明确设置[重新进入](https://developer.android.com/reference/android/view/Window.html#setReenterTransition(android.transition.Transition))或[返回](https://developer.android.com/reference/android/view/Window.html#setReturnTransition(android.transition.Transition))过渡，则分别使用活动的退出和进入共享元素过渡。

- 要启动具有内容过渡和共享元素的活动过渡，请调用 [startActivity(Context, Bundle)](http://developer.android.com/reference/android/app/Activity.html#startActivity%28android.content.Intent,%20android.os.Bundle%29) 方法并将以下 Bundle 作为第二个参数传递：

  ```Java
   ActivityOptions.makeSceneTransitionAnimation(activity, pairs).toBundle();
  ```

  在这里，Pair 是一组 Pair<View, String> 对象列出了你想在活动之间共享的共享元素视图和名称。[4](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#footnote4) 不要忘记给你共享的元素独特的过渡名称，无论是在[程序](https://developer.android.com/reference/android/view/View.html#setTransitionName(java.lang.String))还是  [XML](https://developer.android.com/reference/android/view/View.html#attr_android:transitionName)。 否则，过渡将不会正常工作！

- 要以编程方式触发返回转换，请调用 [finishAfterTransition ( )](https://developer.android.com/reference/android/app/Activity.html#finishAfterTransition()) 而不是 finish ( )。

- 默认情况下，以材质为主题的应用程序的输入 / 返回内容过渡在其退出 / 重新输入内容过渡完成之前开始，创造一个小的重叠，使整体效果更加无缝和戏剧化。如果你希望明确地禁止这种行为，你可以通过调用 [setWindowAllowEnterTransitionOverlap ( )](http://developer.android.com/reference/android/view/Window.html#setAllowEnterTransitionOverlap(boolean)) 和 [setWindowAllowReturnTransitionOverlap ( )](http://developer.android.com/reference/android/view/Window.html#setAllowReturnTransitionOverlap(boolean))

  方法或者通过在主题的 XML 中设置相应的属性来实现。

### 介绍片段过渡 API

如果你正在处理片段过渡，那么 API 有一些小的差异类似:

- 内容[退出](https://developer.android.com/reference/android/app/Fragment.html#setExitTransition(android.transition.Transition))，[进入](https://developer.android.com/reference/android/app/Fragment.html#setEnterTransition(android.transition.Transition))，[重新进入](https://developer.android.com/reference/android/app/Fragment.html#setReenterTransition(android.transition.Transition))和[返回](https://developer.android.com/reference/android/app/Fragment.html#setReturnTransition(android.transition.Transition))过渡应通过调用 Fragment 类中的相应方法或 Fragment 的 XML 标记中的属性来设置。
- 应通过调用 Fragment 类中的相应方法或 Fragment 的 XML 中的属性来设置共享元素[进入](https://developer.android.com/reference/android/app/Fragment.html#setSharedElementEnterTransition(android.transition.Transition))和[返回](https://developer.android.com/reference/android/app/Fragment.html#setSharedElementReturnTransition(android.transition.Transition))过渡。
- 鉴于活动过渡是通过显式调用  startActivity ( ) 和 finishAfterTransition ( ) 来触发的，当片段被添加，删除，附加，分离，显示或被 FragmentTransaction 隐藏时，会自动触发片段过渡。
- 在提交事务之前，通过调用  [addSharedElement(View, String)](https://developer.android.com/reference/android/app/FragmentTransaction.html#addSharedElement(android.view.View,%20java.lang.String)) 方法，将共享元素指定为FragmentTransaction 的一部分。

### 结论

在这篇文章中，我们只简要介绍了新的 Activitiy 和 Fragment 过渡 api。 然而，正如我们将在接下来的几篇文章中看到的那样，对基本知识有一个坚实的理解，将大大加快长期的发展进程，特别是在编写自定义过渡时。 在接下来的文章中，我们将更深入地讨论内容过渡和共享元素过渡，并将获得对活动和片段过渡如何在幕后下工作有更深入的了解。

一如既往，感谢阅读！ 如果你有任何问题，请随时留下评论，如果你觉得有帮助的话，不要忘记 + 1和 / 或分享这篇博客文章！

------

1 如果你想自己尝试这个示例，可以在这里找到 XML 布局代码。 [here](https://gist.github.com/alexjlockwood/a96781b876138c37e88e). [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#ref1)

2 看起来 a 中的视图在屏幕上逐渐消失，但是你真正看到的是活动 b 在活动 a 的上方在屏幕上褪色。 活动 a 中的视图实际上并不是动画，在这段时间。 你可以通过调用 [setTransitionBackgroundFadeDuration ( )](http://developer.android.com/reference/android/view/Window.html#setTransitionBackgroundFadeDuration(long))  来调整背景淡出的持续时间。 [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#ref2)

3 对于描述 FEATURE_ACTIVITY_TRANSITIONS 和 FEATURE_CONTENT_TRANSITIONS 标志之间的差异的解释，请参见 StackOverflow 文章。 [this StackOverflow post](http://stackoverflow.com/q/28975840/844882). [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#ref3)

4 为了启动一个具有内容过渡但没有共享元素的活动过渡，你可以通过调用  ActivityOptions.makeSceneTransitionAnimation(activity).toBundle( ) 来创建 Bundle。为了完全禁用内容过渡和共享元素过渡，不要创建一个 Bundle 对象，而只是传递 null。 [↩](https://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html#ref4)

