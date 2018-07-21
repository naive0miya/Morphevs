# Kotlin 正确使用 Dagger2 @Named 注解

> 原文 (Medium)：[Correct usage of Dagger 2 @Named annotation in Kotlin](https://medium.com/@WindRider/correct-usage-of-dagger-2-named-annotation-in-kotlin-8ab17ced6928)
>
> 作者：[Svetlozar Kostadinov](https://medium.com/@WindRider?source=post_header_lockup)

[TOC]

这是我在 Medium 的第一篇文章。 我喜欢这个地方。 很棒的人和思想！ 

让我告诉你关于最近困扰我的 Dagger 2 中的一些微小而微妙的东西，并且在发生什么事情之后，让你免于花费几个小时。

假设你正在注入一个成员变量：

Java:

```java
class MyPresenter {
    @Inject @Named("api1") Api api;
    ...
}
```

在 Kotlin 1.1中，相同的效果必须有不同的语法：

```kotlin
class MyPresenter {
    @filed:[Inject Named("api1")]
    internal lateinit var api: Api;
    ...
}
```

原因在于，在 Kotlin，注解需要稍微复杂一些，以便能够从 Java 角度进行预期的工作。这是因为一个 Kotlin 元素可能是字节码中多个 Java 元素的立面。 例如，Kotlin 属性是 Java 基础成员变量、 getter 和 setter 的表面。 你对这个属性进行了注解，但 Dagger 希望被注解的是底层的 filed 。 

这就是为什么 JetBrains 为了明确指定使用站点目标而添加了额外的语法  - 特别是 filed 和其他几个（文档中的完整列表：http://kotlinlang.org/docs/reference/annotations.html#annotation-use-site-targets).

在上面的例子中，如果没有 @filed:[Inject Named("api1")]， @Inject 注解将被附加到属性而不是字段上。

注入原始类型 （使用 @set）

现在让我们尝试注入的不是一个对象，而象 Int 这样的原始类型 ：

```kotlin
@filed:[Inject Named("logoIcon")] var logoIcon: Int = 0
```

oops：

```kotlin
error: Dagger does not support injection into private fileds
e: private int logoIcon;
```

事实证明，原始类型的处理方式不同。错误应该来自缺少的 lateinit 说明符。因为在 Kotlin 中，原始变量不能使用 lateinit  。显然，lateinit 使底层的 filed 公开 。那么，让我们尝试 “setter 注入”。

```kotlin
@set:[Inject Named("logoIcon")] var logoIcon: Int = 0
```

用 Java 术语来说，更容易理解。在 Java 中它相当于：

```java
private int logoIcon = 0;
public int getLogoIcon() { return this.logoIcon; }

@Inject
@Named("logoIcon")
public void setLogoIcon(int logoIcon) { this.logoIcon = logoIcon; }
```

现在起作用了。dagger 2 非常灵活，它也可以通过调用 setter 来注入，不仅通过构造函数参数和字段。

在 Kotlin 我们这样注入 Named filed: 

```kotlin
@filed:[Inject Named("api1")] internal lateinit var api: Api
// 或者如果你注入一个原语 
@set:[Inject Named("logoIcon")] var logoIcon: Int = 0
```

不是: 

```kotlin
@Inject @Named("api1") internal lateinit var api: Api
```

Have fun kotling :)

