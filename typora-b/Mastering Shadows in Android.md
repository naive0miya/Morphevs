# 掌握 Android 系统中的阴影

> 原文 (Medium)：[Mastering Shadows in Android](https://android.jlelse.eu/mastering-shadows-in-android-e883ad2c9d5b)
>
> 作者：[Mert Şimşek](https://android.jlelse.eu/@iammert?source=post_header_lockup)

[TOC]

如果我们想创造更好的应用程序，我相信我们需要遵循材料设计指南。一般而言，材料设计是一个包含光线，材质和投影的三维环境。 如果我们想要在我们的应用程序开发过程中遵循材料设计指南，光影对我们来说非常重要。

我将尝试解释本文中的下列主题。

- 3D in Android
- Depth
- Z value, elevation and Translation Z
- Light
- Button state (Pressed and Resting)
- [Outline](https://developer.android.com/reference/android/graphics/Outline.html)
- Custom Outline with [ViewOutlineProvider](https://developer.android.com/reference/android/view/ViewOutlineProvider.html)

在深入到阴影和光线之前，我想让你们看看我们的环境。 

## 3D是什么？

物质环境是一个三维空间，这意味着所有对象都有 x，y 和 z 维度。 Z 轴垂直于显示平面， 正 Z 轴朝向观察者延伸。 在材料设计的世界里，每个物体都有1个 dp 的厚度。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowccpy8yj30m80e13z4.jpg)

## Android 的深度是什么？

材料设计不同于其他设计指南，因为它具有深度。我们可以说深度定义了视图在用户界面中的重要性。我们可以认为我们的桌子上有一个纸层。 如果我们在上面再放一张纸，我们的眼睛就会感觉到它有一个深度。 

让我们通过材料设计指南中的应用截图来想象它。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowchsdznj31jk0mzdmh.jpg)

让我们看看在屏幕上的元素。

- Screen (Surface layer — 0 depth) 
- Cardviews 
- Appbar Layout 
- Floating Action Button 

每个元素都有一个优先级别。 Cardview 可以在 Recyclerview 中滚动。所以我们可以说我们的第一层是可滚动内容。第二层是 AppbarLayout 。第三层（顶层）是 Floating Action Button。

那么我们如何定义这个顺序呢？我们如何让用户感受到深度？答：Z 值。

## 在安卓系统中，Z 值是什么？

视图的 Z 值有两个组成部分：

- Elevation：静态组件。 
- Translation Z：动态组件用于动画。

我一直想知道高度和偏移之间的区别是什么。 

高度是静态的。所以你不能动态地改变它。如果你想在 Z 轴上动画你的视图（如按下和休息），你需要使用 Translation Z 属性。

偏移Z是动态的 在你的空白项目中，如果你创建一个按钮并按下它，你将会看到阴影在动画中变得更大。 实际上，高度并没有变化。Translation Z 属性正在改变。 Android 正在使用默认状态列表动画更改视图的 Z 属性。

> Z-Value = Elevation + TranslationZ

如果我们改变两个交叉视图的 z 的值。   安卓系统能处理屏幕上的顺序吗？ 是的。给你看看我设计的图表。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowcl7cjnj30rs0fhdgx.jpg)

另一个问题，我们如何看到一个阴影？我们需要什么才能看到阴影。答：我们需要一个灯。

## 安卓系统中的光是什么？

实际上，问题不是什么。 问题是在哪里。 

我认为这是本文最令人惊讶的部分。如果我们手持一个手电筒到桌子上的物体（从它的顶部），阴影长度将会缩短。当你降低它，阴影长度将会增加。

那么，Android 框架中的光线从何而来呢？ 从头开始？ 还是中心？ 我会选择中心。 但是经过一番研究，我发现了这张图片。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowcnqrzyj30m80euq34.jpg)

在 Android 框架中有两个亮点。 顶部的那个是关键光。 另一个是环境光。 我们的影子出现在这两个光的组合中。 让我们看看表面的结果。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowcqi1boj30xk0cot8p.jpg)

在 Android 中，我们有很多小工具。 按钮、卡片、对话框、抽屉等。 所有这些都是视图。 如果有一个视图，我们就有阴影。 因此，我们如何决定 Android 中的 z 值。 根据材料设计指南，我们的视图有一个值图表。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frowctljycj31jk0mzwgy.jpg)

> 你可以在左侧找到高度值

## 正常和按压状态

