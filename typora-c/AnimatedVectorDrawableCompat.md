# AnimatedVectorDrawableCompat 教程

> 原文 (Medium)：[AnimatedVectorDrawableCompat](https://android.jlelse.eu/animatedvectordrawablecompat-3d9568727c53)
>
> 作者：[Bartek Lipinski](https://android.jlelse.eu/@blipinsk?source=post_header_lockup)

[TOC]

> 一个简单的逐步解决方案来实现它

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou8whhzzg30b408c0vr.gif)

> (来源: <https://dribbble.com/shots/1618281-Material-Design-Animation>)

尽管 AnimatedVectorDrawableCompat 已经有相当长的一段时间了（自2016年2月起 - 支持库23.2.0），但是 Google 仍然没有设法提供一个关于如何使用它的简单指南。 你可以在这里或那里找到一些信息，但没有什么可靠的。 没有任何包含所有需要的知识。

下是我尝试收集所有必需的信息并将其压缩到你可以轻松消化的东西。

这就是你需要做的: 

## 1. 将 AppCompat 依赖项添加到你的 build.gradle

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou938v42j30m802aq30.jpg)

- 我正在使用最新的（目前）25.0.0版本，但任何版本从23.2.0应该正常工作，并以类似的方式。

## 2. 创建矢量绘图文件

- 这才是真正的动画。 
- 一个矢量可绘制文件需要放在你项目的 res / drawable 目录中。
- 更多的信息在[这里](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html)的内容。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou99dvptj30m80z47bu.jpg)

上面的代码代表了一个基本的黑色菜单（汉堡包）图标：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frou9e153aj30fk0fkjrt.jpg)

## 3. 创建动画文件

- 这些指定了矢量可绘制部分的动画方式。
- 可以有多个动画单个矢量绘制。 每个指定动画的矢量绘制的不同部分。
- 矢量绘制的部分可以使用名称标记（menu，bottom_container，bottom，stem_container，stem，top_container，top 在示例中）引用。
- 根动画对象可以是 set 或者 objectAnimator。
- 这些文件需要放在 res / anim 中。

以下代码指定了 top_container 的动画。它修改了它的四个参数 translateX，translateY，scaleX 和 rotation：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou9jc31pj30m80j2teh.jpg)

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou9n34nxg30ak0akdgl.gif)

## 4 .创建动画矢量可绘制文件

- 动画矢量文件将所有内容联系在一起（向量可以与所有动画文件一起绘制）。 
- 需要放在项目的 res / drawable 目录中。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frou9wobzvj30m80590uh.jpg)

这里有一件重要的事: 

如果你的 minSdkVersion 低于21（如果不是，那我真的不知道你在考虑什么） AnimatedVectorDrawableCompat 只是使用一个常规的 AnimatedVectorDrawable），Android Studio 可能会在你的动画矢量可绘制文件中引发 Lint 警告：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froua00npjj30rs05x40f.jpg)

> 不用担心那个！如果你正确地做了其他的事情，你的 AnimatedVectorDrawableCompat 将工作，尽管有这个 Lint 警告。你可以添加工具：tools:ignore="NewApi"，如果你不想再看到它。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1froua3zmjpj30rs07z778.jpg)

## 5. 修改你的 build.gradle

- 将 vectorDrawables.useSupportLibrary = true 添加到你的模块的 build.gradle 的 android 部分的 defaultConfig 中。 
- 你需要这样 ，动画矢量可绘制文件才能和低于棒棒糖的 API 兼容。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frouaccex8j30m80bv40l.jpg)

## 6. 将 AnimatedVectorDrawableCompat 设置为 ImageView 或 ImageButton

- 你可以使用应用程序添加它在XML中：srcCompat：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frouafkpgdj30m8065wfb.jpg)

或通过代码：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frouaie6rhj30m8035wf3.jpg)

## 7. 在需要时开始动画

- 获取对 AnimatedVectorDrawableCompat 的引用（或者只是对它实现的接口 - Animatable）。如果通过代码添加了 AnimatedVectorDrawableCompat，则可以使用之前获得的引用（并跳过这一点）：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frouatb744j30m80230sx.jpg)

开始动画：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frouaw21f9j30m8023746.jpg)

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frouazq69ug309609640v.gif)

## 好消息和坏消息

我们从好的开始吧：

你可以使用 [Roman Nurik](https://medium.com/@romannurik) 的新工具（目前处于预发布状态，但已经非常有用）简化步骤1-3：[AndroidIconAnimator](https://romannurik.github.io/AndroidIconAnimator/)。 它可以采用一个 svg 文件，并根据你指定的动画参数导出一个动画矢量可绘制文件。

这里真正有趣的是返回的动画矢量文件使用 aapt 工具提供的一些很酷的技巧。 返回的可绘制文件包含整个动画所需的所有数据（向量可绘制和动画文件包含在其中）。 将其视为步骤1-3中的所有文件合并成一个动画矢量可绘制文件。

至于坏消息：

AnimatedVectorDrawableCompat 在 API <21有一些主要限制：

来自 [Chris Banes](https://medium.com/@chrisbanes) 的文章：

> 对于在平台 API 21上运行动画向量可以做的事情也有一些局限性。 以下是目前在这些平台上不起作用的事情: 
>
> Path Morphing (PathType evaluator) 。 这用于将一个路径变形为另一个路径。
>
> Path Interpolation 。 这是用来定义一个灵活的插值器（表示为一个路径），而不是象 LinearInterpolator 这样的系统定义的插值器。
>
> Move along path 。 这很少使用。 几何对象可以沿着任意路径移动。

这基本上意味着你现在可以忘记动画路径对象的 pathData 属性，这是一个重要的警告。让我们只是希望谷歌的聪明人能够找到一个方法把这个功能带到旧的平台上。 

还有一件事：

AnimatedVectorDrawableCompat 似乎有一个小错误（除非它是一个功能）。 如果你试图通过将 startOffset 添加到动画向量中的所有动画来延迟整个 AnimatedVectorDrawableCompat，那么你的动画根本不会工作(至少它不适合我)。 它只是从开始状态跳到结束状态(有些延迟)。 从整个动画矢量开始至少有一个动画运行。 小心那个。 

