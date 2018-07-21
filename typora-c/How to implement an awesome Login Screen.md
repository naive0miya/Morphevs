# 如何实现一个很棒的登录屏幕？

> 原文 (Medium)：[How to implement an awesome Login Screen?](https://medium.com/@vpaliy/do-you-dare-me-to-implement-this-login-screen-bf29b72d9e39)
>
> 作者：[Vasyl Paliy](https://medium.com/@vpaliy?source=post_header_lockup)

[TOC]

今天的大多数应用程序都有授权活动，可以通过 Facebook，Google，Twitter 进行授权活动，也可以通过电子邮件和密码老式的方式进行。 它可能不是您的应用程序的第一个屏幕（一些应用程序有欢迎屏幕，如应用程序的介绍）。

然而，这里的主要观点是，你不想让授权过程比应该的更痛苦，对吧？ 你想让用户对你的应用程序留下更好的印象ーー给他们带来愉悦和兴奋是你的责任。 

## Alright, where are you going with this?

那么，在我偶然发现这个伟大的设计理念之前，我曾经为我的一个 Android 应用程序（它必须是完美的）设计了一个登录/注册屏幕，

![](https://cdn-images-1.medium.com/max/800/1*PN4k4tK_iDOaWeq7E1zteA.gif)

> 原创概念作者 ： [Yaroslav Zubko](https://www.uplabs.com/posts/7-2-log-in-sign-up).

所以，我决定实施这个概念，并将在整篇文章中从头开始提供一个全面的指导。 如果你是这种喜欢阅读代码的人，你可能想跳到文章的末尾。如果不是，我们可以开始吗？ . 

## Structure of the application

除了进行这种精彩的交互之外，我还想使代码可重用，并将主要的关注点分离出来。 也就是说，我的登录屏幕不应该知道关于注册的屏幕，反之亦然。 

现在，我们再来看看这个概念，并定义这个概念。开始了：

- 背景图。
- 注册屏幕。 
- 登录屏幕。 
- 在每个屏幕的边缘的差距。 
- 输入字段。 
- 标志，底部按钮和登录/注册标签。

我认为，从 Android 视图的角度来解释这些组件并不是一件大事。 例如，每个授权屏幕都应该是一个片段，你可以使用 ViewPager 在它们之间切换。 背景、 logo 和按钮可以是简单的 ImageView 对象。 

所以，应用程序将具有以下结构：

- Activity with a ViewPager
- Adapter
- Login Fragment
- Sign up Fragment

### Wait a minute, what about the gap?

这就是为什么我选择使用一个 ViewPager 的原因之一，它的适配器有 PagerAdapter.getPageWidth (int position)方法，该方法将给定页面的比例宽度作为视页宽度的百分比从 (0.f-1.f]  返回给定页面的比例宽度。 因此，你可以调整一些特定页面的宽度以适应您的需求，下一页 / 上一页将被移到左 / 右，以填补当前页所留下的空白。 

结果如下：

![](https://cdn-images-1.medium.com/max/800/1*WhUSUtgWIVY0aM4FMT2WPw.gif)

但是你怎么知道缝隙的宽度呢？ 那么，那些应该垂直放置在空白中间的文本呢？ 

不用担心，我们将创建一个在 PagerAdapter 构造函数中计算的因子，然后在 PagerAdapter.getpagewidth (int position)方法中使用该因子。 

```java
public static class Adapter extends FragmentStatePagerAdapter{
  //our factor value
  private float factor;
  
  public Adapter(FragmentManager manager,final ViewPager pager){
    super(manager);
    final float textSize=pager.getResources().getDimension(R.dimen.folded_size);
    final float textPadding=pager.getResources().getDimension(R.dimen.folded_label_padding);
    factor=1-(textSize+textPadding)/(pager.getWidth());
  }
  
  @Override
  public float getPageWidth(int position) {
    return factor;
  }
}
```

让我们来分析一下: 

- TextSize 是文本被折叠时的大小（垂直文本）。
- 这是一个从左到右的简单填充。
- 我们添加大小和填充，除以屏幕的宽度，从一个减去所有的东西。 现在，如果我们把页面的宽度乘以，我们应该可以剪辑当前页面。 参考上面的结果 

## AuthorizationActivity

好了，是时候谈谈这件事的重点了！就是我们的活动:)

### First things first — the XML file:

```xml
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.vpaliy.loginconcept.LoginActivity">

    <ImageView
        android:id="@+id/scrolling_background"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:scaleType="centerCrop"
        tools:src="@drawable/busy"
        android:scaleX="@dimen/start_scale"
        android:scaleY="@dimen/start_scale"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.0" />

    <com.vpaliy.loginconcept.AnimatedViewPager
        android:id="@+id/pager"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:background="@color/color_log_in"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintHorizontal_bias="1.0"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.0" />

    <ImageView
        android:src="@drawable/facebook"
        android:layout_width="@dimen/option_size"
        android:layout_height="@dimen/option_size"
        android:id="@+id/first"
        app:layout_constraintRight_toLeftOf="@+id/second"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.95" />

    <ImageView
        android:src="@drawable/linkedin"
        android:layout_width="@dimen/option_size"
        android:layout_height="@dimen/option_size"
        android:id="@+id/second"
        app:layout_constraintLeft_toRightOf="@+id/first"
        app:layout_constraintRight_toLeftOf="@+id/last"
        app:layout_constraintTop_toTopOf="@+id/first"
        app:layout_constraintBottom_toBottomOf="@+id/first"
        app:layout_constraintVertical_bias="0.0" />

    <ImageView
        android:src="@drawable/twitter"
        android:id="@+id/last"
        android:layout_width="@dimen/option_size"
        android:layout_height="@dimen/option_size"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintLeft_toRightOf="@+id/second"
        app:layout_constraintBottom_toBottomOf="@+id/second"
        app:layout_constraintTop_toTopOf="@+id/second"
        app:layout_constraintVertical_bias="1.0" />

    <android.support.constraint.Guideline
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/guideline"
        app:layout_constraintGuide_begin="@dimen/guideline_margin"
        android:orientation="horizontal" />

    <ImageView
        android:id="@+id/logo"
        android:focusable="true"
        android:src="@drawable/log"
        android:focusableInTouchMode="true"
        app:layout_constraintTop_toTopOf="@+id/guideline"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_width="@dimen/logo_size"
        android:layout_height="@dimen/logo_size"/>
</android.support.constraint.ConstraintLayout>
```

以下是我们得到的：

![](https://cdn-images-1.medium.com/max/800/1*O9BLNSbcsfcMSvzERd9frA.png)

所以，我使用 ConstraintLayout 作为父视图，它允许您在平面层次结构中构建复杂的响应式布局。 Android 中的一个常见建议是避免布局内的视图的深层次结构，因为它会损害您的 UI 在屏幕上绘制的性能和时间。

我想你已经熟悉了 ConstraintLayout 以及它是如何工作的。如果你不是，我强烈推荐阅读[这个](https://developer.android.com/training/constraint-layout/index.html)。

Whoa there..What the heck is AnimatedViewPager?

从本质上讲，这是一个自定义视图，它不会响应任何触摸事件，并且会改变滑动的持续时间。 你可以看看[这里的代码](https://gist.github.com/vpaliyX/f05ca9dd39fedbd4ee4a3bdabca2f084)。 

Logo and buttons.

这些元素是两个屏幕（登录/注册）的共享元素，这就是为什么把它们放在一个地方是合理的。标志，以及按钮，都是简单的 ImageView 对象，它们可以从一个片段平滑地移动到另一个片段。 

### Second things second — the activity class.

让我们直接跳入代码：

```java
public class AuthActivity extends AppCompatActivity {

    @BindViews(value = {R.id.logo,R.id.first,R.id.second,R.id.last})
    protected List<ImageView> sharedElements;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        ButterKnife.bind(this);
        final AnimatedViewPager pager= ButterKnife.findById(this,R.id.pager);
        final ImageView background=ButterKnife.findById(this,R.id.scrolling_background);
        int[] screenSize=screenSize();

        sharedElements.forEach(element->{
            @ColorRes int color=element.getId()!=R.id.logo?R.color.white_transparent:R.color.color_logo_log_in;
            DrawableCompat.setTint(element.getDrawable(), ContextCompat.getColor(this,color));
        });
        //load a very big image and resize it, so it fits our needs
        Glide.with(this)
                .load(R.drawable.busy)
                .asBitmap()
                .override(screenSize[0]2,screenSize[1])
                .diskCacheStrategy(DiskCacheStrategy.RESULT)
                .into(new ImageViewTarget<Bitmap>(background) {
                    @Override
                    protected void setResource(Bitmap resource) {
                        background.setImageBitmap(resource);
                        background.scrollTo(-pager.getWidth(),0);
                        background.post(()->{
                            //we need to scroll to the very left edge of the image
                            //fire the scale animation
                            ObjectAnimator xAnimator=ObjectAnimator.ofFloat(background,View.SCALE_X,4f,background.getScaleX());
                            ObjectAnimator yAnimator=ObjectAnimator.ofFloat(background,View.SCALE_Y,4f,background.getScaleY());
                            AnimatorSet set=new AnimatorSet();
                            set.playTogether(xAnimator,yAnimator);
                            set.setDuration(getResources().getInteger(R.integer.duration));
                            set.start();
                        });
                        AuthAdapter adapter = new AuthAdapter(getSupportFragmentManager(), pager, background, sharedElements);
                        pager.setAdapter(adapter);
                    }
                });
    }

    private int[] screenSize(){
        Display display = getWindowManager().getDefaultDisplay();
        Point size = new Point();
        display.getSize(size);
        return new int[]{size.x,size.y};
    }
}
```

### What does it do?

着色

加载并设置背景图像。

为 ViewPager 创建一个适配器

### Tinting the shared elements

正如你可能已经注意到的那样，原始设计中的共享元素并不是如此不透明。 底部元素应该接近透明度，比方说＃B3FFFCFC（只是一个白色，不透明度为173）。 徽标随着它进入另一个页面而改变，而底部的元素则不会。 你可以看到它总是比当前背景颜色浅。

但是，你并不需要遵循这一点，我只是想坚持原来的设计:)

这就是着色发生的地方：

```java
sharedElements.forEach(element->{
            @ColorRes int color=element.getId()!=R.id.logo?R.color.white_transparent:R.color.color_logo_log_in;
            DrawableCompat.setTint(element.getDrawable(), ContextCompat.getColor(this,color));
  });
```

注意！请记住，我将改变我们的适配器的标志色调。

### Background Image.

你需要将宽度扩展到背景图像，所以它和你的两个页面一样宽。这是一个很好的选择一个大的图像（大约1,900×1,200 JPEG，24位颜色），所以可以绘制，而不会丢失任何质量。

这非常简单 - 只需获取屏幕宽度，乘以2，然后使用 Glide，Picasso，Fresco 或任何您喜欢的库加载图像。

```java
//load a very big image and resize it, so it fits our needs
Glide.with(this)
 .load(R.drawable.busy)
 .asBitmap()
 .override(screenSize[0]2,screenSize[1])
 .diskCacheStrategy(DiskCacheStrategy.RESULT)
 .into(new ImageViewTarget<Bitmap>(background) {
   @Override
   protected void setResource(Bitmap resource) {
      background.setImageBitmap(resource);
      background.post(()->{
        //we need to scroll to the very left edge of the image
        background.scrollTo(-background.getWidth()/2,0);
        //fire the scale animation
        ObjectAnimator xAnimator=ObjectAnimator.ofFloat(background,View.SCALE_X,4f,background.getScaleX());
        ObjectAnimator yAnimator=ObjectAnimator.ofFloat(background,View.SCALE_Y,4f,background.getScaleY());
        AnimatorSet set=new AnimatorSet();
        set.playTogether(xAnimator,yAnimator);
        set.setDuration(getResources().getInteger(R.integer.duration));
        set.start();
      });
      AuthAdapter adapter = new AuthAdapter(getSupportFragmentManager(), pager, background, sharedElements);
      pager.setAdapter(adapter);
    }
});
```

1. 加载图像后，您需要使用 ImageView.scrollTo ( ) 方法将用户焦点转移到图像的最左边。 
2. 之后你想发起一个比例尺动画，所以你会得到这个美丽的比例效果。

​       Video - [Scale Animation](https://www.youtube.com/watch?v=AlNKUm5z0xk)

3. 最后但并非最不重要的，初始化您的适配器。

## Adapter and its guts

我们需要一个控制器来了解这些碎片之间发生的事情。 这个角色的最佳人选是我们的适配器。 

事实上，我们将为这些片段设置一个基类，它可以作为一个接口，可以由适配器引用。 

这是我们的抽象类：

```java
public abstract class AuthFragment extends Fragment {
    
    protected Callback callback;

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View root=inflater.inflate(authLayout(),container,false);
        ButterKnife.bind(this,root);
        return root;
    }

    public void setCallback(@NonNull Callback callback) {
        this.callback = callback;
    }
  
    @OnClick(R.id.caption)
    public void unfold(){
        / animation code goes here 
           ...   ....
        /
        //after everything's been set up, tell the ViewPager to flip the page
        callback.show(this);
    }
    
    @LayoutRes
    public abstract int authLayout();
    public abstract void fold();
    public abstract void clearFocus();
    
    interface Callback {
        void show(AuthFragment fragment);
        void scale(boolean hasFocus);
    }
}
```

正如你所看到的，对于每种类型的动画我们都有三种方法，所以你可以在不知道谁是谁的情况下引用这两个片段。 为了说清楚，我们来看看上面的方法：

fold( ) 当您切换到当前片段的下一个/上一个页面时，会调用 fold ( )。

unfold( ) 在开关之前，展开下一个/上一个页面将调用 unfold ( )。 请记住，该方法不被适配器调用。 当发生点击事件时，此方法被调用。是的！ 当你点击那个垂直的 TextView。

clearFocus( ) 释放输入时调用 clearFocus ( )。 例如，当您输入密码或登录时，我们需要清除焦点。 在这一刻，一个规模动画也被解雇。
authLayout( ) 提供了一个资源在 Fragment.OnCreateView ( ) 方法中附加。

### So far so good, but what about this Callback interface?

我们需要提供一种能力，通知适配器任何最近发生的事件的片段。 所以，适配器只是实现了这个接口，我们可以从我们的片段中引用适配器。 简单明了。 

振作起来！这里是适配器的代码：

```java
public class AuthAdapter extends FragmentStatePagerAdapter
        implements AuthFragment.Callback{

    private final AnimatedViewPager pager;
    private final SparseArray<AuthFragment> authArray;
    private final List<ImageView> sharedElements;
    private final ImageView authBackground;
    private float factor;

    public AuthAdapter(FragmentManager manager,  AnimatedViewPager pager,
                       ImageView authBackground, List<ImageView> sharedElements){
        super(manager);
        this.authBackground=authBackground;
        this.pager=pager;
        this.authArray=new SparseArray<>(getCount());
        this.sharedElements=sharedElements;
        pager.setDuration(pager.getResources().getInteger(R.integer.duration));
        final float textSize=pager.getResources().getDimension(R.dimen.folded_size);
        final float textPadding=pager.getResources().getDimension(R.dimen.folded_label_padding);
        factor=1-(textSize+textPadding)/(pager.getWidth());
    }

    @Override
    public AuthFragment getItem(int position) {
        AuthFragment fragment=authArray.get(position);
        if(fragment==null){
            fragment=position!=1?new LogInFragment():new SignUpFragment();
            authArray.put(position,fragment);
            fragment.setCallback(this);
        }
        return fragment;
    }

    @Override
    public void show(AuthFragment fragment) {
        final int index=authArray.keyAt(authArray.indexOfValue(fragment));
        pager.setCurrentItem(index,true);
        shiftSharedElements(getPageOffsetX(fragment), index==1);
        for(int jIndex=0;jIndex<authArray.size();jIndex++){
            if(jIndex!=index){
                authArray.get(jIndex).fold();
            }
        }
    }

    private float getPageOffsetX(AuthFragment fragment){
        int pageWidth=fragment.getView().getWidth();
        return pageWidth-pageWidthfactor;
    }

    private void shiftSharedElements(float pageOffsetX, boolean forward){
        final Context context=pager.getContext();
        //since we're clipping the page, we have to adjust the shared elements
        AnimatorSet shiftAnimator=new AnimatorSet();
        for(View view:sharedElements){
            float translationX=forward?pageOffsetX:-pageOffsetX;
            float temp=view.getWidth()/3f;
            translationX-=forward?temp:-temp;
            ObjectAnimator shift=ObjectAnimator.ofFloat(view,View.TRANSLATION_X,0,translationX);
            shiftAnimator.playTogether(shift);
        }

        int color=ContextCompat.getColor(context,forward?R.color.color_logo_sign_up:R.color.color_logo_log_in);
        DrawableCompat.setTint(sharedElements.get(0).getDrawable(),color);
        //scroll the background by x
        int offset=authBackground.getWidth()/2;
        ObjectAnimator scrollAnimator=ObjectAnimator.ofInt(authBackground,"scrollX",forward?offset:-offset);
        shiftAnimator.playTogether(scrollAnimator);
        shiftAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
        shiftAnimator.setDuration(pager.getResources().getInteger(R.integer.duration)/2);
        shiftAnimator.start();
    }

    @Override
    public void scale(boolean hasFocus) {

        final float scale=hasFocus?1:1.4f;
        final float logoScale=hasFocus?0.75f:1f;
        View logo=sharedElements.get(0);

        AnimatorSet scaleAnimation=new AnimatorSet();
        scaleAnimation.playTogether(ObjectAnimator.ofFloat(logo,View.SCALE_X,logoScale));
        scaleAnimation.playTogether(ObjectAnimator.ofFloat(logo,View.SCALE_Y,logoScale));
        scaleAnimation.playTogether(ObjectAnimator.ofFloat(authBackground,View.SCALE_X,scale));
        scaleAnimation.playTogether(ObjectAnimator.ofFloat(authBackground,View.SCALE_Y,scale));
        scaleAnimation.setDuration(200);
        scaleAnimation.setInterpolator(new AccelerateDecelerateInterpolator());
        scaleAnimation.start();
    }

    @Override
    public float getPageWidth(int position) {
        return factor;
    }

    @Override
    public int getCount() {
        return 2;
    }
}
```

这无疑是这里最重要的部分。好吧，我们来看一下！

你已经看过构造函数了，对吗？我们只是简单地计算差距的一个因素，设置 ViewPager 的持续时间。而且，我们创建了一个片段数组，所以更容易从它的位置获取一个片段。

show（AuthFragment）方法负责折叠当前片段，移动共享元素，并使用 ViewPager.setCurrentItem ( )方法切换到另一个片段。 正如我之前所说的，这个方法从一个片段中被调用。
getPageOffsetX（AuthFragment）只是一个辅助方法，它返回每个片段的间隙宽度。
shiftSharedElements（float，float）因为我们剪辑片段（为了获得空隙），我们需要调整共享的元素，所以它们不会相对于片段布局来看起来弯曲。 只要创建一个简单的移动动画，与其他动画一起运行。
scale(boolean) 在文本输入丢失/获得焦点时向上/向下缩放背景图像。

所以，基本上都是关于适配器。

### Login and Sign Up screens.

好的，现在是时候设置主屏幕。 很明显，我们将为每个片段都有类似的 XML 文件。 事实上，我们在注册片段中只有一组视图（2个视图），而对于登录片段只有一个简单的 TextView。 所以，每个“重复”视图都有相同的 id 是有意义的。

据说，让我们来看看注册屏幕的XML文件：

```xml
<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/root"
    tools:background="@color/color_sign_up"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.constraint.Guideline
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/guideline"
        app:layout_constraintGuide_begin="48dp"
        android:orientation="horizontal" />

    <View
        android:id="@+id/logo"
        android:focusable="true"
        android:focusableInTouchMode="true"
        app:layout_constraintTop_toTopOf="@+id/guideline"
        android:layout_marginTop="8dp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_width="64dp"
        android:layout_height="64dp"/>

    <android.support.design.widget.TextInputLayout
        style="@style/Widget.TextInputLayout"
        android:id="@+id/email_input"
        android:layout_marginTop="48dp"
        app:layout_constraintTop_toBottomOf="@+id/logo"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent">
        <android.support.design.widget.TextInputEditText
            style="@style/Widget.TextEdit"
            android:id="@+id/email_input_edit"
            android:hint="@string/email_hint"
            android:inputType="textEmailAddress" />
    </android.support.design.widget.TextInputLayout>

    <android.support.design.widget.TextInputLayout
        style="@style/Widget.TextInputLayout"
        android:id="@+id/password_input"
        app:layout_constraintTop_toBottomOf="@+id/email_input"
        android:layout_marginTop="16dp"
        app:passwordToggleTint="@color/color_input_hint"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent">
        <android.support.design.widget.TextInputEditText
            style="@style/Widget.TextEdit"
            android:id="@+id/password_input_edit"
            android:hint="@string/password_hint"
            android:inputType="textPassword" />
    </android.support.design.widget.TextInputLayout>

    <android.support.design.widget.TextInputLayout
        style="@style/Widget.TextInputLayout"
        android:id="@+id/confirm_password"
        android:layout_marginTop="16dp"
        app:passwordToggleTint="@color/color_input_hint"
        app:layout_constraintTop_toBottomOf="@+id/password_input"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent">
        <android.support.design.widget.TextInputEditText
            style="@style/Widget.TextEdit"
            android:id="@+id/confirm_password_edit"
            android:hint="@string/confirm_hint"
            android:inputType="textPassword"/>
    </android.support.design.widget.TextInputLayout>

    <com.vpaliy.loginconcept.VerticalTextView
        android:id="@+id/caption"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:textSize="@dimen/unfolded_size"
        android:textAllCaps="true"
        android:textStyle="bold"
        android:textColor="@color/color_label"
        android:text="@string/sign_up_label"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.78"
        app:layout_constraintRight_toRightOf="parent" />
</android.support.constraint.ConstraintLayout>
```

![](https://cdn-images-1.medium.com/max/800/1*zTczR1KjHtljNG5lHfktZQ.png)

我为每个输入字段使用一个简单的 TextInputEditText 视图，该视图被包装到一个 TextInputLayout 中，所以我们将这个浮动标签放在顶部。

为了防止文件中每个 TextInputEditText 视图的属性重复，我创建了一个包含所有常用属性的样式。实际上，我对 TextInputLayout 视图做了同样的事情。看看[这里](https://github.com/vpaliyX/LoginConcept/blob/master/app/src/main/res/values/styles.xml)。

顺便问一下，你有没有注意到这里的东西？是啊！密码文本比电子邮件文本更薄。此外，浮动标签还存在一些小问题。为了解决这个问题，你必须在 onCreate ( ) 方法中写一些代码：

```java
views.forEach(editText->{
      if(editText.getId()==R.id.password_input_edit){
          final TextInputLayout inputLayout= ButterKnife.findById(view,R.id.password_input);
          final TextInputLayout confirmLayout=ButterKnife.findById(view,R.id.confirm_password);
          Typeface boldTypeface = Typeface.defaultFromStyle(Typeface.BOLD);
          inputLayout.setTypeface(boldTypeface);
          confirmLayout.setTypeface(boldTypeface);
          editText.addTextChangedListener(new TextWatcherAdapter(){
              @Override
              public void afterTextChanged(Editable editable) {
                  inputLayout.setPasswordVisibilityToggleEnabled(editable.length()>0);
              }
           });
      }
      editText.setOnFocusChangeListener((temp,hasFocus)->{
          if(!hasFocus){
              boolean isEnabled=editText.getText().length()>0;
               editText.setSelected(isEnabled);
           }
      });
});
```

另外，如果你愿意，你可以有一个长方形的输入背景。这是一行代码中的一个简单的改变：

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <corners android:radius="50dp"/>  <!-- Change this line here to 0dp -->
    <stroke android:color="#87eeeeee"/>
    <solid android:color="#87eeeeee"/>
</shape>
```

注意！一旦应用程序打开，输入信号就会聚焦，导致打开键盘。我们不想要这个。这就是为什么我有一个简单的视图在布局的顶部，这是设置捕捉焦点:)

为了完整起见，下面是 Login 片段的快照：

![](https://cdn-images-1.medium.com/max/800/1*BlxtYE3jqvCQx5Ufbc8-Uw.png)

正如你所看到的，我使用了一个名为 VerticalTextView 的自定义视图。是的！这是说明登录/注册的粗体文本。你可以在[这里](https://github.com/vpaliyX/LoginConcept/blob/master/app/src/main/java/com/vpaliy/loginconcept/VerticalTextView.java)查看代码。

你可以说“为什么你需要一个自定义的 TextView，如果你只能使用 View.setRotation（）方法呢？”

那么，我希望，我可以做到这一点。这里的问题是，它并没有真正旋转视图。它只旋转里面的文字，但它根本不会改变视图的宽度或高度。这真的弄乱了布局:(

我在手机上拍了一张快照，如果仔细观察，可以看到即使文字是垂直的，视图边界仍然是水平的。 （在拍摄前，我启用了“显示布局边界”选项）

![](https://cdn-images-1.medium.com/max/800/1*zDXw21RVpyHjYNW09PXjbQ.png)

重点是 VerticalTextView 完成工作。它在动画方面有一些问题（在文章后面）。有关详细信息，请参阅此[链接](https://github.com/vpaliyX/LoginConcept/blob/master/app/src/main/java/com/vpaliy/loginconcept/VerticalTextView.java)。

### It’s time to add some animations!

基本上我们有三种类型的动画：

- fold  animation.
- unfold animation.
- scale animation when the input has been touched.

我们来定义动画的生命周期：

click on Sign Up

→SignUpFragment.unfold( ) 

→LogInFragment.fold( )

click on Log In 

→LogIn.fold ( )

→SignUpFragment.unfold( )

只是稍微回顾一下：

每次调用 unfold ( ) 方法时，都会调用相应的 Callback.show ( )。 由于适配器实现了 Callback 接口，所以我们可以很容易地调用另一个片段的 fold ( ) 方法，结果我们得到了两个同时运行的动画。

## Fold animation

```java
@Override
public void fold() {
   //release the lock, so you can click on the label again and get the unfold() animation running
    lock=false;
    TransitionSet set=new TransitionSet();
    set.setDuration(getResources().getInteger(R.integer.duration));
    //simple rotate transition
    Rotate transition = new Rotate();
    //at the end of the transiton our view will have -90 angle
    transition.setEndAngle(-90f);
    transition.addTarget(caption);
    //this one animates the translation from the bottom to the middle of the screen
    ChangeBounds changeBounds=new ChangeBounds();
    set.addTransition(changeBounds);
    set.addTransition(transition);
    //size and color animation 
    TextSizeTransition sizeTransition=new TextSizeTransition();
    sizeTransition.addTarget(caption);
    set.addTransition(sizeTransition);
    set.setOrdering(TransitionSet.ORDERING_TOGETHER);
    set.addListener(new Transition.TransitionListenerAdapter(){
        @Override
        public void onTransitionEnd(Transition transition) {
            super.onTransitionEnd(transition);
            caption.setTranslationX(getTextPadding());
            caption.setRotation(0);
            caption.setVerticalText(true);
            caption.requestLayout();
        }
     });
   TransitionManager.beginDelayedTransition(parent,set);
   //this is 20 sp
   caption.setTextSize(TypedValue.COMPLEX_UNIT_PX,caption.getTextSize()/2f);
   //change color to super white
   caption.setTextColor(Color.WHITE);
   ConstraintLayout.LayoutParams params=getParams();
   //release the right constraint, so the view gets translated to the left
   params.rightToRight=ConstraintLayout.LayoutParams.UNSET;
   //view is positioned in the center of the screen
   params.verticalBias=0.5f;
   caption.setLayoutParams(params);
   caption.setTranslationX(-caption.getWidth()/8+getTextPadding());
}
```

那么，我们将使用 [Andrey Kulikov](https://medium.com/@andkulikov/animate-all-the-things-transitions-in-android-914af5477d50) 的 Transition Framework。这只是 [Android 过渡 API](http://developer.android.com/reference/android/transition/package-summary.html) 的后端。它们与 Android 2.2+兼容。

ChangeBounds 是一个简单的转换，捕捉场景变化前后视图的布局边界，并在转换过程中为这些变化提供动画。 在这里它用来在屏幕上动画视图的位置。
Rotation 是捕捉视角之前和之后的转换，并在开始角度和最终角度之间创建旋转动画。
TextSizeTransition 类负责大小和颜色转换。 基本上，它使 TextView 透明，并捕获两个位图（一个用于开始字体大小，另一个用于最终字体大小）。 所以它从一开始（第一个位图）开始动画一点点，然后它交换位图，并动画剩余的方式（第二个位图）。 如果你想用一个位图来完成这个工作，你会在转换结束的时候得到一个图像。 这是一个完美的字体转换：

![](https://cdn-images-1.medium.com/max/800/1*sbHb0gfIvlbdPlMViVJy1g.gif)

### IMPORTANT!

TextSizeTransition 无法将 VerticalTextView（被设置为垂直位置）绘制到位图中。它由于某种原因被裁剪。

![](https://cdn-images-1.medium.com/max/800/1*cL0v3lOP1Xtfqo8sb-gc-Q.gif)

但是，我们可以慢慢地将视图旋转90度。 在转换停止之后，我们需要将旋转设置为0，并调用VerticalTextView.setVerticalText（true）方法，因此我们消除旋转视图后留下的额外空间。 另外，我将视图移动成左/右一点。 我这样做有两个原因：

1. 我们需要从左边/右边有一点（取决于什么片段）。 
2. 将其设置为垂直位置时，我们没有生涩的效果。

```java
 set.addListener(new Transition.TransitionListenerAdapter(){
   @Override
   public void onTransitionEnd(Transition transition) {
      caption.setTranslationX(getTextPadding());
      caption.setRotation(0);
      caption.setVerticalText(true);
      caption.requestLayout();
   }
});
```

![](https://cdn-images-1.medium.com/max/800/1*4aYyWlBRgMQT1-ZVoyoh7A.gif)

## Unfold

```java
@OnClick(R.id.caption)
public void unfold(){
    if(!lock) {
      caption.setVerticalText(false);
      caption.requestLayout();
      Rotate transition = new Rotate();
      transition.setStartAngle(-90f);
      transition.setEndAngle(0f);
      transition.addTarget(caption);
      TransitionSet set=new TransitionSet();
      set.setDuration(getResources().getInteger(R.integer.duration));
      ChangeBounds changeBounds=new ChangeBounds();
      set.addTransition(changeBounds);
      set.addTransition(transition);
      TextSizeTransition sizeTransition=new TextSizeTransition();
      sizeTransition.addTarget(caption);
      set.addTransition(sizeTransition);
      set.setOrdering(TransitionSet.ORDERING_TOGETHER);
      caption.post(()->{
         TransitionManager.beginDelayedTransition(parent, set);
         caption.setTextSize(TypedValue.COMPLEX_UNIT_PX,getResources().getDimension(R.dimen.unfolded_size));
         caption.setTextColor(ContextCompat.getColor(getContext(),R.color.color_label));
         caption.setTranslationX(0);
         ConstraintLayout.LayoutParams params = getParams();
         params.rightToRight = ConstraintLayout.LayoutParams.PARENT_ID;
         params.leftToLeft = ConstraintLayout.LayoutParams.PARENT_ID;
         params.verticalBias = 0.78f;
         caption.setLayoutParams(params);
      });
      callback.show(this);
      lock=true;
    }      
}
```

考虑到 TextView 被设置在垂直位置的事实，我们再次遇到同样的问题-- TextSizeTransition 不能有一个垂直的位图。 所以我们要做的和我们在折叠过渡中所做的完全相反。 我们必须将视图设置在水平位置，然后快速旋转视角90度（View.setRotation 方法），并慢慢旋转到0度，然后向下移动到中心位置。

为了将 TextView 设置为水平位置，我们必须使用 VerticalTextView.setVerticalText（false）方法，请求布局，并设置我们的转换。 请注意，在重绘 TextView 之后，您需要调用 TransitionManager.beginDelayedTransition ( ) 方法。 这就是为什么我在 TextView.post ( ) 方法中完成所有设置。

你可以在上面的代码中看到这个过程。 再次，这里是我们得到：

![](https://cdn-images-1.medium.com/max/800/1*4aYyWlBRgMQT1-ZVoyoh7A.gif)

正如你所看到的，从水平位置回到垂直位置的旋转发生得非常快。

## Scale animation

这很简单。 每当键盘出现在屏幕上，这意味着一个文本输入已被触摸，我们需要缩小标志和背景。 为了让我的生活更轻松，我使用了这个[库](https://github.com/yshrsmz/KeyboardVisibilityEvent)，每当键盘出现/消失在屏幕上时，都会提供回调。

```java
@Nullable
@Override
public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
  View root=inflater.inflate(authLayout(),container,false);
  ButterKnife.bind(this,root);
  KeyboardVisibilityEvent.setEventListener(getActivity(), isOpen -> {
    callback.scale(isOpen);
    if(!isOpen){
      clearFocus();
    }
  });
  return root;
}
```

这是我们的抽象类（AuthFragment.class）中的一个方法。 所以，你只需通过回调函数向适配器发送一个通知，然后处理剩下的东西。 当键盘消失后，我们必须调用 clearFocus ( ) 方法来清除焦点（View.clearFocus ( ) 方法）。

## Conclusion

如果你已经做到了这一点，你应该能够在自己的应用程序中重现这一点。在任何情况下，你都可以在 [GitHub](https://github.com/vpaliyX/LoginConcept) 的仓库中查看源代码。

![](https://cdn-images-1.medium.com/max/800/1*2p-frWEVhX33ehkjfmnFZQ.gif)

