# 解剖 RecyclerView: 探索 ViewHolder

> 原文 (Medium)：[Anatomy of RecyclerView: a Search for a ViewHolder](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714)
>
> 作者：[Pavel Shmakov](https://android.jlelse.eu/@pavelshmakov?source=post_header_lockup)

[TOC]

## 介绍

在这一系列文章中，我将分享我对 RecyclerView 内部工作机制的了解。为什么？ 想想吧: 每个现代的 Android 应用程序几乎都需要 RecyclerView，因此开发者使用它的方式影响了数百万用户的体验。然而，我们有什么样的教育材料，我们有吗？你肯定可以找到一些关于如何使用 RecyclerView 的基础教程，但是它是如何工作的呢？ “黑盒子” 方法绝对是不够好的，尤其是当你正在做复杂的定制或者优化性能的时候 ¹ 。那里的 “最深入” 的材料可能是 RecyclerView 在 Google I / O 2016上的演讲 [RecyclerView ins and outs](https://www.youtube.com/watch?v=LqBlYJTfLP4)，但是说真的，这还远远不够， 这只是冰山一角。我的目标是更深入。 

我假设读者具有 RecyclerView 的基本知识，比如知道 LayoutManager 是什么，如何通知 Adapter 对数据中的特定更改或如何使用项目视图类型。

在第一部分中，我们将通过 RecyclerView 中的一个方法来弄清楚发生了什么：  getViewByPosition ( ) 。这是源代码中最重要的部分之一，通过研究，我们将学习 RecyclerView 的许多方面，例如 ViewHolder 回收，隐藏视图，预测动画和稳定的 id 。你可能会惊讶地看到这里的预测动画 。不过，尽管 Google 的人尽最大努力去分解 RecyclerView 不同组件的责任，但他们之间仍然存在很多“知识”，预测动画就是其中之一。没有办法避免谈论他们在某一点或另一点。 

因此，在布局物品时，LayoutManager 会询问 RecyclerView “请给我一个位置8的视图”。以下是 RecycleView 对此的回应：

1. 搜索 RecyclerView 中需要改变的 ViewHolder 列表(即一级缓存) 
2. 搜索未与 RecyclerView 分离的 ViewHolder 列表(即一级缓存) 
3. 搜索 RecyclerView 未删除的隐藏 ViewHolder 列表
4. 搜索 RecyclerView 的 ViewHolder 缓存列表(即一级缓存) 
5. 如果 adapter 具有稳定的 id ，则可以为给定 id 重新搜索未与 RecyclerView 分离的 ViewHolder 列表和 RecyclerView 的 ViewHolder 缓存列表。
6. 搜索用户设置的 RecyclerView 的 ViewHolder 缓存列表扩展(即二级缓存) 
7. 搜索 RecyclerView 的 ViewHolder 缓存池(即三级缓存) 

如果它在所有这些地方都找不到合适的视图，则通过调用适配器 onCreateViewHolder ( ) 方法创建一个视图。 然后，它通过 onBindViewHolder ( ) 绑定视图，并最终返回它。 

正如你所看到的，这里发生了很多事情，远远不止是一个可重复使用的 ViewHolders 池。我们的目标是弄清楚所有这些缓存是关于什么的，它们如何工作以及为什么需要它们。 我们将一个一个地检查它们(按照我认为最好的顺序) ，我们的第一站是 RecycledViewPool。 

## RecycledViewPool

我们希望能够回答一些关于每种缓存的问题: 它的备份数据结构是什么，在这种情况下，viewhoders 存储在哪里并从中检索，最重要的是，它的目的是什么。 

你可能非常了解池的用途：当滚动时，比如说， 向下滚动时，顶部消失的视图会回收到池中，以便重新用于从底部出现的视图。我们将讨论 ViewHolders 稍后访问池的其他场景。但首先让我们来看看一些 RecycledViewPool 的代码（这是 RecyclerView.Recycler 的一个内部类）：

```java
public static class RecycledViewPool {
    private SparseArray<ArrayList<ViewHolder>> mScrap =
                   new SparseArray<>();
    private SparseIntArray mMaxScrap = new SparseIntArray();
    …
    public ViewHolder getRecycledView(int viewType) {
        ArrayList<ViewHolder> scrapHeap = mScrap.get(viewType);
        …
```

首先，不要让名称 mScrap 迷惑你 - 这与上面列表中提到的分离的或改变的 ViewHolder 列表无关。

我们看到每种视图类型都有自己的 ViewHolders 池。当 RecyclerView 在搜索 ViewHolder 的过程中用完了所有其他可能性时，它要求池为 ViewHolder 提供正确的视图类型; 在这一点上，视图类型是唯一重要的事情 。

现在，每种视图类型都有自己的容量。默认情况下它是5，但你可以这样改变它：

```java
recyclerView.getRecycledViewPool()
            .setMaxRecycledViews(SOME_VIEW_TYPE, POOL_CAPACITY);
```

这是灵活性非常重要的一点。如果屏幕上有几十个相同类型的项目，这些项目经常同时更改， 则为该视图类型设置较大的池。如果你知道，某些视图类型的项目非常罕见，以至于它们永远不会在屏幕上出现超过一个的数量，请将该视图类型的池大小设置为1。否则，池迟早会装满 5个这样 的物品，其中4个会在那里闲置，这是浪费内存。

getRecyclerView（），putRecycledView（），clear（）方法是公共的，所以你可以操纵池的内容。手动使用 putRecycledView（），例如为了事先准备一些 ViewHolders，是一个坏主意。你应该只在适配器的 onCreateViewHolder（）方法中创建 ViewHolder，否则 ViewHolders 可能会出现 在 RecyclerView 不期望的状态 [²](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#fe2a)。

另一个很酷的特性是 ，getter getRecycledViewPool ( ) 和 setter setRecycledViewPool ( ) ，因此你可以为多个 RecycleView 重用单个池。

最后，我会注意到，每个视图类型的池作为一个堆栈（后进先出）。我们稍后会看到为什么这是最好的。

何时把 ViewHolders 放入池

现在让我们来讨论什么时候把 ViewHolders 扔入池中的问题。有5种情况：

1. 在滚动期间，视图在 RecyclerView 显示的边界内。
2. 数据已更改，以至于视图不再被看到。当消失动画结束时，扔入池中。
3. 视图缓存中的项被更新或删除。
4. 在搜索 ViewHolder 时，我们发现其中一个位置是我们想要的，但是由于错误的视图类型或 id (如果适配器有稳定 id) ，所以不适合使用。 
5. LayoutManager 在预布局中添加了一个视图，但未在布局后添加该视图。

前两种情况是显而易见的。不过，需要注意的一点是，场景2不仅是通过删除有关项目而触发的，而且，例如，通过插入其他项目，将给定项推出边界。 

其他的场景需要一些批注。我们尚未覆盖视图缓存和 scrap，但场景3和4背后的想法很简单。池是我们知道的 “脏” 且需要重新绑定的视图的地方。除池外，所有缓存中的 ViewHolders 都保留其某些状态（最重要的是位置）。所有这些缓存都按照位置进行搜索，希望某些 ViewHolder 可以按原样重用。相比之下，当一个视图进入池时，它的状态（所有的标志，位置等）被清除。唯一剩下的就是关联视图和视图类型。正如我们所知，池是通过视图类型进行搜索的，当在那里找到 ViewHolder 时，开始新的生命。

考虑到这种情况，场景3和4不应该是一个谜: 例如，如果我们在视图缓存中看到一个项目被移除，那么在缓存中再次保存它是没有意义的，因为它不会被重新使用作为相同的位置。 但是完全扔掉不是一件好事，所以我们把它扔进游泳池。 

最后一个场景要求我们知道什么是布局前（pre-layout）和布局后（post-layout ）。好吧，让我们继续努力，把这个问题搞清楚！ 这个场景绝对不是 pre / post - layout 机制最关键的方面，但是这个机制在总体上是非常重要的，并且在 RecyclerView 的每个部分都显示出来，所以我们无论如何都要知道。 

## pre-layout， post-layout 和预测动画

考虑一个情景，我们有项目 a，b 和 c，其中 a 和 b 正好铺满屏幕。我们删除了 b，这使 c 进入视图: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovkjpbcuj30m806jmxa.jpg)

我们希望看到的是 c 平稳地从底部滑动到它的新地方。 但是这怎么可能发生呢？ 我们从新的布局中知道 c 的最终位置，但是我们怎么知道它应该从哪里滑动呢？ RecyclerView 或 ItemAnimator 通过查看新布局来假设 c 应该从底部出现的，这将是错误的。我们可能有一些自定义的 LayoutManager，它应该从侧面或者其他什么地方来。所以我们需要来自 LayoutManager 的更多帮助。我们可以使用以前的布局吗？No，那里没有 c。在这一点上，没有一个新的 b 将被删除，因此布局管理器正确地考虑 c 是浪费资源。

Google 提供的解决方案如下。在适配器发生更改后，RecyclerView 从 LayoutManager 请求两个布局。第一个 - 预布局，在前面的适配器状态中列出项目，但使用适配器更改作为提示，这可能是一个好的想法，提出一些额外的视图。在我们的例子中，既然我们现在知道 b 正在被删除，我们另外布置 c，尽管事实上它是超出显示界限的。第二个 - 布局后，只是对应于更改后的适配器状态的正常布局。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovko2t13j30m80a5aap.jpg)

现在，通过比较布局前和布局后 c 的位置，我们可以对其外观进行动画处理。 

这种动画 - 当动画视图不在以前的布局或新的布局中时 - 称为预测动画，这是 RecyclerView 中最重要的概念之一。我们将在本系列的后面部分更详细地讨论它。但是，现在让我们快速查看另一个示例：如果 b 被更改而不是被删除，该怎么办？

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovkqbvclj30m809ojrt.jpg)

