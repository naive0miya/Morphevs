# InstaMaterial 概念(第11部分) - 加入设计支持库

>原文 (mirekstanek.online) ： [InstaMaterial meets Design Support Library](https://mirekstanek.online/instamaterial-meets-design-support-library/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

几个月前，我开始了 [InstaMaterial 系列](http://frogermcs.github.io/Instagram-with-Material-Design-concept-is-getting-real-the-summary/)。 目标很简单 - 证明在 Material Design 指南中实现所有那些奇特的动画和 UI 效果是非常简单的。 现在变得更加容易了 - Google 为我们提供了 [Android 设计支持库](http://android-developers.blogspot.com/2015/05/android-design-support-library.html)，它提供了 Material Design 中引入的所有最重要的 UI 元素。 在这篇文章中，我希望更新 InstaMaterial 源代码，从自定义视图实现到这个库提供的源代码。

## 在我们开始更新一切之前
设计支持库意味着这里描述的所有代码是过时的吗？ 一点也不。 是的，使用官方的实现总是更好（是吗？），特别是对于标准的用例。 提供的实现也意味着有人关心我们的代码。 我们不必考虑 Kitkat，Lollipop 和 M 之间的浮动操作按钮兼容性问题。 由于这个原因，我们节省了许多代码行，这些代码甚至可以用于更高级的东西(或者只是代码变得更易读)。

但是，嘿！ 有一件重要的事情 - 另一方面，也有像我们这样的程序员。 他们也犯错误。 他们无法预测所有可能的用例。 而且什么是非常重要的 - 有时候最好知道背后的原理的 - 从头开始工作。 只是为了更好地了解“系统工作”。

## 遇见设计支持库
目前有很多描述新设计支持库的文章：
- [Exploring the new Android Design Support Library](https://medium.com/ribot-labs/exploring-the-new-android-design-support-library-b7cda56d2c32)

  有几个帖子 [Antonio Leiva’s blog](http://antonioleiva.com/category/blog/)

- 可能还有更多

这就是为什么这篇文章只是对从自定义实现到设计支持库提供的过渡事物的快速概述。 顺便说一下，我们将看到我们将保存多少行代码。

## NavigationView
![|center](http://frogermcs.github.io/images/16/navigation_view.gif)

默认导航抽屉的材料设计规则非常清晰。 现在感谢 [NavigationView](https://www.google.com/design/spec/patterns/navigation-drawer.html#) 整个菜单可以在 /res/menu/{filename}.xml 文件中实现。 我们需要做的不是自定义视图层次结构的实现，而是在我们的活动中使用这个代码:

```xml
<android.support.v4.widget.DrawerLayout
    android:id="@+id/drawerLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <FrameLayout
        android:id="@+id/flContentRoot"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <android.support.design.widget.NavigationView
        android:id="@+id/vNavigation"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:background="#ffffff"
        app:headerLayout="@layout/view_global_menu_header"
        app:itemIconTint="#8b8b8b"
        app:itemTextColor="#666666"
        app:menu="@menu/drawer_menu" />

</android.support.v4.widget.DrawerLayout>
```
NavigationView 有两个重要的属性：app:headerLayout 和 app:menu。 首先定义一个自定义视图布局，它将作为我们的导航菜单的标题放置。 第二个定义提供菜单元素的资源。 这是菜单实现在我们的应用程序中的样子：
```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <group android:id="@+id/menu_group_1">
        <item
            android:id="@+id/menu_feed"
            android:icon="@drawable/ic_global_menu_feed"
            android:title="My Feed" />
        <item
            android:id="@+id/menu_direct"
            android:icon="@drawable/ic_global_menu_direct"
            android:title="Instagram Direct" />
        <item
            android:id="@+id/menu_news"
            android:icon="@drawable/ic_global_menu_news"
            android:title="News" />
        <item
            android:id="@+id/menu_popular"
            android:icon="@drawable/ic_global_menu_popular"
            android:title="Popular" />
        <item
            android:id="@+id/menu_photos_nearby"
            android:icon="@drawable/ic_global_menu_nearby"
            android:title="Photos Nearby" />
        <item
            android:id="@+id/menu_photo_you_liked"
            android:icon="@drawable/ic_global_menu_likes"
            android:title="Photos You've Liked" />
    </group>
    <group android:id="@+id/menu_group_2">
        <item
            android:id="@+id/menu_settings"
            android:title="Settings" />
        <item
            android:id="@+id/menu_about"
            android:title="About" />
    </group>
</menu>
```
在我们的应用程序中，我们还定义了 BaseDrawerActivity，它自动将导航抽屉添加到从它继承的每个活动。这里是 [BaseDrawerActivity](https://github.com/frogermcs/InstaMaterial/blob/d14fba84e9114de79ce263c6d68c5fb476ec53f7/app/src/main/java/io/github/froger/instamaterial/ui/activity/BaseDrawerActivity.java) 的源代码。

最后看看这个[提交](https://github.com/frogermcs/InstaMaterial/commit/bdcc2abb89998cfca75487450cc4a9607a71ac41)。这是我们保存了多少代码。定制导航抽屉视图，菜单适配器和布局变得不必要。

## 浮动操作按钮

![|center](http://frogermcs.github.io/images/16/fab.gif)
浮动操作按钮可能是材质设计指南中最引人注目的 UI 元素。 但是直到现在它的实施并不那么明显。 圆形，阴影（也有曲线形状），水波纹效果，高度。 现在一切都由库提供。 我们所要做的就是将 FloatingActionButton 直接放入 .xml 布局文件中：
```xml
<android.support.design.widget.FloatingActionButton
    android:id="@+id/btnCreate"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="bottom|right"
    android:layout_marginBottom="@dimen/btn_fab_margins"
    android:layout_marginRight="@dimen/btn_fab_margins"
    android:src="@drawable/ic_instagram_white"
    app:borderWidth="0dp"
    app:elevation="6dp"
    app:pressedTranslationZ="12dp" />
```
就这样。我们不需要两个不同的 drawables 为 v21 和 pre-21 的系统版本。不需要为圆形阴影等做工作。 除非我们不想以不同寻常的方式定制它，否则一切都是为我们做的。

值得一提的是，FloatingActionButton 的默认实现有两种尺寸 - 迷你和普通（默认）：app:fabSize = “mini” 和 app:fabSize = “normal” 属性

## CoordinatorLayout
这是许多 Android 开发者错过的东西 - 尤其是那些使用交互式布局的人。 CoordinatorLayout 是一个新的 ViewGroup 布局，有助于协调其子视图之间的交互。 在实践中，最常见的用例可能是 ScrollView 和任何其他视图之间的交互。 这也是我们的开始 - 当 Feed 向下滚动时，我们想要隐藏工具栏，并显示它向上滚动。

## AppBarLayout
AppBarLayout 是另一个可以帮助我们的 ViewGroup。 它给了我们一些工具栏的默认行为，可以用来与任何滚动视图进行交互。 下面是示例视图层次结构，显示了如何使用 AppBarLayout 与 CoordinatorLayout 相关联：
```xml
<android.support.design.widget.CoordinatorLayout
    android:id="@+id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/rvFeed"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />

    <android.support.design.widget.AppBarLayout
        android:id="@+id/appBarLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:elevation="@dimen/default_elevation"
            app:layout_scrollFlags="scroll|enterAlways"
            app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

            <ImageView
                android:id="@+id/ivLogo"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="center"
                android:scaleType="center"
                android:src="@drawable/img_toolbar_logo" />
        </android.support.v7.widget.Toolbar>

    </android.support.design.widget.AppBarLayout>

</android.support.design.widget.CoordinatorLayout>
```
这里我们有一些重要的事情：
- 工具栏中的 app:layout_scrollFlags =“scroll | enterAlways” 意味着 AppBarLayout 的这个孩子将对滚动事件作出反应（视图将与滚动事件直接相关），并且视图将滚动到任何向下的滚动事件（更好地称为“快速返回 “模式）。
- 在RecyclerView 中的 app:layout_behavior =“@string / appbar_scrolling_view_behavior” 意味着这个滚动视图将会发送滚动事件给 AppBarLayout 子项。

结果
![|center](http://frogermcs.github.io/images/16/appbarlayout.gif)
很简单，不是吗？现在检查一下你可以用 [scrollFlags](https://developer.android.com/reference/android/support/design/widget/AppBarLayout.LayoutParams.html#setScrollFlags(int)) 的其余部分来做什么。

## Snackbar
![|center](http://frogermcs.github.io/images/16/snackbar.gif)
Snackbar 是作为更成熟的 Toast 实现而推出的。它也用于短消息，但可以提供额外的操作（例如，“撤销”当前操作）。而且它可以与嵌套在 CoordinatorLayout 中的其他视图进行交互。

这个结构和 Toast 一样简单：
```java
public void showLikedSnackbar() {
    Snackbar.make(clContent, "Liked!", Snackbar.LENGTH_SHORT).show();
}
```
make () 方法的第一个参数是从中查找父项的视图。 这意味着 Snackbar 会走上视图树，试图找到合适的家长。 如果是 CoordinatorLayout，Snackbar 将能够与例如 FloatingActionButton 进行交互。 而且通过滑动也会被忽略。

## TabLayout
![|center](http://frogermcs.github.io/images/16/tablayout.gif)
最后，Google 为我们提供了现代的 tabbar 实现方式 - 包括横向滚动，图标或文本，易于标签指示器自定义，制表符重力，涟漪效应等。这是 TabLayout，除此之外别无其他：
```xml

<android.support.design.widget.TabLayout
    android:id="@+id/tlUserProfileTabs"
    android:layout_width="match_parent"
    android:layout_height="48dp"
    android:background="?attr/colorAccent"
    app:tabGravity="fill"
    app:tabIndicatorColor="#5be5ad"
    app:tabIndicatorHeight="4dp"
    app:tabMode="fixed" />
```
```java
private void setupTabs() {
    tlUserProfileTabs.addTab(tlUserProfileTabs.newTab().setIcon(R.drawable.ic_grid_on_white));
    tlUserProfileTabs.addTab(tlUserProfileTabs.newTab().setIcon(R.drawable.ic_list_white));
    tlUserProfileTabs.addTab(tlUserProfileTabs.newTab().setIcon(R.drawable.ic_place_white));
    tlUserProfileTabs.addTab(tlUserProfileTabs.newTab().setIcon(R.drawable.ic_label_white));
}
```
## CollapsingToolbarLayout
最后我们来看看 CollapsingToolbarLayout。 由于工具栏是比旧的 ActionBar 更通用的解决方案，有很多与此视图相关的 UI 效果 - 视差，动态标题大小和位置，展开/折叠内容等等。 简而言之，这就是 CollapsingToolbarLayout 带给我们的。
还有一次是正确的 .xml 布局配置 - 完全没有 Java 代码。

这里是取代我们的 UserProfileAdapter（不相关的行被隐藏）的实现：
```xml
<android.support.design.widget.AppBarLayout
    android:id="@+id/appBarLayout"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <android.support.design.widget.CollapsingToolbarLayout
        android:id="@+id/collapsing_toolbar"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:contentScrim="?attr/colorPrimary"
        app:layout_scrollFlags="scroll|exitUntilCollapsed">

        <LinearLayout
            android:id="@+id/vUserProfileRoot"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="?attr/colorPrimary"
            app:layout_collapseMode="parallax">

        </LinearLayout>

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:layout_collapseMode="pin"
            app:layout_scrollFlags="scroll|enterAlways"
            app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">
        </android.support.v7.widget.Toolbar>

    </android.support.design.widget.CollapsingToolbarLayout>

    <android.support.design.widget.TabLayout
        android:id="@+id/tlUserProfileTabs"
        android:layout_width="match_parent"
        android:layout_height="48dp"
        android:background="?attr/colorAccent"
        app:tabGravity="fill"
        app:tabMode="fixed" />

</android.support.design.widget.AppBarLayout>
```
activity_user_profile.xml 的完整源代码在[这里](https://github.com/frogermcs/InstaMaterial/blob/d14fba84e9114de79ce263c6d68c5fb476ec53f7/app/src/main/res/layout/activity_user_profile.xml)可用。

**重要的事？**
- app:layout_scrollFlags = “scroll | exitUntilCollapsed” 意味着没有固定的孩子会被折叠，直到我们滚动到顶部。固定视图（我们的工具栏与应用程序：layout_collapseMode = “pin”）将保持不变。
- app:layout_collapseMode = “parallax” 导致我们的视图将以视差方式折叠。 app:contentScrim = “？attr / colorPrimary” 在 CollapsingToolbarLayout 意味着折叠的视图将被这个特定的颜色覆盖。

现在看看最后的效果：
![|center](http://frogermcs.github.io/images/16/collapsing_toolbar2.gif)

而这一切都是为了今天。 我们刚刚更新了 InstaMaterial 的新视图和效果，如 Snackbar，FloatingActionButton，TabLayout，CoordinatorLayout，AppBarLayout 和 CollapsingToolbarLayout。 所有这些都是从 [Android 设计支持库](http://android-developers.blogspot.com/2015/05/android-design-support-library.html)提供的。

## 示例代码
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。

