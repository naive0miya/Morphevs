# InstaMaterial 概念(第1部分) - 开始

>原文 (mirekstanek.online) ： [Instagram with Material Design concept is getting real](https://mirekstanek.online/instagram-with-material-design-concept-is-getting-real/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

几个月前，在谷歌推出了移动设计和网页设计的新设计指南之后，设计师 Emmanuel Pacamalan 制作了一个概念视频，展示 Instagram 对 Android 的定义：

<iframe width="560" height="315" src="https://youtu.be/ojwdmgmdR_Q" frameborder="0" allowfullscreen></iframe>

虽然这只是一个图形原型，一些人开始怀疑是否有可能以相对简单的方式在实际应用中实现这一点。 嗯，是的。 不仅仅是在最新的安卓操作系统 Lillipop 上。 事实上，自从 Android 4发布以来，我们可以实现大部分显示的图形和动画效果。

根据这个，我想开始一系列新的帖子，在这些文章中，我将向你展示如何在真正的 Android 设备上用 [Material Design 重现 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q)。当然，由于显而易见的原因，我们将为这个应用程序创建前端模型，但是我们会尽可能地复制尽可能多的细节。

## 开始项目
在这篇文章中，我想重现展示概念的前7秒。 我认为这是第一次，关于我们也必须准备和配置我们的项目。

我希望你们记住，所有的解决方案都不是唯一可行的方法，它们是我最喜欢的。 我也不是一个平面设计师，所以我们项目中使用的所有图片都是从公共资源（主要来自 Material Design [资源页面](http://www.google.com/design/spec/resources/sticker-sheets-icons.html)上的 ）。

好的，这里有截图和两个短片(在 android 4和5上拍摄)显示了我们的最终效果:

![|center](http://frogermcs.github.io/images/2/screen-21.png)
![|center](http://frogermcs.github.io/images/2/screen-pre21.png)

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>

## 准备应用程序
在我们的项目中，我们将使用一些用于 Android 开发的流行工具和库。不是所有的文件我们都会在这篇文章中使用，但是我想尽可能避免在不久的将来处理构建文件。

#### 初始项目

首先，我们必须创建一个新的 Android 项目。 为此，我使用了 Android Studio 和 gradle 构建系统。 最小版本的 Android SDK 为15(Android 4.0.4)。

之后，我将添加一些依赖和 utils。没有什么特别的要说的，所以在这里你有完整的 build.gradle 和 app / build.gradle 文件的源代码：
```groovy
//  build.gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:0.14.0'
        classpath 'com.jakewharton.hugo:hugo-plugin:1.1.+'
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```

```groovy
//  app/build.gradle
apply plugin: 'com.android.application'
apply plugin: 'hugo'

android {
    compileSdkVersion 21
    buildToolsVersion "21.1"

    defaultConfig {
        applicationId "io.github.froger.instamaterial"
        minSdkVersion 15
        targetSdkVersion 21
        versionCode 1
        versionName "1.0"
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])

    compile "com.android.support:appcompat-v7:21.0.0"
    compile 'com.android.support:support-v13:21.+'
    compile 'com.android.support:support-v4:21.+'
    compile 'com.android.support:palette-v7:+'
    compile 'com.android.support:recyclerview-v7:+'
    compile 'com.android.support:cardview-v7:21.0.+'
    compile 'com.jakewharton:butterknife:5.1.2'
    compile 'com.jakewharton.timber:timber:2.5.0'
    compile 'com.facebook.rebound:rebound:0.3.6'
}
```
总之，我们在这里有：
- 一堆支持库（CardView，RecyclerView，Palette，AppCompat） - 我想使用最新的用户界面组件。当然，如果我们想要使用 ListView，ActionBar 或简单的 View / FrameView，那不是问题。但没必要？ 😉
- [ButterKnife ](http://jakewharton.github.io/butterknife/)- 视图注入，为更简洁的代码（即我们不必写 findViewById () ) 来获取视图引用，但它也提供更强大的方法，如 onCli () 注入等。
- [Rebound ](http://facebook.github.io/rebound/)- 我们不会在目前的文章中使用这个库，但我相信我们会在不久的将来做到这一点。 这是来自 Facebook 的强大的库，可以帮助你使动画更加自然（它提供了弹性的动画工具 - 只需从 Facebook Messenger 中检查聊天头的动画是如何工作的，以确保你想要在你的下一个项目中使用它）
- [Timer](https://github.com/JakeWharton/timber) 和 [Hugo](https://github.com/JakeWharton/hugo) - 在这个项目中没有必要。我只用它们来记录日志。

## 图像资源
在这个项目中，我使用了 Material Design 资源中的一些图标。 我还在 Instagram 上做了 GoPro 个人资料的截图，以提供我们的 Feed 的模型。 应用程序图标是从 [INSTAGRAM 与材料设计](https://www.youtube.com/watch?v=ojwdmgmdR_Q)视频中选取的。 这里是项目中使用的[一大堆图像](https://github.com/frogermcs/frogermcs.github.io/raw/master/files/2/resources.zip)（现在是整个 drawable-xxdpi 目录）。

#### 风格
让我们从定义我们的应用程序的默认样式开始。 为 Android 4和5提供 Material Desing 样式基础的最简单方法是从 Theme.AppCompat.NoActionBar 或 Theme.AppCompat.Light.NoActionBar 继承。 为什么 NoActionBar？新的 SDK（和兼容库）为我们提供了一种实现 ActionBar 模式的新方法。在我们的例子中，我们将使用 Toolbar 小部件，原因很简单 - 与 ActionBar 相比，它是更好更灵活的解决方案。我不会深入细节。 相反，您可以在Android 开发者博客上阅读关于 [AppCompat v21](http://android-developers.blogspot.com/2014/10/appcompat-v21-material-design-for-pre.html) 的最新帖子。

基于概念电影，我们将为我们的 AppTheme 定义三种基本颜色：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- styles.xml-->
<resources>
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="colorPrimary">@color/style_color_primary</item>
        <item name="colorPrimaryDark">@color/style_color_primary_dark</item>
        <item name="colorAccent">@color/style_color_accent</item>
    </style>
</resources>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--colors.xml-->
<resources>
    <color name="style_color_primary">#2d5d82</color>
    <color name="style_color_primary_dark">#21425d</color>
    <color name="style_color_accent">#01bcd5</color>
</resources>
```

有关这些颜色的更多细节，请参阅[“材质主题调色板”文档](http://developer.android.com/training/material/theme.html#ColorPalette)。

## 布局
我们目前的项目将由3个主要布局元素构建:

- Toolbar - 带有导航图标和应用标志的工具栏顶栏
- RecyclerView - 用于显示 Feed
- Floating Action Button - 简单的 ImageButton，实现了 Material Design 中引入的新[动作按钮模式](http://www.google.com/design/spec/components/buttons.html#buttons-flat-raised-buttons)。

在我们开始实现我们的布局之前，让我们在 res / values / dimens.xml 中定义一些默认值：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--dimens.xml-->
<resources>
    <dimen name="btn_fab_size">56dp</dimen>
    <dimen name="btn_fab_margins">16dp</dimen>
    <dimen name="default_elevation">8dp</dimen>
</resources>
```
这些尺寸基于 Material Design 指南引入的值。

很好，现在我们可以实现 MainActivity 的布局了:

关于以上代码的一些解释：
- Toolbar 最棒的地方在于现在它是活动布局实现的一部分，它也扩展了 ViewGroup，所以我们可以在里面放置一些 UI 元素（他们将使用剩余的自由空间）。在我们的例子中，我们使用它来放置 logo 图像。 此外，虽然 Toolbar 是比 ActionBar 更灵活的部件，但现在我们必须手动做更多的事情。因此，我们必须为我们的背景设置 colorPrimary 颜色（否则工具栏将是透明的）。
- RecyclerView 在 XML 文件中实现起来非常简单，直到在 java 代码中正确配置它，我们的应用程序才不会抛出 java.lang.NullPointerException（它与未配置的 LayoutAdapter 连接，该适配器可以重新安装 recyclerview 中的项）。
- 高度 不适用于 Android 的 pre-21 设备。所以如果我们想要显示我们的 Floating Action Button 真的浮动，我们必须为 v21 和 pre-21 的设备实现不同的背景。

#### 浮动操作按钮

为了简化我们在 FAB 上的工作，我们将为 v21 和 pre-21 的设备使用一些不同的样式：

-  v21 的样式
  ![|center](http://frogermcs.github.io/images/2/fab-21.png)
- pre-21 的样式
  ![|center](http://frogermcs.github.io/images/2/fab-pre21.png)

为了实现这一点，我们必须为按钮的背景创建两个不同的 XML 文件：用于 v21 的位于 /res/drawable-v21/btn_fab_default.xml 和用于 pre-21 的位于 /res/drawable/btn_fab_default.xml：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable-v21/btn_fab_default.xml-->
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/fab_color_shadow">
    <item>
        <shape android:shape="oval">
            <solid android:color="@color/style_color_accent" />
        </shape>
    </item>
</ripple>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/btn_fab_default.xml-->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="false">
        <layer-list>
            <item android:bottom="0dp" android:left="2dp" android:right="2dp" android:top="2dp">
                <shape android:shape="oval">
                    <solid android:color="@color/fab_color_shadow" />
                </shape>
            </item>

            <item android:bottom="2dp" android:left="2dp" android:right="2dp" android:top="2dp">
                <shape android:shape="oval">
                    <solid android:color="@color/style_color_accent" />
                </shape>
            </item>
        </layer-list>
    </item>
    <item android:state_pressed="true">
        <shape android:bottom="2dp" android:left="2dp" android:right="2dp" android:shape="oval" android:top="2dp">
            <solid android:color="@color/fab_color_pressed" />
        </shape>
    </item>
</selector>
```
此外，我们必须在 res / values / colors.xml 文件中添加两种颜色：
```xml
<color name="btn_default_light_normal">#00000000</color>
<color name="btn_default_light_pressed">#40ffffff</color>
```
正如你所看到的，为 pre-21 的设备制作阴影效果有点棘手。不幸的是，没有简单的方法通过 XML 文件来实现“真实”阴影。 其他选项是使用准备好的较早的阴影图像，或直接在 Java 代码中使用 Outline 类（提示[创建 fab 阴影](http://stackoverflow.com/questions/24480425/android-l-fab-button-shadow)）来完成。

#### 工具栏
现在我们来完成我们的工具栏。 我们有背景和应用标志，所以现在只有导航和菜单图标丢失。

不幸的是，XML 值 app:navigationIcon =“ ” 不工作，而 Android:navigationIcon =“ ” 只适用于 v21。这就是为什么我们必须从 Java 代码设置导航图标：
```java
toolbar.setNavigationIcon(R.drawable.ic_menu_white);
```
对于菜单图标，我们可以使用通过 res / menu / menu_main.xml 定义的标准选项菜单：
```xml
<!--menu_main.xml-->
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".MainActivity">
    <item
        android:id="@+id/action_inbox"
        android:icon="@drawable/ic_inbox_white"
        android:title="Inbox"
        app:showAsAction="always" />
</menu>
```
并从活动源代码中附加：

```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu_main, menu);
    return true;
}
```
现在，一切都应该工作，但正如我在 twitter 上所提到的，在单击导航按钮的选择器中存在一些不一致： 

> 你有没有注意到，单击导航按钮的选择器与 [# materialdesign](https://twitter.com/hashtag/materialdesign?src=hash&ref_src=twsrc%5Etfw) 中的菜单大小不同吗？[#androiddev](https://twitter.com/hashtag/androiddev?src=hash&ref_src=twsrc%5Etfw) [pic.twitter.com/SEuNMx16a1](http://t.co/SEuNMx16a1)
>
> — Mirek Stanek (@froger_mcs) November 8, 2014 [Twitter](https://twitter.com/froger_mcs/status/530995707080359936/photo/1?ref_src=twsrc%5Etfw&ref_url=http%3A%2F%2Ftwitframe.com%2Fshow%3Furl%3Dhttps%3A%2F%2Ftwitter.com%2Ffroger_mcs%2Fstatus%2F530995707080359936)

为了解决这个问题，我们必须做更多的工作。首先让我们为我们的菜单项（res / layout / menu_item_view.xml）创建自定义视图：
```java
<?xml version="1.0" encoding="utf-8"?>
<!--menu_item_view.xml-->
<ImageButton xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="?attr/actionBarSize"
    android:layout_height="?attr/actionBarSize"
    android:background="@drawable/btn_default_light"
    android:src="@drawable/ic_inbox_white" />
```

现在我们必须创建 onClick 选择器，用于 v21（带有水波纹效果）和 pre-21 ：
```java
<?xml version="1.0" encoding="utf-8"?>
<!--drawable-v21/btn_default_light.xml-->
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/btn_default_light_pressed" />
```

```java
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/btn_default_light.xml-->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@color/btn_default_light_normal" android:state_focused="false" android:state_pressed="false" />
    <item android:drawable="@color/btn_default_light_pressed" android:state_pressed="true" />
    <item android:drawable="@color/btn_default_light_pressed" android:state_focused="true" />
</selector>
```
我们项目中的所有颜色列表应该如下所示：
```java
<?xml version="1.0" encoding="utf-8"?>
<!--colors.xml-->
<resources>
    <color name="style_color_primary">#2d5d82</color>
    <color name="style_color_primary_dark">#21425d</color>
    <color name="style_color_accent">#01bcd5</color>

    <color name="fab_color_pressed">#007787</color>
    <color name="fab_color_shadow">#44000000</color>

    <color name="btn_default_light_normal">#00000000</color>
    <color name="btn_default_light_pressed">#40ffffff</color>
</resources>
```
我们应该做的最后一件事就是将我们的自定义视图放到菜单项中。我们可以在 onCreateOptionsMenu () 方法中做到这一点：
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu_main, menu);
    inboxMenuItem = menu.findItem(R.id.action_inbox);
    inboxMenuItem.setActionView(R.layout.menu_item_view);
    return true;
}
```
就这样。我们的工具栏已经准备好了。 菜单项的 onClick 选择器运行正常:
![|center](http://frogermcs.github.io/images/2/toolbar.png)

#### 项目
我们应该实现的最后一件事是基于 RecyclerView 的 Feed 项目。现在我们必须设置两件事：布局管理器（RecyclerView 必须知道如何安排项目）和适配器（提供项目）。

第一件事是简单的 - 虽然我们的布局是简单的 ListView，我们可以使用 LinearLayoutManager 来进行项目的排列。对于第二件，我们必须做更多的工作，但是没有什么魔法可以处理。

我们从定义列表项目布局（res / layout / item_feed.xml）开始：
```java
<?xml version="1.0" encoding="utf-8"?><!-- item_feed.xml -->
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:card_view="http://schemas.android.com/apk/res-auto"
    android:id="@+id/card_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    android:layout_margin="8dp"
    card_view:cardCornerRadius="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <ImageView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:src="@drawable/ic_feed_top" />

        <io.github.froger.instamaterial.SquaredImageView
            android:id="@+id/ivFeedCenter"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <ImageView
            android:id="@+id/ivFeedBottom"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </LinearLayout>
</android.support.v7.widget.CardView>
```
我们看到的是:

- CardView - 包装我们的列表项目的圆角和阴影轮廓（适用于 v21 和 pre-21 版本）。
- 用于 Feed 元素模型的 ImageViews（SquaredImageView 是我们的 ImageView 的实现，它具有方形尺寸）。

FeedAdapter 也非常简单：
```java
public class FeedAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private static final int ANIMATED_ITEMS_COUNT = 2;

    private Context context;
    private int lastAnimatedPosition = -1;
    private int itemsCount = 0;

    public FeedAdapter(Context context) {
        this.context = context;
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.item_feed, parent, false);
        return new CellFeedViewHolder(view);
    }

    private void runEnterAnimation(View view, int position) {
        if (position >= ANIMATED_ITEMS_COUNT - 1) {
            return;
        }

        if (position > lastAnimatedPosition) {
            lastAnimatedPosition = position;
            view.setTranslationY(Utils.getScreenHeight(context));
            view.animate()
                    .translationY(0)
                    .setInterpolator(new DecelerateInterpolator(3.f))
                    .setDuration(700)
                    .start();
        }
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int position) {
        runEnterAnimation(viewHolder.itemView, position);
        CellFeedViewHolder holder = (CellFeedViewHolder) viewHolder;
        if (position % 2 == 0) {
            holder.ivFeedCenter.setImageResource(R.drawable.img_feed_center_1);
            holder.ivFeedBottom.setImageResource(R.drawable.img_feed_bottom_1);
        } else {
            holder.ivFeedCenter.setImageResource(R.drawable.img_feed_center_2);
            holder.ivFeedBottom.setImageResource(R.drawable.img_feed_bottom_2);
        }
    }

    @Override
    public int getItemCount() {
        return itemsCount;
    }

    public static class CellFeedViewHolder extends RecyclerView.ViewHolder {
        @InjectView(R.id.ivFeedCenter)
        SquaredImageView ivFeedCenter;
        @InjectView(R.id.ivFeedBottom)
        ImageView ivFeedBottom;

        public CellFeedViewHolder(View view) {
            super(view);
            ButterKnife.inject(this, view);
        }
    }

    public void updateItems() {
        itemsCount = 10;
        notifyDataSetChanged();
    }
}
```
没什么特别需要说的。

为了将所有东西放在 MainActivity 类中，我们添加了这个方法：
```java
private void setupFeed() {
    LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
    rvFeed.setLayoutManager(linearLayoutManager);
    feedAdapter = new FeedAdapter(this);
    rvFeed.setAdapter(feedAdapter);
}
```
现在就这些了。这里是 MainActivity 类的完整源代码：

```java
//MainActivity.java
public class MainActivity extends ActionBarActivity {
    @InjectView(R.id.toolbar)
    Toolbar toolbar;
    @InjectView(R.id.rvFeed)
    RecyclerView rvFeed;

    private MenuItem inboxMenuItem;
    private FeedAdapter feedAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.inject(this);

        setupToolbar();
        setupFeed();
    }

    private void setupToolbar() {
        setSupportActionBar(toolbar);
        toolbar.setNavigationIcon(R.drawable.ic_menu_white);
    }

    private void setupFeed() {
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
        rvFeed.setLayoutManager(linearLayoutManager);
        feedAdapter = new FeedAdapter(this);
        rvFeed.setAdapter(feedAdapter);
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.menu_main, menu);
        inboxMenuItem = menu.findItem(R.id.action_inbox);
        inboxMenuItem.setActionView(R.layout.menu_item_view);
        return true;
    }
}
```
如果你在这里遗漏了一些东西，你可以查看我们[在这一刻的项目提交](https://github.com/frogermcs/InstaMaterial/commit/1095d5f199ceb5c04bb2433c8865dcea2cba6f2e)。 当您在您的设备上构建并运行应用程序时，您应该看到以下屏幕：
- v21
  ![|center](http://frogermcs.github.io/images/2/screen-21.png)
- pre-21

  ![|center](http://frogermcs.github.io/images/2/screen-pre21.png)

## 动画
要实现的最后和最重要的事情是介绍动画。 再看一遍概念视频，让我们列出所有在 MainActivity 开始时执行的动画。 在我看来，我们可以把它分成两个步骤:

- 显示带有内部元素的工具栏
- 在工具栏动画完成后显示项目和浮动动作按钮。

工具栏项目动画在短时间内一个接一个地执行。实现这个动画的主要问题是导航图标——我们无法动画的唯一元素。 剩下的就很简单了。

#### 工具栏动画

首先，我们只想在启动应用程序时（而不是在这之后，我们将旋转我们的设备）动画我们的应用程序。同时请记住，我们正在启动 onCreate () 方法中没有的菜单项(我们正在 createoptionsmenu () 中初始化它)。

让我们将 pendingIntroAnimation 布尔值添加到我们的 MainActivity 中。我们将在 onCreate () 方法中初始化它：
```java
//...
if (savedInstanceState == null) {
    pendingIntroAnimation = true;
}
```
并在 onCreateOptionsMenu () 中使用：
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu_main, menu);
    inboxMenuItem = menu.findItem(R.id.action_inbox);
    inboxMenuItem.setActionView(R.layout.menu_item_view);
    if (pendingIntroAnimation) {
        pendingIntroAnimation = false;
        startIntroAnimation();
    }
    return true;
}
```
startIntroAnimation () 在我们开始我们的应用程序之后将被调用一次。

现在是时候为工具栏及其内部项目准备动画了。这也是非常重要的。请记住，一般来说，动画是由两个步骤组成的：

- 准备 - 在这里我们必须为每个动画元素设置初始状态。如果我们要执行扩展动画，这里我们必须确保我们的项目是隐藏的。
- 动画 - 在这一步，我们的视图是动画的最终状态或 / 和位置

那么，我们来尝试实现工具栏动画的两个步骤：
```java
//...
private static final int ANIM_DURATION_TOOLBAR = 300;

private void startIntroAnimation() {
    btnCreate.setTranslationY(2 * getResources().getDimensionPixelOffset(R.dimen.btn_fab_size));
    
    int actionbarSize = Utils.dpToPx(56);
    toolbar.setTranslationY(-actionbarSize);
    ivLogo.setTranslationY(-actionbarSize);
    inboxMenuItem.getActionView().setTranslationY(-actionbarSize);

    toolbar.animate()
            .translationY(0)
            .setDuration(ANIM_DURATION_TOOLBAR)
            .setStartDelay(300);
    ivLogo.animate()
            .translationY(0)
            .setDuration(ANIM_DURATION_TOOLBAR)
            .setStartDelay(400);
    inboxMenuItem.getActionView().animate()
            .translationY(0)
            .setDuration(ANIM_DURATION_TOOLBAR)
            .setStartDelay(500)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    startContentAnimation();
                }
            })
            .start();
}
//...
```
我们现在看到的是:

- 首先，把所有的元素都隐藏起来，把它们从屏幕上平移出来(在这一步中，我们还隐藏了 FAB)。
- 动画工具栏和内部项目，一个接一个。
- 动画完成后，为内容添加动画（FAB 和 Feed 项目）。

很简单，对吧？

#### 内容动画

在这个步骤中，我们想要动画两个东西: FAB 和 Feed。 虽然 FAB 动画非常简单(只要做上面描述的类似操作) ，动画 Feed 项目是一个有点棘手。

对于这个例子，简单的解决方案是保持 feedAdapter 为空，并在我们想要动画项目的时候填充它。 然后在适配器中直接动画它们，同时创建项目。 在真正的代码中它看起来像什么？ 看一看:

- startContentAnimation () 方法
```java
//...
//FAB animation
private static final int ANIM_DURATION_FAB = 400;

private void startContentAnimation() {
    btnCreate.animate()
            .translationY(0)
            .setInterpolator(new OvershootInterpolator(1.f))
            .setStartDelay(300)
            .setDuration(ANIM_DURATION_FAB)
            .start();
    feedAdapter.updateItems();
}
//...
```
- FeedAdapter 与动画 feed 项目
```java
public class FeedAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
    private static final int ANIMATED_ITEMS_COUNT = 2;

    private Context context;
    private int lastAnimatedPosition = -1;
    private int itemsCount = 0;

    public FeedAdapter(Context context) {
        this.context = context;
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.item_feed, parent, false);
        return new CellFeedViewHolder(view);
    }

    private void runEnterAnimation(View view, int position) {
        if (position >= ANIMATED_ITEMS_COUNT - 1) {
            return;
        }

        if (position > lastAnimatedPosition) {
            lastAnimatedPosition = position;
            view.setTranslationY(Utils.getScreenHeight(context));
            view.animate()
                    .translationY(0)
                    .setInterpolator(new DecelerateInterpolator(3.f))
                    .setDuration(700)
                    .start();
        }
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int position) {
        runEnterAnimation(viewHolder.itemView, position);
        CellFeedViewHolder holder = (CellFeedViewHolder) viewHolder;
        if (position % 2 == 0) {
            holder.ivFeedCenter.setImageResource(R.drawable.img_feed_center_1);
            holder.ivFeedBottom.setImageResource(R.drawable.img_feed_bottom_1);
        } else {
            holder.ivFeedCenter.setImageResource(R.drawable.img_feed_center_2);
            holder.ivFeedBottom.setImageResource(R.drawable.img_feed_bottom_2);
        }
    }

    @Override
    public int getItemCount() {
        return itemsCount;
    }

    public static class CellFeedViewHolder extends RecyclerView.ViewHolder {
        @InjectView(R.id.ivFeedCenter)
        SquaredImageView ivFeedCenter;
        @InjectView(R.id.ivFeedBottom)
        ImageView ivFeedBottom;

        public CellFeedViewHolder(View view) {
            super(view);
            ButterKnife.inject(this, view);
        }
    }

    public void updateItems() {
        itemsCount = 10;
        notifyDataSetChanged();
    }
}
```
就这样！如果我们构建并运行我们的项目，我们有最后版本显示在本帖的开头。 如果你错过了一些东西，这里是[这里是实现动画的项目提交](https://github.com/frogermcs/InstaMaterial/commit/4e838861c5f858711b1072777cae2325ce12ee21)。

## 源代码 

所描述示例的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。


