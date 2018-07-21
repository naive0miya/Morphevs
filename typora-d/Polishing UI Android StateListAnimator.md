# 打磨用户界面: Android StateListAnimator

> 原文 (Medium)：[Polishing UI: Android StateListAnimator](https://android.jlelse.eu/polishing-ui-android-statelistanimator-7b74a06b85a5)
>
> 作者：[Mert Şimşek](https://android.jlelse.eu/@iammert?source=post_header_lockup)

[TOC]

大多数时候，我们没有花时间开发我们的 Android 应用程序的 UI，我们只是拖拽视图并开始编写我们的应用程序。 我注意到，我们大多数人并不太关心用户界面。 我认为这是错误的。 移动开发者也应该关心用户界面和用户体验。 我不是说"成为移动用户界面的专家"，但我们应该理解设计语言和它的概念。 

之前我写了一些关于材料设计阴影的文章，并得到了很多好的反馈。我要感谢你们所有人。 “[Mastering Shadows in Android](https://android.jlelse.eu/mastering-shadows-in-android-e883ad2c9d5b)” 解释了 android 中的高度和阴影。而且我还将这些更改应用于我的开放源代码用户界面库。 （[Scaling Layout](https://github.com/iammert/ScalingLayout)）。

在这篇文章中，我还想用 StateListAnimator 来完善我的库，并一步一步向你展示我是如何实现这一目标的。

## 目录

这篇博文包括以下主题。

- Drawable States
- [StateListDrawable](https://developer.android.com/reference/android/graphics/drawable/StateListDrawable.html)
- [Property Animation](https://developer.android.com/guide/topics/graphics/prop-animation.html)
- [StateListAnimator](https://developer.android.com/reference/android/animation/StateListAnimator.html)
- ScalingLayout with StateListAnimator

## Drawable States

在 Android 中，Drawable 有17种不同的状态。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frote1jtxvj308108v0sy.jpg)

也许我们甚至不知道其中的一些。我不打算深入每个状态。大多数情况下，我们使用 pressed, enabled, windows focused, checked 等。如果我们没有声明任何可绘制状态，这意味着 Android 中的默认状态。

 我们需要了解这些状态来编写我们的自定义 StateListDrawable。

## StateListDrawable

这基本上是 drawable  项目的列表，每个 drawable  项目都有自己的状态。我们在 res / drawable 文件夹下创建一个 XML 文件来创建 StateListDrawable。

```xml
<item android:drawable="@drawable/i" android:state_pressed="true"/>
```

这是一个项目。它有两个属性。drawable 和 state_pressed。

```xml
<selector>
    <item
        android:drawable="@drawable/p"
        android:state_pressed="true"/>
    <item
        android:drawable="@drawable/default"/>
</selector>
```

这是一个 StateListDrawable。如果我们没有为一个项目声明任何状态，就像我之前说过的那样，这意味着默认状态。

## 我可以使用 ShapeDrawable 吗？

当然。但不是使用 android：drawable，你可以将自定义形状添加到你的项目。这是带有 ShapeDrawable 的项目。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotebwj3kg30rs0eogxj.gif)

你可以使用 StateListDrawable 在 API 级别1。因此，StateListDrawable 上没有 API 级别限制。

```xml
<View
    android:layout_width="50dp"
    android:layout_height="50dp"
    android:foreground="@drawable/state_list_drawable"
    android:clickable="true"/>
```

就是这样。 现在我们的视图已经有状态。当用户按下它时，颜色将会改变。并且用户释放它，它将具有默认状态和颜色。

>可是等等。clickable？为什么我们添加该属性？我们是否应该添加？当然。但只适用于自定义视图。这需要一些时间才能发现。按钮的工作原理完美无缺，因为默认情况下它是可点击的。但是，如果你想为 View，ImageView，Custom View 等使用 StateListDrawable，则需要添加 clickable 属性。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotelwcfqg30go07rgxf.gif)

我添加了 StateListDrawable，这里是[提交](https://github.com/iammert/ScalingLayout/commit/58ebe38db74dee7f6c7c87f1694829e09604be7c)。这就像我上面举的例子。当用户点击布局时，布局变为彩色。但是让我们用 StateListAnimator 改善它。

## StateListAnimator

记住当你点击 FloatingActionButton 时，它的 Z 值随动画增加。它的幕后是 StateListAnimator。一些材质设计小部件在内部拥有自己的 StateListAnimator。

让我们用这个问题来说明一下。 

如果材质设计窗口小部件在内部有自己的 StateListAnimator，我们可以将它们设置为 null 来移除该功能（不推荐，它是为某个原因而开发的）。现在回答听起来更合理。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1froteso1yug30go07rgxf.gif)

那么，我们如何创建一个？

要了解 StateListAnimator，我们需要了解属性动画。我不会在这篇博文中深入探索属性动画。但至少，我想告诉你基本知识。

### Property Animation

这是对象中属性的最基本示例。 X 是一个属性。

```java
class MyObject{

    private int x;

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }
}
```

> 动画系统是一个强大的框架，可以让你几乎所有的东西动画。你可以定义动画来随时间改变任何对象属性，而不管它是否绘制到屏幕上。属性动画会在指定的时间长度内更改属性的（对象中的字段）值。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotewegtzj30gl04gt8y.jpg)

X 是一个属性。 T 是一个时间。在属性动画中，X 在给定时间内更新。这基本上是属性动画的工作原理。该框可以是视图或任何对象。

**ValueAnimator** 是属性动画的基类。你可以设置更新侦听器来为动画设定器赋值并观察属性更改。

**ObjectAnimator** 是一个从 ValueAnimator 扩展的类。如果你有以下条件，你可以使用 ObjectAnimator;

- 你有一个对象(任何类都有一些属性) 
- 你不想观察值动画侦听器
- 你想自动更新对象的属性

因此，如果我们有一个视图（这是一个对象），并且我们想更新视图的属性（x 坐标，y 坐标，旋转，平移或视图具有属性的 getter / setter 的任何属性），我们可以使用 ObjectAnimator。让我们继续创建 StateListAnimator。

```xml
<selector>

    <item android:state_pressed="true">
        <objectAnimator
            android:duration="200"
            android:propertyName="translationZ"
            android:valueTo="6dp"
            android:valueType="floatType" />
    </item>

    <item>
        <objectAnimator
            android:duration="200"
            android:propertyName="translationZ"
            android:valueTo="0dp"
            android:valueType="floatType"/>
    </item>

</selector>
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotf02y5fj30m506pa9y.jpg)

正如我之前所说的，我们可以直接使用对象的属性，而无需观察动画的变化。每个视图都有 translationZ 属性。所以我们可以使用 ObjectAnimator 来直接动画 translationZ。

我们还可以在 \<set> 中组合多个 \<objectAnimator>。让我们改变视图的另一个属性。缩放 X 和 缩放 Y.

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">

    <item android:state_pressed="true">
        <set>
            <objectAnimator
                android:duration="200"
                android:propertyName="translationZ"
                android:valueTo="6dp"
                android:valueType="floatType" />

            <objectAnimator
                android:duration="200"
                android:propertyName="scaleX"
                android:valueTo="1.1"
                android:valueType="floatType" />

            <objectAnimator
                android:duration="200"
                android:propertyName="scaleY"
                android:valueTo="1.1"
                android:valueType="floatType" />

        </set>
    </item>

    <item>
        <set>
            <objectAnimator
                android:duration="200"
                android:propertyName="translationZ"
                android:valueTo="0dp"
                android:valueType="floatType" />

            <objectAnimator
                android:duration="200"
                android:propertyName="scaleX"
                android:valueTo="1"
                android:valueType="floatType" />

            <objectAnimator
                android:duration="200"
                android:propertyName="scaleY"
                android:valueTo="1"
                android:valueType="floatType" />
        </set>
    </item>

</selector>
```

结果如下！现在，当用户按下时它也被放大。这是[提交](https://github.com/iammert/ScalingLayout/commit/d1906af5fdd45a823740842dd9c25a97b18cc8c3#diff-99440eb578ffa46f57b794eb48783a8d)。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frotfmxsvgg30go06idtk.gif)

你还可以在 animator.xml 中定义其他属性。在这里你可以找到更多关于使用 ObjectAnimator 的信息。

就这样。我打算写一些关于 ValueAnimator 和 ObjectAnimator 的东西。它是一个伟大的 API 动画物体。 

## 阅读

**Property Animation | Android Developers** The property animation system is a robust framework that allows you to animate almost anything. You can define an… | [developer.android.com](https://developer.android.com/guide/topics/graphics/prop-animation.html)

**ObjectAnimator | Android Developers** Edit description | [developer.android.com](https://developer.android.com/reference/android/animation/ObjectAnimator.html)

**StateListAnimator** One of the fundamental principles of Material Design is "motion provides meaning" and one important area where this… | [blog.stylingandroid.com](https://blog.stylingandroid.com/statelistanimator/)

