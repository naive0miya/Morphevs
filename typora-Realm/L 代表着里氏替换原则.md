# L 代表着里氏替换原则

> 原文：[L 代表着里氏替换原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-3/)
>
> 作者：[Donn Felker](https://twitter.com/donnfelker)

[TOC]

这是 SOLID 安卓开发系列原则的第三部分。如果你错过了第一部分或者你不熟悉 SOLID 原则，请看 [第一部分](https://academy.realm.io/cn/posts/donn-felker-solid-part-1/)，这里我们介绍了 SOLID 而且讨论了单一职责原则，和 [第二部分](https://academy.realm.io/cn/posts/donn-felker-solid-part-2/)，这里我们讨论了开闭原则。

## 里氏替换原则

SOLID 缩写的第三个字母是 L，代表着里氏替换原则（LSP）。里氏替换原则在 1987 年的一次会议的演讲上由 Barbara Liskov 提出。里氏替换原则如下描述：

> 一个程序里面的对象应该可以被它的子类替换，而不改变程序的正确性。

那么……这意味着什么？

我很肯定你和我一样，你可能读了好多遍，想理解它到底说的是什么意思，最后往往都是越来越迷糊。不幸的是 (或者有幸的是 - 这取决于你怎么看这个问题)，如果你在维基百科网站上深入理解 [里氏替换原则](https://en.wikipedia.org/wiki/Liskov_substitution_principle) 你会发现文章往往都深入涉及到计算机科学的相关知识，而下面的链接又转到了各个其他的领域。令人吃惊。

问题是，原则是十分容易理解的，你每时每刻都在运用它，你自己都可能没有意识到。你可能已经写过遵循 LSP 原则的代码了。

## 一个用子类代替父类对象的例子

Java 是一个静态类型的语言。编译器非常擅长捕捉类型错误，并通过报告错误引起我们的注意。我们这样做过很多次了。你试图把一个 `String` 类型赋给一个 `Long` 类型，或者反过来，编译器会告诉你，你犯了一个错误。编译器对于让我们写出符合里氏替换原则的代码非常有帮助。

让我们假设你在写一些安卓的代码，你和 Java 中的 [`List`](http://developer.android.com/reference/java/util/List.html) 类型一起工作。我肯定你在你的应用里面时不时地都写过这样的代码：

```java
// Get the ids somehow (loop, lookup, etc)
ArrayList<Integer> ids = getCustomerIds();
List<Customer> customers = customerRepository.getCustomersWithIds(ids);
```

此时，你只关心 `customerRepository` 返回的一个 `List<Customers>`。甚至于你写完了 `CustomerRepository`，但是因为你的后台还没有完成，你决定把你的接口和实现解耦。(嗯，’Interface’…… 也许会和 SOLID 里面的 ‘I’ 有关系？ 😉)

假设你的代码看起来像这样：

```java
public interface CustomerRepository {
   List<Customer> getCustomersWithIds(List<Integer> ids);
}

public class CustomerRepositoryImpl implements CustomerRepository {
   @Override
   public List<Customer> getCustomersWithIds(List<Integer> ids) {
        // Go to API, DB, etc and get the customers.
        ArrayList<Customer> customers = api.getWholeLottaCustomers(ids);
        return customers;
   }
}
```

上面的例子代码中，customer repository 需要一个 customer IDs 的 list，这样它就可以获取那些 customers。 customer repository 只需要一个 customer IDs 的 list，它的类型是 `List<Integer>`。当我们调用 repository 的时候，我们提供了一个如下的 `ArrayList<Integer>`：

```java
// Get the ids somehow (loop, lookup, etc)
ArrayList<Integer> ids = getCustomerIds();
List<Customer> customers = customerRepository.getCustomersWithIds(ids);
```

等等…… customer repository 需要的是一个 `List<Integer>`，不是 `ArrayList<Integer>`。它是如何工作的呢？

这就是一个里氏替换原则起作用的地方。因为 `ArrayList<Integer>` 是 `List<Integer>` 的子类，程序不会出错：我们把要求的类型 (`List<Integer>`) 的实例用它的子类型 (`ArrayList<Integer>`) 的实例替换了。

换句话说，在上面的代码中，我们依赖于一个抽象 (`List<Integer>`)，而且我们能提供一个 (`ArrayList<Integer>`) 的子类，程序就仍然可以正常运行。为什么会是这样呢？

原因是 customer repository 依赖于 `List` 接口提供的契约。`ArrayList` 是 `List` 接口的一个实现，所以，当程序运行的时候，customer repository 不会知道类型是 `ArrayList`，而把它认为是接口 `List`。[LSP 维基百科文章](https://en.wikipedia.org/wiki/Liskov_substitution_principle#Principle) 的原则部分解释得非常好，所以我引用如下：

> Liskov 的可替换性的子类型的概念定义了关于变化的对象具备可替代性的概念；这就是说，如果 S 是 T 的子类型，那么程序中类型 T 的对象是可以被类型 S 的对象替换的，而不会改变任何程序预期的行为。(例如，正确性)。

简而言之，我们可以用 `List` 的任何扩展来替换 `List`，而且不会破坏程序的行为(但是我仍需要测试来保证这点)。

我肯定你可以在你的应用中到处都能看到像例子中的那样的代码。这很普遍。如果你写了这样的代码，你就遵循了里氏替换原则。这很容易，对吧？ 👍

我可以从上面的例子中再深入一点！因为 `CustomerRepositoryImpl` 实现了 `CustomerRepository` 接口的契约，我们知道任何依赖 `CustomerRepository` 接口的地方都会从里氏替换原则中受益。

### 但是 怎么做呢？

非常容易。假设你在测试中，你使用 Dagger 来注入 `MockCustomerRepository`，这也是个实现 `CustomerRepository` 接口的类。调用代码只知道它是和 `CustomerRepository` 的实例一起工作的，而不知道是一个桩。因为子类 (`MockCustomerRepository`) 遵循了父类型 (`CustomerRepository`) 一样的契约，程序的正确性不会受影响。这给测试和打桩提供了无数的便利，而这一切都是因为我们遵循了里氏替换原则。

## 如果需要一个特殊的类型呢？

让我们在脑海里再想想这个原则。如果你熟悉 Java 类型系统，你可能知道 [`List`](http://developer.android.com/reference/java/util/List.html) 实际上实现了 [`Collection`](http://developer.android.com/reference/java/util/Collection.html)。

下面能编译成功吗？(**提示： 🙅**)

```java
// Get the ids somehow (loop, lookup, etc)
Collection<Integer> ids = getCustomerIds();
List<Customer> customers = customerRepository.getCustomersWithIds(ids);
```

**为什么不呢？**

`getCustomersWithIds` 仅仅接受 `List<Integer>`。`List` 实现了 `Collection`，但是 `Collection` 没有实现 `List`。所以当 `List` 是一个 `Collection`，`Collection` 不必要是一个 `List`。在这个例子中，编译器不能证明 `Collection<Integer>` 肯定是一个 `List<Integer>`。在这种情况下，这些类型是不能兼容的。

## 返回类型，参数和更多

里氏替换原则也不限于参数。例如，原始的代码表述了 customer repository 是如何返回一个 `ArrayList<Customer>`：

```java
public interface CustomerRepository {
   List<Customer> getCustomersWithIds(List<Integer> ids);
}

public class CustomerRepositoryImpl implements CustomerRepository {
   @Override
   public List<Customer> getCustomersWithids(List<Integer> ids) {
        // Go to API, DB, etc and get the customers.
        ArrayList<Customer> customers = api.getWholeLottaCustomers(ids);
        return customers;
   }
}

// Somewhere else in the program
List<Customer> customers = customerRepository.getCustomersWithIds(...);
```

customer repository 返回来一个 `ArrayList<Customer>`。调用者什么都不知道，唯一知道的事情是它知道它得到了一个 `List<Customer>`。这和 Realm 的行为很相似。例如，[`RealmResults`](https://realm.io/docs/java/latest/api/io/realm/RealmResults.html) 类型也实现了 `List<E>`。

所以这意味着什么？当然，我可以从我的 repository 返回一个 `RealmResults<Customer>`，而且调用者仅仅知道它们接收到了一个 `List<E>`。这给你的代码带来了疯狂的灵活性。我可以创建我自己订制化的 list，它扩展了 `LinkedList` 而且返回它。因为 `LinkedList` 返回了一个 `List`，应用继续使用这个 `List`，保证我们没有改变程序的正确性。

你可以依赖一个抽象，而不用担心你的程序会打破这个规则。这在安卓框架中到处可见。

## 结论

里氏替换原则是很简单的，你可能都没有意识到它有一个名字。每天你都在使用它，而且它给开发者带来了许多好处。你可以不仅仅实现 List 也可以实现你自己的接口。

敬请期待这个系列的第四部分，我们会谈论更多的接口方面的好处。

请看本系列的第四部分，[接口隔离原则](https://realm.io/news/donn-felker-solid-part-4/)！