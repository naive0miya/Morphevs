# 让所有的一切都动起来.Android 系统的过渡

> 原文 (Medium)：[Animate all the things. Transitions in Android](https://medium.com/@andkulikov/animate-all-the-things-transitions-in-android-914af5477d50)
>
> 作者：[Andrey Kulikov](https://medium.com/@andkulikov?source=post_header_lockup)

[TOC]

谷歌在 Material Design 有这样一个阐述：动画不再仅是 iOS 才有。这里有一种新概念叫 [Material motion](https://material.io/guidelines/motion/material-motion.html)。 

> Motion 提供了另外一层含义：让呈现在用户面前的对象没有被破坏连贯性，即使它们改变或者重组。在Material Design 中的 Motion 被用来描述空间的关系、功能和意图。 

事实上，创建动画的过程是需要时间的。但只需要调用 setVisibily(View.VISIBLE) 和将业务逻辑移到你的最棒的新特性中，因为它已经超越了所有的界限。但是记住：每一次你忽略了添加有意义的 UI 过渡，一个可悲的设计师就将出现在这个世界上。 

假使我告诉你动画比你想象的需要的工作更少，你会怎么样？你曾听说过 Transitions API 吗？是的，它是谷歌对 Activity 之间奇幻的动画所推出的。不幸的是，这仅适用于5.0以上的系统。所以，没有人实际使用过它。但是想象这种 API 能够被高效地使用在不同的情况中， 甚至更令人激动的是，能够适用在 Android 旧版本中。 

## 让我们从历史开始

在 Android 4.0 中为 ViewGroup 引入了新的 animateLayoutChange 参数。但即使调用 getLayoutTransition ( ) 并配置一些内部的东西，它仍然不稳定，不够灵活。所以你不能用它做太多事情。

4.4版 Kitkat 为我们带来了场景（Scenes）和过渡（Transitions） 的观念。 场景在技术上是我们场景根（布局容器）中所有视图的状态。 过渡是一组动画，以执行从一个场景到另一个场景的平滑过渡。这样的例子会更吸引人吗？ 当然。 

## 想象一下我们有一个按钮

点击之后，我们希望在按钮下面出现一个文本。 下面是我们的布局: 

```xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
             android:id="@+id/transitions_container"
             android:layout_width="match_parent"
             android:layout_height="match_parent"
             android:gravity="center"
             android:orientation="vertical">
 
    <Button
       android:id="@+id/button"
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:text="DO MAGIC"/>
 
    <TextView
       android:id="@+id/text"
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:layout_marginTop="16dp"
       android:text="Transitions are awesome!"
       android:visibility="gone"/>
 
</LinearLayout>
```

在 Java 中，我们有一个点击侦听器：

```java
final ViewGroup transitionsContainer = (ViewGroup) view.findViewById(R.id.transitions_container);
final TextView text = (TextView) transitionsContainer.findViewById(R.id.text);
final Button button = (Button) transitionsContainer.findViewById(R.id.button);
 
button.setOnClickListener(new View.OnClickListener() {

    boolean visible;

    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(transitionsContainer);
        visible = !visible;
        text.setVisibility(visible ? View.VISIBLE : View.GONE);          
    }

});
```

结果 :

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou1vw09gg309t054ahq.gif)

不错。 仅有一行代码的动画。 这里有趣的是，不仅文本的可见性是动画，还有一个按钮的位置也会动画。Transition 框架会自动为由 TextView 的外观引起的布局更改进行动画处理，所以你不必独自完成。 你甚至可以开始新的动画，而旧的仍在运行。Transition 框架将将停止动画的进程，然后继续动画视图从他们当前的位置。 所有这些都是自动的。 

我们可以指定将通过 beginDelayedTransition 方法的第二个参数应用的确切的 Transition 类型。

## 简单的过渡类型

- ChangeBounds。 改变视图的位置和大小 
- Fade。 继承自 Visibility 类，执行最流行的动画-渐入渐出 
- TransitionSet。 一系列过渡动画集合。它们可以混合一起开始执行，也可以按次序执行，只需要通过调用 setOrdering 方法来设置。 
- AutoTransition。 它的 TransitionSet 按顺序包含 Fade out ，ChangeBounds 和 Fade in 。 首先，第二个场景中不存在的视图被淡出，然后改变位置和大小的变化边界，最后，新的视图出现淡入。在 beginDelayedTransition 的第二个参数中，当你没有指定任何过渡时，默认使用 AutoTransition 。

## 后端

每个人都希望只编写一个在每个 Android 版本上都有一致行为的实现。希望我们可以在使用 Transitions API 的同时实现这一点。前一段时间，我找到了两个非常相似的库：

[github.com/guerwan/TransitionsBackport](https://github.com/guerwan/TransitionsBackport)

[github.com/pardom/TransitionSupportLibrary](https://github.com/pardom/TransitionSupportLibrary)

他们不再被维护，他们都错过了一些还可以移植的东西。所以，我在这两个基础上创建了自己的库，并且添加了很多新的东西来兼容旧的 Android 版本。来自棒棒糖和棉花糖的所有 API 更改也已合并。

所以，我们拥有它：[Transitions Everywhere](https://github.com/andkulikov/Transitions-Everywhere) 是 Android 4.0 及更高版本的 Transitions API 的后端。要开始使用它，只需指定 gradle 依赖关系：

```groovy
dependencies {
    compile "com.andkulikov:transitionseverywhere:1.7.8"
}
```

并且对于所有相关的类， 从 android.transition。替换为 com.transitionseverywhere 。

Google 刚刚发布了支持库过渡框架。[我写了一篇文章与我的库进行比较](https://medium.com/@andkulikov/support-library-for-transitions-overview-and-comparison-c41be713cf8c)。请检查一下。

## 我们还能做什么

首先，我们可以改变动画在过渡过程中的持续时间，插值器和启动延迟：

```java
transition.setDuration(300);
transition.setInterpolator(new FastOutSlowInInterpolator());
transition.setStartDelay(200);
```

让我们看看其他可用的过渡类型。 

## Slide

与渐变过渡相似，继承自 Visibility 类 。它帮助场景里的视图从一边滑到另外一边。 slide（Gravity.RIGHT）示例：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou3g2zcvg30b604ftdx.gif)

## Explode and Propagation

Explode 非常像 Slide，但是视图在滑动时会用基于过渡中心的一些计算方向（你应该为它提供 setEpicenterCallback 方法）。 

TransitionPropagation 为每一个动画绘画者计算启动延迟。例  如，Explode 默认使用 CircularPropagation。动画延迟取决于视图和中心的距离。要使用它，在 Transition 中调用 setPropagation。 

比方说，我们有 GridLayoutManager 的 RecyclerView，而且我们想要在点击任何一个指定的元素之后删除所有的元素。像这样： 

```java
public void onClick(View clickedView) {
    // save rect of view in screen coordinates
    final Rect viewRect = new Rect();
    clickedView.getGlobalVisibleRect(viewRect);
 
    // create Explode transition with epicenter
    Transition explode = new Explode()
        .setEpicenterCallback(new Transition.EpicenterCallback() {
            @Override
            public Rect onGetEpicenter(Transition transition) {
                return viewRect;
            }
        });
    explode.setDuration(1000);
    TransitionManager.beginDelayedTransition(recyclerView, explode);
 
    // remove all views from Recycler View
    recyclerView.setAdapter(null);
}
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou3s3lsqg307i0cfn59.gif)

## ChangeImageTransform

ChangeImageTransform 将会改变图片矩阵。它运用的场景是在当改变 ImageView 的 scaleType 时候。在大多数情况下，你想要与 ChangeBounds 配对使用来改变位置，大小和 scaleType。 

```java
TransitionManager.beginDelayedTransition(transitionsContainer, new TransitionSet()
    .addTransition(new ChangeBounds())
    .addTransition(new ChangeImageTransform()));
 
ViewGroup.LayoutParams params = imageView.getLayoutParams();
params.height = expanded ? ViewGroup.LayoutParams.MATCH_PARENT : 
    ViewGroup.LayoutParams.WRAP_CONTENT;
imageView.setLayoutParams(params);
 
imageView.setScaleType(expanded ? ImageView.ScaleType.CENTER_CROP : 
    ImageView.ScaleType.FIT_CENTER);
```

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou4pj2rig307i0cgqv5.gif)

## Path (Curved) motion

> 真实世界的力量，如重力，激发元素的运动是沿着弧线而不是直线。 

每一个过渡都操作两个坐标（例如：ChangeBounds 改变视图的位置），我们可以通过 setPathMotion 方法来运用曲线运动。 

```java
TransitionManager.beginDelayedTransition(transitionsContainer,
    new ChangeBounds().setPathMotion(new ArcMotion()).setDuration(500));
 
FrameLayout.LayoutParams params = (FrameLayout.LayoutParams) button.getLayoutParams();
params.gravity = isReturnAnimation ? (Gravity.LEFT | Gravity.TOP) :
    (Gravity.BOTTOM | Gravity.RIGHT);
button.setLayoutParams(params);
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frou5wp322g307106tju6.gif)

## TransitionName

我们需要删除容器的所有 view，并添加一组新的 view。而一些新的元素实际上和之前创建的是相同的。我们如何告诉框架，删除哪些元素，哪些元素移动到一个新的位置？ 简单。只需调用静态方法TransitionManager.setTransitionName(View v , String transitionName)，为每个 view 提供唯一的名称。 例如，如果我们创建标题列表，每次点击按钮重新创建 View 并打乱排序： 

```java
createViews(inflater, layout, titles);
shuffleButton.setOnClickListener(new View.OnClickListener() {
 
    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(layout, new ChangeBounds());
        Collections.shuffle(titles);
        createViews(inflater, layout, titles);
    }
 
});
 
// In createViews we should provide transition name for every view.

private static void createViews(LayoutInflater inflater, ViewGroup layout, List<String> titles) {
    layout.removeAllViews();
    for (String title : titles) {
        TextView textView = (TextView) inflater.inflate(R.layout.text_view, layout, false);
        textView.setText(title);
        TransitionManager.setTransitionName(textView, title);
        layout.addView(textView);
    }
}
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frou7r1q29g304007gjts.gif)

## Scale

这个实际上并不是 Transitions API 的一部分，而是我增加的。它可以使用缩放动画来显示或隐藏 view。新的 Scale ( ) 的简单示例：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou7y8335g30780550xo.gif)

也可以与其他 Transition 组合使用，例如 Fade。

```java
TransitionSet set = new TransitionSet()
    .addTransition(new Scale(0.7f))
    .addTransition(new Fade())
    .setInterpolator(visible ? new LinearOutSlowInInterpolator() : 
        new FastOutLinearInInterpolator());

TransitionManager.beginDelayedTransition(transitionsContainer, set);
text2.setVisibility(visible ? View.VISIBLE : View.INVISIBLE);
```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou83rqo3g307804njw8.gif)

## Recolor

猜一下这一个是什么动画。背景和/或文本颜色的变化动画。 

```java
TransitionManager.beginDelayedTransition(transitionsContainer, new Recolor());
 
button.setTextColor(getResources().getColor(!isColorsInverted ? R.color.second_accent :R.color.accent));
button.setBackgroundDrawable(
    new ColorDrawable(getResources().getColor(!mColorsInverted ? R.color.accent :
        R.color.second_accent)));
```

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frou88d4rig306o049who.gif)

## Rotate

例如：

```java
TransitionManager.beginDelayedTransition(transitionsContainer, new Rotate());
icon.setRotation(isRotated ? 135 : 0);
```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frou8h68slg302d02baad.gif)

## ChangeText

帮助为文本更改添加简单的淡入淡出动画。

```java
TransitionManager.beginDelayedTransition(transitionsContainer,
    new ChangeText().setChangeBehavior(ChangeText.CHANGE_BEHAVIOR_OUT_IN));
 textView.setText(isSecondText ? TEXT_2 : TEXT_1);
```

## Targets

Transitions 很容易配置. 你可以为任何 Transition 指定目标 view，而且只有它们的可以动画. 

添加目标的方法：

- addTarget(View target) — view itself
- addTarget(int targetViewId) — id of view
- addTarget(String targetName) — do you remember about method TransitionManager.setTransitionName?
- addTarget(Class targetType) — for example android.widget.TextView.class

要删除目标：

- removeTarget(View target)
- removeTarget(int targetId)
- removeTarget(String targetName)
- removeTarget(Class target)

排除一些视图：

- excludeTarget(View target, boolean exclude)
- excludeTarget(int targetId, boolean exclude)
- excludeTarget(Class type, boolean exclude)
- excludeTarget(Class type, boolean exclude)

为了排除一些 ViewGroup 的所有孩子：

- excludeChildren(View target, boolean exclude)
- excludeChildren(int targetId, boolean exclude)
- excludeChildren(Class type, boolean exclude)

## Create Transition with xml

Transition 可以 xml 创建， 文件目录 res/anim。例如: 

```xml
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:app="http://schemas.android.com/apk/res-auto"
              app:transitionOrdering="together"
              app:duration="400">
    <changeBounds/>
    <changeImageTransform/>
    <fade
       app:fadingMode="fade_in"
       app:startDelay="200">
        <targets>
            <target app:targetId="@id/transition_title"/>
        </targets>
    </fade>
</transitionSet>

// And inflating:
TransitionInflater.from(getContext()).inflateTransition(R.anim.my_the_best_transition);
```

## Activity and Fragment transitions

Activity Transitions 没办法向后移植 ，抱歉。大量的逻辑隐藏在 Activity 中。这同样适用于 Fragment transitions。我们要创造我们自己的 Fragment transitions 逻辑。 

## Custom Transitions

对于每个视图，过渡都可以用于各种目的。 让我们尝试创造独特的东西 - 我们自己的过渡。

我们所要做的就是实现三个方法：captureStartValues， captureEndValues 和 createAnimator。 前两个方法是动画之前和动画之后捕获 view 的状态。 

我们将为水平 ProgressBar 进度平滑变化的动画：

```java
private class ProgressTransition extends Transition {
 
    /**
     * Property is like a helper that contain setter and getter in one place
     */
    private static final Property<ProgressBar, Integer> PROGRESS_PROPERTY = 
        new IntProperty<ProgressBar>() {
 
        @Override
        public void setValue(ProgressBar progressBar, int value) {
            progressBar.setProgress(value);
        }
 
        @Override
        public Integer get(ProgressBar progressBar) {
            return progressBar.getProgress();
        }
    };
 
    /**
     * Internal name of property. Like a intent bundles 
     */
    private static final String PROPNAME_PROGRESS = "ProgressTransition:progress";
 
    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        captureValues(transitionValues);
    }
 
    @Override
    public void captureEndValues(TransitionValues transitionValues) {
        captureValues(transitionValues);
    }
 
    private void captureValues(TransitionValues transitionValues) {
        if (transitionValues.view instanceof ProgressBar) {
            // save current progress in the values map
            ProgressBar progressBar = ((ProgressBar) transitionValues.view);
            transitionValues.values.put(PROPNAME_PROGRESS, progressBar.getProgress());
        }
    }
 
    @Override
    public Animator createAnimator(ViewGroup sceneRoot, TransitionValues startValues, 
            TransitionValues endValues) {
        if (startValues != null && endValues != null && endValues.view instanceof ProgressBar) {
            ProgressBar progressBar = (ProgressBar) endValues.view;
            int start = (Integer) startValues.values.get(PROPNAME_PROGRESS);
            int end = (Integer) endValues.values.get(PROPNAME_PROGRESS);
            if (start != end) {
                // first of all we need to apply the start value, because right now
                // the view has end value
                progressBar.setProgress(start);
                // create animator with our progressBar, property and end value
                return ObjectAnimator.ofInt(progressBar, PROGRESS_PROPERTY, end);
            }
         }
         return null;
    }
}
```

而如何使用我们全新的 Transition？

```java
private void setProgress(int value) {
    TransitionManager.beginDelayedTransition(mTransitionsContainer, new ProgressTransition());
    value = Math.max(0, Math.min(100, value));
    mProgressBar.setProgress(value);
}
```

结果 :

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frou8nyi11g30b903m0wu.gif)

## PS

本文中所有的例子，都在 Github 的工程中： 

[github.com/andkulikov/transitions-everywhere](https://github.com/andkulikov/transitions-everywhere)

