# 如何在 Android 上使用动画

> 原文 (Medium)：[How to animate on Android](https://proandroiddev.com/how-to-animate-on-android-f8d227135613)
>
> 作者：[Irina Galata](https://proandroiddev.com/@igalata13?source=post_header_lockup)

[TOC]

之前我已经描述了如何使用 OpenGL 为原生的 Android 和 React Native 应用创建动画。 这一次我想讲述一些不同的工具，它们可能对任何困难的 Android 动画及其优缺点有所帮助。 这篇文章不是一个教程或者类似的东西; 它只是我关于 Android 中动画的一些想法和建议的一小部分。 

## 我不使用的工具

然而，我首先想从工具开始，这些工具我并不使用，也不知道为什么。 其中一些我在开源动画中使用过，但是我不想进一步使用它们。 当然，这只是我的主观意见，并不是要求重构你最近的动画:) 

### ObjectAnimator

```kotlin
ObjectAnimator.ofFloat(image, "x", 0f, 300f).apply {
    duration = 1000
    start()
}
```

上面的代码片段显示了 ObjectAnimator 的使用。 在这种情况下，我的图像的 x 属性通过反射而改变，这不是一个好主意的动画。 反射机制非常好，但是对于简单的动画来说，有点过于开销了。 当然，我们可以通过使用属性而不是硬编码属性名称来解决这个问题。 

```kotlin
ObjectAnimator.ofFloat(image, View.X, 0f, 300f).apply {
    duration = 1000
    start()
}
```

现在，使用适当的 setter 直接更改属性值而不反射。 所以现在我们可以看到 ObjectAnimator 的第一个小缺点ーー你需要为每个要修改的自定义属性创建属性，以避免使用反射。 

另一个问题是，你需要一个新的 ObjectAnimator 对于你想要修改的每个视图，因为它不支持同时更改几个对象。 

无论如何，值得一提的是 ObjectAnimator 被广泛用于 AnimatedVectorDrawable 中动画 SVG，因为它能够动画任何类型的属性。在我看来，在任何其他情况下，动画都有更好的解决方案。

**动画及其子类**

```kotlin
image.startAnimation(TranslateAnimation(0f, 300f, 0f, 0f).apply { 
    duration = 1000
})
```

Animation 是 TranslateAnimation，RotateAnimation，AlphaAnimation，ScaleAnimation 和 AnimationSet 的抽象父类。

它们可以对单视图属性进行动画处理，在任何其他情况下，动画子类都需要设置几个实例来播放 AnimatiionSet，就像 ObjectAnimator 那样。 另一个很大的缺点是，你只能对基本属性进行动画，比如 rotation 、scale 、alpiha 和 position (例如，不是背景颜色) ，而这些工具仅限于 View 的子类。 如果你使用动画的话，你将面临的另一个问题是它只会激活视图的像素，而不是视图本身，例如，你将 TransitionAnimation 应用到你的对象上，但是如果不是为了指定不同的行为，它在之前的位置仍然是可点击的。 

目前，动画的唯一合理使用是活动或片段之间的转换。

### ViewPropertyAnimator

为了替代 ObjectAnimator，创建了 ViewPropertyAnimator，它适用于由于优化了 invalidate ( ) 方法调用而同时进行的修改，这毫无疑问是个好消息。 它是一个非常好的工具，可以使一个视图的几个属性并行运行。 

```kotlin
image.animate().apply { 
    duration = 1000
    x(300f)
    y(150f)
    alpha(0.5f)
    start()
}
```

看起来好多了，不是吗？ 但是通常动画需要在同一时间动画几个视图，而不仅仅是视图，而是其他对象，所以它仍然不够好。 

## 我使用的工具

在阅读前面的部分时，你可能会认为我对这些工具过于苛刻。但在大多数情况下，它们可以很容易地被 ValueAnimator 取代。

### ValueAnimator

```kotlin
ValueAnimator.ofFloat(0f, 300f).apply {
    duration = 1000
    addUpdateListener {
        image.x = it.animatedValue as Float
        anotherImage.y = interpolate(100f, 500f, it.animatedFraction)
    }
    start()
}
fun interpolate(a: Float, b: Float, f: Float) = a + f * (b - a)
```

所以，ValueAnimator 允许我们使用其中的一个实例在同一时间对任何类型的任何对象进行动画处理。 你不仅可以使用动画值，还可以使用分数来插值其他两个值，而不需要创建一个新的 ValueAnimator 实例。 太好了！ 

![动画由Marcus J Potter使用ValueAnimator - UX概念实现](https://cdn-images-1.medium.com/max/800/1*6Na3PAlROqChQdPujN1-wA.gif)

上述动画的两个部分（录制按钮和倒计时）都是使用 ValueAnimator 实现的。

看一看使用 ValueAnimator 是 [Yalantis](http://yalantis.com/) 创建的另一个动画及其[源代码](https://github.com/Yalantis/SearchFilter)：

![使用ValueAnimator实现动画](https://cdn-images-1.medium.com/max/800/1*gxgp-u08B4RTUF2lBrfpFw.gif)

何时使用： 1.对于简单和中等难度的动画，可以将其转换为线性函数或任何其他数学函数。 

### Pzhysics-based animations

最近，一个新的 Android 动画工具被引入开发者 - [基于物理的动画](https://developer.android.com/topic/libraries/support-library/preview/physics-based-animation.html)。 一个伟大的事情，这有助于使物体移动物理化，似乎没有像简单的动画物理引擎的开销。 它由两个主要类组成-- SpringAnimation 和 FlingAnimation。

使用 SpringAnimation，你可以使你的视图像一个具有指定阻尼，刚度和最终位置的弹簧一样移动：

```kotlin
SpringAnimation(image, DynamicAnimation.TRANSLATION_Y, 700f).apply {
    spring.stiffness = 40f 
    spring.dampingRatio = 0.2f 
}.start()
```

FlingAnimation 有助于创建更平滑，更可行的 View 的投掷运动。它的起始速度和摩擦都可以定制。

```kotlin
FlingAnimation(image, DynamicAnimation.TRANSLATION_X).apply {
    startVelocity = 100 // pixels per second
    friction = 0.5f 
}.start()
```

何时使用： 1.对于动画来说，它需要物体以一种物理上现实的方式运动，而不需要与其他对象(如碰撞)进行复杂的交互 

### Canvas

正如我之前提到的，有时候，仅仅对 View 的子类进行动画化是不够的。 要自己绘制所有的东西，你需要使用 Canvas，它可以在任何视图的 onDraw ( ) 方法中访问。 它允许绘制任何东西，从简单的圆圈到贝塞尔曲线或文本。 

```kotlin
path.apply {
    moveTo(0f, 0f)
    lineTo(width.toFloat(), 0f)
    lineTo(width.toFloat(), height.toFloat())
    lineTo(0f, height.toFloat())
    quadTo(0f, height / 2f, 150f, 0f)
}
canvas?.drawPath(path, paint)
```

路径和画图是 Canvas 的辅助类，它们分别包含几何路径和样式数据。

查看使用 Canvas 和 ValueAnimator 功能创建的动画：

![使用Canvas和ValueAnimator实现动画](https://cdn-images-1.medium.com/max/800/1*LLjRWwtx1McVSAgPlXM2lA.gif)

[**Yalantis/JellyToolbar**](https://github.com/Yalantis/JellyToolbar)

Canvas 是一个伟大的工具，它的工作速度相当快，但对于小范围。如果你尝试使用它来绘制整个屏幕，你会注意到每个帧都被绘制超过16毫秒，这会导致动画故障。

何时使用： 1. 对于更复杂的动画，这是不可能的 / 很容易作为视图组合表示的。 

### OpenGL

OpenGL 是 Android 动画中的重炮。当 Canvas 无法应付重要的绘图区域时，它会正常工作。

OpenGL 的使用正在逐年扩大 - 游戏，2D 和 3D 动画，照片和视频效果，增强现实和虚拟现实应用程序。不要失去时间，今天就开始学习 OpenGL！

要了解 OpenGL 的基础知识并查看示例动画，请查看我的文章：[How to Create a Bubble Selection Animation on Android](https://proandroiddev.com/how-to-create-a-bubble-selection-animation-on-android-627044da4854)

另外，我强烈建议阅读本书以更熟悉 OpenGL：

[OpenGL ES 2 for Android: A Quick-Start Guide (Pragmatic Programmers)](https://www.amazon.com/OpenGL-Android-Quick-Start-Pragmatic-Programmers/dp/1937785343)

这个工具可以方便的检查你的着色器，或与你的朋友或同事分享：[GLSL Shader Editor](http://thebookofshaders.com/edit.php)

何时使用： 1. 对于巨大且不断渲染动画  2. 对于3D动画  3. 对于复杂转换的动画

### Physics engines

要创建物体间相互作用的难以置信的动画，你肯定需要使用一个物理引擎。 

我为自己选择了 [Box2D](http://box2d.org/) 。 这是一个 C++ 库，在其他编程语言中有很多端口，如果你熟悉其中的一个，你可以使用任何一个。 这是非常轻量级的，并不需要你做任何架构的改变（与 [LibGDX](https://github.com/libgdx/libgdx) 相反，这对游戏来说是相当不错的）。 此外，它的社区是相当大的，你可以从任何平台和编程语言独立的问题找到答案。

你可以在 Android 上使用原始本机库或称为 [JBox2D](https://github.com/jbox2d/jbox2d) 的 Java 端口。

![使用OpenGL和JBox2D实现动画](https://cdn-images-1.medium.com/max/800/1*vyvr3oBPc5976CVoIjDXog.gif)

何时使用： 1. 对于简单的 2D 游戏  2. 对于高级动画来说，这需要对象间复杂的交互 

### Interpolators

要创建动画，你需要学习的一个主要的东西是插值器。 它们被用在每个平台和任何编程语言的动画中。 描述了在时间线上你的动画价值将如何改变。 你可以在 Android SDK 中找到一些准备好的插值器，比如加速插值器，减速插值器，加速度减速插值器，overshotinterator 等。 要了解他们之间的区别，请看这篇文章: 

[Android Interpolators: A Visual Guide](https://robots.thoughtbot.com/android-interpolators-a-visual-guide)

要深入了解一下这篇文章：

[Android Animations Tutorial 5: More on Interpolators](http://cogitolearning.co.uk/?p=1078)

要在 Android 上创建一个，你需要实现 [Interpolator](https://developer.android.com/reference/android/view/animation/Interpolator.html) 接口。

```kotlin
class CustomInterpolator: Interpolator {
    override fun getInterpolation(input: Float): Float {
        // todo return your function value
    }
}
```

之后，你可以使用 ValueAnimator：

```kotlin
ValueAnimator.ofFloat(0f, 300f).apply {
    duration = 1000
    interpolator = CustomInterpolator()
    addUpdateListener {
        image.x = it.animatedValue as Float
        anotherImage.y = interpolate(100f, 500f, it.animatedFraction)
    }
    start()
}
fun interpolate(a: Float, b: Float, f: Float) = a + f * (b - a)
```

我可以推荐你一个非常有用的工具来帮助你创建你的插值器，在这里你可以看到任何插值器的动画例子：

[Interpolator](http://inloop.github.io/interpolator/)

另外，可以使用 [PathInterpolator](https://developer.android.com/reference/android/view/animation/PathInterpolator.html) 在运行时通过所需贝塞尔曲线的控制点生成自定义插值器：

这可以取代为棒棒糖以上代码：

```
interpolator = PathInterpolatorCompat.create(0f, 0.3f, 0.1f, 0.2f)
```

总之，我想说的是，如何在 Android 上创建动画并没有一个通用的工具或者建议，这个解决方案依赖于很多情况 - 比如组件的难度，时间资源的可用性等 在选择仪器之前，每次都要权衡利弊。

现在看起来就是这样。感谢你阅读我的文章，并享受动画的乐趣！  :)