这可能会让人感到意外，但 LayoutManager 仍然在布局前阶段展示 c。因为也许 b 的变化会使它变得更小，谁知道呢？如果 b 变小，c 可能会从底部弹出，所以我们最好在布局前进行布局。但后来，在后期布局中，似乎并非如此，我们只是改变了 b 中的一些 TextView。所以 c 是不需要的 ，因而被扔进池中。这就是场景5中让自己进入池。现在希望你们清楚了，我们可以回到 RecycledViewPool。

## RecycledViewPool

当我们遇到 ViewHolder 应该进入池的场景之一时，在途中仍然存在两个障碍物：它可能无法回收，并且 View 可能处于瞬态状态。

可回收

可回收性只是 ViewHolder 中的一个标志，你可以使用 ViewHolder 类的 setIsRecyclable ( ) 方法来操作。 RecycleView 也使用它，使 ViewHolders 在动画过程中不可回收。

从不同的独立地点操纵单个标志通常是一个糟糕的主意。例如，RecyclerView 在动画结束时调用 setIsRecyclable（true），由于某些原因，你不希望它可以被回收。但事实上在这种情况下并不会中断，因为对 setIsRecyclable ( ) 的调用是成对的。也就是说，如果你调用 setIsRecyclable（false）两次，那么只调用 setIsRecyclable（true）一次不会使 ViewHolder 可回收，你也需要调用它两次。

