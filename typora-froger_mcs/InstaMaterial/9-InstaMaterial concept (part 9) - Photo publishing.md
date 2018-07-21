# InstaMaterial 概念(第9部分) - 照片发布

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 9) - Photo publishing](https://mirekstanek.online/instamaterial-concept-part-9-photo-publishing/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的最后一部分，展示了使用 [Material Design 概念的 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q) 的 Android 实现。 今天，我们将通过创建最后一个元素（PublishActivity 和 SendingProgressView）来完成我们的项目。这个功能介于概念视频的第41和第49秒之间。

从今天发布的代码构建的 APK 文件可以在[这里](https://github.com/frogermcs/frogermcs.github.io/raw/master/files/10/InstaMaterial-release-1.0.1-2.apk)找到。是的，这是 InstaMaterial 的最终版本。 😄

以下是在这篇文章中实现的最终效果（介绍整个项目的视频将在下面显示）：

<iframe width="560" height="315" src="https://youtu.be/YgvE3cl34ps" frameborder="0" allowfullscreen></iframe>

## 概论
这篇文章与之前的文章有些不同。 我不会把注意力集中在所做的每一个改变上。 很多代码是一个简单的模板，另一部分在之前的一篇文章中有描述。 这就是为什么我会试着专注于单一的细节，而不是复杂的变化。

## 上传进度
让我们从当前应用版本中引入的最大变化  SendingProgressView 开始。 这个元素可能是我们项目中自定义的视图。 很好，我们有机会练习一些绘画技巧。

这个视图可以有四种状态之一：
- STATE_NOT_STARTED - 这里不应该画任何东西
- STATE_PROGRESS_STARTED - 这里我们必须绘制与当前加载进度相对应的弧
- STATE_DONE_STARTED - 在这里我们应该动画完成的元素（进入背景和复选标记）
- STATE_FINISHED - 具有完整进度圈和完成背景的静态视图。

以下是实际情况：
![](http://frogermcs.github.io/images/10/progress_states.png)
在开始实施我们的 SendingProgressView 之前，有一个非常重要的绘图规则。 如果要达到最佳性能，则必须将绘图操作和工具初始化分开。 虽然 onDraw () 可以被频繁地调用，但是我们不应该在其中进行任何分配。 分配过程可能会导致垃圾收集，从而导致卡顿（GC 操作导致应用程序冻结）。

进行初始化的两个最佳位置是构造函数和 onSizeChanged () 方法（对于视图的大小而定的对象）。

## 进度条
![|center](http://frogermcs.github.io/images/10/progress.gif)
我们从进度条和 STATE_PROGRESS_STARTED 状态开始。 这是简单的弧形笔画。 所有我们需要准备的是一个 STROKE 风格的油漆（我们只想绘制视图范围，没有填充），反锯齿（因为它是圆形的），给定的颜色和笔画宽度：

```java
private void setupProgressPaint() {
    progressPaint = new Paint();
    progressPaint.setAntiAlias(true);
    progressPaint.setStyle(Paint.Style.STROKE);
    progressPaint.setColor(0xffffffff);
    progressPaint.setStrokeWidth(PROGRESS_STROKE_SIZE);
}
```
这个方法从 SendingProgressView 构造器中调用（我们只准备一次 paint）。

绘图的进度甚至更简单。只需一行代码：
```java
private void drawArcForCurrentProgress() {
    tempCanvas.drawArc(progressBounds, -90f, 360  currentProgress / 100, false, progressPaint);
}
```
progressBounds 参数是矩形，用于填充所有给定的视图大小（并在 onSizeChanged () 中测量）。其余的很简单。这个方法在 onDraw () 里面调用：
```java
@Override
protected void onDraw(Canvas canvas) {
    if (state == STATE_PROGRESS_STARTED) {
        drawArcForCurrentProgress();
    }

	//..

    canvas.drawBitmap(tempBitmap, 0, 0, null);
}
```
也许你想知道为什么我们使用 tempCanvas / tempBitmap 而不是在 onDraw () 参数中给出的画布。这是故意做的，用于掩蔽过程。我们稍后会回到它。

根据我们的项目只是模型，我们必须模拟进度变化。这是动画，这将做到这一点：
```java
private void setupSimulateProgressAnimator() {
    simulateProgressAnimator = ObjectAnimator.ofFloat(this, "currentProgress", 0, 100).setDuration(2000);
    simulateProgressAnimator.setInterpolator(new AccelerateInterpolator());
    simulateProgressAnimator.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            changeState(STATE_DONE_STARTED);
        }
    });
}
```
它也在构造器中初始化。如果您错过了关于 ObjectAnimators 的帖子，simulateProgressAnimator 会在2000毫秒内将 float 值从 0.f 设置为 100.f。进度是通过 setCurrentProgress（float）方法更新的。

我们现在要做的就是调用：
```java
setCurrentProgress(0);
simulateProgressAnimator.start();
```

## 完成动画
现在我们准备“完成”动画，在进度达到 100％ 后立即开始动画。看看我们想要达到什么目的：
![|center](http://frogermcs.github.io/images/10/done-animation.gif)
很简单，对吧？ 只是与复选标记图像的圆形背景。 但是这里有一个重要的细节。 在动画的时候，背景和复选标记都是从底部开始的。 在这个时候我们应该把圆形的视图剪切掉，以避免它们与进度环相交。 这就是为什么我们必须掩蔽过程。 总之，我们必须提供圆形遮罩层，以预定的方式切割动画视图。

要做到这一点，我们有两个建议的方式 - 通过使用着色器或使用 Porter-Duff 混合模式。对于第一个，看看 [Romain Guy’s recipe](http://www.curious-creature.com/2012/12/13/android-recipe-2-fun-with-shaders/)。

在我们的项目中我们将尝试混合模式的方式。

## Porter / Duff  合成和混合模式
简而言之，这个过程是关于组合两个图像的。 PorterDuff 定义了如何基于 alpha 值（[Alpha 合成](http://en.wikipedia.org/wiki/Alpha_compositing)）来合成图像。 只要检查这篇文章：[Porter/Duff Compositing and Blend Modes](http://ssp.impulsetrain.com/porterduff.html)，看看我们有什么可能的混合。

在我们看来，我们想要应用 alpha 蒙版，这就是为什么我们使用 PorterDuff.Mode.DST_IN。在实践中，这种模式将把我们的图像剪切成适用的掩模的精确形状。这是可视化的：
![|center](http://frogermcs.github.io/images/10/masking.png)
为了实现这个效果，我们从 Paints 配置开始：
```java
private void setupDonePaints() {
    doneBgPaint = new Paint();
    doneBgPaint.setAntiAlias(true);
    doneBgPaint.setStyle(Paint.Style.FILL);
    doneBgPaint.setColor(0xff39cb72);

    checkmarkPaint = new Paint();

    maskPaint = new Paint();
    maskPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_IN));
}
```
第一个用于完成背景，第二个用于绘制复选标记位图，第三个用于遮罩。这个方法在我们视图的构造器中被调用。

现在让我们准备我们的遮罩（从 onSizeChanged () 方法调用）：
```java
private void setupDoneMaskBitmap() {
    innerCircleMaskBitmap = Bitmap.createBitmap(getWidth(), getWidth(), Bitmap.Config.ARGB_8888);
    Canvas srcCanvas = new Canvas(innerCircleMaskBitmap);
    srcCanvas.drawCircle(getWidth() / 2, getWidth() / 2, getWidth() / 2 - INNER_CIRCLE_PADDING, new Paint());
}
```
最后让我们为我们完成的动画画框：

```java
@Override
protected void onDraw(Canvas canvas) {
    //...
    } else if (state == STATE_DONE_STARTED) {
        drawFrameForDoneAnimation();
        postInvalidate();
    } else if (state == STATE_FINISHED) {
        drawFinishedState();
    }
    //...
    canvas.drawBitmap(tempBitmap, 0, 0, null);
}

private void drawFrameForDoneAnimation() {
    tempCanvas.drawCircle(getWidth() / 2, getWidth() / 2 + currentDoneBgOffset, getWidth() / 2 - INNER_CIRCLE_PADDING, doneBgPaint);
    tempCanvas.drawBitmap(checkmarkBitmap, checkmarkXPosition, checkmarkYPosition + currentCheckmarkOffset, checkmarkPaint);
    tempCanvas.drawBitmap(innerCircleMaskBitmap, 0, 0, maskPaint);
    tempCanvas.drawArc(progressBounds, 0, 360f, false, progressPaint);
}
```
正如你可以看到，我们正在绘制背景和复选标记，然后是遮罩层。 onDraw () 方法中调用的 postInvalidate () 调度下一帧动画。背景和对号的当前位置来自两个动画：
```java
private void setupDoneAnimators() {
    doneBgAnimator = ObjectAnimator.ofFloat(this, "currentDoneBgOffset", MAX_DONE_BG_OFFSET, 0).setDuration(300);
    doneBgAnimator.setInterpolator(new DecelerateInterpolator());

    checkmarkAnimator = ObjectAnimator.ofFloat(this, "currentCheckmarkOffset", MAX_DONE_IMG_OFFSET, 0).setDuration(300);
    checkmarkAnimator.setInterpolator(new OvershootInterpolator());
    checkmarkAnimator.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            changeState(STATE_FINISHED);
        }
    });
}
```
现在几句关于 tempCanvas 的话。 正如我所说，我们故意使用它。 通过使用 PorterDuff 和混合模式，我们玩 alpha 通道。 而这种方法只适用于由透明度填充的位图。 当我们直接在作为 onDraw () 参数给定的画布上绘制时，目标已经被窗口背景填充。 这就是为什么你可以看到黑色而不是什么（透明度）。

其余的代码非常简单。而不是阅读它只是检查[完整的源代码](https://github.com/frogermcs/InstaMaterial/blob/master/app/src/main/java/io/github/froger/instamaterial/ui/view/SendingProgressView.java) SendingProgressView。

最后结果：
![|center](http://frogermcs.github.io/images/10/progressview.gif)

## 活动堆栈管理
当它显示在[概念视频](https://www.youtube.com/watch?v=ojwdmgmdR_Q)上时，当用户点击完成按钮时，MainActivity 被带到顶端（而不是打开新的 Activity），并且内容被滚动到第一个元素。 这里是简短的片段如何用 Intent 标志实现这个效果：
```java
// PublishActivity.java

private void bringMainActivityToTop() {
    Intent intent = new Intent(this, MainActivity.class);
    intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP | Intent.FLAG_ACTIVITY_CLEAR_TOP);
    intent.setAction(MainActivity.ACTION_SHOW_LOADING_ITEM);
    startActivity(intent);
}
```
- Intent.FLAG_ACTIVITY_SINGLE_TOP - 如果 MainActivity 已经在历史堆栈的顶部运行，则该标志不会重新启动。没有这个标志，第二个删除并重新创建 MainActivity。
- Intent.FLAG_ACTIVITY_CLEAR_TOP 会导致如果正在启动的活动已经在运行，那么不是启动该活动的一个新实例，而是关闭其上的所有其他活动，这个 Intent 将被传递到（现在在上面） 作为一个新的意图旧活动。 在我们的例子中，MainActivity 中的这个方法将被调用：
```java
// MainActivity.java

@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
    if (ACTION_SHOW_LOADING_ITEM.equals(intent.getAction())) {
        showFeedLoadingItemDelayed();
    }
}
```
那么为什么我们需要 FLAG_ACTIVITY_SINGLE_TOP 标志呢？没有它也 MainActivity 将被从堆栈中删除。正如我所说，那么它将被重新创建。

以下是它在实践中的工作原理：
![](http://frogermcs.github.io/images/10/backstack.jpg)
还有...其实这是今天的一切。当然，[最近提交的变化](https://github.com/frogermcs/InstaMaterial/commit/7af4e8c967217c0238957f1d4103cc7ec6155d76)要比今天的帖子中描述的代码更广泛。几乎所有的技巧都在本系列之前的文章中有所描述。

恭喜，我们刚刚完成了 Emmanuel Pacamalan 的 [Material Design 概念的 Instagram](https://www.youtube.com/watch?v=ojwdmgmdR_Q) 的实施。 而且我们证明，在这个宽屏中提供的几乎所有效果都可以在较老的 Android 版本中实现。 谢谢大家的阅读和分享你的想法。 我希望能在未来的项目中见到你！😄

## 示例代码
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。


