# InstaMaterial 概念(第12部分) - RecyclerView 动画正确做法(感谢 Android Dev Summit!)

>原文 (mirekstanek.online) ： [InstaMaterial - RecyclerView animations done right (thanks to Android Dev Summit!)](https://mirekstanek.online/instamaterial-recyclerview-animations-done-right-thanks-to-android-dev-summit/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

我们生活在这样一个时代，应用不仅要有用，而且要平滑愉快。与几年前稍微不同的是，我们所要做的只是 ListView 适配器上的 notifyDataSetChanged ()。屏幕刚刚眨眼，新的数据就出现了，就这些了。

今天，在 RenderThread 时代，MaterialDesign 动画和过渡应用程序应该能够准确地显示出正在发生的事情。用户应该看到他的集合中的数据何时发生变化，或者是什么新的东西出现或者被移除了。

几个星期前，我们可以看到（实时或流媒体）伟大的 Android 会议 - Android 开发者峰会。 在这两天的深度技术会议中，我们可以看到 Android 工程团队展示了新的东西-- [Android Studio 2.0](http://android-developers.blogspot.com/2015/11/android-studio-20-preview.html)，新的 Gradle 插件，[即时运行功能](http://tools.android.com/tech-docs/instant-run)，新的官方模拟器声明等等。

顺便说一下，如果你错过了，我强烈推荐观看整个[播放列表 ](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc_Tt7q77qwyKRgytF1RzRx8) - 有关 Android 开发，工具和解决方案的很多很棒的会议。

其中一个视频 - [RecyclerView 动画和幕后原理](https://www.youtube.com/watch?v=imsr8NrIAMs&index=6&list=PLWz5rJ2EKKc_Tt7q77qwyKRgytF1RzRx8)是这篇文章的原因。[ Chet Haase](https://twitter.com/chethaase) 和 [Yigit Boyar](https://twitter.com/yigitboyar) 浏览了 RecyclerView 中的动画项目，并展示了如何做到这一点。 这可以是一个很好的起点，如何使我们的 RecyclerView 更加互动和愉快。

虽然这些只是基础，提出的解决方案可以帮助您清理您的巨大的 RecyclerView.Adapter 和解决最常见的问题。

## InstaMaterial 遇见 RecyclerView 使用指南
今天我们将从实际的角度来看看 RecyclerView 动画（很快我会尝试将这些事情正式化，深入研究整个回收视图）。

我们要清理的 InstaMaterial 源代码在此[提交](https://github.com/frogermcs/InstaMaterial/tree/0b321ce58c6d98cf8cd8e461fe2db004fc9ab17c)下可用（最新的头版本已经更新，下面描述的更改）。

同样重要的是，从用户的角度来看，我们在当前的应用程序中不会改变任何东西。 但是从代码中我们会尝试做正确的事情(或者至少在应用程序中尽可能的干净)。

## 预期的效果
我们要重建的代码负责两个类似的操作：
- 点击图像喜爱 feed
- 点击喜爱按钮喜爱 feed

这些动画应该在我们的 RecyclerView 中触发

- 外观动画 - 首次添加对象时，Feed 项目应从下方滑动
- 喜爱动画放大 - 当用户点击主图像时，应该为圆形背景的心做动画
- 喜爱按钮动画 - 当用户点击按钮或用户点击图像（和2.播放动画）时，心脏应旋转并填充，

下面是描述的动画(从最近的应用程序中记录) :

**外观动画**
![|center](http://frogermcs.github.io/images/21/add_anim.gif)
**喜爱动画放大**
![|center](http://frogermcs.github.io/images/21/big_like_anim.gif)
**喜爱按钮动画**
![|center](http://frogermcs.github.io/images/21/small_like_anim.gif)

## 代码
以前，我们的动画直接在 [FeedAdapter](https://github.com/frogermcs/InstaMaterial/blob/0b321ce58c6d98cf8cd8e461fe2db004fc9ab17c/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedAdapter.java)（RecyclerView.Adapter子类）中实现。一切正常，那么这种方法有什么问题呢？
- 动画不是 RecyclerView.Adapter 的设计目的。 据[文档](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.html#nestedclasses)记载：适配器从应用程序特定的数据集中提供一个绑定，以显示在 RecyclerView 中的视图。我们的适配器将有足够数量的绑定代码，使它有两次与动画相关的东西。
- 通过在适配器中进行动画，我们必须注意取消它们，以适当的方式处理视图回收，确保它们在适当的时间和更多的时间里播放。一切都靠我们自己。
- 虽然单项动画是可行的，但对象交互(移动 / 交换项目，在新对象出现 / 消失时更新项目位置)更为复杂。
- RecyclerView 创造者给了我们他们的官方解决方案：RecyclerView.ItemAnimator：
  该类在对适配器进行更改时对项目进行的动画。
  它接近 RecyclerView，为我们处理上述所有情况。 正因为如此，我们可以更关心动画质量，而不是在滚动生命周期中正确处理它们的逻辑。

让我们再看看我们的 [FeedAdapter](https://github.com/frogermcs/InstaMaterial/blob/0b321ce58c6d98cf8cd8e461fe2db004fc9ab17c/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedAdapter.java)。
这些代码行不应该在这里：
```java
private static final DecelerateInterpolator DECCELERATE_INTERPOLATOR = new DecelerateInterpolator();
private static final AccelerateInterpolator ACCELERATE_INTERPOLATOR = new AccelerateInterpolator();
private static final OvershootInterpolator OVERSHOOT_INTERPOLATOR = new OvershootInterpolator(4);

private static final int ANIMATED_ITEMS_COUNT = 2;
```
```java
private boolean showLoadingView = false;
```
我们需要控制什么时候我们应该动画项目(项目应该在第一次出现时动画，而不是在活动恢复过程中恢复视图时)。

```java
private final Map<RecyclerView.ViewHolder, AnimatorSet> likeAnimations = new HashMap<>();
```
我们应该把动画保存在某个地方，以防我们需要检查他们是在动还是在回收过程中取消它们。
```java
private void runEnterAnimation(View view, int position) {
    if (!animateItems || position >= ANIMATED_ITEMS_COUNT - 1) {
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
    //...
}
```
在这里，当视图被绑定时，我们每次运行进入动画，并检查是否正确的时间做这个(项目应该动画一次)。 可能方法名对于描述的情况来说有点混乱。

```java
@Override
public void onBindViewHolder(RecyclerView.ViewHolder viewHolder, int position) {
    //...
    bindLoadingFeedItem(holder);
}

private void bindDefaultFeedItem(int position, CellFeedViewHolder holder) {
    //...
    updateLikesCounter(holder, false);
    updateHeartButton(holder, false);

    holder.btnComments.setTag(position);
    holder.btnMore.setTag(position);
    holder.ivFeedCenter.setTag(holder);
    holder.btnLike.setTag(holder);

    if (likeAnimations.containsKey(holder)) {
        likeAnimations.get(holder).cancel();
    }
    resetLikeAnimationState(holder);
}
```
在 onBindViewHolder () 的绑定视图中，如果它们已经在运行，我们将取消动画。这是因为视图可以回收，我们不知道是否有任何未完成的动画。

方法 updateLikesCounter () 和 updateHeartButton () 负责以两种方式（动画和静态）设置内容。

在我们的代码中也有一个问题。
我们正在向我们的按钮传递位置：
```java
holder.btnComments.setTag(position);
holder.btnMore.setTag(position);
```
稍后在我们的 onClick () 方法中使用它：
```java
@Override
public void onClick(View view) {
//...
    if (viewId == R.id.btnComments) {
    //...
    } else if (viewId == R.id.btnMore) {
        if (onFeedItemClickListener != null) {
            onFeedItemClickListener.onMoreClick(view, (Integer) view.getTag());
        }
    }
//...
}
```
这个位置索引并不总是正确的。特别是当我们在这两种方法中使用这个位置时：将上下文菜单放置在屏幕的正确位置，并将适配器项目传递给它（理论上在这种情况下）。

由于 RecyclerView 可以以异步方式更新数据（项目视图可以在不更新数据的情况下移动，例如使用notifyItemMoved () ），所以我们的位置可能会显示错误的数据。

这与 Yigit Boyar 谈到的情况非常相似：
![|center](http://frogermcs.github.io/images/21/adapter_position.png)
我们不能认为该项目的位置将是最终的（和这张幻灯片上的代码可能会导致问题）。 
相反，我们应该使用 RecyclerView.ViewHolder 中的这两个方法：
 - [getLayoutPosition( )](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html#getLayoutPosition())
 - [getAdapterPosition( )](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html#getAdapterPosition())

## 新的实现
我们从头开始。我们的 Feed 将由这些组件建成：
- FeedItemAnimator，它扩展了 DefaultItemAnimator（它扩展了 RecyclerView.ItemAnimator）。它将为我们提供 RecyclerView 默认动画的基础（主要是淡入/淡出），我们可以在对我们很重要的地方扩展。
- LinearLayoutManager - 像以前一样，使我们的 Feed 看起来像标准列表视图。
- FeedAdapter - 用于将数据与视图绑定（仅为此目的）。

## FeedItemAnimator
以下是 [FeedItemAnimator](https://github.com/frogermcs/InstaMaterial/blob/master/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedItemAnimator.java) 的完整代码。

在这里，我们有更多有趣的代码：
```java
@Override
public boolean canReuseUpdatedViewHolder(RecyclerView.ViewHolder viewHolder) {
    return true;
}
```
当我们为 RecyclerView 项目设置动画时，我们有机会要求 RecyclerView 保留项目的前一个 ViewHolder，并提供一个新的 ViewHolder，这个 ViewHolder 将会改变上一个 ViewHolder（在我们的 RecyclerView 中只有新的ViewHolder）。 但是当我们为自己的布局编写自定义项目动画时，我们应该使用相同的 ViewHolder 并手动为内容更改设置动画。 这就是为什么我们的方法在这种情况下返回 true。

```java
@NonNull
@Override
public ItemHolderInfo recordPreLayoutInformation(@NonNull RecyclerView.State state,
                                                 @NonNull RecyclerView.ViewHolder viewHolder,
                                                 int changeFlags, @NonNull List<Object> payloads) {
    if (changeFlags == FLAG_CHANGED) {
        for (Object payload : payloads) {
            if (payload instanceof String) {
                return new FeedItemHolderInfo((String) payload);
            }
        }
    }

    return super.recordPreLayoutInformation(state, viewHolder, changeFlags, payloads);
}
```
当我们调用 notifyItemChanged () 方法时，我们可以传递额外的参数，这将有助于我们决定应该执行什么动画。

来自 FeedAdapter 的示例：
- notifyItemChanged(adapterPosition, ACTION_LIKE_IMAGE_CLICKED);
- notifyItemChanged(adapterPosition, ACTION_LIKE_BUTTON_CLICKED);

方法 recordPreLayoutInformation () 用于在更改之前捕获数据。然后 RecyclerView 调用 onBindViewHolder ()（在适配器中），最后 ItemAnimator 调用 recordPostLayoutInformation（），在此之后捕获

由于这些操作，我们可以在更改之前和之后获得有关项目状态的信息。

在所有方法的最后调用 animateChange ()，同时使用前后 ItemHolderInfo 对象。以下是我们示例中的外观：
```java
@Override
public boolean animateChange(@NonNull RecyclerView.ViewHolder oldHolder,
                             @NonNull RecyclerView.ViewHolder newHolder,
                             @NonNull ItemHolderInfo preInfo,
                             @NonNull ItemHolderInfo postInfo) {
    cancelCurrentAnimationIfExists(newHolder);

    if (preInfo instanceof FeedItemHolderInfo) {
        FeedItemHolderInfo feedItemHolderInfo = (FeedItemHolderInfo) preInfo;
        FeedAdapter.CellFeedViewHolder holder = (FeedAdapter.CellFeedViewHolder) newHolder;

        animateHeartButton(holder);
        updateLikesCounter(holder, holder.getFeedItem().likesCount);
        if (FeedAdapter.ACTION_LIKE_IMAGE_CLICKED.equals(feedItemHolderInfo.updateAction)) {
            animatePhotoLike(holder);
        }
    }

    return false;
}
```
我们可以清楚地看到，心脏按钮动画总是被触发，但是只有当用户点击图像时才会触发大图片动画。这是我们在我们所期望的效果列表中所假设的。

第二件事 - 出现动画。当我们第一次看到我们的列表时应该触发它。这是如何处理的：

```java
@Override
public boolean animateAdd(RecyclerView.ViewHolder viewHolder) {
    if (viewHolder.getItemViewType() == FeedAdapter.VIEW_TYPE_DEFAULT) {
        if (viewHolder.getLayoutPosition() > lastAddAnimatedItem) {
            lastAddAnimatedItem++;
            runEnterAnimation((FeedAdapter.CellFeedViewHolder) viewHolder);
            return false;
        }
    }

    dispatchAddFinished(viewHolder);
    return false;
}
```
当我们在 FeedAdapter 中调用 notifyItemRangeInserted () 时，会调用 RecyclerView.ItemAnimator 的方法。另一种方法是调用 notifyItemInserted ()。

还有什么？
```java
@Override
public void endAnimation(RecyclerView.ViewHolder item) {
    super.endAnimation(item);
    cancelCurrentAnimationIfExists(item);
}

@Override
public void endAnimations() {
    super.endAnimations();
    for (AnimatorSet animatorSet : likeAnimationsMap.values()) {
        animatorSet.cancel();
    }
}
```
实施这两种方法是很好的。这样做 RecyclerView 将能够停止项目视图上的动画，当它从屏幕上消失（并将准备好回收）。

这两种使用的方法也值得一提：
- dispatchAddFinished () - 应该在 animateAdd() 完成动画时调用（这会通知 RecyclerView 该视图已准备好进行回收）。
- dispatchAnimationFinished () - 如上所述，用于 animateChange ()。

就这样。我们更新的 FeedAdapter 有200行以下的代码，只负责数据视图绑定。这是它的[完整源代码](https://github.com/frogermcs/InstaMaterial/blob/984e245c0c792700e1d47a9a726f213868664ab6/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedAdapter.java)。

## 示例代码

 Github [存储库](https://github.com/frogermcs/InstaMaterial/)上提供了最新版本的 InstaMaterial 源代码。 

