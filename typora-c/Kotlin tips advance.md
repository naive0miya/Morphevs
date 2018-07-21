# Kotlin 进阶技巧

> 原文 (Medium)：[Kotlin tips: Singleton, Utility Functions, group Object Initialization and more…](https://medium.com/default-to-open/kotlin-tips-singleton-utility-functions-group-object-initialization-and-more-27cdd6f63a41)
>
> 作者：[Mostafa Gazar](https://medium.com/@mgazar?source=post_header_lockup)

[TOC]

写出好的 Kotlin 代码和使用语言提供的技巧。

使用 Kotlin 有很多好处，它简洁，安全，最重要的是它可以100% 的与 Java 互操作。 它还试图解决 Java 的一些局限性。 看看 Jake Wharton 在2015年写的一篇关于为什么 Square 应该开始使用 Kotlin 的[文章](https://docs.google.com/document/d/1ReS3ep-hjxWA8kZi0YqDbEhCqTt29hG8P44aA9W0DM8)。 

现在是考虑 Kotlin下一个重要功能或项目的好时机，因为 Google 在 Google I / O 中宣布它将成为编写 Android 应用程序的一流语言。

所以这里有一些技巧可以帮助你开始使用语言提供的东西，编写好的 Kotlin 代码。

## Singleton

在 Kotlin ，实现惰性加载和线程安全的单例非常简单，不像 Java， 你必须依靠复杂的[双重检查锁定模式](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)。 尽管Java 枚举单例是线程安全的，但它们并不是懒加载。

```kotlin
object Singleton {
    var s: String? = null
}
```

对于 Kotlin 类，一个[对象](https://kotlinlang.org/docs/reference/object-declarations.html#object-declarations)不能有任何构造函数，但是如果需要一些初始化代码，可以使用 init 块。

```kotlin
Singleton.s = "test" // class is initialized at this point
```

该对象将被实例化，并且它的 init 块将在第一次访问时以一种线程安全的方式被懒惰地执行。

## Utility Functions

支持 Kotlin 对典型 Java 实用程序类的顶级扩展函数 。为了更容易在 Java 代码中使用，使用 @file：JvmName 来指定 Kotlin 编译器生成的 Java 类的名称。

```kotlin
// Use this annotation so you can call it from Java code like StringUtil.
@file:JvmName("StringUtil")
fun String.lengthIsEven(): Boolean = length % 2 == 0
val lengthIsEven = "someString".lengthIsEven()
```

## Object Initialization

使用适用于组对象初始化语句，以便更清晰，更容易阅读的代码。

```kotlin
// Don't 
val textView = TextView(this)
textView.visibility = View.VISIBLE
textView.text = "test"
// Do
val textView = TextView(this).apply {
    visibility = View.VISIBLE
    text = "test"
}
```

一些更多的小技巧！

### let()

使用 let ( ) 可以作为 if 的简洁选择。看看下面的代码：

```kotlin
val listWithNulls: List<String?> = listOf("A", null)
for (item in listWithNulls) {
    if (item != null) {
        println(item!!)
    }
}
```

有了 let ( )，不需要 if。

```kotlin
val listWithNulls: List<String?> = listOf("A", null)
for (item in listWithNulls) {
    item?.let { println(it) } // prints A and ignores null
}
```

### when()

当替换 Java 的开关操作符时，它也可以用来清理一些难以读取的条件。 

```java
// Java
public Product parseResponse(Response response) {
   if (response == null) {
       throw new HTTPException("Something bad happened");
   }
   int code = response.code();
   if (code == 200 || code == 201) {
       return parse(response.body());
   }
   if (code >= 400 && code <= 499) {
       throw new HTTPException("Invalid request");
   }
   if (code >= 500 && code <= 599) {
       throw new HTTPException("Server error");
   }
   throw new HTTPException("Error! Code " + code);
}
```

在 Kotlin ，情况将是这样: 

```kotlin
// Kotlin
fun parseResponse(response: Response?) = when (response?.code()) {
   null -> throw HTTPException("Something bad happened")
   200, 201 -> parse(response.body())
   in 400..499 -> throw HTTPException("Invalid request")
   in 500..599 -> throw HTTPException("Server error")
   else -> throw HTTPException("Error! Code ${response.code()}")
}
```

## Read-only lists, maps, …

Kotlin 区分可变集合和不可变集合（列表，集合，映射等）。 精确控制集合什么时候可以被编辑，是设计良好的 API ，可以用来消除错误。

Kotlin 标准库包含用于可变集合和不可变集合的实用函数和类型。 例如 listOf ( ) 和 mutableListOf ( )。

```kotlin
val list = listOf(“a”, “b”, “c”)
val map = mapOf("a" to 1, "b" to 2, "c" to 3)
```

## Lazy property

懒惰属性是好的，有意义的时候就用吧！保持较低的内存占用是一个很好的用例，如果初始化成本很高，也可以节省 CPU 周期。

懒惰属性只能在第一次访问时计算。 

```kotlin
val str: String by lazy {
     // Compute the string
}
```

## 抵抗诱惑，把所有的东西挤在一个单一的表达

试图把所有东西都压缩成一个单一的表达式是相当诱人的。 只是因为你可以，并不意味着这是一个好主意。

如果单个表达式函数被强制换行，它应该是一个标准函数。

旨在代替可读和干净的代码。

