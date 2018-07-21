# 适配器的组合

> 原文 (Medium)：[Adapter’s composition](https://medium.com/@programmerr47/adapters-composition-7fd49e840546)
>
> 作者：[Michael Spitsin](https://medium.com/@programmerr47?source=post_header_lockup)

[TOC]

> 另一种消除适配器地狱的方法

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frott5sxu7j30go0jrwh0.jpg)

最近，当我为下一篇文章选择主题时，我阅读了 [Hannes Dorfmann](https://medium.com/@sockeqwe) 关于许多列表应用程序中称为“适配器地狱”问题的[伟大文章](http://hannesdorfmann.com/android/adapter-delegates)。  简而言之，当你用一堆列表创建或多或少复杂的应用程序时，你将面临适配器代码重复的问题。 你有一个机会：

- 你可以保持复制，每次根据[复制粘贴原则](https://en.wikipedia.org/wiki/Copy_and_paste_programming)写入新的适配器;
- 你可以消除样板代码和重复。

有几种方法可以做到这一点，我认为很明显的一个方法就是继承。 但是，正如前面的文章所说，不幸的是，最近你会遇到适配器地狱。 此外，由于 java 的继承限制，你将不得不生成新的重复，一遍又一遍地重复你的适配器代码，因为 [java 的继承限制](http://beginnersbook.com/2013/05/java-multiple-inheritance/)。 在本文中，我将尝试更多地揭示"适配器组织"的主题。 

## 第一部分适配器装饰

假设你有一个统一项目的适配器（我们可以有一个文本列表，或新闻，音频，出售汽车和 e.t.c）。而且你想要为其中的一些评级栏添加一个，它将出现在第一个元素和第二个元素之间。 比如"请评估我们的申请"。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frottdhwdwj307i0dcq4l.jpg)

在截图中，我们有这样的例子。 因此，对于一些列表，我们希望放置评分条并显示它(或隐藏，当用户评价应用程序或按下关闭按钮时)。 在这种情况下，我们当然可以使用继承。 我们可以有一个抽象的 RateAdapter 类，然后只需要适配器来继承它。 而且它会很好地工作。 但是突然间我们需要为一些列表添加广告横幅(假设在第五和第六项之间)。 所以我们只需要列出物品清单、物品列表和评级条，列出项目和广告; 列出条目和标语和评级。 我们如何组织适配器层次结构？ 我们没有多重继承，所以我们将使用 RatingAdapter，它的继承者将是广告适配器和所有的适配器，它们需要显示评级或广告，继承广告适配器。 但这并不是好事，因为这并不是显而易见的，广告适配器的继承者也有能力展示评级。 此外，也许它不需要显示评级，但它有这个可能性，这是一个潜在的缺陷在未来。 

所以解决这个问题的第一个方法是引入[适配器装饰](https://en.wikipedia.org/wiki/Decorator_pattern)。 是的，我知道，这是非常奇特的，因为 [RecyclerView.Adapter](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.0_r1/android/support/v7/widget/RecyclerView.java#RecyclerView.Adapter) 没有接口，因此包含了一些逻辑，所以引入装饰可能会对应用程序的稳定性构成潜在的风险。 此外，它的复杂性迫使我们编写一个订阅机制，为更新元素提供更方便的机制。 但是，我的目标是为适配器提供额外的选择: 

```kotlin
class RateAdapter<T : ViewHolder>(private val origin: RecyclerView.Adapter<T>) : RecyclerView.Adapter<T>() {
    var isRateShown = false
        private set

    init {
        //binding decorator adapter with notifications from decorated one
        origin.registerAdapterDataObserver(WrappedObserver(this))
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) = when(viewType) {
        rateLayout() -> RateHolder(parent.inflate(viewType, false)) as T
        else -> origin.onCreateViewHolder(parent, viewType)
    }
    
    private fun ViewGroup.inflate(@LayoutRes layoutId: Int, attachToRoot: Boolean) = 
            LayoutInflater.from(context).inflate(layoutId, this, attachToRoot)

    override fun onBindViewHolder(holder: T, position: Int) = when(position) {
        RATING_POSITION -> {/work with rate holder/}
        else -> origin.onBindViewHolder(holder, itemPositionWithRateOffset(position))
    }

    override fun onBindViewHolder(holder: T, position: Int, payloads: List<Any>?) = when(position) {
        RATING_POSITION -> onBindViewHolder(holder, position)
        else -> origin.onBindViewHolder(holder, itemPositionWithRateOffset(position), payloads)
    }

    override fun getItemCount() = itemPositionWithRateOffset(origin.itemCount, 1)

    override fun getItemViewType(position: Int) = when(position) {
        RATING_POSITION -> rateLayout()
        else -> origin.getItemViewType(itemPositionWithRateOffset(position))
    }
    
    fun ratePosition() = RATING_POSITION

    private fun rateLayout() = R.layout.item_rate

    private fun itemPositionWithRateOffset(oldPosition: Int) = itemPositionWithRateOffset(oldPosition, -1)

    private fun itemPositionWithRateOffset(oldPosition: Int, offset: Int) = oldPosition + if (isItemBelowRate(oldPosition)) offset else 0

    private fun isItemBelowRate(position: Int) = isRateShown && position >= RATING_POSITION

    //functions like show or hide rating

    companion object {
        private val RATING_POSITION = 1
    }
}
```

```kotlin
class RateHolder(val rateAppView: View) : ViewHolder(rateAppView) {
    //some info or binding stuff
}
```

```kotlin
class WrappedObserver(private val adapter: RateRecyclerAdapter<out RecyclerView.ViewHolder>) : RecyclerView.AdapterDataObserver() {
    override fun onChanged() {
        adapter.notifyDataSetChanged()
    }

    override fun onItemRangeChanged(positionStart: Int, itemCount: Int) {
        analyzeRange(positionStart, itemCount, adapter::notifyItemRangeChanged)
    }

    override fun onItemRangeChanged(positionStart: Int, itemCount: Int, payload: Any?) {
        analyzeRange(positionStart, itemCount, { pos, count -> adapter.notifyItemRangeChanged(pos, count, payload) })
    }

    override fun onItemRangeInserted(positionStart: Int, itemCount: Int) {
        analyzeRange(positionStart, itemCount, adapter::notifyItemRangeInserted)
    }

    override fun onItemRangeRemoved(positionStart: Int, itemCount: Int) {
        analyzeRange(positionStart, itemCount, adapter::notifyItemRangeRemoved)
    }

    override fun onItemRangeMoved(fromPosition: Int, toPosition: Int, itemCount: Int) {
        for (i in 0..itemCount - 1) {
            onItemMoved(fromPosition + i, toPosition + i)
        }
    }

    private fun onItemMoved(fromPosition: Int, toPosition: Int) = adapter.run {
        val finalFrom = itemPositionWithRateOffset(fromPosition, 1)
        val finalTo = itemPositionWithRateOffset(toPosition, 1)
        notifyItemMoved(finalFrom, finalTo)
    }

    private fun analyzeRange(positionStart: Int, itemCount: Int, action: (Int, Int) -> Unit) {
        if (adapter.isRateShown) {
            val startDelta = adapter.ratePosition - positionStart
            val firstPartCount = if (startDelta > itemCount) itemCount else if (startDelta < 0) 0 else startDelta
            val secondPartCount = itemCount - firstPartCount
            if (firstPartCount > 0 && secondPartCount > 0) {
                action(positionStart, firstPartCount)
                action(adapter.ratePosition + 1, secondPartCount)
            } else if (firstPartCount == 0) {
                action(positionStart + 1, itemCount)
            } else {
                action(positionStart, itemCount)
            }
        } else {
            action(positionStart, itemCount)
        }
    }
}
```

第一类是我们的 RateAdapter，我们的包装器，我们的装饰器，它通过构造函数应用任何回收器适配器，然后在原始元素之间插入评级项。 此外，我们需要为它提供额外的逻辑和功能，但这不是重点。 关键是，尽管我们摆脱了继承，我们还是摆脱了装饰。 

将这个类演化为一个可以插入不是一个而是一组项目的抽象类并不是那么难。 例如，在引入了 RateAdapter 和 AdvertisementAdapter 类之后，我们可以将 Kotlin 的扩展语法糖应用到: 

```kotlin
fun <T : RecyclerView.ViewHolder> RecyclerView.Adapter<T>.withRating() = RateAdapter<T>(this)

fun <T : RecyclerView.ViewHolder> RecyclerView.Adapter<T>.withAdvertisement() = AdvertisementAdapter<T>(this)

fun sampleBuiling() = new NewsAdapter().withRating().withAdvertisement()
```

## 缺点

从我的角度来看，RecyclerView 适配器的这种方法的缺点比优势更强：

- 首先，我们需要创建和管理适配器之间的观察机制。 因此，当包装适配器将更新其项目时，包装器适配器将获得关于该事项的通知，并能够将通知传递到 RecyclerView 回收视图。 
- 其次，从外部管理适配器的行为是相当困难的。 要做到这一点，你需要保持对所有包装层的引用。 在一个片段中看到两个或三个... 或四个适配器字段是很奇怪的。 我们可以通过聚合一个管理器中的所有适配器来稍微改进它，这个管理器将把责任委托给适当的适配器 。

```kotlin
class AdapterAggregator {
    var rateAdapter: RateAdapter? = null
    var advertisementAdapter: AdvertisementAdapter? = null
    
    fun buildAdapter() = NewsAdapter()
            .withRating().let { rateAdapter = it }
            .withAdvertisement().let { advertisementAdapter = it }

    fun showRating() = rateAdapter?.showRating()
    
    fun hideRating() = rateAdapter?.showRating()

    fun showAdvertisement() = advertisementAdapter?.showAdvertisement()

    fun hideAdvertisement() = advertisementAdapter?.hideAdvertisement()
}
```

通过引入适当的接口和 Kotlin 的委托，我们将拥有：

```kotlin
interface RatingInteractor {
    fun showRating()
    fun hideRating()
}

interface AdvertisementInteractor {
    fun showAdvertisement()
    fun hideAdvertisement()
}

class AdapterAggregator(rateAdapter: RateAdapter) {
    var rateAdapter: RateAdapter? = null
    var advertisementAdapter: AdvertisementAdapter? = null

    fun buildAdapter() = NewsAdapter()
            .withRating().let { rateAdapter = it }
            .withAdvertisement().let { advertisementAdapter = it }
}

class AdapterDelegator(adapterAggregator: AdapterAggregator): 
        RatingInteractor by adapterAggregator.rateAdapter, 
        AdvertisementInteractor by adapterAggregator.advertisementAdapter
```

但它不会消除这种方法的缺点。 

## 第二部分每个项目都是 binder

另一种防止继承的方法看起来类似于 Hannes Dorfmann 的解决方案。我们将制作一个通用适配器。 但不同之处在于，每个项目都有责任约束自己，而不是把这个责任转移到特别的委托人身上。 要做到这一点，我们需要声明一个接口适配项目: 

```kotlin
interface AdapterItem<T: RecyclerView.ViewHolder> {
    val viewHolderProducer: HolderProducer<T>
    fun bindView(viewHolder: T, position: Int)
}
```

它的实现将负责绑定数据，以查看并提供特殊工厂 HolderProducer，以创建新的视图持有者。 

```kotlin
interface HolderProducer<T: RecyclerView.ViewHolder> {
    fun produce(parentView: ViewGroup): T
}
```

此外，我们需要考虑如何组织生成不同元素类型的不同 id。 为了做到这一点，我们引入了一个 [MutableList](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-mutable-list/) 装饰器: 

```kotlin
class RecyclerItems<T: RecyclerView.ViewHolder, Item : AdapterItem<T>>(private val items: MutableList<Item>) : MutableList<Item> by items {
    private val holderMap: MutableMap<Int, HolderProducer<T>>
    private val typeNames: MutableList<String>
    val typesMap: Map<Int, HolderProducer<T>>
        get() = Collections.unmodifiableMap(holderMap)

    init {
        typeNames = getAllDifferentClassesFromCollection(items)
        holderMap = retrieveProducers(items)
    }

    override fun add(location: Int, item: Item) = items.add(location, item).apply { checkAndAddNewType(item) }

    override fun add(item: Item) = items.add(item).apply { checkAndAddNewType(item) }

    override fun addAll(location: Int, collection: Collection<Item>) = items.addAll(location, collection).apply { checkAndAddNewType(collection) }

    override fun addAll(collection: Collection<Item>) = items.addAll(collection).apply { checkAndAddNewType(collection) }

    override fun clear() {
        items.clear()
        typeNames.clear()
        holderMap.clear()
    }

    override fun subList(start: Int, end: Int) = RecyclerItems(items.subList(start, end))

    override fun toString() = items.toString()

    private fun retrieveProducers(items: List<Item>) = HashMap<Int, HolderProducer<T>>().apply {
        for (item in items) {
            val key = getItemType(item)

            if (key == -1) {
                throw IllegalArgumentException("Not filled collection of typeNames")
            } else if (!containsKey(key)) {
                put(key, item.viewHolderProducer)
            }
        }
    }

    private fun checkAndAddNewType(collection: Collection<Item>) = collection.forEach { checkAndAddNewType(it) }

    private fun checkAndAddNewType(item: Item) {
        if (getItemType(item) == -1) {
            typeNames.add(item.javaClass.name)
            holderMap.put(typeNames.size - 1, item.viewHolderProducer)
        }
    }

    private fun getAllDifferentClassesFromCollection(collection: Collection<>) =
            collection.mapTo(HashSet<String?>()) { it?.javaClass?.name }
                    .filterNotNull()
                    .toMutableList()

    fun getItemType(position: Int) = getItemType(get(position))

    fun getItemType(item: Item) = typeNames.indexOf(item.javaClass.name)
}
```

因此，这个过程将以这种方式构建: 

- 我们有一份数据列表。它可以是任何东西。
- 我们用自己特定的 AdapterItem 实现来包装每个元素。
- 我们将所有结果项目添加到 RecyclerItems 列表
- 我们通过这个列表到一个特殊的适配器。下面就是：

```kotlin
class AbstractMultiTypeRecyclerAdapter<T: RecyclerView.ViewHolder, Item : AdapterItem<T>>(protected var items: RecyclerItems<T, Item>) : RecyclerView.Adapter<T>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): T {
        return items.typesMap[viewType]!!.produce(parent)
    }

    override fun onBindViewHolder(holder: T, position: Int) {
        items[position].bindView(holder, position)
    }

    override fun getItemCount() = items.size

    override fun getItemViewType(position: Int) = items.getItemType(position)
}
```

## 缺点

尽管有机会在使用上述方法时忘记了项目类型问题，但你将面临下一个缺陷: 

你将不得不为列表中的每个元素创建一个新对象，以将其包装在 AdapterItem 中，以便与此类适配器一起使用。 作为替代，而不是在 AdapterItem 中组合模型，你可以继承具有 AdapterItem 实现的模型类。 例如：

```kotlin
open class Model(val str1: String, val str2: String)

class Holder(itemView: View?) : RecyclerView.ViewHolder(itemView)

class ModelAdapterItem(str1: String, str2: String) : Model(str1, str2), AdapterItem<Holder> {
    override val viewHolderProducer = object: HolderProducer<Holder> {
        override fun produce(parentView: ViewGroup) = Holder(/create you item view/)
    }

    override fun bindView(viewHolder: Holder, position: Int) {
        //bind view
    }
}
```

- 如果我们有可变项的情况，我们需要考虑将 AdapterItem 和 AbstractMultiTypeRecyclerAdapter 绑定在一起。

## 第三部分 Databinding

最后的解决方案是使用数据绑定来清除代码样板。 这不是“适配器继承”问题的完全解决方案，而是一种简化代码的方法。 因此，它可以很容易地与前两种方法和 Hannes Dorfmann 代表结合使用。 以下是适配器的外观：

```kotlin
class BindingAdapter(private val items: List<Any>) : RecyclerView.Adapter<BindingHolder>() {
    override fun onBindViewHolder(holder: BindingHolder, position: Int) = holder.binding.run {
        setVariable(BR.item, items[position])
        executePendingBindings()
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) = 
            BindingHolder(DataBindingUtil.inflate(parent.inflater(), viewType, parent, false))

    override fun getItemCount() = items.size

    override fun getItemViewType(position: Int) = when(items[position]) {
        is String -> R.layout.item_string
        is Int -> R.layout.item_int
        else -> R.layout.other_layout
    }
}
```

就这样！你不需要考虑项目类型和 e.t.c. 而且，我们可以将 getItemViewType 函数逻辑移出并传递给 BindingAdapter 布局生成器：

```kotlin
class BindingAdapter(private val items: List<Any>,
                     private val layoutGenerator: (Int) -> Int) : RecyclerView.Adapter<BindingHolder>() {
    override fun onBindViewHolder(holder: BindingHolder, position: Int) = holder.binding.run {
        setVariable(BR.item, items[position])
        executePendingBindings()
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
            BindingHolder(DataBindingUtil.inflate(parent.inflater(), viewType, parent, false))

    override fun getItemCount() = items.size

    override fun getItemViewType(position: Int) = layoutGenerator(position)
}
```

## 缺点

与往常一样，这种解决方法有其自身的缺陷和局限性：

- 对于布局，你只能使用 xml-s。这很明显，但仍然。由于你使用的是数据绑定库，因此你只需要在 xml 文件中定义项目布局。
- 每个想要绑定 item 的布局，必须在其中定义如下的 [variable](https://developer.android.com/topic/libraries/data-binding/index.html#variables) 变量：

```xml
<data>
    <variable name="item" type="ItemType"/>
</data>
```

## 第四部分几乎没有关于比较的话

总结一下，你至少有五六种方法来管理适配器。你可以：

1. 每次需要新的适配机制时，创建一个新的适配器类
2. 使用继承来带来一些适配器管理并减少重复
3. 实现适配器授权器，将不同类型的绑定分开 
4. 使用装饰来允许组合适配器而不是建立刚性的继承层次结构 
5. 允许识别每个元素作为绑定机制 
6. 应用 [data-binding](https://developer.android.com/topic/libraries/data-binding/index.html) 来快速实现通用适配器

对我来说，没有什么灵丹妙药。 你不能只是选择其中一种方式，然后说:"好吧，伙计。 我会用那个的。 对傻瓜来说，还有其他方法。"。 嗯，不是。 我认为我们需要为每个问题单独找到最好的选择。 

如果你有简单的应用程序或具有少量列表的应用程序。也许它有许多列表，但它们都是静态的（它们不会动态地改变，在运行时添加新的类型的项目）并且有一种类型的项目。在这种情况下，你不需要创建一堆委托者或者包装 AdapterItem 实现者中的每个元素。你只需要为每个列表创建一个简单的适配器。然后重构它：创建一个抽象的基础适配器，并把所有的普通适配器的东西，然后在孩子，你只需要添加绑定和附加机制。没有委托人，没有包装，没有装饰。简单而有效。

如果你有多种类型的列表，但是没有火箭科学。 你知道，只是列表有不同的元素。 例如，在个人资料页面上的帖子。 你可以有图片，音乐，视频，短信。 它们都是不同的，所以你需要一个适配器来处理不同类型的项目。 在这种情况下，[Hannes Dorfmann](https://medium.com/@sockeqwe) 的委托者可以帮助你。 因为你不需要在运行时粘贴新的类型，所以你不需要在列表中间粘贴一些特殊的项目。 你就是没有这种感觉。 你只有不同文章的列表，而且你想显示它，而不是乱丢适配器类。 

现在假设你有一个应用程序。 然后产品所有者过来说:"嘿，我们需要对这些列表进行评分，并在列表中做广告"。 在这种情况下，你可以实现适配器装饰器，因为你知道要放置哪些列表以及在哪里放置它，但是你不知道在这个功能上需要扩展什么类型的列表。 因此，最好的装饰器就是这些任务的最佳人选。 考虑一下，你有一个用户发布列表，而且，在每个帖子中你都有一个引用列表。 总而言之，你有两种类型的列表，但是你需要在所有的列表中显示评级，但是在随机的一些列表中。 你不在乎它是否提供了引用，你只是随机保留一定数量的引用，然后插入评分条。 在这种情况下，PostsAdapter 和 referencecadapter 都不应该知道评级。 你需要在运行时定义它。 而装饰器是最好的。 

在需要处理复杂列表的情况下，应该充满新信息、新项目、动态的新类型时，你可以为每个列表元素使用 AdapterItem 包装器。 如果你害怕大量的内存分配，你可以扩展模型，并由其继承人实现 AdapterItem 接口。 但是这种方法的主要意义在于，你不需要考虑哪些类型的适配器需要处理。 你在运行时移动这个问题的解决方案，因此你可以利用这个机会调整一个列表，一个适配器。 你有一个文字帖子的列表，音频帖子出来了？ 你不需要为你的适配器添加一个新的类型，只需要包装你的模型并添加到列表中。 

如果你能够处理第三部分中描述的限制，那么就去将数据绑定注入适配器中，你将清理适配器 / 委托者 / 装饰器 / 项目代码，因为所有绑定的东西都会在 XML 文件中发生。 

## 结束

列表是所有应用程序中常见的一部分。 而处理适配器是构建应用程序的基本部分之一。 我相信你一定有一些"适配器地狱"问题的解决方案。 现在，除此之外，你有三种方法。 顺便说一下，如果你们能在下面的评论中分享你处理适配器的方法。 

