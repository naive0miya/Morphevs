# “Effective Java”如何影响了 Kotlin 的设计ー第2部分

> 原文 (Medium)：[How “Effective Java” may have influenced the design of Kotlin — Part 2](https://medium.com/@lukleDev/how-effective-java-may-have-influenced-the-design-of-kotlin-part-2-89844d62ddf3)
>
> 作者：[Lukas Lechner](https://medium.com/@lukleDev?source=post_header_lockup)

[TOC]

又见面了！ 

这是博客系列的第二部分， 关于《 [Effective Java](https://www.amazon.com/gp/product/0321356683/ref=as_li_tl?ie=UTF8&tag=lukle-20&camp=1789&creative=9325&linkCode=as2&creativeASIN=0321356683&linkId=8ec63da61faf8c3b6114ea35d8f13208) 》如何影响了 Kotlin 的设计。在继续之前，如果你还没有读过的话，先看看第一部分。 

我们继续！

## 6. 默认情况下的 final

“Effective Java” 中的第17项表明，每个类都应该不是可以子类化的，也不应该被精心设计和记录来支持继承。在 Java 中，除非明确指定类为 final，否则每个类都可以进行子类化。如果你忘记为继承设计和记录类为 final，那么当客户认为他们可以创建子类、覆盖某些方法并假设一切仍然如预期的那样工作时，就会出现问题。 

在 Kotlin，每个类都是默认 final。你必须明确地使用关键字 open，以使该类能够继承，这与 Java 的 final 相反。这可以防止创建非有意为继承而设计的 non-final 类。

并非 Kotlin 社区的每个人都对 “默认 final” 设计选择感到满意。 [Kotlin 论坛](https://discuss.kotlinlang.org/)正在讨论这个[有争议的话题](https://discuss.kotlinlang.org/t/classes-final-by-default/166)。

最新消息：[最近宣布](https://blog.jetbrains.com/kotlin/2017/01/kotlin-1-1-beta-is-here/)的 Kotlin 1.1版测试版提供了一个编译器插件，用于默认开放类。

## 7. 没有检查异常

Java 有一个称为 “检查异常” 的特性，其中编译器强制函数的调用者捕获（或重新抛出）异常。这个功能通常是有问题的。“Effective Java” 有一整节介绍如何正确使用和处理 checked 和 unchecked（运行时）异常。Kotlin [文档](https://kotlinlang.org/docs/reference/exceptions.html#checked-exceptions)中描述的一个检查异常的问题是，你有时不得不捕捉永远不会发生的异常。 这就导致了空 catch 块和冗长代码。更糟糕的是，当开发人员被迫对可能的异常做出反应时，他们往往会感到恼火，导致他们忽略它们，同时导致空的捕获块。 第65条说,"不要忽视异常"。 

根据第59项，检查异常通常是不必要的，应该通过在调用对象的状态之前检查对象的状态或者通过区分返回值（如 null）来避免。

在我的研究中，我发现了许多其他的 异常：

1. throws 子句将实现细节添加到接口，这是不好的;
2. 版本控制可能有问题; 如果你更改实现并向函数添加 throws 子句，则会破坏 API 更改。
3. 调用函数不应该指定调用者如何执行其异常处理。

由于存在大量潜在问题，Kotlin 和其他优秀的编程语言（如 C＃）没有异常检查。为了让调用者知道可能的异常，你应该用 throwstag 在函数的 [kdoc](https://kotlinlang.org/docs/reference/kotlin-doc.html) 文档中定义它们。

## 8. 强制空检查

在 Java 中，公共方法的方法签名不会告诉你返回的值是否可以为 null。例如拿这个签名：

```java
public List<Item> getSelectedItems()
```

没有选择项目时会发生什么？这个方法然后返回 null 吗？如果不考虑这个方法的实现，我们不能确定(除非我们很幸运，这个签名对返回类型有一个很好的 javadoc 描述)。开发人员可能犯的两个错误是：(1) 当 null 可以为 null 时，忘记检查 null 的返回值，导致著名的 NullPointerException，或者; (2)检查 null，尽管它永远不能为 null，从而导致不必要的代码。 

在项目43中，Joshua Bloch 建议总是返回一个空的集合数组而不是 null。这个项目让我想到了可以为空和不可为空的类型。使用 Kotlin 的[空安全性](https://kotlinlang.org/docs/reference/null-safety.html)，你知道返回值是否可以为 null。以上例为例：返回类型 List \<Item>？ 意味着它可以为 null，而 List \<Item> 则意味着它不能为 null。如果它可以为 null，则编译器会强制我们在访问其属性或调用其函数之前检查它是否为 null。所以编译器的更强大的类型系统可以防止开发人员犯错误。生活可以如此简单。

## 9. 没有原始类型

泛型在1.5版本中被添加到 Java 中，并且它们可以成为完成类型安全性的一种很好的方式。为了实现向后兼容性，仍然可以使用原始类型，但是 Joshua Bloch 建议在第23项中总是使用泛型类型（List \<Integer> 而不是 List）来避免 ClassCastExceptions。Kotlin 不允许使用原始类型，因此你必须始终指定泛型类型的类型参数，从而生成更多类型安全的代码。

## 10. 更易于定义差异的方法

第28项讨论了使用有界通配符，这在 Java 中非常棘手，难以理解。Joshua Bloch 定义了 PECS 助记符（生产者 extends，消费者 super）来帮助你决定是否使用 <？extends E>（协方差）或 <？super E>（逆变）。Kotlin 团队努力使方差处理更容易。他们摆脱了通配符，并且像 C＃ 一样提供了关键字 in 表达协变性和 out 表达逆向性。这篇博文太短，无法详细解释这些关键字的工作原理，但如果你有兴趣，可以查找 [Declaration-Site variance](https://kotlinlang.org/docs/reference/generics.html#declaration-site-variance)， [Type Projections](https://kotlinlang.org/docs/reference/generics.html#type-projections) 或 [Generics](https://kotlinlang.org/docs/reference/generics.html)。根据他们的文档，这些关键词是更加自我解释的，可以在不记住记忆法的情况下理解。

## 结束

所以这些是来自 “Effective Java” 的10件事，在我看来，它影响了 Kotlin 编程语言的设计。我很好奇，如果你对这本很棒的书如何影响 Kotlin 的其他方面有进一步的想法。我真的很感谢你的评论！

在 twitter 上[关注我](https://twitter.com/lukleDev)，以后再也不会错过任何内容;）

链接：[第3部分](https://medium.com/@lukleDev/how-effective-java-may-have-influenced-the-design-of-kotlin-part-3-7a01c9627e86)

