# ADAPTERCOMMANDS 库

> 原文 (http://hannesdorfmann.com)：[ADAPTERCOMMANDS](http://hannesdorfmann.com/android/adapter-commands)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

上周我很荣幸能够在 Artem Zinnatullin 的播客 [The Context](https://github.com/artem-zinnatullin/TheContext-Podcast) 做客，在那里我们谈论了 android 上的软件架构。在本期节目中，我强调了 MVP 中的演示模型的重要性，例如如何处理回收视图适配器数据集的更改。之后，人们问我如何准确地为数据集的更改应用动画，以及为什么一个演示模型在这种情况下是有帮助的。 

在我们深入展示模型部分之前，这里有一个好消息：我把所有这些东西放在一起并捆绑到一个名为 [AdapterCommands](https://github.com/sockeqwe/AdapterCommands) 的库中。

那么这个库是关于什么的？ 嗯，RecyclerView 有这个不错的组件称为 ItemAnimator ，它负责RecyclerView 项目的动画 。使用 adapter.setHasStableId (true) 时，已经有了对动画的支持。然而，如果你没有稳定的 id，那么调用 notifstyletchanged ( ) 将不会运行任何动画。例如，假设我们在回收视图中显示一个项目列表。 当添加一个新项目时，我们可以称之为 adapter.notifyItemInserted (position) ，而不仅仅是适配器。 现在，ItemAnimator 启动并启动了这个项目。 

AdapterCommands 基本上实现了命令对象被用来封装执行操作所需的所有信息的[命令模式](https://en.wikipedia.org/wiki/Command_pattern)。 因此，这个库不直接调用 adapter.notifyItemInserted (position)，而是提供一个 ItemInsertedCommand，如下所示：

```java
public class ItemInsertedCommand implements AdapterCommand {

  private final int position;

  public ItemInsertedCommand(int position) {
    this.position = position;
  }

  @Override
  public void execute(RecyclerView.Adapter<?> adapter) {
    adapter.notifyItemInserted(position);
  }

}
```

正如你所看到的，这个类实现了 AdapterCommand 接口的 execute (adapter) 方法。这个库为所有这些操作提供了这样的命令，如 ItemRemovedCommand， ItemChangedCommand， ItemRemovedCommand 等等。 这个库还提供了一个 AdapterCommandProcessor 类，它采用 List 并执行每个命令：

```java
public class AdapterCommandProcessor {

  private final RecyclerView.Adapter<?> adapter;

  public AdapterCommandProcessor(RecyclerView.Adapter<?> adapter) {
    this.adapter = adapter;
  }

  public void execute(List<AdapterCommand> commands) {
    for (int i = 0; i < commands.size(); i++) {
      commands.get(i).execute(adapter);
    }
  }
}
```

我知道，第一眼看上去并不那么令人印象深刻。 那么这种模式的好处是什么呢？ 通常情况下，应用程序会在回收视图中显示一个项目列表，并且基础数据集会被更改，例如，结合 SwipeRefreshLayout，用户可以重新加载项目的更新列表(即从后端加载项目)。 我们该怎么处理这份新的名单？ 只需要调用 adapter.notifdatetchanged ( ) 通知新的项目列表应该显示？ 但是 ItemAnimator 怎么办呢？ AdapterCommands  库提供 DiffCommandsCalculator 类。 这个类计算旧列表和新列表的差异，并返回一个 list，然后可以由   AdapterCommandProcessor 执行。 让我们来看看演示:    [[Video]](https://youtu.be/z05IK8ejERM)

正如你在上面的演示视频中看到的，每当我们点击添加或删除按钮项变化时，都会发生动画。 实现方法如下: 

```java
public class MainActivity extends AppCompatActivity
    implements SwipeRefreshLayout.OnRefreshListener {

  @Bind(R.id.recyclerView) RecyclerView recyclerView;

  List<Item> items = new ArrayList<Item>();
  Random random = new Random();
  ItemAdapter adapter; // RecyclerView adapter
  AdapterCommandProcessor commandProcessor;
  DiffCommandsCalculator<Item> commandsCalculator = new DiffCommandsCalculator<Item>();

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    ButterKnife.bind(this);

    refreshLayout.setOnRefreshListener(this);

    adapter = new ItemAdapter(this, items);
    recyclerView.setAdapter(adapter);
    recyclerView.setLayoutManager(new GridLayoutManager(this, 4));

    commandProcessor = new AdapterCommandProcessor(adapter);
  }

  @OnClick(R.id.add) public void addClicked() {

    int addCount = random.nextInt(3) + 1;

    for (int i = 0; i < addCount; i++) {
      int position = random.nextInt(items.size());
      Item item = new Item(id(), randomColor());
      items.add(position, item);
    }
    updateAdapter();
  }

  @OnClick(R.id.remove) public void removeClicked() {

    int removeCount = random.nextInt(3) + 1;

    for (int i = 0; i < removeCount; i++) {
      int position = random.nextInt(items.size());
      Item item = items.remove(position);
    }
    updateAdapter();
  }

  private void updateAdapter() {
    // calculate the difference to previous items
    List<AdapterCommand> commands = commandsCalculator.diff(items);
    commandProcessor.execute(commands);
  }

}
```

# 演示模型

我想你明白我的意思了，但是这和演示模型和 MVP 有什么关系？ 在 MVP 中，演示者生成 (optional) 演示模型，这是为视图优化的另一个数据模型，包含视图需要知道的所有信息，以便视图可以简单地采用这个演示模型，并可以直接显示它，而无需计算事物。 请到[此处](https://github.com/sockeqwe/mosby/issues/85)浏览更多资料。 

所以假设我们正在为报纸开发一个应用程序，应用 MVP，Retrofit 来加载 NewsItems 列表，并使用 RxJava 来连接这些点。 我们不是直接从 Presenter 到 View 传递一个 List \<newsitem>，而是引入了一个 NewsItemsPresentationModel，看起来是这样的: 

```java
class NewsItemsPresentationModel {
  List<NewsItem> newsItems;
  List<AdapterCommand> adapterComamnds;
}
```

使用 RxJava，通过定义这样的 Func1，将 List \<newsitem> 转换为 NewsItemsPresentationModel 非常容易: 

```java
class PresentationModelTransformer extends Func1< List<NewsItem>, NewsItemsPresentationModel> {

  private DiffCommandsCalculator<NewsItem> diffCalculator = new DiffCommandsCalculator<>();

  @Override
  public NewsItemsPresentationModel call(List<NewsItem> items){
    List<AdapterCommand> commands = diffCalculator.diff(items);
    return new PresentationModelTransformer(items, commands);
  }
}
```

演示者看起来像这样：

```java
class NewsItemsPresenter extends MvpBasePresenter<NewsItemView> {

  private BackendApi backendApi; // Retrofit service to load news items
  private PresentationModelTransformer pmTransformer = new PresentationModelTransformer()

  public void loadItems(){
    view.showLoading();
    backendApi.getNewsItems()
              .map(pmTransformer) // Creates NewsItemsPresentationModel
              .subscribeOn(Schedulers.io())
              .observeOn(AndroidSchedulers.mainThread())
              .subscribe(new Subscriber() {
                  public void onNext(NewsItemsPresentationModel pm){
                      view.setNewsItems(pm);
                      view.showContent();
                  }
                  public void onError(Throwable t){
                    view.showError(t);
                  }
              });
  }
}
```

正如你所看到的 RxJava，我们可以使用 map ( ) 操作符将模型转换为演示模型。 另一个值得注意的事情是，这个转换和差异的计算运行在后台线程( Schedulers.io ( ))上。 

视图现在非常的简单，并不包含如何从列表中插入新项目等复杂的计算。 该视图从演示者中获取了 NewsItemsPresentationModel，并且拥有视图需要显示的所有新列表(包含动画) : 

```java
class NewsItemsActivity extends Activity implements NewsItemView, OnRefreshListener {

  @Bind(R.id.recyclerView) RecyclerView recyclerView;  
   NewsItemsPresenter presenter;
   AdapterCommandProcessor commandProcessor;

   @Override
   protected void onCreate(Bundle b){
     super.onCreate(b);
     setContentView(R.layout.activity_newsitems);

     presenter = new NewsItemsPresenter();
     Adapter adapter = new NewsItemsAdapter();
     commandProcessor = new AdapterCommandProcessor(adapter);

     recyclerView.setAdapter(adapter);
     presenter.loadItems();
   }

   @Override
   public void onRefresh(){
     presenter.loadItems();
   }

   @Override
   public void setNewsItems(NewsItemsPresentationModel pm) {
     adapter.setItems(pm.newsItems);
     commandProcessor.execute(pm.commands);
   }

   ...
}
```

希望你现在看到的视图是相当愚蠢，解耦，更容易维护和测试。 

## 原理

你可能认为你不需要第三方库来做这件事。 事实上，对于那些你知道列表是按时间顺序排列的简单用例，新列表中的项目将总是被添加到旧列表的顶部(或者最后)。 在这种情况下，你只需要编写 diff newList-oldList，然后调用 adapter.notifingitemrangeinserted (0，diff.size ( )) ，对吗？ 同样在这个用例中，你可以使用这个库只是为了不再自己编写命令类和命令处理器。 但是如果你实现了上面描述的报纸应用程序和一个已经在旧列表中的新闻项目的标题与新列表相比发生了改变，那么 adapter.notifyItemChanged (position)必须调用？ 或者，如果列表并不总是以同样的方式排序呢？ 如果一个项目被删除了怎么办？ 

在这种情况下，DiffCommandsCalculator 就是解决方案。但是它是如何工作的？我们来比较两个列表：

```
  oldList     newList
    A           A
    B           B
    C           B2
    D           C
    E           F
    F           G
    G           E
                H
```

我们已经插入 B2，移除 d 和移动 e，并在列表末尾插入 h。 让我们计算一下两者的区别: 

```
> B2   2
 < D   3
 < E   4  
 > E   6  
 > H   7
```

第一列指出它是插入还是删除。 第二列是受影响的项目，最后一列是列表项的索引(从零开始)。 这个模式似乎很熟悉，不是吗？ 如果你使用 git 和 diff (命令行工具，GUIs 也可用)来检测和解决合并冲突，那么你几乎每天都会看到类似的情况。 Diffcommandalscalculator 实现与 diff 相同的算法。 这种问题叫做[最长公共子序列](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem)。 在任意数量的输入数据解决这个问题是 [NP-hard](https://en.wikipedia.org/wiki/NP-hardness)。 幸运的是，我们已经确定了列表的大小和项目的数量。 因此，我们可以实现一种运用[动态规划](https://en.wikipedia.org/wiki/Dynamic_programming)概念的算法，在多项式时间 o (n  m)(其中 n 是 oldList 中元素的数目，m 是 newList 中的元素数)。 这听起来在理论上是真的，不是吗？ 实际上，它比你想象的更容易实现。 我发现这个 [youtube](https://www.youtube.com/watch?v=P-mMvhfJhu8) 视频很有用。 

## 结束

这个名为 AdapterCommands 的小型库在 maven central 上可用，源代码可以在 [Github](https://github.com/sockeqwe/AdapterCommands) 上找到。这个库是 [AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates) 的小兄弟（组合优于继承），并通过实现命令模式帮助你在 RecyclerView 中对数据集更改（如果没有稳定的 id）进行动画。这个库和 adapter.setHasStableId ( ) 之间的主要区别在于后者依赖于数据集中每个项的唯一和稳定的 id，而 AdapterCommands 使用 java 的 equals ( )方法来确定数据集的变化。此外，这个 MVP 和 PresentationModel 相当不错，如本文所示。请记住，将 oldList 与 newList 的每个元素进行比较的运行时间是 O (n  m)，因此，如果数据集中有多个项目，则应考虑在后台线程上运行 DiffCommandsCalculator。RxJava 提供了一个很好的线程模型，并且在 MVP 和演示模型中表现得非常好 ，因此我建议如何将所有的东西连接在一起。

> 注：在 Artem Zinnatullin 的播客 “ The Context ” 中，我曾经说过，我不会做很多功能性的 UI 测试，因为我认为没有必要这么做。我的观点是，我根据 MVP 实现我的应用程序，而且我的视图是非常愚蠢的，所以在视图层不会有太多的错误。使用 AdapterCommands 强调这篇论文，因为我确实测试了 Presenters 和 PresentationModel。此外，由于 AdapterCommands 库本身已经被测试过，我可以继续使用它，并且在我的应用程序中少写一个测试。但是，这并不意味着你不应该写功能性的 UI 测试！测试视图层也不会有什么坏处，即使视图是愚蠢的， 也不会出现太多错误。 我相信 TDD。然而，当一个测试需要超过10秒执行时，整个 TDD 工作流程和生产力就会被破坏。 因此，我确保从 Presenter 向下测试所有的内容，而且愚蠢 View 会获得一个优化的演示模型，这样就不会出现太多错误。 