瞬态状态

视图的瞬态状态是非常类似的事情。它是由 setHasTransientState ( ) 方法操作的视图中的一个标志，调用也是成对的。视图类本身不使用该标志，只是保存它。 它提示了  ListView  和 recyclareview 这样的小工具，在这个时候最好不要重用这个视图来获取新内容。 

你可以自己设置这个标志，但是 ViewPropertyAnimator（也就是当你做一些 View.animate ( ) 时）会自动在开始时将它设置为 true，并在动画结束时自动设置为 false [⁴](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#d49c) 。请注意，如果你使用例如 ValueAnimator 为视图设置动画，则必须自行管理瞬态状态。

关于瞬态的最后一点要注意的是，它是从孩子传播到父母，一直到根视图。因此，如果你为列表中的项目的某些内部视图制作动画，则不仅该视图，而且 ViewHolder 保存引用的项目的根视图会转变为瞬态。

OnFailedToRecycleView

如果要回收的 ViewHolder 失败，可回收性或瞬态检查失败，则调用适配器的 onFailedToRecycleView ( ) 方法。现在，这是一个非常重要的点：这种方法不仅仅是一个事件的通知，而是一个关于如何处理这种情况的问题。

从 onFailedToRecycledView ( ) 返回 true 意味着 “无论如何回收它”。在合适的情况下，一种情况是在绑定新项目时清除所有动画和其他麻烦来源。或者，你可以在 onFailedToRecycledView ( ) 方法中正确处理这些事情。

你不应该做的是完全忽略 onFailedToRecycledView ( ) 。有一种情况可能会伤害到你，那就是下面这个场景。 想象一下，当项目进入视野时，你正在淡入物品中的图像。如果用户以足够快的速度滚动列表，则图像在离开显示界限还没有完成淡入动画，从而导致 ViewHolders 无法回收。所以你会有一个滞后的滚动，并且最重要的是，新的 ViewHolders 将不断被创建，导致内存混乱。对于说俄语的读者，我推荐 Konstantin Zaikin 做这个演讲，其中，这种场景在行动中显示：https://events.yandex.ru/lib/talks/3456/

成功的回收再利用 ViewHolder 导致了对 onViewRecycled ( ) 方法的调用，这是一个释放重量资源的好地方，比如图像。 记住，有些 ViewHolder 实例最终可能会长时间不使用，这可能是对内存的巨大浪费。 

我们现在转到下一个缓存 - 视图缓存。

## View Cache

当我说 “视图缓存” 或只是 “缓存” 我所指的是 RecyclerView.Recycler 类中找到的 mCachedViews 字段。它在代码的一些注释中也被称为 “第一级缓存”。

这只是 ViewHolders 的一个 ArrayList，在这里没有视图类型的分割。默认容量是 2，你可以通过 RecyclerView 的 setItemViewCacheSize（）方法来调整它。

正如我之前提到的，池和其他缓存（包括视图缓存）之间最重要的区别在于，其他缓存会搜索与给定位置关联的 ViewHolder，而池按照视图类型搜索。当 ViewHolder 位于视图缓存中时，我们希望在不重新绑定的情况下，以 “原样” 的方式重用它，并将其放入缓存中的位置。所以让我们清楚地说明这个区别：

- 如果 ViewHolder 无法被发现，它将被创建并绑定。
- 如果在池中找到 ViewHolder，它将被绑定。
- ViewHolder 被发现在缓存中，不用做什么。

在这一点上，一件重要的事情变得非常清楚： ViewHolder 绑定和回收到池中的（onViewRecycled ( )）与进入和离开可见边界的列表中的项不同。当它离开时，它的 ViewHolder 可以去视图缓存而不是池，并且当它出现时，ViewHolder 有时从视图缓存中检索并且不需要绑定。 如果你需要跟踪屏幕上的项目是否存在， 请使用适配器的 onViewAttachedToWindow ( ) 和 onViewDetachedFromWindow ( ) 回调。

填充池和缓存

现在，我们来谈谈下一个问题: 如何在视图缓存中结束 ViewHolder？ 当我谈到导致进入池的场景时，我实际上欺骗了你一点点。在这些场景中（除了第三个场景），ViewHolder 会转到缓存或池。⁵

让我说明选择缓存还是池的规则。比方说，我们最初有空的池和缓存，并逐一回收这些项目。这是池和缓存的填充方式（假设默认容量和一种视图类型）：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovkwet4cj30ef0cyt8u.jpg)

