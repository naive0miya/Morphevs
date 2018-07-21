# 解剖 RecyclerView: 探索 ViewHolder (续)

> 原文 (Medium)：[Anatomy of RecyclerView: a Search for a ViewHolder (continued)](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91)
>
> 作者：[Pavel Shmakov](https://android.jlelse.eu/@pavelshmakov?source=post_header_lockup)

[TOC]

我们继续讨论 RecyclerView 搜索给定位置的视图的方式，是从这里开始的：[第1部分](https://medium.com/@pavelshmakov/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#.smf5acu1n)

为了便于参考，我在这里复制了 RecyclerView 的视图搜索算法的大纲。

1. 搜索 RecyclerView 中需要改变的 ViewHolder 列表(即一级缓存) 
2. 搜索未与 RecyclerView 分离的 ViewHolder 列表(即一级缓存) 
3. 搜索 RecyclerView 未删除的隐藏 ViewHolder 列表
4. 搜索 RecyclerView 的 ViewHolder 缓存列表(即一级缓存) 
5. 如果 adapter 具有稳定的 id ，则可以为给定 id 重新搜索未与 RecyclerView 分离的 ViewHolder 列表和 RecyclerView 的 ViewHolder 缓存列表。
6. 搜索用户设置的 RecyclerView 的 ViewHolder 缓存列表扩展(即二级缓存) 
7. 搜索 RecyclerView 的 ViewHolder 缓存池(即三级缓存) 

到目前为止，我们已经介绍了 RecycledViewPool 和视图缓存。

## ViewCacheExtension

ViewCacheExtension 的想法很有吸引力：它是一种按你想要的方式行事的缓存。除非你通过 RecyclerView 的 setViewCacheExtension ( ) 方法明确设置并实现一个简单的接口，否则它就不存在：

```java
View getViewForPositionAndType(Recycler recycler, int position, 
                               int type);
```

看起来很好，很简单。 但仔细观察后发现，它既不简单也不友好。 

首先，你不能只是返回任何你想要的视图。你需要返回某个 ViewHolder 拥有的 View 。当创建一个 ViewHolder 时，对它的引用被存储在 View 的 LayoutParams 中， 这就是当你返回 ViewCacheExtension 的视图时，RecyclerView 就会在这里查看。如果它没有在那里找到 ViewHolder，那么你会崩溃。这很奇怪，不是吗？为什么需要返回视图，实际上需要的是一个 ViewHolder？

但是还有一个更大的问题。请记住，ViewHolders 不是无状态的，它们是可变的，具有由 RecyclerView 在内部管理的位置，标志和其他内容。如果你返回与 ViewHolder 相关联的 View，该 ViewHolder 属于与参数中传递的位置不同的某个位置，那么 RecyclerView 和/或 LayoutManager 会混淆并产生错误。

这个位置问题似乎超出了你的控制范围。让我们仔细看看它来自哪里。当你添加或删除一些项目。在布局过程开始时，在 LayoutManager 执行任何工作之前，要求 AdapterHelper 处理自上次布局以来发生的适配器更改（我们将在本系列后面的部分介绍 AdapterHelper）。然后它会回调 RecyclerView 以偏移应用于 ViewHolder 的位置。RecyclerView 通过当前显示的视图的 ViewHolders 循环并应用偏移量（请参阅 offsetPositionsForInsert ( ) 和类似的方法）。但它并不知道你在某处缓存的 ViewHolders，所以它不能抵消它们的位置！而且你也不能，因为这个位置是封装在本地化的。

由于这些限制，ViewCacheExtension 的用处似乎非常有限。对我来说，想到一个有效的例子并给予一些好处并不容易，但是事实就是如此。 

ViewCacheExtension 例子

想象一下你有一些 “特殊” 项目，其行为如下：

1. 他们的位置是固定的。例如，例如，这些是广告，你必须把它们放在特定的位置。
2. 它们从来不会在视觉上改变。 
3. 它们的数量是合理的，所以可以把所有的视图保存在内存中。

你想避免的是重新绑定这些项目。比如说，你们有3个项目，它们在列表上相距甚远。然后通常会有一个 ViewHolder，在你滚动的这三个项目之间将会反弹。重新绑定可能会损害滚动的平滑性，因此你宁愿将全部3个视图保留在内存中。池不能帮助你，因为进入池总是意味着稍后重新绑定，而视图缓存也不能使用，因为它不关心视图类型。

ViewCacheExtension 来拯救！[¹](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#3048)

你可以这样写一些代码: 

```java
SparseArray<View> specials = new SparseArray<>();
...

recyclerView.getRecycledViewPool().setMaxRecycledViews(SPECIAL, 0);

recyclerView.setViewCacheExtension(new RecyclerView.ViewCacheExtension() {
   @Override
   public View getViewForPositionAndType(RecyclerView.Recycler recycler,
                                         int position, int type) {
       return type == SPECIAL ? specials.get(position) : null;
   }
});

...
class SpecialViewHolder extends RecyclerView.ViewHolder {
	   ...		
   public void bindTo(int position) {
       ...
       specials.put(position, itemView);
   }
}
```

它实际上是有效的: 每个"特殊"的 ViewHolder 都是被创建的，并且只与数据绑定一次。 

但是，一旦你引入"特殊"项目的任何位置变化，例如由于删除或添加周围的项目，它们都会崩溃。 你可能认为重新计算 SparseArray 的键就足够了，但事实并非如此: 正如我们之前讨论的那样，整个整体都保持着陈旧的状态。 

## 隐藏视图

我们的下一站是大纲的第三点，神秘的 “搜索 RecyclerView 未删除的隐藏 ViewHolder 列表”。

什么是隐藏的视图？它们是目前正在走出 RecyclerView 边界的视图。他们仍然被保留作为 RecyclerView 的孩子，但仅限于动画它们消失的目的。从 LayoutManager 的角度来看，它们已经消失了，不应该包含在任何计算中。例如，如果调用 LayoutManager 的 getChildAt ( ) 方法，而一些视图由于项目被移除而消失，这个视图不会含在任何计算，这种情况是有道理的。

所有对来自 LayoutManager 的 getChildAt ( )，getChildCount ( )，addView ( ) 等所有的调用都是通过 ChildHelper 进行路由，然后再应用到 RecyclerView 的实际子节点上。ChildHelper 可能是 RecyclerView 家族中最简单和最好的类。它的责任是重新计算非隐藏子类名单和所有子类名单之间的指数。[²](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#95e3)

现在，在我刚才所说的事实之后，隐藏的视图被用于寻找 ViewHolder，这一事实似乎更加神秘。  记住，虽然我们寻找一个视图给 LayoutManager，但是 LayoutManager 不应该知道隐藏的视图！ 

际上我并不知道它们，但是 RecyclerView 知道，在非常特殊的情况下可以反馈给 LayoutManager。 这有点奇怪,"从隐藏的视图反弹"机制只是处理以下情况所必需的。设想我们插入一个项目，然后快速，在插入动画完成之前，删除该项: 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovlgzg0wj30m807tdg9.jpg)

我们想看到的是 b 从 c 被移除的那一刻到底在哪里。但是在那个时候 b 是一个隐藏的视图！如果我们忽略它，就会在现有的 b 之下创建一个新的 b，它们会重叠，因为新的 b 上升，旧的继续下降。 为了避免这种史诗般的错误，在搜索 ViewHolder 的早期步骤之一 RecyclerView 询问 ChildHelper 是否有一个合适的隐藏视图。通过合适，它意味着与我们需要的位置相关联的视图，具有正确的视图类型，并且它被隐藏的原因不是移除（显然，我们不应该将它们带回生命中）。

如果有这样的视图，RecyclerView 将它返回给 LayoutManager，并将其添加到预先布局，以标记它应该动画的位置(参见记录 animationinfoifbouncedhiddenview ( ))。现在，如果你一直在关注我，在读完前一句话后，你应该有一种强烈的 “wtf” 感觉。在布局之前或之后添加内容必须是 LayoutManager 的业务，现在 RecyclerView 本身正在添加一些预先布局的东西。是的，这个机制看起来非常陈腐和不稳定，这是一个足够好的理由去了解它。 

## Scrap

在搜索 ViewHolder 时，Scrap 列表是第一个查找 ViewHolder 的地方，而且它和 Pool 或者 View Cache 有很大的不同。 

Scrap 仅在布局过程中不为空。当 LayoutManager 启动一个布局（预处理或后处理）时，当前添加到布局的所有视图都将转储到 Scrap 中（请参见 LinearLayoutManager＃onLayoutChildren ( ) 中的 detachAndScrapAttachedViews ( ) 调用）。然后，LayoutManager 逐个检索视图，除非 View 发生了什么 ，否则它会从 Scrap 中回来：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovljtkv4j30m808twew.jpg)

在这个例子中，我们删除 b，这会导致重新布局。 a，b 和 c 被丢弃，抛入 scrap；然后 a 和 c 从 scrap 中回收。

作为一个小题外话，b 会怎么样？ 在布局过程的最后阶段，RecyclerView 发现 b 从未被添加到布局后。它解开它，使它隐藏起来，并开始消失动画。当动画结束时，b 进入池。

为什么 LayoutManager 不能立即使用没有变化的孩子？我相信 scrap 存在的原因是 LayoutManager 和 RecyclerView 之间的关注点分离。LayoutManager 不应该知道某个特定的孩子是否适合留在身边，或者应该去池或其他地方看看。这是 Recycler 的业务。

除了只在布局中是非空的，似乎还有另一个与 scrap 相关的不变量：所有 scrap 的视图都从 RecyclerView 中分离出来。ViewGroup 的 attachView ( ) 和 detachView ( ) 方法类似于 addView ( ) 和 removeView ( ) ，但不会导致布局请求或者失效。他们只是把一个孩子从 ViewGroup 的孩子列表中删除，并且将这个孩子的父级设置为 null。 分离状态必须是暂时的，然后是附加或移除。在计算一个新的布局的时候，你已经有一大堆的孩子了，首先分离他们是很方便的，这就是 RecyclerView 所做的。 

附加vs更改 scrap

在回收站中，你可以看到两个单独的 scrap 容器：mAttachedScrap 和 mChangedScrap。为什么我们需要两个？

我们来看看如何选择两个废料容器中的一个。 ViewHolder 进入 mChangedScrap 只有在关联的项目被更改（notifyItemChanged ( ) 或 notifyItemRangeChanged ( ) 被调用）且 ItemAnimator 在被询问是否可以重新使用 ViewHolder 时说“不”，否则，ViewHolder 会进入mAttachedScrap 。这个"不"意味着我们想要一个变化动画，其中一个视图取代另一个视图，例如一个交叉淡出动画。 "是"将意味着变化动画将发生在一个视图。 

mAttachedScrap 和 mChangedScrap 的使用只有一个区别：mAttachedScrap 可用于布局前和布局后，而 mChangedScrap  仅限用于布局前。这是有道理的：在布局后，一个新的 ViewHolder 应该替换一个 “已更改” 的，所以 mChangedScrap 在布局后没有用处。如预期的那样，当变化动画完成时 ，mChangedScrap 将转储到池中。[³](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#f22f)

默认的 ItemAnimator 可以在3种情况下重用更新的 ViewHolder：

1. 调用 setSupportsChangeAnimations（false）。
2. 调用 notifyDataSetChanged ( ) 而不是 notifyItemChanged ( ) 或 notifyItemRangeChanged ( ) 。
3. 提供了变更的 payload：即 adapter.notifyItemChanged（index，anyObject）。

最后一个情况展示了一个很好的方法，当你只是想改变一些内在的元素时，避免创建和 / 或绑定新的 ViewHolder。 

## 稳定的 id

关于稳定 id 的特性，最重要的是它只影响 notifyDataSetChanged ( ) 在通知数据库调用后的行为。 

当我在前面讨论 scrap 时，我说在新的布局开始的时候，所有的子类都被扔进 scrap 容器。实际上有一个例外。如果你调用 notifyDataSetChanged ( ) 而且没有稳定的 id，那么 RecyclerView 不知道发生了什么，到底发生了什么变化，所以它假设一切都已改变，每个 ViewHolder 都是无效的，并被扔进池而不是 scrap。因此，我们有一张我们之前已经看到的照片：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovlnbh3hj30m80doq43.jpg)

如果你的 id 是稳定的，那么情况就完全不同了: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovlr46j2j30m80demyh.jpg)

因此，ViewHolders 转储 scrap 而不是池，然后搜索一个带有特定 id 的 ViewHolder (通过 getItemId ( ) 你的适配器的方法检索) ，而不是一个特定的位置。 

有什么好处呢？ 第一个问题是我们没有溢出池的问题，所以除非必要，不会创建新的 ViewHolders 。 然而，ViewHolders 还是会恢复的，因为这个 id 没有改变并不意味着内容没有改变。 

第二个好处，也是一个更大的好处，就是我们得到了动画！ 在上面的图片中，我将第4项移至位置6。通常情况下，你必须调用 notifyItemMoved（4,6）来获取移动动画，但对于稳定的 id，notifyDataSetChanged ( ) 就足够了。可以看到具有特定 id 的视图位于以前的布局中，它出现在新布局中的位置。 或者它可以看到某个 id 不再存在，所以项目必须已经被删除，等等。 

但需要注意的是，你只能通过这种方式获得简单(非预测)的动画。事实上，预测动画在这里是如何起作用的？ 如果我们在新的布局中看到一些 id，这在之前的布局中是不存在的，那么我们怎么知道它是一个插入的项目还是一个从某个地方进入的项目，而在后一种情况下，它到底是从哪里来的呢？ 通常情况下，这些问题的答案都会在预先布局中找到，根据适配器的更改，这些答案超出了 recycleroview 的界限，但是现在我们不知道这些变化是什么。 

总的来说，稳定 id 的作用似乎是有限的。 不过，我可以想出一个好的理由来使用它: 如果你正在从 ListView 迁移到 RecyclerView，那么将所有 notifyDataSetChanged ( ) 都转换为特定更改通知可能会很痛苦。 在这种情况下，稳定的 id 会给你一个简单的免费回收视图动画。 作为下一步，你可能想尝试一下 [DiffUtil](https://medium.com/@nullthemall/diffutil-is-a-must-797502bc1149)。 

这就是我们进入 RecyclerView 内部工作的第一部分。 

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#1186) ¹ 我们还假设在编译时还不知道"特殊"项的数量，否则你可以通过为每个视图分配一个独立的视图类型来解决问题，而忽略除第一个视图外的绑定调用。 如果数量未知，这种方法仍然可行，但是混乱。 。

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#d736) ² 如果你看看 ChildHelper 的代码，那么 Bucket 内部类似乎在第一眼看起来很可怕。 事实上，它只是一个可扩展的 bit 字符串，其中一些字符串标记隐藏视图的索引。 

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91#862e) ³ 这是一个叫做 dispatchAnimationFinished ( ) 的 ItemAnimator 的结果。 

