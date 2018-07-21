# Android 开发技巧

> 原文 (Medium)：[Android Dev Tip #1](https://android.jlelse.eu/android-pro-tip-1-443f423b4de6)
>
> 原文 (Medium)：[Android Dev Tip #2](https://android.jlelse.eu/android-dev-tip-2-b1e97bd3ad5b)
>
> 原文 (Medium)：[Android Dev Tip #3](https://android.jlelse.eu/android-dev-tip-3-99da754151ad)
>
> 原文 (Medium)：[Android Dev Tip #4](https://android.jlelse.eu/android-dev-tip-4-91b7757b1f0a)
>
> 原文 (Medium)：[Android Dev Tip #5](https://android.jlelse.eu/android-dev-tip-5-55226527e780)
>
> 原文 (Medium)：[Android Dev-Tip #6](https://android.jlelse.eu/android-dev-tip-6-e2d967935756)
>
> 作者：[Bartek Lipinski](https://android.jlelse.eu/@blipinsk?source=post_header_lockup)

[TOC]

# #1

> 保持 ViewGroup 引用



## Tip:

如果你打算保留对任何 ViewGroup（LinearLayout，FrameLayout，RelativeLayout 等）的引用，并且不想使用特定于此特定布局类型的任何方法，请将其保留为 ViewGroup 对象。

## Explanation:

一路走来，你可能会对布局 xml 进行很多改变，包括改变一个 ViewGroup 的实际类型（例如，你决定 FrameLayout 不再适合你，而实际上你需要一个 RelativeLayout）。 将 Layout 对象保持为 ViewGroup（在可能的情况下）将减少 ViewGroup 强制转换的问题，并且可以更轻松地将布局 xml 中的更改传播给正在使用它的任何类。

Important: 如果你正在使用 Butterknife（而且说实话，你可能是这样），这种错误不是编译器能够发现的，这在运行时会暴露在你的面前。 所以如果你的应用程序拥有大量的资源，并且构建/安装时间真的非常长，那么这种错误尤其令人讨厌。 使用 ViewGroup 引用是一个非常简单的方法来减少你可能花费在错误跟踪上的时间。

## Example:

如果你需要一个 FrameLayout 的引用，但你不需要调用这些方法（API 16的 FrameLayout 特有的方法）：

1. [generateLayoutParams](https://developer.android.com/reference/android/widget/FrameLayout.html#generateLayoutParams%28android.util.AttributeSet%29)([AttributeSet](https://developer.android.com/reference/android/util/AttributeSet.html) attrs)
2. [getConsiderGoneChildrenWhenMeasuring](https://developer.android.com/reference/android/widget/FrameLayout.html#getConsiderGoneChildrenWhenMeasuring%28%29)( )
3. [getMeasureAllChildren](https://developer.android.com/reference/android/widget/FrameLayout.html#getMeasureAllChildren%28%29)( )
4. [setForegroundGravity](https://developer.android.com/reference/android/widget/FrameLayout.html#setForegroundGravity%28int%29)(int foregroundGravity)
5. [setMeasureAllChildren](https://developer.android.com/reference/android/widget/FrameLayout.html#setMeasureAllChildren%28boolean%29)(boolean measureAll)
6. [shouldDelayChildPressedState](https://developer.android.com/reference/android/widget/FrameLayout.html#shouldDelayChildPressedState%28%29)( )

只要保持它作为一个 ViewGroup 对象。

如果你的代码如下所示：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotu83kfsj30m80b60up.jpg)

你决定改变你的 FrameLayout 为 RelativeLayout：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotuckmlej30m803u74k.jpg)

而且你不记得更改 exampleLayout 字段类型，那么在运行应用程序时就会看到（更多）

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotuhhqelj30rs08wn1q.jpg)

一个简单的技巧就是在你的 exampleLayout 字段中使用 ViewGroup 而不是 FrameLayout，你甚至不用担心代码中的这种东西。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotul8ll7j30m80bfq4s.jpg)



# #2

> @IntDef



## Tip:

使用 @IntDef，不仅你的代码更具可读性，而且 lint 可以防止你犯错误，所以你可以更快地编写代码。

## Explanation:

@Idef 是我最喜欢的支持注解包之一。它的主要目标是为一个特定的整型变量指定可能的值，但它可以做的远不止这些。

不仅它可以让你轻松地将你的枚举改成一堆整数（＃perfmatters / #enummatters 选择你喜欢的更多），而且仍然有用。 它与 Android Studio（版本 > = 2.0）的互操作提供了一种加速工作的方法。 特别是如果你是那种喜欢他/她的转换语句的人。

## Example 1:

使用 @IntDef 注解将枚举转换为一组整数：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotur32d6j30m805zdh5.jpg)

除了性能差异（如果实际上有任何差异），与简单的枚举相比，这可能看起来像是大量的代码编写，但请记住，这些整数...实际上是整数。 他们不一定只是一些虚拟的价值观。 你可以使用实际上代表某种信息的值（只要它们是唯一的），包括资源 ID！ 如果你这样想，并且试图比较一个存储这种数据的枚举到 @IntDef 结构，差别就不大了：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotv11pigj30rs06jjuc.jpg)

如果你担心编写 @IndDef 的烦人的过程，请查看我的[其他文章](https://android.jlelse.eu/ctrl-g-d94c88cd4475#.inwvhu6uk)，我将解释如何使用 Multicursor 功能来加速这一过程。

## Example 2:

如果你有一堆 int 常量，你不想重写为一个枚举（比如系统常量），并且你通常在 switch 语句中使用它们，@IntDef 可以帮助你。 你可以用 @IntDef 接口包装一次，然后在代码中的所有 switch 语句中重用它。

这是一个例子：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotvjexyyj30m80bdn0k.jpg)

然后，每当你编写一个 OnTouchListener 时，只要让 Android Studio 帮助你创建 switch 语句：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotvwgsczg30ec0cln8d.gif)

对我来说，@Idef 的重用已经到了一个地步，这可以从 Jake 本人的引用中得到最好的描述：

> 我将这个类复制到我制作的所有小应用程序中。我厌倦了这样做。现在是一个库,
>
> 所以这里是： [IntDefs](https://github.com/blipinsk/IntDefs)。任何对它的贡献都是值得欢迎的！。



#  #3

> 在渐变中使用 @android：color / transparent

## Tip:

如果你在 xml 中创建了一个渐变，其中一部分是完全透明的，那么在使用 @android：color / transparent 时要特别小心。

## Explanation:

绘制渐变时，Android 框架将采用两种颜色来表示渐变区域的两个边（startColor - centerColor 或 centerColor - endColor 或 startColor - endColor，当没有指定 centerColor 时），并在它们之间插值。

这意味着该框架将获取颜色值（Alpha，Red，Green 和 Blue）的所有四个分量，并针对渐变的特定部分插入每个分量。

以下是在两种颜色之间创建渐变的示例：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotw10l15j30hx06y0sr.jpg)

如果他们之间只有3个步骤（颜色）。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotw46eerj30m804hq2t.jpg)

特定颜色分量（A，R，G 和 B）在梯度的特定阶段的值可以表示为：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotw754wej30m80f5glv.jpg)

当使用 @android：color / transparent 时，你需要记住它代表了一个完全透明的特定颜色版本（不透明度设置为 0％;一个组件等于＃00）。 当你计算梯度值时，你不能忘记这个颜色有自己的 RGB 分量。

如果你查看 Color 类（从 android.graphics），你可以看到 Color.TRANSPARENT（代表与 @android：color / transparent 相同的东西）等于0。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotwboii0j30ea07ggo7.jpg)

0的十六进制表示是＃00000000，这意味着 Color.TRANSPARENT 本质上是一个完全透明的 Color.BLACK。

## Example:

当创建一个简单的绿色到透明的渐变，你可能会试图使用这样的东西：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotweo70ej30m804b3zb.jpg)

视觉效果是：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotwhyb7hj308w08w3ya.jpg)

看起来很丑，不是吗？绿色（#FF 阿尔法通道意味着 100％ 不透明度）的梯度变换＃FF27AE60 完全透明黑色（＃00000000）。看看颜色的特定组件如何变化：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotwlynhvj30ir0cmaa8.jpg)

>（右下角的框是完全透明的，因为它是黑色的，不透明度为0％）

如果你想一下，你应该认识到，创建一个简单的 COLOR-TO-CRARENT 渐变不应该修改 RGB 值。它只应该影响Alpha 组件。这显然不是这种情况。

当使用 @android：color / transparent 时，我们不能忘记在计算渐变的颜色值时也会考虑这种颜色的红色，绿色和蓝色。不仅是 Alpha 通道。

生成这种梯度的正确方法是使用这个：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotwqfx7wj30lo046glx.jpg)