所以，只要缓存未满，ViewHolders 就会去那里。如果已满，则新的 ViewHolder 将 ViewHolder 从缓存的 “另一端” 推入池中。如果一个池已满，该 ViewHolder 将被遗忘，丢到垃圾收集器。[⁶](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#db80)

## 池和缓存的行为

现在我们来看一下实际的 RecyclerView 使用场景中的方式 - 池和缓存的行为。

考虑滚动：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovkzcpv5j30m80gljs0.jpg)

当我们向下滚动时，当前看到的物品后面有一个 “尾巴”，包括缓存项目和绑定项目。当项目8出现在屏幕上时，在缓存中找不到合适的 ViewHolder：没有与位置8关联的 ViewHolder。因此，我们使用先前在位置3的池中的 ViewHolder。当项目6在顶部消失时，它将进入缓存，将4推入池中。

当我们开始向相反的方向滚动时，图片是不同的：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovl3y7o5j30m80f2dg8.jpg)

在这里，我们在视图缓存中为位置5找到一个 ViewHolder，并立即重用它，而不需要重新绑定。这似乎是缓存的主要用例 - 使我们看到的项目向相反的方向滚动，效率更高。所以如果你有新的内容，缓存可能没用，因为用户不会经常回去。但是，如果它是一系列可供选择的东西，比如一组壁纸，则可能需要扩展缓存的容量。

