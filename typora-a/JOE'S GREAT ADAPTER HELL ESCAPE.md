# 逃离 adapter 的地狱

> 原文 (hannesdorfmann.com)：[JOE'S GREAT ADAPTER HELL ESCAPE](http://hannesdorfmann.com/android/adapter-delegates)
>
> 作者：[Hannes Dorfmann ](http://hannesdorfmann.com/about/)

[TOC]

让我告诉你一个关于 Joe 的故事，MyLittleZoo 公司的一个 android 开发人员，关于他如何尝试创建可重用的不同视图类型的 RecyclerView 适配器，以及他如何最终成功实现了可重用的适配器。 

曾经，一位 Android 开发人员 Joe 曾在一家名为 MyLittleZoo 的初创公司工作。这家创业公司正在网上销售宠物用品。 Joe 的工作是创建和维护一个与在线商店(网站)功能相同的 android 原生 app 。 因此 90％ 的开发工作只是在 RecyclerView 中显示一个项目列表。第一个版本1.0只需要显示一个配料列表。因此 Joe 实现了一个 AccessoiresAdapter 来显示配料列表，特殊推荐的配料用 item_accessory_offer.xml 显示，而普通配料则用  item_accessory.xml 显示。所以适配器有两种  view types 。ViewTypes 允许您为适配器中的不同项目渲染不同的 xml 布局。在内部 ViewTypes 只是一个唯一的 id，一个整数。所以 Joe 的 AccessoiresAdapter 实现如下所示：

```java
public class AccessoiresAdapter extends RecyclerView.Adapter {

  final int VIEW_TYPE_ACCESSORY = 0;
  final int VIEW_TYPE_ACCESSORY_SPECIAL_OFFER = 1;

  List<Accessory> items;

  @Override public int getItemViewType(int position) {
     Accessory accessory = items.get(postion);
     if (accessory.hasSpecialOffer()){
       return VIEW_TYPE_ACCESSORY_SPECIAL_OFFER;
     } else {
       return VIEW_TYPE_ACCESSORY;
     }
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (VIEW_TYPE_ACCESSORY_SPECIAL_OFFER == viewType){
      return new SpecialOfferAccessoryViewHolder(inflater.inflate(R.layout.item_accessory_offer, parent));
    } else {
      return new AccessoryViewHolder (inflater.inflate(R.layout.item_accessory)):
    }
  }

  ...

}
```

到目前为止，MyLittelZoo 的 Android 应用程序1.0在 Play Store 上发布 。一切都很酷。

然后，MyLittelZoo 成长起来，应用程序也随之增长。 Joe 需要实现一个新的启动 Activity ，这里需要显示不同 item ： NewsTeaser 需要和配料一起显示。由于 HomeAdapter 需要同时显示配料，所以他决定通过继承 AccessoriesAdapter 来重用之前的代码 ：

```java
public class HomeAdapter extends AccessoriesAdapter {

  final int VIEW_TYP_NEWS_TEASER = 2;

  @Override public int getItemViewType(int position) {
     if (items.get(position) instanceof NewsTeaser){
       return VIEW_TYP_NEWS_TEASER;
     } else {
       // accessories and special offers
       return super.getItemViewType(position);
     }
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (VIEW_TYP_NEWS_TEASER == viewType){
      return new NewsTeaserItem( inflater.inflate(R.layout.item_news_teaser, parent));
    } else {
      // accessories and special offers
      return super.onCreateViewHolder(parent, viewType);
    }
  }

  ...
}
```

另外，还需要实现一个新的 Activity，只显示一些关于宠物食品的小贴士。因此，Joe 实现了 PetFoodTipAdapter : 

```java
public class PetFoodTipAdapter extends RecyclerView.Adapter {

  final int VIEW_TYP_FOOD_TIP = 0;

  @Override public int getItemViewType(int position) {
     return VIEW_TYP_FOOD_TIP;
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return new PetFoodViewHolder(inflater.inflate(R.layout.item_pet_food, parent))
  }

  ...

}
```

他的项目经理很高兴，因为他能及时交付。MyLittelZoo 2.0在 Play Store 上成功发布。

几个星期后，产品经理来到 Joe，告诉他生意并没有像预期的那样发展 。为了赚钱，公司决定与一家大型广告公司签订合同。广告公司可以在 MyLittleZoo android 应用程序中显示横幅广告。 换句话说 : 他们把自己的灵魂卖给了魔鬼。Joe 的工作是通过使用提供的广告 sdk 在应用程序中加入广告横幅。时间紧迫 ，公司需要钱（广告收入）。应用更新必须尽快发布。由于广告横幅需要和其他项目一起显示在 RecyclerView 中，Joe 决定创建一个名为 AdvertismentAdapter 的适配器基类：

```java
public class AdvertismentAdapter extends RecyclerView.Adapter {

  final int VIEW_TYP_ADVERTISEMENT = 0;

  @Override public int getItemViewType(int position) {
     return VIEW_TYP_ADVERTISEMENT;
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return new AdvertismentViewHolder(inflater.inflate(R.layout.item_advertisment, parent))
  }

  ...

}
```

从此，每个其他的 adapter 都继承自 AdvertisementAdapter ：

- AccessoiresAdapter extends AdvertisementAdapter
- HomeAdapter extends AccessoiresAdapter extends AdvertisementAdapter
- PetFoodTipAdapter extends AdvertisementAdapter

到处充满广告的 3.0版发布在 Play Store。再一次，产品经理对 Joe 的工作感到满意。

过了半年，产品经理又敲了一下 Joe 的门，告诉他事情有了变化。MyLittleZoo android 应用程序的用户不喜欢3.0版中引入的闪烁的广告横幅，应用程序在 Play Store 中获得了大量的负面评价。 访问量大减，公司不再盈利了。但 MyLittleZoo 不能简单地从 app 中删除广告，因为他们已经与"魔鬼"签署了一个有效的长期合同，当然，这里是指广告公司。

然后 MyLittleZoo 的一个聪明的营销人员想出了一个绝妙的主意，就是在 RecyclerView 中展示 NewsTeaser 和 PetFoodTip。没有广告，没有推荐。这个计划是为了重新获得用户的信心 。此外，产品经理告诉乔说，应用程序必须在未来两天内发布，因为在即将到来的周末是大型宠物博览会  app 需要在那里呈现。Joe 认为这是可行的。 他已经有了 NewsTeaser 和 PetFoodTip 的 xml 布局，adapter 也已经实现了。所以 Joe 所要做的就是把它移植到一个 android 库中，在原来的 MyLittleZoo app 和新的无广告 app 之间共享。

当 Joe 正准备将事情转移到库, 意识到自己面临的困境：你记得 adapter 的继承关系吗？

1. 每个 adapter 都继承自 AdvertisementAdapter。 但是新的 app 不需要显示广告。 而且，提供的广告 sdk 显示横幅实在是太 bug 了，经常导致内存泄漏和崩溃。 即使没有展示广告横幅，广告 sdk 在背后做了太多事情。 因此，在新 app 中包括广告 SDK 是不能接受的。
2. 没有可重用的 adapter，可以显示 NewsTeaser（HomeAdapter 的一部分）和 PetFoodTip（PetFoodTipAdapter 的一部分）。 Joe 应该怎么做？ 他可以创建一个新的 NewsTipAdapter 继承自 HomeAdapter，然后把 PetFoodTip 作为新的 ViewTypes 添加进去。 但是这意味着他将有两个 adapter 来维护同一个 ViewTypes。

## Joe 欢迎你到 adapter 地狱！

哦，孩子，Joe 很沮丧。 然后，恐慌随之而来。 他应该如何解决这个问题？ 一个月后，当一个新功能(一种新的 ViewTypes )必须被实现时，他应该如何避免？ 

于是，Joe 开始在白板上写下他的要求。 但是没有一个好主意。他很伤心，回想起他小时候的那段日子。童年时期的生活如此简单。那些日子里，他唯一需要担心的事情就是在他玩完乐高后清理他的房间。乐高？ 等等等等！ Joe 有一个绝妙的主意：他真正需要的是类似于搭建乐高房子一样建一个 adapter :  做一个空的基座，然后把你真正需要的乐高零件粘在一起。如果你需要一个窗口在你的乐高房子，去取一个窗户零件。如果您需要屋顶，去取一个屋顶零件。如果您的乐高房子需要后院，那么就取一个乐高花零件 。

该死的，然后他得到了总体情况：

> ==组合优于继承==

很多时候，他与其他开发者一起讨论时同意“组合优于继承 ”。到目前为止，这只是一个好的口号，但他从来没有真正根据这个原则建立一些东西。所以一个空的 adapter 是基座。ViewTypes 是可重复使用的组件（乐高零件）。

因此，Joe 开始定义可重复使用的乐高零件，比如 NewsTeaserAdapterDelegate 和 PetFoodTipAdapterDelegate :

```java
public class NewsTeaserAdapterDelegate {

  private int viewType;

  public NewsTeaserAdapterDelegate(int viewType){
    this.viewType = viewType;
  }

  public int getViewType(){
    return viewType;
  }

  public boolean isForViewType(List items, int position) {
    return  items.get(position) instanceof NewsTeaser;
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent) {
    return new NewsTeaserViewHolder(inflater.inflate(R.layout.item_news_teaser, parent, false));
  }

  public void onBindViewHolder(List items, int position, RecyclerView.ViewHolder holder) {
      NewsTeaser teaser = (NewsTeaser) items.get(position);
      NewsTeaserViewHolder vh = (NewsTeaserViewHolder) vh;

      vh.title.setText(teaser.getTitle());
      vh.text.setText(teaser.getText());
  }
}
```

```java
public class PetFoodTipAdapterDelegate {

  private int viewType;

  public PetFoodTipAdapterDelegate(int viewType){
    this.viewType = viewType;
  }

  public int getViewType(){
    return viewType;
  }

  public boolean isForViewType(List items, int position) {
    return  items.get(position) instanceof PetFoodTip;
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent) {
    return new PetFoodTipViewHolder(inflater.inflate(R.layout.item_pet_food, parent, false));
  }

  public void onBindViewHolder(List items, int position, RecyclerView.ViewHolder holder) {
      PetFoodTip tip = (PetFoodTip) items.get(position);
      PetFoodTipViewHolder vh = (PetFoodTipViewHolder) vh;

      vh.image.setImageRes(tip.getImage());
      vh.text.setText(tip.getText());
  }
}
```

然后他拿出了一个空的 adapter，然后把乐高零件放在上面，创建了 NewsTipAdapter，用于新的 app: 

```java
public class NewsTipAdapter extends RecyclerView.Adapter{

  final int VIEW_TYP_NEWS_TEASER = 0;
  final int VIEW_TYP_FOOD_TIP = 1;

  NewsTeaserAdapterDelegate newsTeaserDelegate;
  PetFoodTipAdapterDelegate foodTipDelegate;

  List items;

  public NewsTipAdapter(){
    newsTeaserDelegate = new NewsTeaserAdapterDelegate(VIEW_TYP_NEWS_TEASER);
    foodTipDelegate = new PetFoodTipAdapterDelegate(VIEW_TYP_FOOD_TIP);
  }

  @Override public int getItemViewType(int position) {
     if (newsTeaserDelegate.isForViewType(items, position)){
       return newsTeaserDelegate.getViewType();
     }
     else if (foodTipDelegate.isForViewType(items, position)){
       return foodTipDelegate.getViewType();
     }

     throw new IllegalArgumentException("No delegate found");
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

    if (newsTeaserDelegate.getViewType() == viewType){
      return newsTeaserDelegate.onCreateViewHolder(parent);
    }
    else if (foodTipDelegate.getViewType() == viewType){
      return foodTipDelegate.onCreateViewHolder(parent);
    }

    throw new IllegalArgumentException("No delegate found");
  }


  @Override public void onBindViewHolder(VH holder, int position){
    int viewType = holder.getViewType();
    if (newsTeaserDelegate.getViewType() == viewType){
      newsTeaserDelegate.onBindViewHolder(items, position, holder);
    }
    else if (foodTipDelegate.getViewType == viewType){
      foodTipDelegate.onBindViewHolder(items, position, holder);
    }
  }
}
```

我猜你应该看明白了。与使用继承不同，Joe 为每个 ViewTypes 定义了一个 delegate。每个 delegate 负责创建和绑定 ViewHolder。就如你看到的，上面的代码片段有许多散乱的代码。Joe 发现了一个插件式的解决办法： 

```java
/*
 * @param <T> the type of adapters data source i.e. List<Accessory>
 */
public interface AdapterDelegate<T> {

  /**
   * Called to determine whether this AdapterDelegate is the responsible for the given data
   * element.
   *
   * @param items The data source of the Adapter
   * @param position The position in the datasource
   * @return true, if this item is responsible,  otherwise false
   */
  public boolean isForViewType(@NonNull T items, int position);

  /**
   * Creates the  {@link RecyclerView.ViewHolder} for the given data source item
   *
   * @param parent The ViewGroup parent of the given datasource
   * @return The new instantiated {@link RecyclerView.ViewHolder}
   */
  @NonNull public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent);

  /**
   * Called to bind the {@link RecyclerView.ViewHolder} to the item of the datas source set
   *
   * @param items The data source
   * @param position The position in the datasource
   * @param holder The {@link RecyclerView.ViewHolder} to bind
   */
  public void onBindViewHolder(@NonNull T items, int position, @NonNull RecyclerView.ViewHolder holder);
}
public class AdapterDelegatesManager<T> {

  public AdapterDelegatesManager<T> addDelegate(@NonNull AdapterDelegate<T> delegate) {
    ...
  }

  public int getItemViewType(@NonNull T items, int position) {
    ...
  }

  public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    ...
  }

  public void onBindViewHolder(@NonNull T items, int position, @NonNull RecyclerView.ViewHolder viewHolder) {
    ...
  }
}
```

这个想法是将 AdapterDelegates 注册到 AdapterDelegatesManager。Adapterdelegatesmanager 内部具有根据给定 ViewTypes 确定适合的 AdapterDelegate 的逻辑，并调用相应的委托方法。 因此，应用到 newspaceadapter 中的代码大致如下 : 

```java
public class NewsTipAdapter extends RecyclerView.Adapter{

  final int VIEW_TYP_NEWS_TEASER = 0;
  final int VIEW_TYP_FOOD_TIP = 1;

  List items;

  AdapterDelegatesManager delegates = new AdapterDelegatesManager();

  public NewsTipAdapter(){
    delegates.add(new NewsTeaserAdapterDelegate()); // Assigns internally ViewType integer
    delegates.add(new PetFoodTipAdapterDelegate());
  }

  @Override public int getItemViewType(int position) {
     return delegates.getItemViewType(items, position);
  }

  @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    return delegates.onCreateViewHolder(parent, viewType);
  }

  @Override public void onBindViewHolder(VH holder, int position){
      delegates.onBindViewHolder(items, position, holder);
  }
}
```

我猜你可以想象现在 MyLittleZoo app 的其它 adapter 的样子。 有一个 AdvertisementAdapterDelegate， NewsTeaserAdapterDelegate， PetFoodTipAdapterDelegate 和 AccessoryAdapterDelegate。 从现在开始，adapter 可以由真正需要的 ViewTypes（AdapterDelegates）组合。 另一个优点是，你将 inflating 布局，创建 view holder，绑定 view holder 的过程从 adapter 中分离出来，成了单独的，模块化的，可复用的AdapterDelegate。你有没有注意到现在的 adapter 代码看起来有多小巧，你有一个关注的分离，使事情更容易扩展和解耦？ 另一个好的作用是，更多的团队成员可以在同一个“adapter”上并行工作，而不必担心复杂的合并冲突，因为不是每个人都触及巨大的 adapter 文件，而是团队成员可以同时在专用的 AdapterDelegate 文件上工作。

Joe 很高兴，产品经理很高兴，应用程序的用户很高兴，事实上大家都很开心。Joe 决定把AdapterDelegates 放在一个自己的 library 中，然后开源。真是皆大欢喜。

你可以在 Github 上找到 [AdapterDelegates](https://github.com/sockeqwe/AdapterDelegates)，并且也可以在 maven central 找到。

附： AdapterDelegates 库还提供了一个基类 ListDelegationAdapter，它已经将 RecyclerView.Adapter 方法与 AdapterDelegatesManager 方法放在一起，这样就可以减少更多的样板代码：

```java
public class NewsTipAdapter extends ListDelegationAdapter {

  public NewsTipAdapter(){
    // delegatesManager is a field defined in super class
    // ViewType integer is assigned internally by delegatesManager
    delegatesManager.add(new NewsTeaserAdapterDelegate());
    delegatesManager.add(new PetFoodTipAdapterDelegate());
  }

}
```

查看 [Github](https://github.com/sockeqwe/AdapterDelegates) 上的库了解更多详细的内容。

免责声明：Joe 不是真正的人，MyLittleZoo Inc. 也不是真正的公司。 两者都是我想象的生物。 请注意，本文中显示的代码片段可能无法编译。 这是一种类似于 java 的伪代码，可以让您了解真实代码的大致样子。

