# 在 Android 上探索有意义的运动

> 原文 (Medium)：[Exploring Meaningful Motion on Android](https://labs.ribot.co.uk/exploring-meaningful-motion-on-android-1cd95a4bc61d)
>
> 作者：[Joe Birch](https://labs.ribot.co.uk/@hitherejoe)

[TOC]

![图片来源：Google](https://ws1.sinaimg.cn/large/006tNc79gy1fron6n93v0j30rs0fmt91.jpg)

在 [ribot](http://www.ribot.co.uk/) 我们关心为人们创造美丽而有意义的体验，在这种体验中，[运动起着很大的作用](http://ribot.co.uk/thought/motional-intelligence/)。 

在 Droidcon London 看到一个[鼓舞人心的演讲](https://photos.google.com/share/AF1QipMRnZL6gNbS06fnBNtKffRm9HBaxW8iP6w0L1T4nZYLI6s3wi_l8daT6mq4nwPf-w?key=LThZNmFXUUtmNi04bWlEYmVfcWdPenlvaDdCRU13)后，我决定深入研究安卓系统。 由此，我把我的发现整合在一起，帮助开发者和设计师们意识到在你的 Android 应用程序中添加漂亮的动作是多么容易。 

如果你想为自己尝试这些动画，所有这些示例都打包到 [Github](https://github.com/hitherejoe/animate)上的 Android 应用程序中。

我喜欢运动，它不仅能提高参与度，而且能立刻引起注意。 想想你使用的应用程序，你使用的特征运动设计和他们感觉如何取悦，满意，流畅和自然。 

![Falcon Pro：即使是微妙的动作也可以对用户体验产生巨大的影响](https://ws2.sinaimg.cn/large/006tNc79gy1fron9dqntmg307i0dcnpe.gif)

现在把它和那些你喜欢的缺乏同样感觉的应用进行比较。 

![Medium：就像我喜欢这个mediem应用程序一样，它确实缺乏应有的地方](https://ws3.sinaimg.cn/large/006tNc79gy1frona0yckag307i0dcnpe.gif)

> 小事情有很大的不同

我们可以通过多种方式来利用这些运动效果：

- 通过导航上下文传输用户
- 加强元素层次结构
- 解释屏幕上显示的组件之间的变化

本文的目的是向你展示在应用程序中实现有意义的动作是多么容易，所以让我们开始吧。

## 触摸反馈

当用户触摸屏幕时，提供反馈有助于以视觉形式进行交流，已经进行了交互。 这些动画不应该分散用户的注意力，但要足够让他们享受，从中获得清晰度，并鼓励进一步的探索。 

Android 框架为这个级别的反馈提供了波纹状态，可以通过将视图的背景设置为下列之一来使用：

- ?android:attr / selectableItemBackground - 波纹从触点开始，填充视图的背景。

  ![波纹从触摸点开始，填充视图的背景](https://ws4.sinaimg.cn/large/006tNc79gy1frongtbahtg30dk06a0w4.gif)

- ?android:attr / selectableItemBackgroundBorderless - 一个循环纹波效应从触点开始，填充一个半径，扩展视图的边界。

![在触摸点处开始出现圆形的波纹效果，填充延伸视图边界的半径](https://ws3.sinaimg.cn/large/006tNc79gy1fronh0ich6g306703wtad.gif)

## 视图属性动画

 [ViewPropertyAnimator](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html) 是在 API 级别 12 引入的，允许你使用单个 [Animator](http://developer.android.com/reference/android/animation/Animator.html) 实例简单高效地对多个视图属性执行动画操作（并行）。

![在这里我动画所有属性如下所示。](https://ws1.sinaimg.cn/large/006tNc79gy1fronh7yn2rg30dj03f75f.gif)

- [alpha ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#alpha%28float%29) - 将 alpha 值设置为动画
- [scaleX ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#scaleX%28float%29) & [scaleY ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#scaleY%28float%29) - 在 X 轴和/或 Y 轴上缩放视图
- [translationZ ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#translationZ%28float%29) - 平移 Z 轴上的视图
- [setDuration ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#setDuration%28long%29) - 设置动画的持续时间
- [setStartDelay ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#setStartDelay%28long%29) - 设置动画的延迟
- [setInterpolator ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#setInterpolator%28android.animation.TimeInterpolator%29) - 设置动画插值器
- [setListener ( )](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html#setListener%28android.animation.Animator.AnimatorListener%29) - 为动画开始，结束，重复或取消时设置侦听器。

**注意：**当一个监听器被设置在一个视图上时，如果你在同一个视图上执行其他动画并且不希望使用该回调，那么你必须将监听器设置为 null。

编程实现对我们来说既简单又整洁：

```java
mButton.animate()
        .alpha(1f)
        .scaleX(1f)
        .scaleY(1f)
        .translationZ(10f)
        .setInterpolator(new FastOutSlowInInterpolator())
        .setStartDelay(200)
        .setListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) { }

            @Override
            public void onAnimationEnd(Animator animation) { }

            @Override
            public void onAnimationCancel(Animator animation) { }

            @Override
            public void onAnimationRepeat(Animator animation) { }
        })
        .start();
```

注意：我们实际上并不需要在我们的动画构建器上调用 start ( )，因为一旦我们停止声明动画，动画就会自动同时开始。 在这种情况下，直到从 UI 工具箱事件队列中进行下一次更新，动画才会启动。

![动画FAB的Alpha值](https://ws1.sinaimg.cn/large/006tNc79gy1fronhbwef5g303b03aq3k.gif)

![动画FAB的比例（X和Y轴）](https://cdn-images-1.medium.com/max/800/1*XHfJhQyuMAzD_gH3B775iw.gif)

![动画FAB的Z轴](https://cdn-images-1.medium.com/max/800/1*MAAZAw5g732VDHYQqw9uMg.gif)

**注意：**为了向后兼容，你可以使用 [ViewCompat](http://developer.android.com/reference/android/support/v4/view/ViewCompat.html) 类从 Android API 版本4及更高版本实现 ViewPropertyAnimator。

## 对象动画

与 [ViewPropertyAnimator](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html) 类似， [ObjectAnimator](http://developer.android.com/reference/android/animation/ObjectAnimator.html) 允许我们在目标视图的各种属性（包括代码和 XML 资源文件）上执行动画。但是，有一些差异：

- ObjectAnimator 只允许每个实例的单个属性的动画，例如缩放 X，然后缩放 Y.
- 不过，它允许自定义[属性](http://developer.android.com/reference/android/util/Property.html)上的动画，例如视图的前景色。

使用一个自定义属性为视图的缩放设置动画并改变其前景色，我们可以实现这一点：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fronhmncvwg307i0dcqka.gif)

使用我们的 [Custom Property](https://github.com/hitherejoe/animate/blob/master/app/src/main/java/com/hitherejoe/animate/ui/widget/ForegroundFrame.java#L90)，我们可以通过调用 [ObjectAnimator.ofInt( )](http://developer.android.com/reference/android/animation/ObjectAnimator.html#ofInt%28T,%20android.util.Property%3CT,%20java.lang.Integer%3E,%20int...%29) 来创建一个 ObjectAnimator 实例，我们声明了：

- **view** - 应用动画的视图
- **property** - 要动画的属性
- **start color** - 动画视图开始的颜色
- **target color**- 视图应该生成的颜色

然后设置评估者（我们使用 [ArgbEvaluator](http://developer.android.com/reference/android/animation/ArgbEvaluator.html) ，因为我们在颜色值之间进行动画处理），设置延迟并 start ( ) 动画。

```java
private void animateForegroundColor(@ColorInt final int targetColor) {
    ObjectAnimator animator = 
        ObjectAnimator.ofInt(YOUR_VIEW, FOREGROUND_COLOR, Color.TRANSPARENT, targetColor);
    animator.setEvaluator(new ArgbEvaluator());
    animator.setStartDelay(DELAY_COLOR_CHANGE);
    animator.start();
}
```

接下来我们需要动画视图的缩放比例，我们采用类似的方法。 主要区别在于：

- 我们使用 [ObjectAnimator.ofFloat( )](http://developer.android.com/reference/android/animation/ValueAnimator.html#ofFloat%28float...%29) 创建一个 ObjectAnimator 实例，因为在处理视图大小时我们并不是动画 int 值
- 我们不使用自定义属性，而是使用 View.SCALE_X 和 View.SCALE_Y 视图属性

```java
private void resizeView() {
    final float widthHeightRatio = (float) getHeight() / (float) getWidth();
    resizeViewProperty(View.SCALE_X, .5f, 200);
    resizeViewProperty(View.SCALE_Y, .5f / widthHeightRatio, 250);
}

private void resizeViewProperty(Property<View, Float> property,
                                float targetScale, 
                                int durationOffset) {
    ObjectAnimator animator = ObjectAnimator.ofFloat(this, property, 1f, targetScale);
    animator.setInterpolator(new LinearOutSlowInInterpolator());
    animator.setStartDelay(DELAY_COLOR_CHANGE + durationOffset);
    animator.start();
}
```

最后，我们需要将我们调整大小的视图从屏幕上移开。 在这种情况下，我们使用一个 [AdapterViewFlipper](http://developer.android.com/reference/android/widget/AdapterViewFlipper.html) 来包含我们在屏幕外动画的视图。 这意味着我们可以在 ViewFlipper 实例上调用 [showNext ( )](http://developer.android.com/reference/android/widget/AdapterViewFlipper.html#showNext%28%29)，它将使用[我们定义的动画]([animation that we’ve defined](https://github.com/hitherejoe/animate/blob/master/app/src/main/java/com/hitherejoe/animate/ui/activity/ObjectAnimatorActivity.java#L42))处理屏幕上的动画视图。 然后，下一个视图在屏幕上自动生成动画，也使用[我们定义的入口动画](https://github.com/hitherejoe/animate/blob/master/app/src/main/java/com/hitherejoe/animate/ui/activity/ObjectAnimatorActivity.java#L40)。

## 插值器

[插值器](http://developer.android.com/reference/android/view/animation/Interpolator.html)可用于定义动画的变化率，这意味着动画期间的速度、加速度和行为可以被改变。有几种不同类型的插值器，其中一些插值器之间的差异是微妙的，所以我建议在一个设备上尝试它们。 

- 无插值器 - 视图[动画](https://github.com/hitherejoe/animate/blob/master/images/no_interpolator.gif?raw=true)没有变化率的变化
- [Fast-Out Linear-In](http://developer.android.com/reference/android/support/v4/view/animation/FastOutLinearInInterpolator.html) - 视图开始动画并以直线运动结束

![视图开始动画并以直线运动结束](https://ws3.sinaimg.cn/large/006tNc79gy1fronhw6jwqg30m803mjvj.gif)

- [Fast-Out Slow-In](http://developer.android.com/reference/android/support/v4/view/animation/FastOutSlowInInterpolator.html) - 视图快速开始动画并放慢速度完成

  ![视图快速开始动画并放慢速度完成](https://ws3.sinaimg.cn/large/006tNc79gy1froni30zw1g30m803mjwg.gif)

- [Linear-Out Slow-In](http://developer.android.com/reference/android/support/v4/view/animation/LinearOutSlowInInterpolator.html) - 视图以直线运动开始并减速至结束

![视图以直线运动开始并减速至结束](https://ws4.sinaimg.cn/large/006tNc79gy1froni9166tg30m803m793.gif)

- [Accelerate-Decelerate](http://developer.android.com/reference/android/view/animation/AccelerateDecelerateInterpolator.html) - 在动画开始时，视图看起来会加速，并在结束时逐渐减速

![在动画开始时，视图看起来会加速，并在结束时逐渐减速](https://ws3.sinaimg.cn/large/006tNc79gy1froniepzcog30m803m798.gif)

- [Accelerate](http://developer.android.com/reference/android/view/animation/AccelerateInterpolator.html) - 视图逐渐加速直到动画结束

![视图逐渐加速直到动画结束](https://ws2.sinaimg.cn/large/006tNc79gy1fronivaglvg30m803m7qu.gif)

- [Decelerate](http://developer.android.com/reference/android/view/animation/DecelerateInterpolator.html) - 视图逐渐减速直到动画结束

![视图逐渐减速直到动画结束](https://ws2.sinaimg.cn/large/006tNc79gy1fronj6gby8g30m803mgqz.gif)

- [Anticipate](http://developer.android.com/reference/android/view/animation/AnticipateInterpolator.html) - 在以标准方式制作动画之前，视图从所述动画的轻微倒转开始

![在以标准方式制作动画之前，视图从所述动画的轻微倒转开始](https://ws2.sinaimg.cn/large/006tNc79gy1fronjdt9n8g30m803mwjg.gif)

- [Anticipate-Overshoot](http://developer.android.com/reference/android/view/animation/AnticipateOvershootInterpolator.html) - 与 “ Anticipate ” 类似，但动画过程中发生的拉回动作略微更夸张

![与“Anticipate”类似，但动画过程中发生的拉回动作略微更夸张](https://ws4.sinaimg.cn/large/006tNc79gy1fronjteaxgg30m803mwjk.gif)

- [BounceInterpolator](http://developer.android.com/reference/android/view/animation/BounceInterpolator.html) - 该视图在结束之前激发“反弹”效果

![该视图在结束之前激发“反弹”效果](https://ws1.sinaimg.cn/large/006tNc79gy1fronk1zglfg30m803mael.gif)

- [LinearInterpolator](http://developer.android.com/reference/android/view/animation/LinearInterpolator.html) - 以直线和平滑的动作从头到尾进行动画的视图

![以直线和平滑的动作从头到尾进行动画的视图](https://ws3.sinaimg.cn/large/006tNc79gy1fronk96aysg30m803mn23.gif)

- [OvershootInterpolator](http://developer.android.com/reference/android/view/animation/OvershootInterpolator.html) - 视图可以夸大给定的值，缩回到所需的值

![视图可以夸大给定的值，缩回到所需的值](https://ws3.sinaimg.cn/large/006tNc79gy1fronkf54t1g30m803mtcy.gif)

## 圆形揭露

[CircularReveal](http://developer.android.com/training/material/animations.html#Reveal) 动画使用剪切圈来显示或隐藏一组 UI 元素。除了帮助提供视觉连续性外，这也是一个令人愉快和愉快的互动，以帮助获得用户的参与。

![](https://ws4.sinaimg.cn/large/006tNc79ly1fronktfcbig30ay0hfqfn.gif)

如上所示，我们首先使用 ViewPropertyAnimator 来隐藏浮动动作按钮，然后开始在视图上显示动画。设置我们的通告只需要定义几个属性：

- **startView** - CircularReveal 将从（即按下的视图）开始的视图
- **centerX** - 中心坐标为点击视图的 X 轴
- **centerY** - 中心坐标为点击视图的 Y 轴
- **targetView** - 要揭露的视图
- **finalRadius** - 剪切圆的半径，等于我们 centerX 和 centerY 值的(直角三角形的)斜边

```java
int centerX = (startView.getLeft() + startView.getRight()) / 2;
int centerY = (startView.getTop() + startView.getBottom()) / 2;
float finalRadius = (float) Math.hypot((double) centerX, (double) centerY);
Animator mCircularReveal = ViewAnimationUtils.createCircularReveal(
  targetView, centerX, centerY, 0, finalRadius);
```

## 窗口过渡

[自定义](http://developer.android.com/training/material/animations.html#Transitions)用于在活动之间导航的过渡允许你在应用程序状态之间生成更强大的可视化连接。默认情况下，我们可以自定义以下转换：

- **enter** - 确定活动的视图如何进入场景
- **exit** - 确定活动的视图如何退出场景
- **reenter**- 确定之前退出后活动重新进入的方式
- **shared elements** - 确定共享视图如何在活动之间转换

从 API 级别21开始，引入了几个新的转换：

## Explode

爆炸过渡允许视图从屏幕的所有侧面退出，从按下的视图创建爆炸效果。

![爆炸效果对于基于网格的布局非常有效](https://ws2.sinaimg.cn/large/006tNc79ly1fronl4zimig307i0dch1l.gif)

这种效果很容易实现 - 首先，你需要在 res / transition 目录中创建以下转换。

```xml
<explode xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"/>
```

我们在这里完成的是：

1. 声明爆炸过渡	
2. 将持续时间设置为300毫秒

接下来，我们需要将其设置为活动的过渡。 我们通过将其添加到我们的活动主题来做到这一点：

```xml
<style name="AppTheme.Explode" parent="AppTheme.NoActionBar">
  <item name="android:windowExitTransition">@transition/slide_explode</item>
  <item name="android:windowReenterTransition">@android:transition/slide_top</item>
</style>
```

或以编程方式：

```java
Transition explode = TransitionInflater.from(this).inflateTransition(R.transition.explode);
getWindow().setEnterTransition(explode);
```

## Slide

幻灯片切换允许你从屏幕的右侧或底部滑入/滑出活动。虽然以前可以实现类似的功能，但是这种新的过渡更为灵活。

![幻灯片转换允许我们顺序滑动子视图](https://ws2.sinaimg.cn/large/006tNc79gy1fronvmb412g307i0dc4iy.gif)

当过渡活动时，这种过渡很可能是常见的，由于流畅的感觉，我特别喜欢正确的滑动。再次，这很容易创建：

```xml
<slide xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@android:interpolator/decelerate_cubic"
    android:slideEdge="end"/>
```

我们在这里：

1. 声明幻灯片过渡
2. 设置过渡的 slideEdge 结束（右），所以幻灯片从右侧执行 - 底部幻灯片将被设置为底部

## Fade

淡入淡出转换允许你使用淡入淡出效果来过渡进出的活动。

![淡入淡出的过渡是一个简单而愉快的过渡，用于淡入视图](https://ws2.sinaimg.cn/large/006tNc79ly1fronlry8ujg307i0dch0t.gif)

创建它比以前的过渡更简单：

```xml
<fade xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"/>
```

我们在这里：

1. 声明淡入淡出的过渡
2. 将持续时间设置为 300 毫秒

## 优化过渡

在尝试的过程中，我发现了一些可以帮助改善上述过渡效果的方法。

**允许窗口内容过渡** - 你需要在从材质主题继承的主题中启用以下属性：

```xml
<item name="android:windowContentTransitions">true</item>
```

**启用/禁用过渡重叠** - 转换时，如果某个活动正在等待另一个转换完成转换，则可能会延迟，然后才能启动它。 根据使用情况，如果启用这些属性，过渡趋于更加流畅和自然：

```xml
<item name="android:windowAllowEnterTransitionOverlap">true</item>
<item name="android:windowAllowReturnTransitionOverlap">true</item>
```

**从转换中排除视图** - 有时我们可能不想转换所有活动的视图。 我发现在大多数情况下，状态栏和工具栏引起了过渡故障。 幸运的是，我们可以排除具体的视图被纳入我们的过渡：

```xml
<explode xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="200">
    <targets>
        <target android:excludeId="@android:id/navigationBarBackground"/>
        <target android:excludeId="@android:id/statusBarBackground"/>
    </targets>
</explode>
```

**工具栏和 ActionBar** - 当使用工具栏的活动与使用工具栏的活动（反之亦然）之间过渡时，有时候我发现过渡并不总是平滑的。 为了解决这个问题，我确保了两个涉及到过渡的活动使用了相同的组件。

**过渡时间** - 你不想让用户等待太久，但你也不希望组件以光速出现。 这取决于你正在使用的过渡，所以最好进行实验，但是我发现在大多数情况下，200-500 毫秒的持续时间是有效的。

## Shared Element Transitions

[共享元素](共享元素转换允许我们为跨活动的共享视图之间的转换制作动画，这将创建更愉悦的转换，并让用户更好地了解他们的旅程。)转换允许我们为跨活动的共享视图之间的过渡制作动画，这将创建更愉悦的过渡，并让用户更好地了解他们的旅程。

![在这里，我们第一次活动的视图将扩展到第二个活动中的标题图像](https://ws1.sinaimg.cn/large/006tNc79gy1fronme8jejg307i0dc48i.gif)

在我们的布局中，我们必须使用 transitionName 属性链接任何共享视图 - 这说明了视图之间的过渡关系。下面显示了以上动画的共享视图：

![这些是共享的视图，这意味着他们将在活动过渡期间彼此之间的动画](https://ws3.sinaimg.cn/large/006tNc79gy1fronmitr4jj30go0a8gm2.jpg)

为了在这两者之间过渡，我们首先声明共享转换名称，通过在我们的 XML 布局中使用 transitionName 属性来完成。

Screen 1)

```xml
<RelativeLayout>
    <LinearLayout>

        <View 
            android:id="@+id/view_shared_transition"
            android:transitionName="@string/transition_view"/>

        <!-- Your other views -->

    </LinearLayout>
</RelativeLayout>
```

Screen 2)

```xml
<LinearLayout>

    <View
        android:id="@+id/view_shared_transition"
        android:transitionName="@string/transition_view"/>

    <View
        android:id="@+id/view_separator"/>

    <TextView
        android:id="@+id/text_detail"/>

    <TextView
        android:id="@+id/text_close"/>

</LinearLayout>
```

一旦完成，我们在活动1）中创建一个包含我们的过渡视图和它的 transitionName 的 Pair 对象。 然后我们把它传递给我们的活动选项实例（ActivityOptionsCompat），所以这两个活动都知道共享组件。 从那里我们开始我们的活动，通过我们的活动选项实例：

```java
Pair participants = new Pair<>(mSquareView, ViewCompat.getTransitionName(mSquareView));

ActivityOptionsCompat transitionActivityOptions = 
        ActivityOptionsCompat.makeSceneTransitionAnimation(
                SharedTransitionsActivity.this, participants);

ActivityCompat.startActivity(SharedTransitionsActivity.this, 
                      intent, transitionActivityOptions.toBundle());
```

![在转换过程中滑动这些视图确实有助于完成转换](https://ws1.sinaimg.cn/large/006tNc79gy1fronmqdeewj30go09pq43.jpg)

那么这就是这两个观点之间的过渡，那么第二个活动中的观点从底部滑落呢？

我很高兴你问！这也很简单，如下所示：

```java
Slide slide = new Slide(Gravity.BOTTOM);
slide.addTarget(R.id.view_separator);
slide.addTarget(R.id.text_detail);
slide.addTarget(R.id.text_close);
getWindow().setEnterTransition(slide);
```

正如你所看到的，我们正在创建一个 Slide 过渡的新实例，为转换添加目标视图并将该幻灯片设置为活动的条目转换。

## 自定义过渡

我们也有能力使用我们目前看过的任何动画 API 来创建我们自己的自定义过渡。 例如，我们可以进一步采用共享元素转换来对变换视图进行变形 - 当我们希望显示对话框（或类似的弹出视图）时，这可能会非常有用，如下所示：

![这个动作有助于引导用户对组件状态的关注](https://ws4.sinaimg.cn/large/006tNc79gy1fronnv7e1wg30ay0hfkjl.gif)

让我们快速看看这里发生了什么：

- 我们首先创建一个 SharedTransition，传入按下的视图和转换名称来引用共享组件
- 接下来我们创建一个 [ArcMotion](https://developer.android.com/reference/android/transition/ArcMotion.html) 实例，这允许我们在两个视图之间转换时创建一个弯曲的运动效果
- 然后，我们扩展 [ChangeBounds](http://developer.android.com/reference/android/transition/ChangeBounds.html) 来创建一个自定义的转换，以使两个形状变形（我们有单独的按钮和 FAB 类）。 在这里，我们重写类的各种方法，以便我们可以动画所需的属性。 我们利用 [ViewPropertyAnimator](http://developer.android.com/reference/android/view/ViewPropertyAnimator.html) 在对话框视图的透明度中进行动画处理，使用 [ObjectAnimator](http://developer.android.com/reference/android/animation/ObjectAnimator.html) 在两个视图颜色和一个 [AnimatorSet ](http://developer.android.com/reference/android/animation/AnimatorSet.html)实例之间动画化，以便我们可以将这两个效果一起动画。

## Animated Vector Drawables

从 API 版本 21（棒棒糖）开始，可以使用 [AnimatedVectorDrawable](http://developer.android.com/reference/android/graphics/drawable/AnimatedVectorDrawable.html) 为 [VectorDrawable](http://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html) 的属性制作动画，以生成动画可绘画。

![现在很容易在drawable上做几种不同的动画](https://ws1.sinaimg.cn/large/006tNc79gy1fronptnzu3g307i0dctbx.gif)

但是，我们如何做到这一点？那么让我们来看看这个：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fronrf52pvg304h02xdhf.gif)

由几个不同的文件组成，我们开始创建我们的两个单独的矢量文件，每个文件有几个属性：

- **Height & Width** - 矢量图像的实际大小
- **Viewport Height & Width** - 声明绘制矢量路径的虚拟画布的大小
- **Group name** - 声明路径所属的组
- **Pivot X & Y** - 声明用于组缩放和旋转的 pivot 
- **Path Fill Color** - 矢量路径的填充颜色
- **Path Data** - 声明用于绘制矢量的矢量路径数据

**注意：**所有引用的属性都存储在一个[通用的字符串文件](https://github.com/hitherejoe/animate/blob/master/app/src/main/res/values/add_remove.xml)中，这有助于保持事物的美观和整洁。

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
        android:height="56dp" 
        android:width="56dp"
        android:viewportHeight="24.0"
        android:viewportWidth="24.0">
    <group
        android:name="@string/groupAddRemove"
        android:pivotX="12"
        android:pivotY="12">
        <path 
            android:fillColor="@color/stroke_color"
            android:pathData="@string/path_add"/>
    </group>
</vector>
```

![从我们的ic_add.xml文件（如下）](https://ws2.sinaimg.cn/large/006tNc79gy1fronrn5yekj30ia0590no.jpg)

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
        android:height="56dp"
        android:width="56dp"
        android:viewportHeight="24.0"
        android:viewportWidth="24.0">
    <group
        android:name="@string/groupAddRemove"
        android:pivotX="12"
        android:pivotY="12">
        <path 
            android:fillColor="@color/stroke_color" 
            android:pathData="@string/path_remove"/>
    </group>
</vector>
```

![从我们的ic_remove.xml文件生成的向量（下面）](https://ws1.sinaimg.cn/large/006tNc79gy1fronrzbx7wj30ia0570md.jpg)

接下来，我们声明“动画矢量可绘制文件”，其中说明了矢量绘图和用于每个可绘制状态的动画（添加或删除）。看着添加到删除动画矢量，我们宣布一个目标：

- 从一个状态到另一个状态的动画
- 动画可绘制的旋转

```xml
<animated-vector
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_add">

    <target
        android:name="@string/add"
        android:animation="@animator/add_to_remove" />

    <target
        android:name="@string/groupAddRemove"
        android:animation="@animator/rotate_add_to_remove" />

</animated-vector>
```

然后，我们需要创建这些目标中引用的每个文件。

## 改变可绘制状态

在我们的 [add_to_remove.xml](https://github.com/hitherejoe/animate/blob/master/app/src/main/res/animator/add_to_remove.xml) 中，我们使用 ObjectAnimator 来使用以下属性在形状之间进行变形：

- **propertyName** - 要动画的属性
- **valueFrom** - 矢量路径的起始值
- **valueTo** - 矢量路径的目标值
- **duration** - 动画的持续时间
- **interpolator** - 用于动画的插值器
- **valueType** - 我们正在动画的值类型

```xml
<objectAnimator
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:propertyName="pathData"
    android:valueFrom="@string/path_add"
    android:valueTo="@string/path_remove"
    android:duration="@integer/duration"
    android:interpolator="@android:interpolator/fast_out_slow_in"
    android:valueType="pathType" />
```

## 旋转形状

我们采用类似的方法来旋转形状，使用旋转属性和值来代替：

```xml
<objectAnimator
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:propertyName="rotation"
    android:valueFrom="-180"
    android:valueTo="0"
    android:duration="@integer/duration"
    android:interpolator="@android:interpolator/fast_out_slow_in" />
```

动画相反（从删除到添加）工作相同，只是动画值颠倒。

![我们完成的动画矢量绘制看起来不错，不是！](https://ws2.sinaimg.cn/large/006tNc79gy1fronsbg1s3g304h02xdhf.gif)

如果你喜欢这篇文章，那么请点击推荐按钮！

我很想听听你的想法和你使用这些动画的地方 - 请留言或给我发一条推文！

## 阅读

以下是你可能会对此主题感兴趣的一些资源：

(A **big** thanks to the authors!)

- [Meaning Motion Droidcon London 2015](https://photos.google.com/share/AF1QipMRnZL6gNbS06fnBNtKffRm9HBaxW8iP6w0L1T4nZYLI6s3wi_l8daT6mq4nwPf-w?key=LThZNmFXUUtmNi04bWlEYmVfcWdPenlvaDdCRU13) — Nick Butcher & Ben Weiss
- [Meaningful Motion on Udacity](https://www.udacity.com/course/viewer#!/c-ud862/l-4969789009)
- [Plaid](https://github.com/nickbutcher/plaid) — Nick Butcher
- [Topeka](https://github.com/googlesamples/android-topeka) — Ben Weiss
- [ioSched](https://github.com/google/iosched) — Google
- [Exploring the new Android Design Support Library](https://medium.com/ribot-labs/exploring-the-new-android-design-support-library-b7cda56d2c32)

Thanks to [Iván Carballo](https://medium.com/@ivanc?source=post_page), [Robert Douglas](https://medium.com/@anucreative?source=post_page), [Jemma Slater](https://medium.com/@jemmsla?source=post_page), and [Kerry O'Brien-Manley](https://medium.com/@kerryobrien?source=post_page).