有几件事要注意。首先，如果我们向上滚动查看3，该怎么办？请记住池的工作原理像一个堆栈，所以如果我们自上次看到3以来没有进行任何操作，那么 ViewHolder 3 将是最后一个放入池中的，因此现在选择反弹在位置3。如果数据没有改变，我们实际上不需要做任何重新绑定。如果你真的需要改变这个 TextView 或者 ImageView 等，你应该总是检查你的 onBindViewHolder ( )，这是一个不需要的情况的例子。 

第二，注意到在滚动时，池中的每个视图类型总是有一个以上的项！ (当然，如果你有一个带有 n 列的多列网格，那么在池中就会有 n 个项目。) 通过场景2-5最终进入池中的其他项目，在滚动过程中无用地停留在那里。 

现在让我们来看一个场景，相比之下，大量项目会进入池：调用 notifyDataSetChanged ( )  （或notifyItemRangeChanged ( ) ）：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovl8q8t3j30m80h00ug.jpg)

所有的 ViewHolders 变得无效，缓存对他们来说不是一个合适的地方， 他们都试图去池。可能没有足够的空间，所以一些不幸的将被垃圾收集，然后再次创建。与滚动相比，在这种情况下你可能需要更大的池。另一个大池有用的情况是通过 scrollToPosition ( ) 从一个位置跳转到另一个位置。

那么，我们如何选择池子的最佳大小呢？ 看起来最好的策略是在你需要大的时候扩大池，然后马上缩小。 实现这一点的一个脏方式如下: 

```java
recyclerView.getRecycledViewPool().setMaxRecycledViews(0, 20);
adapter.notifyDataSetChanged();
new Handler().post(new Runnable() {
    @Override
    public void run() {
        recyclerView.getRecycledViewPool()
                    .setMaxRecycledViews(0, 1);
    }
});
```

链接 ：[第2部分](https://medium.com/@pavelshmakov/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-continued-d81c631a2b91)

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#a02e) ¹ 事实上，即使了解 RecyclerView 的公共 API，也需要你了解一些内部工作原理。例如，javadoc 中 setHasStableIds（）方法不会告诉你为什么要使用它。

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#0c01) ² 例如。正确的视图类型在 Adapter 调用之后立即在 createViewHolder（）方法中设置，并且该字段为本地包，因此你无法自行设置它。

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#9b88) ³ 发生这种情况的示例：更改项目，以便它的视图类型发生更改，调用 notifyItemChanged（）。另外，禁用更改你的 ItemAnimator 动画，否则场景2会发生。

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#c3f9) ⁴ 另一个处于瞬态状态的 View 的例子是 EditText，其中选择了一些文本或者在编辑过程中。

[^](http://0a34/) ⁵ 可回收性和瞬态状态检查在高速缓存和池之间进行选择之前进行，说实话对我来说没有什么意义，因为缓存中的视图应该重新出现在它们消失时的状态。

[^](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714#8179) ⁶ 在支持版本23中，这种机制被一个简单的逐个索引错误所打破。随着我们逐个回收 ViewHolders，缓存中 ViewHolders 的数量在1和2之间变化。

