# I 是接口隔离原则

> 原文：[I 是接口隔离原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-4/)
>
> 作者：[Donn Felker](https://twitter.com/donnfelker)

[TOC]

欢迎回到 SOLID 安卓开发系列的第四部分。今天我将说道 SOLID 缩写中的 I - 接口隔离原则 (ISP)。

如果你错过了前三部分，非常容易跟上：

- [S: 单一职责原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-1/)
- [O: 开、闭原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-2/)
- [L: 里氏替换原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-3/)
- **I: 接口隔离原则 (本文)**
- D: 敬请期待。

现在我们开始系列的第四部分……

## 接口隔离原则

接口隔离原则这样告知作为开发者的我们：

> 客户端应该依赖最小的接口。

另一种说法是：

> 使用多个专门的接口比 
> 单一接口要好。

## 多个专门的接口？

当我第一次读到 ISP 的定义的时候，我不太明白这是什么意思 - 多个接口？

什么？啊 ？

事实上这很容易理解。让我们从一个例子开始，我们都非常熟悉安卓系统了 - 安卓的视图系统。

如你所知的那样，安卓 [View](http://developer.android.com/reference/android/view/View.html) 类是所有安卓视图的超级根父类。它是 [TextView](http://developer.android.com/reference/android/widget/TextView.html)， [Button](http://developer.android.com/reference/android/widget/Button.html)， [LinearLayout](http://developer.android.com/reference/android/widget/LinearLayout.html)， [CheckBox](http://developer.android.com/reference/android/widget/CheckBox.html)， [Switch](http://developer.android.com/reference/android/widget/Switch.html)，等等类的根。比如说，如果是一个按钮，超级根父类就是 [View](http://developer.android.com/reference/android/view/View.html)。

让我们假设你是一个在 2000 年代中后期的黑暗时代中，是一个编写安卓操作系统的开发者。你发现每一个视图都有可能被点击。所以，作为一个好的 Java 开发者，你创建了一个接口叫做 OnClickListener，它像如下那样嵌在视图类中：

```java
public interface OnClickListener {
    void onClick(View v);
}
```

我肯定你觉得这看起来熟悉，对吗？：）

随着时间流逝，你发现你需要一个监听长按事件的接口。这是一个非常简单的决定，你就把他放进 OnClickListener 里面就好了。这就是一个长按，对吧？没什么大不了的……现在你的类看起来像这样：

```java
public interface OnClickListener {
    void onClick(View v);
    void onLongClick(View v);
}
```

又过了一段时间，你发现你需要给你的视图添加一些触摸监听接口。接口依旧很小，所以你又增加了一个方法。什么都不会受影响，对吧？

于是你的接口变成了下面的样子：

```java
public interface OnClickListener {
    void onClick(View v);
    void onLongClick(View v);
    void onTouch(View v, MotionEvent event);
}
```

这时，你决定把接口名字从 `OnClickListener` 变成 `ViewInteractions`，或者类似的名字。为何？主要是因为触摸事件和点击事件不太一样了。

正如你所说，如果你给这个接口持续添加方法的话，它会很快就会变得十分笨重 (在我看来它已经是了)。

接口现在变成了一个问题 - 它变得通用而且被污染了。

## 为什么通用的被污染的接口是个问题？

就上面的这个同样的 `OnClickListener` 来说，让我们假设我们正在用安卓 SDK 来开发应用 (记住，我们假设我们在编写安卓 SDK……)。我想给一个按钮添加一个点击监听者，所以我这样写：

```java
Button create = (Button)findViewById(R.id.create);
create.setOnClickListener(new View.OnClickListener {
    public void onClick(View v) {
       // assume this is a todo based app.
       myDatabase.createTask(...);
    }

    public void onLongClick(View v) {
        // do nothing, we're not long clicking
    }

    public void onTouch(View v, MotionEvent event) {
        // do nothing, we're not worried about touch
    }
});
```

最后的那两个方法，`onLongClick` 和 `onTouch` 什么都不做。当然，我们可以在那里写些代码，但是如果我不需要它呢？大部分的情况是我只关心点击事件，不是触摸，不是长按。这些当然是在同一个类里面你需要关心的接口，但是大部分开发者在这种情况下只关心点击监听者。

接口要是太通用了就会要求客户端实现所有的方法，哪怕他不需要它们。做的事情太多了。让我们重新看看 ISP 的定义：

> 客户端应该依赖最小的接口。

通过这样，你减少了对客户端产生不必要的回调函数的污染 (使用我们接口的应用，或者在这个例子里面，安卓应用) 和代码，这是我们实现和运行中不需要的东西。客户端应该只实现它需要的接口，多一个都不需要。上面的例子中，客户端应用不需要 `onLongClick` 和 `onTouch`。

谢谢谷歌的开发团队们都非常熟悉接口隔离原则，而且在这个原则下做了许多很棒的工作。安卓 SDK 中我们不需要重载 `OnClickListener`，我们有一个非常简单和专门的监听接口 - 一个只需要实现 `onClick` 的接口：

```java
// nested in the View class
public interface OnClickListener {
    void onClick(View v);
}
```

如果开发者想对一个视图上的长按时间做响应，它们需要实现 `OnLongClickListener`。如果他们想和触摸事件一起工作，他们可以实现 `OnTouchListener`。

每一个接口都是专门为他做的工作而定制的。

## 真实世界的 ISP

我在许多不同的公司工作过，过去几年中我碰到的通用的接口数不胜数。一个例子就是我曾经想要开发的 SDK。这个 SDK 的客户需要实现庞大的通用接口，哪怕它只关心这些接口中的一个。一个 3-4 行代码能够解决的问题，现在需要 40-60 行代码来实现，就是因为这些无用的函数。

当你在写一个接口，并且打算给他增加一个函数的时候，问问你自己你是不是应该创建另外一个对于客户来说更具体的接口。

有些时候你需要你的接口有多个函数，那没有问题。比如，安卓 [TextView](http://developer.android.com/reference/android/widget/TextView.html) 有函数 [addTextChangedListener](http://developer.android.com/reference/android/widget/TextView.html#addTextChangedListener(android.text.TextWatcher))。 [TextWatcher](http://developer.android.com/reference/android/text/TextWatcher.html) 接口有三个方法：

```java
public interface TextWatcher extends NoCopySpan {

    public void beforeTextChanged(CharSequence s, int start, int count, int after);

    public void onTextChanged(CharSequence s, int start, int before, int count);

    public void afterTextChanged(Editable s);
}
```

这不是一个通用的接口。这些函数作为接口都非常具体而且客户很可能很想和它们交互，因此把它们放在同一个接口中是一个正确的事情。

## 结论

接口隔离原则的目标是帮助你的应用解耦，这样，维护、更新和重新部署都会更容易。我在我的应用里面创建了成千上万的接口，特别是当我使用注入依赖容器的时候 (这是我们 SOLID 系类的最后一篇中会谈及)。

为什么它使你的应用更容易部署？

假设你已经发布了一个庞大的通用的受污染的接口，晚些时候你才返现你需要拆分它。那些依赖你的接口的客户应用也需要有很大的拆分改变。它们依赖在这个接口上，且有着特定的签名，现在你需要改变它。

这不仅仅对编写 SDK 的同事们有用，对编写应用的开发者也适用。某些时候，你需要创建一些抽象而那些抽象就是你需要交互的接口 API。当你需要这些抽象的时候，确保这些接口是专门的而且不要太通用。这样，你的代码会少点麻烦，容易维护，更重要的是，你会在你的项目里工作的很愉快。

敬请期待下周的 SOLID 系列的最后一篇！