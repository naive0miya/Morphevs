# Kotlin 进阶技巧(续)

> 原文 (Medium)：[ Advanced Kotlin tips: tail recursion, sealed classes, local, infix and inline functions and more…](https://android.jlelse.eu/advanced-kotlin-tips-local-infix-and-inline-functions-tail-recursion-sealed-classes-and-more-2a53b00d5423)
>
> 作者：[mgazar](https://medium.com/@mgazar) 

[TOC]

## 本地函数（Local ）
本地函数有利于代码重用，只要注意不要过度使用它们以避免混淆。
```kotlin
fun foo(a: Int) {
    fun local(b: Int) {
        return a + b
    }
    return local(1)
}
```

## 中缀函数（Infix ）
中缀函数可读性好，因为它允许输入诸如 “test” foo “x” 的东西，很酷！
```kotlin
infix fun String.foo(s: String) {
    ...
}
// Call extension function.
"test".foo("x")
// Or call extension function using infix notation.
"test" foo "x"
```
中缀函数必须有一个参数。

## 内联函数（Inline ）
Kotlin 中的 lambda 表达式转换为 Java 6或 Java 7中的 Java 匿名类，这是一个开销。 Lambda 调用正在影响具有性能影响的调用堆栈。

内联函数可以用于平滑调用，而不是调用另一个方法调用并将其添加到调用堆栈。 因此，当我们传递 lambda 表达式时，使用内联函数是有意义的。

```kotlin
inline fun callBlock(block: () -> Unit) {
    println("Before calling block")
    block()
    println("After calling block")
}
```
当我们调用 callBlock 时，它会被翻译成如下所示：
```kotlin
callBlock { println("The block operation") }
// Rough java bytecode
String var1 = "Before calling block";
System.out.println(var1)
String var2 = "The block operation";
System.out.println(var2);
var1 = "After calling block";
System.out.println(var1);
```
如果函数未标记为内联，则与以下内容对比
```kotlin
callBlock { println("The block operation") }
// 粗糙的java字节码
callBlock((Functinos0)null.INSTANCE);
```
你必须小心地使用内联函数，因为复制了被调用的方法内容，如果函数的主体太大时，你不会真的想这样做。

知道了这一点，下面这些显然没有任何意义，因为它没有任何效果。 

```kotlin
inline fun foo(noinline block: () -> Unit) {// 单个lambda标记为noinline
inline fun foo() { // No lambdas
```

## 尾递归（tailrec ）
通过使用 tailrec，我们让编译器知道它可以用 for 循环或 goto 语句替换方法调用。只有当函数中的最后一个调用只调用它自己，你才能使用它。 

```kotlin
tailrec fun findFixPoint(x: Double = 1.0): Double         
        = if (x == Math.cos(x)) x else findFixPoint(Math.cos(x))
```

## 密封类（Sealed ）
根据 [Kotlin 的参考](https://kotlinlang.org/docs/reference/sealed-classes.html)，我们应该使用密封类来表示受限类层次结构，当一个值可以从一个有限集合中拥有一个类型，但不能有任何其他类型。 

换句话说，它们在返回不同但相关的类型时是很好的。
```kotlin
sealed class Response 
data class Success(val content: String) : Response() 
data class Error(val code: Int, val message: String) : Response() 
fun getUrlPage(url: String) : Response {
    val valid = // Some logic here!
    if (valid) {
        return Success("Content")
    else {
        return Error(404, "Not found")
    }
}
// Here is the beauty of it.
val response = getUrlPage("/")
when (response) {
     is Success -> println(response.content)
     is Error -> println(response.message)
}
```
密封类必须在一个文件中定义。 

## 一些小的建议
### 局部返回（Local return）

他们主要是帮助 lambdas，但让我们用一个基本的代码示例来解释它。你认为在下面的代码中的第一个返回是什么？
```kotlin
fun foo(list: List<String>): Boolean {
    list.forEach {
        if (...) {// Some condition.
           return true
        }
    }
    return false
}
```
它得到 foo 返回 true 。您也可以选择只从 forEach 范围返回。
```kotlin
fun foo(list: List<String>): Boolean {
    list.forEach {
        if (...) {// Some condition.
           return@forEach // Just like calling break
        }
    }
    return false
}
```
如果这没有多大意义，请查看以下代码：
```kotlin
fun foo() {
    Observable.just(1)
                .map{ intValue ->
                    return@map intValue.toString() 
                }
                ...
}
```
如果我们在上面的代码片段中使用了 return，它会返回到 foo，这在这里没有多大意义。返回 @map 虽然 map 函数的返回结果是预期的行为 

### 运算符重载（Operator overloading）
使用 operator 覆盖支持的操作符。
```kotlin
operator fun plus(time: Time) {
    ...
}
// This will allow the following statement.
time1 + time2 
```
小心不要过度使用操作符重载，大部分时间使用它是没有意义的。查看[规范](https://kotlinlang.org/docs/reference/operator-overloading.html)操作符重载不同操作符的约定列表。

### Lambda扩展（Lambda extensions）
他们只是更好地阅读和键入，就像标记。
```kotlin
class Status(var code: Int, var description: String)
fun status(status: Status.() -> Unit) {}
// This will allow the following statement
status {
   code = 404
   description = "Not found"
}
```

### lateinit
如果在初始化之前尝试使用它，lateinit 属性将抛出 UninitializedPropertyAccessException 类型的异常。

### 伴随对象
[Companion Objects](https://kotlinlang.org/docs/reference/object-declarations.html#companion-objects) 是最接近 Java 静态方法的东西。
```kotlin
class MyClass {
  @Jvmstatic
  companion object Factory {
    fun create(): MyClass = MyClass()
  }
}
```
可以简单地使用类名作为限定符来调用伴随对象的成员: 

```kotlin
val instance = MyClass.create()
```
 如果你想要在 java util class 中合并多个 kotlin 文件，我会推荐使用 @file：JvmName（“SomethingUtil ”）和  @file：JvmMultifileClass 作为扩展名。 