使用完全相同的颜色，但将不透明度（Alpha）值更改为＃00。你会达到的效果是（如预期）：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotwtk84hj308w08sglm.jpg)

而组件会按以下方式变化：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotwxp4mgj30if0cxwen.jpg)

>（25％ 不透明度和 0％ 之间的区别似乎比其他区别更大，但这是完全正确的...请记住，它基本上是从颜色跳到没有颜色，而其他颜色则从颜色变成更透明的颜色）

R，G 和 B 组件不变。只有 Alpha 通道受到渐变的影响。



# #4

> 在 RecyclerView 中包装内容的性能



## Tip:

当 RecyclerView 包装其内容时，它不再被回收。只要 RecyclerView 在布局层次结构中，数据集中的每条记录都有一个 View 保存在内存中。

## Explanation:

RecyclerView 的基本思想非常简单。

1. 你有一个可滚动的区域充满了项目的视图。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotx1we6oj30rs0d7jra.jpg)

2. 当一个项目从可见区域滚动时，RecyclerView 将其从组件中取出。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotx4tizqj30rs0d7glj.jpg)

3. 取出的项目，可以用来显示另一个项目的视图，来到滚动的可见区域。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotx86o99j30rs0d7t8m.jpg)

当你想要 RecyclerView 来包装它的内容时，整个事情变得非常不直观。 在这种情况下，RecyclerView 的大小成为所有项目视图的大小。 因此，RecyclerView 本身不再可以滚动，并且从 RecyclerView 的可见区域不能滚动任何项目。 RecyclerView 将整个区域视为可见区域。

