# 使用 ConstraintLayout 来维护  StaggeredLayoutManager  中视图的宽高比

> 原文 (Medium)：[Aspect Ratio in StaggeredLayoutManager using Constraint Layout](https://medium.com/@burhanrashid52/aspect-ratio-in-staggered-layoutmanager-using-constraint-layout-9845d04d1962)
>
> 作者：[Burhanuddin Rashid](https://medium.com/@burhanrashid52?source=post_header_lockup)

[TOC]

你好开发者! !！ 今天，我们将学习一种有效的方法即使用 ConstraintLayout 来维护  StaggeredLayout  中视图的纵横比。我们将以 Pinterest 应用程序设计为例。本文假定您已经具备 [LayoutManager](https://guides.codepath.com/android/using-the-recyclerview) 和 [ConstraintLayout](https://medium.com/google-developers/building-interfaces-with-constraintlayout-3958fa38a9f7) 的基本知识，如果不是，我建议在进一步阅读之前阅读链接。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froubwt1axj307i0dcdjn.jpg)

我们的目标是设计类似 Pinterest 的布局。为此，我们需要在 RecycleView 中使用 StaggeredGridLayoutManager 。

## 什么是 StaggeredGridLayoutManager？

正如 doc 所述，StaggeredGridLayoutManager 是一个 LayoutManager ，它将孩子排列成一个交错的网格。它支持水平和垂直布局以及交错布局孩子的能力。

在交错布局中，列表中每个项目的长宽比可能是不同的 ，当我们尝试从 url 加载图像时，它会变得更加复杂。因此，从 url 加载图片时，我们不知道图片的确切长宽比。所以我们要做的是，在我们的行布局中，保持图像宽度为 match_parent 和高度为 wrap_content，所以每当图像加载时 ，它都会根据图像宽度自动包装其高度。

## 问题 ？

我们知道，有时互联网连接可能会非常缓慢，这需要大量的时间来加载图像，所以在正常情况下 ，我们在加载实际图像时使用一个占位符图像来提高用户体验。 

但是在交错布局中，由于我们不知道图像的宽高比，直到其完全加载，而且我们的占位符宽高比可能与实际图像不同，所以最初在列表中加载图像时，我们看到所有项目中具有相同宽高比的占位符，并且当实际图像完全加载时，它才开始设置具有其自身宽高比的实际图像，这使得用户体验不好，因为当用户滚动列表时，项目会非常频繁地改变其高度/宽度。

不用担心，我们现在可以使用约束布局宽高比特性来解决这个问题。但是首先您需要知道宽高比在 ConstraintLayout 中如何工作。

## 约束布局中的宽高比

使用宽高比可以完成与 [PercentFrameLayout](https://developer.android.com/reference/android/support/percent/PercentFrameLayout.html) 大致相同的操作，即将 View 限制为设置的宽高比，而不需要额外的 ViewGroup 增加在你的层次结构中的开销 。

![](https://cdn-images-1.medium.com/max/800/1*RfgavVsO88a44_F5xGnUog.gif)

要为 ConstraintLayout 内的任何视图设置比率，请执行以下操作：

- 确保至少有一个大小约束是动态的，即不是 “Fixed” 也不是 “Wrap Content”。
- 点击框左上角的 “切换宽高比约束”。
- 输入 width : height 格式中所需的宽高比例 ，例如：16：9

有关 ConstraintLayout  的更多细节，您可以阅读这篇很棒的文章。

[Building interfaces with ConstraintLayout In this article I’d like to highlight recent additions to ConstraintLayout in Android Studio 2.3 (Beta): chains and… medium.com](https://medium.com/google-developers/building-interfaces-with-constraintlayout-3958fa38a9f7)

## 解决方案

正如我们上面讨论的那样，开发人员遇到的一个解决方案是直接从后端 API 获取图像的高度/宽度，并设置从后端 API 获取的占位符比率，这将在用户滚动列表时带来顺畅的用户体验。

但是在  ConstraintLayout  出现之前，开发人员常常需要大量的工作来手动设置图片的高度/宽度。

首先，我们需要根据设备大小和跨度计数来计算图像宽度，而不是将该 px 值转换为 dp 并且在 onBindViewHolder ( ) 上以编程方式设置图片的高度/宽度，并在计算视图的宽度时还需要考虑填充。

这是一个大量的手工工作 ，但我们现在有 ConstraintLayout   ，现在我们可以直接设置图像视图的比例，ConstraintLayout  将会处理所有这些问题 

让我们看一个例子。

首先，我们将构建我们的 row_poster.xml。

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardCornerRadius="4dp"
    android:layout_margin="4dp">

    <android.support.constraint.ConstraintLayout
        android:id="@+id/parentContsraint"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <ImageView
            android:id="@+id/imgSource"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:scaleType="centerCrop"
            android:src="@drawable/placeholder"
            app:layout_constraintDimensionRatio="1:1"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <TextView
            android:id="@+id/txtName"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginBottom="4dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:layout_marginTop="4dp"
            android:text="TextView"
            android:textColor="@android:color/black"
            android:textStyle="italic"
            android:textAppearance="@style/Base.TextAppearance.AppCompat.Medium"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/imgSource" />

    </android.support.constraint.ConstraintLayout>
</android.support.v7.widget.CardView>
```

在上面的布局中，我们将 ImageView 宽度/高度设置为 0 dp，这是约束布局中的 match_constraint，意思是它的行为与 match_parent 相同。

默认情况下，我们保留了应用程序：layout_constraintDimensionRatio =“1：1”，这意味着我们的图像默认为正方形。

现在我们有了 MoviePoster 模型。

```java
class MoviePoster(var name: String,
                  var imageUrl: String,
                  var width: Int,
                  var height: Int)
```

它具有从 webservices 获取的图像 url 的宽度和高度。

默认情况下，我们保留了 app：layout_constraintDimensionRatio =“1：1”，这意味着我们的图像将是正方形。现在将以编程方式设置每个图像的宽高比。

这就是我们如何使用 ConstraintLayout 设置图像的宽高比，我们将创建一个全局的 ContraintSet ( ) 对象，并在 onBindViewHolder ( ) 中使用它。

```java
class MoviePosterAdapter : RecyclerView.Adapter<MoviePosterAdapter.ViewHolder>() {

private val set = ConstraintSet()
 
override fun onBindViewHolder(holder: ViewHolder, position: Int)         {
    val poster = mMoviePosters[position]
    
    holder.mMovieName.text = poster.name
    
    Glide.with(holder.itemView.context)
                .setDefaultRequestOptions(requestOptions)
                .load(poster.imageUrl)
                .into(holder.mImgPoster)
    val ratio =String.format("%d:%d", poster.width,poster.height)
    set.clone(holder.mConstraintLayout)
    set.setDimensionRatio(holder.mImgPoster.id, ratio)
    set.applyTo(holder.mConstraintLayout)
}
```

我们将克隆当前 ContraintSet ( ) 对象中视图的约束 ，并将宽高比率设置为 “width：height” 格式，并提供我们想要设置该比例的视图的 id，即在我们的场景中是 ImageView 的 id。

最后，我们将改变的约束从 set 对象应用到 Constraintlayout。

就是这样:) ......这就是我们需要做的...... ConstraintLayout 将管理所有的东西。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frouccpeg8j30600aodjb.jpg)

你可以在 GitHub 上检出这个例子。这个例子是用 Kotlin 编写的。[[PR](https://github.com/burhanrashid52/AspectRatioExample)]

谢谢 ！！！ 如果你觉得这篇文章有用。请喜欢，分享和鼓掌 ，这样其他人会在 Medium 看到它。如果您有任何问题或建议，请随时在 [Twitter](https://twitter.com/burhanrashid52) 上找我。

