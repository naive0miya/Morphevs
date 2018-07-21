# 使用 CoordinatorLayout Behaviors 拦截任何东西

> 原文 (Medium)：[Intercepting everything with CoordinatorLayout Behaviors](https://medium.com/google-developers/intercepting-everything-with-coordinatorlayout-behaviors-8c6adc140c26)
>
> 作者：[Ian Lake](https://medium.com/@ianhlake?source=post_header_lockup)

[TOC]

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frow78krwoj30bg09g3yf.jpg)

在探索 [Android 设计支持库](http://android-developers.blogspot.com/2015/05/android-design-support-library.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)时， 如果不运行到 [CoordinatorLayout](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)，你就不会得到多少帮助  - 设计库中的很多视图都需要 CoordinatorLayout。 但是为什么呢？  CoordinatorLayout 本身实际上并没有多大作用：使用标准框架视图， 它就像一个常规的 FrameLayout。 那么，魔法是从哪里来的呢？ 这就是 [CoordinatorLayout.Behaviors](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog) 的作用。通过将一个 Behavior 添加到 CoordinatorLayout 的直接子项中，您将能够拦截 touch events ，window insets ，measure ，layout 和 nested scrolling 。 设计库大量使用行为来提供您所看到的大部分功能。

## 创建一种行为

创建一个行为很简单：继承 Behavior。

```java
public class FancyBehavior<V extends View>
    extends CoordinatorLayout.Behavior<V> {
  /
    Default constructor for instantiating a FancyBehavior in code.
   /
  public FancyBehavior() {
  }
  /
    Default constructor for inflating a FancyBehavior from layout.
   
    @param context The {@link Context}.
    @param attrs The {@link AttributeSet}.
   /
  public FancyBehavior(Context context, AttributeSet attrs) {
    super(context, attrs);
    // Extract any custom attributes out
    // preferably prefixed with behavior_ to denote they
    // belong to a behavior
  }
}
```

注意这个类附加的泛型类型。在这里，我们要说的是，您可以将 FancyBehavior 附加到任何 View 类。但是，如果您只想将行为附加到一个特定的视图中，你可以把它写成 : 

```java
public class FancyFrameLayoutBehavior
    extends CoordinatorLayout.Behavior<FancyFrameLayout>
```

这样可以避免你在方法调用中从 View 转换到正确的子类型时所接收到的许多参数，简单方便就是一切。 

有些方法可以通过 [Behavior.setTag( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#setTag%28android.view.View,%20java.lang.Object%29)/[Behavior.getTag( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#getTag%28android.view.View%29) 保存临时数据，以及通过 [onSaveInstanceState()](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onSaveInstanceState%28android.support.design.widget.CoordinatorLayout,%20V%29)/[onRestoreInstanceState( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onRestoreInstanceState%28android.support.design.widget.CoordinatorLayout,%20V,%20android.os.Parcelable%29) 保存行为相关实例状态。我鼓励你尽可能轻量化地建立你的行为，但是这些方法有助于使有状态的行为成为可能。 

## 附加行为

当然，Behaviors 本身没有任何作用 - 它们需要附加到 CoordinatorLayout 的子视图上才能真正被调用。有三种主要方式可以完成：编程方式，XML，或通过注解自动执行。

### 以编程方式附加行为

当你认为一个行为附加到  CoordinatorLayout 中每个视图时，你不应该感到惊讶（如果您已阅读我们的[布局博客文章](https://medium.com/@ianhlake/layouts-attributes-and-you-9e5a4b4fe32c)），就会发现该行为实际上存储在每个视图的 LayoutParams 中ー这也是为什么需要向 CoordinatorLayout 的直接子视图声明行为，因为只有这些孩子有 LayoutParams 的特定行为存储子类。 

```java
FancyBehavior fancyBehavior = new FancyBehavior();
CoordinatorLayout.LayoutParams params =
    (CoordinatorLayout.LayoutParams) yourView.getLayoutParams();
params.setBehavior(fancyBehavior);
```

在这种情况下，你会看到我们正在使用默认的，没有参数的构造函数。这并不意味着你不可能有一个构造函数来获取你想要的任何参数ーー当你用代码做事的时候，你能做的事情是没有限制的。 

### 在 XML 中附加行为

当然，每次用代码做任何事情都会有点混乱。 和大多数定制的 LayoutParams 一样，有一个相应的布局属性来做同样的事情。 在这种情况下，这是 layout_behavior 属性 : 

```xml
<FrameLayout
  android:layout_height=”wrap_content”
  android:layout_width=”match_parent”
  app:layout_behavior=”.FancyBehavior” />
```

在这里，与代码案例不同，FancyBehavior（Context context，AttributeSet attrs）构造函数总是被调用。尽管如此，作为一个额外的奖励，如果你希望开发人员能够通过 XML 自定义你的行为的功能，你可以声明任何你想要的其他自定义属性，并从 XML AttributeSet 中提取这些属性。 

> 注：类似于父类负责解析和理解的属性的 layout_ 命名约定，对行为专门使用的任何属性使用 behavior_ 前缀。

### 自动附加一个行为

如果您正在构建一个需要自定义行为的自定义视图（如设计库中许多组件的情况），那么您可能希望默认附加该行为，而不需要每次手动在代码或 XML 中指定它。 要做到这一点，您的自定义视图只需要一个简单的注解附加到它的顶部: 

```java
@CoordinatorLayout.DefaultBehavior(FancyFrameLayoutBehavior.class)
public class FancyFrameLayout extends FrameLayout {
}
```

你会发现你的行为会被缺省构造函数调用，这与以编程方式附加行为非常相似。 请注意，任何当前的 layout_behavior 属性都会重写  [DefaultBehavior](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.DefaultBehavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)。

## 拦截触摸事件

一旦你把你的行为都设置好了，你就可以真正做点什么了。 一个行为可以做的事情之一就是拦截触摸事件。 

在没有 CoordinatorLayout 的情况下，这通常涉及每个 ViewGroup 的子类， 这通常会涉及到在[管理触摸事件训练](http://developer.android.com/training/gestures/viewgroup.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)中所谈到的那样 。 然而，使用 CoordinatorLayout，将会调用到它的 [onInterceptTouchEvent ( )](http://developer.android.com/reference/android/view/ViewGroup.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onInterceptTouchEvent%28android.view.Motio) 传递给行为的 onInterceptTouchEvent ( )，使你的行为有机会截获触摸事件。 通过返回 true，你的行为就会通过[onTouchEvent ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onTouchE) 接收所有未来的触摸事件 - 所有的其它 View 都不知道任何事情发生了什么。 例如， [SwipeDismissBehavior](http://developer.android.com/reference/android/support/design/widget/SwipeDismissBehavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog) 可以在任何 View 上工作。

还有另外一个更重的触摸拦截技术：无论如何都要阻止所有交互 。只要在 [blocksInteractionBelow ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#blocksIn) 中返回 true，就是这样。 当然，你可能希望有一些视觉信号，表明交互被阻止（以免他们认为应用程序已经完全被破坏了） - 这就是为什么 blocksInteractionBelow ( ) 的默认功能实际上依赖于 [getScrimOpacity ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#getScrim) 的值 - 返回一个非零值， 这里将在视图（颜色 [getScrimColor ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#getScrim)，默认为黑色）上绘制叠加颜色，并且一次性禁用触摸交互。 很方便。 

## 拦截窗口嵌套

假设您阅读了 [Why would I want to fitsSystemWindows?](https://medium.com/google-developers/why-would-i-want-to-fitssystemwindows-4e26d9ce1eec?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog) 博客。 在那里，我们深入地讨论了 SystemWindows 实际上是做什么的，但归根结底就是为您提供了避免在系统窗口下绘制窗口所需的窗口(如状态栏和导航栏)。在这里，行为也有自己的机会 - 如果你的视图 fitsSystemWindows = “true”，那么任何附加的行为都会调用 [onApplyWindowInsets( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onApplyWindowInsets%28android.support.design.widget.CoordinatorLayout,%20V,%20android.support.v4.view.WindowInsetsCompat%29)，优先于 View 本身。 

> 注意：在大多数情况下，如果您的行为不消耗整个窗口嵌套，它应该通过[ViewCompat.dispatchApplyWindowInsets ( )](http://developer.android.com/reference/android/support/v4/view/ViewCompat.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#dispatchApplyWindowInsets%28an) 传递窗口嵌套，确保任何子视图都有机会看到窗口嵌套。

## 拦截测量及布局

测量和布局是 Android [绘制视图](http://developer.android.com/guide/topics/ui/how-android-draws.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)的关键组件。只有行为，作为所有事物的拦截器，也通过 [onMeasureChild ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onMeasureChild%28android.support.design.widget.CoordinatorLayout,%20V,%20int,%20int,%20int,%20int%29) 和 [onLayoutChild ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onLayoutChild%28android.support.design.widget.CoordinatorLayout,%20V,%20int%29) 回调获得测量和布局的第一次机会。 

例如，让我们采取任何通用的 ViewGroup，并添加一个 maxWidth：

```java
/
  Copyright 2015 Google Inc.
 
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
 
      http://www.apache.org/licenses/LICENSE-2.0
 
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
 /
 
 package com.example.behaviors;

import android.content.Context;
import android.content.res.TypedArray;
import android.support.design.widget.CoordinatorLayout;
import android.util.AttributeSet;
import android.view.ViewGroup;

import static android.view.View.MeasureSpec;

/
  Behavior that imposes a maximum width on any ViewGroup.
 
  <p />Requires an attrs.xml of something like
 
  <pre>
  &lt;declare-styleable name="MaxWidthBehavior_Params"&gt;
      &lt;attr name="behavior_maxWidth" format="dimension"/&gt;
  &lt;/declare-styleable&gt;
  </pre>
 /
public class MaxWidthBehavior<V extends ViewGroup> extends CoordinatorLayout.Behavior<V> {
    private int mMaxWidth;

    public MaxWidthBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
        TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.MaxWidthBehavior_Params);
        mMaxWidth = a.getDimensionPixelSize(
                R.styleable.MaxWidthBehavior_Params_behavior_maxWidth, 0);
        a.recycle();
    }
    
    @Override
    public boolean onMeasureChild(CoordinatorLayout parent, V child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        if (mMaxWidth <= 0) {
            // 没有最大宽度意味着这个行为是一个没有操作
            return false;
        }
        int widthMode = MeasureSpec.getMode(parentWidthMeasureSpec);
        int width = MeasureSpec.getSize(parentWidthMeasureSpec);
        
        if (widthMode == MeasureSpec.UNSPECIFIED || width > mMaxWidth) {
            // 很抱歉在这里强加，但最大宽度是一个很大的交易
            width = mMaxWidth;
            widthMode = MeasureSpec.AT_MOST;
            parent.onMeasureChild(child,
                    MeasureSpec.makeMeasureSpec(width, widthMode), widthUsed,
                    parentHeightMeasureSpec, heightUsed);
            // 我们测量了视图，所以CoordinatorLayout不必再测
            return true;
        }

        // 看起来像默认的测量会很好
        return false;
    }
}
```

写通用的行为对任何事情都有用，但是要记住，你可以通过假设应用内部行为是如何使用的来简化你的生活。 (并不是每个行为都需要完全通用!) 

## 理解视图之间的依赖关系

上述所有功能都只需要单个视图。 但是，行为的真正力量来自于建立视图之间的依赖关系——也就是说，当另一个视图发生变化时，你的行为可以得到一个回调，根据外部条件改变它的功能。 

行为可以通过两种不同的方式依赖于视图 ：当其视图锚定到另一个视图（一个隐含的依赖关系）上，或者当您在 [layoutDependsOn ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#layoutDependsOn%28android.support.design.widget.CoordinatorLayout,%20V,%20android.view.View%29) 中显式返回 true 时。

当您的视图使用 CoordinatorLayout 的 [layout_anchor](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.LayoutParams.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#getAnchorId%28%29) 属性时，锚定就发生了。这与 [layout_anchorGravity](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.LayoutParams.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#anchorGravity) 属性结合使用，允许您有效地将两个视图的位置结合在一起。例如，您可以将 FloatingActionButton 锚定到 AppBarLayout，如果 AppBarLayout 滚动屏幕，FloatingActionButton.Behavior 将使用隐式依赖来隐藏 FAB。

在这两种情况下，当依赖视图被移除，您的行为会得到 [onDependentViewRemoved ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onDependentViewChanged%28android.support.design.widget.CoordinatorLayout,%20V,%20android.view.View%29) 的回调。当依赖视图被改变 (例，调整大小或重新定位本身 )，你的行为会得到 [onDependentViewChanged ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onDependentViewRemoved%28android.support.design.widget.CoordinatorLayout,%20V,%20android.view.View%29) 的回调。 

这种将视图连接在一起的能力是设计库的功能，例如 FloatingActionButton 和 Snackbar 之间的交互。 FAB 的行为取决于将 Snackbar 添加到 CoordinatorLayout 的实例，然后使用 onDependentViewChanged ( ) 回调向上平移 FAB 以避免重叠 Snackbar。

> 注意：当添加一个依赖关系时，不论子女的顺序如何，在依赖的视图被显示之后，将始终被排除在外。 

## 嵌套滚动

我只是在这里触及它。有几件事要记住：

- 您不需要声明嵌套滚动视图的依赖关系。 CoordinatorLayout 的每个孩子都有机会获得嵌套的滚动事件。
- 嵌套滚动不仅可以来自 CoordinatorLayout 的直接子节点，还可以起始于任何子节点（例如CoordinatorLayout 子节点的子节点）。
- 我将它称为嵌套滚动，但这确实涵盖了滚动和投掷 。

因此，在一个嵌套滚动事件中声明您的兴趣是从 [onStartNestedScroll ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onStartNestedScroll%28android.support.design.widget.CoordinatorLayout,%20V,%20android.view.View,%20android.view.View,%20int%29) 开始。您将收到滚动轴（例如，水平或垂直），这使得忽略在某个方向滚动很容易，并且必须返回 true 以接收该方向上的更多滚动事件。

在返回 true 返回到 onStartNestedScroll ( ) 之后，嵌套滚动分两步进行：

- [onNestedPreScroll ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onNested) 在滚动视图获取滚动事件之前运行，并允许您的行为消耗一些或所有滚动(最后消耗的 int []是一个'out'参数，你可以用它来表示你所消耗的滚动内容。
- [onNestedScroll ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onNested) 被称为滚动视图的滚动 - 你将得到多少滚动的视图和未消耗的(滚动)数量。

也有一个等价的投掷操作（虽然前投掷回调必须消耗投掷的所有部或全部 - 没有部分消耗）。

当嵌套滚动完成时，您将得到一个调用 [onStopNestedScroll ( )](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog#onStopNestedScroll%28android.support.design.widget.CoordinatorLayout,%20V,%20android.view.View%29) 的调用。 这标志着滚动的结束ーー在下一个滚动滚动开始之前，期望对 onStartNestedScroll ( ) 进行一个新调用。 

举个例子，如果你想在滚动时隐藏一个 FloatingActionButton，并在滚动时显示它ーー这只涉及到重写 onStartNestedScroll ( ) 和 onNestedScroll ( ) ，如下面的 [FABAwareScrollingViewBehavior](https://github.com/ianhanniballake/cheesesquare/blob/scroll_aware_fab/app/src/main/java/com/support/android/designlibdemo/FABAwareScrollingViewBehavior.java?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog) 中所示。 。

## 这仅仅是个开始

当一个行为的每一个单独的部分都是有趣的，当它们聚集在一起的时候——这就是魔法发生的地方。 我强烈建议您查看设计库的源代码以获得更高级的行为  - [Android SDK Search Chrome 扩展插件](https://chrome.google.com/webstore/detail/android-sdk-search/hgcbffeicehlpmgmnhnkjbjoldkfhoin?utm_campaign=adp_series_coordinatorlayoutbehavior_021716&utm_source=medium&utm_medium=blog)仍然是我最喜欢探索 AOSP 代码的资源之一（ \<android-sdk> / extras / android / m2repository 中包含的源代码总是最新的）。

在一个行为可以做的事情上有一个坚实的基础，让我知道你如何使用它们  

#BuildBetterApps

