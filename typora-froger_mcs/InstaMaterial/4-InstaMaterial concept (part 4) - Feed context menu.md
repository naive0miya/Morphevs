# InstaMaterial 概念(第4部分) - Feed 上下文菜单

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 4) - Feed context menu](https://mirekstanek.online/instamaterial-concept-part-4-feed-context-menu/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的一部分，这些帖子展示了 Android 实现 [INSTAGRAM 的 Material Design 概念](https://www.youtube.com/watch?v=ojwdmgmdR_Q)。今天我们将为 feed 项目的 "more" 按钮打开上下文菜单。 这个元素是从概念视频18到20秒的时间内呈现的。

这是今天发布的最终效果（针对 v21 和 Pre_21 版本）：

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>

## 热身
在我们开始使用上下文菜单之前，让我们做一些重构。

正如 @Riyaz 在他的[评论](http://frogermcs.github.io/Instagram-with-Material-Design-concept-part-2-Comments-transition/#comment-1731187178)中指出的，MainActivity 和 CommentsActivity 中工具栏的代码是相同的。由于 `<include />` 标签，我们可以将部分布局导出到新文件，并在需要的地方重新使用。

这是在实践中的样子。在我们的项目中，我们创建了新的文件 res / layout / view_feed_toolbar.xml：
```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.Toolbar xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/tools"
    android:id="@+id/toolbar"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:background="?attr/colorPrimary"
    android:elevation="@dimen/default_elevation"
    app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

    <ImageView
        android:id="@+id/ivLogo"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="center"
        android:scaleType="center"
        android:src="@drawable/img_toolbar_logo" />
</android.support.v7.widget.Toolbar>
```
而我们现在所需要的就是在 activity_main.xml 和 activity_comments.xml 中使用它，如下所示：
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <!--...-->
    
    <include
        android:id="@+id/toolbar"
        layout="@layout/view_feed_toolbar" />

    <!--...-->

</RelativeLayout>
```
值得一提的是，我们可以覆盖 `<include />` 标签中的属性（它们将从包含文件中覆盖根视图中的属性）。就像我们用 android:id 属性所做的一样（我们必须做，为了在 RelativeLayout 中正确的布局定位）。

这里是描述重构的[提交](https://github.com/frogermcs/InstaMaterial/commit/861cc0a84a5fdd97a56d23709018c68728f97e53)。

## 上下文菜单

**初始配置**

像往常一样，在我们开始编程之前做一些准备。以下是期望效果的截图：
![|center](http://frogermcs.github.io/images/5/context_menu_screen.png)
首先，我们必须在 item_feed.xml 布局中添加另一个按钮，并处理其 onClick () 事件。 非常直接，只是复制和粘贴一些以前的代码，并添加缺少的三点图像（我使用官方的材料设计图标包[这一个](https://github.com/google/material-design-icons/blob/master/navigation/drawable-xxhdpi/ic_more_vert_grey600_24dp.png)）。 这是全部变化的完全[提交](https://github.com/frogermcs/InstaMaterial/commit/9eb019a75d58a14a9cee1654dd18279ed2db2a31)。

## 上下文菜单布局
我们从视图的布局开始。 我们再一次使用 `<merge />` 标记并构建我们的 .xml 视图和 java 代码。
res / layout / view_context_menu.xml 很简单：
```xml
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <Button
        android:id="@+id/btnReport"
        style="@style/ContextMenuButton"
        android:text="REPORT"
        android:textColor="@color/btn_context_menu_text_red" />

    <Button
        android:id="@+id/btnSharePhoto"
        style="@style/ContextMenuButton"
        android:text="SHARE PHOTO" />

    <Button
        android:id="@+id/btnCopyShareUrl"
        style="@style/ContextMenuButton"
        android:text="COPY SHARE URL" />

    <View
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:background="#eeeeee" />

    <Button
        android:id="@+id/btnCancel"
        style="@style/ContextMenuButton"
        android:text="CANCEL" />

</merge>
```
四个按钮共享样式定义如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- styles.xml-->
<resources>

    <!--...-->

    <style name="ContextMenuButton">
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:background">@drawable/btn_context_menu</item>
        <item name="android:gravity">left|center_vertical</item>
        <item name="android:paddingLeft">20dp</item>
        <item name="android:paddingRight">20dp</item>
        <item name="android:textColor">?attr/colorPrimary</item>
        <item name="android:textSize">14sp</item>
    </style>

</resources>
```
没什么特别的。与我们视图的源代码一样

```java
public class FeedContextMenu extends LinearLayout {
    private static final int CONTEXT_MENU_WIDTH = Utils.dpToPx(240);

    private int feedItem = -1;

    private OnFeedContextMenuItemClickListener onItemClickListener;

    public FeedContextMenu(Context context) {
        super(context);
        init();
    }

    private void init() {
        LayoutInflater.from(getContext()).inflate(R.layout.view_context_menu, this, true);
        setBackgroundResource(R.drawable.bg_container_shadow);
        setOrientation(VERTICAL);
        setLayoutParams(new LayoutParams(CONTEXT_MENU_WIDTH, ViewGroup.LayoutParams.WRAP_CONTENT));
    }

    public void bindToItem(int feedItem) {
        this.feedItem = feedItem;
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        ButterKnife.inject(this);
    }

    public void dismiss() {
        ((ViewGroup) getParent()).removeView(FeedContextMenu.this);
    }

    @OnClick(R.id.btnReport)
    public void onReportClick() {
        if (onItemClickListener != null) {
            onItemClickListener.onReportClick(feedItem);
        }
    }

    @OnClick(R.id.btnSharePhoto)
    public void onSharePhotoClick() {
        if (onItemClickListener != null) {
            onItemClickListener.onSharePhotoClick(feedItem);
        }
    }

    @OnClick(R.id.btnCopyShareUrl)
    public void onCopyShareUrlClick() {
        if (onItemClickListener != null) {
            onItemClickListener.onCopyShareUrlClick(feedItem);
        }
    }

    @OnClick(R.id.btnCancel)
    public void onCancelClick() {
        if (onItemClickListener != null) {
            onItemClickListener.onCancelClick(feedItem);
        }
    }

    public void setOnFeedMenuItemClickListener(OnFeedContextMenuItemClickListener onItemClickListener) {
        this.onItemClickListener = onItemClickListener;
    }

    public interface OnFeedContextMenuItemClickListener {
        public void onReportClick(int feedItem);

        public void onSharePhotoClick(int feedItem);

        public void onCopyShareUrlClick(int feedItem);

        public void onCancelClick(int feedItem);
    }
}
```

在第20行，我们有一个模拟绑定视图模型。在真实的项目中可能会更复杂。 第30行的 dismiss () 方法帮助我们从父视图中删除我们的菜单。其余的很简单。

## Ninepatch 背景
当你很可能注意到我们的菜单背景有阴影。 在 v21 中，我们可以通过分配高度值来实现这种效果。 不幸的是，它只适用于最新的安卓系统，即使我们使用 ViewCompat.setElevation (); （如果你深入研究 ViewCompat 的实现，你会发现简单的 if 语句来检查方法是在 v21 上调用还是在 pre-21 的版本中调用，而在第二种情况下它什么也不做）。

这就是为什么我们的菜单的背景，我们将使用 Ninepatch 背景。 对于那些之前没有使用过这个的人来说，这就是一张图，其中规定了拉伸和内容放置的规则。 它有助于创建可调整大小而不会丢失清晰度和质量的图像（Android 使用此方法即可用于按钮背景）。

下面是我们的背景图片:
![|center](http://frogermcs.github.io/images/5/bg_container_shadow.9.png)

左边和上边的线定义了应该拉伸的区域（我们只定义这个区域，它可以被安全地移除或者垂直或水平重叠）。 右和底线定义内容区域（将行开始和结束转换为查看填充的位置）。

在这里你可以找到关于 [NinePatch](http://radleymarx.com/blog/simple-guide-to-9-patch/) 更详细的解释。

## 选择器
上下文菜单布局中的最后一件事情当然是 onClick 选择器。再次，它只是从现有的代码复制和粘贴（v21 和 pre-21 版本的两个不同的文件）：

- res/drawable/btn_context_menu.xml/:
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/btn_context_menu.xml-->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@color/btn_context_menu_normal" android:state_focused="false" android:state_pressed="false" />
    <item android:drawable="@color/btn_context_menu_pressed" android:state_pressed="true" />
    <item android:drawable="@color/btn_context_menu_pressed" android:state_focused="true" />
</selector>
```
- res/drawable-v21/btn_context_menu.xml/:
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable-v21/btn_context_menu.xml-->
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/btn_context_menu_pressed">
    <item>
        <shape android:shape="rectangle">
            <solid android:color="@color/btn_context_menu_normal" />
        </shape>
    </item>
</ripple>
```

上下文菜单管理器
![|center](http://frogermcs.github.io/images/5/context-menu.gif)
好极了，现在当我们为上下文菜单进行整体布局时，让我们准备好 FeedContextMenuManager，以便于管理。这是要求：
- 屏幕上只能显示一个上下文菜单
- 我们点击“更多”按钮后应该出现上下文菜单，再次点击此按钮后会消失
- 上下文菜单应该出现在点击按钮的正上方
- 当我们开始滚动项目时，上下文菜单应该会消失

实现 FeedContextMenuManager 作为单例模式的最简单的方法。

我们的管理器将处理 contextMenuView 引用，所以最重要的是在适当的时候将其删除。 为此，我们可以使用 OnAttachStateChangeListener 及其 onViewDetachedFromWindow（View v）方法。 谢谢这个视图引用, 当我们分离它或当托管这个菜单的活动将被销毁，视图将被删除。
```java
public class FeedContextMenuManager implements View.OnAttachStateChangeListener {

    private static FeedContextMenuManager instance;

    private FeedContextMenu contextMenuView;

    public static FeedContextMenuManager getInstance() {
        if (instance == null) {
            instance = new FeedContextMenuManager();
        }
        return instance;
    }


    @Override
    public void onViewAttachedToWindow(View v) {

    }

    @Override
    public void onViewDetachedFromWindow(View v) {
        contextMenuView = null;
    }

}
```
对于显示和隐藏视图，我们将非常简单：
```java
public void toggleContextMenuFromView(View openingView, int feedItem, FeedContextMenu.OnFeedContextMenuItemClickListener listener) {
    if (contextMenuView == null) {
        showContextMenuFromView(openingView, feedItem, listener);
    } else {
        hideContextMenu();
    }
}
```
我们将直接从“更多按钮” onClick 回调中调用它

很好，现在让我们来看看我们的菜单。 首先，我们必须关注 "isAnimating" 状态(隐藏和显示情况)。 当动画执行时，屏蔽我们的管理器是很重要的，这样可以避免同时执行两次或者更多的动画(同时避免与此相关的错误)。

其余部分的代码（下面的一些解释）：
```java
private void showContextMenuFromView(final View openingView, int feedItem, FeedContextMenu.OnFeedContextMenuItemClickListener listener) {
    if (!isContextMenuShowing) {
        isContextMenuShowing = true;
        contextMenuView = new FeedContextMenu(openingView.getContext());
        contextMenuView.bindToItem(feedItem);
        contextMenuView.addOnAttachStateChangeListener(this);
        contextMenuView.setOnFeedMenuItemClickListener(listener);

        ((ViewGroup) openingView.getRootView().findViewById(android.R.id.content)).addView(contextMenuView);

        contextMenuView.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
            @Override
            public boolean onPreDraw() {
                contextMenuView.getViewTreeObserver().removeOnPreDrawListener(this);
                setupContextMenuInitialPosition(openingView);
                performShowAnimation();
                return false;
            }
        });
    }
}

private void setupContextMenuInitialPosition(View openingView) {
    final int[] openingViewLocation = new int[2];
    openingView.getLocationOnScreen(openingViewLocation);
    int additionalBottomMargin = Utils.dpToPx(16);
    contextMenuView.setTranslationX(openingViewLocation[0] - contextMenuView.getWidth() / 3);
    contextMenuView.setTranslationY(openingViewLocation[1] - contextMenuView.getHeight() - additionalBottomMargin);
}

private void performShowAnimation() {
    contextMenuView.setPivotX(contextMenuView.getWidth() / 2);
    contextMenuView.setPivotY(contextMenuView.getHeight());
    contextMenuView.setScaleX(0.1f);
    contextMenuView.setScaleY(0.1f);
    contextMenuView.animate()
            .scaleX(1f).scaleY(1f)
            .setDuration(150)
            .setInterpolator(new OvershootInterpolator())
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    isContextMenuShowing = false;
                }
            });
}
```
我们在这里有三个阶段：
- 上下文菜单初始化 - 构建对象，设置一些东西，并添加菜单到我们的根视图。 对于最后一个，我们使用了android.R.id.content，它总是指向活动的根视图。 在我们的例子中添加视图非常简单，因为我们知道我们的根视图是 RelativeLayout（它也适用于 FrameLayout）。 在其他情况下，我们必须准备更复杂的解决方案。
- 菜单定位 - 我们知道最初我们的菜单放置在活动的左上角。 我们也知道在屏幕上的开放视图和他的位置。 这只是一个简单的数学。 这一刻最重要的是在适当的时候开始定位。 我们必须在 onPreDraw () 回调中执行此操作，以确保我们的视图已经布局。 只有这样我们才能在视图上使用 getWidth () 或 getHeight () 方法（在此之前这些方法返回0）。
- 动画菜单 - 无非是 ViewPropertyAnimator

隐藏菜单更简单，不需要解释:

```java
public void hideContextMenu() {
    if (!isContextMenuDismissing) {
        isContextMenuDismissing = true;
        performDismissAnimation();
    }
}

private void performDismissAnimation() {
    contextMenuView.setPivotX(contextMenuView.getWidth() / 2);
    contextMenuView.setPivotY(contextMenuView.getHeight());
    contextMenuView.animate()
            .scaleX(0.1f).scaleY(0.1f)
            .setDuration(150)
            .setInterpolator(new AccelerateInterpolator())
            .setStartDelay(100)
            .setListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    if (contextMenuView != null) {
                        contextMenuView.dismiss();
                    }
                    isContextMenuDismissing = false;
                }
            });
}
```
我们要求的最后一件事是在用户滚动 Feed 时隐藏视图。我们将为此扩展 RecyclerView.OnScrollListener 类：
```java
public class FeedContextMenuManager extends RecyclerView.OnScrollListener implements View.OnAttachStateChangeListener {

    //...

    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
        if (contextMenuView != null) {
            hideContextMenu();
            contextMenuView.setTranslationY(contextMenuView.getTranslationY() - dy);
        }
    }

    //..
}
```
正如你所看到的那样有点棘手。在我们的菜单隐藏的同时，我们使用 setTranslationY ()。感谢这个我们的菜单如下滚动：
![|center](http://frogermcs.github.io/images/5/context-menu-hiding.gif)
我们现在要做的就是使用我们的 FeedContextMenuManager 。这是[最后一次提交](https://github.com/frogermcs/InstaMaterial/commit/510f4d8ae88afeb9996513de077c0983bba686d8)，它显示了我们在项目中是如何做到的。今天就到这里吧。 😄

## 示例代码 
所描述示例的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。 



