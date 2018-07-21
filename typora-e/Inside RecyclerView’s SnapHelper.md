# 深入RecyclerView 的 SnapHelper

> 原文：[Inside RecyclerView’s SnapHelper](https://proandroiddev.com/android-recyclerview-snaphelper-19eaa9598da6)
>
> 作者：[Jag Saund](https://proandroiddev.com/@jagsaund)

[TOC]

试图让你的 [RecyclerView](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html) 捕捉到一个特定的项目？尝试使用 [SnapHelper](https://developer.android.com/reference/android/support/v7/widget/SnapHelper.html)。看看你如何定制它，让它为你工作！

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvl6wxyrj31jk0qiwfq.jpg)

RecyclerView 是 ListView 的演变。 它带来了强大的功能。 我们可以使其表现得像一个旋转木马或使其捕捉到一个特定的位置。 但手动做这件事很困难。 这就是 Android 支持库所涵盖的地方。 SnapHelper 使得这件事变得轻而易举，但是当我们需要定制一些细节时呢？ 本文将深入研究 SnapHelper 的内部，以便你可以调整并使其适用于你。

## 什么是 SnapHelper

所以你需要扩大你的 RecyclerView 来捕捉到列表中的第一个可见的项目。 或者，也许你需要它捕捉到最接近中间的项目。 或者也许有其他的变化。 一个选项可能是附加一个 RecyclerView.OnFlingListener 并拦截事件。 这是很多工作，容易忽略一些事情。

这是 SnapHelper 的功能。SnapHelper 组件是一个抽象类。 Android 为你提供两种变体： [LinearSnapHelper](https://developer.android.com/reference/android/support/v7/widget/LinearSnapHelper.html) 和 [PagerSnapHelper](https://developer.android.com/reference/android/support/v7/widget/PagerSnapHelper.html)。 他们提供了几乎所有你需要的东西，但是如果这还不够，你应该考虑扩展其中的一个。
LinearSnapHelper 容易捕捉到最接近 RecyclerView 中间的项目。

PagerSnapHelper 提供了与 ViewPager 类似的行为，但要求你的项目视图的布局参数设置为 MATCH_PARENT。

## 在你的项目中使用 SnapHelper

集成其中一个具体的 SnapHelper 实现非常简单。

```kotlin
class MyActivity : AppCompatActivity {
  
  override fun onCreate(savedInstanceState: Bundle?) {
    val recyclerView = findViewById(R.id.list) as RecyclerView
    recyclerView.layoutManager = LinearLayoutManager(this)
    recyclerView.adapter = ExampleAdapter()
    LinearSnapHelper().attachToRecyclerView(recyclerView) // Configures the snap helper and attaches itself to the recycler view -- now items will snap to the center
  }
}
```

> 我们来深入了解一下 SnapHelper 的组件。	

## 组件

- SnapHelper 实现了 RecyclerView.OnFlingListener。
- LinearSnapHelper 或 PagerSnapHelper 是 SnapHelper 的具体实现。 他们提供实现自定义 SnapHelper 所需的大部分功能。 考虑扩展其中的一个以满足你的需求。 例如，LinearSnapHelper 捕捉到最接近父级中间的视图。
- Scroller  提供了一种计算 RecyclerView 需要滚动到目标视图的距离的方法。
- [RecyclerView.SmoothScroller](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.SmoothScroller.html) 执行平滑滚动到目标视图的行为。
- [LinearSmoothScroller](https://developer.android.com/reference/android/support/v7/widget/LinearSmoothScroller.html) 扩展了 SmoothScroller，并使用线性插值器平滑滚动到目标视图。一旦插值器接近目标视图，插值器就切换到减速插值器。

## 怎么运行的

1. 我们构建我们的 SnapHelper 并将其附加到 RecyclerView：
   LinearSnapHelper ( ).attachToRecyclerView（recyclerView）
    SnapHelper 创建一个新的 OnScrollListener 和 OnFlingListener，并将其注册到 RecyclerView。 它也构建了一个新的 Scroller。  Scroller 计算 RecyclerView 到达目标视图所需的距离。
2. RecyclerView.fling（int velocityX，int velocityY）捕获投掷速度并将其转发给 SnapHelper。回想一下，SnapHelper 实现 RecyclerView.OnFlingListener 并注册自己的 RecyclerView。
3. 调用 SnapHelper 的 onFling（int velocityX，int velocityY）方法。我们检查 x 或 y 的速度是否超过了最小的速度。如果这样做，则调用 snapFromFling。
4. snapFromFling 将完成所有工作以平滑滚动并捕捉到目标位置。 这需要调用几个方法来完成繁重的工作。 首先，创建一个 LinearSmoothScroller 实例（通过调用 createSnapScroller）。 然后， theSnapHelper 找到适配器中的目标位置，捕捉给定的 x 和 y 速度（通过调用 findTargetSnapPosition）。 这个位置被传递给 LinearSmoothScroller：smoothScroller.setTargetPosition（targetPosition）。 最后，SnapHelper 通知 LayoutManager 平滑滚动到目标视图：

```kotlin
layoutManager.startSmoothScroll(smoothScroller);
```

当一切完成后，目标视图将被捕捉到屏幕上的位置。

![How SnapHelper works](https://ws4.sinaimg.cn/large/006tNc79gy1ftbxtw7aa7g30rs0g876l.gif)

### 提示

需要限制最大的投掷距离？就是这样：

```kotlin
private var scroller: Scroller? = null
private val minX = -2000
private val maxX = 2000
private val minY = -2000
private val maxY = 2000

override fun attachToRecyclerView(recyclerView: RecyclerView?) {
    scroller = Scroller(recyclerView?.context, DecelerateInterpolator())
    super.attachToRecyclerView(recyclerView)
}
override fun calculateScrollDistance(velocityX: Int, velocityY: Int): IntArray {
    val out = IntArray(2)
    scroller?.fling(0, 0, velocityX, velocityY, minX, maxX, minY, maxY)
    out[0] = scroller?.finalX ?: 0
    out[1] = scroller?.finalY ?: 0
    return out
}
```

## SmoothScroller

RecyclerView 提供了一个抽象类：RecyclerView.SmoothScroller。 平滑的滚动器提供原始功能来跟踪目标视图位置并以编程方式触发滚动。 Android 提供了一个具体的实现：LinearSmoothScroller。 这个类使用 LinearInterpolator。 一旦目标视图变成 RecyclerView 的子项，它切换到一个 DecelerateInterpolator。 这给人的印象是 RecyclerView 慢慢接近目标视图。
有三种特别重要的方法：

1. `calculateSpeedPerPixel`：最简单也是最简单的一种，我们将显示指标转换为像素每秒 返回*MILLISECONDS_PER_INCH / displayMetrics.densityDpi* ;
2. `calculateTimeForScrolling`：默认值不会在时间上设置上限，但可以确保滚动的数量非常小，总是返回正值（避免舍入错误）。
   *（int）Math.ceil（Math.abs（dx）    MILLISECONDS_PER_PX）*;
3. `onTargetFound`：当目标视图成为 RecyclerView 的子项时调用。 这是 SmoothScroller 收到的最后一个回调。 我们需要计算到捕捉位置的距离和时间。calculateDistanceToFinalSnap 将提供滚动距离（SnapHelper 将其定义为抽象方法）。calculateTimeForDeceleration 将提供给定上面计算的距离的时间。 动作对象更新时间和距离的详细信息。 LinearSmoothScroller 使用此信息来完成滚动。

### 提示

需要限制平滑滚动可以采取的时间？就是这样：

```kotlin
private const val MILLISECONDS_PER_INCH = 100
private const val MAX_SCROLL_ON_FLING_DURATION = 250 // ms

override fun createSnapScroller(layoutManager: LayoutManager?): LinearSmoothScroller? {
    if (layoutManager !is ScrollVectorProvider) return null
    return object : LinearSmoothScroller(context) {
        override fun onTargetFound(targetView: View?, state: State?, action: Action?) {
            if (targetView == null) return
            val snapDistances = calculateDistanceToFinalSnap(layoutManager, targetView)
            val dx = snapDistances[0]
            val dy = snapDistances[1]
            val time = calculateTimeForDeceleration(Math.max(Math.abs(dx), Math.abs(dy)))
            if (time > 0) {
                action?.update(dx, dy, time, mDecelerateInterpolator)
            }
        }

        override fun calculateSpeedPerPixel(displayMetrics: DisplayMetrics?): Float =
                MILLISECONDS_PER_INCH / displayMetrics.densityDpi

        override fun calculateTimeForScrolling(dx: Int): Int =
                Math.min(MAX_SCROLL_ON_FLING_DURATION, super.calculateTimeForScrolling(dx))
    }
}
```

## 如何找到目标适配器的位置

SnapHelper 需要将闪光速度转换为适配器中的物品位置。 为了做到这一点，你需要实现 findTargetSnapPosition（LayoutManager layoutManager，int velocityX，int velocityY）。 注意：这不一定是准确的 - 将其视为更多的估计。

为了更好地理解如何做到这一点，我们来深入了解 LinearSnapHelper 如何确定目标位置。 回想一下， LinearSnapHelper 的目标是捕捉到最接近中心的项目。
首先，它找到当前的中心视图及其在适配器中的位置。 接下来，在 Scroller 的帮助下，LinearSnapHelper计算 RecyclerView应该滚动的距离 - 使用 calculateScrollDistance（int velocityX，int velocityY）。 我们需要将总卷动距离映射到多个适配器项目。 LinearSnapHelper 通过计算当前可见的平均 RecyclerView 子高度来完成此操作。 这代表了每个孩子的平均滚动量。 现在我们可以估计要滚动的孩子总数：

*val delta = totalScrollDistance / avgScrollDistancePerChild .*

最后，新的目标捕捉位置被估计为 currentCenterPos + delta。

![估计要滚动的孩子的数量](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvlrnl7bj30bz0cwaa3.jpg)

这里需要注意的一点是，滚动的方向来自速度。 但是，这可能与 LayoutManager 中的子级的顺序不匹配。 为了克服这个问题，使用 ScrollVectorProvider 来获取方向（注意：Android 提供的所有 LayoutManagers 实现 ScrollVectorProvider）：

```kotlin
val vectorProvider = layoutManager as RecyclerView.SmoothScroller.ScrollVectorProvider
val vectorForEnd = vectorProvider.computeScrollVectorForPosition(itemCount - 1)
```

如果我们假设我们正在垂直滚动，那么 vectorForEnd 中的值允许我们将方向定义为：

```kotlin
if (vectorForEnd.y < 0) {
  delta = -delta;
}
val targetPos = Math.max(0, Math.min(itemCount - 1, currentPos + delta))
```

## 如何找到目标快照视图

一旦 RecyclerView 开始解决，我们需要找到一个参考视图来从当前的子视图中捕捉。 这是 findSnapView（LayoutManager layoutManager） 的用途。 让我们深入了解 LinearSnapHelper 如何确定目标快照视图。 LinearSnapHelper 遍历当前连接到 RecyclerView 的孩子。 它跟踪最接近中心的视图。 最后，它返回它发现最接近中心的视图。

创建一个捕捉特定位置的小部件曾经非常困难。 用 ListView 实现这一点非常困难。 RecyclerView 使得它更容易，但仍然是一个巨大的繁重。 使用 SnapHelper，这是两个具体的实现 - LinearSnapHelper 和 PagerHelper，这变得无痛和简单。 扩展其功能也很简单。 通常这需要修改 findTargetSnapPosition， findSnapView 或 calculateDistanceToFinalSnap。 我们查看了 SnapHelper 组件的细节以及它们如何相互交互。 这个知识应该有助于扩展 SnapHelper 来根据你的需求来定制它。 分享你的技巧，以及如何使用SnapHelper！

