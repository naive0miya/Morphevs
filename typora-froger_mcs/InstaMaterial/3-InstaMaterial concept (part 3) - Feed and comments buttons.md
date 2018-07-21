# InstaMaterial 概念(第3部分) - Feed 和评论按钮

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 3) - Feed and comments buttons](https://mirekstanek.online/instamaterial-concept-part-3-feed-and-comments-buttons/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的一部分，展示了使用 [Material Design 概念的 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q) 的 Android 实现。 今天我们会关注一些我们之前跳过的细节。 这意味着我们仍然在9秒到13秒的时间内从概念视频。

这是今天发布的最终效果（针对 v21和 pre_21 版本）：

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>



## 初始配置
这里没有什么特别的事情发生。 我们只需要为 feed 元素中的按钮添加两个图像（喜爱和评论图标）并修改当前的模拟图像。 这是在这个[提交](https://github.com/frogermcs/InstaMaterial/commit/570645d2ec2f792be3f2f4523af083fe9df2799d)中完成的。 在那之后，但是在我们开始实现新的东西之前，我们必须研究。

## 错误修正和性能改进
没错，即使在像 InstaMaterial 这样的小项目中，你总是可以找到一些可以修复或改进的东西。 这里也是一样。

## 工具栏主题
首先，我错过了工具栏的样式。 这就是为什么菜单按钮的选择器(工具栏上的左键)有默认的黑色版本(下面的屏幕)。

![|center](http://frogermcs.github.io/images/4/toolbar_without_styling.png)
收件箱按钮（工具栏上的右键）具有亮版本，因为它使用了自定义视图的选择器，
位于 menu_item_view.xml 中的 android:background =“@ drawable / btn_default_light”。

这种不一致只在 v21中出现，修复非常简单。我们所需要做的就是在 activity_comments.xml 和 activity_main.xml 中向工具栏添加单行：
```xml
app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
```
由于这一行，工具栏视图中的所有项目都将使用 Dark.Actionbar 主题的默认样式。顺便说一下，如果你对 Android 的风格和主题感兴趣，并且想知道它们之间的主要区别是什么，那么有必要提出一些关于 [Android 样式视图的文章(不要疯狂)](http://blog.danlew.net/2014/11/19/styles-on-android/)。

我们将应用新的菜单选择器应如下所示（仅限 v21）：
![|center](http://frogermcs.github.io/images/4/toolbar_with_styling.png)

## RecyclerView 元素预加载
我在项目中发现的另一个问题是在我们启动应用程序之后的 feed 滚动平滑性。 几乎每次我们滚动到第二个项目，布局结束一段时间。 幸运的是，这个问题的原因很简单。 您可能已经知道了 RecyclerView (以及其他类似 ListView，GridView 等适配器视图)使用回收机制重用视图(简而言之，系统只对屏幕上可见的项目保持内存布局，并重用它们，而不是创建新的，如果你滚动它)。

我们的项目的问题是，我们启动应用程序后，屏幕上只有一个可见的 feed 元素。当我们开始滚动时， RecyclerView 没有可以重复使用的视图，所以当我们到达第二个元素的时候，它必须被排除。不幸的是，这需要一些时间，因此我们的 RecyclerView 在这一刻结束了。之后滚动变得平滑，因为我们在内存中至少有两个元素可以重用(我们不需要从头创建)。

## 如何解决这个问题？
在我们的例子中，由于我们使用的 LinearLaymanager，它非常简单。 我们所要做的就是重写 getExtraLayoutSpace () 方法。 根据[文档](https://developer.android.com/reference/android/support/v7/widget/LinearLayoutManager.html#getExtraLayoutSpace)，它应该返回额外的空间(像素) ，应该由我们的管理器布局:

```java
public class MainActivity extends ActionBarActivity implements FeedAdapter.OnFeedItemClickListener {
   
    //...

    private void setupFeed() {
        //Increase the amount of extra space that should be laid out by LayoutManager.
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this) {
            @Override
            protected int getExtraLayoutSpace(RecyclerView.State state) {
                return 300;
            }
        };
        rvFeed.setLayoutManager(linearLayoutManager);

        //...
    }

    //...
}
```
这是它在实践中的样子：
覆盖 getExtraLayoutSpace () 之前：
![|center](http://frogermcs.github.io/images/4/stuttered.gif)
之后：
![|center](http://frogermcs.github.io/images/4/not_stuttered.gif)

## Feed 按钮
让我们从 feed 按钮开始（喜爱和评论）。正如我们在概念视频中看到的那样，他们有着奇特的圆形选择器。

![](http://frogermcs.github.io/images/4/ripple.gif)

## v21 中的水波纹
Feed 按钮中使用的选择器只不过是安卓 v21 中引入的水纹波效果。有很多关于水波纹的教程和博客文章，所以我不想深入探讨这个话题。这里有几个链接：

- [水波纹- 第1部分](https://blog.stylingandroid.com/ripples-part-1/)和[水波纹 - 第2部分](https://blog.stylingandroid.com/ripples-part-2/) 出自 Styling Android 博客
- 实现安卓应用程序的[材质设计](http://android-developers.blogspot.com/2014/10/implementing-material-design-in-your.html) 出自官方安卓开发人员博客

在我们的例子中，水纹波的实现是简短而简单的:
位于 res/drawable-v21/btn_feed_action.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable-v21/btn_feed_action.xml-->
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorPrimaryDark" />
```
这就是我们在 feed_item.xml 中将它用作按钮背景（第17行和第24行）的方法：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- item_feed.xml -->
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android">

        <!--...-->
        
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:paddingLeft="8dp"
            android:paddingRight="8dp">

            <ImageButton
                android:id="@+id/btnLike"
                android:layout_width="48dp"
                android:layout_height="48dp"
                android:background="@drawable/btn_feed_action"
                android:src="@drawable/ic_heart_outline_grey" />

            <ImageButton
                android:id="@+id/btnComments"
                android:layout_width="48dp"
                android:layout_height="48dp"
                android:background="@drawable/btn_feed_action"
                android:src="@drawable/ic_comment_outline_grey" />
        </LinearLayout>

        <!--...-->
        
</android.support.v7.widget.CardView>
```
## 不要在 pre-21 的版本中使用水波纹

我知道我承诺从概念视频中复制几乎所有的东西，但有时最好不要做一些不符合期望的事情。正如在 pre-21 的版本中使用水波纹。

当然，现在我们有一些模仿水波纹效果使用在 pre-21的库，但是它们都不能正常工作。他们需要额外的代码（即额外的布局包装），不要在 v21 使用原生水纹波效果，有性能问题等。

但为什么如此难以将水纹波效果复制到 pre-21 的设备？

## 水波纹 - 背后的原理
在 pre-21，整个 UI 在一个“主”线程 - UI 工具包线程中进行管理。 几乎每个人都知道 ANR 对话框，我们中的大多数人面临着 [NetworkOnMainThreadException](http://developer.android.com/reference/android/os/NetworkOnMainThreadException.html) 并知道黄金法则 - 将所有长时间运行的操作（网络，数据库访问，图像处理）移动到工作线程，并仅处理 UI 线程中的结果。 除了第一句中的一条规则——整个 UI 必须在主线程中管理，一切都会好的。

随着更复杂和丰富的应用程序布局，用户界面本身变得更加苛刻，需要更多时间进行测量，绘制和布局操作。问题就在这里。 如果我们处于动画的中间，并开始另一个 UI 任务(比如新打开的 Acitvity 的附加布局) ，问题是这两个任务都是在同一个线程中执行的，因此动画会在活动启动时停止。

在 v21 中引入的渲染线程通过拆分两个渲染过程来帮助处理这些情况。 简而言之，我们有在 UI 工具线程中创建的原子动画列表，然后我们将它们发送到独立存在的呈现线程中。 感谢它将继续执行这些原子动画，即使 UI 工具包线程正在做一个昂贵的操作(例如附加一个活动)。

事实上这就是水波纹的工作原理。 它们在渲染线程中执行，完全独立于 UI 工具包线程，即使新的活动窗口出现，它们也不能被中断或停止。

这就是为什么在 pre-21 中没有（简单的）方法来实现水波纹效果。

## 旧版安卓 Pre-ripple 选择器

因为我们不能实现水波纹，让我们做一些类似的事情。 我们创建带有淡入淡出的圆形选择器。 也许它没有水波纹那么有效，但看起来还可以。 这里有一个例子:

位于 res / drawable / btn_feed_action.xml：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/btn_feed_action.xml-->
<selector xmlns:android="http://schemas.android.com/apk/res/android" android:enterFadeDuration="200" android:exitFadeDuration="200">
    <item android:state_pressed="false">
        <shape android:shape="oval">
            <solid android:color="@android:color/transparent" />
        </shape>
    </item>
    <item android:state_pressed="true">
        <shape android:shape="oval">
            <solid android:color="#6621425d" />
        </shape>
    </item>
</selector>
```

## 发送评论按钮
对我们来说更有趣的是 SendCommentButton。正如你在概念视频中所注意到的，它在两个状态之间简单的平移动画和水波纹 onClick 选择器。
![|center](http://frogermcs.github.io/images/4/send_comment.gif)
对于这个按钮，我们将使用一些安卓元素：

- [ViewAnimator](http://developer.android.com/reference/android/widget/ViewAnimator.html) 作为我们按钮的基类。它有一个功能，对我们来说最有趣 - 它可以在切换其子视图时执行进入和退出动画。
- 在 in.xml 中构建的自定义视图，并在代码中附加
- 消除冗余视图组的 `<merge>` 标签

## 实现
好的，我们从动画开始。其实我们需要4个 - 发送状态2（进入和退出）和完成状态2（进入和退出，但反过来）：
- 发送状态:

  1.从顶部滑出:
```xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="100"
    android:fromYDelta="0"
    android:interpolator="@android:anim/linear_interpolator"
    android:toYDelta="-80%p" />
```

​      2.从顶部滑入:

```xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="100"
    android:fromYDelta="-80%p"
    android:interpolator="@android:anim/linear_interpolator"
    android:toYDelta="0" />
```
- 完成状态:

  1.从底部滑入:
```xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="100"
    android:fromYDelta="80%p"
    android:interpolator="@android:anim/linear_interpolator"
    android:toYDelta="0" />
```

​      2.从底部滑出:

```xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="100"
    android:fromYDelta="0"
    android:interpolator="@android:anim/linear_interpolator"
    android:toYDelta="80%p" />
```
现在我们来实现按钮的布局：
```xml
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <TextView
        android:id="@+id/tvSend"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:text="SEND"
        android:textColor="#ffffff"
        android:textSize="12sp" />

    <TextView
        android:id="@+id/tvDone"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:text="✓"
        android:textColor="#ffffff"
        android:textSize="12sp" />
</merge>
```
我们使用了 `<merge>` 标签，因为我们的 SendCommentButton 本身是 ViewGroup。谢谢这个设计，我们可以消除一个冗余的 ViewGroup。

最后一件事 - 实现。现在我们对按钮只有一个要求 - 在将状态切换到 STATE_DONE 之后，应该在两秒钟之后将其切换回 SEND_STATE。

整个代码非常简单：
```java
public class SendCommentButton extends ViewAnimator implements View.OnClickListener {
    public static final int STATE_SEND = 0;
    public static final int STATE_DONE = 1;

    private static final long RESET_STATE_DELAY_MILLIS = 2000;

    private int currentState;

    private OnSendClickListener onSendClickListener;

    private Runnable revertStateRunnable = new Runnable() {
        @Override
        public void run() {
            setCurrentState(STATE_SEND);
        }
    };

    public SendCommentButton(Context context) {
        super(context);
        init();
    }

    public SendCommentButton(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        LayoutInflater.from(getContext()).inflate(R.layout.view_send_comment_button, this, true);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        currentState = STATE_SEND;
        super.setOnClickListener(this);
    }

    @Override
    protected void onDetachedFromWindow() {
        removeCallbacks(revertStateRunnable);
        super.onDetachedFromWindow();
    }

    public void setCurrentState(int state) {
        if (state == currentState) {
            return;
        }

        currentState = state;
        if (state == STATE_DONE) {
            setEnabled(false);
            postDelayed(revertStateRunnable, RESET_STATE_DELAY_MILLIS);
            setInAnimation(getContext(), R.anim.slide_in_done);
            setOutAnimation(getContext(), R.anim.slide_out_send);
        } else if (state == STATE_SEND) {
            setEnabled(true);
            setInAnimation(getContext(), R.anim.slide_in_send);
            setOutAnimation(getContext(), R.anim.slide_out_done);
        }
        showNext();
    }

    @Override
    public void onClick(View v) {
        if (onSendClickListener != null) {
            onSendClickListener.onSendClickListener(this);
        }
    }

    public void setOnSendClickListener(OnSendClickListener onSendClickListener) {
        this.onSendClickListener = onSendClickListener;
    }

    @Override
    public void setOnClickListener(OnClickListener l) {
        //Do nothing, you have you own onClickListener implementation (OnSendClickListener)
    }

    public interface OnSendClickListener {
        public void onSendClickListener(View v);
    }
}
```
- init () 方法（第29行）直接在我们的 ViewAnimator 中附加我们以前创建的布局。顺便说一下，看看这篇关于

  [正确的布局附加](http://www.doubleencore.com/2013/05/layout-inflation-as-intended/)以及它的参数，你可能以前并不关心。

- onDetachedFromWindow () 你有一个调度，但还没有完成（即你在按钮状态切换回去之前销毁 Acitivty）的情况下，删除 revertStateRunnable 回调。

其余的很简单。通过调用 setInAnimation () 和 setOutAnimation () 我们切换动画进入和退出动作。

太棒了，我们刚刚完成了 SendCommentButton 实现。

## 设计
与 Feed 动作按钮相同，我们必须为 SendCommentButton 准备两个不同的选择器。 对于 v21 的版本，我们将再次使用水波纹效果（这个时间限制在按钮框架中）。 对于 pre-21 的版本，我们将创建标准版本，并按照本系列第一篇文章中描述的 FAB 按钮的相同方式模拟按下状态和阴影。

这里是两者得 .xml 文件的代码：
位于 res/drawable-v21/btn_send_comment.xml:
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable-v21/btn_send_comment.xml-->
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="#ffffff">
    <item>
        <shape android:shape="rectangle">
            <solid android:color="@color/btn_send_normal" />
        </shape>
    </item>
</ripple>
```
位于 res/drawable/btn_send_comment.xml:
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--drawable/btn_send_comment.xml-->
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="false">
        <layer-list>
            <item android:bottom="0dp" android:left="2dp" android:right="0dp" android:top="2dp">
                <shape android:shape="rectangle">
                    <solid android:color="@color/fab_color_shadow" />
                    <corners android:radius="2dp" />
                </shape>
            </item>

            <item android:bottom="1dp" android:left="1dp" android:right="1dp" android:top="1dp">
                <shape android:shape="rectangle">
                    <solid android:color="@color/btn_send_normal" />
                    <corners android:radius="2dp" />
                </shape>
            </item>
        </layer-list>
    </item>
    <item android:state_pressed="true">
        <shape android:bottom="0dp" android:left="2dp" android:right="0dp" android:shape="rectangle" android:top="2dp">
            <solid android:color="@color/btn_send_pressed" />
            <corners android:radius="2dp" />
        </shape>
    </item>

</selector>
```

## 奖励 - 错误时摇动

最后，我们将添加一个在概念视频动画中没有显示的功能，以防止注释文本字段为空。 下面是这个效果的预览(在真正的设备上看起来好多了) :
![|center](http://frogermcs.github.io/images/4/shake_error.gif)

实现非常简单 - 我们必须创建摇动动画和自定义 CycleInterpolator 来重复此动画。一切都在几行代码中：
res/anim/shake_error.xml:
```xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:fromXDelta="0%"
    android:interpolator="@anim/cycle_2"
    android:toXDelta="2%" />
```
res/anim/cycle_2.xml:
```xml
<?xml version="1.0" encoding="utf-8"?>
<cycleInterpolator xmlns:android="http://schemas.android.com/apk/res/android"
                   android:cycles="2"/>
```
如果我们想在代码中使用它，我们已经开始这个视图的动画，如下所示：
```java
btnSendComment.startAnimation(AnimationUtils.loadAnimation(this, R.anim.shake_error));
```
而今天就是这样！这里是我们的 SendCommentButton 用法的[最后一个提交](https://github.com/frogermcs/InstaMaterial/commit/1dac9dd2ac6f5f89e09c2791b6db06738e6b6b93)（这只是一个不值得一提的样板）。在下一篇文章中，我们将最后进入概念视频的下一个时期。

## 示例代码 
所描述示例的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。