所有这些都导致一个简单的事实：如果没有项目可以滚动，那么没有项目可以被重新使用。他们都需要在任何时候都可用。

只要你的 RecyclerView 位于布局层次结构中，所有的项目视图都保存在内存中。

如果这仍然不会引起你的红旗，那么让我这样说：如果你的 RecyclerView 为你的数据库表的每一行显示一个项目，那么如果这个表有726行，你将最终有726个项目视图 一次加载你的内存。

从本质上讲，最终你会得到一个组件，它可以使用某种数据集的适配器创建许多视图。 但是，视图的性能与在 ScrollView 中手动添加所有视图相同。 RecyclerView 没有任何性能收益。

## Example:

假设布局：

```xml
<android.support.v4.widget.NestedScrollView
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <ImageView
           android:layout_width="match_parent"
           android:layout_height="100dp"
           android:scaleType="centerCrop"
           android:src="@drawable/test_image"/>
        
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="#3F51B5"
            android:text="Example"/>

        <android.support.v7.widget.RecyclerView
            android:id="@+id/recycler"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"/>
    </LinearLayout>
</android.support.v4.widget.NestedScrollView>
```

在 RecyclerView 中附加的项目：

```xml
<TextView
    android:id="@+id/text"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

如果 RecyclerView 的适配器在其数据集中有30条记录，屏幕可能会看起来（和工作）是这样的：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotxemifsj30rs0g9go7.jpg)

乍一看，一切看起来不错，但是如果你看一下布局层次结构（在 Layout Inspector 工具中），你可以看到：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotxqc3qwj30gg0uuaiu.jpg)

RecyclerView 中的每一个 View 都始终保存在内存中。包括当前不可见的多个项目视图（使用 NestedScrollView 一直滚动到顶部的布局层次结构捕获）。

当没有 NestedScrollView 并且 RecyclerView 没有包装它的内容（相同的数据集）时，你可以把它和一个等价的情况进行比较：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotxxl2roj30gg0d2mzo.jpg)

在任何时候，保存在内存中的唯一视图表示当前在 RecyclerView 的 Scroll 界限内的项目。



# #5

> tools:parentTag



## Tip:

当使用合并作为复合视图布局的根元素时，tools:parentTag 可以帮助你利用设计工具的所有功能。

## Explanation:

大多数情况下，在创建复合视图时，要将布局xml直接附加到创建的 ViewGroup 中。 不介绍任何额外的布局之间。 这就是为什么你应该非常熟悉 merge 标签如何在布局 xmls 中工作。 或者至少（可能）是这个标签最典型的用例。

理解它的最简单的方法是：

- 创建复合视图时：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1froty2gezoj30m80ch0ve.jpg)

- R.layout.layout_example_compound_view 是：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froty73hl7j30m80bngo9.jpg)

- ExampleCompoundView 附加时：

这个：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotyaqtokj30m80bngoi.jpg)

变成这样：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotyfkws0j30m80bnq5p.jpg)

默认情况下，你的布局设计工具不知道你的合并标签变成了什么样的最终类型。这可能有点烦人，如果：

a）你喜欢使用设计工具来设置一些 View 的参数。 

要么

b）你喜欢看设计可视化，看看你在 xml 中做的是否正确。

这里是（工具名称空间的）parentTag 参数派上用场的地方。它为你提供了一种告诉设计工具的方法，合并标记的最终 ViewGroup 是什么。

```xml
tools:parentTag="your.package.ExampleCompoundView"
```

（你还需要将 android：layout_width，android：layout_height 添加到你的合并标记中，设计工具才能正常工作）。

## Example:

对于布局：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotyqtoqfj30m80cxadl.jpg)

设计工具如下所示：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotyxnf14j30m80dzaac.jpg)

它不知道 merge 标签表示一个 ConstraintLayout，因此它不能考虑任何封闭的 ImageView 的布局参数（因此不能显示任何东西）。

如果将 parentTag 参数添加到布局（+ layout_width 和 layout_height）：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotz2jb5sj30m80epgpr.jpg)

设计工具将会明白 merge 标签确实是一个 ConstraintLayout，它将相应地显示封闭的元素：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotza4lh4j30m80e10xk.jpg)

请记住，它不必完全是 ConstraintLayout。它可以是任何其他 ViewGroup（包括将扩展某个 ViewGroup 的任何复合视图）。



# #6

> Layout comments



## Tip:

如果你的布局 xml 的某些部分没有清除掉（例如看似不必要的视图或奇怪的参数），那么用注释标记你的意图（在这个 xml 中）。

## Explanation:

你可以把这个技巧当作 [Robert C. Martin’s Clean Code](http://amzn.eu/bCZuEzS) 一书的意向解释部分（章节：Comments）的解释。

有了 Android SDK 的细节，如果考虑布局 xmls，这个提示会有特殊的含义。

你可以在布局 xml 中设置的东西有时对于你的情况是不够的。 例如。 你可能需要引入一个看起来不必要的视图，使你的布局按预期工作，或者从代码中调整视图的一些参数，使其看起来完全正确。

在一个特定的视图中添加一个简单的注释（在 xml 中）是一个简单的解决方案，对于将来你或者其他需要使用代码的开发人员来说是无价的。 通过阅读提示可以节省数小时的时间来了解看似破碎的布局。

## Example 1:

假设你不能仅依赖维度来计算 View 的填充值。 你需要使用你的代码的几行来做到这一点。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou0ab07mj30m8057my9.jpg)

添加一个简单的评论到你的布局 xml，所以当你回到你的应用程序的这一部分，你不需要花费时间搞清楚这个 FrameLayout 的填充源。

## Example 2:

See [Saul](https://medium.com/@saulmm2)’s [response to the post](https://medium.com/@saulmm2/agree-with-the-tip-but-adding-my-point-i-think-that-because-an-xml-constraint-some-comments-cant-7914653fecfd).

同意提示，但加入我的观点我认为，因为 XML 约束，有些评论不能像它们那样有意义，因为行注释不存在于基于标记的语言上。

Example:

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frou0dzyz8j30dz07r3z0.jpg)

在这个布局中，我重复使用了一个 @layout / view_price_Header，这个对于 land 和 portrait 的配置是一样的。

tools:ignore 属性背后的原因是因为该工具无法识别引用的视图，所以显示了一个 lint。

对其他人来说，这可能不是一眼就能理解的，所以在这里解释原因是有意义的。

```xml
<!-- Known at @layout/view_price_header -->
```

在一个理想的世界里，我会添加这条评论的权利：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou0natdxj30j007umxr.jpg)

> 工具属性的解释性评论，编译失败

这在编译时会失败，因为在评论之后，其余属性将被误解。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou0x3itbj30f2085dge.jpg)

> 工具属性的解释性注释在编译的上下文外

唯一的解决方案是在标签的末尾写下该评论，并且可能会出现在上下文之外，并且没有意义。



# #7

1 - 简单使用：[tools:listitem](https://developer.android.com/studio/write/tool-attributes.html#toolslistitem_toolslistheader_toolslistfooter)，允许你在 Android Studio 布局编辑器中替换 “List Item” 或 “Recycler View” 行的随机行。

通过在我的 Recycler View 中添加这行代码，如下所示：

```xml
tools:listitem="@layout/item_subject_row"
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou11zr36j30m80h6gn5.jpg)

