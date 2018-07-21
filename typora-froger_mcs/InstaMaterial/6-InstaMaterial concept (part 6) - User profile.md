# InstaMaterial 概念(第6部分) - 用户资料

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 6) - User profile](https://mirekstanek.online/instamaterial-concept-part-6-user-profile/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的一部分，展示了使用 [Material Design 概念的 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q)的 Android 实现。 今天我们会关注一些我们之前跳过的细节。 这意味着我们仍然在9秒到13秒的时间内从概念视频。 

这是今天发布的最终效果（针对 v21 和 Pre_21 版本）：

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>

## 概论
老实说，如果你仔细阅读我以前的帖子，并试图实现描述的效果和意见，今天你不会学到任何新东西。 即使最后的效果看起来非常复杂，但几乎每一个使用过的技术或工具都被描述过。 其实这是一个好消息 - 安卓平台上不同解决方案的数量是有限的。 但是，使用它们的方式仅受限于你的想象力。😄

## 准备
一如既往，让我们开始添加新的元素。 我们必须用 Toolbar，RecyclerView 和 FloatingActionButton 创建新的 UserProfileActivity。 由于自定义过渡，它应该使用我们在 CommentsActivity 中使用的相同样式。此外，我们还需要在 FeedAdapter 中添加 onClick 侦听器到配置照片。顺便说一下，我通过创建 BaseActivity 来进行了小的重构。 下面是 commit 列表:

- [Add UserProfileActivity](https://github.com/frogermcs/InstaMaterial/commit/97dc05bc340709f0995052ed98a4693cb634f96d)
- [FeedAdapter update](FeedAdapter update)
- [code refactoring](https://github.com/frogermcs/InstaMaterial/commit/aa89624a8404532f0fd71993914a7bb0ade2316b)

## UserProfileActivity 圆形显示过渡
![|center](http://frogermcs.github.io/images/7/circural_reveal.gif)
首先让我们从实现用户资料的过渡效果开始。正如你可能注意到的，Material Design 引入了花哨的圆形动画，主要是为了显示效果。不幸的是，对于我们开发者来说，它在设计指南中看起来不错，但是我们仍然没有任何好的工具在实际应用中实现这一点。 让我们看看我们现在所拥有的可能性:

- ViewAnimationUtils.createCircularReveal ()
  这是实现圆形显示动画的最简单的方法。 主要的问题是，它只适用于 v21，现在没有 pre-21 版本的兼容性库。 像水波纹效果一样，它使用渲染线程，在旧系统版本上不可用。
- CircularReveal 库有一个开源项目，它为 pre-21（> = 2.3）提供 ViewAnimationUtils.createCircularReveal ()。它仍然是新鲜的，但看起来很有希望。 它提供了两个 ViewGroups：RevealFrameLayout 和RevealLinearLayout，可以动画。
- 也许我们不需要这个圆形？有时我们没有太多的时间去寻找理想的解决方案。 如果我们只想要快速原型显示效果，也许方形动画可以吗？ 我们可以在30秒内通过 ViewPropertyAnimator 来实现它。 怎么样？ 只需动画缩放 X 和 Y，并通过 setPivotX / Y 设置起始点，如下所示：

![|center](http://frogermcs.github.io/images/7/scale_animation.gif)

- 使用 shaders 我最喜欢的方法，特别是当我们有更复杂的视图动画的时候。 在开始的时候，Shaders 可能有点难以理解，但它却是最有效的方式之一。 在 Romain 的 Guy 博客上，你可以找到用 shaders 制作的动画的[配方](http://www.curious-creature.com/2012/12/13/android-recipe-2-fun-with-shaders/)。
- 用你自己的方式去做如果上面提到的方法对你都不起作用，只需要考虑需求并尝试用自己的方式来实现它。 有时最简单的解决方案可以满足你的期望，就像在我们的例子中，最终结果的性能没有差别。 我在我们的应用中提到过这个方法吗？

## 自定义 RevealBackgroundView
我们将实现我们可以执行圆形显示动画的自定义视图。要求很简单：
- 圆形显示
- 显示动画应该从点击处开始
- 我们想要处理动画状态（没有开始，开始，完成）
- 我们也可以手动选择完成状态

在我们的自定义视图中，我们使用[前一篇文章](http://frogermcs.github.io/InstaMaterial-concept-part-5-like_action_effects/)中介绍的 ObjectAnimator。 它通过设置 currentRadius 值来动画圈大小。 对于绘制圆，我们重写 onDraw（）方法，如果视图的状态完成，它会在给定位置或全屏矩形上绘制给定半径的圆。 这里是 RevealBackgroundView 的源代码：

```java
public class RevealBackgroundView extends View {
    public static final int STATE_NOT_STARTED = 0;
    public static final int STATE_FILL_STARTED = 1;
    public static final int STATE_FINISHED = 2;

    private static final Interpolator INTERPOLATOR = new AccelerateInterpolator();
    private static final int FILL_TIME = 400;

    private int state = STATE_NOT_STARTED;

    private Paint fillPaint;
    private int currentRadius;
    ObjectAnimator revealAnimator;

    private int startLocationX;
    private int startLocationY;


    private OnStateChangeListener onStateChangeListener;

    public RevealBackgroundView(Context context) {
        super(context);
        init();
    }

    public RevealBackgroundView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public RevealBackgroundView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public RevealBackgroundView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        init();
    }

    private void init() {
        fillPaint = new Paint();
        fillPaint.setStyle(Paint.Style.FILL);
        fillPaint.setColor(Color.WHITE);
    }

    public void startFromLocation(int[] tapLocationOnScreen) {
        changeState(STATE_FILL_STARTED);
        startLocationX = tapLocationOnScreen[0];
        startLocationY = tapLocationOnScreen[1];
        revealAnimator = ObjectAnimator.ofInt(this, "currentRadius", 0, getWidth() + getHeight()).setDuration(FILL_TIME);
        revealAnimator.setInterpolator(INTERPOLATOR);
        revealAnimator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                changeState(STATE_FINISHED);
            }
        });
        revealAnimator.start();
    }

    public void setToFinishedFrame() {
        changeState(STATE_FINISHED);
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (state == STATE_FINISHED) {
            canvas.drawRect(0, 0, getWidth(), getHeight(), fillPaint);
        } else {
            canvas.drawCircle(startLocationX, startLocationY, currentRadius, fillPaint);
        }
    }

    private void changeState(int state) {
        if (this.state == state) {
            return;
        }

        this.state = state;
        if (onStateChangeListener != null) {
            onStateChangeListener.onStateChange(state);
        }
    }

    public void setOnStateChangeListener(OnStateChangeListener onStateChangeListener) {
        this.onStateChangeListener = onStateChangeListener;
    }

    public void setCurrentRadius(int radius) {
        this.currentRadius = radius;
        invalidate();
    }

    public static interface OnStateChangeListener {
        void onStateChange(int state);
    }
}
```
现在让我们在 MainActivity 中添加用户配置文件的开始方法：
```java
@Override
public void onProfileClick(View v) {
    int[] startingLocation = new int[2];
    v.getLocationOnScreen(startingLocation);
    startingLocation[0] += v.getWidth() / 2;
    UserProfileActivity.startUserProfileFromLocation(startingLocation, this);
    overridePendingTransition(0, 0);
}
```
现在，当我们有了开始的位置，只需要在 UserProfileActivity 中使用它，并在适当的时候动画背景（感谢 onPreDrawListener）。 UserProfileActivity 的最终源代码如下所示：
```java
public class UserProfileActivity extends BaseActivity implements RevealBackgroundView.OnStateChangeListener {
    public static final String ARG_REVEAL_START_LOCATION = "reveal_start_location";

    @InjectView(R.id.vRevealBackground)
    RevealBackgroundView vRevealBackground;
    @InjectView(R.id.rvUserProfile)
    RecyclerView rvUserProfile;

    private UserProfileAdapter userPhotosAdapter;

    public static void startUserProfileFromLocation(int[] startingLocation, Activity startingActivity) {
        Intent intent = new Intent(startingActivity, UserProfileActivity.class);
        intent.putExtra(ARG_REVEAL_START_LOCATION, startingLocation);
        startingActivity.startActivity(intent);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_user_profile);
        setupUserProfileGrid();
        setupRevealBackground(savedInstanceState);
    }

    private void setupRevealBackground(Bundle savedInstanceState) {
        vRevealBackground.setOnStateChangeListener(this);
        if (savedInstanceState == null) {
            final int[] startingLocation = getIntent().getIntArrayExtra(ARG_REVEAL_START_LOCATION);
            vRevealBackground.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                @Override
                public boolean onPreDraw() {
                    vRevealBackground.getViewTreeObserver().removeOnPreDrawListener(this);
                    vRevealBackground.startFromLocation(startingLocation);
                    return false;
                }
            });
        } else {
            userPhotosAdapter.setLockedAnimations(true);
            vRevealBackground.setToFinishedFrame();
        }
    }

    private void setupUserProfileGrid() {
        final StaggeredGridLayoutManager layoutManager = new StaggeredGridLayoutManager(3, StaggeredGridLayoutManager.VERTICAL);
        rvUserProfile.setLayoutManager(layoutManager);
        rvUserProfile.setOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                userPhotosAdapter.setLockedAnimations(true);
            }
        });
    }

    @Override
    public void onStateChange(int state) {
        if (RevealBackgroundView.STATE_FINISHED == state) {
            rvUserProfile.setVisibility(View.VISIBLE);
            userPhotosAdapter = new UserProfileAdapter(this);
            rvUserProfile.setAdapter(userPhotosAdapter);
        } else {
            rvUserProfile.setVisibility(View.INVISIBLE);
        }
    }
}
```
我们这里有什么？如果活动正在开始，那么显示动画运行。  否则，如果我们恢复活动，则通过 vRevealBackground.setToFinishedFrame () 将背景设置为完成状态。带有用户配置文件的 RecyclerView 在显示动画正在运行时被隐藏，并在状态更改为 STATE_FINISHED 时显示（此操作在 onStateChange () 方法中完成）。 userPhotosAdapter.setLockedAnimations（true）在用户开始滚动或活动正在恢复时禁用配置文件动画。

## 用户资料布局文件
[概念视频](https://www.youtube.com/watch?v=ojwdmgmdR_Q)中显示的用户资料布局文件可以分为三个主要部分：
- 包含所有用户数据和统计信息的用户资料头，
- 用户资料选项
- 用户照片

首先让我们准备他们每个人的布局。其实没什么特别的，这里你可以看到截图。如果你想，只要试着自己复制这个，或者检查这个[提交](https://github.com/frogermcs/InstaMaterial/commit/7dc54888b37ae29651b9f242ca15c62ed6ccdb28)的源代码。

**用户资料头**
![|center](http://frogermcs.github.io/images/7/profile_header.png)
**用户资料选项**
![|center](http://frogermcs.github.io/images/7/profile_options.png)
**用户照片**
![|center](http://frogermcs.github.io/images/7/profile_photo.png)

接下来我们可以开始实现 UserProfileAdapter。 用户资料布局将由 RecyclerView 呈示。 前两个元素（标题和选项）应该占据整个宽度。 照片应该显示在它们的下方，最多3个元素。 这就是为什么我们在活动中使用 StaggeredGridLayoutManager 的原因。 它可以定义多少个元素应该包含一行（对于垂直滚动）或一列（在水平滚动中）。 还要感谢这个管理器，我们可以定义每个 RecyclerView 项目应该占据多少位置。

我们开始构建我们的适配器。首先，我们必须定义项目类型：
```java
public static final int TYPE_PROFILE_HEADER = 0;
public static final int TYPE_PROFILE_OPTIONS = 1;
public static final int TYPE_PHOTO = 2;

@Override
public int getItemViewType(int position) {
    if (position == 0) {
        return TYPE_PROFILE_HEADER;
    } else if (position == 1) {
        return TYPE_PROFILE_OPTIONS;
    } else {
        return TYPE_PHOTO;
    }
}
```
每种类型应该有他自己的 ViewHolder：
```java
static class ProfileHeaderViewHolder extends RecyclerView.ViewHolder {
    @InjectView(R.id.ivUserProfilePhoto)
    ImageView ivUserProfilePhoto;
    @InjectView(R.id.vUserDetails)
    View vUserDetails;
    @InjectView(R.id.btnFollow)
    Button btnFollow;
    @InjectView(R.id.vUserStats)
    View vUserStats;
    @InjectView(R.id.vUserProfileRoot)
    View vUserProfileRoot;

    public ProfileHeaderViewHolder(View view) {
        super(view);
        ButterKnife.inject(this, view);
    }
}

static class ProfileOptionsViewHolder extends RecyclerView.ViewHolder {
    @InjectView(R.id.btnGrid)
    ImageButton btnGrid;
    @InjectView(R.id.btnList)
    ImageButton btnList;
    @InjectView(R.id.btnMap)
    ImageButton btnMap;
    @InjectView(R.id.btnTagged)
    ImageButton btnComments;
    @InjectView(R.id.vUnderline)
    View vUnderline;
    @InjectView(R.id.vButtons)
    View vButtons;

    public ProfileOptionsViewHolder(View view) {
        super(view);
        ButterKnife.inject(this, view);
    }
}

static class PhotoViewHolder extends RecyclerView.ViewHolder {
    @InjectView(R.id.flRoot)
    FrameLayout flRoot;
    @InjectView(R.id.ivPhoto)
    ImageView ivPhoto;

    public PhotoViewHolder(View view) {
        super(view);
        ButterKnife.inject(this, view);
    }
}
```
我们在 onCreateViewHolder () 方法中使用它们，在这里我们必须附加我们的视图并为我们的布局管理器设置它们：
```java
@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (TYPE_PROFILE_HEADER == viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.view_user_profile_header, parent, false);
        StaggeredGridLayoutManager.LayoutParams layoutParams = (StaggeredGridLayoutManager.LayoutParams) view.getLayoutParams();
        layoutParams.setFullSpan(true);
        view.setLayoutParams(layoutParams);
        return new ProfileHeaderViewHolder(view);
    } else if (TYPE_PROFILE_OPTIONS == viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.view_user_profile_options, parent, false);
        StaggeredGridLayoutManager.LayoutParams layoutParams = (StaggeredGridLayoutManager.LayoutParams) view.getLayoutParams();
        layoutParams.setFullSpan(true);
        view.setLayoutParams(layoutParams);
        return new ProfileOptionsViewHolder(view);
    } else if (TYPE_PHOTO == viewType) {
        final View view = LayoutInflater.from(context).inflate(R.layout.item_photo, parent, false);
        StaggeredGridLayoutManager.LayoutParams layoutParams = (StaggeredGridLayoutManager.LayoutParams) view.getLayoutParams();
        layoutParams.height = cellSize;
        layoutParams.width = cellSize;
        layoutParams.setFullSpan(false);
        view.setLayoutParams(layoutParams);
        return new PhotoViewHolder(view);
    }

    return null;
}
```
在第6,12和20行中的 setFullSpan () ，我们可以指出哪些视图应该占据整行宽度。 我们应该做的最后一件事（动画之前）绑定 ViewHolders：
```java
@Override
public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
    int viewType = getItemViewType(position);
    if (TYPE_PROFILE_HEADER == viewType) {
        bindProfileHeader((ProfileHeaderViewHolder) holder);
    } else if (TYPE_PROFILE_OPTIONS == viewType) {
        bindProfileOptions((ProfileOptionsViewHolder) holder);
    } else if (TYPE_PHOTO == viewType) {
        bindPhoto((PhotoViewHolder) holder, position);
    }
}
```
我们不会将注意力集中在绑定方法上，但有一件事值得一提。对于照片加载，我们使用 [Picasso](http://frogermcs.github.io/InstaMaterial-concept-part-6-user-profile/square.github.io/picasso/) 库。它有一个非常强大的功能。我们可以在后台线程中将图像转填充到 ImageView。

## 圆形的用户照片
正如您可能注意到的，用户个人资料照片具有圆形的白色轮廓。这个效果可以通过 Picasso 的图像转换简单地实现，只需少量的着色器就可以实现。 我们所要做的就是实现修改给定位图的 Transformation 接口。 对于圆形形状，我们可以使用着色器（查看另一个[Romain Guy 的配方](http://www.curious-creature.com/2012/12/11/android-recipe-1-image-with-rounded-corners/)，了解着色器使用的简单示例）。
我们的代码非常简单：
```java
public class CircleTransformation implements Transformation {

    private static final int STROKE_WIDTH = 6;

    @Override
    public Bitmap transform(Bitmap source) {
        int size = Math.min(source.getWidth(), source.getHeight());

        int x = (source.getWidth() - size) / 2;
        int y = (source.getHeight() - size) / 2;

        Bitmap squaredBitmap = Bitmap.createBitmap(source, x, y, size, size);
        if (squaredBitmap != source) {
            source.recycle();
        }

        Bitmap bitmap = Bitmap.createBitmap(size, size, source.getConfig());

        Canvas canvas = new Canvas(bitmap);

        Paint avatarPaint = new Paint();
        BitmapShader shader = new BitmapShader(squaredBitmap, BitmapShader.TileMode.CLAMP, BitmapShader.TileMode.CLAMP);
        avatarPaint.setShader(shader);

        Paint outlinePaint = new Paint();
        outlinePaint.setColor(Color.WHITE);
        outlinePaint.setStyle(Paint.Style.STROKE);
        outlinePaint.setStrokeWidth(STROKE_WIDTH);
        outlinePaint.setAntiAlias(true);

        float r = size / 2f;
        canvas.drawCircle(r, r, r, avatarPaint);
        canvas.drawCircle(r, r, r - STROKE_WIDTH / 2, outlinePaint);

        squaredBitmap.recycle();
        return bitmap;
    }

    @Override
    public String key() {
        return "circleTransformation()";
    }
}
```
我们的 CircularTransformation 绘制了白色轮廓的圆形位图。 至此描述的 UserProfileAdapter 完整的源代码可以在[这里](https://github.com/frogermcs/InstaMaterial/commit/fb2d3689d2c6b6f28e4ad05908a96d4661872604)找到。这是当前实施的最终效果：
![|center](http://frogermcs.github.io/images/7/user_profile.png)

## 用户资料介绍动画
![|center](http://frogermcs.github.io/images/7/profile_animation.gif)
最后的事情 - 介绍动画。 在[概念视频](https://www.youtube.com/watch?v=ojwdmgmdR_Q)看起来很复杂，但我认为这是今天的帖子最简单的部分。 实际上，我们所要做的就是使用 ViewPropertyAnimator 在正确的时间和正确的时间和效果动画每个视图。

用户资料头和用户资料选项动画应该立即开始，所以最好的时刻是从 PreDrawListener 回调。动画非常简单 - 他们只是在视图中更改 translation 或 alpha。一切都在约20行代码中完成：

```java
private void animateUserProfileHeader(ProfileHeaderViewHolder viewHolder) {
    if (!lockedAnimations) {
        profileHeaderAnimationStartTime = System.currentTimeMillis();

        viewHolder.vUserProfileRoot.setTranslationY(-viewHolder.vUserProfileRoot.getHeight());
        viewHolder.ivUserProfilePhoto.setTranslationY(-viewHolder.ivUserProfilePhoto.getHeight());
        viewHolder.vUserDetails.setTranslationY(-viewHolder.vUserDetails.getHeight());
        viewHolder.vUserStats.setAlpha(0);

        viewHolder.vUserProfileRoot.animate().translationY(0).setDuration(300).setInterpolator(INTERPOLATOR);
        viewHolder.ivUserProfilePhoto.animate().translationY(0).setDuration(300).setStartDelay(100).setInterpolator(INTERPOLATOR);
        viewHolder.vUserDetails.animate().translationY(0).setDuration(300).setStartDelay(200).setInterpolator(INTERPOLATOR);
        viewHolder.vUserStats.animate().alpha(1).setDuration(200).setStartDelay(400).setInterpolator(INTERPOLATOR).start();
    }
}

private void animateUserProfileOptions(ProfileOptionsViewHolder viewHolder) {
    if (!lockedAnimations) {
        viewHolder.vButtons.setTranslationY(-viewHolder.vButtons.getHeight());
        viewHolder.vUnderline.setScaleX(0);

        viewHolder.vButtons.animate().translationY(0).setDuration(300).setStartDelay(USER_OPTIONS_ANIMATION_DELAY).setInterpolator(INTERPOLATOR);
        viewHolder.vUnderline.animate().scaleX(1).setDuration(200).setStartDelay(USER_OPTIONS_ANIMATION_DELAY + 300).setInterpolator(INTERPOLATOR).start();
    }
}
```
正如我所说 - 最重要的事情是时机和开始的正确时刻。我这种情况是通过试错法来实现的。 😄

有点不同的是照片动画。 我们不确定从 Internet 上加载它们需要多少时间，因此运行这些动画的最佳位置是显示照片的时刻。幸运的是，Picasso 有一个简单的 onSuccess () 和 onError () 方法回调这将用于启动动画。
另外我们还要考虑第二个选项 - 在配置文件动画完成之前，照片加载速度更快（即从缓存中）。 在这种情况下，我们所要做的只是一个简单的逻辑来计算适当的延迟值。
最终的代码如下所示：
```java
private void bindPhoto(final PhotoViewHolder holder, int position) {
    Picasso.with(context)
            .load(photos.get(position - MIN_ITEMS_COUNT))
            .resize(cellSize, cellSize)
            .centerCrop()
            .into(holder.ivPhoto, new Callback() {
                @Override
                public void onSuccess() {
                    animatePhoto(holder);
                }

                @Override
                public void onError() {

                }
            });
    if (lastAnimatedItem < position) lastAnimatedItem = position;
}

private void animatePhoto(PhotoViewHolder viewHolder) {
    if (!lockedAnimations) {
        if (lastAnimatedItem == viewHolder.getPosition()) {
            setLockedAnimations(true);
        }

        long animationDelay = profileHeaderAnimationStartTime + MAX_PHOTO_ANIMATION_DELAY - System.currentTimeMillis();
        if (profileHeaderAnimationStartTime == 0) {
            animationDelay = viewHolder.getPosition() * 30 + MAX_PHOTO_ANIMATION_DELAY;
        } else if (animationDelay < 0) {
            animationDelay = viewHolder.getPosition() * 30;
        } else {
            animationDelay += viewHolder.getPosition() * 30;
        }

        viewHolder.flRoot.setScaleY(0);
        viewHolder.flRoot.setScaleX(0);
        viewHolder.flRoot.animate()
                .scaleY(1)
                .scaleX(1)
                .setDuration(200)
                .setInterpolator(INTERPOLATOR)
                .setStartDelay(animationDelay)
                .start();
    }
}
```
完整的提交所有必需的更改可在[此处获得](https://github.com/frogermcs/InstaMaterial/commit/d352d689e6ba1666b1042c52434638b0bf7654d2)。 这就是今天。我们刚刚完成了打开过渡和用户资料布局的所有动画。谢谢阅读！ 😄

## 示例代码
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。


