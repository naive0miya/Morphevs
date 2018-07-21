# InstaMaterial 概念(第2部分) - 评论窗口转换 

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 2) - Comments window transition](https://mirekstanek.online/instamaterial-concept-part-2-comments-window-transition/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的一部分，这些帖子展示了 Android 实现 [INSTAGRAM 的 Material Design 概念](https://www.youtube.com/watch?v=ojwdmgmdR_Q)。 今天我们将实现 feed 和 comments 活动之间的过渡(在概念视频中显示在9到13秒之间)。 我们将跳过按钮效果(波纹、发送完整动画等) ，只关注进入和退出动画的评论的活动。

这是今天发布的最终效果（针对 Android v21 和 pre_21 版本）：

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>

## 初始配置

让我们从添加样板和不重要的东西开始，到我们[之前创建的项目](https://github.com/frogermcs/InstaMaterial)。 我们不得不补充一点:

- 用于异步加载图片的 [Picasso](http://square.github.io/picasso/) 库（用于评论列表，供用户头像使用），
- AndroidManifest.xml 文件中带有样式和声明的新评论活动。

同时我们也为这个活动创建布局。一切都和 MainAcitvity 几乎一样，除了添加评论的底部组件。 我们再次使用工具栏，RecyclerView 和一些额外的东西。一切都很简单，不需要额外的评论：
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".CommentsActivity">

    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="?attr/colorPrimary"
        android:elevation="@dimen/default_elevation">

        <ImageView
            android:id="@+id/ivLogo"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="center"
            android:scaleType="center"
            android:src="@drawable/img_toolbar_logo" />
    </android.support.v7.widget.Toolbar>

    <LinearLayout
        android:id="@+id/contentRoot"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_below="@id/toolbar"
        android:background="@color/bg_comments"
        android:elevation="@dimen/default_elevation"
        android:orientation="vertical">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/rvComments"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:scrollbars="none" />

        <LinearLayout
            android:id="@+id/llAddComment"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="@color/bg_comments"
            android:elevation="@dimen/default_elevation">

            <EditText
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1" />

            <Button
                android:id="@+id/btnSendComment"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Send" />
        </LinearLayout>
    </LinearLayout>
</RelativeLayout>
```
![|center](http://frogermcs.github.io/images/3/comments_activity_preview.png)
接下来，在评论列表中为项目创建布局：
```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:paddingBottom="8dp"
        android:paddingTop="8dp">

        <ImageView
            android:id="@+id/ivUserAvatar"
            android:layout_width="@dimen/comment_avatar_size"
            android:layout_height="@dimen/comment_avatar_size"
            android:layout_marginLeft="16dp"
            android:layout_marginRight="16dp"
            android:background="@drawable/bg_comment_avatar" />

        <TextView
            android:id="@+id/tvComment"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_gravity="center_vertical"
            android:layout_marginRight="16dp"
            android:layout_weight="1"
            android:text="Lorem ipsum dolor sit amet" />
    </LinearLayout>

    <View
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:layout_gravity="bottom"
        android:layout_marginLeft="88dp"
        android:background="#cccccc" />
</FrameLayout>
```
![|center](http://frogermcs.github.io/images/3/comments_list_item.png)
在评论头像中使用的圆形背景的片段：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/bg_comment_avar.xml-->
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="oval">
    <solid android:color="#999999" />
</shape>
```
最后，我们来处理点击，点击 feed 项目的底部，它应该为当前的照片打开 CommentsActivity。现在我们使用模拟视图的整个底部，但在不久的将来，我们会用真正的按钮替换它。 在这一步中，我们必须在 RecyclerView 适配器中添加 onClick 监听器。 这个侦听器将被设置在列表中的每个项目的底部视图中。 另外，我们将创建一个简单的界面来将 MainActivity 与我们的适配器集成。 它在实践中看起来如何？
 FeedAdapter 类（仅更改）：

```java
//.. implements View.OnClickListener
public class FeedAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> implements View.OnClickListener {

    private OnFeedItemClickListener onFeedItemClickListener;

    //..

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int position) {
        //...
        holder.ivFeedBottom.setOnClickListener(this);
        holder.ivFeedBottom.setTag(position);
    }

    //..

    @Override
    public void onClick(View v) {
        if (v.getId() == R.id.ivFeedBottom) {
            if (onFeedItemClickListener != null) {
                onFeedItemClickListener.onCommentsClick(v, (Integer) v.getTag());
            }
        }
    }

    public void setOnFeedItemClickListener(OnFeedItemClickListener onFeedItemClickListener) {
        this.onFeedItemClickListener = onFeedItemClickListener;
    }

    //..

    public interface OnFeedItemClickListener {
        public void onCommentsClick(View v, int position);
    }
}
```
MainActivity 类（仅更改）：
```java
//... implements FeedAdapter.OnFeedItemClickListener
public class MainActivity extends ActionBarActivity implements FeedAdapter.OnFeedItemClickListener {

    //...

    private void setupFeed() {
        //...
        feedAdapter.setOnFeedItemClickListener(this);
        //...
    }

    //...
    
    @Override
    public void onCommentsClick(View v, int position) {
    }
}
```
如果我们错过了某些东西，可以在 [RecyclerView 适配器中点击 onClick 项目](https://github.com/frogermcs/InstaMaterial/commit/ec3d47bd546f4bdcb7ba1a2a5afe58112972ea0a)中查看。 ...这是所有初始配置。

## 进入过渡
一开始我们将创建进入过渡。基于概念视频，这里是我们想要实现的效果列表：
- Static Toolbar - 新的活动应该打开没有 ActionBar 移动（我们要欺骗用户，他仍然在同一个窗口）
- 评论视图应该从用户点击的位置展开（不管当前的滚动位置是什么）
- 评论应该一个接一个地显示，扩展动画之后就会完成。

## 静态工具栏

这篇文章中最简单的一步。 感谢那个工具栏在 MainActivity 和 CommentsActivity 上看起来类似，我们只需要在 Acitvities 之间禁用默认转换。 之后，新窗口将在前一个窗口上绘制，而没有任何移动。 它会模仿静态的工具栏效果。 这是它在代码中的外观：

```java
public class MainActivity extends ActionBarActivity implements FeedAdapter.OnFeedItemClickListener {
    //...
    @Override
    public void onCommentsClick(View v, int position) {
        final Intent intent = new Intent(this, CommentsActivity.class);
        startActivity(intent);
        //Disable enter transition for new Acitvity
        overridePendingTransition(0, 0);
    }
}
```
通过调用  overridePendingTransition(0, 0); 我们禁用退出动画（对于 MainActivity）和进入动画（对于CommentsActivity）。

## 从点击的地方扩展评论活动
现在我们要创建可以从屏幕上任意（垂直）位置开始的展开动画。这个动画包括两部分：
- 展开背景
- 显示内容

在我们开始编写我们的动画之前，我们必须使评论活动半透明。 否则，将在默认窗口背景之上对我们的后台进行扩展，而不是 MainAcitvity 视图。这是因为每个活动都有 windowBackground 属性定义在当前使用的主题。 如果我们想要禁用它，使我们的 Acitivty 半透明，我们必须修改下面的样式:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- styles.xml-->
<resources>

    <!--...-->

    <style name="AppTheme.CommentsActivity" parent="AppTheme">
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowIsTranslucent">true</item>
    </style>
</resources>
```
以下是区别：
![|center](http://frogermcs.github.io/images/3/expand_translucent.gif)
![|center](http://frogermcs.github.io/images/3/expand_not_translucent.gif)

现在我们可以创造出扩展效果。 首先，我们必须获得动画的初始位置。 在我们的例子中，我们不需要知道选择地点的确切位置(动画会非常快，用户不会注意到起始点不是超级精确)。 我们可以使用 y 位置的点击视图并将其传递给 CommentsActivity:

```java
public class MainActivity extends ActionBarActivity implements FeedAdapter.OnFeedItemClickListener {
    
    //...

    @Override
    public void onCommentsClick(View v, int position) {
        final Intent intent = new Intent(this, CommentsActivity.class);
        
        //Get location on screen for tapped view
        int[] startingLocation = new int[2];
        v.getLocationOnScreen(startingLocation);
        intent.putExtra(CommentsActivity.ARG_DRAWING_START_LOCATION, startingLocation[1]);
        
        startActivity(intent);
        overridePendingTransition(0, 0);
    }
}
```
接下来，在 CommentsActivity 实现扩展动画的背景。 为此，我们可以使用简单的 Scale 动画(我们没有任何可见的内容在那一刻，所以没有人会知道我们正在拉伸背景而不是扩展它)。 不要忘记使用 setPivotY () 方法来设置正确的初始位置。

```java
public class CommentsActivity extends ActionBarActivity {
    public static final String ARG_DRAWING_START_LOCATION = "arg_drawing_start_location";

    @InjectView(R.id.toolbar)
    Toolbar toolbar;
    @InjectView(R.id.contentRoot)
    LinearLayout contentRoot;
    @InjectView(R.id.rvComments)
    RecyclerView rvComments;
    @InjectView(R.id.llAddComment)
    LinearLayout llAddComment;

    private CommentsAdapter commentsAdapter;
    private int drawingStartLocation;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //...
        
        drawingStartLocation = getIntent().getIntExtra(ARG_DRAWING_START_LOCATION, 0);
        if (savedInstanceState == null) {
            contentRoot.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                @Override
                public boolean onPreDraw() {
                    contentRoot.getViewTreeObserver().removeOnPreDrawListener(this);
                    startIntroAnimation();
                    return true;
                }
            });
        }
    }

    //...

    private void startIntroAnimation() {
        contentRoot.setScaleY(0.1f);
        contentRoot.setPivotY(drawingStartLocation);
        llAddComment.setTranslationY(100);

        contentRoot.animate()
                .scaleY(1)
                .setDuration(200)
                .setInterpolator(new AccelerateInterpolator())
                .setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationEnd(Animator animation) {
                        animateContent();
                    }
                })
                .start();
    }

    private void animateContent() {
        commentsAdapter.updateItems();
        llAddComment.animate().translationY(0)
                .setInterpolator(new DecelerateInterpolator())
                .setDuration(200)
                .start();
    }

    //...

}
```
如上所示，在我们打开 CommentsActivity 之后，我们的动画将会启动一次。由于 PreDrawListener ，动画将在测量树的所有视图并给出框架的时候执行，但是绘图操作尚未开始。

在上面的代码中，我们两者兼有，扩展背景和显示内容动画实现。 这就是它在实践中的样子:
![|center](http://frogermcs.github.io/images/3/expand_animation.gif)

还是少了点什么，对吧？

现在我们必须为评论列表中的每个元素准备动画。 这很简单，但是我们必须记住一些重要的事情:

- 每个项目的动画应该稍微延迟一点。否则，所有动画将同时执行，用户将看到整个内容的“一个”动画，而不是每个项目的动画。
- 适配器应该有可能锁定动画，因为我们不会在用户滚动内容时动画项目。
- 我们还必须提供临时解锁的方法，并为单个项目执行动画（即添加新评论）

现在评论适配器可以如下所示：
```java
public class CommentsAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private Context context;
    private int itemsCount = 0;
    private int lastAnimatedPosition = -1;
    private int avatarSize;

    private boolean animationsLocked = false;
    private boolean delayEnterAnimation = true;

    public CommentsAdapter(Context context) {
        this.context = context;
        avatarSize = context.getResources().getDimensionPixelSize(R.dimen.btn_fab_size);
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.item_comment, parent, false);
        return new CommentViewHolder(view);
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int position) {
        runEnterAnimation(viewHolder.itemView, position);
        CommentViewHolder holder = (CommentViewHolder) viewHolder;
        switch (position % 3) {
            case 0:
                holder.tvComment.setText("Lorem ipsum dolor sit amet, consectetur adipisicing elit.");
                break;
            case 1:
                holder.tvComment.setText("Cupcake ipsum dolor sit amet bear claw.");
                break;
            case 2:
                holder.tvComment.setText("Cupcake ipsum dolor sit. Amet gingerbread cupcake. Gummies ice cream dessert icing marzipan apple pie dessert sugar plum.");
                break;
        }

        Picasso.with(context)
                .load(R.drawable.ic_launcher)
                .centerCrop()
                .resize(avatarSize, avatarSize)
                .transform(new RoundedTransformation())
                .into(holder.ivUserAvatar);
    }

    private void runEnterAnimation(View view, int position) {
        if (animationsLocked) return;

        if (position > lastAnimatedPosition) {
            lastAnimatedPosition = position;
            view.setTranslationY(100);
            view.setAlpha(0.f);
            view.animate()
                    .translationY(0).alpha(1.f)
                    .setStartDelay(delayEnterAnimation ? 20 * (position) : 0)
                    .setInterpolator(new DecelerateInterpolator(2.f))
                    .setDuration(300)
                    .setListener(new AnimatorListenerAdapter() {
                        @Override
                        public void onAnimationEnd(Animator animation) {
                            animationsLocked = true;
                        }
                    })
                    .start();
        }
    }

    @Override
    public int getItemCount() {
        return itemsCount;
    }

    public void updateItems() {
        itemsCount = 10;
        notifyDataSetChanged();
    }

    public void addItem() {
        itemsCount++;
        notifyItemInserted(itemsCount - 1);
    }

    public void setAnimationsLocked(boolean animationsLocked) {
        this.animationsLocked = animationsLocked;
    }

    public void setDelayEnterAnimation(boolean delayEnterAnimation) {
        this.delayEnterAnimation = delayEnterAnimation;
    }

    public static class CommentViewHolder extends RecyclerView.ViewHolder {
        @InjectView(R.id.ivUserAvatar)
        ImageView ivUserAvatar;
        @InjectView(R.id.tvComment)
        TextView tvComment;

        public CommentViewHolder(View view) {
            super(view);
            ButterKnife.inject(this, view);
        }
    }
}
```
为了显示头像我们使用 [Picasso](http://square.github.io/picasso/) 库的  [CircleTransformation](https://gist.github.com/julianshen/5829333) 。 感谢 RecyclerView 和他的适配器，我们可以使用为新插入的项目执行默认动画的 notifyItemInserted () 方法（第80行）。 其余的代码很简单。

这是如何在评论动态类中使用的：
```java
public class CommentsActivity extends ActionBarActivity {

