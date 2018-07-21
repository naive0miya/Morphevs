# 6种语法糖，可以让你的 Kotlin 编程更加快乐

> 原文 (Medium)：[6 magic sugars that can make your Kotlin codebase happier — Part 1](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-1-ceee3c2bc9d3)
>
> 原文 (Medium)：[6 magic sugars that can make your Kotlin codebase happier — Part 2](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-2-843bf096fa45)
>
> 原文 (Medium)：[6 magic sugars that can make your Kotlin codebase happier — Part 3](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-3-6319a451cd5d)
>
> 作者：[Piotr Ślesarew](https://medium.com/@piotr.slesarew)

[TOC]

# 第1部分



我们经常失去一些东西。其中一些比其他的更重要，但追赶这些人永远不会太迟。Kotlin 语言为你的悲惨编程生活带来了大量新概念和新功能，在你的日常工作中很难使用所有的概念和特征。在使用 Kotlin 生产近两年后，语言本身就能给我带来快乐和满足。 这怎么可能呢？ 因为它含有许多语法糖。 
在本文中，我想与你分享我最喜欢的 Kotlin 语法糖，当我需要为 Android 应用程序编写健壮而简洁的组件时发现的。 为了使这篇文章读起来更加友好，我把它分成三部分。 在第一部分中，你将能够看到 sealed  类和 when ( ) 控制流函数，一些很酷的特性。 享受吧！ 

## Seal your class with a kiss of “pattern matching”

最近我有机会使用 Swift 来工作。我不仅需要检查代码，还必须将一些组件翻译成 Kotlin。我阅读的代码越多，我就越感到惊讶，但是对我来说最有吸引力的特征是枚举🚀。不幸的是，Kotlin 枚举不是很通用，所以我不得不为它们寻找合适的替代品：密封类。

在编程世界中，密封类并不是什么新东西。其实，密封类是一个非常有名的语言概念。Kotlin 引入了可以添加到类声明中并用于表示受限类层次结构的 sealed 关键字，值可以来自有限集合中的一种类型，但不能具有任何其他类型。 长话短说，你可以使用密封类来替代枚举等等。

```kotlin
sealed class Response

data class Success(val body: String): Response()

data class Error(val code: Int, val message: String): Response()

object Timeout: Response()
```
乍一看，这个代码除了声明一个可怜的继承关系之外什么都不做，但是在一步一步的解构之后，它显示出惊人的潜力。 那么，sealed 关键字实际上是如何添加到 Response 类中的？ 打开它的最好方法是使用 IntelliJ IDEA Kotlin 字节码工具。 
![@Step 1. Reveal Kotlin Bytecode](https://ws2.sinaimg.cn/large/006tNc79gy1fromzrd0lpj30m805vwf8.jpg)
![@Step 2. Decompile Kotlin Bytecode to Java code](https://ws1.sinaimg.cn/large/006tNc79gy1fromzgeq76j30m803d74q.jpg)在这个超级简单的转换之后，你可以开始阅读 Kotlin 代码的 Java 表示。 

```kotlin
public abstract class Response {
   private Response() {
   }

   // $FF: synthetic method
   public Response(DefaultConstructorMarker $constructor_marker) {
      this();
   }
}
```
正如你可能已经猜到的那样，密封类是为了继承而特别制作的，所以它们是开箱即用的抽象类。但是它们和枚举有什么相似之处？ 在这个时候，Kotlin 编译器允许你使用响应类的子类作为 when ( ) 函数的情况，从而为你提供了一个巨大的优势。 此外，Kotlin 提供了很大的灵活性，从密封类继承的结构可以被声明为数据或甚至作为一个对象。 

```kotlin
fun sugar(response: Response) = when (response) {
    is Success -> ...
    is Error -> ...
    Timeout -> ...
}
```
它不仅提供完全详尽的表达式，而且还提供自动铸造，因此你可以使用 Response 实例而不需要任何额外的转换。 

```kotlin
fun sugar(response: Response) = when (response) {
    is Success -> println(response.body)
    is Error -> println("${response.code} ${response.message}")
    Timeout -> println(response.javaClass.simpleName)
}
```
你能想象如果没有密封的特征，或者没有 Kotlin，它看起来会有多么丑陋和复杂吗？ 如果你已经忘记了一些 Java 语言，请再次使用 IntelliJ IDEA Kotlin Bytecode，但是坐稳了，它可能会让你晕倒。 

```kotlin
public final void sugar(@NotNull Response response) {
   Intrinsics.checkParameterIsNotNull(response, "response");
  
   String var3;
   if (response instanceof Success) {
      var3 = ((Success)response).getBody();
      System.out.println(var3);
   } else if (response instanceof Error) {
      var3 = "" + ((Error)response).getCode() + ' ' + ((Error)response).getMessage();
      System.out.println(var3);
   } else {
      if (!Intrinsics.areEqual(response, Timeout.INSTANCE)) {
         throw new NoWhenBranchMatchedException();
      }

      var3 = response.getClass().getSimpleName();
      System.out.println(var3);
   }
}
```
总结起来，我很高兴在这种情况下使用 sealed 关键字，因为它让我以一种类似于 Swift 版❤的方式来塑造我的 Kotlin 代码。 

## Use the when ( ) function to permute like a boss
由于你已经有机会看到 when ( ) 与密封类使用时的力量，所以我决定与你分享它的强大能力。 想象一下，你必须实现一个接受两个 enums 并产生一个不可变状态的函数。 

```kotlin
enum class Employee {
    DEV_LEAD,
    SENIOR_ENGINEER,
    REGULAR_ENGINEER,
    JUNIOR_ENGINEER
}

enum class Contract {
    PROBATION,
    PERMANENT,
    CONTRACTOR,
}
```
enum class Employee 描述了在公司 XYZ 和 enum class Contract 中可以找到的所有角色，包含了所有类型的雇佣合同。基于这两个枚举，你应该返回一个正确的 SafariBookAccess。 此外，你的函数必须生成给定 enums 的所有排列的状态。 作为第一步，让我们对产生状态函数的签名进行原型化。 

```kotlin
fun access(employee: Employee,
           contract: Contract): SafariBookAccess
```
现在是时候定义 SafariBooksAccess 结构了，因为你已经知道 sealed 关键字了，所以现在是使用它的最佳时机。 不需要密封 SafariBookAccess，但它是在 SafariBookAccess 的不同情况下封装不同状态定义的好方法。

```kotlin
sealed class SafariBookAccess

data class Granted(val expirationDate: DateTime) : SafariBookAccess()

data class NotGranted(val error: AssertionError) : SafariBookAccess()

data class Blocked(val message: String) : SafariBookAccess()
```
那么 access ( ) 函数的主要思想是什么？排列！让我们排列👊。
```kotlin
fun access(employee: Employee,
           contract: Contract): SafariBookAccess {
    return when (employee) {
        SENIOR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Granted(DateTime())
            CONTRACTOR -> Granted(DateTime())
        }
        REGULAR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Granted(DateTime())
            CONTRACTOR -> Blocked("Access blocked for $contract.")
        }
        JUNIOR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Blocked("Access blocked for $contract.")
            CONTRACTOR -> Blocked("Access blocked for $contract.")
        }
        else -> throw AssertionError()
    }
}
```
这个代码是完美无缺的，但是你能让它更像 Kotlin 风格吗？ 在对同事的 pr / mr 进行日复一日的代码检查时，你会提出什么建议？ 你的评论可以包含这样的内容: 

> - when ( ) 函数太多了。使用 Pair 来避免嵌套。
> - 更改枚举参数的顺序。将 Pair 定义为 Pair \<Contract，Employee> ( ) 对象，以使其更具可读性。
> -  合并重复的返回用例。
> - 更改为单个表达式函数。
```kotlin
fun access(contract: Contract,
           employee: Employee) = when (Pair(contract, employee)) {
    Pair(PROBATION, SENIOR_ENGINEER),
    Pair(PROBATION, REGULAR_ENGINEER),
    Pair(PROBATION, JUNIOR_ENGINEER) -> NotGranted(AssertionError("Access not allowed on probation contract."))
    Pair(PERMANENT, SENIOR_ENGINEER),
    Pair(PERMANENT, REGULAR_ENGINEER),
    Pair(PERMANENT, JUNIOR_ENGINEER),
    Pair(CONTRACTOR, SENIOR_ENGINEER) -> Granted(DateTime(1))
    Pair(CONTRACTOR, REGULAR_ENGINEER),
    Pair(CONTRACTOR, JUNIOR_ENGINEER) -> Blocked("Access for junior contractors is blocked.")
    else -> throw AssertionError("Unsupported case of $employee and $contract")
}
```
现在看起来更干净，更简洁，但 Kotlin 有一些额外的语法糖，可以让你完全省略 Pair 定义。

```kotlin
fun access(contract: Contract,
           employee: Employee) = when (contract to employee) {
    PROBATION to SENIOR_ENGINEER,
    PROBATION to REGULAR_ENGINEER -> NotGranted(AssertionError("Access not allowed on probation contract."))
    PERMANENT to SENIOR_ENGINEER,
    PERMANENT to REGULAR_ENGINEER,
    PERMANENT to JUNIOR_ENGINEER,
    CONTRACTOR to SENIOR_ENGINEER -> Granted(DateTime(1))
    CONTRACTOR to REGULAR_ENGINEER,
    PROBATION to JUNIOR_ENGINEER,
    CONTRACTOR to JUNIOR_ENGINEER -> Blocked("Access for junior contractors is blocked.")
    else -> throw AssertionError("Unsupported case of $employee and $contract")
}
```
我希望你已经发现了这个有用的东西，因为这个结构让我的生活变得更加轻松，我的 Kotlin 代码库也更快乐。 不幸的是，不可能在三个元素中使用这种语法。 

# 第2部分

我们正在继续我们的旅程，通过一些我最喜欢的 kotlin 构造。 在第一部分，你学习如何使用密封类和置换两个甚至三个枚举，以一种优雅和简洁的方式。 

在这一部分中，我想向你展示一些我经常在具有表示层的组件中使用的东西。还会有一些特别的东西可以完美地演示如何使用内联指定函数。我希望你会像我一样欣赏我使用的语法糖。

我们从着名的 with ( ) 函数开始吧。

## Use the with ( ) function to scope invocations
假设你从未使用过 with ( ) 函数，那么让我们看看你可以在文档中找到什么：

> inline fun \<T, R> with (receiver: [T](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html#T) , block: T.( ) -> [R](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html#R)) : R ([source](http://github.com/JetBrains/kotlin/blob/1.2.0/libraries/stdlib/src/kotlin/util/Standard.kt#L55))
>
> 调用给定的函数 block 与给定的 receiver ，把 receiver 作为 block 函数的接收器并返回结果。

乍一看，它看起来有点复杂，但前提是你不知道一个块或一个接收器是什么。像往常一样，一个例子会为你澄清所有这些问题: 

```kotlin
val receiver: String = "Fructos"

val block: String.() -> Unit = {
    println(toUpperCase())
    println(toLowerCase())
    println(capitalize())
}
        
val result: Unit = with<String, Unit>(receiver, block)
```
第一句话是不用动脑筋的。 它显示字符串类型的接收器。 接收器类型非常重要，因为在第三行中，可以看到具有指定接收对象的匿名函数的定义。 这个符号可能会提醒你一个扩展函数机制，这是一件好事。 在指定的块内，你可以在接收器对象上调用方法，而不需要任何限定符。 最后一个有趣的事情是返回单元类型的块函数。 

现在，块和接收器不应该有任何隐藏的逻辑。 运用你的新知识，将块和接收器应用到 with ( ) 函数不仅是一个简单的任务，而且也是一个有趣的任务。 

```kotlin
val sugar = "Fructos"

with(sugar) {
    println(toUpperCase())
    println(toLowerCase())
    println(capitalize())
}
```
块 lambda 是 with ( ) 函数的最后一个参数，因此，你可以将它放置在圆括号之外。 就是这样！ 

那么，你如何将一个 with ( ) 函数应用到你的代码库中，使它更加快乐呢？ 在过去的几个月里，我经常用它来省略 view.show ( ) 和 view.hide ( ) 之类的限定符: 

```kotlin
interface View {

    fun show()
    fun hide()
    fun reset()
    fun clear()
}

class Presenter(private val view: View) {

    fun present(isFructos: Boolean) = with(view) {
        if (isFructos) {
            show()
            hide()
        } else {
            hide()
            clear()
        }
    }
}
```
看起来不错，不是吗？ 此外，在未来使用这样的代码确实很容易。 没有必要去考虑视图，因为我们的函数已经被它包围了。 

现在是时候进一步，使用 reified 关键字来编写自己的 withCorrectType ( ) 函数！

## Try inline reified to invoke withCorrectType ( )
我真的很喜欢代码评论。 在我看来，这是在你的团队中快速学习和交流知识的最好方法之一。 下一个语法糖是从我的队友的评论到我最近重新考虑的代码。 你是否曾经有过这样的感觉，那就是代码酶有问题，但是你不知道如何使它变得更好？ 如果是这样的话，不要因为它是完全正常的而感到难过。 

让我们分析以下几点。 

```kotlin
abstract class Item

class MediaItem : Item() {
    val media = ...
}

class IconItem : Item() {
    val icon = ...
}

interface Renderer {
    fun render(view: View, item: Item)
}

class MediaItemRenderer : Renderer {

    override fun render(view: View, item: Item) {
        if (item !is MediaItem) {
            throw AssertionError("Item is not an instance of MediaItem")
        }

        view.showMedia(item.media)
        view.reset()
    }
}

class IconItemRenderer : Renderer {

    override fun render(view: View, item: Item) {
        if (item !is IconItem) {
            throw AssertionError("Item is not an instance of IconItem")
        }

        view.showIcon(item.icon)
        view.reset()
    }
}
```
你能指出这个代码中的任何弱行吗？ 很明显，MediaItemRenderer 和 IconItemRenderer 在 render ( ) 方法中有相似的逻辑。 此外，你可以使用前面的语法糖的知识和省略 view  限定符。 放松！ 

```kotlin
class MediaItemRenderer : Renderer {

    override fun render(view: View, item: Item) = with(view) {
        if (item !is MediaItem) {
            throw AssertionError("Item is not an instance of MediaItem")
        }

        showMedia(item.media)
        reset()
    }
}

class IconItemRenderer : Renderer {

    override fun render(view: View, item: Item) = with(view) {
        if (item !is IconItem) {
            throw AssertionError("Item is not an instance of IconItem")
        }

        showIcon(item.icon)
        reset()
    }
}
```
现在是时候想出一些聪明的办法了。 如何创建一个类似于 when ( ) 和尝试移动类型检查代码的函数？ 

```kotlin
fun <T> withCorrectType(toBeChecked: Item, block: (T) -> Unit) {
    if (toBeChecked !is T) {
        throw IllegalArgumentException("Invalid type")
    }
    block.invoke(toBeChecked)
}
```
很棒！现在你很好去重构你的 render ( ) 函数，不是吗？嗯...有一个小问题，因为你的代码不能编译。

![|center](https://ws2.sinaimg.cn/large/006tNc79gy1fron03y1vxj30cr02zt8v.jpg)

它甚至意味着不可能检查一个被擦除的类型的实例 : T？ 这个错误被严格限制在对泛型类型的擦除和泛型的工作方式。 

>在类型擦除过程中，如果类型参数是有界的，Java 编译器会删除所有类型参数，并用第一个绑定来替换每个参数，如果类型参数是无界的，则使用 Object。   - [docs.oracle.com](https://docs.oracle.com/javase/tutorial/java/generics/genTypes.html)

那么，是否有可能防止你的 T 被擦除？ 对于 Kotlin，一切皆有可能！ 我是说，几乎所有的事。 通过使用 inline  和 reified  关键字的组合，你可以很容易地解决你的问题。 

重构代码：
```kotlin
class MediaItemRenderer {

    fun render(view: View, item: Item) = with(view) {
        withCorrectType<MediaItem>(item) {
            show { it.media() }
            reset()
        }
    }
}

class IconItemRenderer {

    fun render(view: View, item: Item) = with(view) {
        withCorrectType<IconItem>(item) {
            clear()
            show { it.icon() }
        }
    }
}

inline fun <reified T> withCorrectType(toBeChecked: Item, block: (T) -> Unit) {
    if (toBeChecked !is T) {
        throw IllegalArgumentException("Invalid type, should be ${T::class.java.simpleName}")
    }
    block.invoke(toBeChecked)
}
```
你可以更深入地使用 IntelliJ IDEA Kotlin Bytecode 工具来找出 Kotlin 编译器如何处理你的  reified 类型以及 inline 关键字如何提供帮助。

```kotlin
public final class MediaItemRenderer {
   public final void render(@NotNull View view, @NotNull Item item) {
      if (!(item instanceof MediaItem)) {
         throw (Throwable)(new IllegalArgumentException("Invalid type, should be " + MediaItem.class.getSimpleName()));
      } else {
         MediaItem it = (MediaItem)item;
         view.show((Function0)(new MediaItemRenderer$render$1$1$1(it)));
         view.reset();
      }
   }
}

public final class IconItemRenderer {
   public final void render(@NotNull View view, @NotNull Item item) {
      if (!(item instanceof IconItem)) {
         throw (Throwable)(new IllegalArgumentException("Invalid type, should be " + IconItem.class.getSimpleName()));
      } else {
         IconItem it = (IconItem)item;
         view.clear();
         view.show((Function0)(new IconItemRenderer$render$1$1$1(it)));
      }
   }
}
```
这是纯粹的魔法，对吧？ Kotlin 编译器之所以离开你的类型，是因为你把它标记为 reified 了。 如果不使用 inline  标记函数，这是不可能的，因为 withCorrentType ( ) 的代码必须直接注入调用位置。 

总结一下，具有 reified 类型的 inline 函数不仅使你的代码更具可读性，而且在性能方面也没有负面影响

# 第3部分

不幸的是，所有的旅程都必须结束。这是我最后一个 - 但并非最不重要的 - 关于Kotlin 语法糖的三部分系列的一部分。在之前的文章中，你有机会熟悉：

`sealed` - 在不同的情况下封装不同的状态定义 

`when()` - 以干净优雅的方式进行排列。

`with()` - 省略修饰符。

`inline` - inline 函数和 reified 类型来编写自己的 withCorrectType ( ) 函数。

这次我有一些非常特别的东西。 首先，我想向你介绍关联代码，以及如何使用 Kotlin 语言的委托机制来简化和缩短关联代码。 然后，在本文的第二部分，你将更加熟悉编写自己的 DSL（特定于域的语言）结构。

倒一杯你最喜欢的饮料，并享受阅读。

## KProperty: make your aggregationgreat again

据说，错误的抽象比没有抽象更糟糕。 实际上，这里指出的是，继承是一个非常棘手的机制，它应该被仔细地使用。 有些书说你应该使用组合而不是继承，这是好的设计的关键。 

Kotlin 有一个内置的委托模式，使用它非常容易使用组合。

```kotlin
interface Navigable {

    val onNavigationClick: (() -> Unit)?
}

interface Searchable {

    var searchText: String
}

class Component(navigable: Navigable, 
                searchable: Searchable) 
    : Navigable by navigable, 
      Searchable by searchable
```

正如你所看到的，这里没有魔法。 Component 类实现 Navigable 和 Searchable 接口，然后使用 by 语言关键字 , 应用 navigable 和 searchable 的构造函数参数的行为。 这种语言结构是非常有用的，因为它可以减少大量的样板代码，但它不是所有关联情景的银弹。

想象一下，你需要把你的接口标记为 internal 接口，因为你想让它们成为你内部架构的一部分：

```kotlin
internal interface Navigable {

    val onNavigationClick: (() -> Unit)?
}

internal interface Searchable {

    var searchText: String
}
```

它如何影响 Component 类？可悲的是，你破坏了代码，现在你得到了一个编译错误💔。这是因为 Kotlin 编译器不允许暴露模块的内部组件。

![](https://ws2.sinaimg.cn/large/006tNc79gy1fron0cwae5j30m8041my9.jpg)

因此，如果不需要公开 Navigable 和 Searchable 依赖关系，你可以：

- 删除构造函数。
- 使用组合而不是聚合。
- 定义包含来自 Navigable 和 Searchable 接口的方法签名的 ComponentInterface。

```kotlin
interface ComponentInterface {

    val onNavigationClick: (() -> Unit)?

    var searchText: String
}

class Component : ComponentInterface {

    private val navigable: Navigable = NavigableImpl()
    private val searchable: Searchable = SearchableImpl()

    override val onNavigationClick: (() -> Unit)?
        get() = navigable.onNavigationClick

    override var searchText: String = ""
        get() = searchable.searchText
        set(value) {
            field = value
            searchable.searchText = value
        }
}
```

哇！这种情况迅速升级。 从零样板聚合，你去到了令人讨厌的组合。 但是不要哭泣，因为 Kotlin 总是有一些糖让你的代码库更加快乐。  Kotlin 不仅支持使用 by 关键字对指定对象进行方法委托，还具有委托属性的机制。你可能通过将 lazy ( ) 委托应用到在访问时应该初始化的属性。 

![](https://ws3.sinaimg.cn/large/006tNc79gy1fron0hygn4j30g401ejrb.jpg)

懒惰很好，但它与你的情况有什么关系？这意味着你可以使用与定义 lazy ( ) 函数相同的模式来重构你的组合代码。 

警告！ 这段代码可以彻底改变你的生活。

```kotlin
class ReferencedProperty<T>(private val get: () -> T,
                            private val set: (T) -> Unit = {}) {

    operator fun getValue(thisRef: Any?,
                          property: KProperty<*>): T = get()

    operator fun setValue(thisRef: Any?,
                          property: KProperty<*>,
                          value: T) = set(value)
}

fun <T> ref(property: KMutableProperty0<T>) = ReferencedProperty(property::get, property::set)

fun <T> ref(property: KProperty0<T>) = ReferencedProperty(property::get)
```

ReferencedProperty 一个在构造函数中以两个函数为参数并定义两个运算符的类 。

- get 函数不需要任何东西，并返回一个泛型类型 T 。它是一个访问函数。
- set 函数采用通用类型 T 并返回 Unit。这是一个增变函数。另外，set 有一个默认值被设置为空函数。
- fun getValue ( ) 运算符调用 get ( ) 函数。
- fun setValue ( ) 运算符调用带有 value 参数的 set ( ) 函数

最重要的要知道的是，属性委托机制使用操作符。 在 ReferencedProperty 类声明之后，可以找到两个返回 ReferencedProperty 类型实例的两个泛型 ref ( ) 方法 。 为什么是两个？ 第一个将用于标记为 var 的属性，第二个标记为 val 的属性。 

让我们使用 ref ( ) 函数来清理你的组合。

```kotlin
class Component : ComponentInterface {

    private val navigable: Navigable = NavigableImpl()
    private val searchable: Searchable = SearchableImpl()

    override val onNavigationClick by ref(navigable::onNavigationClick)
    override var searchText by ref(searchable::searchText)
}
```

现在看起来干净多了，不是吗？  ref ( ) 函数允许你将属性定义委托给另一个组件。 通过使用 `::` 你可以得到 val KProperty0 和 var KMutableProperty0 。这些是属性的表示形式，你可以很容易地从它们中访问 getter 和 setter 函数。 这可能听起来很复杂，但是我真的鼓励你使用这种语法。 希望，你会发现它是有用的，因为它可以节省大量的代码。 它救了我的命！ 有一个需要你注意的警告。 这种语法在幕后使用反射，所以确保你已经准备好采取这种权衡。 

现在是时候潜入 DSL 的水域。做好准备。

## Write DSLs to cover up ugliness

Kotlin 最酷的特点之一就是它让你有能力写出清晰的 dsl。 领域特定语言是一种专门针对特定应用程序域的计算机语言。 最好的例子就是一个著名的 Anko 库，它被用来编程构建 Android 应用程序的用户界面。 此外，你可以使用它来对照 SQLite 数据库和计划后台任务(幕后使用 Coroutines)编写语句。 。

它不仅简化了代码，而且使它更具可读性。 

```kotlin
verticalLayout {
    val name = editText()
    button("Say Hello") {
        onClick { 
            toast("Hello, ${name.text}!") 
        }
    }
}
```

你可能认为建立这样一个令人愉快的 API 需要先进的 Kotlin 知识，但是你大错特错！ 最近，我决定在验收测试的顶部添加一个图层。 我的主要目标是把 WHAT 和 HOW 分开。 这是什么意思？ 

- WHAT 应该是只包含要测试的信息的测试本身 
- HOW  描述用于测试的框架的内部部分 

此外，我希望我的测试用例在 Kotlin 的 Android 和在 Swift 中写的 iOS 看起来是一样的。 为这些问题编写 dsl 是一个完美的解决方案。 

让我们来看看我想实现的语法。 

```kotlin
class SearchTest : UITest() {

    @Before
    fun setup() {
        navigation {
            openScreenOne()
        }
    }

    @Test
    fun `shows searched items`() {
        val searchQuery = "foo"

        toolbar {
        } openSearch {
            type(searchQuery)
        }

        screenOne {
            hasItemWithText(searchQuery)
        }
    }
}
```

这是一个非常漂亮，看起来很干净的代码，不是吗？ setup ( ) 函数包含主要测试用例的先决条件。 它使用导航控制器开发一个应用程序来打开 ScreenOne。 你能告诉我这个 DSL 是如何在幕后实现的吗？ 当然，我并不是在问将命令发送到目标设备的框架。 如果你不知道，让我解释给你听。 

```kotlin
fun navigation(block: NavController.() -> Unit): Unit = NavController.block()

object NavController {

    fun openScreenOne() {
        // Testing framework code
    }
}
```

正如你所看到的，实现是微不足道的。 navigation ( ) 函数将 block 作为参数，它只是一个带有接收器的匿名函数。 函数的主体调用 block ( ) 直接在 NavController 对象上。 NavController 是一个定义行为函数的对象。 在这里，你可以使用你最喜欢的测试框架来实现 HOW。 

在这个测试案例中还有一个值得解释的结构。 

```kotlin
al searchQuery = "foo"

toolbar {
} openSearch {
    type(searchQuery)
}
```

上面提到的 DSL 有什么有趣的地方？ 它和 navigation { openScreenOne( ) } 几乎一样，除了在关闭括号之后，还有另外一个链式结构。 怎样才能把这样的结构链接起来呢？ 是函数调用还是完全不同的东西？ 更清楚地说，我会把这个 DSL 改成更明显的东西。 

```kotlin
val searchQuery = "foo"

toolbar {
}. openSearch() {
    type(searchQuery)
}
```

主要的区别是，在调用 openSearch 函数之前(代码显示使用括号的函数) ，你可以看到点符号。 这以一种非常明确的方式显示了 openSearch ( ) 是一个对从 toolbar { } 返回的对象调用的函数。 现在实现这样的 DSL 应该是一个简单的任务，而且实现可以是这样的: 

```kotlin
fun toolbar(block: Toolbar.() -> Unit): Toolbar = Toolbar.apply(block)

object Toolbar {

    fun openSearch(block: SearchToolbar.() -> Unit): SearchToolbar {
        // Testing framework code

        return SearchToolbar.apply(block)
    }
}

object SearchToolbar {

    fun type(text: String) {
        // Testing framework code
    }
}
```

这个 toolbar ( ) 的功能看起来和之前实现的导航相似。 它将带有接收器的匿名函数作为参数，然后返回 Toolbar   对象的实例。 它允许你调用 openSearch ( ) 函数，该函数包含了 HOW，并返回 SearchToolbar 对象的一个实例。 同样，SearchToolbar 是一个定义包含 HOW 的行为函数 type ( ) 的对象。 太棒了！ 

 最后一个难题是点符号以及如何实现省略它的 DSL 语法。 老实说，实现更多的是基于黑客，而不是一个优雅的专用解决方案。 

```kotlin
infix fun openSearch(block: SearchToolbar.() -> Unit): SearchToolbar {
    // Testing framework code

    return SearchToolbar.apply(block)
}
```

标记 openSearch ( ) 函数为 infix 允许你使用 infix 符号来调用它。 为了满足编写算术和逻辑语句的需要，在 Kotlin 语言中添加了 infix 关键字。 它不应该被滥用来实现其他结构。 此外，补丁只能用于一个参数的函数。 这种"限制"会导致 DSL 中的不一致。 

> 所有旅程结束。

