# 如何绘制自定义视图？

> 原文 (Medium)：[How to draw a custom view?](https://proandroiddev.com/how-to-draw-a-custom-view-9da8016fe94)
>
> 作者：[Mateusz Budzar](https://proandroiddev.com/@mateuszbudzar)

[TOC]

在 Android SDK 中，我们可以使用许多有用的预定义和现成的视图，如 TextView，Button，CheckBox，ProgressBar 等等。然而，有些情况下，我们必须做的不仅仅是使用其中的一个  - 我们必须创建一个自定义的。 如果视图与 Android 平台提供的视图非常相似，那么我们可以尝试对其进行扩展并以适当的方式对其进行自定义以符合我们的期望。 但是，总有一天，我们将无处可逃ーー最终我们需要从头开始创建一个完全自定义的视图。 

我一想到要扩展 View 类并且自己实现所有的事情，我总是感到害怕。 但事实证明并没有那么难。 让我们一起努力实现这一点。 

想象一下，我们需要创建一个自定义音量栏。 我们要做的第一件事就是创建一个扩展视图的类。 

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo9woltrj30m80100sz.jpg)

只有一个构造函数被重写。 当我们从 xml 文件中创建一个视图时，我们使用的那个。 为了这篇文章的目的，这就足够了。 

现在是时候画一些东西了！我们从一条线开始。

## 如何画线？

要绘制一些东西，我们必须重写 onDraw ( ) [模板方法](http://www.oodesign.com/template-method-pattern.html)。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo9z5xxxj30f2030glj.jpg)

这个方法只给我们一个参数 - [画布](https://developer.android.com/reference/android/graphics/Canvas.html)。我们会在上面绘制。 但是怎么做呢？ 让我们来看看这个 API 是怎样绘制的。 

![](https://ws4.sinaimg.cn/large/006tNc79gy1frooa68narj30m809h0w1.jpg)

画布给出了很多方法来绘制一些东西，比如 drawCircle ( )，drawPath ( )，drawRect ( ) 等等。我们需要一条线，所以我们将使用 drawLine ( )。

![](https://ws1.sinaimg.cn/large/006tNc79gy1frooa8pufyj30m802574n.jpg)

该方法有五个参数：

- startX, startY - 该线应该从哪一点开始，
- stopX, stopY - 该行的结束点，
- paint  - 我们用来画线的颜料。

前四个是非常明显的，最后一个我们将在一分钟内涵盖。首先，我们应该关心警告。

![](https://ws1.sinaimg.cn/large/006tNc79gy1frooac0unnj30rs03lwg1.jpg)

简单地说，我们不应该在 onDraw ( ) 方法中创建任何新的对象实例，因为如果我们分配了大量的新对象，最终会被垃圾收集，我们的 UI 将不会流畅。 我们需要将 [Paint](https://developer.android.com/reference/android/graphics/Paint.html) 对象的创建移出该方法。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frooaf0izrj30m8052wfk.jpg)

最后一件事是将我们的视图添加到布局文件。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frooai9jjnj30m808nju8.jpg)

现在让我们运行我们的项目，看看结果！

![](https://ws4.sinaimg.cn/large/006tNc79gy1frooalqvamj30970gc74i.jpg)

耶！ 我们已经画了一条线但它并不像我们想的那样集中。 实际上，我们的视图占据了整个空间。 可能是因为我们没有说明视图应该占用多少空间。 是时候重写  onMeasure ( ) 方法了。 

![](https://ws2.sinaimg.cn/large/006tNc79gy1frooaq2supj30m80fj0xy.jpg)

首先，我们需要解码视图的模式和大小。为什么？让我们来看看 onMeasure ( ) 的[文档](https://developer.android.com/reference/android/view/View.html#onMeasure%28int,%20int%29)对这些参数的说明。

> 参数：
> widthMeasureSpec - 由父级强加的水平空间要求。 需求使用 View.MeasureSpec 进行编码。
> heightMeasureSpec -由父级强加的垂直空间要求。 需求使用 View.MeasureSpec 进行编码。

父视图以编码形式传递这些参数，所以我们需要首先对它们进行解码 - 这可能与某种优化有关。

那么我们需要检查模式。有三种不同的模式：

- EXACTLY  - 当我们在视图中设置准确的 dp 值（比如 300dp）或者它是 match_parent 时，
- AT_MOST  - 当它是 wrap_content 时，
- UNSPECIFIED -  当它并不重要，一个视图可以像它想要的一样大，因为父母是一个 ScrollView ( )。那么在图像模式中的高度是不相关的 

你可以尝试它，并检查是否是真实的，例如通过在 onMeasure ( ) 方法中添加一些日志并更改布局参数。

所以我们可以看到，如果提供给定的 dp 值，我们就接受给定的 dp 值，否则将采用在 res / values / dimens.xml资源文件中定义的默认值。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frooaucpr3j30m801a3z7.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1frooaxplorj30m803kmy8.jpg)

毕竟，记得调用 setMeasuredDimension ( ) 方法来应用视图尺寸。

现在让我们改变一下参数，让它有一条水平线并运行这个项目！

![](https://ws4.sinaimg.cn/large/006tNc79gy1froob0dx3lj30m802274p.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1froocrj8olj30970gcweo.jpg)

酷！ 再次，现在你应该尝试一些不同的 layout_width 和 layout_height 参数来看看有什么变化。 在“开发人员选项”中打开“显示布局边界”选项，以查看我们的视图正在采取的严格的空间。如果你不熟悉这个选项，你可以在我的关于 margin 和 padding 的文章中找到如何启用它。

[Android Developer Beginner. FAQ #1 — Margin vs Padding](https://proandroiddev.com/android-developer-beginner-faq-1-margin-vs-padding-f5403c81d7e6)

好的，但是我们希望有更粗的线条，那种条形，所以我们可以操纵它的高度和宽度。让我们把这条线改成矩形

## 如何绘制一个矩形？

幸运的是，我们的例子非常简单。我们所需要做的就是将 drawLine ( ) 方法改为 drawRect ( )，并将高度值而不是 0.0F 应用到底部参数。我们也应该重构 linePaint 的名字给 barPaint。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froocw9sogj30m8023t95.jpg)

我们还要将 volume_bar_default_height 更改为 12dp 并运行该项目。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froocxyxklj30970gcglt.jpg)

现在我们想要改变颜色，比方说灰色。 但 drawRectangle ( ) 方法没有颜色参数。 不应该是这样，因为通过 Canvas 我们要说什么来绘制，但是我们使用 Paint 类来说明我们将如何去做。 所以我们将使用我们的 barPaint 来改变颜色。

![](https://ws4.sinaimg.cn/large/006tNc79gy1frood1przaj30d0032a9z.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1frood5iu2rj30970gc74h.jpg)

下一步是绘制一个显示当前音量的圆。

## 如何绘制一个圆圈？

要做到这一点，我们将使用 drawCircle ( ) 方法，我们将创建一个新的 Paint 对象来为圆圈设置不同的颜色。

![](https://ws3.sinaimg.cn/large/006tNc79gy1frooda0elqj30rs02cgmk.jpg)

![](https://ws1.sinaimg.cn/large/006tNc79gy1froodbzfkmj30m8052abe.jpg)

现在我们已经将它居中（我们圆的中心点是条宽的一半和高度的一半），我们设置半径以适应条的高度。

![](https://ws2.sinaimg.cn/large/006tNc79gy1froodewhg7j30970gcjrl.jpg)

现在是时候对音量变化做出反应，并改变圆的位置。

## 如何对音量水平的变化做出反应？

在动态更改视图之前，让我们在创建 MainActivity 视图时设置当前音量级别。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froodlac4oj30m809wtbi.jpg)

我们使用 AudioManager 类来检索音量级别 - 最大音量和当前音量。 Android 有[几个不同的音量流](https://developer.android.com/reference/android/media/AudioManager.html#STREAM_ALARM)，如STREAM_ALARM 用于定义闹铃的音量，STREAM_MUSIC 用于音乐的设置。 按下音量按钮并展开列表时，你可以看到不同的流。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froodomm8kj30fs06gjr9.jpg)

我们使用的是 STREAM_SYSTEM，因为当我们运行我们的应用程序，这是默认的应用程序，当我们点击一个音量按钮时，它将会改变。

让我们看看里面的 calibrateVolumeLevels ( ) 方法。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froods3qsxj30m803gmxz.jpg)

我们保存了参数并调用了将调用 onDraw ( ) 的 invalidate ( ) 方法。在第二个，我们必须做一些计算来正确地绘制圆。

![](https://ws4.sinaimg.cn/large/006tNc79gy1frooduxbtrj30m808zq51.jpg)

为了清理代码，我们已经提取了绘制条和绘制拇指来分离方法。 drawThumb ( ) 实际上是具有新计算的那个。 我们没有改变 *Y* 或半径值，只是 *X* 的值，因为我们想要在水平轴上移动我们的拇指。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froody0tblj30m806pmyq.jpg)

我们所要做的就是把我们的条的宽度除以音量水平，并乘以当前的音量水平。例如，如果我们的音量栏宽度为700，音量级数为7，当前值为5，则方法返回500。

不幸的是，这两个卷值可能为空，因为我们不确定是否调用了 calibrateVolumeLevels ( ) 方法。 我们还必须将值存储在本地值中，因为存在[竞争条件](https://en.wikipedia.org/wiki/Race_condition)的可能性 - volumeLevelsCount 和 currentVolumeLevel 是可变属性，检查后它们可以立即更改为非空值。

让我们看看我们是否正确校准我们的视图。

![](https://ws1.sinaimg.cn/large/006tNc79gy1frooe1rhn2j30970gcaa0.jpg)

它似乎在工作。但不完全如我们想要的那样...在我告诉你这个问题之前，让我们来实现每当音量改变时改变圆的位置。

![](https://ws1.sinaimg.cn/large/006tNc79gy1frooe5239gj30m804ewfy.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1frooe6z6oij30fe03q0sv.jpg)

在 MainActivity 中，我们重写了 dispatchKeyEvent ( ) 方法。 我们不想消耗这个事件，所以返回 false 并且首先调用 super 方法。 感谢音量的改变，我们可以从 AudioManager中获取当前音量。

在 VolumeBarView 上设置当前音量水平非常简单，因为我们只是更新属性并使视图无效。

让我们运行该项目并将音量设置为最小或最大级别。我已经把视野做得更大了，所以我们会更清楚地看到它。

![](https://ws4.sinaimg.cn/large/006tNc79gy1frooe9g22lg307i0dctas.gif)



在边缘，我们只能看到拇指的一半。 那是因为我们正在画圆心，从左边开始ーー在右边，这是同样的问题。 嗯... 我们只需要一半的圆在左边，另一半在右边，差不多。 因此，也许我们应该从宽度中减去圆的大小(体积条的高度) ，而在计算结束时，只需将圆的大小减半？ 让我们来实现这一点，看看是否有帮助。 

![](https://ws1.sinaimg.cn/large/006tNc79gy1frooedachtj30m805975j.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1frooehm1itg307i0dcdip.gif)

现在计算正确完成。我们的自定义音量栏正在工作！ 你可以在 GitHub Repo 上找到整个代码。[[PR](https://github.com/makorowy/customview-volumebar)]

## 结束

实施自定义视图是一开始的挑战。 复杂的计算可以很容易地阻止我们这样做， 这就是为什么在这篇文章中我们实现了一个非常简单的自定义音量栏版本。 当你开始阅读时，你可能并没有想象的那么令人愉快， 但这只是一个开始。 我们已经学会了基本知识，现在我们可以进一步定制它，我鼓励你这样做。 希望这篇文章是有帮助的。