    //...

    private void setupComments() {

        //...

        rvComments.setOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                if (newState == RecyclerView.SCROLL_STATE_DRAGGING) {
                    commentsAdapter.setAnimationsLocked(true);
                }
            }
        });
    }

    @OnClick(R.id.btnSendComment)
    public void onSendCommentClick() {
        commentsAdapter.addItem();
        commentsAdapter.setAnimationsLocked(false);
        commentsAdapter.setDelayEnterAnimation(false);
        rvComments.smoothScrollBy(0, rvComments.getChildAt(0).getHeight() * commentsAdapter.getItemCount());
    }
}
```
当用户开始拖动 RecyclerView 时，项目动画会被阻塞（当用户添加一个新条目时，它们会暂时解锁）。

就这样。进入过渡评论活动完成。

## 退出过渡
我们必须执行的最后一件事是退出过渡。 基于概念视频，没有什么特别的事情要做。 我们必须创建过渡动画，将我们的 Acitivty（底部）滑出来。 我们必须记住，我们不能移动我们的工具栏。 这就是为什么我们使用 overridePendingTransition（0，0）; 再次动画背景视图：

```java
public class CommentsActivity extends ActionBarActivity {
    
    //...

    @Override
    public void onBackPressed() {
        contentRoot.animate()
                .translationY(Utils.getScreenHeight(this))
                .setDuration(200)
                .setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationEnd(Animator animation) {
                        CommentsActivity.super.onBackPressed();
                        overridePendingTransition(0, 0);
                    }
                })
                .start();
    }

}
```
就这样！我们刚刚完成 InstaMaterial 概念实施的第二步。在下一篇文章中，我想把重点放在我们目前错过的细节上。

## 示例代码 
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。




