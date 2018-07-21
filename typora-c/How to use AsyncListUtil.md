# 如何使用 AsyncListUtil

> 原文 (Medium)：[How to use AsyncListUtil](https://android.jlelse.eu/how-to-use-asynclistutil-16b5175bb468)
>
> 作者：[Jason Feinstein](https://android.jlelse.eu/@JasonWyatt?source=post_header_lockup)

>持库的 AsyncListUtil 教程，以及如何通过 SQLite 数据库正确地备份你的 RecyclerView 的数据。

[TOC]

RecyclerView 非常棒，但是当你需要使用适配器提供成百上千的显示项时，你很快就会失去内存。 

上周，我在柏林 Droidcon 上做了一个名为 [“Get Creative to squeeze performance from SQLite”](https://speakerdeck.com/jasonwyatt/get-creative-to-squeeze-performance-from-sqlite) 的演讲。 在这篇文章中，我提出了一种由一个游标支持实现 RecyclerView.Adapter 的方法。尽管你必须在 UI 线程上访问数据库（也就是磁盘），我的意思是最好偶尔删除几帧数据，而游标会取出数据而不是崩溃应用程序(因为其他选择是将太多的数据加载到内存中)。 

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouzi2uzij30m80ch3zq.jpg)

事实证明，[Riccardo Ciovati](https://twitter.com/rciovati) 当天就在观众席上，在我的演讲结束后给我发了一封电子邮件，让我知道了 AsyncListUtil 的存在：这个工具可以用游标驱动的适配器来备份 RecyclerView ，因为在这里游标不需要在 UI 线程上访问。

我采纳了 Riccardo 的建议，并且研究了 [AsyncListUtil](https://developer.android.com/reference/android/support/v7/util/AsyncListUtil.html)，乍一看：这是一个丑陋的 API。让我们试着打破它。 

## RecyclerView 架构

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouz93a3oj30g4075wej.jpg)

这就是我们可能已经知道并热爱的东西。 

你有一个 RecyclerView，并为其分配一个 LayoutManager 和一个 Adapter，适配器为它提供了 ViewHolder 对象，然后通过应用程序的数据更新这些对象。目前，你需要能够回答一些问题，并同步执行一些操作，以便完成适配器的合同。 其中最常见的是: 

- 提供 RecyclerView 可以从适配器中获得的项目总数，以及
- 将项目的数据绑定到 ViewHolder 实例。

## RecyclerView + AsyncListUtil 架构

当你希望你的 RecyclerView 能够显示更多的数据时，你应该使用 AsyncListUtil。 

为了做到这一点，你需要扩展两个抽象类: AsyncListUtil.DataCallback 和 AsyncListUtil.ViewCallback，并创建一个使用 AsyncListUtil 的实例。 从那里开始，文档并没有说清楚，但是我已经尽我最大的努力把我如何构建事物的方法整合在一起，以便使它发挥作用: 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frov30jxc7j30kn0cw762.jpg)

这张图表是大约一个小时的揣摩和调整的最终结果， 以使其至少具有可读性。 在这里，你可以看到适配器，AsyncListUtil 和 ViewCallback 以一种循环的方式进行通信，以更新和填充 RecyclerView 。 

此外，我们还添加了 OnScrollListener，用于让 AsyncListView 知道视图已经改变(并且随之而来的是可能很快就会出现的项目范围)。 

最后，AsyncListView 使用 DataCallback 来获取数据。 我们已经向 DataCallback 提供了一个 ItemSource，以便抽象出数据收集方式的细节。 

也许只看一些代码会更好。 

## 实现 AsyncListUtil

我已经组装了一个应用程序，它显示了一个 RecyclerView 通过使用了 AsyncListUtil 的适配器。

这并不是一个令人印象深刻的用户界面，但是像这样的博客往往有一个动画的 gif，你应该期望在最后。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovgr722og307i0dcu0z.gif)

你可以在 [Github](https://github.com/jasonwyatt/AsyncListUtil-Example) 上找到这个代码，

## 数据

为了达到本文的目的，我编写了一个简短的 [python 脚本](https://github.com/jasonwyatt/AsyncListUtil-Example/blob/master/generate_data.py)，它生成一个 SQLite 数据库文件，其中包含一个名为100,000个记录的数据表，并将其放在 app / src / main / assets / database.sqlite 中。 每个记录都是一个虚拟博客帖子或文章，有三个字段：id，标题和内容。 标题和内容的文本只是由 [DWYL 的英文词库](https://github.com/dwyl/english-words)中的随机单词组成。

## ItemSource

我认为最好在数据回调中定义一个接口并使用该接口的实现，而不是使用从数据库获取项目的逻辑被埋藏在我们的适配器代码中，或者紧紧绑定到真正的 DataCallback 扩展，我认为最好定义一个接口，并在数据回调中使用该接口的实现。 

```kotlin
class Item(var title: String, var content: String)

interface ItemSource {
    fun getCount(): Int
    fun getItem(position: Int): Item
    fun close()
}
```

我们的接口：ItemSource 声明了三个方法：

- getCount ( )  - 确定源中可用项目的总数，
- getItem (position) - 获取给定位置的特定项目，以及
- close ( ) - 我们将调用的方法来释放 ItemSource 正在使用的任何资源。

为了从 SQLite 中提供 Item 对象，我们来实现 ItemSource。

```kotlin
class SQLiteItemSource(val database: SQLiteDatabase) : ItemSource {
    private var _cursor: Cursor? = null
    private val cursor: Cursor
        get() {
            if (_cursor == null || _cursor?.isClosed != false) {
                _cursor = database.rawQuery("SELECT title, content FROM data", null)
            }
            return _cursor ?: throw AssertionError("Set to null or closed by another thread")
        }

    override fun getCount() = cursor.count

    override fun getItem(position: Int): Item {
        cursor.moveToPosition(position)
        return Item(cursor)
    }

    override fun close() {
        _cursor?.close()
    }
}

private fun Item(c: Cursor): Item = Item(c.getString(0), c.getString(1))
```

在 SQLiteItemSource 中，有一些事情需要注意 : 

- 我们正在定义一个由变量支持的游标属性。这让我们可以检查支持变量是否已经关闭(或是空)并重新生成它 
- getItem (position) 方法是通过将游标移动到位置并从新位置实例化一个新的 Item 。
- 一个私有函数：Item (c：Cursor ) 的行为就像一个“扩展构造函数”，并使用游标生成一个新的 Item 实例。

## 回调

为了创建一个 AsyncListUtil，我们需要传递一个 DataCallback 和一个 ViewCallback。

我们从 DataCallback 开始。

```kotlin
private class DataCallback(val itemSource: ItemSource) : AsyncListUtil.DataCallback<Item>() {
    override fun fillData(data: Array<Item>?, startPosition: Int, itemCount: Int) {
        if (data != null) {
            for (i in 0 until itemCount) {
                data[i] = itemSource.getItem(startPosition + i)
            }
        }
    }

    override fun refreshData(): Int = itemSource.getCount()

    fun close() {
        itemSource.close()
    }
}
```

我们已经定义了 DataCallback 来接受它的构造函数中的一个 ItemSource，并且有两个由 asynclistutil.DataCallback 定义的抽象方法: 

- fillData (data，startPosition，itemCount )- 在确定需要更多数据时，由 AsyncListUtil 在后台线程上调用。 它调用 ItemSource 的 getItem 并使用 itemCount 项目填充数据，从 ItemSource 中的 startPosition 开始。
- refreshData ( ) - 也由 AsyncListUtil 在后台线程上调用，但仅在初始化时或响应在 AsyncListUtil 对象本身上调用 refresh ( ) 时才调用。 它负责返回数据集中的项目总数。 我们的实现只是再次调用 ItemSource。

重要的是要注意，我们还在 DataCallback 中定义了一个新的方法 close：调用 ItemSource 的 close 方法。

现在的 ViewCallback：

```kotlin
private class ViewCallback(val recyclerView: RecyclerView) : AsyncListUtil.ViewCallback() {
    override fun onDataRefresh() {
        recyclerView.adapter.notifyDataSetChanged()
    }

    override fun getItemRangeInto(outRange: IntArray?) {
        if (outRange == null) {
            return
        }
        (recyclerView.layoutManager as LinearLayoutManager).let { llm ->
            outRange[0] = llm.findFirstVisibleItemPosition()
            outRange[1] = llm.findLastVisibleItemPosition()
        }

        if (outRange[0] == -1 && outRange[1] == -1) {
            outRange[0] = 0
            outRange[1] = 0
        }
    }

    override fun onItemLoaded(position: Int) {
        recyclerView.adapter.notifyItemChanged(position)
    }
}
```

使用 ViewCallback 通过两种主要方式传递给构造函数: 

- 表示数据已经更改或已加载的视图 。
- 为了确定当前由视图显示的数据的位置，目的是知道何时是时候获取更多的项目，或者什么时候可以释放一些当前不在视图中的旧项目所占用的内存。 

在我们的实现中，点 # 1由 onDataRefresh ( ) 和 onItemLoaded (position) 方法完成。 他们调用RecyclerView，该视图分别传递给构造函数和 notifyDataSetChanged ( )  ，还有 notifyItemChanged(position) 。 

确定当前的视区是由 getItemRangeInto ( ) 处理的。 不是返回一个值，而是需要填充两个整数: 第一个和最后一个项目对象的位置，这些对象目前可以在 RecyclerView 中看到。 

这里有一个问题：如果 RecyclerView 还没有初始化数据，调用两个方法：findFirstVisibleItemPosition 和 findLastVisibleItemPosition 布局管理器上都会返回 -1，但是 AsyncListUtil 使用 -1 来表示不需要获取数据。因此，为了解决这个问题，当我们看到两个位置的-1时，我们只需要用零来填充。 

## OnScrollListener：使 AsyncListUtil 知道视图更改

```kotlin
private class ScrollListener(val listUtil: AsyncListUtil<in Item>) : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView?, newState: Int) {
        listUtil.onRangeChanged()
    }
}
```

Scrolllistener 的构造函数需要 AsyncListUtil，在 onScrollStateChanged 的实现中，只需调用 AsyncListUtil 的 onRangeChanged ( ) 方法即可。 

## ViewHolder

超级简单，我们的 ViewHolder 实现只是用 Item 中的内容更新两个 TextView 小部件。这是这个类：

```kotlin
class ViewHolder(itemView: View?) : RecyclerView.ViewHolder(itemView) {
    private val title: TextView? = itemView?.findViewById(R.id.title)
    private val content: TextView? = itemView?.findViewById(R.id.content)

    fun bindView(item: Item?, position: Int) {
        title?.text = "$position ${item?.title ?: "loading"}"
        content?.text = item?.content ?: "loading"
    }
}
```

注意: 当 AsyncListUtil 正在加载数据时，传递给 bindView 的项将为 null。 我们在这里通过在两个文本视图中显示"加载"文本来处理这种情况。 

以下是它的布局 XML: 

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="10dp"
    >
    <TextView
        android:id="@+id/title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textAppearance="@style/TextAppearance.AppCompat.Title"
        tools:text="Title"
        />
    <TextView
        android:id="@+id/content"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textAppearance="@style/TextAppearance.AppCompat.Body1"
        tools:text="Content"
        />
</LinearLayout>
```

## Adapter

现在所有的部分已经准备就绪，我们可以最终实现我们的 RecyclerView.Adapter：AsyncAdapter！

```kotlin
class AsyncAdapter(itemSource: ItemSource, recyclerView: RecyclerView) : RecyclerView.Adapter<ViewHolder>() {
    private val dataCallback = DataCallback(itemSource)
    private val listUtil = AsyncListUtil(Item::class.java, 500, dataCallback, ViewCallback(recyclerView))
    private val onScrollListener = ScrollListener(listUtil)

    fun onStart(recyclerView: RecyclerView?) {
        recyclerView?.addOnScrollListener(onScrollListener)
        listUtil.refresh()
    }

    fun onStop(recyclerView: RecyclerView?) {
        recyclerView?.removeOnScrollListener(onScrollListener)
        dataCallback.close()
    }

    override fun onBindViewHolder(holder: ViewHolder?, position: Int) {
        holder?.bindView(listUtil.getItem(position), position)
    }

    override fun getItemCount(): Int = listUtil.itemCount

    override fun onCreateViewHolder(@NonNull parent: ViewGroup, viewType: Int): ViewHolder {
        val inf = LayoutInflater.from(parent.context)
        return ViewHolder(inf.inflate(R.layout.item, parent, false))
    }
}
```

AsyncAdapter的构造函数接受两个参数: 一个 ItemSource 和一个 RecyclerView。 是用来创建 DataCallback 的，而 RecyclerView 则用于创建 ViewCallback。 接下来，listUtil 字段通过创建一个具有500个项目页面大小的项目对象的 AsyncListUtil 的新实例，并传递我们创建的两个回调。 最后: 我们使用 listUtil 创建一个 ScrollListener。 

AsyncListUtil 实现 onBindViewHolder ( ) 和 getItemCount ( ) 变得轻而易举。 有一点需要注意的是，listUtil.getItem (position) 可以返回 null , 而给定位置的项目仍然在加载。这意味着你需要在 ViewHolder 中处理空值绑定值，这与我们在 ViewHolder.kt 中的做法类似。

除了正常的 RecyclerView 适配器的东西，注意两个方法：onStart (recyclerView) 和 onStop (recyclerView)。 他们是我们可以在 Activity 的生命周期方法中使用的钩子，来添加/删除 OnScrollListener，并关闭 DataCallback 中的 ItemSource 所持有的资源。

## 活动：把它放在一起

我们刚刚完成，让我们使用我们闪亮的新的异步适配器和一个回收视图，就像我们将会使用任何其他 Adapter 一样。 

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var recyclerView: RecyclerView
    private lateinit var adapter: AsyncAdapter
    private lateinit var itemSource: SQLiteItemSource

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        recyclerView = findViewById(R.id.recycler)

        itemSource = SQLiteItemSource(getDatabase(this, "database.sqlite"))
        adapter = AsyncAdapter(itemSource, recyclerView)

        recyclerView.layoutManager = LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false)
        recyclerView.adapter = adapter
    }

    override fun onStart() {
        super.onStart()
        adapter.onStart(recyclerView)
    }

    override fun onStop() {
        super.onStop()
        adapter.onStop(recyclerView)
    }
}
```

同样，请注意 onStart 方法和 onStop 方法，他们调用适配器让它设置并相应地拆除它的资源。 

## 结束

没有 Riccardo 的建议，我永远不会知道 AsyncListUtil 的存在。 自从23版本以来，它一直在支持库中，我无法找到关于如何使用它的单个教程，指南或文章。

这个 API 有点笨拙，但是我希望这个指南给了你一个很好的感觉，它可以适应你的应用程序，在你有太多的数据保存在内存中，但是需要在用户界面中看到它。 

再一次，你可以在 Github 上找到这个教程的源代码：

[jasonwyatt/AsyncListUtil-Example](https://github.com/jasonwyatt/AsyncListUtil-Example)

有一种方法来获得两个世界的最佳状态是很好的: 通过从数据库中加载数据来保持堆的整洁，以及将这些数据库操作保持在 UI 线程之外。 

