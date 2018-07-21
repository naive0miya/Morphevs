# 在 Android 系统中使用高度

> 原文 (Medium)： [Playing with elevation in Android](https://blog.usejournal.com/playing-with-elevation-in-android-91af4f3be596)
> 作者： [Sebastiano Poggi](https://blog.usejournal.com/@seebrock3r)

[TOC]

每个人都知道 Material Design 有阴影。大多数人知道可以通过使用 elevation 属性来控制 Android 中 Material 元素的虚拟 Z 坐标来控制阴影。 很少有人知道如何做更多的事情来调整 UI 元素投射的阴影！

## 阴影是什么？
在“材质设计”中，elevation 是材质平面相对于屏幕“基本”平面的虚拟 Z 坐标的表现形式。例如：（[Material Design guidelines](https://material.io/guidelines/material-design/elevation-shadows.html)）
![@Image shamelessly ripped out of the Material Design guidelines.](https://ws3.sinaimg.cn/large/006tKfTcgy1frowfsf1ltj30ov048jr6.jpg)

在材料设计系统中，有两个光源。一个是位于屏幕顶部上方的一盏关键光，以及一盏位于屏幕中心正上方的环境光：（[Touchlab blog](https://www.touchlab.co/blog/2016/1/consistent-lighting-in-material-design)）
![@Image shamelessly ripped out of the Touchlab blog.](https://ws4.sinaimg.cn/large/006tKfTcgy1frowfzy24lj31jk113my8.jpg)

这两个光线分别投射出自己的阴影，一个主要影响材料片（关键光）底部边缘的影子，另一个则影响到所有边缘（环境光）：
![@Image derived from the Material Design guidelines.](https://ws1.sinaimg.cn/large/006tKfTcgy1frowg4r19tj31400gb0ss.jpg)

elevation 属性直接控制所得阴影的形状; 你可以通过按钮清楚地看到这一点，这些按钮根据它们的状态改变了它们的高度: 

![@Image from Vadim Gromov’s Dribbble.](https://cdn-images-1.medium.com/max/1600/0*ufoUeMLtFh0yp6Sa.)

你可能认为 elevation 属性是控制阴影外观的唯一方法，但事实并非如此。

在安卓系统中，有一个非常鲜为人知的 API 叫做 [Outline](https://developer.android.com/reference/android/graphics/Outline.html)，它为一个材质片提供了所需的信息来投射阴影。默认的视图行为是将轮廓定义委托给它们的 drawable 背景。 例如 ShapeDrawables 提供了符合其形状的轮廓，而 ColorDrawables，BitmapDrawables 等提供了一个矩形匹配他们的边界。但是并没有什么地方说我们不能改变这一点，所以可以调用 setOutlineProvider（）方法告诉视图使用不同的 [ViewOutlineProvider](https://developer.android.com/reference/android/view/ViewOutlineProvider.html)：

```kotlin
view.outlineProvider = outlineProvider
```
如果我们控制了 ViewOutlineProvider，我们可以调整生成的 Outline，操作系统将为我们设置想要的任何阴影：

```kotlin
inner class OutlineProvider(
  private val rect: Rect = Rect(),
  var scaleX: Float,
  var scaleY: Float,
  var yShift: Int
) : ViewOutlineProvider() {

  override fun getOutline(view: View?, outline: Outline?) {
    view?.background?.copyBounds(rect)
    rect.scale(scaleX, scaleY)
    rect.offset(0, yShift)
    
    val cornerRadius = 
        resources.getDimensionPixelSize(R.dimen.control_corner_material).toFloat()
    
    outline?.setRoundRect(rect, cornerRadius)
  }
}
```
您可以使用 elevation 和 Outline 对高度阴影的形状和位置进行各种调整：
![@Believe it or not, I have actually captured this one myself on my phone.](https://cdn-images-1.medium.com/max/1600/1*nFcOfSdcmwflAnboYVGqyg.gif)

你会注意到这里的阴影不仅适应不同的 elevation 值，而且还被转换并获得比视图本身更大或更小的尺寸。

你可以做的另一件事就是分配一个与视图本身的实际轮廓不同的形状 - 我想不出任何有意义的情况 ，但是你可以。唯一的限制是形状必须是凸的。在 Outline 上有方便的方法来使用椭圆形，矩形和圆角矩形，但是你也可以使用任何任意的 Path，只要它是凸显的。

不幸的是，不应该过分夸大某些效果，在渲染阴影时，你可以看到系统会有一些快捷方式，这会在你点击它们的时候产生一些相当恼人的效果。 。

> 如果你对 Android 中的阴影是如何渲染的感到好奇，相关的代码位于 AOSP 的 [hwui](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/libs/hwui/) 包中 - 你可以开始查看 AOSP 中的软件包  [AmbientShadow.cpp](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/libs/hwui/AmbientShadow.cpp)。

另一个限制是我们不能对高度阴影进行着色，我们只能选择默认灰色，但老实说，我不认为这是一件坏事😉

## 调整 elevation 的动作
我用这种方法在 [Squanchy](https://squanchy.net/) 为这些卡片设计了一个独特的立面，这是我去年一直在开发的一个开源会议应用程序：
![|center](https://ws2.sinaimg.cn/large/006tKfTcgy1froxmot6nbj318g27mx30.jpg)

正如你所看到的，这些卡片的阴影看起来比正常的高度阴影更为明显。这是通过增加比卡片小 4dp 的 Outline 和 4dp 的 elevation 来获得的：
（card-styles.xml ）

```xml
<declare-styleable name="CardLayout">
  <attr name="cardCornerRadius" format="dimension" />
  <attr name="cardInsetHorizontal" format="dimension" />
  <attr name="cardInsetTop" format="dimension" />
  <attr name="cardInsetBottom" format="dimension" />
</declare-styleable>

<attr name="cardViewDefaultStyle" format="reference" />

<style name="Widget.Squanchy.CardLayout" parent="None">
  <item name="android:background">@drawable/card_background</item>
  <item name="android:foreground">@drawable/primary_touch_feedback</item>
  <item name="android:elevation">@dimen/card_elevation</item>
  <item name="android:stateListAnimator">@anim/clickable_state_list_anim</item>
  <item name="cardCornerRadius">@dimen/card_corner_radius</item> <!-- 4dp -->
  <item name="cardInsetHorizontal">@dimen/card_inset_horizontal</item> <!-- 4dp -->
  <item name="cardInsetTop">@dimen/card_inset_top</item> <!-- 4dp -->
  <item name="cardInsetBottom">@dimen/card_inset_bottom</item> <!-- 4dp -->
</style>
```
卡片上有一个 android：stateListAnimator，它也根据按下的状态来调整它们的 elevation 和 translationZ 。你可以看到 cardInset  属性如何在 [CardLayout](https://github.com/squanchy-dev/squanchy-android/blob/develop/app/src/main/java/net/squanchy/support/widget/CardLayout.java) 代码中使用，以缩小我们提供给系统的 Outline。

当你在 Squanchy 中滚动时，你可能会注意到，阴影会随着 y 轴上的卡片滚动而变化 : 

![|center](https://cdn-images-1.medium.com/max/1600/1*9GmsJF_94gMn3XQGdG_OvA.gif)

如果这个效果在你看到的 gif 中太微妙，那么这张图片就很明显：
![](https://ws3.sinaimg.cn/large/006tKfTcgy1froxaok41nj31jk0bumyv.jpg)

这怎么可能？我们绝对不会根据项目的 y 位置来更改高度和轮廓（我们可以这样做，但这不是一个好主意，因为它需要重新计算每个滚动项的轮廓）。

你会记得我之前提到过材质设计环境中有两个阴影，一个位于屏幕上方，另一个位于中心正上方。当一个物体离它越来越远的时候，顶部的光线(就是关键光线)投下了更长的阴影。这在安卓系统中总是正确的，只是在正常情况下你并没有注意到这一点。Squanchy 风格使其更加明显，你 甚至可以通过使用更高的 elevation 值来进一步夸大它 ：
![](https://cdn-images-1.medium.com/max/1600/1*kIo32J6ax6IkPpXNOECVVg.gif)

## 在你离开前还有最后一件事

最后，请记住，轮廓不仅仅用于阴影，默认情况下它们也定义了视图的裁剪！ 如果你有一个奇怪的轮廓，不希望它影响你的实际视图的绘制，你会想要在它上面调用 [setClipToOutline (false)](https://developer.android.com/reference/android/view/View.html#setClipToOutline%28boolean%29) ，以避免令人讨厌的意外。 

只有当你提供的 Outline 有 canClip ( ) 返回 true 时，这一点很重要，当轮廓是矩形，圆角矩形或者圆形的时候。非圆形椭圆和任意路径不能提供裁剪，因此 setClipToOutline ( ) 在这些情况下不起作用。

> 有趣的事实是：矩形和圆形都是[内部表示](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/graphics/java/android/graphics/Outline.java)为圆角矩形的特殊情况。矩形是一个圆角半径为零的圆角矩形，圆是一个圆角矩形，其圆角半径等于圆形高度 / 宽度的一半。

如果你想阅读更多关于这个主题的内容，Android 开发者网站上有一个关于 定义阴影和裁剪视图 的页面，通过代码示例来浏览相同的主题，并链接到一些更多的 javadocs 。

如何想在自己的 Android 设备上实践 elevations，我做了一个简单的演示应用程序 [Play Store](https://play.google.com/store/apps/details?id=me.seebrock3r.elevationtester)：应用程序的代码开源在 [GitHub](https://github.com/rock3r/uplift) 。
![@|center](https://ws1.sinaimg.cn/large/006tKfTcgy1frox14toi2j30a30i2jta.jpg)








