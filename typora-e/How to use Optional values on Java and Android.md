# 如何在 Java 和 Android 上使用 Optional

> 原文 (Medium) ：	[How to use Optional values on Java and Android](https://fernandocejas.com/2016/02/20/how-to-use-optional-on-android-and-java/)
>
> 作者 ：[Fernando Cejas](https://fernandocejas.com/)

首先，这[不是一个新的话题](https://dzone.com/articles/guavas-new-optional-class)，关于这个问题已经讨论了很多。

这就是说，在本文中，我想解释什么是可 Optional\<T>，公开一些用例场景，比较不同的替代方案（用其他语言），最后，我想告诉你我们如何有效地使用 （[目前尚不存在](https://github.com/android10/arrow)）Android 上的 Optional\<T> API（尽管这可以应用于任何 Java 项目，特别是那些针对 Java 7 的项目）。

首先，让我引用这个(从[官方 Java 8文档](http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html)中检索到的) :"一个智者曾经说过，除非你处理了一个 null 指针异常，否则你不是一个真正的 Java 程序员，这是许多问题的根源，因为它经常被用来表示值的缺失。

虽然这个说法是正确的，而 Java 8文档指的是使用 Optional\<T> 作为 NullPointerException 保护程序，但在我看来，它不仅有助于最小化 NPE 的影响，而且可以创建更有意义和可读性更强的 API。

此外，众所周知，在使用空值时不小心会导致各种错误，例如，null 是含糊的，我们并不总是有一个明确的意义: 它是一个不存在的价值吗？ 例如，当 Map.get ( ) 法返回 null 时，可能意味着值不存在或当前值是空。

在这个小小的旅程中，我们将努力回答这些问题。 让我们把手弄脏！

## 什么是 Optional？

第一个 Java 8文档的定义是:

"Optional 对象用来表示空值为 null。 它提供了各种实用方法，以便于代码处理可用或不可用的值，而不是检查空值。

这是 Guava 官方文件中的另一个类似定义:

“一个不可变的对象，可能包含对另一个对象的非空引用，该类型的每个实例都包含非空引用，或者不包含任何内容（在这种情况下，我们说引用是”不存在“）;它永远不会表示 “包含 null”，一个非空的 Optional\<T> 引用可以用来替代可为空的 T 引用，它允许你表示 “一个必须存在的T” 和一个 “可能不存在的T “ 作为你程序中两种不同的类型，这可以帮助清晰”。

简而言之，Optional 类 API 提供了用于包含非空对象的容器对象。 让我们看一个简单的例子，这样你就能更好地理解我所说的:

```Java
Optional<Integer> possible = Optional.of(5);
possible.isPresent();   // returns true
possible.get();         // returns 5
```

正如你所看到的，我们在一个 Optional\<T> 中封装了一个 \<T> 类型的对象，所以我们可以稍后检查它的存在。 换句话说，Optional\<T> 强制你关注值，因为为了检索它，你必须调用 get ( ) 方法（作为一个好习惯，总是首先检查它的存在或者返回一个默认值）。 要明确，我们在这里使用 Guava 的 Optional\<T>。
如果你还不明白的话，不要太担心，我们会在之后探讨更多。

## Java 8, Scala, Groovy and Kotlin Optional/Option APIs

正如我上面提到的，在本文中，我们将关注 Guava Optional\<T>，尽管我们有必要对其他编程语言所提供的内容给出一个快速的视角。

让我们来看看 Groovy 和 Kotlin 带来了什么。 这两种语言对于无安全性提供了类似的方法: Elvis 操作符。 他们添加了一些语法糖和语法在他们两个看起来相似。 让我们来检查一下这段 Kotlin 代码: 当我们有一个可空的引用 r 时，我们可以说"如果 r 不是 null，使用它，否则使用一些非空值 x":

```kotlin
val l: Int = if (b != null) b.length else -1
```

除了完整的 if-expression 之外，还可以用 Elvis 操作符来表达吗？ 

```kotlin
val l = b?.length ?: -1
```

如果 ?: 左边的表达式不是 null，Elvis 操作符返回它，否则它返回右边的表达式。 注意，只有在左侧为空时才对右侧表达式进行求值。 为了记录在案，Kotlin 还有一个在编译时对空条件的检查。

你可以通过查看官方文档来更深入地探索，顺便说一下，我既不是一个 Groovy 的人也不是 Kotlin 人，所以我将把这个问题留给专家:)。

在 Java 8和 Scala 两个方面，我们为 Optional\<T>（Java）和 Option [T]（Scala）找到了一个单元方法，允许我们使用 flatMap ( )、 map ( ) 等。 这意味着我们可以在函数式编程风格中使用 OptionI\<T>  来编写数据流。 Kotlin 也提供了一个 OptionIF\<T> 目的相同。

让我们来看看这个来自 [Sean Parsons](http://seanparsons.github.io/scalawat/) 的 Scala 例子，以便更好地理解:

```kotlin
case class Player(name: String)
def lookupPlayer(id: Int): Option[Player] = {
  if (id == 1) Some(new Player("Sean"))
  else if(id == 2) Some(new Player("Greg"))
  else None
}
def lookupScore(player: Player): Option[Int] = {
  if (player.name == "Sean") Some(1000000) else None
}

println(lookupPlayer(1).map(lookupScore))  // Some(Some(1000000))
println(lookupPlayer(2).map(lookupScore))  // Some(None)
println(lookupPlayer(3).map(lookupScore))  // None

println(lookupPlayer(1).flatMap(lookupScore))  // Some(1000000)
println(lookupPlayer(2).flatMap(lookupScore))  // None
println(lookupPlayer(3).flatMap(lookupScore))  // None
```

最后但并非最不重要的是，我们有 Guava 的 Optional\<T>。 为了赞成它，让我们假设它的简化 API 完全符合 Java 7模型：只有一种方法可以使用它，因为实际上它是为缺乏一级函数的 Java 而开发的。

我猜到目前为止这么好，但没有 Android Java 7示例代码...好吧，你是对的，但你必须继续阅读，所以要耐心等待，还有更多。 此外，如果你想知道是否要在 Android 上使用它，你将不得不编译 Guava 和它的20k方法，答案是否定的，还有一种方法可以将 Optional\<T> 带到游戏中。

## 我们如何在安卓系统中使用  Optional\<T> ？

在这里提出的第一点是，我们只能使用 Java 7，所以没有内置的 Optional\<T>，我们不得不不幸地向[第三方库](https://github.com/android10/arrow)寻求帮助。

我们的第一个想到的是 Guava，这对于 Android 来说可能不是一个很好的选择，特别是(如上所述) ，因为它带来20k方法会提升你的 .apk。 (我相信你已经听说过[65k 方法限制问题](https://developers.soundcloud.com/blog/congratulations-you-have-a-lot-of-code-remedying-androids-method-limit-part-1);))。

第二个选择是使用 Arrow，它是一个轻量级的开源库，通过收集并包括我在 Android 开发中使用的有用内容，以及我自己编写的一些其他实用程序，例如代码装饰注释等。你可以在 Github 上检查项目、文档和功能。 有一件事需要大声说出来，那就是所有的信用都归功于这些神奇的 API 的创造者。

## 如何创建 Optional\<T>？

Optional\<T> API 非常简单:

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvhbetdpj30i90amjsd.jpg)

以下是 Optional\<T> 查询方法:

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvhkz3e6j30i40ggq4v.jpg)

现在是时候进行代码示例和用例了，所以不要离开房间。

## 案例 # 1

这是一个众所周知的历史  [Tony Hoare](https://en.wikipedia.org/wiki/Tony_Hoare) 在创建空参考时的一句话:

> "我称之为我的十亿美元的错误。 这是1965年空参考的发明。 我忍不住诱惑要提供一个空参考，仅仅是因为它实现起来很容易。"

```Java
public class Car {
  private final String brand;
  private final String model;
  private final String registrationNumber;

  public Car(String brand, String model, String registrationNumber) {
    this.brand = brand;
    this.model = model;
    this.registrationNumber = registrationNumber;
  }

  public String information() {
    final StringBuilder builder = new StringBuilder();
    builder.append("Model: ").append(model);
    builder.append("Brand: ").append(brand);
    if (registrationNumber != null) {
      builder.append("Registration Number: ").append(registrationNumber);
    }
    return builder.toString();
  }
}
```

下面的代码的主要问题是依赖于空引用来指示没有注册号（一种不好的做法），所以我们可以通过使用 Optional \<T> 来解决这个问题，并根据值是否存在来打印：

```Java
public class Car {
  ...
  private final Optional<String> registrationNumber;

  public Car(String brand, String model, String registrationNumber) {
    ...
    this.registrationNumber = Optional.fromNullable(registrationNumber);
  }

  public String information() {
  ...
    if (registrationNumber.isPresent()) {
      builder.append("Registration Number: ").append(registrationNumber.get());
    }
    return builder.toString();
  }
}
```

最明显的用例是避免无意义的空白。 [检查 Github 上的全部类实现](https://github.com/android10/java-code-examples/blob/master/src/main/java/com/fernandocejas/java/samples/optional/UseCaseScenario02.java)。 让我们进入下一个场景。

## 案例 # 2

假设我们需要解析来自 API 响应的 JSON 文件（在 Mobile Development 中非常常见）。 在这种情况下，我们可以在实体中使用 Optional\<T> 来强制客户在使用或执行任何操作之前关心值的存在。

在下面的示例代码中检查 "nickname" 字段和 getter:

```Java
public class User {
  @SerializedName("id")
  private int userId;

  @SerializedName("full_name")
  private String fullname;

  @SerializedName("nickname")
  private String nickname;

  public int id() {
    return userId;
  }

  public String fullname() {
    return fullname;
  }

  public Optional<String> nickname() {
    return Optional.fromNullable(nickname);
  }
}
```

完成的示例类在 [Github](https://github.com/android10/java-code-examples/blob/master/src/main/java/com/fernandocejas/java/samples/optional/UseCaseScenario02.java) 上。

## 案例 # 3

这是我们在 Android 应用程序中偶然发现的另一个用例。

当我们需要构建 Feed 或任何项目列表并在 UI 级别（演示模型）中显示它们时，我们有来自不同数据源的项目，其中一些可能是 Optional\<T>，例如，Facebook邀请 ，推广赛道等。

看看这个小例子，它试图以一种非常简单的方式(为了学习目的)使用 RxJava:

```Java
public class Sample {

  public static final Func1<Optional<List<String>>, Observable<List<String>>> TO_AD_ITEM =
      ads -> ads.isPresent()
          ? Observable.just(ads.get())
          : Observable.just(Collections.<String>emptyList());

  public static final Func1<List<String>, Boolean> EMPTY_ELEMENTS = ads -> !ads.isEmpty();

  public Observable<List<String>> feed() {
    return ads()
        .flatMap(TO_AD_ITEM)
        .filter(EMPTY_ELEMENTS)
        .concatWith(tracks())
        .observeOn(Schedulers.immediate());
  }

  private Observable<Optional<List<String>>> ads() {
    return Observable.just(Optional.fromNullable(Collections.singletonList("This is and Ad")));
  }

  private Observable<List<String>> tracks() {
    return Observable.just(Arrays.asList("IronMan Song", "Wolverine Song", "Batman Sound"));
  }
}
```

这里最重要的部分是，当我们结合 Observables \<T>（tracks ( ) 和 ads ( ) 方法）时，我们使用 flatMap ( ) 和 filter ( ) 运算符来确定我们是否要发出广告，例如，显示他们在 UI 级别（我在这里使用 Java 8 lambda 来使代码更具可读性）：

```Java
public static final Func1<Optional<List<String>>, Observable<List<String>>> TO_AD_ITEM =
      ads -> ads.isPresent()
          ? Observable.just(ads.get())
          : Observable.just(Collections.<String>emptyList());

public static final Func1<List<String>, Boolean> EMPTY_ELEMENTS = ads -> !ads.isEmpty();
```

查看 [Github](https://github.com/android10/java-code-examples/blob/master/src/main/java/com/fernandocejas/java/samples/optional/UseCaseScenario03.java) 上的完整实现。

## 结论

总结一下，在软件开发中没有银弹，作为程序员，我们倾向于过度思考和过度使用事物，所以不要随意使用Optional\<T> 污染代码，请在适当的地方仔细使用它们。

让我引用 Joshua Bloch 在《[How to Design a Good API and Why it Matters](http://www.infoq.com/articles/API-Design-Joshua-Bloch)》中的话:

"API 应该易于使用，并且很难被滥用: 做简单的事情应该很容易; 做复杂事情是可能的; 做错事情是不可能的，或者至少是困难的。"

我完全同意这一点，从 API 设计的角度来看，Optional\<T> 是一个很好的设计 API 的好例子：它将帮助你解决和避免 NullPointerException 问题（尽管不能完全消除它们），编写简明易懂的代码 并且还会提供更有意义的代码库。

## 样本代码

你可以在我为此目的创建的 Github 仓库中找到所有的示例代码:

[android10/java-code-examples | github.com](https://github.com/android10/java-code-examples)

还可以访问 Arrow 项目仓库，以使用 Android 中的 Optional\<T>:

[android10/arrow | github.com](https://github.com/android10/arrow)

## 参考文献:

- <http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html>
- <https://github.com/google/guava/wiki/UsingAndAvoidingNullExplained>
- <https://dzone.com/articles/guavas-new-optional-class>
- <https://kerflyn.wordpress.com/2011/12/05/from-optional-to-monad-with-guava/>
- <http://techblog.bozho.net/the-optional-type/>
- <http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/base/Optional.html>
- <http://seanparsons.github.io/scalawat/Using+flatMap+With+Option.html>
- <http://www.nurkiewicz.com/2013/05/null-safety-in-kotlin.html>
- <https://kotlinlang.org/docs/reference/null-safety.html>



