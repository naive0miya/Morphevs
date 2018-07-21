# 动画 RecyclerView 的 ItemDecoration

> 原文 ：[Animating RecyclerView’s ItemDecoration](https://blog.daftmobile.com/animating-recyclerviews-itemdecoration-4aa84dc34c1b)
>
> 作者 ：[Konrad Kowalewski](https://blog.daftmobile.com/@mechatronik?source=post_header_lockup)
>
> 一个实现ItemDecoration动画的研究案例。

![1-_vcipR_IxrsbKEkNHoyUKg.gif](http://www.jcodecraeer.com/uploads/20170825/1503602221360907.gif)

长期以来DaftMobile的博客都是iOS开发的领地，里面都是关于iOS的东西。今天我们将开始一个新的旅程，进入绿色机器人－安卓的王国。

我们的外卖app [RocketLuncher](http://rocketluncher.com/) ，是DaftMobile的第一个双平台app。开发过程中我们遇到了这样一个问题。我们有多个菜品供用户选择，为了提示用户哪些被选中了我们需要一个选择动画。每个菜品由一个图标和圆圈表示。

![1-teNmForHzGLTdAeNAvFZIQ.gif](https://ws4.sinaimg.cn/large/006tKfTcgy1ftbujdlvvyg307i0dctgp.gif)

为了达到这种效果我们，我们创建了一个背景为ShapeDrawable的ImageView，把它们放到RecyclerView中然后使用ItemAnimator来做动画。记得重写 animateXyz() 的时候要调用dispatchXyzFinished()，不然会出现怪异的效果。

### Behold the lunch sets!

后来我们的设计师想了个改进的方案：把套餐中菜品的圆圈融入到一个长的胶囊中。它们还提供了细节规范：

![1-10FVFI99qJse8zbwDV_btA.png](https://ws3.sinaimg.cn/large/006tKfTcgy1ftbuji3s9tj30f007gdg4.jpg)

我们的第一反应是我们没有那样的能力来做这种动画，因为这里的圆圈都跳跃的。但是产品才不信什么能力的鬼话，所以我们开始考虑可行的方案。由于已经在使用RecyclerView的ItemAnimator来做菜品圆圈的动画，所以自然想到进一步探索一下RecyclerView。

### ItemDecoration 的拯救

我们的第一个想法是为lunch set创建另一个 item type。然后使用ItemAnimator 的onAnimateChange() 回调来制造一个从dish到set的来回过渡。但是如果菜品超过了两个事情就变得复杂了。那样的话需要另一个嵌套的带动画RecyclerView。肯定还有更好的办法。

当我们意识到可以把item的圆圈看成decoration而不是view自身的一部分的时候，我们找到了突破口，于是我们转向了ItemDecoration类。

这里的关键是，ItemDecoration对象的绘制是在整个RecyclerView上，而ItemAnimator是对每个子view动画。所以我们的想法就是用ItemDecoration绘制，单个dish绘制圆，一套dish绘制胶囊形状的圆。

还有一件事情－如何增加decoration的动画？

#### Do it the old way…

Android API 没有关于ItemDecoration动画的任何说明。旧的动画API以及新的Animators也不是可选的方案，因为它们都是基于某个属性，而ItemDecoration没有一个明显的属性可以用来动画。

实现ItemDecoration的时候你可以实现onDraw() 和 onDrawOver()方法，两者基本上是做相同的事情，不同之处在于调用的时期（前者在item绘制之前绘制，后者在之后）。

每次RecyclerView重绘它的children的时候这些方法都将被触发，而ItemAnimator动画的每一帧都将触发重绘。因此我们可以按帧绘制圆或者胶囊。

```
override fun onDrawOver(canvas: Canvas,                        parent: RecyclerView,                        state: RecyclerView.State) {    // update animation progress    // update items state    // draw appropriate stuff}
```

但是我们还需要一个东西：一个告诉 item decorator 该绘制什么动画状态的方法。详细点说就是需要一个在动画期间变化的值，所以我们又回到了ValueAnimator。

#### 使用新的API

ValueAnimator持有一个动画期间不断发生改变的值，我们的ItemDecoration就需要得到这个值来设置dish进入和退出动画的当前进度，然后绘制从圆到胶囊过渡的特定阶段。ValueAnimator必须是跟dish图标的动画同步的，因此我们让ItemAnimator来管理它。ValueAnimator暴露给ItemDecoration，用它来查询当前的动画进度。

```
private fun updateAnimationProgress(animator: OurItemAnimator) {    if (animator.isRunning)        animationProgress = animator.animationProgress    else        animationProgress = 1f}
```

在圆进入或者退出的时候我们根据这个值得到透明度的变化，过渡到胶囊的时候我们得到胶囊的长度。当值为一的时候我们绘制完全展开的胶囊或者完全可见的圆。

#### 管理 state(s)

实现这个效果的关键部分可能就是如何恰当的处理view的过渡状态。每当ItemDecoration的方法被调用的时候就意味着布局的动画正在进行，而RecyclerView只知道正在进入的那个item的当前的状态，没有哪个组建持有历史信息，所以ItemDecoration对象内部必须保存前一个状态的信息。比如，在已经选择了一个菜的基础上在添加一个菜，那么初始状态就该是dish，最终状态就该是dish set（胶囊状态）。

这里要提取出的关键信息就是：当前选择的是一套（绘制加长的胶囊），还是前一次选择的是一套，这次减少套餐中的数目（绘制缩短的胶囊），或者都不是（只绘制圆）。

```
if (isInSet() || (wasInSet() && state.current.size == 1)) {    drawPillAroundSet(canvas, parent)    drawSetLabel(canvas)} else {    drawCircleAroundEachSingleDish(canvas, parent)}
```

#### 添加边距

你可能注意到了设计图中的一个细节，在菜品还没有合并在一起之前，图标之间的距离要宽一些（这个细节其实对最终效果影响不大）。这个问题使用ItemDecoration的getItemOffsets()方法很容易解决，getItemOffsets()为item之间添加间隔。我们只需检查item是不是lunch set的一部分。

```
override fun getItemOffsets(outRect: Rect,                            view: View,                            parent: RecyclerView,                            state: RecyclerView.State) {    if (isItemViewPartOfSet(parent, view).not())        outRect.right = spacing}
```

### 结语

比较有意思的是我们结合部分新的 Android API制造了一个过时的逐帧动画，最终效果如下：

![1-c2a7kiLJmWJczIhGk5N7Sw.gif](https://ws1.sinaimg.cn/large/006tKfTcgy1ftbukincl8g30m80cj1l1.gif)

幸运的是我们有足够的时间去完成前面的效果。如果你没有那么幸运，我希望这篇文章对你有所帮助！如果有什么更好的建议请留言！

别忘了点击💚 关注 [DaftMobile 博客](https://blog.daftmobile.com/)！你也可以在 [Facebook](https://www.facebook.com/DaftMobile) 和 [Twitter](https://twitter.com/DaftMobile) 上找到我们。