# Android 中的 ItemDecoration

> 原文 (Medium)：[ItemDecoration in Android](https://proandroiddev.com/itemdecoration-in-android-e18a0692d848)
>
> 作者：[Riyaz Ahamed](https://proandroiddev.com/@DevAhamed?source=post_header_lockup)

[TOC]

> 第1部分：避免将分隔线添加到视图布局

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frosuy02pnj30m8092gln.jpg)

> ItemDecoration 可以绘制到 RecyclerView 项目的所有四面

首先的事情。ItemDecoration 是什么？官方文档就是这么定义的。

> ItemDecoration 允许应用程序从适配器的数据集中为特定的项目视图添加特殊的图形和布局偏移量。这对于在项目之间绘制分隔线，突出显示，视觉分组边界等等非常有用。 

我们不能简单地说 ItemDecoration 只是一个花哨名字的分隔线。这远不止于此。顾名思义，分隔线只能在项目之间绘制。但 ItemDecoration 可以在项目的所有四个方面绘制。 ItemDecoration 完全控制开发者测量和绘制 decorations 。decorations 可以是一个 divider 或只是一个 inset 。

但不幸的是，大多数的 Android 开发人员都没有使用 ItemDecoration 。在三部分系列中，我们将学习 ItemDecoration 的力量。

第1部分：不要添加分隔符视图 - 使用 ItemDecoration 

第2部分：不要为 inset 使用填充 - 使用 ItemDecoration 

第3部分：在 GridLayoutManager 中有效地绘制 decorations 

这里是系列的第一部分。

## 不要添加分隔线作为视图 - 它会影响性能

我亲眼见过一些开发者采取快捷方式向 recycleroview 添加分隔符 。原因很明显。 ListView 有一个原生的方式来添加分隔线。您可以通过 xml 本身添加分隔线。

但是使用 RecyclerView，您不能直接添加分隔线。您需要添加一个可以绘制分隔线的 ItemDecoration。但开发人员觉得很困难，直接将分隔线添加到视图中，而不是使用 ItemDecoration。 

```java
<LinearLayout android:orientation="vertical">
    <LinearLayout android:orientation="horizontal">
        <ImageView />
        <TextView />
    </LinearLayout>
    <View
        android:width="match_parent"
        android:height="1dp"
        android:background="#333" />
</LinearLayout>
```

每当我们采取捷径，它可能会有一些不利影响。在这种情况下，它会影响性能。 

当向布局中添加分隔符时，我们正在增加视图计数。 我们知道，在布局中拥有更少的视图对性能更有好处。 有时添加分隔符作为视图增加布局层次结构。 考虑上面的例子，我们真的不需要外部线性布局。 要添加分隔符，我们必须创建一个额外的外部布局。 

## 不要添加分隔线作为视图ーー它有副作用

在项目动画分隔线，它将与动画一起动，因为分隔线是视图的一部分。为了更清晰，看看下面的 gif。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frosvewsf3g308w0ft4qp.gif)

显然，分隔线不应与项目视图一起动画。 它应该与项目视图的动画分开。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1froswogxphg308w0ft4oj.gif)

## 不要添加分隔线作为视图ーー它缺乏灵活性

如果分隔线是视图的一部分，你就不能控制它。 您唯一可以做的事情是基于项目的位置可见性分隔线可以改变。 如果你想做的不仅仅是可见性，ItemDecoration 是灵活的。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frosxlcem6j30f00qot9l.jpg)



在上图中，组分隔线的最后一个项填充整个宽度。 其他分隔线的左侧边缘有 56dp。 这里是 ItemDecorator 的 onDraw 代码。 

```java
@Override
public void onDraw(Canvas canvas, RecyclerView parent, RecyclerView.State state) {
  canvas.save();
  final int leftWithMargin = convertDpToPixel(56);
  final int right = parent.getWidth();

  final int childCount = parent.getChildCount();
  for (int i = 0; i < childCount; i++) {
    final View child = parent.getChildAt(i);
    int adapterPosition = parent.getChildAdapterPosition(child);
    left = (adapterPosition == lastPosition) ?  0 : leftWithMargin;
    parent.getDecoratedBoundsWithMargins(child, mBounds);
    final int bottom = mBounds.bottom + Math.round(ViewCompat.getTranslationY(child));
    final int top = bottom - mDivider.getIntrinsicHeight();
    mDivider.setBounds(left, top, right, bottom);
    mDivider.draw(canvas);
  }
  canvas.restore();
}
```

## 不要添加分隔线作为视图 - 使用 ItemDecoration

写自己的 ItemDecoration 很简单。 你只需要创建一个从 ItemDecoration 扩展的类。 重写 getItemOffsets ( ) 和 onDraw ( ) 方法。 对于示例实现，请看[这里](https://developer.android.com/reference/android/support/v7/widget/DividerItemDecoration.html)。 

随着支持库25.0.0版本的发布，我们有了一个新的 "DividerItemDecoration" 类。 这是一个可以通过装饰添加简单分隔线的实用工具类。 

```kotlin
DividerItemDecoration decoration = new DividerItemDecoration(getApplicationContext(), VERTICAL);
recyclerView.addItemDecoration(decoration);
```

## 注意事项

- 多个 ItemDecorations 可以添加到单个 RecyclerView。是时候让你的创造力疯狂起来了 。
- 所有的 ItemDecorations  都是在绘制项目之前绘制。 如果您想在绘制视图后绘制 ItemDecorations ，请重写 onDrawOver ( ) 而不是 onDraw ( ) 方法。

因此，下次当您想要向 "RecyclerView" 添加分隔符时，不要在布局中使用视图。 相反，使用 ItemDecoration。 