我的布局不再显示随机线，而是我为实际应用程序构建的真实项目，直接显示在编辑器内部。

> 这不是很简单吗？但更多的来

2 -  [Nicolas Roard](https://medium.com/@camaelon) 也谈到了 Sample Data。

从 Android Studio 3.0 开始，你可以使用 Android Studio 提供的预定义数据，或创建一个 “样本数据” 的特定文件夹，你可以直接在布局编辑器中将假数据直接显示在布局中。

预定义数据

Android studio 3.0 通过工具属性带来了一些预定义的数据，可以轻松地将你的布局结构可视化。

在 `tools:text` 属性里面，只需使用 `@tools/data/ `，例如：

```xml
tools:text="@tools:sample/last_names"
```

Android Studio正在为你提供一个列表，其中包括：

last_names,  first_names,  full_names,  cities,  avatars,  backgrounds/scnenic,  date/ddmmyy,  date/day_of_week,  date/ddmm,  date/hhmmss,  date/mmddyy,  lorem,  orem/ramdom,  us_phones,  us_zipcodes



我自己的样本数据

要创建你的假/示例数据文件夹，只需右键单击 “应用程序” 文件夹，然后 “new > Sample Data directory”

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou17628nj30m80b475z.jpg)

它将创建一个 “sampledata” 文件夹，你可以在简单文件中放置数据。

