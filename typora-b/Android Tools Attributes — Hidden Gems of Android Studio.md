# Android Tools 属性ーAndroid Studio 隐藏的宝石

> 原文 (Medium)：[Android Tools Attributes — Hidden Gems of Android Studio](https://android.jlelse.eu/tools-attributes-hidden-gems-of-android-studio-d7451b194e0b)
>
> 作者：[Orkhan Gasimli](https://android.jlelse.eu/@orkhan.gasimli?source=post_header_lockup)

[TOC]

也许你们中的大多数人已经看到甚至使用奇怪的 `tools:` 前缀的 XML 属性，在 Android Studio 中设计应用布局时。就我个人而言，我在使用 `tools:text` 属性在一个运行时应该动态添加的地方 ，当然使用 `android：text` 属性是不可行的。

然而，我与这个强大的工具的关系并没有超越  `tools:text` , `tools:visibility` 和 `tools:context` 属性直到几个月前，我在设计个人应用项目的布局时遇到了一个问题。RecyclerView下方的视图块在布局预览中不可见，因为 RecyclerView 覆盖了布局屏幕。在搜索解决方案时，我发现自己正在阅读 Android Developers 网站上发布的[工具属性的官方用户指南](https://developer.android.com/studio/write/tool-attributes.html)。在阅读了指南之后，我意识到，我低估了这个工具很长一段时间，并且发现这个工具可以帮助我在 Android Studio 的预览窗格中直接看到我的布局。 

你可能已经明白了，今天我想告诉你关于工具属性的事情。 因此，让我们开始更深入的探索，更好地理解这些属性是如何工作的，以及它们如何简化我们的生活。 

## 总体概览

首先要做的是！ 在讨论细节之前，我想简要介绍一下工具属性，并解释如何使用它们。 

一般来说，工具属性是特殊的 XML 属性，它不仅允许设计时特性(比如应该在片段中绘制哪个布局)以及编译时行为(如 lint 警告)。 

> 根据官方用户指南，这些属性仅由 Android Studio 使用，并通过在应用程序生成时构建工具来移除。因此，使用这些属性对 APK 的大小或运行时行为没有影响 

为了使用这些属性，应该在 XML 文件的根元素中添加 tools 命名空间，如下所示: 

```xml
<RootTag xmlns:android=”http://schemas.android.com/apk/res/android"
 xmlns:tools=”http://schemas.android.com/tools" >
```

基于属性完成的功能，我们可以将工具属性分为3类：

1. [错误处理属性](https://developer.android.com/studio/write/tool-attributes.html#error_handling_attributes) - 用于取缔 lint 警告消息。该类别包含以下属性：

- tools:ignore  - 用于任何元素取缔 lint 警告。它应该提供一个逗号分隔的 lint 问题 ID 列表，这些工具应该忽略这个元素或其任何编辑 。例如，你可以强制 Android Studio 忽略 \<ImageView> 缺少的内容描述：

```xml
<ImageView
    android:layout_width="@dimen/error_image_width"
    android:layout_height="@dimen/error_image_height"
    android:src="@drawable/ic_person_off" 
    tools:ignore="ContentDescription"/>
```

- tools:targetApi  - 用于指示支持该元素的任何元素的 API 级别(或者作为整数或代码名称)来支持这个元素。该属性的工作原理与 Java 代码中的 @TargetApi 注解相同。例如，你可能会使用这个，因为 android：elevation 只能在 API 级别21以上，而你知道这个布局不适用于任何 API 版本: 

```xml
<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:elevation="4dp"
    tools:targetApi="lollipop"/>
```

- tools:locale - 用于资源并用于指示给定 \<resources> 元素的默认语言/区域，以避免拼写检查器发出警告。该值必须是有效的 [LocaleQualifier ](https://developer.android.com/guide/topics/resources/providing-resources.html#LocaleQualifier)。在示例中，你可以将其添加到 values / strings.xml 文件（默认字符串值），以指示用于默认字符串的语言是西班牙语而不是英语：

```xml
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:locale="es">
```

2. [资源收缩属性](https://developer.android.com/studio/write/tool-attributes.html#resource_shrinking_attributes) - 用于在使用[资源收缩](https://developer.android.com/studio/build/shrink-code.html#shrink-resources)时进行严格的引用检查。 这个类别包括以下属性 : 

- tools:shrinkMode  - 用于表示收缩模式。 默认的收缩模式是安全模式。要改为使用严格模式，应该在 \<resources> 标记中添加 shrinkMode =“strict”，如下所示：

```xml
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="strict" />
```

- tools:keep  - 用于手动指定在资源收缩期间应该保留的资源（通常是因为它们在运行时以间接方式引用）。要手动保留资源，必须使用 \<resources> 标记在资源目录（即 res / raw / keep.xml 中）中创建一个 XML 文件，在 tools:keep 属性指定逗号分隔名单以保留每个资源。你也可以使用星号作为通配符。例如：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/used_1,@layout/used_2,@layout/_3" />
```

- tools:discard  - 用于手动指定在收缩资源时应剥离的资源（通常是因为资源被引用但不影响你的应用程序，或者因为 Gradle 插件错误地推断资源被引用）。此属性的用法与工具的 keep 属性用法相同：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:discard="@layout/unused_1" />
```

3. [设计时视图属性](https://developer.android.com/studio/write/tool-attributes.html#design-time_view_attributes) - 用于定义仅在 Android Studio 预览窗格中可见的布局特性。 换句话说，这些属性允许我们改变 Android Studio 布局的渲染，而不影响它们的构建。 

## 设计时视图属性

这是我想在这个故事中集中注意的主要工具属性类别。 除了解释每个设计时属性之外，我还会尝试在一个样例联系应用程序(我称之为联系人 + 应用程序)为基础，给出一些现实中的使用例子。 那么，我们开始吧。 

### tools: 代替 android:

使用 tools: 前缀替换 android: 前缀在任何 view 属性上的前缀将允许你在布局预览中插入示例数据，同时只为布局预览设置一个属性。你还记得 tools:text 和 tools:visibility 我在这个故事的开头提到？这些属性属于这个类别，当属性的值在运行时之前没有填充时是有用的，但是它需要在布局预览中提前显示效果。 

让我们在联系人 item.xml 布局中尝试这些属性，它定义了我们联系人 + 应用程序中每个联系人项目的布局。 下面你可以看到这个布局的预览和当前的 XML 代码。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovp9769zj30ac04n74e.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/contact_card_view"
    android:layout_gravity="center"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardCornerRadius="9dp"
    android:clickable="true"
    android:focusable="true"
    app:cardElevation="3dp"
    android:padding="16dp"
    app:cardUseCompatPadding="true"
    android:foreground="?attr/selectableItemBackground">

    <android.support.constraint.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="8dp">

        <ImageView
            android:id="@+id/avatar_image"
            android:layout_width="38dp"
            android:layout_height="38dp"
            android:contentDescription="User avatar"
            android:scaleType="centerCrop"
            tools:src="@drawable/ic_face_male"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"
            app:layout_constraintLeft_toLeftOf="parent"
            tools:ignore="HardcodedText"/>

        <TextView
            android:id="@+id/contact_name_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Subhead"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"
            tools:text="John Doe"
            app:layout_constraintTop_toTopOf="parent"/>

        <TextView
            android:id="@+id/contact_mobile_number_static_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            android:text="@string/mobile_number_static_text"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_name_textview"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_mobile_number_textview"
            android:layout_width="wrap_content"
            android:layout_height="16dp"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            tools:text="(800) 555-7150"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_name_textview"
            app:layout_constraintLeft_toRightOf="@+id/contact_mobile_number_static_textview"
            android:layout_marginLeft="@dimen/contact_item_static_textview_margin"
            android:layout_marginStart="@dimen/contact_item_static_textview_margin"/>

        <TextView
            android:id="@+id/contact_home_number_static_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            android:text="@string/home_number_static_text"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_mobile_number_static_textview"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_home_number_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            tools:text="(800) 555-7150"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_mobile_number_textview"
            app:layout_constraintLeft_toRightOf="@+id/contact_home_number_static_textview"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_office_number_static_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            android:text="@string/office_number_static_text"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_home_number_textview"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_office_number_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            tools:text="(800) 555-7150"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_home_number_textview"
            app:layout_constraintLeft_toRightOf="@+id/contact_office_number_static_textview"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_email_static_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            android:text="@string/email_static_text"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_office_number_static_textview"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_email_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            tools:text="john.doe@mail.com"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_office_number_textview"
            app:layout_constraintLeft_toRightOf="@+id/contact_email_static_textview"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_address_static_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            android:text="@string/address_static_text"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_email_static_textview"
            app:layout_constraintLeft_toRightOf="@+id/avatar_image"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <TextView
            android:id="@+id/contact_address_textview"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textAppearance="@style/TextAppearance.AppCompat.Body1"
            android:textColor="@color/colorSecondaryText"
            tools:text="San-Fransisco, US"
            android:layout_marginTop="8dp"
            app:layout_constraintTop_toBottomOf="@+id/contact_email_textview"
            app:layout_constraintLeft_toRightOf="@+id/contact_address_static_textview"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <android.support.v7.widget.AppCompatImageButton
            android:id="@+id/favourite_button"
            android:layout_width="38dp"
            android:layout_height="38dp"
            android:src="@drawable/ic_star_empty_green"
            android:background="?selectableItemBackground"
            android:layout_marginRight="4dp"
            android:layout_marginEnd="4dp"
            app:layout_constraintRight_toLeftOf="@+id/divider_line"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginLeft="8dp"
            android:layout_marginStart="8dp"/>

        <View
            android:id="@+id/divider_line"
            android:layout_height="38dp"
            android:layout_width="1dp"
            android:background="@color/colorSecondaryText"
            android:layout_marginStart="4dp"
            android:layout_marginLeft="4dp"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintRight_toLeftOf="@+id/call_button"
            android:layout_marginRight="@dimen/contact_item_divider_margin"
            android:layout_marginEnd="@dimen/contact_item_divider_margin"/>

        <android.support.v7.widget.AppCompatImageButton
            android:id="@+id/call_button"
            android:layout_width="38dp"
            android:layout_height="38dp"
            android:src="@drawable/ic_phone"
            android:background="?selectableItemBackground"
            android:layout_marginRight="8dp"
            android:layout_marginEnd="8dp"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            android:layout_marginStart="4dp"
            android:layout_marginLeft="4dp"/>

    </android.support.constraint.ConstraintLayout>

</android.support.v7.widget.CardView>
```

请注意 tools: textview 元素和工具中的文本属性: imageview 元素中使用的 tools: src 属性。 我们使用这些属性是因为所有数据将在运行时从数据库或 API 中检索，并显示在相关的视图元素中。 如果没有工具属性，我们的卡片项目将会是这样的: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovphzv1lj30av04tglh.jpg)

相当混乱。对吧？因此，使用 tools: 前缀属性使我们能够在布局预览中可视化数据，并更精确地测试我们的布局。

我们也可以使用 tools: 前缀属性来设置不能用于布局预览的 android：前缀属性。例如，假设我们希望我们的联系人项目显示只显示联系人的名字和移动号码，并通过扩展用户点击来显示其他二级数据。 为了实现这个功能，我只需将 cardview 的高度设置为80 dp，然后添加 onClick 监听器来扩展它并显示次要信息。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovpox94ij30av025wed.jpg)

正如你所看到的，设置高度为80 dp 后，我们不再能够在布局预览中看到次要的字段。 然而，解决这个问题很容易。 我们只需要在 cardview 中添加 tools:layout_height=”wrap_content”  。 这也意味着它可以同时使用 android: 命名空间属性(在运行时使用)和匹配 tools: 属性(只在布局预览中重写运行时属性)同时使用同一视图元素。 

### tools:context

这个属性声明了默认情况下此布局与哪些活动相关联。 这使编辑器或者布局预览中的功能需要对活动有所了解，比如为布局选择正确的 theme，渲染 action bar (与活动相关联) ，一个可以添加 onClick 处理程序的地方等。 

在我们的应用程序的例子中，我们将把这个属性添加到我们的联系人片段的根标签中，以便告知这个片段将被添加到主活动中。 

```xml
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.activity.MainActivity">
```

### tools:layout

这个属性只能通过  \<fragment> 标记使用，并通知编辑应该在片段内部通过布局预览来绘制。下面你可以看到我们活动 main.xml 布局在添加工具之前和之后的区别:  添加 tools：layout =“@ layout / fragment_contacts” 到 \<fragment> 标记。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovq2brumj30lj0ihmyh.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.activity.MainActivity">

    <android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            app:layout_scrollFlags="scroll|enterAlways"
            app:popupTheme="@style/ThemeOverlay.AppCompat.Light">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Contacts+"
                android:textStyle="bold"
                android:textSize="18sp"/>

        </android.support.v7.widget.Toolbar>

    </android.support.design.widget.AppBarLayout>

    <fragment
        android:id="@+id/home_fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"
        tools:layout="@layout/fragment_contacts"/>

</android.support.design.widget.CoordinatorLayout>
```

### tools:showIn

此属性用于指向一个使用此布局作为 include 的布局，并通过 include 标签引用它。 这允许你预览和编辑该文件，因为它出现的同时嵌入其父布局。 例如，在我们的 contact_item.xml 的根标签中添加 tools:showIn=”@layout/activity_main” 。 xml 将迫使编辑在主要活动中重新绘制我们的布局: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovqp8numj30lj0ihmyh.jpg)

### tools:listitem | tools:listheader | tools:listfooter

这些属性用于 \<AdapterView>（及其子类，如 \<ListView> 和 \<RecyclerView>），用于指定应该在该适配器内作为列表项，页眉或页脚绘制的布局。例如，我们的联系人 + 应用程序的 fragment_contacts_xml 布局声明 \<RecyclerView>，这是在添加 tools：listitem =“@ layout / contact_item” 之前和之后的样子：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovqwm20uj30lj0ihmyh.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/contact_fragment_relative_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.activity.MainActivity"
    tools:showIn="@layout/activity_main">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerview_contacts"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:visibility="visible"
        app:layoutManager="LinearLayoutManager"
        tools:listitem="@layout/contact_item"/>

</RelativeLayout>
```

### tools:itemCount

该属性仅用于 \<RecyclerView>，用于指定布局编辑器在布局预览中应该呈现的列表项数。 

根据我的观察，默认 Android Studio 为 \<RecyclerView> 显示10个列表项。因此，通常在添加 tools : listitem 属性之后， \<RecyclerView> 将覆盖整个布局屏幕，你不能再看到下面的其他视图元素。在这种情况下，tools ：itemCount 属性将帮助你查看 \<RecyclerView> 下面的元素。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovsmrd80j30aw0jeq37.jpg)

### tools:menu

该属性用于指定应该在应用栏中显示的菜单布局。菜单布局应添加 @menu / 或任何其他 ID 前缀且不带 .xml 扩展名。

在我们的 activity_main.xml 的根标签中添加 tools：menu =“main” 后，我们将开始在应用栏中看到菜单图标：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovsr0wsij30m80283yh.jpg)

根据官方文件，还可以传入多个菜单 ID，以逗号分隔。然而，在布局预览中传递多个菜单项不会有任何效果。 

### tools:openDrawer

该属性仅用于 \<DrawerLayout>，并允许在布局编辑器的预览窗格中控制其状态（打开，关闭）和位置（左侧，右侧）。下表列出了这个属性接受为参数的常量的名称和描述: 

| Constant | Value  | Description                                      |      |
| -------- | ------ | ------------------------------------------------ | ---- |
| end    | 800005 | Push object to the end of its container.       |      |
| left   | 3      | Push object to the left of its container.      |      |
| right  | 5      | Push object to the right of its container.     |      |
| start  | 800003 | Push object to the beginning of its container. |      |

例如，下面的代码将使你能够在预览窗格中看到处于打开状态的 \<DrawerLayout>：

```xml
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:openDrawer="start" />
```

### tools:minValue | tools:maxValue

此属性为布局编辑器预览窗格中的 \<NumberPicker> 设置最小值和最大值。

```xml
<NumberPicker 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/numberPicker"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    tools:minValue="0"
    tools:maxValue="10" />
```

### "@tools:sample/" 资源

这是允许我们在视图中注入占位符数据或图像的最有用的属性之一。 目前，Android Studio  提供了以下类型的预定义数据，可以被注入到我们的视图元素中: 

| Attribute value                  | Description of placeholder data                              |      |
| -------------------------------- | ------------------------------------------------------------ | ---- |
| @tools:sample/full_names         | Full names that are randomly generated from the combination of @tools:sample/first_names and @tools:sample/last_names. |      |
| @tools:sample/first_names        | Common first names.                                          |      |
| @tools:sample/last_names         | Common last names.                                           |      |
| @tools:sample/cities             | Names of cities from across the world.                       |      |
| @tools:sample/us_zipcodes        | Randomly generated US zipcodes.                              |      |
| @tools:sample/us_phones          | Randomly generated phone numbers with the following format:(800) 555-xxxx. |      |
| @tools:sample/lorem              | Placeholder text that is derived from Latin.                 |      |
| @tools:sample/date/day_of_week   | Randomized dates and times for the specified format.         |      |
| @tools:sample/date/ddmmyy        |                                                              |      |
| @tools:sample/date/mmddyy        |                                                              |      |
| @tools:sample/date/hhmm          |                                                              |      |
| @tools:sample/date/hhmmss        |                                                              |      |
| @tools:sample/avatars            | Vector drawables that you can use as profile avatars.        |      |
| @tools:sample/backgrounds/scenic | Images that you can use as backgrounds.                      |      |

在联系人 + 应用程序的情况下，这些预定义的数据将允许我们在不使用硬编码文本和图像资源的情况下，可视化联系人的名称、姓氏、电话号码、地址甚至是联系人的形象。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovsygysjj30m80jbdiu.jpg)

漂亮整洁，是吧？！ 👍

我们的布局现在看起来更真实。但是，由于没有预定义的电子邮件地址数据，我们仍然使用硬编码的文本来显示它。另外，我们只是使用仅显示城市名称的 “@tools：sample / cities”，而不是完整的地址。好消息是，Android Studio 3 现在使我们能够创建自己的预定义示例数据。

### Sample Data

为了声明和使用我们的样本数据，首先我们需要创建一个样本数据文件夹。为此，我们必须右键单击 app 目录 New -> Sample Data directory。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovt2njmgj30m809341r.jpg)

之后，你会注意到应用程序下名为 sampledata 的新文件夹。现在我们可以将数据放入该文件夹中。那时我们有两个选择：

- 添加纯文本文件，逐行插入原始数据，然后使用 “@ sample / fileName” 引用该数据。在我们的应用程序中，我们将创建两个不同的文件（电子邮件和地址），并在该文件中插入电子邮件和地址数据。然后，我们将使用 tools：text =“@ sample / emails” 和 tools：text =“@ sample / addresses” 引用 contact_item.xml 中的数据。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovt5km3lj30m807utaa.jpg)

- 创建一个文件，其中包含我们需要的所有 JSON 格式的数据，并使用 “@ sample / fileName.json / arrayName / fieldName” 引用该数据。如果你有复杂的数据，则应优先选择此选项。对于我们的 Contacts+应用，我们可以创建一个名为 sample.json 的文件，其内容如下，并通过引用字段 tools：text =“@ sample / sample.json / data / email” 和 tools：text =“@ sample / sample.json / data /address”。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovt8x39bj30m806u0uh.jpg)

## 祝贺你

我们已经做了一个非常令人印象深刻的工作，在所有的变化之后，我们的应用看起来非常干净和真实。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovtbq68lj30av0jejs0.jpg)

希望你觉得文章有用有趣。请随意分享你的反馈和评论，不要忘记鼓掌。 

Happy coding!

