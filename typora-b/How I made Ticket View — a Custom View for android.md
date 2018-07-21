# 我是如何定制 Ticket View —— Android 自定义视图

> 原文 (Medium)：[How I made Ticket View — a Custom View for android](https://android.jlelse.eu/how-i-made-ticket-view-a-custom-view-for-android-20b83b175f8e)
>
> 作者：[Vipul Asri](https://android.jlelse.eu/@vipulasri?source=post_header_lockup)

[TOC]

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frow6j559lj30rs0ag0te.jpg)

在今天的情况下，一切都已经存在。 我遇到了一种情况，就是根据用户界面设计开发一个卷视图。我的第一个想法是搜索 [Android Arsenal](https://android-arsenal.com/) 和 GitHub，看看是否有人已经开发出这种视图，这将减轻我的工作。但不幸的是我没有找到一个，我就像是 “Well Jarvis，我们回到了硬件模式”

我会在这里谈论两件事情：

- 创建卷视图
- 增加高度/阴影（感谢 [Nick Butcher](https://medium.com/@crafty)🙌）

## 卷视图

我用了两种方法才终于做对了。 

1. 初步方法

我的第一个方法是从自定义视图中消除凹形的角，这个方法的问题是当视图超过一定背景或颜色时，没有显示透明角。 因为它显示了有白色背景的凹角，或者我必须明确提供角的背景颜色。  甚至为这个视图增加高度也是难以想象的。所以，我不得不想出一个不同的方法。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frow6n3ivtj30dw07ia9v.jpg)

2. 更好的方法

为了获得透明的凹角，我想创建一个弧形而不是删除最终对我有效的角。因此， 我不得不在宽度中心创建4个凹角和2个凹角来显示分隔线。 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frow6oziayj3074074dfr.jpg)

在安卓系统中，圆弧是按顺时针方向绘制而不是按坐标系顺时针方向绘制的。 绘制弧线的方向可以通过提供负角来逆转。 

开始角度= 0度 

扫描角度= 90度

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frow6qwoprj3074074mx9.jpg)

阴影部分是呈现的弧形部分。 正如你所看到的，它从坐标系所描绘的0度开始，并且在顺时针方向中扫描了90度。 

上面的例子可以用这样的代码实现，它将帮助创建右下角的凹角。

```java
canvas.drawArc(rect, 0, 90, false, paint);
```

你可以在这个详细的博客中了解更多关于 “drawArc” 的方法：[[guide]](https://robots.thoughtbot.com/android-canvas-drawarc-method-a-visual-guide)

我还添加了不同角度的特性: 扇贝(凹角)、圆角和普通角。 卷视图有各种可用的定制：

- 方向：设置分隔线和中心扇贝的方向
- 背景颜色：设置卷的背景颜色
- 扇贝半径：扇贝半径（凹）
- 扇贝位置百分比：设置扇贝和分隔线的位置

和边框颜色，分隔线颜色，分隔线类型和高度。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frow6sszgsj30dc0fuq31.jpg)

> 卷视图样本（无高度/阴影）

## Elevation/Shadow

要将高度或阴影添加到我的视图中，我使用的是线性渐变，但是当我开始实现时，它变得太复杂了，我不得不记住它可能变成的每一个形状。 

所以，我呼吁最好的人（plaid 的创造者） -[Nick Butcher](https://medium.com/@crafty) 的帮助和指导

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frow6wv0lij30m80dan1b.jpg)

他建议我用模糊而不是线性渐变来表示我的视图。我需要他的指导，但他却给我发了一个请求。  那是我从他那里学习的最令人惊奇的时刻。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frow6znmgbj30m805gt9s.jpg)

他不仅给我发了一个加高度的请求，而且还提供了一个更好的方法来画出我的视图。  为了提升画面的高度，一旦我的视图被布置，他就创建了视图的高度和宽度的位图，然后将 20％ 的黑色和 alpha 的颜色应用到画布上。 在这之后，位图变得模糊，形成了高度 / 阴影。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frow71puzgj30dc0fujrl.jpg)

P.S：当前高度/阴影在 API 17+上受支持。我试图增加对 API 15+的支持，但不幸的是它将库的大小从〜12 KB 增加到了 2.2 Mb，因此放弃了针对库的最低 SDK 的想法。

vipulasri/TicketView An Android library to implement TicketView in android with normal, rounded and scallop corners [github.com](https://github.com/vipulasri/TicketView)