在你的文件夹内，添加纯文本文件，你可以将原始数据一行一行（可以是颜色 ＃ff33aa 或只是文本等）。然后在布局中，通过引用 `@sample / nameOfTheFileGiven` 访问工具：`tools:text` 

```xml
tools:text="@sample/my-subjects"
```

> 但更好！你可以使用 Json 对象来显示更复杂的数据

在我的 “sampledata” 文件夹中，我已经创建了一个文件 calles subects.json，我把这些数据放在里面。

```json
{
  "data" :[
    {"idweb": -1, "sourceId" : 1, "name" : "AIRCRAFT ACCIDENT ..."},
    {"idweb": 1, "sourceId" : 1, "name" : "ANNEX 1 ..."},
    {"idweb": 2, "sourceId" : 1, "name" : "ANNEX 11 ..."},
    {"idweb": 3, "sourceId" : 1, "name" : "ANNEX 12 ..."}
  ]
}
```

棘手的事情困扰了我片刻，它不能通过表启动你的 Json 文件。它必须是根的 Json 对象。并且必须编译你的项目以查看你的新/更新的数据

然后你访问你的数据，如：

```xml
tools:text="@sample/subjects.json/data/name"
```

Android Studio 是非常有用的，通常，只要你输入@sample /（不要忘记首先创建项目），就可以直接向你展示从文件中知道的所有选项。

改变后：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou1brnltj30m80h6wg4.jpg)



# #8

>  [Timber](https://github.com/JakeWharton/timber)

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou1evpcrj30in02n0t1.jpg)

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import timber.log.Timber;

public class LinkingDebugTree extends Timber.DebugTree {

    private static final int CALL_STACK_INDEX = 4;
    private static final Pattern ANONYMOUS_CLASS = Pattern.compile("(\\$\\d+)+$");

    @Override
    protected void log(int priority, String tag, String message, Throwable t) {
        // DO NOT switch this to Thread.getCurrentThread().getStackTrace(). The test will pass
        // because Robolectric runs them on the JVM but on Android the elements are different.
        StackTraceElement[] stackTrace = new Throwable().getStackTrace();
        if (stackTrace.length <= CALL_STACK_INDEX) {
            throw new IllegalStateException(
                    "Synthetic stacktrace didn't have enough elements: are you using proguard?");
        }
        String clazz = extractClassName(stackTrace[CALL_STACK_INDEX]);
        int lineNumber = stackTrace[CALL_STACK_INDEX].getLineNumber();
        message = ".(" + clazz + ".java:" + lineNumber + ") - " + message;
        super.log(priority, tag, message, t);
    }

    /**
     * Extract the class name without any anonymous class suffixes (e.g., {@code Foo$1}
     * becomes {@code Foo}).
     */
    private String extractClassName(StackTraceElement element) {
        String tag = element.getClassName();
        Matcher m = ANONYMOUS_CLASS.matcher(tag);
        if (m.find()) {
            tag = m.replaceAll("");
        }
        return tag.substring(tag.lastIndexOf('.') + 1);
    }
}
```