正如我早些时候提到的，在 Android 框架中， 一些动画是为小部件实现的。 如果你在布局中放置浮动操作按钮，默认情况下它将具有6个 dp 高度。 但是你会注意到当你按下按钮时，fab 的高度将会提高到12 dp。 让我告诉你幕后发生的事情。

其实 FAB 有6个 dp 的高度。 当你按下按钮时，translation Z 值开始增加。 ViewPropertyAnimator 通过将 translation Z 值从0 dp 变为6 dp 来动画你的视图。 如果你释放按钮，ViewPropertyAnimator 来播放和动画  translation Z 值从6 dp 变为0  dp。 你可以为你的视图创建自定义状态列表动画([StateListAnimator](https://developer.android.com/reference/android/animation/StateListAnimator))，并将其添加到 xml 中的视图中。让我们用一个基本的图表来看看。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowcwwvg7j31jk0nb0vw.jpg)

> ViewPropertyAnimator 动画 translation Z 属性

## 阴影背后的秘密: Outline!

Outline 是一个属于 android.graphic 包的 API 类。让我们看看文档是怎么说的; 

> 定义一个简单的形状，用于包围图形区域。 可以为 View 计算，或由 Drawable 计算，以驱动由视图投射的阴影形状，或者剪辑视图的内容 

每个视图都有默认 Outline 来显示它的阴影。 如果我们创建一个自定义的 Drawable ，它的 Outline 将根据它的形状内部计算。 所以，如果我们画圆，Outline 就是圆。 如果我们画矩形，Outline 将是长方形的。 

总而言之，有一个 Outline 可以让你看到这个视图的阴影。但是，如果我想创建一个自定义视图并动态改变其边框呢？ 安卓系统是否会为我的自定义视图提供 outline ？

不会。 那么我们该怎么办呢？ 安卓应该为我们提供一种方式，使我们能够为自己的自定义视图提供自定义 Outline 。 没错。 

这里有一个魔法类：ViewOutlineProvider

## ViewOutlineProvider 是什么

![](https://cdn-images-1.medium.com/max/600/1*ppe2F4ofnwWan2Z_tHwKLw.gif)

在我最近的 [ScalingLayout](https://github.com/iammert/ScalingLayout) 库中，我没有对自定义视图实现阴影效果。我觉得它很非常漂亮，没有阴影。但请记住材料设计的基础知识。 3D，Depth，Z - index 。

我们可能无法在这个动画中精确地选择，但是在 ScalingLayout 中没有阴影。 让我们为视图提供一个自定义和动态的 Outline 。 

```java
public class ScalingLayoutOutlineProvider extends ViewOutlineProvider {

    @Override
    public void getOutline(View view, Outline outline) {
        outline.setRoundRect(0, 0, width, height, radius);
    }
}
```

```java
public class ScalingLayout extends FrameLayout {
    
    //...
    viewOutline = new ScalingLayoutOutlineProvider(w, h, currentRadius);
    setOutlineProvider(viewOutline);
    //..
    
}
```

就这样。现在我的自定义视图有高度支持。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frowetinohj31jk0efdhe.jpg)

> ScalingLayout  现在支持高度（使用自定义的 ViewOutlineProvider）

简化了 Gists 以提供关于 ViewOutlineProvider 的基础知识。 你可以在这里找到提交。 [[PR]](https://github.com/iammert/ScalingLayout/commit/6cfd01f5f4e9c867f3555e68237de46b56c11004)

Happy codings.

## 推荐阅读：

iammert/ScalingLayout ScalingLayout - With Scaling Layout scale your layout on user interaction. [github.com](https://github.com/iammert/ScalingLayout)

Environment - Material Design Shadows in the material environment are cast by these two light sources. In Android development, shadows occur when… [material.io](https://material.io/guidelines/material-design/environment.html#)

Defining Shadows and Clipping Views | Android Developers Material design introduces elevation for UI elements. Elevation helps users understand the relative importance of each… [developer.android.com](https://developer.android.com/training/material/shadows-clipping.html)

Playing with elevation in Android Elevation in Android is way more flexible than you’d think… [blog.usejournal.com](https://blog.usejournal.com/playing-with-elevation-in-android-91af4f3be596)

Elevation & shadows - Material Design All material objects, regardless of size, have a resting elevation, or default elevation that does not change. If an… [material.io](https://material.io/guidelines/material-design/elevation-shadows.html)

