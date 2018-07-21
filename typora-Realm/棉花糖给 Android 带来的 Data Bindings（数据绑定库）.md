# 棉花糖给 Android 带来的 Data Bindings（数据绑定库）

> 原文：[棉花糖给 Android 带来的 Data Bindings（数据绑定库）](https://academy.realm.io/cn/posts/data-binding-android-boyar-mount/)
>
> 作者：[Yiğit Boyar](https://twitter.com/yigitboyar)

[TOC]

谷歌的 Yigit Boyar 和 George Mount 为 Android 开发的 Data Binding 库可以使开发者以最小的力气，快速构建丰富的具有响应性的用户体验。在这次[海湾 Android 开发者大会](http://www.meetup.com/bayareaandroid/)讲座中，他们演示了通过删除 样板数据驱动的用户界面，来说明如何使用 [Data Binding](https://developer.android.com/tools/data-binding/guide.html) 改善您应用程序的开发，使代码更加干净优雅。可以切确地说：只要你在 Android 开发中使用 Data Binding，很快你就会感觉到它所带来到好处。💰

------

## 介绍 [(0:00)](javascript:presentz.changeChapter(0,0,true);)

我们是 George Mount 和 Yigit Boyar，在谷歌 Android UI 工具团队。我们有很多关于 Data Binding 的信息要分享给你们。我们会讨论到一些重要到方面，比如 Data Binding 是怎么工作的、如何集成到您到 App 当中、它的原理和如何与一些其他组件共同使用，同时，我们也会给出一些最佳实践示例。

## 为什么要使用 Data Binding? [(0:44)](javascript:presentz.changeChapter(0,1,true);)

你可能会想我们为什么决定去实现这么一个库，关于这点，可以先看以下一些传统代码写法的例子：

```xml
<LinearLayout …>
    <TextView android:id="@+id/name"/>
    <TextView android:id="@+id/lastName"/>
</LinearLayout>
```

这是一个你经常会看到的 Android UI。 假设你有一堆带 ID 的视频内容。你的设计师来了，说：“好吧，让我们尝试添加新的信息到这个布局，”这样，当你添加任何视频，你需要跟随另外一个 ID。你需要回到你的 Java 代码，修改 UI。

```java
private TextView mName
protected void onCreate(Bundle savedInstanceState) {
  setContentView(R.layout.activity_main);
  mName = (TextView) findViewById(R.id.name);
}

public void updateUI(User user) {
  if (user == null) {
    mName.setText(null);
  } else {
    mName.setText(user.getName());
  }
}
```

你写了一个新的 TextView，你通过 findViewById 在代码中找到它，同时你把它赋值给了你的变量，使得当你需要更新用户信息的时候，再通过它去给这个 TextView 设置上用户信息。

```java
private TextView mName
protected void onCreate(Bundle savedInstanceState) {
  setContentView(R.layout.activity_main);
  mName = (TextView) findViewById(R.id.name);
  mLastName = (TextView) findViewById(R.id.lastName);
}
public void updateUI(User user) {
  if (user == null) {
    mName.setText(null);
    mLastName.setText(null);
  } else {
    mName.setText(user.getName());
    mLastName.setText(user.getLastName());
  }
}
```

总的来说，仅仅为了增加一个 View 到你的 UI 当中，你得做好几个步骤的事情。这似乎是比较愚蠢的样板代码，虽然有时它不需要任何脑力。

幸运的是，已经有一些漂亮的库可以简化我们的这些操作。比如你使用 ButterKnife 这个库，能够摆脱讨厌的 findViewById 而获得组件，它让代码更加简洁易读。通过它可以节省很多额外的代码。

```java
private TextView mName
protected void onCreate(Bundle savedInstanceState) {
  setContentView(R.layout.activity_main);
  ButterKnife.bind(this);
}
public void updateUI(User user) {
  if (user == null) {
    mName.setText(null);
    mLastName.setText(null);
  } else {
    mName.setText(user.getName());
    mLastName.setText(user.getLastName());
  }
}
```

这是很好的一步，但是我们想走得更远。另外，我们可以说：”好吧，为什么我需要这些项目呢？有什么可以直接生成它。我有一个布局文件，我有它们的 IDs.” 所以，你可以使用 Holdr，它可以替你可以处理布局文件，然后为他们创建 View 组件。你通过 Holder，转换 View 的 ID 到组件变量。

```java
private Holdr_ActivityMain holder;
protected void onCreate(Bundle savedInstanceState) {
  setContentView(R.layout.activity_main);
  holder = new Holdr_ActivityMain(findViewById(content));
}
public void updateUI(User user) {
  if (user == null) {
    holder.name.setText(null);
    holder.lastName.setText(null);
  } else {
    holder.name.setText(user.getName());
    holder.lastName.setText(user.getLastName());
  }
} 
```

这种方式也不错，但其中仍然有很多是不必要的代码。而且这里还有一个可能我们没有想到的麻烦、一个我们无论如何无法减少代码量的地方，它也是很简单的代码：我有一个 User 对象，我只是想把数据内容从这个 User 对象转移到 View 当中，但我们往往改了一个地方，忘了另一个地方，最终导致了产品运行崩溃。这也是我们关注的一部分，我们想摆脱所有这些比较蠢的代码。

当你使用 Data Binding，它很像 Holder 模式，而且你只要做一点点事情，其余的内容 Data Binding 会帮你完成。

```java
private ActivityMainBinding mBinding;
protected void onCreate(Bundle savedInstanceState) {
  mBinding = DataBindingUtil.setContentView(this,
                           R.layout.activity_main);
}

public void updateUI(User user) {
  mBinding.setUser(user);
}
```

## “幕后” [(3:53)](javascript:presentz.changeChapter(0,15,true);)

那么，Data Binding 在幕后是怎么工作的呢? 在此之前可以先看看我们的布局文件：

```xml
<LinearLayout …>
  <TextView android:id="@id/name"    />
  <TextView android:id="@id/lastName"    />
</LinearLayout>
```

我们有这些 Views 的 ID，但如果我们能够直接通过 Java 代码找到它们，为什么还需要这些 ID 呢？嗯，我们现在已经不需要它们了，所以我们可以摆脱它们，在这些地方，我可以放一些更加明显的内容。

```xml
<LinearLayout …>
  <TextView android:text="@{user.name}"/>
  <TextView android:text="@{user.lastName}"/>
</LinearLayout>
```

现在，当我看这些布局文件，我可以知道这些 TextView 要显示什么，它变得非常明显，所以我没必要再回到我当 Java 代码去阅读它们到底是干什么的。我们设计 Data Binding 库其中一个原因就是不想去用一些看起来不明显不直接的表达方式。使用 Data Binding，你只要简单地告诉它：“我们用这种类型的用户标记这个布局文件，现在我们将要找到它。”而如果你的产品设计经理要求你添加另一个新的 View 进去这个布局，你只要在这个布局文件中加上它无需改变其他 Java 代码：

```xml
<layout>
    <data>
        <variable name="user"
                  type="com.android.example.User"/>
    </data>
    <LinearLayout …>
        <TextView android:text="@{user.name}"/>
        <TextView android:text="@{user.lastName}"/>
        <TextView android:text='@{"" + user.age}'/>
    </LinearLayout>
</layout>
```

同时，它也非常容易寻找 bug. 你可以看着类似上面的代码，然后说：“哦，这是空的字符串加上 user.age!” 这样写是安全正确的，而如果你只是把整形设置给 text 了，它会误以为这是资源索引 resId，找不到对应的资源从而导致崩溃。

## 但是它怎么工作的呢？ [(5:57)](javascript:presentz.changeChapter(0,19,true);)

Data Binding 做的第一件事就是进入并处理您的布局文件，其中的“进入”指的是，在你的程序代码正在被编译的过程中，它会找出布局文件中所有关于它的内容，获取到它所需要的信息，然后删掉它们，删掉它们的原因是如果继续存着视图系统并不认得它们。

第二步骤就是通过语法来解析这些表达式，例如：

```xml
<TextView android:visibility="@user.isAdmin ? View.VISIBLE : View.GONE}"/>
```

这个 `user` 是一个索引，之后的 `View` 也是个索引，另一个 `View` 也是个索引。它们都是索引或称标识，像是真正的对象，在这边我们真的不知道它们是什么。同样的对于 `VISIBLE` 和 `GONE` 也是如此。这里面有对象数据访问，是一个三目运算表达式，这就是目前为止我们所理解到的。而对于我们的库，它的工作就是从这些文件中把东西解析出来，了解里面有什么。

第三步就是在你代码编译过程中解决相关依赖问题。在这一步中，例如，我们看一下 `user.isAdmin`，想：“这是在运行时获取到 User 类对象中的一个布尔值。”

最后一步就是 Data Binding 会自动生成一些你不需要再写的类文件，总之到这里你只要享受它带来的好处💰就是了。

## 一个针对上述的示例 [(7:40)](javascript:presentz.changeChapter(0,27,true);)

这是一个真实场景中的布局文件：

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="user" type="com.android.example.User"/>
    </data>
   <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">
        <TextView android:text="@{user.name}"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"/>
        <TextView android:text="@{user.lastname}"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"/>
    </RelativeLayout>
</layout>

```

当编译时 Data Binding 进入，它会读懂并丢掉所有视图系统不认识的内容，连接它们，同时放上我们的绑定 tags（标志），变成这样:

```xml
   <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/
            android:layout_width="match_parent"
            android:layout_height="match_parent">
        <TextView android:tag="binding_1"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"/>
        <TextView android:tag="binding_2"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"/>
    </RelativeLayout>
```

这实际上就是我们如何做到数据绑定向后兼容，如果没有这样，当你把它放在一个旧的系统设备上，这个可怜的家伙可能根本不知道发生了什么事。

## 表达式 [(8:01)](javascript:presentz.changeChapter(0,30,true);)

```xml
<TextView android:text="@{user.age < 18 ? @string/redacted : user.name}"/>
```

上面这是另一个示例。当我们解析它的时候，它会在编译的时候被换成对应的一些表达式代码，以至于当程序开始运行的时候，程序能知道该怎么做。我们检查到这个表达式左边是一个布尔值，右边是一个字符串，资源索引的也是一个字符串。就是说，这里有一个布尔值，一个字符串，和另一个字符串，这是一个三目运算，同时这个整体也是一个字符串。所以，我们知道布局文件这里有一个 text 属性，和它的字符串，那么 Data Binding 要怎么把它们连接在一起呢？

答案就是通过代码上 `setText(CharSequence)` 这个方法。现在，Data Binding 知道怎么把布局表达式转换为 Java 代码了。如果你要看一下具体示例，可以看看类似如下的 TextView 和 ImageView ：

```xml
<TextView android:text="@{myVariable}"/>
textView.setText(myVariable);
<ImageView android:src="@{user.image}"/>
imageView.setSrc(user.image);
```

上面的这段代码里有一个问题，ImageView 有一个布局属性为 src，那么根据上面的示例，它会被转换为 `setSrc`? 当然不，因为 ImageView 类并没有这么一个方法，而是有一个不叫这个名字的方法用来设置它的图片源，可是，Data Binding 怎么能够知道？

src 它被称作源属性，一旦你使用了这样的属性，Data Binding 也得能够支持它。

```xml
<TextView …/>
textView.setText(myVariable);
<ImageView android:src="@{user.image}"/> 
imageView.setImageResource(user.image);
          @BindingMethod(
              type = android.widget.ImageView.class,
              attribute = "android:src",
              method = "setImageResource") 
```

我们创造了一些注解，借助它们我们可以简单地说：“在 ImageView 这个类中，属性 src 对应这个方法” 我们只要这么写一次就好，我们实际上也只是形成一次框架。我们提供了它，但你可能有很多自定义 View 也面临需要这么添加对应关系。使用了这些注解方法，数据绑定就能够知道如何解决。同样，这一切都是发生在编译期间。

## Data Binding 特别吸引人的地方 [(9:54)](javascript:presentz.changeChapter(0,36,true);)

Data Binding 使你的工作更加简单。让我们来看一下我们所支持的语言：支持*绝大部分*的 Java 写法，它允许变量数据访问、方法调用、参数传递、比较、通过索引访问数组，甚至还支持三目运算表达式。 这基本上能够满足你的需求，但也有一些事情无法办到，比如 `new`，我们真的不想你在布局中的表达式里写 `new`.

我们的基本目标就是让你的布局文件中的表达式尽可能短和可读，我们不像让你不得不写超级长的表达式，而只为了访问你的联系人的名字。我们希望你能够使用 `contact.name`，我们会想：“这是一个可访问的属性变量，或者是一个 getter？”又或者你可能有一个叫做 “name” 的函数方法。

我们也会自动检查是否为 null，这实际上非常酷。如果你想访问 contact 的 name 变量，而 contact 是 null 的，以前你不得不痛苦地写`contact null ? null : contact.friend null ? :`，而现在，如果 contact 是 null 的，那么整个表达式就是 null 的，你无需判断也不会出错。

我们还提供了一个 null 的合并运算符号 `??`，你可能已经从其他语言中看到过了，这是一个三目运算符的简便写法：

```kotlin
contact.lastName ?? contact.name
contact.lastName != null ? contact.lastName : contact.name
```

它表达的是，如果左边不是 null 的，那么使用左边的值，否者使用右边的值。

我们还可以使用中括号操作符来访问 list 或者 map，如果你有使用 `contacts[0]`，它有可能是一个 list 或者是一个数组，都可以。使用中括号来访问项目，更加容易简洁。

## 资源内容 [(12:20)](javascript:presentz.changeChapter(0,48,true);)

我们希望你能在你的表达式中使用资源引用内容，因此你可以在你的表达式中使用资源和字符串格式化方法。

在表达式中:

```xml
android:padding="@{isBig ? @dimen/bigPadding : @dimen/smallPadding}"
```

内联字符串格式:

```xml
android:text="@{@string/nameFormat(firstName, lastName)}"
```

内联复数:

```xml
android:text="@{@plurals/banana(bananaCount)}"
```

## 自动属性 [(13:00)](javascript:presentz.changeChapter(0,49,true);)

以下我们有一个 DrawerLayout…

```xml
<android.support.v4.widget.DrawerLayout
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  app:scrimColor="@{@color/scrim}"/>
```

```java
drawerLayout.setScrimColor(
  resources.getColor(R.color.scrim))
```

我们使用了 `app:scrimColor` 属性，但实际上 DrawerLayout 并没有提供这么一个 xml 布局属性，但我们会将它和 DrawerLayout 类的 `setScrimColor` 方法联系起来。当我们看到有一个布局属性名为 `scrimColor` 同时我们找到了一个 `setScrimColor`，我们就会判断它的参数类型是否匹配布局中的值。首先我们找到这个 color，它是一个 `int` 类型，而如果 `setScrimColor` 接受一个 `int` 类型的参数，我们就视为它们是匹配的，这多么方便啊！

## 事件处理 [(13:41)](javascript:presentz.changeChapter(0,50,true);)

我不知道你们写过多少的 `clicked` 来响应一个 button 或者 view 的点击，我们使用 Data Binding 也是支持点击响应的。你可以使用 `clicked`，同时，另外的一些事件也都支持。当然， 支持到 Android 2.3 版本。你甚至可以通过表达式指定程序处理者（我并不是建议你这么做，但是它可以做到！）。你还可以做一些偏门的监听器，比如 `onTextChanged`，TextWatcher 有三个方法，但大部分人都只关心它的 `onTextChanged`，借助 Data Binding，你可以直接访问任意一个或全部你想访问的：

```xml
<Button android:onClick="clicked" …/>

<Button android:onClick="@{handlers.clicked}" …/> 

<Button android:onClick="@{isAdult ? handlers.adultClick : handlers.childClick}" …/> 

<Button android:onTextChanged="@{handlers.textChanged}" …/>
```

## 详细的可观测性 [(14:56)](javascript:presentz.changeChapter(0,51,true);)

很多时候，我们不知道数据和更新 View 的关系。想像一下我们有一个商店，我们有一个商品最近更新了价格。这就需要我们自动跟随着更新我们的 UI，怎么办？通过 Data Binding，这件事可以变得非常简易。

首先我们需要创建一个项目，包含一些可以被观测的对象。在这里，我已经继承了个 BaseObservable，然后我们在这个新的类当中加入我们的属性。

```java
public class Item extends BaseObservable {
    private String price;

    @Bindable
    public String getPrice() {
        return this.name;
    }

    public void setPrice(String price) {
        this.price = price;
        notifyPropertyChanged(BR.price);
    }
}
```

我们使用 `notifyPropertyChanged` 来进行数据改变完成通知，但我们怎么通知一个数据即将改变？我们不得不写一个 `@Bindable` 注解在 `getPrice`。这将会自动产生一个 `BR.price`，这个 BR 很像我们经常使用的 R 类文件，我们通过这些注解会自动生成它。但是，你可能不想让我们入侵你的整个代码体系，所以我们允许你去实现这些可被观测的类。自己实现的示例如下：

```java
public class Item implements Observable {
    private PropertyChangeRegistry callbacks = new …
    …
    @Override
    public void addOnPropertyChangedCallback( 
            OnPropertyChangedCallback callback) {
        callbacks.add(callback);
    }
    @Override
    public void removeOnPropertyChangedCallback( 
            OnPropertyChangedCallback callback) {
        callbacks.remove(callback);
    }
}
```

我们有一个很方便的类 `PropertyChangedRegistry`，让你很方便地进行回调和通知。你们中的一些人可能会觉得这是一件很讨厌的事情，他们更多的是想要一个可观测的属性变量。从本质上讲，它们每一个都是一个可观测都对象。你可以很方便地说，accessImage， 它实际上就会去访问对应图片地内容，如果你要访问 price，它就会去访问价格的字符串内容。

关于这些对象的特别之处在于，以前使用 Java 代码，你必须得调用 set 或者 get 方法，但在你的绑定表达式中，你只要写 `item.price`，我们能够自动知道你需要调用它的 getter 方法，所以当价格发生变化时，它才设定它的值。

```java
public class Item {
    public final ObservableField<Drawable> image =
            new ObservableField<>();
    public final ObservableField<String> price =
            new ObservableField<>();
    public final ObservableInt inventory =
            new ObservableInt();
}

item.price.set("$33.41");
```

在其他情况下，你可能有更多的“细碎、不确定”的数据，这通常发生在你开发周期的一开始，特别是在原型阶段，可能你会有很多键值数据对，并且你很不想定义这些类，所以你可能会想使用 map 来存储这些数据对。这种情况下，你可以使用一个可观测的 ObservableMap 来放你的项目，然后就可以访问它们了。不幸的是，你只能使用中括号类似访问数组那样访问它们：

```java
ObservableMap<String, Object> item =
        new ObservableArrayMap<>();

item.put("price", "$33.41"); 
```

```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" 
    android:text='@{item["price"]}'/>
```

## 在任意线程中进行通知 [(18:29)](javascript:presentz.changeChapter(0,62,true);)

这里的一个方便的是，你不必在UI线程进行通知：你可以在任何你想要的线程上进行更新。然而，我们仍然是在将在UI线程进行响应，所以你必须小心。此外，对于列表请不要这样做的！对于列表，你仍然应该在UI线程上通知，因为我们将在UI线程读取到，并且我们将需要在UI线程读取到它的长度，我们不做任何类型的同步。你可能已经知道这样对于 RecyclerView 和 ListView 会造成很多问题。这是因为列表本身的问题，而不是因为这些类的问题。

## 性能 [(19:21)](javascript:presentz.changeChapter(0,63,true);)

也许很多人最关心的是它的性能。数据绑定这类库是臭名昭著的缓慢，所以在 Android 上，我们考虑了很多，我们相信我们能做出比较好的性能。

对于性能的最重要的方面是，我们基本上是零反射。一切都发生在编译期。有时候，这么做是不方便的，因为它发生在编译时，但在长期运行，我们关心不到。当应用程序运行时，我们不希望解决任何问题。

其次，你还可以得到一些额外的好处。让我们来讨论在一个布局中使用数据绑定，你把一个对象命名为price，然后价格变化了。新的价格来了，通知来了。数据绑定会更新 TextView，TextView 自动进行重新测量。如果你是用手工写这些功能的代码，那你就不太可能写下这样的代码，而是会重新关闭再加载新内容。所以这是一个额外的好处。

Data Binding 的另一个性能好处是在你有两个表达式的情况下，例如：

```xml
<TextView android:text="@{user.address.street}"/>
<TextView android:text="@{user.address.city}"/>
```

你有一个 `user.address` 和另一个 `user.address`，Data Binding 将会为此生成如下代码：

```java
Address address = user.getAddress();
String street = address.getStreet();
String city = address.getCity();
```

它将把 address 移到一个本地变量，然后进行操作。现在想象一下，有一些重复计算是比较耗费性能的。数据绑定只会做一次。这是另一个好处，不需要你手动去做这些优化。

关于性能的另一个积极作用是针对 `findById`。当你在Android系统上使用代码`findById`，它实际上是对它对所有子内容说：“孩子，你能通过ID找到这个 view 吗？”那个孩子（子内容）问自己的孩子，然后找不到就去问下一个孩子，直到你找到了这个 view。然后，你的代码`findViewById`对于其他 View 会再重新做一次，重复的没必要的同样事情再次发生。

然而，当你初始化数据绑定时，我们实际上知道在编译时我们对应的 View，所以我们有一个方法来找到我们想要的所有 View。我们只需遍历布局层次一次，就可以收集到所有的 View。这个计算只对于一个布局只发生一次。当第二次我们需要另一个其中的子 View 的时候，不需要再次遍历所有子 View，因为我们已经找到了所有的 View。

性能有时关键就在一些小细节上。你在你的代码中包含一个库，一些行为会改变，但有时也会有一些成本。但有了这些性能上的优化，我认为我们做了它值得了，甚至有时比你写的代码还好，这是非常重要的。

## RecyclerView 和 Data Binding [(22:14)](javascript:presentz.changeChapter(0,71,true);)

使用 ViewHolders 对于 ListView 是很常见的，在 RecyclerView 中也是强制实施这个模式。如果你看看 Data Binding 生成的代码，你会发现它实际上会产生 ViewHolder。并带有变量属性，它绑定着那些 View。你也可以很容易地在 RecyclerView 里面中使用。我们创建了一个 ViewHolder，有一些基本方法，和一个静态方法，它传递参数给 UserItemBinding（根据用户的布局文件自动生成的）。你需要调用 UserItemBinding 的 inflate。现在你有一个非常简单的 ViewHolder 类，绑定方法类似这样：

```java
public class UserViewHolder extends RecyclerView.ViewHolder {

  static UserViewHolder create(LayoutInflater inflater, ViewGroup parent) {
    UserItemBinding binding = UserItemBinding
        .inflate(inflater, parent, false);
    return new UserViewHolder(binding);
  } 

  private UserItemBinding mBinding;

  private UserViewHolder(UserItemBinding binding) {
    super(binding.getRoot());
    mBinding = binding;
  }

  public void bindTo(User user) {
    mBinding.setUser(user);
    mBinding.executePendingBindings();
  }

}
```

其中有一个小细节要小心，就是调用这个`executePendingBindings`，当你的数据还无效的时候，数据绑定是等到下一个动画帧之前才设置布局。这就让我们不可以一次性批量绑定完所有的数据内容，因为 RecyclerView 的机制并不是这样。当要绑定一个数据的时候，RecyclerView 会调用 BindView，让你去准备测量这个布局。这就是为什么我们叫这个方法为 `executePendingBindings`，它使数据绑定刷新所有挂起的更改。否则，它将视为另一个布局失效了。

对于 `onCreateViewHolder`，它只是调用第一个方法，和 `onBind` 传到 ViewHolder 对象。就是这样，我们没有写 `findViewById`，没有设置。一切都已经在你的布局文件中封装好了。

```java
public UserViewHolder onCreateViewHolder(ViewGroup viewGroup, int i) {
  return UserViewHolder.create(mLayoutInflater, viewGroup);
}

public void onBindViewHolder(UserViewHolder userViewHolder, int position) {
  userViewHolder.bindTo(mUserList.get(position));
}
```

在前面的代码中，我们显示了一个非常简单直接的实现。比如说，用户对象的名称更改了。绑定系统将关联它，并重新布局在下一个动画帧。下一个动画帧开始，计算出发生了什么变化，和更新 TextView。然后，TextView 说，“好吧，我的文字已经变了，我已经重新布局，现在的情况是我不知道我的新尺寸。让我们去告诉 RecyclerView 它的这个孩子有点困难吧，它需要重新布局它本身。”当这一切发生的时候，你不会得到任何的动画，因为你在一切都*发生了之后*才告诉 RecyclerView。RecyclerView 将尝试修复自身。**结果：就没有动画了**，但这不是我们想要的。

我们希望发生的是，当用户的对象是无效的，我们告诉适配器项目已经改变了。反过来，它会告诉 RecyclerView，“嘿，你的一个孩子要改变，自己做好准备。”RecyclerView 知道了布局和那些子 View 已经改变了，它会指导他们重新绑定。当他们重新绑定，TextView 会说，“好吧，我的文字设置好了，我需要布局。”RecyclerView 会说，“好了，别担心，我准备好了，让我量量你。”**结果：很多动画**。你会得到所有的动画，因为一切都发生 RecyclerView 控制下。

## 绑定回调和有效载荷 [(25:50)](javascript:presentz.changeChapter(0,76,true);)

这实际上是我们需要发布为一个库的内容，但在此期间，我想让你知道如何做到这一点。在数据绑定时，我们有一个API，你可以添加一个绑定回调。当数据绑定将要计算，通过这个回调，可以得到通知。例如，也许你可能希望冻结更改用户界面。你可以“勾住” `onPreBind`，此时会返回一个布尔值，你可以说，“不，不用再绑定它了。”如果一个监听器插入进来，就会说，“嘿，我取消了绑定。你得先调用我，因为我准备告诉你不要做任何事情。”

现在我们要做的是如果 RecyclerView 不计算你的布局，返回 false。View 不需要更新。这个新的 RecyclerView API，将会在今年夏季发布。当 `onCanceled`来临的时候，我们只是告诉适配器，“嘿，这项改变了，去把它的位置找出来。”我们已经知道它在 holder 中的位置了后，让 RecyclerView 去更新它：

```java
public UserViewHolder onCreateViewHolder(ViewGroup viewGroup, int i) {
  final UserViewHolder holder = UserViewHolder.create(mLayoutInflater,   
      viewGroup);

  holder.getBinding().addOnRebindCallback(new OnRebindCallback() {

    public boolean onPreBind(ViewDataBinding binding) {
      return mRecyclerView != null && mRecyclerView.isComputingLayout();
    }

    public void onCanceled(ViewDataBinding binding) {
      if (mRecyclerView == null || mRecyclerView.isComputingLayout()) {
        return;
      }
      int position = holder.getAdapterPosition();
      if (position != RecyclerView.NO_POSITION) {
        notifyItemChanged(position, DATA_INVALIDATION);
      }
    }

  });
  return holder;
}
```

以前，我们只有一个 `onBind` 方法，所以我们开始写这个新的 RecyclerView API，通过这些 API 你可以得到载荷的列表。它是 ViewHolder 中内容改变的列表。关于这个 API 很酷的是，只有当你的 RecyclerView 要重新绑定相同的 View 时候，你才会收到这些载荷。你知道的，那个 View 已经显示了相同的项目，但也有一些不一样的变化你想执行。我们返回的数据无效标识会到这里。如果接受到它，我们就可以调用 `executePendingBindings`。你还记得我们没有让它更新自己吗？现在，是时候更新本身，因为 RecyclerView 叫它这么做了。

Data Binding 是怎么做到的呢，其实是通过简单遍历的这些载荷，并检查该数据，验证是它收到的唯一的有效载荷。例如，也许别人也在发送一些你不了解的载荷，你应该将它们传递出来，因为你并不知道这边变化是什么。

```java
public void onBindViewHolder(UserViewHolder userViewHolder, int position) {
  userViewHolder.bindTo(mUserList.get(position));
}

public void onBindViewHolder(UserViewHolder holder, int position,
      List<Object> payloads) {
              notifyItemChanged(position, DATA_INVALIDATION);
...
}
```

我们会将这发布为一个库，因为它给了你更好的表现，给了你动画，和让一切都变得更好，让 RecyclerView 使用起来更加愉快。Data Binding 将更像是它的一个快乐孩子！

“数据无效”只是一个简单的对象，但我想你可能比较好奇它是什么样的，所以我可以给你看一下:

```java
static Object DATA_INVALIDATION = new Object();
private boolean isForDataBinding(List<Object> payloads) { 
  if (payloads == null || payloads.size() == 0) {
    return false;
  }
  for (Object obj : payloads) {
    if (obj != DATA_INVALIDATION) {
      return false;
    }
  }
return true;
}
```

## 多种类型的视图项目 [(28:50)](javascript:presentz.changeChapter(0,81,true);)

另一个使用数据绑定的案例是多种视图类型。经常会发生：你有一个头视图，或者类似从谷歌获得的搜索结果，在那里你可能有一个照片结果，或一个链接文本结果。如果是 RecyclerView，你要怎么去构建？你有一个布局文件，使用一个变量，你把它命名为“data”，这个名字“data”是很重要的，因为你将重用相同的名字。你使用常规布局文件：

```xml
<layout>
    <data>
        <variable name="data" type="com.example.Photo"/>
    </data>
    <ImageView android:src="@{data.url}" …/>
</layout>
```

如果你需要另一种类型的结果，比如说叫“place”，那么你就需要有一个完全不同的布局，另一个XML文件：

```xml
<layout>
    <data>
        <variable name="data" type="com.example.Place"/>
    </data>
    <ImageView android:src="@{data.url}" …/>
</layout>
```

这两个布局文件之间唯一共享的，就是变量名，即所谓的“data”。当我们这样做时，我们创造的东西称为 `DataBoundViewHolder`：

```java
public class DataBoundViewHolder extends RecyclerView.ViewHolder {

  private ViewDataBinding mBinding;

  public DataBoundViewHolder(ViewDataBinding binding) {
    super(binding.getRoot());
    mBinding = binding;
  }

  public ViewDataBinding getBinding() {
    return mBinding;
  }

  public void bindTo(Place place) {
    mBinding.setPlace(place);
    mBinding.executePendingBindings();
  }

}
```

然后我们看一下下面这个，这是一个和上面几乎一样的 holder，这是一个能够保持绑定的数据绑定基类，这个基类将作为给所有生成的代码的父类，这就是为什么这个类能保持绑定的引用。我们之前创建的那个绑定 user 的 bindTo 方法，现在它将可以绑定 place。

不幸的是，在这个基类当中并没有一个 `setPlace` 方法，但幸运的是我们提供了另外一个 API 叫做 `setVariable`：

```java
public void bindTo(Object obj) {
  mBinding.setVariable(BR.data, obj);
  mBinding.executePendingBindings();
}
```

你可以提供需要绑定的目标 ID 和对应的整个对象，自动生成的代码会去检查它的类型，并将其赋值。

这个 `setVariable` 看起来像这样：“如果传进来的 ID 是我知道的，我就将它转型并且赋值。”

```java
boolean setVariable(int id, Object obj) {
  if (id == BR.data) {
    setPhoto((Photo) obj);
    return true;
  }
  return false;
}
```

对于 `onBind`, `onCreate` 方法类似以往的写法，比较特别的就是 `getItemViewType`，我们返回布局文件的 ID 作为这个方法的返回值。RecyclerView 会将它传递给 `onCreateViewHolder`，`onCreateViewHolder` 将通过 DataBindingUtil 去创建对应的绑定类。这样所有项目都会有它们的布局，你不需要手动去创建这些不同布局类型的对象了。

```java
DataBoundViewHolder onCreateViewHolder(ViewGroup viewGroup, int type) {
  return DataBoundViewHolder.create(mLayoutInflater, viewGroup, type);
}

void onBindViewHolder(DataBoundViewHolder viewHolder, int position) {
  viewHolder.bindTo(mDataList.get(position));
}

public int getItemViewType(int position) {
  Object item = mItems.get(position);
  if (item instanceof Place) {
    return R.layout.place_layout;
  } else if (item instanceof Photo) {
    return R.layout.photo_layout;
  }
  throw new RuntimeException("invalid obj");
}
```

当然，如果你是在一个生产应用程序中，你可能会保留做实例检查。你应该有一个基本的类，知道如何返回布局，这是通常的想法。

## 绑定适配器和回调 [(31:27)](javascript:presentz.changeChapter(0,91,true);)

根据民意测验，接下来讲的可能是最酷最受欢迎的特点…（我所做的）。它甚至可能是 Android 开发中最酷的功能。好吧，也许是我在炒作，哈哈。

让我们想象一下，有什么比 `setText` 更复杂，或许可以是一个图像的URL。你想设置为 ImageView，你需要设置一个图片的网址。当然，你不想在 UI 线程上做这件事（请记住，这些东西是在 UI 线程上进行评估）。你想用 Picasso 或其他图片加载库。于是或许你会这么做：

```xml
<ImageView …
    android:src="@{contact.largeImageUrl}" />
```

这基本上无法正常工作。context 要从哪里来，你把它放在什么地方？这里取不到 view。所以正确的写法应该是创建一个 BindingAdapter：

```java
@BindingAdapter("android:src")
public static void setImageUrl(ImageView view, String url) {
    Picasso.with(view.getContext()).load(url).into(view);
}
```

现在这里的 BindingAdapter 是一个注解。这是为了我们设置的 android:src 和 Android 源码的关联。这是一个静态的方法，它需要2个参数。它需要一个View 和一个字符串，但注意它也可以采取其他类型。如果你想要一个不同的方法，接受一个 int 参数或 drawable，你也是可以做到的。你可以把任何你想要的填入这里。在这种情况下，我们已经支持了 Picasso 这个库的使用了。我们所有的代码都在那里。既然我们有了这个 View，我们就可以得到它的背景。我们可以写任何我们想写的代码。现在我们可以在用户界面上把图像加载出来，如你所愿。

## 利用属性 [(33:12)](javascript:presentz.changeChapter(0,97,true);)

你可能想做一些更复杂的内容，比如说，我们想设置一个 PlaceHolder，设置它的图片源文件，或者它的图片 URL：

```xml
<ImageView …
   android:src="@{contact.largeImageUrl}"
   app:placeHolder="@{R.drawable.contact_placeholder}"/>
```

我们有两个不同的属性，同时他们对应的有两个不同的静态方法，但数据绑定却不知道，所以目前还不能直接如你所愿地进行工作。实际上，现在我们在 BindingAdapter 注解中有两个同样的属性，你只要取得这里他们的值，把他们传递给 Picasso 正确的方法即可：

```xml
<ImageView …
   android:src="@{contact.largeImageUrl}"
   app:placeHolder="@{R.drawable.contact_placeholder}"/> 
```

```java
@BindingAdapter(value = {"android:src", "placeHolder"},
                requireAll = false)
public static void setImageUrl(ImageView view, String url,
                               int placeHolder) {
    RequestCreator requestCreator =
        Picasso.with(view.getContext()).load(url);
    if (placeHolder != 0) {
        requestCreator.placeholder(placeHolder);
    }
    requestCreator.into(view);
}
```

如果你现在有三个属性呢？有一个是 Android 源码的图片源属性，一个是占位图片属性（PlaceHolder），还有一个是设置图像的 URL 属性。所有这些不同的 BindingAdapter。真的，我有点太懒了，所以让我们做点别的了。我们可以有一个 BindingAdapter 注解需要所有这些属性，也可以是只要一个或两个，或它们的任意组合。我们所要做的就是把所需的一切都设为 false，然后我们获得所有这些参数。如果有值就会传递过来，如果没有提供，它会将它们作为默认值传递给它们。如果你没有在你的布局中写一个占位符属性，那么占位符将为零。我们在 Picasso 中调用 setter 的时候会进行检查。

## 旧的值 [(34:55)](javascript:presentz.changeChapter(0,98,true);)

你可能也需要一些旧的值。在这个例子中，我们有一个 OnLayoutChanged：

```java
@BindingAdapter("android:onLayoutChange")
public static void setOnLayoutChangeListener(View view,
        View.OnLayoutChangeListener oldValue,
        View.OnLayoutChangeListener newValue) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
        if (oldValue != null) {
            view.removeOnLayoutChangeListener(oldValue);
        }
        if (newValue != null) {
            view.addOnLayoutChangeListener(newValue);
        }
    }
}
```

我们想在我们添加一个新的值的时候，删除一个旧的值，但在这个场景中，我们不知道旧的那个值是什么。我们可以添加新的，这很容易，但我们如何删除旧的呢？嗯，其实你也可以获得这个值，我们会给你的。如果你有这种代码，我们将会知道要把旧的直给你。对于这样的情况，它将在你的内存中，这样很好。每次它改变，比如开始正确的动画，你会想知道它之前是什么。

只要使用这个 API，我们的数据绑定库就会实现你想要的。你只需考虑如何响应这个变化。当然，你也可以使用多属性来实现同样的结果。我们会把值都传给你。

## 依赖注入 [(36:20)](javascript:presentz.changeChapter(0,99,true);)

让我们想象一下我们有一个这样的适配器：

```java
public interface TestableAdapter {
    @BindingAdapter("android:src")
    void setImageUrl(ImageView imageView, String url);
}

public interface DataBindingComponent {
    TestableAdapter getTestableAdapter();
}

DataBindingUtil.setDefaultComponent(myComponent); 
 ‐ or ‐
binding = MyLayoutBinding.inflate(layoutInflater, myComponent);
```

显然将会发生的事情是，它会调用我设置的这个绑定目标 `setImageUrl`。如果我有一些状态，我想在我的 BindingAdapter 中获取呢？或者说，我有各种不同的 BindingAdapter 取决于我在我的应用程序中想做的。在这种情况下，它会是一种痛苦。我们想要的只是其中的一个 BindingAdapter 实例。它要从哪里来？

我们所能做的，就是创建一个绑定组件，`DataBindingComponent`，这是一个接口。当您有一个实例方法，我们将通过这个接口生成我们具体的适配器。它是由你来实现。我们不知道你是如何实现它，但你实现它，然后你可以设置默认组件（component）。

你也可以按每一个布局来做这件事。在这种情况下，一个设置好的默认值，它可以使用在所有的布局。然后我们知道用什么组件来得到你的适配器。

你可能还想用你的组件作为参数。例如，我们刚才看到了这 `setImageURL` 方法。

```java
@BindingAdapter("android:src")
public static void setImageUrl(MyAppComponent component, 
                               ImageView view, 
                               String imageUrl) {
    component.getImageCache().loadInto(view, imageUrl);
}
```

我们要用某种状态。让我们想象一下，图像缓存，我们要加载的图像与图像缓存。那这些项相关状态来自哪里？我们要做的是使用组件。你可以在这里放置你想要的任意方法：在这种情况下，它是[somestate].get[somestate]。你要把它作为你的 BindingAdapter 第一个参数，然后你可以做任何你想做的事情。我们不知道你的组件在做什么，对吗？这是你想做的，很方便的。

## Event Handlers（事件处理） [(38:56)](javascript:presentz.changeChapter(0,101,true);)

我们有一个 `onClick` 属性，我们还有一个 `clicked` 方法在 handler 上，`clicked` 有可能对应的是 `getClicked` 或者 `isClicked` 方法，也可能是一个变量属性，”clicked”，所以这种情况下我们要怎么才能确定这个事情呢？

```xml
<Button …
 android:onClick="@{isAdmin ? handler.adminClick : handler.userClick}" />
```

```java
// View 中并没有 "setOnClick" 方法，需要开发者帮忙去寻找到属性对应的方法
@BindingMethods({
    @BindingMethod(type = View.class,
                   attribute = "android:onClick",
                   method = "setOnClickListener"}) 
// 在 View 中寻找 setOnClickListener  
void setOnClickListener(View.OnClickListener l) 

// 在 OnClickListener 中寻找到单一的一个抽象方法  
void onClick(View v);
```

首先，我们需要明白 `onClick` 对应的意思。我们知道 `onClick` 并不是 `setOnClick`，因为我们找不到 `setOnClick` 方法，但有一个绑定方法。这个绑定方法说：“`onClick` 表示 `setOnClickListener`，所以我们着眼于 `setOnClickListener` 方法，它需要一个 `onClickListener` 参数，让我们来看一下它。

在 `onClickListener` 中只有一个抽象函数，所以我们可以确定地知道它就是这个事件监听的处理者（handler）。现在我们看看这个处理者，我们找到一个方法，叫做 `clicked`.

```java
static class OnClickListenerImpl1 implements OnClickListener {
    public Handler mHandler;
    @Override
    public void onClick(android.view.View arg0) {
        mHandler.adminClick(arg0);
    }
}
static class OnClickListenerImpl2 implements OnClickListener {
    public Handler mHandler;
    @Override
    public void onClick(android.view.View arg0) {
        mHandler.userClick(arg0);
    }
}
```

我们找到了这个 `clicked` 方法，它需要一样的参数，我们正好有匹配的参数，我们知道这就是一个事件处理者，所以我们知道该怎么做了：我们会把它视为一个事件，这个事件有它的处理者。

那么你在 TextWatcher 的情况下怎么办？因为 TextWatcher 没有单一的抽象方法，而是有三个抽象方法。在这种情况下，你所要做的就是你弥补自己的接口，然后你将它们合并在一起。实际上，这就是我所做的，我把他们合并放在一起。从本质上讲，你在做什么，就等于合并所有“在什么之前”和“在什么之后”的变化。如果他们是为 null，那么你就不做任何操作，如果他们不是 null，那么你做一些事情，结合你所需要的做就行了。

## 最佳实践 [(41:32)](javascript:presentz.changeChapter(0,103,true);)

数据绑定正在逐渐变得更好，我们想分享这个信息。这是一个非常新的库，所以我们仍然在调查人们如何使用它，以及人们在什么场景中使用它。也有一些事情我们知道需要小心的。在使用数据绑定时，这里有一些是你应该记住的事情。

首先，你*能*在 XML 中使用一些表达式，但并不意味着你*应该*这么做。在表达式中应该没有业务逻辑！因为这不是这个库的目的。如果你试着这么做：“当这个按钮被调用，让我通过网络服务发送信息给某人”，你可能认为“太好了，现在我不需要写任何 Java 代码了！”

```xml
<ImageView android:click="@{webservice.sendMoneyAsync}"/>
```

可是对不起，你错了。这不是关于不用写任何 Java 代码的问题，而是如果你这样做，你的代码架构将崩溃。正确的做法应该是，任何你在这里放的东西应该是关于用户界面的。

```xml
<ImageView android:click="@{presenter.onSendClick}"/>
```

如果用户点击了某个界面交互的部分，那么最好的就是调用你的处理者来处理点击。其余的是你的应用程序逻辑，它们是数据绑定的范围之外的，所以不要尝试把它放在那里。

不过，简单的用户界面逻辑是值得欢迎的。这是另一个例子：

```xml
<ImageView android:src="@{user.age > 18 ? @drawable/adult : @drawable/kid}"/>
```

这正是你想展示的用户界面，所以把代码逻辑放在表达式里。当我看到这个 ImageView，我清楚地明白它绑定的图片源是什么。这意味着，这是最好的开发实践，试图找到相关的代码，并找出在这种情况下，它需要显示的是什么。只要它是简单和清晰的，就可以把它放在 XML 表达式那里。

保持你的表达式简单。它要能够易读，以至于你看一眼就知道它要显示的。下面是另一个例子。你能看出它要显示的是什么吗？

```xml
<TextView android:text='@users.age > 18 ? user.name.substring(0,1).toUpperCase() + "." + user.lastName : @string/redacted'/>
```

不要这么做，这么做不好。这样就不再易读了，因为有人试图把东西挤在里面。相反，将它移到另一个方法，通过方法名来显露它的意图，这样才好。

```java
<TextView android:text='@{user.age > 18 ? user.displayName : @string/redacted}'/>                                          
        
@Bindable
public String getDisplayName() {
  return mName.substring(0, 1).toUpperCase()  
      + "." + mLastName;
}

public void setLastName(String lastName) {
  …
  notifyPropertyChanged(BR.displayName);
} 
```

现在这个布局文件是可读的。把复杂的逻辑代码放到你的 Java 代码去。你只需要告诉数据绑定，什么时候该更新它自己，然后去调度需要通知事件即可。尽量保持简单明了最好了。

试想一下使用一个 ViewModel。ViewModel 的作用就是，保持你想要在 UI显示的信息。通过你的 XML 访问对象是完全可以的，只要他们是一目了然的代码，所以把 user.name 写在那里是可取的，因为数据对象应该只持有和它相关的对象。但是，如果你把显示名放在那里，这是你的用户界面信息，你不想把它放进你的对象。这显然也是一种情况，这些事情没有一个“一定要怎么做”的结论。这取决于你的团队和你的喜好，但你仍然可以使用这个ViewModel。

我的用户界面显示一个好友计数和订阅日期：

```java
public class ProfileViewModel {
  public final ObservableInt friendCount;
  public final ObservableField<Date> subscriptionDate;
}
```

设置这些东西，你的XML文件将引用视图模型。但有时你想要更好的分离。

```java
public class ProfileViewModel extends BaseObservable {
  private @Bindable String mDisplayName;

  public void setUser(User user) {
    mDisplayName.set(user.getName() + " " + user.getLastName());
    notifyPropertyChange(BR.displayName);
  }

  public String getDisplayName(){…}
}
```

这个视图模型（view model）仅仅是一个值的持有者，你将它设置给你的 activity 或者你的 presenter。如果你不喜欢它，或者如果你想对这些东西做一些抽象，你可以写你的 ViewModels，同时继承基础 observer 类。

以下这有一个私有的属性，我知道它会接受用户的信息，因为这是关于用户的配置文件。无论它想显示什么到这个界面，他们都需要有一个用户。他们把它放到你的用户界面，然后你设置你的变量属性。当然，你需要尝试改变，你需要一个 getter，使数据绑定可以接收那个属性。我们知道这是可绑定的，但是数据绑定将永远不会使用反射，所以没有办法得到私有属性内容。所以，你必须为这个属性提供一个公开的 getter 方法。

```java
public class ProfileViewModel implements Observable {
  private @Bindable String mDisplayName;
  PropertyChangeRegistry mRegistry = new PropertyChangeRegistry();

  public void setUser(User user) {
    mDisplayName.set(user.getName() + " " + user.getLastName());
    mRegistry.notifyPropertyChange(BR.displayName);
  }

  public String getDisplayName(){…}

  public void addOnPropertyChangedCallback(OnPropertyChangedCallback cb)
    mRegistry.add(cb);
  }

  public void removeOnPropertyChangedCallback(OnPropertyChangedCallback cb) {
    mRegistry.remove(cb);
  }
}
```

这对于测试是很友好的。它是孤立的，它与用户界面无关，但它也可以给用户界面内容。如果你不想扩展代码，你可以实现基础 observable 接口。你可以在内部使用 PropertyChangedRegistry 这个辅助类。只是将所有的这些方法都转发到这个类，然后你想要的任何对象都可以被观察到。

这就是为什么它是最流行的功能。我看到人们使用这个他们渴望了很多年的功能实现了各种很酷的东西，真是高兴。下面是一个例子，一个 Android 开发者创建的：

```java
<TextView app:font="@{`Source-Sans-Pro-Regular.ttf`}"/> 

public class AppAdapters {

  @BindingAdapter({"font"})
  public static void setFont(TextView textView, String fontName){
    AssetManager assetManager = textView.getContext().getAssets();
    String path = "fonts/" + fontName;
    Typeface typeface = sCache.get(path);
    if (typeface == null) {
      typeface = Typeface.createFromAsset(assetManager, path);
      sCache.put(path, typeface);
    }
    textView.setTypeface(typeface);
  }
}
```

开发者想要这个字体属性，它原本是不存在的，所以她决定使用数据绑定来创建它。其实很多人都梦想过有这么一个方法！文件也将更清晰明了，事情也会更加集中。

这里是另一个例子，使用 DataBindingComponent。这些适配器的大部分优点就是他们都是静态的，但静态的在测试方面并不是很好，他们没有很好的可测试性。所以，你创建一个 ImageAdapter，方法不要再是静态的了。

```java
public class MockImageAdapter extends ImageAdapter {

@BindingAdapter(value={"photoUrl", "default"}, requireAll = false)
  public void setUrl(ImageView imageView, String url, Drawable
      defaultImage) {
    Glide.with(imageView.getContext())
        .load(url).into(imageView)
        .onLoadStarted(defaultImage);
  }
}

public class AppComponent implements DataBindingComponent { 
  ImageAdapter mImageAdapter = new ImageAdapter();

  public ImageAdapter getImageAdapter() {
    return mImageAdapter;
  }
}    
```

你写这个 app component，知道如何提供 GetImageAdapter。很酷的是你可以创建一个 MockImageAdapter 继承自你有的。然后在一些与用户界面无关的测试进行模拟运行，这里不用任何图片的网址，我们只是使用ImageDrawable。当然，您可以创建我们在测试中使用的测试组件，并将其提供给返回模拟图像组件的数据绑定。

如果你是使用依赖注入，这些组件能够非常好地与其一起工作。你实际上可以只注入这个：

```java
public class AppComponent implements DataBindingComponent { 
  @Inject ImageAdapter mImageAdapter;

  public ImageAdapter getImageAdapter() {
    return mImageAdapter;
  }
}
```

最后，这是一个真实的 Dagger 2 示例：

```java
@Component(modules = ProductionModule.class)
public interface ProductionComponent extends DataBindingComponent {} 

@Module
public class ProductionModule {
    @Provides
    TestableAdapter provideTestableAdapter() {
        return new ProductionAdapter();
    }
} 
DataBindingUtil.setDefaultComponent( 
    ProductionComponent.builder().build());
```

你仅需要做的是，有一个组件，继承数据绑定组件。然后就可以编译了。数据绑定将产生这个数据绑定组件，然后将 Dagger 将采取这一生产组件，并生成相应的方法给你。他们一起能很好地共同工作。所以说，这真的是一个免费而且特别好的库。

## 问与答 [(50:35)](javascript:presentz.changeChapter(0,118,true);)

***问: 幕后自动重写的方法会和我自己实现的冲突吗？比如 setTag 方法。如果是这样，我就没有办法使用 Data Binding 了吗？***

**Yigit:**这个问题实际上有两个答案。一是，如果你在除了根布局的布局中使用 tag，那就完全可以了。另外，当我们做布局文件的解析时，我们会把你的 tag 去掉，然后把它放回我们的数据绑定。

***问: 是否有方法可以创建一个注解或控制一些 Java 代码的生成？我知道使用 Gradle 可以提供某种布局处理器，那么数据绑定也可以吗***

**Yigit:** 这实际上是一种 hack 做法。如果你尝试使用它作为一个单独的插件，我们还没有集成。我不知道我们是否会提供一个接口，但如果有一些用例，也许你应该向工具组报告。

***问: 你们是否有计划增加一些功能，例如在 xml 中定义的表达式中支持 lint？***

**George:** 我们有另一个可谓悬而未决的功能，如果你编译失败，你可以点击并进入错误。你将能够更准确地获得错误报告。这功能会随着时间的推移而改善。现在，我们还处于“婴儿阶段”。它只是进入Android Studio。如你所说，我们谈论的是 Gradle 插件，但我们没有成为 Android Studio 插件的一部分。随着时间的推移，它会变得越来越好。我想在某个时候，你不会再想不使用数据绑定，它会你的应用程序必要的一部分。

***问: 如果你要生成图形用户界面的代码，你可以使用 Data Binding 吗？***

**George:** 不可以。

***问: 如果你要绑定适配器，你能预先进行一些定制，当屏幕上有某些变量的变化，你可以在代码中安装一个回调来广播这个事件吗？***

**George:** 你可以在布局中添加一个事件监听器，它可以调用回你的代码。然后你可以在你的代码中做任何你想做的事，包括改变或触发通知的变更。

***问: 可以使用嵌套布局吗，使用包含（include）和合并（merge）？***

**George:** 您可以使用 include。唯一限制是你不能把它们作为根标签。不过你通常也无法将它们作为 root。在正常布局中，如果你有一个 merge 标签，你可以有多个 include 在根布局中。在数据绑定中，你不能这样做，但这是唯一的限制。

**Yigit:** 如果你包含一个不继承任何变量的布局，这个布局有它自己的变量，你可以将所有的变量传递到这个包含的布局中，还可以对这些变量进行重命名。

**George:** 你也可以有一些绑定表达式，发送一些不一样的内容到 include 布局中。

***问: 这些实现会影响 debug 调试吗？能否支持单步调试？***

**Yigit:** 目前来说，如果你只是使用数据绑定库，你不能直接看到生成的代码。你可以配置 Android 的视图，但默认情况下，任何视图的代码都会自动完成，如果你要绑定你的用户，当你点击它，它将带你到 XML 文件。这是他们的联系。然而，如果你结合 APT 使用它，你可以看到生成的代码。但是，如果你使用生成的代码，你就会失去对代码的实时更新。现在，结合 APT，如果你改变了你的 XML 文件，您可以轻松地开始使用新的 Java 文件，而无需重新编译。我们实际上已经提供了你想要的了，在这些开发的地方，你可以和 XML 相互作用。但是，如果你在调试时尝试去那些类，它得带你到执行的代码行。但它还没有，它还需要一段时间。

***问: 你是在哪里遇到 Data Binding 的？是什么促使你把它添加到 Android 开发当中？在其他框架中，你看到过类似的东西吗？***

**George:** 我不知道这么说是否具体。我们在别的一些框架上也用过 Data Binding，不过我们还是把它做得很 Android 化，所以说它并不是从其他一些数据绑定库中复制过来的。

**Yigit:** 在我们以前的专业作品中，我在做 Adobe Flex 做了很多工作，所以你可以说它是相似的。然而，这些平台都遭受了数据绑定的性能问题。 如果你去看 [维基百科中关于 MVVM 到文章](https://en.wikipedia.org/wiki/Model_View_ViewModel), 他们提到了数据绑定的性能问题。从第一天起，我们就觉得：Android 的数据绑定是永远不能慢。我们最好不要释放出一些慢的东西，因为这样它不会有帮助。

***问: 对于这样一个新的 XML 代码层内容在测试方面有什么变化？***

**George:** 我们正在做的本质上是想，如果你写你自己的代码，你会做什么。我们在幕后执行方法，所以在这方面没有什么变化。

**Yigit:** 在测试方面，它取决于你的应用程序的偏好。如果你的团队喜欢写集成测试，一些人就这样做，他们写的是重的集成测试，测试用户界面，这样他们会被覆盖。如果你的团队更喜欢做单元测试的样式，最好使用视图模型，你可以轻松测试。如果一些东西改变了，你只需要测试的视图模型（view model）。这是数据绑定库，这不是你的代码。但这确实取决于你的应用架构。我们提供这些灵活的组件，所以你可以在测试阶段注入不同的东西。在可测性方面，没有什么变化。事实上，我认为我们提供了一个更好的模型，使它可以更容易测试。有一个好消息就是，Android Studio 会持续变得越来越好，我可以想象在一些遥远的未来日期，它将允许您将数据注入到您的布局，并能够看到它。