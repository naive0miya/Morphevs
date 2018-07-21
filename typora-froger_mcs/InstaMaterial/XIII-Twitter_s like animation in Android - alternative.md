# InstaMaterial 概念(第13部分) - 推特 Android 的喜爱动画

>原文 (mirekstanek.online) ： [Twitter's like animation in Android - alternative](https://mirekstanek.online/twitters-like-animation-in-android-alternative/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

前段时间 Twitter 推出了心形—取代星形的图标，用现代化的动画改变他们的状态。[Twitter](https://twitter.com/Twitter/status/661558661131558915?ref_src=twsrc%5Etfw&ref_url=http%3A%2F%2Ffrogermcs.github.io%2Ftwitters-like-animation-in-android-alternative%2F)

虽然心形符号更具普遍性和表现力，但今天我们将尝试使用旧的星形图标来重现新的动画，以替代现实。我们的工作效果看起来像这样（它比动画 GIF 快一点）：
![|center](http://frogermcs.github.io/images/22/button_anim.gif)

<iframe width="560" height="315" src="https://youtu.be/EdZjYTbRNuA" frameborder="0" allowfullscreen></iframe>

实现这个（和原始的心形）动画的最简单的方法是使用[帧动画](https://www.bignerdranch.com/blog/frame-animations-in-android/)，我们将尝试用更灵活的解决方案实现它 - 通过手工绘制并用 ObjectAnimator 进行动画处理。 这篇文章只是一个快速的概述，没有深入的技术细节



## 实现
我们将创建一个名为 LikeButtonView 的新视图，该视图将构建在 FrameLayout 之上，该 FrameLayout 承载三个子视图 - CircleView 显示星形图标下方的圆形，ImageView（我们的星形）以及 DotsView 呈现点在我们按钮周围浮动的点。

**CircleView**
![|center](http://frogermcs.github.io/images/22/circle_anim.gif)
这个视图负责在星形图标下面画一个大圆圈。它可以更容易实现（通过 xml 和 \<shape android：shape =“oval”>），但在这种情况下，我们应该关心我们的按钮下面的背景颜色。

我们的实现在画布上绘制圆圈：
```java
Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
	tempCanvas.drawColor(0xffffff, PorterDuff.Mode.CLEAR);
    tempCanvas.drawCircle(getWidth() / 2, getHeight() / 2, outerCircleRadiusProgress * maxCircleSize, circlePaint);
    tempCanvas.drawCircle(getWidth() / 2, getHeight() / 2, innerCircleRadiusProgress * maxCircleSize, maskPaint);
    canvas.drawBitmap(tempBitmap, 0, 0, null);
}
```
每个帧都从清除整个画布开始，通过 CLEAR 模式绘制颜色。然后视图根据给定的进度绘制内外圈（分别为他们两个）。

内圈使用这样定义的遮罩颜料：
```java
maskPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.CLEAR));
```

这意味着我们的圈子将在我们的外圈内部创造一个透明的洞。
我们的视图使用 tempCanvas 和 tempBitmap 在这里定义：
```java
Override
protected void onSizeChanged(int w, int h, int oldw, int oldh) {
    super.onSizeChanged(w, h, oldw, oldh);
    maxCircleSize = w / 2;
    tempBitmap = Bitmap.createBitmap(getWidth(), getWidth(), Bitmap.Config.ARGB_8888);
    tempCanvas = new Canvas(tempBitmap);
}
```
我们需要做到这一点真正的透明度。否则，我们的内圈将显示窗口颜色。

对于技术熟练的人来说，还有一件事 - 我们的外圈根据当前的进度改变颜色。它由 [ArgbEvaluator](http://developer.android.com/reference/android/animation/ArgbEvaluator.html) 类完成，它根据给定的分数转换两种颜色：
```java
private void updateCircleColor() {
    float colorProgress = (float) Utils.clamp(outerCircleRadiusProgress, 0.5, 1);
    colorProgress = (float) Utils.mapValueFromRangeToRange(colorProgress, 0.5f, 1f, 0f, 1f);
    this.circlePaint.setColor((Integer) argbEvaluator.evaluate(colorProgress, START_COLOR, END_COLOR));
}
```
CircleView 代码的其余部分只是一个实现。完整的源代码可以在这里找到：[CircleView](https://github.com/frogermcs/LikeAnimation/blob/master/app/src/main/java/frogermcs/io/likeanimation/CircleView.java)。

**DotsView**
![|center](http://frogermcs.github.io/images/22/dots_anim.gif)
这个视图将会在我们的星形图标周围绘制点。与 CircleView 相同，它将使用 onDraw () 来执行此操作：
```java
@Override
protected void onDraw(Canvas canvas) {
    drawOuterDotsFrame(canvas);
    drawInnerDotsFrame(canvas);
}

private void drawOuterDotsFrame(Canvas canvas) {
    for (int i = 0; i < DOTS_COUNT; i++) {
        int cX = (int) (centerX + currentRadius1 * Math.cos(i * OUTER_DOTS_POSITION_ANGLE * Math.PI / 180));
        int cY = (int) (centerY + currentRadius1 * Math.sin(i * OUTER_DOTS_POSITION_ANGLE * Math.PI / 180));
        canvas.drawCircle(cX, cY, currentDotSize1, circlePaints[i % circlePaints.length]);
    }
}

private void drawInnerDotsFrame(Canvas canvas) {
    for (int i = 0; i < DOTS_COUNT; i++) {
        int cX = (int) (centerX + currentRadius2 * Math.cos((i * OUTER_DOTS_POSITION_ANGLE - 10) * Math.PI / 180));
        int cY = (int) (centerY + currentRadius2 * Math.sin((i * OUTER_DOTS_POSITION_ANGLE - 10) * Math.PI / 180));
        canvas.drawCircle(cX, cY, currentDotSize2, circlePaints[(i + 1) % circlePaints.length]);
    }
}
```
点是基于由数学支持的 currentProgress 绘制的，说实话，很难在这里指出一些有趣的东西（从 Android SDK 的角度来看）。相反，这里有几个数学相关的东西：
- 圆点被安排在隐形圆圈上 - 它们的位置由以下决定：
```java
int cX = (int) (centerX + currentRadius1  Math.cos(i * OUTER_DOTS_POSITION_ANGLE * Math.PI / 180));
int cY = (int) (centerY + currentRadius1 * Math.sin(i * OUTER_DOTS_POSITION_ANGLE * Math.PI / 180)); 
```
什么意思：每个 OUTER_DOTS_POSITION_ANGLE（51度）设置点。
- 每个点都有自己的颜色动画：
```java
private void updateDotsPaints() {
    if (currentProgress < 0.5f) {
        float progress = (float) Utils.mapValueFromRangeToRange(currentProgress, 0f, 0.5f, 0, 1f);
        circlePaints[0].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_1, COLOR_2));
        circlePaints[1].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_2, COLOR_3));
        circlePaints[2].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_3, COLOR_4));
        circlePaints[3].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_4, COLOR_1));
    } else {
        float progress = (float) Utils.mapValueFromRangeToRange(currentProgress, 0.5f, 1f, 0, 1f);
        circlePaints[0].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_2, COLOR_3));
        circlePaints[1].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_3, COLOR_4));
        circlePaints[2].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_4, COLOR_1));
        circlePaints[3].setColor((Integer) argbEvaluator.evaluate(progress, COLOR_1, COLOR_2));
    }
}
```
这意味着点的颜色在3个值之间变动，范围分别为 [0,0.5] 和 [0.5,1]。我们再次使用 ArgbEvaluator 使其平滑。

其余的很简单。这个类的完整源代码可以在这里找到：[DotsView](https://github.com/frogermcs/LikeAnimation/blob/master/app/src/main/java/frogermcs/io/likeanimation/DotsView.java)

**LikeButtonView**
我们的最终视图组由 CircleView，ImageView 和 DotsView 组成。
```java
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android"
       android:layout_width="match_parent"
       android:layout_height="match_parent">

    <frogermcs.io.likeanimation.DotsView
        android:id="@+id/vDotsView"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:layout_gravity="center"/>

    <frogermcs.io.likeanimation.CircleView
        android:id="@+id/vCircle"
        android:layout_width="80dp"
        android:layout_height="80dp"
        android:layout_gravity="center"/>

    <ImageView
        android:id="@+id/ivStar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:src="@drawable/ic_star_rate_off"/>

</merge>
```
我们使用了[merge 标签](http://developer.android.com/training/improving-layouts/reusing-layouts.html#Merge)，这有助于消除多余的视图组。 LikeButtonView 本身是 FrameLayout，所以不需要两次。

我们的最终视图动画由 AnimatorSet 一起播放的几个较小的动画组成：
```java
@Override
public void onClick(View v) {
    //...

    animatorSet = new AnimatorSet();

    ObjectAnimator outerCircleAnimator = ObjectAnimator.ofFloat(vCircle, CircleView.OUTER_CIRCLE_RADIUS_PROGRESS, 0.1f, 1f);
    outerCircleAnimator.setDuration(250);
    outerCircleAnimator.setInterpolator(DECCELERATE_INTERPOLATOR);

    ObjectAnimator innerCircleAnimator = ObjectAnimator.ofFloat(vCircle, CircleView.INNER_CIRCLE_RADIUS_PROGRESS, 0.1f, 1f);
    innerCircleAnimator.setDuration(200);
    innerCircleAnimator.setStartDelay(200);
    innerCircleAnimator.setInterpolator(DECCELERATE_INTERPOLATOR);

    ObjectAnimator starScaleYAnimator = ObjectAnimator.ofFloat(ivStar, ImageView.SCALE_Y, 0.2f, 1f);
    starScaleYAnimator.setDuration(350);
    starScaleYAnimator.setStartDelay(250);
    starScaleYAnimator.setInterpolator(OVERSHOOT_INTERPOLATOR);

    ObjectAnimator starScaleXAnimator = ObjectAnimator.ofFloat(ivStar, ImageView.SCALE_X, 0.2f, 1f);
    starScaleXAnimator.setDuration(350);
    starScaleXAnimator.setStartDelay(250);
    starScaleXAnimator.setInterpolator(OVERSHOOT_INTERPOLATOR);

    ObjectAnimator dotsAnimator = ObjectAnimator.ofFloat(vDotsView, DotsView.DOTS_PROGRESS, 0, 1f);
    dotsAnimator.setDuration(900);
    dotsAnimator.setStartDelay(50);
    dotsAnimator.setInterpolator(ACCELERATE_DECELERATE_INTERPOLATOR);

    animatorSet.playTogether(
            outerCircleAnimator,
            innerCircleAnimator,
            starScaleYAnimator,
            starScaleXAnimator,
            dotsAnimator
    );

    //...

    animatorSet.start();
}
```
这是关于适当的时间和插值器。
![|center](http://frogermcs.github.io/images/22/touch_anim.gif)
我们的 LikeButtonView 也对触摸事件作出反应（使用缩放动画）：
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            ivStar.animate().scaleX(0.7f).scaleY(0.7f).setDuration(150).setInterpolator(DECCELERATE_INTERPOLATOR);
            setPressed(true);
            break;

        case MotionEvent.ACTION_MOVE:
            float x = event.getX();
            float y = event.getY();
            boolean isInside = (x > 0 && x < getWidth() && y > 0 && y < getHeight());
            if (isPressed() != isInside) {
                setPressed(isInside);
            }
            break;

        case MotionEvent.ACTION_UP:
            ivStar.animate().scaleX(1).scaleY(1).setInterpolator(DECCELERATE_INTERPOLATOR);
            if (isPressed()) {
                performClick();
                setPressed(false);
            }
            break;
    }
    return true;
}
```
就这样。 😃正如你所看到的，这里没有魔法，但最终的效果可以非常好。所以现在怎么办？让我们的应用程序更加高兴。

## 示例代码

 所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/LikeAnimation/)上可用。 






