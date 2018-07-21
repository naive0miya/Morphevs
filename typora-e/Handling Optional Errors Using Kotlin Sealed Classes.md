# 	使用 Kotlin 密封类处理 Optional 错误

> 原文 (articles.caster.io) ：[HANDLING OPTIONAL ERRORS USING KOTLIN SEALED CLASSES](https://articles.caster.io/android/handling-optional-errors-using-kotlin-sealed-classes/)
>
> 作者 ：[CRAIG RUSSELL](https://articles.caster.io/author/craigrussell/)

在调用返回数据的函数时，通常需要处理可能发生的错误。 这篇文章详细介绍了如何利用 Kotlin 的密封类返回一个可以表示你所期望的数据或错误的对象。

让我们想象一下，我们有一个函数，它接受某些数据作为参数并返回解析结果。 它依赖一个 regex 来帮助解析; 由于我们不控制输入，那么根据提供给函数的字符串，regex 可能无法匹配。

```kotlin
private fun parse(url: String): ParsedData {
    val result = URL_PARSE_REGEX.find(url)

    if (result == null) {
        // What do we do here? Throw an exception? Return null?
    }

    val mimeType = result.groupValues[2]
    val data = result.groupValues[4]

    return ParsedData(data, mimeType)
}

data class ParsedData(
    val data: String,
    val mimeType: String
)
```

如果 regex 无法匹配任何东西，结果将是 null。

```Kotlin
if (result == null) {
    // What do we do here? Throw an Exception? Return null?
}
```

注意这个代码。 如果在解析期间，我们决定不能继续，我们必须找到一种方法来突破函数。

## 潜在解决方案(不要这样做)

#### 抛出一个异常

我们可能会决定一个实现这个目标的方法就是抛出一个异常。

```kotlin
if (result == null) {        
    throw IllegalArgumentException()
}
```

然而，强制将异常处理强加到这个函数的调用者是相当沉重的。 即使是运行时异常也是一个需要处理的负担，而且在 regex 上找不到匹配，听起来也不是那么特别，所以这可能不是正确的配合。

#### 返回 NULL

当 regex 匹配失败时，我们可能会决定返回 null。

```kotlin
if (result == null) {        
    return null
}
```

现在此函数的调用方可以执行 null 检查，并使用 null 作为指示错误流发生。 但是我们还必须将函数定义为返回 ParsedData？现在，我们无法提供有关哪里出错的详细信息。所有调用者都知道有些事情没有奏效。

#### 引入一个包装器对象

我们可以引入一个新的类来完全负责包装所需的数据或错误。 因为在这种情况下，数据和错误消息是相互排斥的，所以我们必须使两者都可以为空。

```kotlin
class ResultWrapper (
    val data: ParsedData? = null,
    val errorMessage: String? = null
)
```

然后我们审问包装器，以确定是否发生错误; 在这种情况下，检查 result.errorMessage 是否为 null。 即使在确定一个错误没有发生之后，我们仍然有烦恼，我们的真实数据不是必要的，所以我们不得不用 ! ！解开它。

```kotlin
val result = parse(url)

if (result.errorMessage != null) {
    // error occurred - log error message etc... 
} else {
    // happy path - use the data. But result.data is nullable 🙄
    val mimeType = result.data!!.mimeType
}
```

使用可能包含数据或错误的包装器对象比返回 null 或抛出异常要好，但是它仍然有点笨重。

## 更好的解决方案(这样做)

#### 密封类

> 密封类用于表示受限类层次结构，当一个值可以从一个有限集中具有一个类型，但不能有任何其他类型。

这个密封类的定义与这个场景非常吻合。 我们希望能够从一个函数返回一个对象:

- 可以代表我们要求的数据，或者
- 可能代表某种错误

我们不想只返回任何类型的对象，因为我们希望在返回的内容上有一些编译时安全。与上面介绍的基本包装方法一样，我们可以改变 parse 函数，使它不会直接返回 ParsedData; 这一次它返回一个密封类。

```kotlin
sealed class ParseResult 

data class Error(val errorMessage: String) : ParseResult()

data class ParsedData(
    val data: String,
    val mimeType: String) : ParseResult()
```

通过定义一个密封类，我们就可以开始定义从它继承的其他类。 在上面的例子中，我们同时声明了一个错误类和一个包含我们试图获取的数据的 ParsedData 类。 你甚至可以进一步返回许多不同类型的错误，每种错误都可以有自己的字段。

我们现在可以修改 parse 函数，以便它:

- 声明 ParseResult 密封类作为其返回类型
- 当解析成功时返回 ParsedData
- 当解析失败时返回 Error

我们的新 parse 函数现在看起来是这样的

```kotlin
private fun parse(url: String): ParseResult {
    val result = URL_PARSE_REGEX.find(url)
    if (result == null) {
        return ParseResult.Error("No match found")
    }

    val mimeType = result.groupValues[2]
    val data = result.groupValues[4]

    return ParseResult.ParsedData(data, mimeType)
}
```

在这一点上，我们没有从以前的基本包装方法中获得多少收益。 但是当我们想使用 ParseResult 对象时，我们就会看到它的优点。

#### 密封类和 When()

密封类与 when 表达式很好地配对。 这很像 Java 的 Switch 操作符，只不过 Kotlin 版本会自动在每个 when 块中为你添加值。

```kotlin
val result = parse(url)
when (result) {
    is ParseResult.ParsedData -> analyzeMimeType(result.mimeType)
    is ParseResult.Error -> log(result.errorMessage)
}
```

在尝试访问 mimeType 之前，不需要转换为 ParsedData，并且在尝试访问 errorMessage 之前不需要转换为 Error。

如果返回 when 表达式的结果，编译器将开始突出显示你尚未处理的任何 ParseResult 子类; 提供进一步的编译时安全，以防忘记处理新类型的密封类。

#### 密封或抽象

你可以把 ParseResult 定义为抽象的而不是密封的，你可能想知道为什么密封更好。 虽然抽象在上面这个例子中是可行的，但它也会允许一些潜在的不合需要的东西。

除了我们定义的 ParsedData 和 Error 子类，ParseResult 作为抽象的子类也允许在项目中的任何地方定义子类。

对于 ParseResult，使用 sealed 类来定义子类的范围有限。

> 密封类可以有子类，但所有子类必须与 sealed 类本身在同一文件中声明。

通过确保可能的 ParseResult 的所有不同类型必须在同一个类中定义来帮助减少错误。

## 结论

在调用返回数据的函数时，常常需要处理可能发生的错误。 通过使用 Kotlin 密封类，你可以有目的地限制从任何给定函数返回的数据类型; 将返回类型限制为不同类型的数据或一个或多个错误类型。

返回的不同类可以彼此保存完全不同的数据。 然而，与 Kotlin 的 when 表达式结合时，每一个不同的类型都可以被优雅地处理。

