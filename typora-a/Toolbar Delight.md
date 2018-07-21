# Toolbar 使用案例

> 原文 (Medium)：[Toolbar Delight](https://androiduipatterns.com/toolbar-delight-8c5e4500b899)
>
> 作者：[Juhani Lehtimäki](https://androiduipatterns.com/@lehtimaeki)

[TOC]

在这篇文章中，我们解释了如何以及为什么从实现的角度来做我们的 [Social Steps](https://play.google.com/store/apps/details?id=com.socialstepsapp.android) 应用程序自定义工具栏。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo5l0m3pj30b40jrmzg.jpg)

## 设计

在你的用户界面中添加令人愉快的细节是将你的应用推向竞争之上的好方法(当然，假设所有重要的功能都存在并且设计良好)。 

工具栏是 Android 上的一个操场。 我们决定充分利用它在有趣但有意义的动画和状态变化。 

这个功能的设计，就像其他的[社交步骤](https://play.google.com/store/apps/details?id=com.socialstepsapp.android)应用程序一样，由 [Pierluigi Rufo](https://twitter.com/pierluigirufo) 完成。它已经承诺尽快写更详细的设计方面。敬请关注！

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo5pa39aj30m809wad8.jpg)

## 实施

Android 的 UI 框架是非常强大和灵活的。 如果你花时间去学习如何使用它，你就会在你的工具箱中添加一个非常强大的工具。 就我个人而言，我认为本地的 Android 用户界面是目前最强大的原型设计工具。 几乎你的设计师提出的所有东西都可以在几个小时内实现(或者至少创建一个预期特性的近似值)。 

这种灵活性延伸到适当的、可扩展的、可实现的生产准备特性。 在我们的  Social Steps 应用程序中，工具栏是一个显而易见的地方，在这里可以推动应用程序的品牌和用户喜悦方面。 

为了保持可伸缩性，滚动容器在 Android 屏幕上非常普遍。 如此之多以至于谷歌为开发人员引入了特殊的组件，以至于可以为 Android 工具栏添加有趣而有用的行为: AppBarLayout 和 collapsing toolbarlayout。 

有了上述两个组件和一个小的自定义视图，可以在你的工具栏设计中使用魔法。 

### 跟踪滚动事件

Youtube - [Toolbar scrolling animation](https://youtu.be/wyhV5HClXBo)

AppBarLayout.OnOffsetChangedListener

当用户滚动你的主视图（折叠你的工具栏）时，可以使用这个工具来处理事件。 

这段代码在我的主 Activity 中，但是如果你的工具栏被定义在一个 Fragment 中，它也可以工作。

```java
appbarLayout.addOnOffsetChangedListener(object : AppBarLayout.OnOffsetChangedListener {
    internal var scrollRange = -1

    override fun onOffsetChanged(appBarLayout: AppBarLayout,      verticalOffset: Int) {
        //Initialize the size of the scroll
        if (scrollRange == -1) {
            scrollRange = appBarLayout.totalScrollRange
        }


        val scale = 1 + verticalOffset / scrollRange.toFloat()

        toolbarArcBackground.setScale(scale)

        if (scale <= 0) {
            appbarLayout.elevation = toolbarElevation
        } else {
            appbarLayout.elevation = 0f
        }

    }
})
```

这个代码有一个非常简单的责任 --> 以百分比计算滚动容器的当前规模。 这个代码不知道它是用来做什么的，但它只是计算它，并将值传递到我的自定义视图(见下文)。 

啊，还有。 当整个工具栏折叠时，此代码处理工具栏的高度。 

布局很简单（这可能有点乐观）。 我的自定义 ToolbarArcBackground 是这里最繁重的工作。 其余的都是标准的 Android ..除了 NonClickableToolbar 这是需要在这里使折叠工具栏工作。 它什么都不做。

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/rootLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/content_background">


    <android.support.design.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">


        <android.support.design.widget.AppBarLayout
            android:id="@+id/appbarLayout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:elevation="0dp">


            <android.support.design.widget.CollapsingToolbarLayout
                android:id="@+id/collapsing_toolbar"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:fitsSystemWindows="true"
                app:layout_scrollFlags="scroll|exitUntilCollapsed">

                <FrameLayout
                    android:id="@+id/collapsing_content"
                    android:layout_width="match_parent"
                    android:layout_height="160dp"
                    app:layout_collapseMode="pin">

                    <com.socialstepsapp.socialsteps.widget.
                     ToolbarArcBackground
                        android:id="@+id/toolbarArcBackground"
                        android:layout_width="match_parent"
                        android:layout_height="match_parent"
                        android:background=
                        "@color/content_background" />


                </FrameLayout>


                <com.socialstepsapp.socialsteps.widget.
                    NonClickableToolbar
                    android:layout_width="match_parent"
                    android:layout_height="?attr/actionBarSize"
                    android:layout_marginTop="24dp" />


            </android.support.design.widget.CollapsingToolbarLayout>


        </android.support.design.widget.AppBarLayout>

        <android.support.v4.widget.NestedScrollView
            android:id="@+id/scroll_view"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fillViewport="true"
            app:layout_behavior=
            "@string/appbar_scrolling_view_behavior">
<!-- Here's some views of the app logic -->

         </android.support.v4.widget.NestedScrollView>
    </android.support.design.widget.CoordinatorLayout>


    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:layout_marginTop="24dp"
        android:background="#00000000"
        android:elevation="0dp">
<!-- Here's couple of irrelevant views ->
       


    </android.support.v7.widget.Toolbar>
</FrameLayout>
```

### 实现弧

ToolbarArcBackground 自定义视图是魔术发生的地方。 这是 Android 视图的一个相当简单的子类。 因为我们已经有一个组件提供给我们规模(见上文) ，我们需要做的只是找出如何绘制我们想要的东西。 

我尝试了一些不同的方法来把圆弧做好。 我的第一个方法是使用一个路径来切除我的画布的底部。 不幸的是，似乎不可能使路径使用反锯齿，边缘变成锯齿状。 

和很多用户界面细节一样，最好的方法往往是最简单的。 . 也就是说作弊。 

我利用了这样一个事实: 主屏幕的背景是恒定的颜色。 简单的答案是在工具栏内容的其他所有操作之上画一个白色椭圆。 :-) 

为了避免这个问题，我在左边和右边的视界之外稍微画了一个椭圆。 

自定义视图的 setScale 方法只是存储当前值并使内容失效。 

```kotlin
fun setScale(scale: Float) {
    this.scale = if (scale < 0) {
        0f
    } else {
        scale
    }

    invalidate()
}
```

然后 OnDraw 简单地在底部绘制一个合适的椭圆

```kotlin
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
// draw some other stuff here first
    canvas.drawOval(
        (-extendOverBoundary).toFloat(), height - arcSize * scale,    
        (width + extendOverBoundary).toFloat(),
        height + arcSize * scale,
        ovalPaint)
}
```

当用户在工具栏边缘变为直线的过渡点处滚动时，缩放接近0，椭圆完全消失。

### 滚动实现云

云是简单的位图，具有固定的起始位置（虽然未来我不会感到惊讶，他们根据当前的风况移动）。 为了让它们在折叠时从工具栏中移出，我只是简单地预先计算了一个我想要的位置，当工具栏被折叠，剩下的就是简单的乘法运算。

```kotlin
canvas.drawBitmap(cloud1Bitmap, cloud1X + cloud1OffsetX * (1 - scale), cloud1Y + cloud1OffsetY * (1 - scale), bitmapPaint)
```

### 实施日常时间更改

在这个第一个版本中，时间只是基于时间（匹配真正的太阳位置将需要用户的位置，这不是我们想要的许可）。

当然，在附加的视频中，每天的时间是动画的，在发布的版本中，它几乎是静态的。 

为了使事情可靠地工作，我添加了另一个尺度到工具栏视图组件，时间尺度。 这只是一个介于0和1之间的数字，告诉观察太阳或月亮从左到右有多远。 isNight 是一个自我解释。 它定义要使用哪个调色板以及使用哪个天体。

```kotlin
fun setTimeScale(isNight: Boolean, timeScale: Float) {
    this.timeScale  = timeScale.coerceIn(0f, 1f)

    this.isNight = isNight
    invalidate()
}
```

Youtube - [Toolbar, time-of-day animation](https://www.youtube.com/watch?v=Hsg3iOx06dY)

工具栏颜色从几个预定义的颜色点进行调整，并使用 ArgbEvaluator.evaluate ( ) 进行插值，并使用渐变着色器绘制为背景进行绘制。

为了改善色彩效果，我们在进行色彩计算之前，将插值值添加到时间尺度值中。 这使得黄昏和黎明只能在早晨和晚些时候更准确地模仿真实的灯光。 

```kotlin
private fun calculateColour2(): Int {
    return colourEvaluator.evaluate(scale, 
      ContextCompat.getColor(context, 
      R.color.toolbar_gradient_2_noon), calculateColour2Base())
      as Int

}



private fun calculateColour2Base(): Int {

    val interpolatedScale = interpolate(timeScale)

    return if (isNight) {
        when (interpolatedScale) {
            in 0.0f..0.5f -> 
                colourEvaluator.evaluate(interpolatedScale * 2,
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_evening), 
                ContextCompat.getColor(context,
                R.color.toolbar_ gradient_2_midnight)) as Int
            else -> colourEvaluator.evaluate((interpolatedScale - 
                0.5f) * 2, ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_midnight), 
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_morning)) as Int
        }
    } else {
        when (interpolatedScale) {
            in 0.0f..0.5f -> 
                colourEvaluator.evaluate(interpolatedScale * 2, 
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_morning), 
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_noon)) as Int
            in 0.5f..0.75f -> 
                colourEvaluator.evaluate((interpolatedScale - 0.5f) 
                * 4, ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_noon), 
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_noon_evening)) as Int
            else -> colourEvaluator.evaluate((interpolatedScale - 
                0.75f) * 4, ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_noon_evening), 
                ContextCompat.getColor(context, 
                R.color.toolbar_gradient_2_evening)) as Int
        }
    }
}
```

对于晚上颜色，我们增加了一个手动点（0.75f），因为中午和晚上之间的插值颜色看起来很差。

为了确保工具栏在折叠时总是返回到品牌颜色，第二种颜色的渐变也会根据滚动比例向品牌颜色进行插值。

```kotlin
LinearGradient(0f, 0f, scale * width, scale * height, calculateColour1(), calculateColour2(), Shader.TileMode.CLAMP)
```

## 保持可扩展性

所谓的 Android 碎片只对非 Android 开发者造成问题（因为他们不知道更好）。 作为 Android 开发者，我们知道如何处理多种屏幕尺寸。 加上有能力的设计师，我们不必锁定在一个方向的设备或防止安装到平板电脑（或 Chromebook）。

当思考动画、渐变等等时，从一开始就记住了可伸缩性。 以后增加可伸缩性是很困难的。 

这很简单。 不要使用固定资产进行渐变，请使用代码绘制它们。 不要考虑屏幕的宽度，请使用 Android 操作系统提供的值。 尽可能使用百分比，并且每当需要绝对值时，请记住使用 dip。

就是这样。正如你所看到的，在我们的情况下，应用程序不是锁定在一个方向，也不阻止平板电脑安装它。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo5x5itij30m80cignb.jpg)

![在Chromebook上。可能不完美，但它仍然有效！](https://ws3.sinaimg.cn/large/006tNc79gy1froo61bhzej30m80etmyz.jpg)

## 技术总结

通常情况下，构建复杂的外观设计都是要找到一个简单的方法来作弊，但是仍然让事情看起来和设计一模一样，并把问题分解成易于管理的部分。 在这种情况下，使用椭圆形，而不是试图在工具栏上切一个洞，这样可以节省大量的汗水和时间。 

将时间尺度和滚动比例简化为简单的 0-1 范围，使我能够专注于在有限的问题空间中实施，而不必担心外部因素。 事实上，在应用程序中，白天和黑夜的长度不等，因为它们不是必须的。 传递时间与时间尺度变量1-1不匹配。 我们可以很容易地将真正的太阳位置连接到现有的代码上，而不会出现任何问题。 

PS。另外，我们还有很长的路要走。 我刚来 Kotlin。 示例代码的大部分是从 Java 代码转换而来的，之后手动进行了调整。 这些例子不太可能遵循最好的 Kotlin 编程风格，但是这个想法应该被传达。 

PPS。社交步骤是一个免费的（广告）在谷歌 Play 商店的应用程序：

[[Social Step]](https://play.google.com/store/apps/details?id=com.socialstepsapp.android)

试一试，看看自己的自定义工具栏。它也可以在 iOS 上使用！

