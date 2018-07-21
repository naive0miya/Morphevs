# InstaMaterial 概念(第14部分) - RecyclerView 动画 - AndroidDevSummit 的写作

>原文 (mirekstanek.online) ： [RecyclerView animations - AndroidDevSummit write-up)](https://mirekstanek.online/recyclerview-animations-androiddevsummit-write-up/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

[RecyclerView 动画和背后原理](https://www.youtube.com/watch?v=imsr8NrIAMs) - 又一次我们回到这个演示。 事实上，在所有移动平台上，列表视图（或更通用的集合视图）是应用程序中最常用的视图模式。 所以尽可能地了解它们是非常重要的。

今天基于 AndroidDev Summit 演示文稿，我们将仔细看看 RecyclerView 项目动画。



## RecyclerView 中的默认动画
默认情况下，添加，删除和更改项目等基本操作 SDK 为我们提供了这些动画：
- 淡入淡出（添加和删除项目）
- 平移（将剩余的项目转移到正确的位置）
- 交叉淡入淡出（特定项目更新）

所有这些都由 DefaultItemAnimator 类提供，它扩展了 RecyclerView.ItemAnimator 并在 RecyclerView 中用作默认项目动画。

所有这些都由 DefaultItemAnimator 类提供，它扩展了 RecyclerView.ItemAnimator 并在 RecyclerView中用作默认项目动画。

如果你有兴趣了解它如何工作，我建议你检查它的源代码：[DefaultItemAnimator](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/master/v7/recyclerview/src/android/support/v7/widget/DefaultItemAnimator.java)。它只有600行代码非常具有描述性和清晰性。 在非常简短的解释中，它有两个主要步骤：
- 准备每个项目的动画，并把它们放到挂起动画的 ArrayList 中。
- 播放全部项目

基于应用程序示例，让我们看看它是如何工作的删除动画：
![|center](http://frogermcs.github.io/images/23/remove_item.gif)
（步骤1-3可以以不同的或混合的顺序执行）
1. 准备“添加”动画（对于尚不可见的项目，当有更多的空间时应该出现在屏幕上）
2. 准备“删除”动画（点击项目）
3. 准备“移动”动画（对于所有应移到正确位置的项目）
4. 播放所有这些动画（多一些逻辑，例如延迟移动动画，直到删除动画完成 - 查看[ViewCompat.postOnAnimationDelayed（）](http://developer.android.com/reference/android/support/v4/view/ViewCompat.html#postOnAnimationDelayed(android.view.View%2C%20java.lang.Runnable%2C%20long))了解详细信息） PredictiveItemAnimations

## PredictiveItemAnimations
从用户体验角度来看，它看起来非常直观，从实现方面来说，我们应该关心一件事情 - 在屏幕上看不到的项目的动画，但是在执行添加/删除操作后应该出现。

我们必须记住，在技术上只有在屏幕上可见的项目是存活和存在 RecyclerView。 因此，如果我们前面介绍过（删除项目操作），最底层的元素没有被移位，因为它以前不存在。 在这种情况下，我们应该播放外观动画...

或者

我们可以预测这个元素来自哪里。 值得回顾的是，RecyclerView.ItemAnimator 只负责开始和最终状态之间的动画。 视图定位的责任在于 LinearLayoutManager（或任何其他 RecyclerView.LayoutManager 类）。

LayoutManager 具有 public 方法 supportsPredictiveItemAnimations（），默认返回 false 值。 当确定我们的 LayoutManager 符合要求时将其设置为 true 将有助于播放移位动画而不是外观动画。 是的 -  LinearLayoutManager 支持它。

在我们的代码中，我们以这种方式启用它：
```java
//MainActivity.java
LinearLayoutManager layoutManager = new LinearLayoutManager(this) {
    @Override
    public boolean supportsPredictiveItemAnimations() {
        return cbPredictive.isChecked();
    }
};
rvColors.setLayoutManager(layoutManager);
```
结果在这里是可见的（仔细看看从屏幕底部移开的最后一个元素）：
![|center](http://frogermcs.github.io/images/23/remove_item_predictive.gif)
如果您对如何在底层工作感兴趣，或者您正在构建自己的 LayoutManager 并想知道需求，那么可以从以下方法文档中获得一个好的起点：[supportsPredictiveItemAnimations（）](http://developer.android.com/reference/android/support/v7/widget/LinearLayoutManager.html#supportsPredictiveItemAnimations())。

## 自定义更改项目动画
在 DefaultItemAnimator 中更改动画会在项目视图的前后状态之间播放淡入淡出动画。基于示例应用程序，我们想要实现更复杂的东西：

![|center](http://frogermcs.github.io/images/23/change_item.gif)
我们的项目应该扭曲文本从一个到另一个值和背景应该使其颜色的方式动画：开始颜色 - >黑色 - >结束颜色。为了实现这个，我们将扩展 DefaultItemAnimator。

## 项目播放动画的原理
更改项目动画是可视化 ItemAnimator 如何工作的最简单的方法。 值得一提的是，它与添加或删除动画没有太大的区别。

[这是](https://github.com/frogermcs/RecyclerViewAnimations/blob/master/app/src/main/java/frogermcs/io/recyclerviewanimations/MyItemAnimator.java) MyItemAnimator 的最终实现（查看它的全貌）。


1）通知变更
那好吧，我们从头开始。我们的列表显示，我们的 RecyclerView 使用自定义项目动画（MyItemAnimator）：
![|center](http://frogermcs.github.io/images/23/list.png)
在我们点击项目之后，这个代码被执行：
```java
//ColorsAdapter.java
public void changeItemAtPosition(int position) {
    colors.set(position, ColorsHelper.getRandomColor());
    notifyItemChanged(position);
}
```
从 notifyItemChanged () 方法开始（它直接调用 notifyItemRangeChanged (); 给定的点击位置和 itemsCount = 1），我们通知 RecyclerView 哪个项目应该更新。

2）记录最近的状态
现在我们的动画通过调用[ recordPreLayoutInformation(RecyclerView.State state, RecyclerView.ViewHolder viewHolder, int changeFlags, List payloads) ](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ItemAnimator.html#recordPreLayoutInformation(android.support.v7.widget.RecyclerView.State%2C%20android.support.v7.widget.RecyclerView.ViewHolder%2C%20int%2C%20java.util.List%3Cjava.lang.Object%3E))来开始记录最近的视图状态（更多详细的描述参见文档）。 在这个方法中，我们可以访问已更改视图的当前 ViewHolder。 这是保存状态的最佳地点（尤其是那些应该动画的属性）。

状态可以保存在返回的 ItemHolderInfo 对象中

这是我们的实现：
```java
private class ColorTextInfo extends ItemHolderInfo {
    int color;
    String text;
    
    //...
    
}
```
默认 ItemHolderInfo 保留视图边界的数据。此外，我们需要背景颜色和文字，这将是播放动画。

 recordPreLayoutInformation () 收集所有显示视图的数据，即使它们没有改变（在这种情况下，param changeFlags 被设置为 ViewHolder.FLAG_BOUND，它等于0）。

其他标志值描述请求的操作（更改动画具有值2 - ViewHolder.FLAG_UPDATE）。

在这一刻，我们的观点仍然是这样的：
![|center](http://frogermcs.github.io/images/23/item-start.png)

3）绑定新视图
现在，我们的适配器被要求将 ViewHolder 绑定到新项目的值（最终状态）：
```java

@Override
public void onBindViewHolder(ColorViewHolder holder, int position) {
    int color = colors.get(position);
    holder.itemView.setBackgroundColor(color);
    holder.tvColor.setText("#" + Integer.toHexString(color));
}
```
这意味着在这一步我们的视图被改变，看起来像这里：
![|center](http://frogermcs.github.io/images/23/item-end.png)

4）记录新的状态
现在回到 ItemAnimator。 是时候通过调用 [recordPostLayoutInformation（RecyclerView.State状态，RecyclerView.ViewHolder viewHolder）](http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ItemAnimator.html#recordPostLayoutInformation(android.support.v7.widget.RecyclerView.State%2C%20android.support.v7.widget.RecyclerView.ViewHolder)) 来记录我们视图的最终状态（更多细节请参阅文档）。 我们再次访问 ViewHolder，但是这次有了新的观点。 我们再次可以在 ItemHolderInfo（我们的 ColorTextInfo）中保存重要的细节。

4）播放（或只是准备动画）
RecyclerView.ItemAnimator 有一个动画不同操作的方法列表。在我们的例子中，animateChange () 将被调用。这是准备我们的动画的最好的地方。我们有4个参数帮助这个：

- RecyclerView.ViewHolder oldHolder
- RecyclerView.ViewHolder newHolder
- ItemHolderInfo preInfo
- ItemHolderInfo postInfo

oldHolder 和 newHolder 对象代表布局前后的项目。在我们的情况下，两者是相同的对象，因为：
```java
@Override
public boolean canReuseUpdatedViewHolder(RecyclerView.ViewHolder viewHolder) {
    return true;
}
```
这意味着对于动画我们不需要有 ViewHolder 的分离对象（因为我们只能基于 ItemHolderInfo 对象来处理它）。

preInfo 和 postInfo 对象来自 recordPreLayoutInformation () 和 recordPostLayoutInformation ()。

如你所知，我们的观点已经束缚到最后的状态。我们在 animateChange () 方法中所要做的就是将视图设置为初始状态（保存在 preInfo 中），并准备和可选地播放动画到最终状态。

**为什么选择？**
animateChange () 方法返回布尔值，告诉 RecyclerView 我们的动画是否已经执行（false）或者只是设置，保存并等待执行（true）。

如果错误流程结束。我们只需要通过调用 dispatchAnimationFinished（）来记得在我们的动画完成时告诉 RecyclerView：
```java
overallAnim.addListener(new AnimatorListenerAdapter() {
    @Override
    public void onAnimationEnd(Animator animation) {
        dispatchAnimationFinished(newHolder);
    }
});
```
它会告诉 ViewHolder 已准备好重用。 否则（animateChange () 返回 true）：

5）播放所有等待的动画
有一个机会，我们有一次需要执行多个动画（想想在添加/删除操作中所有的轮班，出现或失踪）。 在这种情况下，animate ... () 方法应该以某种方式保存挂起的动画（DefaultItemAnimator 使用简单的 ArrayLists），然后在 runPendingAnimations () 中一起播放所有的动画。

还有一些额外的步骤（例如，取消动画或动画），但我会留下你自己来解决这个问题。

整个描述流程很短的版本是这样的：
![](http://frogermcs.github.io/images/23/flow.png)

## 示例应用程序和源代码
这个项目再现了由 Chet Haase 和 Yigit Boyar 演讲的源代码：[RecyclerView 动画和背后原理](https://www.youtube.com/watch?v=imsr8NrIAMs)。这不是来自 Google 员工的官方源代码，但我相信它非常相似。

这里没有描述，但是我们的代码可以处理用户通过取消最近的动画并从上一次停止播放新的动画重复点击视图的情况。这也是演示谈话中提出的。

这里是显示我们的应用程序的视频：	

<iframe width="560" height="315" src="https://youtu.be/HMd_aaFBM20" frameborder="0" allowfullscreen></iframe>

## 示例代码 
所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/RecyclerViewAnimations)上可用。




