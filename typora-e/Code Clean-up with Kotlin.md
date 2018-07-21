# 使用 Kotlin 清理代码

>原文 (Medium)：[Code Clean-up with Kotlin](https://proandroiddev.com/code-clean-up-with-kotlin-19ee1c8c0719)
>
>作者 ：[Gabor Varadi ](https://proandroiddev.com/@Zhuinden?source=post_header_lockup)

自从十一月份那篇[关于 Dagger 的文章](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97)以来，我还没有写过一篇完整的文章！ 显然，这种情况不可能永远这样下去，不是吗？

然而，忙于工作和忙碌通常是一件好事ーー特别是如果你的工作报酬是用100% 的 Kotlin 编写的！ 一段时间后，在 Java 中你认为理所当然的东西看起来就像是无意义的混乱，可以用 Kotlin 语言特性重构—— Java 的语言规范不允许这样做。

我见过很多人(包括我在内)认为"Kotlin 的使用直接导致了一个难以理解的混乱"; 但显然有一些规范，它可以写出更简洁，更易读的代码。

因此，我们不再赘述了，这里有一些技巧和技巧，可以减少 Java 中的混乱，同时保持它的可读性而不只是简短。

### 1.) 能够将控制流关键字用作分配的一部分

虽然这看起来很基本，但我经常看到新写的 Kotlin 样本根本没有使用这个功能。

在 Java 中，人们可以很容易地写出这样的代码:

```Java
String name;
if(something) {
    ...
    name = "Something";
} else {
    name = "Other thing";
}
```

在 Kotlin，我们可以这样减少这种情况:

```kotlin
val name = if(something) {
    ...
    "Something" 
} else "Other thing"
```

就我个人而言，我并不喜欢放弃 if 语句的 { 和 } ，我甚至设置自动格式化程序来强制大括号。 我们可以使用 when，允许我们很好地封装这块代码时，我们可以做得更好。

```kotlin
val name = when {
    something -> {
        ...
        "Something"
    }
    else -> "Other thing"
}
```

不错的是，我们可以把这个更进一步ーー我们甚至可以在 {…} 时写出类似 return 的内容。

### 2.) 使用内联通用扩展函数来减少重复

#### a.) apply

有时候，我们有一些我们根本不能用 Java 来减少的设置代码。 一个很好的例子是人们倾向于为片段定义静态工厂方法 newInstance ( ) 。

```Java
public class CatFragment extends Fragment {
    public static CatFragment newInstance(String catId) {
        CatFragment catFragment = new CatFragment();
        Bundle bundle = new Bundle();
        bundle.putString("catId", catId);
        catFragment.setArguments(bundle);
        return catFragment;
    }
}
```

然后我们有另一个片段，以一种非常相似的方式初始化:

```kotlin
public class DogFragment extends Fragment {
    public static DogFragment newInstance(String dogId) {
        DogFragment dogFragment = new DogFragment();
        Bundle bundle = new Bundle();
        bundle.putString("dogId", dogId);
        dogFragment.setArguments(bundle);
        return dogFragment;
    }
}
```

现在对于 Kotlin，我们可以保留这个完全相同的逻辑:

```kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String): CatFragment {
            val catFragment = CatFragment()
            val bundle = Bundle()
            bundle.putString("catId", catId)
            catFragment.arguments = bundle
            return catFragment
        }
    }
}
```

事实上，如果你在网上检查样本，这就是你经常发现的。

然而，我用 val 定义的所有局部变量？ 他们完全没有必要。 我可以使用标准的通用内联函数 apply { } ，并在 lambda 中移动这些实例。

```kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String) = CatFragment().apply {
             arguments = Bundle().apply {
                 putString("catId", catId)
             }
        }
    }
}
class DogFragment: Fragment() {
    companion object {
        fun newInstance(dogId: String) = DogFragment().apply {
             arguments = Bundle().apply {
                 putString("dogId", dogId)
             }
        }
    }
}
```

我们已经消除了大量的复制和不必要的局部变量，我们只是为了“使事情有效”而写的。

然而，有人可能会说我们嵌套了很多。 我们正在复制 `apply { arguments =` 的东西。 难道不能做得更好吗？ 当然可以，如果我们为每个片段定义自己的扩展函数。

我们可以编写这个代码:

```kotlin
class CatFragment: Fragment() {
    companion object {
        fun newInstance(catId: String) = CatFragment().withArgs {
             putString("catId", catId)
        }
    }
}
class DogFragment: Fragment() {
    companion object {
        fun newInstance(dogId: String) = DogFragment().withArgs {
             putString("dogId", dogId)
        }
    }
}
```

withArgs 在哪里:

```kotlin
inline fun <T: Fragment> T.withArgs(
                             argsBuilder: Bundle.() -> Unit): T = 
    this.apply {
        arguments = Bundle().apply(argsBuilder)
    }
```

有了这个，我们的代码就更具可读性了，在 Kotlin 的内联关键字的帮助下，这种简化没有任何性能成本。

如果 withArgs 的签名看起来有点复杂，我向你保证ーー读这些东西在和 Kotlin 合作了一段时间后变成了第一手资料。 毕竟，我们只是在传递一个 lambda，并且围绕 this 是谁！我最初是在[这里](https://academy.realm.io/posts/kau-jake-wharton-testing-robots/)知道的。

#### b.) let

通常你会检查一些 Kotlin 代码，它看起来是这样的:

```kotlin
activity?.childFragmentManager
        ?.beginTransaction()
        ?.setCustomAnimations(...)
        ?.replace(...)
        ?.addToBackStack(...)
        ?.commit()
```

这么多 ?，显然我们可以让这一切变得更好？

```kotlin
activity?.let { activity ->
    activity.childFragmentManager
            .beginTransaction(...)
            .setCustomAnimations(...)
            .replace(...)
            .addToBackStack(...)
            .commit()
}
```

通过使用单一  ?.let 。我们可以移除所有的安全调用操作符！ 当然，还有一件事情需要提及，而不是让我们把这个参数重命名，我可以为它指定一个实际的名称。

对于长期使用 Kotlin 的用户来说，这可能并不令人惊讶，但你仍然会发现这样的例子:

```kotlin
placeAutocompleteResult.predictions.forEach {
    placeList.add(PlaceModel(it.description, it.placeId))
    placeNameList.add(it.description)
}
```

所以你看看它，你需要从上下文中找出它是什么。

我更喜欢写:

```kotlin
placeAutocompleteResult.predictions.forEach { prediction ->
    placeList.add(PlaceModel(prediction))
    placeNameList.add(prediction.description)
}
```

我尽可能避免使用 it。 它肯定解决了"嵌套 it"问题。

#### c.) takeIf

在某些情况下，例如:

```Kotlin
if(something != null && something.thingy) {
    ...
} else {
    ...
}
```

你可以把这个逻辑替换成

```Kotlin
something?.takeIf { it.thingy }?.let {
   ...
} ?: {
   ...
}
```

这在一些分配中是很方便的。 不过我不经常用 it。

### 3.) 使用 anko / kotlin-android-extensions 杀死 Android 视图样板

首先，[anko](https://github.com/Kotlin/anko) 是一个模块化的库，它的目的是帮助 Android 开发。 它也完全独立于 kotlin-android-extensions。

很多 anko 是一把双刃剑: 例如，我肯定不会使用 anko-layouts，anko-sqlite，可能也不用 anko-coroutines (但我个人并没有深入研究过)。

然而，anko 有两个有用的东西: anko-commons 和 anko-listeners。

```groovy
implementation “org.jetbrains.anko:anko-commons:0.10.4” 
implementation “org.jetbrains.anko:anko-sdk15-listeners:0.10.4”
```

一旦我们添加了这些，我们也会应用

```groovy
apply plugin: ‘kotlin-android-extensions’
```

我们现在可以替换所有的视图绑定

```kotlin
@BindView(R.id.my_button)
lateinit var myButton: Button
@OnClick(R.id.my_button)
fun onButtonClick() {
    ...
}
override void onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.my_activity)
    ButterKnife.bind(this)
}
```

用

```kotlin
import kotlinx.synthetic.my_activity.myButton
override void onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.my_activity)
    myButton.onClick {
       ...
    }
}
```

我曾经喜欢 ButterKnife，但是因为没有人(包括我)写过一个注解处理器，它可以生成基于 @BindView 和 @OnClick 注解的 Espresso 测试的页面对象，我想这个转变是值得的。

(编辑来自未来: 事实上，检查我的库 [espresso-helper](https://github.com/Zhuinden/espresso-helper)，利用扩展功能!)

不要忘了在 RecyclerView 的 viewhoders 中使用 LayoutContainer。

此外，如果你使用 kotlin-android-extensions，请使用 camelCase 作为视图 id。

### 4.) 用@Parcelize 杀死 Parcelable 的样板

手动实现 Parcelable 的实现是一种痛苦。 生成一个 Parcelable 的实现是容易的，但维护困难。 auto-value-parcel 不错，但是如果我们能让它变得更容易呢？

果然，多亏了 kotlin-android-extensions，我们现在可以使用 @Parcelize 来转化 parcelable 的东西。

```kotlin
androidExtensions { 
    experimental = true 
}
```

然后我们可以做:

```kotlin
@SuppressLint("ParcelCreator")
@Parcelize
data class CatKey(val clazz: String) : Parcelable {
    constructor() : this("CatKey")
    ...
}
```

@Parcelize 使用主构造函数来确定保存为 Parcelable 的内容。 对于额外的配置，可以用 object: Parceler\<T> 覆盖默认行为，如果需要，我们也可以提供 @TypeParceler。 你也可以 @IgnoredOnParcel 忽略属性。

不过，这些都在[官方页面](https://kotlinlang.org/docs/tutorials/android-plugin.html)上被描述了。 有时候文档比 Stack Overflow 更有帮助(因为我找不到任何关于它的问题——然而文档描述得很好!) .

### 5.) 关于 DI 框架的说明

我通常会暂时忽略 Kodein，或者[至少在5.0之前](https://salomonbrys.github.io/Kodein/?5.0/migration-4to5)。 这似乎会让 DSL 变得更好。

然而，在我看来，[Dagger 并不像人们常说的那么难](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97)，对于像 Koin 和 Kodein 这样的新兴"DI"库(或者 service locators)时，Dagger 仍然是最安全的选择。

```kotlin
@Singleton class MyRepository @Inject constructor(
    private val service: MyService,
    private val dao: MyDao
) { ...
```

只要你在 JVM 的世界里使用它，Dagger 就可以工作得很好。

### 结论

希望这篇文章有助于展示一些利用 Kotlin 及其扩展的方法，以简化我们的代码，同时保持其可读性。

我没有真正提到懒惰的委托，也没有提到 tuple / decomposition，这些也有它们的使用时间。 我也没有提到[新发布的 android-ktx](https://github.com/android/android-ktx)，有很多想法可以从源代码中收集！

值得一提的是，Kotlin 的生态系统正在增长，而且它确实有一些特点(apply，when ，内联扩展功能和高阶功能) ，这使得日常问题更容易解决。