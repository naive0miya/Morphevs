# Android 隐藏的 api 第2部分ー线性布局的分割线

> 原文 (Medium)：[Hidden APIs of Android, Part 2— LinearLayout dividers](https://medium.com/@azizbekian/hidden-apis-of-android-part-2-linearlayout-dividers-b564064570eb)
>
> 作者：[Andranik Azizbekian](https://medium.com/@azizbekian?source=post_header_lockup)

[TOC]

在这个系列中，我将讨论 Android 的 API，尽管对 Android SDK 有丰富的经验，但大多数开发人员并不知道他们的存在。

在本集中，我们将深入探讨 LinearLayout 的一个功能，这是我从未见过任何人使用的功能。

设想一个简单的水平 LinearLayout，其中显示了5个花式方块：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frow64vmtdj30j204g0ja.jpg)

现在需要在这5个视图之间引入一个分隔符，如下所示：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frow674tn8j30j204g0jq.jpg)

你将如何解决这个问题？

你会在每个这些方块之间引入一个新的分隔视图吗？

```xml
<LinearLayout ...>
    <View .../> 
    <include layout="@layout/divider"/>
    <View .../> 
    <include layout="@layout/divider"/>
    <View .../>
    <include layout="@layout/divider"/>
    <View .../>
    <include layout="@layout/divider"/>
    <View .../>
</LinearLayout>
```

当然，这可以解决问题。 但是线性布局在内部具有这样一个特性。欢迎 [android：divider](https://developer.android.com/reference/android/widget/LinearLayout.html#attr_android:divider) 标签：

```xml
<LinearLayout
    ...
    android:divider="@drawable/divider"
    android:showDividers="middle">
 
    <View .../>
    <View .../>
    <View .../>
    <View .../>
    <View .../>
</LinearLayout>
```

应用 android：divider  - android：showDividers 也应该设置，否则它默认为 none。共有四个选项：`middle` ，`beginning` ，`end` 和 `none` 。

正如你已经猜到的那样，`beginning` 和 `end` 将分别只应用于开始和结束。

