# InstaMaterial 概念(第7部分) - 导航抽屉

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 7) - Navigation Drawer](https://mirekstanek.online/instamaterial-concept-part-7-navigation-drawer/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列帖子的一部分，展示了使用 [Material Design 概念的 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q) 的 Android 实现。 今天，我们将创建导航抽屉 - 左侧滑动面板，显示全局应用程序菜单。 这个元素在概念视频的第32和第35秒之间呈现。

此外，我们还将创建 [DrawerLayoutInstaller](https://github.com/frogermcs/DrawerLayoutInstaller)-简单的工具，用于将 DrawerLayout 注入活动布局而不会破坏 xml 文件。

以下是今天的文章中描述的最后效果:

<iframe width="560" height="315" src="https://youtu.be/rRYN1le1-ZM" frameborder="0" allowfullscreen></iframe>

## 概论
导航抽屉是在安卓（和其他移动平台）中使用的非常着名的设计模式。 这是一个从屏幕左边过渡的面板，并显示应用程序的主要导航选项。 有时它会从右边过渡，但是在你有充分的理由之前，这是相当糟糕的设计实现，你不应该复制。

导航抽屉模式也有很好的文档（从设计和编程两个方面）。 如果你想深入设计规则，只需检查[导航抽屉设计指南](https://developer.android.com/design/patterns/navigation-drawer.html)。 今天我也不会写太多关于这个模式的编程。 相反，我建议您阅读[创建导航抽屉的官方文档](https://developer.android.com/training/implementing-navigation/nav-drawer.html)。

在这篇文章中，我们不是复制文档，而是准备 DrawerLayoutInstaller - 一个简单的工具的存根，它可以帮助我们配置和注入 DrawerLayout 到每个活动，而不是把每个 xml 文件都弄乱。

## 导航抽屉实施
**准备**

在我们开始之前，让我们通过增加资源做一些准备，而不是那些意味深长的事情。。 首先，我们必须添加应用程序菜单中使用的所有图标：
![|center](http://frogermcs.github.io/images/8/new-resources.png)

并不是每一个都像概念视频中的那样，但都是从[谷歌的材料设计图标](https://github.com/google/material-design-icons)。 我们的菜单是建立在 ListView 之上的，所以我们必须为其中使用的列表项目准备布局。这里有三种不同的布局：

- 标题与用户头像和昵称
  ![|center](http://frogermcs.github.io/images/8/menu_header.png)
- 菜单按钮
  ![|center](http://frogermcs.github.io/images/8/menu_item.png)
- 分隔符（在菜单按钮之间使用）
```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <View
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:background="#dddddd" />
</FrameLayout>
```
这里是[所有新资源的全部提交](https://github.com/frogermcs/InstaMaterial/commit/4f42f9693fb74839261dd26759f15b04d0034073)。

## GlobalMenuView
现在让我们为我们的全局菜单准备布局。 基本上它是一个简单的 ListView，我们可以在 xml 文件中构建它。 但是如果我们想把这个视图注入到更多的活动中，最好以编程的方式进行准备。

这个视图的要求很简单 - 只显示选项列表和用户资料布局。 所以实现不是太复杂：
```java
public class GlobalMenuView extends ListView implements View.OnClickListener {

    private OnHeaderClickListener onHeaderClickListener;
    private GlobalMenuAdapter globalMenuAdapter;
    private ImageView ivUserProfilePhoto;
    private int avatarSize;
    private String profilePhoto;

    public GlobalMenuView(Context context) {
        super(context);
        init();
    }

    private void init() {
        setChoiceMode(CHOICE_MODE_SINGLE);
        setDivider(getResources().getDrawable(android.R.color.transparent));
        setDividerHeight(0);
        setBackgroundColor(Color.WHITE);

        setupHeader();
        setupAdapter();
    }

    private void setupAdapter() {
        globalMenuAdapter = new GlobalMenuAdapter(getContext());
        setAdapter(globalMenuAdapter);
    }

    private void setupHeader() {
        this.avatarSize = getResources().getDimensionPixelSize(R.dimen.global_menu_avatar_size);
        this.profilePhoto = getResources().getString(R.string.user_profile_photo);

        setHeaderDividersEnabled(true);
        View vHeader = LayoutInflater.from(getContext()).inflate(R.layout.view_global_menu_header, null);
        ivUserProfilePhoto = (ImageView) vHeader.findViewById(R.id.ivUserProfilePhoto);
        Picasso.with(getContext())
                .load(profilePhoto)
                .placeholder(R.drawable.img_circle_placeholder)
                .resize(avatarSize, avatarSize)
                .centerCrop()
                .transform(new CircleTransformation())
                .into(ivUserProfilePhoto);
        addHeaderView(vHeader);
        vHeader.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        if (onHeaderClickListener != null) {
            onHeaderClickListener.onGlobalMenuHeaderClick(v);
        }
    }

    public interface OnHeaderClickListener {
        public void onGlobalMenuHeaderClick(View v);
    }

    public void setOnHeaderClickListener(OnHeaderClickListener onHeaderClickListener) {
        this.onHeaderClickListener = onHeaderClickListener;
    }
}
```
您可能注意到，用户资料视图用作 ListView 的标题。这就是为什么我们必须为 onClick 处理添加自定义侦听器。其余的很简单。

GlobalMenuAdapter 更简单，所以我不会在这里粘贴源代码。相反，使用 [GlobalMenuView 实现来检查这个提交](https://github.com/frogermcs/InstaMaterial/commit/cfb4ee819fea9fad852a2eeda671b15ccac6ade2)。

## 导航抽屉
现在是创建导航抽屉实施的时候了。首先让我们为我们的 DrawerLayout 准备 xml 根视图：
```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/drawerLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <FrameLayout
        android:id="@+id/vContentFrame"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <FrameLayout
        android:id="@+id/vLeftDrawer"
        android:layout_width="280dp"
        android:layout_height="match_parent"
        android:layout_gravity="start" />
</android.support.v4.widget.DrawerLayout>
```
此通用布局可以托管自定义根视图（在 vContentFrame 元素中）和自定义左抽屉项（vLeftDrawer 元素）。

## DrawerLayoutInstaller
现在是时候准备我们的工具，将 DrawerLayout 注入到 Activity 布局树中。可能 DrawerLayoutInstaller 将在未来开发，但现在需求列表是简短而简单的:

- 将 DrawerLayout 注入到现有的 Activity 布局而不会搞乱它的 xml 文件，
- 可以定义自定义 DrawerLayout xml 根目录和自定义左侧项目视图，
- 一些自定义（左项目宽度，打开/关闭抽屉自定义切换器）

**第一点是用三个简单的步骤完成的：**
- 从给定的 Activity 获取根视图（我们可以通过 activity.findViewById（android.R.id.content）得到它），并把它的唯一的孩子
- 将抽屉布局作为新的孩子在根视图中
- 把带孩子放进抽屉的内容视图

**实现**
```java
private void addDrawerToActivity(DrawerLayout drawerLayout) {
    ViewGroup rootView = (ViewGroup) activity.findViewById(android.R.id.content);
    ViewGroup drawerContentRoot = (ViewGroup) drawerLayout.getChildAt(0);
    View contentView = rootView.getChildAt(0);

    rootView.removeView(contentView);

    drawerContentRoot.addView(contentView, new ViewGroup.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT
    ));

    rootView.addView(drawerLayout, new ViewGroup.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT
    ));
}
```
其余的你可以在完整的 [DrawerLayoutInstaller](https://github.com/frogermcs/DrawerLayoutInstaller) 实现中找到。

最后我们可以把所有东西首先在 BaseActivity 中，我们必须构建我们的 GlobalMenuView，然后把它放到 DrawerLayout 中，然后把所有的内容都注入活动布局中:

```java
private void setupDrawer() {
    GlobalMenuView menuView = new GlobalMenuView(this);
    menuView.setOnHeaderClickListener(this);

    drawerLayout = DrawerLayoutInstaller.from(this)
            .drawerRoot(R.layout.drawer_root)
            .drawerLeftView(menuView)
            .drawerLeftWidth(Utils.dpToPx(300))
            .withNavigationIconToggler(getToolbar())
            .build();
}
```
最后一件事 - 实现 onGlobalMenuHeaderClick () 应该打开 ProfileAcitivty：
```java
@Override
public void onGlobalMenuHeaderClick(final View v) {
    drawerLayout.closeDrawer(Gravity.START);
    new Handler().postDelayed(new Runnable() {
        @Override
        public void run() {
            int[] startingLocation = new int[2];
            v.getLocationOnScreen(startingLocation);
            startingLocation[0] += v.getWidth() / 2;
            UserProfileActivity.startUserProfileFromLocation(startingLocation, BaseActivity.this);
            overridePendingTransition(0, 0);
        }
    }, 200);
}
```
在这种情况下处理程序延迟打开，直到 DrawerLayout 关闭。这是最后的效果：

![|center](http://frogermcs.github.io/images/8/navigation_drawer1.gif)
这就是今天。这是非常快速和简单，一如既往。谢谢阅读！ 😄

## 示例代码
 所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。


