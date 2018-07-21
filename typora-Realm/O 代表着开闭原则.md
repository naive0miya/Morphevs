# O 代表着开闭原则

> 原文：[O 代表着开闭原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-2/)
>
> 作者：[Donn Felker](https://twitter.com/donnfelker)

[TOC]

这是 SOLID 安卓开发系列原则的第二部分。如果你错过了第一部分或者不明白 SOLID 原则是什么，看看 [第一部分](https://academy.realm.io/cn/posts/donn-felker-solid-part-1/)，这里介绍了 SOLID 和单一职责原则。

## 开闭原则

SOLID 中的 ‘O’ 指的是开闭原则。开闭原则如是说：

> 软件实体（类，模块，方法，等等）应该对扩展开放，对修改关闭。

这听起来简单，但它也是那些你在脑海里重复足够多次后，你会发觉非常迷惑的原则。基本原则是你应该 **力图写出你不需要每次需求一改变代码就要改变的代码**。在安卓中，我们使用 Java，所以这个原则可以用继承和多态来实现。[1](https://academy.realm.io/cn/posts/donn-felker-solid-part-2/#fn1)

## 一个开闭原则的例子

下面是个工程上常见的例子，它简单地说明了开闭原则的内容和实现方法。下面的这个例子非常简单，而且清晰地可视化了开闭原则。

让我们假设你有一个这样的应用，这个应用计算你提供的图形的面积。虽然简单，我几年前在明尼苏达的一家农作物保险公司工作的时候就遇到过一模一样的问题。那个应用需要计算一个给定区域的所有农作物的保险报价。正如你可能知道的那样 - 农作物有各种图形和大小，对的，甚至是圆形、三角形和各种其他的多边形。

好吧，回到例子……

作为一个好的程序员，我们把面积计算抽象到一个叫做 `AreaManager` 的类中。`AreaManager` 类具备 [单一职责](https://academy.realm.io/cn/posts/donn-felker-solid-part-1/) - 计算形状的全部面积。

让我们假设我只工作在矩形的农作物上，所以我们有一个 `Rectangle` 类来表示它。这是这些类的代码：

*Rectangle.java*

```java
public class Rectangle {
    private double length;
    private double height;
    // getters/setters ...
}
```

*AreaManager.java*

```java
public class AreaManager {
    public double calculateArea(ArrayList<Rectangle>... shapes) {
        double area = 0;
        for (Rectangle rect : shapes) {
            area += (rect.getLength() * rect.getHeight());
        }
        return area;
    }
}
```

`AreaManager` 类一直都把它的工作完成得很好，直到下周我们有一个新的作物的类型出现 - 一个圆形：

*Circle.java*

```java
public class Circle {
    private double radius;
    // getters/setters ...
}
```

因为我们有一个新的形状需要计算，我们需要修改 `AreaManager`：

*AreaManager*

```java
public class AreaManager {
    public double calculateArea(ArrayList<Object>... shapes) {
        double area = 0;
        for (Object shape : shapes) {
            if (shape instanceof Rectangle) {
                Rectangle rect = (Rectangle)shape;
                area += (rect.getLength() * rect.getHeight());
            } else if (shape instanceof Circle) {
                Circle circle = (Circle)shape;
                area += (circle.getRadius() * cirlce.getRadius() * Math.PI;
            } else {
                throw new RuntimeException("Shape not supported");
            }
        }
        return area;
    }
}
```

这段代码有些坏代码的味道了。

如果我们有一个三角形，或者其他多边形，我们需要一遍一遍地修改这段代码。

这个类违反了开闭原则。它没有针对修改关闭和针对扩展开放。每一次新的形状出现的时候，我们都要修改 AreaManager。我们想避免这种情况。

我们如何使得 `AreaManager` 类的开闭原则友好些呢？

## 采用继承实现开闭原则

因为 `AreaManager` 负责计算所有形状的全部面积，而且形状计算对每一个图形都是唯一的，所以看起来我们需要做的就是把每个形状的面积计算移动到对应的类中间。

嗯，但是 `AreaManager` 仍然需要知道所有的图形，对吧？不然它是怎么知道它遍历的对象有一个面积函数呢？当然，这可以用反射来实现 (*咳咳* 又 *咳咳*) 或者我们可以让每一个形状类继承自一个接口：`Shape` 接口 (这也可以是一个抽象类)：

*Shape.java*

```java
public interface Shape {
    double getArea();
}
```

每一个类必须实现这个接口 (或者扩展抽象类，如果这是你想要的话 - 无论什么原因) 像这样：

*Rectangle.java*

```java
public class Rectangle implements Shape {
    private double length;
    private double height;
    // getters/setters ...

    @Override
    public double getArea() {
        return (length * height);
    }
}
```

*Circle.java*

```java
public class Circle implements Shape {
    private double radius;
    // getters/setters ...

    @Override
    public double getArea() {
        return (radius * radius * Math.PI);
    }
}
```

我们现在使 `AreaManager` 通过这个抽象符合开闭原则了：

```java
public class AreaManager {
    public double calculateArea(ArrayList<Shape> shapes) {
        double area = 0;
        for (Shape shape : shapes) {
            area += shape.getArea();
        }
        return area;
    }
}
```

我们对 `AreaManager` 做了一些改变，来使它对于修改闭合但是对于扩展开放。如果我们需要增加一个新的形状，例如八角形，`AreaManager` 就不需要做修改了，因为它对于扩展 `Shape` 接口是开放的。

## 安卓中的开闭原则

形状是个学习的好例子，而且当你在农作物保险公司工作的时候是非常有用的。但是这对于安卓系统来说，如何应用呢？好吧，它不仅仅用于安卓，它适用于任何语言。安卓有一些非常优美的开闭原则的实现，它们十分有用。让我们看看 -

安卓开发的初学者一般都不知道的事情是，那些内嵌的安卓视图，像 [Button](http://developer.android.com/reference/android/widget/Button.html)， [Switch](http://developer.android.com/reference/android/widget/Switch.html) 和 [Checkbox](http://developer.android.com/reference/android/widget/CheckBox.html) 都是 [TextView](http://developer.android.com/reference/android/widget/TextView.html) 对象。看看这个截图，你可以发现其他许多视图也是继承于 TextView 的。

![Android TextView](https://images.contentful.com/s72atsk5w5jo/3Y5aHUDzRCC00sssmYiYei/4775397762d74132e48712c7f1e3c250/donn-felker-solid-part-2-android-textview-ocp.png)

这意味着安卓的视图系统是对修改闭合但对扩展开发的。如果你打算通过创建你自己的 CurrencyTextView 改变文字的样子，你只需要扩展（继承） TextView，然后在那里实现你的视图逻辑。安卓视图系统不关心你使用的是一个新的 CurrencyTextView, 它只关心你的类是不是遵循了一个特定的 TextView 的要求。安卓会依赖一些特定的函数来展示自己，然后你的视图就被画在屏幕上了。

同样的事情也发生在 [ViewGroup](http://developer.android.com/reference/android/view/ViewGroup.html) 类中：

![Android ViewGroup](https://images.contentful.com/s72atsk5w5jo/s5Oroa259IU6KisAOKMSm/18246bc70f4a76ad506bbd9baa75b3a3/donn-felker-solid-part-2-android-viewgroup-ocp.png)

有许多不同的 ViewGroups (RelativeLayout, LinearLayout, etc) 而且安卓视图系统知道它们是如何一起工作的。你可以非常容易地 [创建你自己的 ViewGroup](http://javatechig.com/android/how-to-create-custom-layout-in-android-by-extending-viewgroup-class) 通过扩展 `ViewGroup`。最后，你可以写些依赖于 `ViewGroup`， `TextView`，或者 `View` 类的代码来做些特别的事情。

依赖于抽象类，比如 `View`， `TextView`， `ViewGroup` 和其他更多的类，允许你写出对修改闭合对扩展开发的代码。

## 结论

开闭原则不仅限于安卓视图系统，但是安卓视图系统是该原则在真实世界里运用的一个简单体现，成千上万的开发者每天都在使用它。你也可以写些开闭原则友好的代码。通过少许计划和抽象，你可以写出更容易维护和扩展的代码，每次当新功能来临的时候，你不需要每次都做代码修改了。

正如所有的事情一样，你不可能在项目开始的时候看到创建抽象的可能性。而且，这样做只会带来过度复杂的代码，而且只是实现了某个模式。在我的经验里面，我发现我常常在多次修改某个类的时候才会使用到开闭原则。那时，我会保证代码是被测试验证过的，然后我会运用开闭原则来重构它。这使我能够有一个安全的测试保障和写出更多的可维护的代码，同时也保持轻量和 [每天开发的最小变化](https://academy.realm.io/posts/kau-donn-felker-minimum-viable-development-android/)。

把里氏替换原则作为本系列的下一篇，它是我最喜欢的原则之一。

\1. 从技术上说，开闭原则有两个变种。开闭原则是 Bertrand Meyer 创建的，它的一个变种是 Meyer 的开闭原则。另一个变种是 Polymorphic 的开闭原则。这两个原则都使用继承作为解决方案。这里，我引用 Robert C. Martin 的多态解释 [↩](https://academy.realm.io/cn/posts/donn-felker-solid-part-2/#ref1)

看看本系列的第三部分，[里氏替换原则](https://academy.realm.io/cn/posts/donn-felker-solid-part-3/) ！