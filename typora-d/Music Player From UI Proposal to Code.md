# 音乐播放器: 从用户界面提案到代码

> 原文 (Medium)：[Music Player: From UI Proposal to Code](https://stories.uplabs.com/music-player-3a85864d6df7)
>
> 作者：[André Mion](https://stories.uplabs.com/@andremion?source=post_header_lockup)

[TOC]

有些开发人员在用户界面方案有点"复杂"或"复杂"时，很难编写代码。 他们中的许多人在编码时会剥夺很大一部分用户界面，甚至是动议，最终结果与原来的提案大不相同。 

这篇文章讲述了如何编写一个用户界面提案，跳过一些基本的 Android 细节，专注于过渡和动画方法。 

## [MaterialUp](http://www.materialup.com/)

> 一个伟大的网站，设计师和开发者可以找到并共享资源来使用 Material Design 来构建应用程序和网站。 有很多用户界面、实验、开源应用程序、库以及你可以在 Android、 Web 和 iOS 上找到的即用产品。 

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbxa54osyg30m80gou0x.gif)

浏览该网站，你可以找到由 [Anish Chandran](https://dribbble.com/anish_chandran) 创建的名为[音乐播放器](https://dribbble.com/shots/1850527-Music-Player-Transition)的用户界面资源。

这个提议给了我们一个很好的例子，说明一个音乐播放器应用程序如何以流畅和一致的方式使用 Material 和 Motion 设计。 

Music Player Demo - Android Apps on Google Play From UI Proposal to Code | [play.google.com](https://play.google.com/store/apps/details?id=com.sample.andremion.musicplayer)

## 热身

首先，我们需要做一些有助于我们编码这些 motion 的事情。 

把运动的提案分成几部分 。

将动画提案文件转换为单个帧。 这将帮助我们查看动画和过渡的每一步。 

按类型分开

我们有很多视图在同一时间进行过渡和动画，并且认为如何以这种方式编写代码将是非常困难的。 我们可以通过类型来分离这些过渡和动画，例如: 视图滑到底部，视图消失，视图移出到新活动等等。 

下一个技巧是在每个布局中使用的一个很好的技巧，不论是否存在运动。

简化你的视图层次结构

尽可能简化创建视图层次结构，避免在同一个布局中使用大量视图组。这将简化过渡编排，将有助于维护，并且主要在动画期间显着提高应用性能。

## 黑科技

在布局文件中，某些视图组将 android：transitionGroup 属性设置为 true 因为在播放列表信息容器（主布局文件）或控件容器（详细布局文件）中，它们需要在活动转换期间视为单个实体。

```xml
<RelativeLayout
    android:id="@+id/playlist"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_below="@id/cover"
    android:gravity="center_vertical"
    android:padding="@dimen/activity_vertical_margin"
    android:transitionGroup="true">
…

<LinearLayout
    android:id="@+id/controls"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_alignParentBottom="true"
    android:gravity="center_horizontal"
    android:transitionGroup="true"
    app:layout_marginBottomPercent="5%">
…
```

在 styles.xml 中，我们有主活动和详细活动中使用的主题。

- AppTheme.Main

```xml
<item name="android:windowSharedElementsUseOverlay">false</item>
```

禁用共享元素视图的覆盖。在此音乐播放器布局中，我们需要在共享元素视图从主活动移动到细节活动时禁用覆盖。如果启用，某些共享元素视图可能会以错误的方式覆盖其他视图。

```xml
<item name="android:windowExitTransition">@transition/list_content_exit_transition</item>
<item name="android:windowReenterTransition">@transition/list_content_reenter_transition</item>
```

 ![](https://ws4.sinaimg.cn/large/006tKfTcgy1frot2o2vi3g306y0cc484.gif)

它在列表内容的退出和重新进入过渡中具有相同的过渡方法。

[list_content_exit_reenter_transition.xml](https://gist.github.com/andremion/358e43666bd5679dc71cd955af10a927#file-list_content_exit_reenter_transition-xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@integer/anim_duration_default"
    
    // 1
    android:startDelay="@integer/anim_duration_default">

    // 2
    <fade>
        <targets>
            <target android:targetId="@id/pane" />
        </targets>
    </fade>
    
    // 3
    <slide android:slideEdge="bottom">
        <targets>
            <target android:excludeId="@android:id/statusBarBackground" />
            <target android:excludeId="@id/pane" />
            <target android:excludeId="@android:id/navigationBarBackground" />
        </targets>
    </slide>

</transitionSet>
```

- 设置一个启动延迟，以同步这些过渡与 FAB 变形动画 
- 淡出由 targetId 属性指定的窗格视图。
- 将 RecyclerView 子视图和播放列表信息容器滑动到底部，不包括由 excludeId 属性指定的状态栏，窗格和导航栏。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frot2vad76g306y0ccgtx.gif)

它在列表内容的退出和重新进入转换中具有几乎相同的转换方法。

[list_shared_element_exit_reenter_transition.xml](https://gist.github.com/andremion/7dff009434c75a7c74c0ff1954e42753#file-list_shared_element_exit_reenter_transition-xml) 

```xml
<item name="android:windowSharedElementExitTransition">@transition/list_shared_element_exit_transition</item>
<item name="android:windowSharedElementReenterTransition">@transition/list_shared_element_reenter_transition</item>

```

- [PlayButtonTransition](https://github.com/andremion/Music-Player/blob/master/app/src/main/java/com/sample/andremion/musicplayer/transition/PlayButtonTransition.java) 是一个自定义过渡，它包装 [AnimatedVectorDrawable](https://github.com/andremion/Android-Animated-Icons) 并用于将播放图标转变为暂停图标，反之亦然，具体取决于模式值。
- AppTheme.Detail

```xml
<item name="android:windowEnterTransition">@transition/detail_content_enter_transition</item>
<item name="android:windowReturnTransition">@transition/detail_content_return_transition</item>
```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frot31rjs2g306y0cc78t.gif)

它在细节内容的进入和退出过渡中具有相同的过渡方法。

[detail_content_enter_return_transition.xml](https://gist.github.com/andremion/88bdc5d07b81a749c6454303d1f91035#file-detail_content_enter_return_transition-xml) 

```xml
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@integer/anim_duration_default">

    // 1
    <fade>
        <targets>
            <target android:targetId="@id/ordering" />
        </targets>
    </fade>
    
    // 2
    <slide android:slideEdge="bottom">
        <targets>
            <target android:targetId="@id/controls" />
        </targets>
    </slide>

</transitionSet>
```

- 淡化由 targetId 属性指定的排序容器。
- 向下滑动仅由 targetId 属性指定的控件容器。

```xml
<item name="android:windowSharedElementEnterTransition">@transition/detail_shared_element_enter_transition</item>

```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frot3uz41sg306y0ccqp5.gif)

[detail_shared_element_enter_transition.xml](https://gist.github.com/andremion/79701230626c13523157a43c0c4b0e79#file-detail_shared_element_enter_transition-xml) 

```xml
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="@integer/anim_duration_default"
    
    // 1
    android:interpolator="@android:interpolator/accelerate_quad">
    
    // 2
    <transition class="com.sample.andremion.musicplayer.transition.ProgressViewTransition" />
    
    // 3
    <transition class="com.sample.andremion.musicplayer.transition.CoverViewTransition" />
    
    // 4
    <transitionSet>
        <changeBounds />
        <changeTransform />
        <changeClipBounds />
        <changeImageTransform />
    </transitionSet>

</transitionSet>
```

- 定义一个插值器的速度变化的过渡，允许一个非线性运动 。
- [ProgressViewTransition](https://github.com/andremion/Music-Player/blob/master/app/src/main/java/com/sample/andremion/musicplayer/transition/ProgressViewTransition.java) 是使用 [ProgressView](https://github.com/andremion/Music-Player/blob/master/app/src/main/java/com/sample/andremion/musicplayer/view/ProgressView.java) 将水平进度视图 “变形” 为弧进度视图的自定义过渡。
- [MusicCoverViewTransition](https://github.com/andremion/Music-Cover-View/blob/master/library/src/main/java/com/andremion/music/MusicCoverViewTransition.java) 是另一种使用 [MusicCoverView](https://github.com/andremion/Music-Cover-View) 将 “矩形” 封面视图 “变形” 为带有轨迹线的封面视图的自定义过渡。
- 使用默认移动转换到其他共享元素视图。

```xml
<item name="android:windowSharedElementReturnTransition">@transition/detail_shared_element_return_transition</item>

```

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frot4m5c7cg306y0cc4qp.gif)

在这个转换中使用了几乎相同的方法 detail_shared_element_enter_transition。但是为了使这个转换与提案相匹配，每个部分都增加了一些延迟。

```xml
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:duration="@integer/anim_duration_default"
    android:interpolator="@android:interpolator/accelerate_quad">

    // 1
    <transitionSet>
        <changeBounds />
        <changeTransform />
        <changeClipBounds />
        <changeImageTransform />
        <transition
            class="com.sample.andremion.musicplayer.transition.ProgressViewTransition"
            app:morph="1" />
        <targets>
            <target android:targetId="@id/progress" />
        </targets>
    </transitionSet>

    // 2
    <transitionSet android:startDelay="@integer/anim_duration_short">
        <changeBounds />
        <changeTransform />
        <changeClipBounds />
        <changeImageTransform />
        <transition
            class="com.sample.andremion.musicplayer.transition.CoverViewTransition"
            app:shape="circle" />
        <targets>
            <target android:targetId="@id/cover" />
        </targets>
    </transitionSet>

    // 3
    <transitionSet android:startDelay="@integer/anim_duration_default">
        <changeBounds />
        <changeTransform />
        <changeClipBounds />
        <changeImageTransform />
    </transitionSet>

</transitionSet>
```

- 反转 “变形” 模式，从圆弧进度视图到水平进度视图。
- 反转 “变形” 模式，从圆形封面视图到矩形封面视图。
- 使用默认移动转换到其他共享元素视图。

## 最后结果

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frot5hnoy0g306y0cc4qp.gif)

最终的结果应该是这样。当然，在最后的项目中可能会漏掉一些微不足道的细节，但这只是一件小事。

在这里探索整个项目：[[PR]](https://github.com/andremion/Music-Player)

## 阅读

**Animated icons on Android** How to improve the user experience using animated icons with vector drawables on Android | [André Mion](https://stories.uplabs.com/animated-icons-on-android-ee635307bd6)

**Applying meaningful motion on Android** How to apply meaningful and delightful motion in a sample Android app | [André Mion](https://blog.prototypr.io/applying-meaningful-motion-on-android-a271a873bd78)

