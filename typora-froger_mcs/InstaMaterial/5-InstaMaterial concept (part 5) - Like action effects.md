# InstaMaterial 概念(第5部分) - 喜爱动画效果

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 5) - Like action effects](https://mirekstanek.online/instamaterial-concept-part-5-like-action-effects/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列文章的一部分，这些帖子展示了 Android 实现 [INSTAGRAM 的 Material Design 概念](https://www.youtube.com/watch?v=ojwdmgmdR_Q)。 今天，我们将在概念视频的第20和27秒之间提供的 feed 项目中创建类似的效果。

这是今天发布的最终效果（针对 v21和 Pre_21 版本）：

<iframe width="560" height="315" src="https://youtu.be/eQwFwJ4Glyc" frameborder="0" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://youtu.be/DNT7j0JjrtE" frameborder="0" allowfullscreen></iframe>

## 喜爱项目

基于[概念视频](https://www.youtube.com/watch?v=ojwdmgmdR_Q)，在我们的项目中，我们有两种喜爱项目的方法，一种是按心键，另一种是点击图片(实际上我很确定它在双击之后应该能够工作，就像 Instagram 应用程序一样，但是让它成为你实现双击处理程序的小练习)。

像往常一样，我们必须更新和添加我们项目中使用的一些资源。 今天，我们不得不增加一些心 ：(从左边开始：从左边开始: heart for like counter，heart for liked button，heart for photo like animation)
![|center](http://frogermcs.github.io/images/6/heart_small_blue.png)
![|center](http://frogermcs.github.io/images/6/heart_red.png)
![|center](http://frogermcs.github.io/images/6/heart_outline_white.png)

## 喜爱计数
让我们从最简单的效果开始。 类似计数器应该用 in / out 动画来更新(旧值增加，而新的则来自下面)。 这就是它在实践中的样子:
![|center](http://frogermcs.github.io/images/6/likes_counter.gif)
看起来很熟悉？这是正确的 - 这与在系列的[第三篇文章](http://frogermcs.github.io/InstaMaterial-concept-part-3-feed-and-comments-buttons/)中描述的发送评论按钮一样。

这次我们将使用 [TextSwitcher ](http://developer.android.com/reference/android/widget/TextSwitcher.html)- ViewGroup 类，这个类帮助在由几个 TextViews 组成的标签之间动画处理。为了我们的需要，我们将使用其中的两个（我们将通过调用 TestSwitcher.setText（）方法在它们之间使用更新的文本值）。

首先让我们通过在底部面板中添加类似的计数器来更新我们的 item_feed.xml 布局：
```xml
<!--...-->
<LinearLayout
    android:layout_width="0dp"
    android:layout_height="match_parent"
    android:layout_weight="1"
    android:gravity="center_vertical|right">

    <ImageView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@drawable/ic_heart_small_blue" />

    <TextSwitcher
        android:id="@+id/tsLikesCounter"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="8dp"
        android:layout_marginRight="8dp"
        android:inAnimation="@anim/slide_in_likes_counter"
        android:outAnimation="@anim/slide_out_likes_counter">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="123 likes"
            android:textColor="@color/text_like_counter" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/text_like_counter" />
    </TextSwitcher>
</LinearLayout>
<!--...-->
```
以下是来自 Android Studio 的视觉预览：
![](http://frogermcs.github.io/images/6/likes_counter_preview.png)
在我们的代码中使用的 TextSwitcher 将在其两个子 TextView 之间进行动画处理。我们不必担心隐藏它们 - 它是自动生成的。
android：inAnimation 和 android：outAnimation 值用于前一个和下一个子节点之间的转换。

### 0dp 为了更好的优化

包装我们的 喜爱计数器的 LinearLayout 使用 layout_weight = “1”（请记住，它的父母也是 LinearLayout）。 当 LinearLayout 中只有一个孩子使用权重值时，它将吸收父母的所有剩余空间（就像我们的情况一样 - 查看上面的截图）。 通过使用 layout_width = “0dp”（或垂直定向的 LinearLayouts 中的高度），我们做了一些改进，因为我们的视图不必首先测量它自己的大小。

现在我们来添加一些代码给我们的 FeedAdapter。唯一值得一提的代码是 updateLikesCounter（）方法（这里是所有[更改的完整提交](https://github.com/frogermcs/InstaMaterial/commit/090cf15ac0a00ef9bbc5f952fd9c9339838a580f)）：
```java
private void updateLikesCounter(CellFeedViewHolder holder, boolean animated) {
    int currentLikesCount = likesCount.get(holder.getPosition()) + 1;
    String likesCountText = context.getResources().getQuantityString(
            R.plurals.likes_count, currentLikesCount, currentLikesCount
    );

    if (animated) {
        holder.tsLikesCounter.setText(likesCountText);
    } else {
        holder.tsLikesCounter.setCurrentText(likesCountText);
    }

    likesCount.put(holder.getPosition(), currentLikesCount);
}
```
这种方法可以通过两种方式更新 喜爱计数器:

- 有动画（用于动态更新 - 用户执行动作） - 它使用 TextSwitcher.setText（）方法
- 没有动画 （用于 onBindViewHolder () 来设置 feed 项目的当前 喜爱计数）- 此方法更新当前显示 TextView 的文本。

还有，你可能已经注意到了，我们正在使用数量字符串来处理带数量的单词。 在这里你可以找到更多关于在 Android 中使用 [Plurals](http://developer.android.com/guide/topics/resources/string-resource.html#Plurals) 的详细信息。

## 按钮的喜爱动画
现在我们将专注于 喜爱按钮动画。它的行为比概念视频有点奇特，看起来像下面的截图
![|center](http://frogermcs.github.io/images/6/heart_button.gif)

### 编排多个动画

这个效果是由一个集合中的多个动画组成的。 这就是为什么我们不能像以前那样使用 ViewPropertyAnimator。 幸运的是，有一个简单的方法来播放多个动画，这些动画是相互依赖的。 感谢 [AnimatorSet](http://developer.android.com/reference/android/animation/AnimatorSet.html)，我们可以将动画捆绑在一起，同时、顺序或混合的方式播放它们。 这里有更详细的描述：[编排多个动画](http://developer.android.com/guide/topics/graphics/prop-animation.html#choreography)

我们的 喜爱按钮效果由3个动画组成：
- 360度旋转 - 在我们的动画集开始时播放
- 反弹动画 - 由两个动画组合（scaleX 和 scaleY 动画）组成，旋转完成后立即发射。
下面是负责设置按钮图像的方法的完整源代码(以动画和静态的方式，如喜欢计数器) ：

```java
private void updateHeartButton(final CellFeedViewHolder holder, boolean animated) {
    if (animated) {
        if (!likeAnimations.containsKey(holder)) {
            AnimatorSet animatorSet = new AnimatorSet();
            likeAnimations.put(holder, animatorSet);

            ObjectAnimator rotationAnim = ObjectAnimator.ofFloat(holder.btnLike, "rotation", 0f, 360f);
            rotationAnim.setDuration(300);
            rotationAnim.setInterpolator(ACCELERATE_INTERPOLATOR);

            ObjectAnimator bounceAnimX = ObjectAnimator.ofFloat(holder.btnLike, "scaleX", 0.2f, 1f);
            bounceAnimX.setDuration(300);
            bounceAnimX.setInterpolator(OVERSHOOT_INTERPOLATOR);

            ObjectAnimator bounceAnimY = ObjectAnimator.ofFloat(holder.btnLike, "scaleY", 0.2f, 1f);
            bounceAnimY.setDuration(300);
            bounceAnimY.setInterpolator(OVERSHOOT_INTERPOLATOR);
            bounceAnimY.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationStart(Animator animation) {
                    holder.btnLike.setImageResource(R.drawable.ic_heart_red);
                }
            });

            animatorSet.play(rotationAnim);
            animatorSet.play(bounceAnimX).with(bounceAnimY).after(rotationAnim);

            animatorSet.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    resetLikeAnimationState(holder);
                }
            });

            animatorSet.start();
        }
    } else {
        if (likedPositions.contains(holder.getPosition())) {
            holder.btnLike.setImageResource(R.drawable.ic_heart_red);
        } else {
            holder.btnLike.setImageResource(R.drawable.ic_heart_outline_grey);
        }
    }
}
```
在第25行和第26行，我们组成了我们的最终动画。
这次我们使用了 ObjectAnimator，它可以动画目标对象的属性。 我们可以使用 int，float 和 argb 值来动画和使用具有接受选定类型值的 setter 的对象的每个属性。所以在我们使用 ObjectAnimator.ofFloat（）方法的情况下，使用旋转值通过调用 setRotation（Float f）方法来动画旋转。

这里是完整的提交，其中添加了[喜爱按钮动画](https://github.com/frogermcs/InstaMaterial/commit/1dcbcf491436c70cae8772bbc3cb57098810aa07)。

## 图像的喜爱动画
更复杂的是像图片一样的动画。这一次我们对两个不同的对象进行了动画（但仍然在一个 AnimatorSet 中）：

- 动画心形下面的圆形背景（缩放和淡入淡出）
- 心形图标（放大和缩小）

效果如下所示：
![](http://frogermcs.github.io/images/6/photo_like.gif)
这里是这个动画的源代码：
```java
private void animatePhotoLike(final CellFeedViewHolder holder) {
    if (!likeAnimations.containsKey(holder)) {
        holder.vBgLike.setVisibility(View.VISIBLE);
        holder.ivLike.setVisibility(View.VISIBLE);

        holder.vBgLike.setScaleY(0.1f);
        holder.vBgLike.setScaleX(0.1f);
        holder.vBgLike.setAlpha(1f);
        holder.ivLike.setScaleY(0.1f);
        holder.ivLike.setScaleX(0.1f);

        AnimatorSet animatorSet = new AnimatorSet();
        likeAnimations.put(holder, animatorSet);

        ObjectAnimator bgScaleYAnim = ObjectAnimator.ofFloat(holder.vBgLike, "scaleY", 0.1f, 1f);
        bgScaleYAnim.setDuration(200);
        bgScaleYAnim.setInterpolator(DECCELERATE_INTERPOLATOR);
        ObjectAnimator bgScaleXAnim = ObjectAnimator.ofFloat(holder.vBgLike, "scaleX", 0.1f, 1f);
        bgScaleXAnim.setDuration(200);
        bgScaleXAnim.setInterpolator(DECCELERATE_INTERPOLATOR);
        ObjectAnimator bgAlphaAnim = ObjectAnimator.ofFloat(holder.vBgLike, "alpha", 1f, 0f);
        bgAlphaAnim.setDuration(200);
        bgAlphaAnim.setStartDelay(150);
        bgAlphaAnim.setInterpolator(DECCELERATE_INTERPOLATOR);

        ObjectAnimator imgScaleUpYAnim = ObjectAnimator.ofFloat(holder.ivLike, "scaleY", 0.1f, 1f);
        imgScaleUpYAnim.setDuration(300);
        imgScaleUpYAnim.setInterpolator(DECCELERATE_INTERPOLATOR);
        ObjectAnimator imgScaleUpXAnim = ObjectAnimator.ofFloat(holder.ivLike, "scaleX", 0.1f, 1f);
        imgScaleUpXAnim.setDuration(300);
        imgScaleUpXAnim.setInterpolator(DECCELERATE_INTERPOLATOR);

        ObjectAnimator imgScaleDownYAnim = ObjectAnimator.ofFloat(holder.ivLike, "scaleY", 1f, 0f);
        imgScaleDownYAnim.setDuration(300);
        imgScaleDownYAnim.setInterpolator(ACCELERATE_INTERPOLATOR);
        ObjectAnimator imgScaleDownXAnim = ObjectAnimator.ofFloat(holder.ivLike, "scaleX", 1f, 0f);
        imgScaleDownXAnim.setDuration(300);
        imgScaleDownXAnim.setInterpolator(ACCELERATE_INTERPOLATOR);

        animatorSet.playTogether(bgScaleYAnim, bgScaleXAnim, bgAlphaAnim, imgScaleUpYAnim, imgScaleUpXAnim);
        animatorSet.play(imgScaleDownYAnim).with(imgScaleDownXAnim).after(imgScaleUpYAnim);

        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                resetLikeAnimationState(holder);
            }
        });
        animatorSet.start();
    }
}
```
我们再次使用了 ObjectAnimator 和 AnimatorSet（第40-41行）。 今天就到这里，感谢阅读！

## 示例代码 
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。



