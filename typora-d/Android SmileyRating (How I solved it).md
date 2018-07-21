# Android SmileyRating 库

> 原文 (Medium)：[Android SmileyRating (How I solved it?)](https://medium.com/mindorks/android-smileyrating-how-i-solved-it-9b5ee30f2c34)
>
> 作者：[Sujith Niraikulathan](https://medium.com/@sujith.niraikulathan?source=post_header_lockup)

[TOC]

![](https://ws1.sinaimg.cn/large/006tNc79gy1frosb8brttg30fz05phdt.gif)

在这里，我要写下我如何创建 [SmileyRating](https://github.com/sujithkanna/SmileyRating) 库以及我遇到的问题。在此之前，我想感谢 [@Yousef Kama](https://medium.com/@yousefkama/react-native-ui-challenge-1-42db390905c) 根据 [BillLabus](https://dribbble.com/blabus) 的设计分享他的工作经验。

## SVG -> VectorDrawables -> Canvas

开始我学习 SVG 来实现这个动画，因为我们可以避免使用图像和简单地绘制路径来创建简单的形状。但在 android 中，如果不使用第三方库，我不能使用 SVG。其次，Android 有另一种名为 VectorDrawable 的图像类型。我开始探索它并碰到这篇文章（[链接](https://lewismcgeary.github.io/posts/animated-vector-drawable-pathMorphing/)），它帮助我理解路径变形动画。这还不够。即使我可以创建不同的表情并在它们之间变化，我也无法控制动画。我只能开始和停止它。我无法在特定帧中暂停动画。所以我决定使用 Java（Android）本地画布 API 编写它。

## Mouth

第一步是创建一个嘴巴。所以根据链接，我向你介绍了路径变形动画，“如果要将一个对象变形为另一个对象（例如：线到曲线），它应该拥有相同数量的命令和相同数量的参数和相同的命令序列顺序。 

例如：线到曲线

![](https://ws2.sinaimg.cn/large/006tNc79gy1frosbkzxs0g30go062arg.gif)

在这个例子中，我刚刚使用 Android 画布创建了贝塞尔曲线。贝塞尔曲线有两个实际点和两个控制点。

以红色绘制的两个圆圈是控制点，黑色圆圈是实际点。在这里，当我向下移动控制点时，曲线变成一条线，当我将控制点向上移动时，曲线再次形成。现在，如果你看到这种上下颠倒，你会知道我如何实现嘴型动画。

## 加入曲线

绘制贝塞尔曲线似乎很简单。但问题是，我不得不保持弯曲的边缘来绘制嘴巴。使用一条曲线，这是不可能的，使用两条曲线很难保持曲线的边缘，所以我想出了将四条曲线连接在一起的想法。这时我发现了一个关于贝塞尔曲线（[链接](https://www.youtube.com/watch?v=Qu-QK3uoMdY)）的好视频。如果组合贝塞尔曲线，则必须确保控制点的实际点相同。（如: 曲线一的端点和曲线二的起点在组合时变得相同 ）应该彼此相反，并且与实际点相距相同 。例如，如果一个控制点与实际点的距离为90°和30个像素，则相反的控制点必须与实际点的距离为270°和30个像素。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frosh036t9g30b40b47q6.gif)

上图是连接在一起的两条曲线的例子。在这里，第一条曲线的 EndControl 点和曲线2的 StartControl 点彼此相反，并以关节(两个曲线交点)作为中心点。只要将这两个控制点保持在相对的位置，并且离接合点的距离相同，曲线就会是连续的。 像这样，我把四条曲线连在一起，形成了笑脸的嘴。 

## 笑脸模型

我面临的下一个问题是管理绘制嘴巴所需的所有坐标（笑脸：TERRIBLE，BAD，OKAY，GOOD ，GREAT）。共有4条曲线（左，上，右和底部），有12个坐标。于是我创建了一个模型，减少了创建笑脸的复杂性。基本的想法是，在每个笑脸中，左侧总是看起来像右侧，但变化不大。该模型中有三种创建状态。

1. MIRROR
2. MIRROR_PARTIAL_INVERSE
3. MIRROR_INVERSE

### Mirror

在这种类型中，我只需提供笑脸的中心点（x，y）和左侧曲线的坐标（start，end，startControl 和 endControl 点）。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frosh4antsj30e709faa2.jpg)

在这个图像中，白色点代表嘴巴区域的中心，黑色圆圈代表控制点，蓝色圆圈代表实际点，绿色圆圈是为要形成和连接的贝塞尔曲线创建的对应点。 蓝色虚线代表左侧曲线。 现在我必须告诉我的模型通过镜像我们现有的坐标来创建右侧曲线，以获得正确的侧面曲线和相反的坐标点。 

![](https://ws1.sinaimg.cn/large/006tNc79gy1froshb8zucj30e709faa2.jpg)

如果仔细观察，我们现在可以获得所有坐标以创建顶部曲线和底部曲线。是的，过去两张照片中的绿色点是控制点，蓝色点是顶部和底部曲线的实际点。让我们形成底部曲线，看看它看起来如何。 

![](https://ws1.sinaimg.cn/large/006tNc79gy1froshdj6r6j30fa0a5t8r.jpg)

同样的方式，顶部曲线也使用由该模型形成的坐标形成。通过调整左侧曲线，我可以创建不同形式的表情。

例如：GOOD 笑脸

![](https://ws2.sinaimg.cn/large/006tNc79gy1froshgatpqj30cm08dq2w.jpg)

### MIRROR_PARTIAL_INVERSE

该模型仅用于一个笑脸（OKAY）。这个笑脸与 MIRROR 几乎相同，只不过多了一个功能。 在这里，右边也是自动形成的镜像左侧曲线，但右侧也将完全翻转垂直。 

![](https://ws4.sinaimg.cn/large/006tNc79gy1froshio0q3j30b707hmx2.jpg)

你可以清楚地看到，左边是镜像和倒置在右边。 由于左右两个控制点是相互平行形成的，所以顶部和底部的曲线被压平。 

### MIRROR_INVERSE

这也与 MIRROR 和 MIRROR_PARTIAL_INVERSE 类似。左曲线将被镜像，此时整个笑脸的嘴部将被垂直翻转。所以基本上，如果你有良好的笑脸左曲线的坐标，那么你可以在这里传递给坏笑脸。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froshlhif3j30c3080glk.jpg)

你可以看到，这完全像垂直翻转的好笑脸。

## Animate Smileys

动画是这里最简单的部分，你所要做的就是用0到1的分数来评估微笑坐标的值。 

```java
IntEvaluator evaluator = new IntEvaluator();
float fraction = 0; // 0 to 1
int leftCurvePointX = evaluator.evaluate(fraction, greatLeftStartX, goodLeftPointX);
```

此示例将计算要绘制的计算出的笑脸的 leftCurvePointX 值。通过结合所有的笑脸，你会得到像这样的变形动画。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froskny945g30gg07chdu.gif)

## Eyes

眼睛是用弧线创造的，因为笑脸中的眼睛也应该表现出情绪。因此，当你使用 SVG 或 Canvas 绘制路径时，如果在绘制弧后关闭路径，它会尝试通过从 startPoint 到弧的终点画一条线来关闭弧线。在这个 SmileyRating 中，只有三种眼睛反应，

![](https://ws1.sinaimg.cn/large/006tNc79gy1froslscwfvj30m806tjrd.jpg)

在上面的图片中，我已经显示了左眼的所有三种反应。像往常一样，右眼是左图像的镜子。在这张照片中，前两只眼睛看起来像一个切成圆形的圆圈，但它实际上是一个半圆弧，它是闭合的。不要让它动起来，看看它的外观。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froso4at36g30bg06k7wh.gif)

这两条虚线显示了弧线的起始和扫描角度。 你只需要改变电弧的起动和扫描角度，旋转电弧来创建这个动画。 

下面提供了库的链接：[[PR](https://github.com/sujithkanna/SmileyRating)]

## 阅读

<https://medium.com/@yousefkama/react-native-ui-challenge-1-42db390905c#.pg5pscewe>

<https://dribbble.com/shots/2790473-Feedback>

<https://www.youtube.com/watch?v=Qu-QK3uoMdY>

<https://lewismcgeary.github.io/posts/animated-vector-drawable-pathMorphing/>

