# “Effective Java”如何影响了 Kotlin 的设计ー第1部分

> 原文 (Medium)：[How “Effective Java” may have influenced the design of Kotlin — Part 1](https://hackernoon.com/how-effective-java-may-have-influenced-the-design-of-kotlin-part-1-45fd64c2f974)
>
> 作者：[Lukas Lechner](https://hackernoon.com/@lukleDev?source=post_header_lockup)

[TOC]

Java 是一种伟大的编程语言，但是有一些已知的缺陷、常见的陷阱和从早期继承下来的不太好的元素(1995年发布了1.0)。一本关于如何编写好 Java 代码，避免常见的编码错误和处理它的弱点的书是 Joshua Bloch 的《 Effective Java 》 它包含了78个部分，叫做"项目"，给读者提供了有关语言不同方面的宝贵建议。 

现代编程语言的创造者有一个巨大的优势，因为他们能够分析已有语言的弱点，让事情变得更好。 Jetbrains 是一家创建了几个非常流行的 IDE 的公司，它决定在2010年为自己的开发创建一种编程语言 Kotlin。它的目标是在消除 Java 的一些弱点的同时更加简洁和表达。以前所有 IDE 的代码都是用 Java 编写的，所以他们需要一种与 Java 高度互操作的语言，并编译为 Java 字节码。他们还想让 Java 开发者很容易地跳转到 Kotlin。，Jetbrains 想要建立一个更好的 Java。 

在重读 “Effective Java” 时，我发现很多这些项目对于 Kotlin 来说都不是必需的，所以本博文系列关于这本书的内容如何影响 Kotlin 设计的想法诞生了。

## 1. Kotlin 拥有默认值不再需要构建器

当 Java 中的构造函数有很多可选参数时，代码就会变得冗长，难以阅读和容易出错。 为了帮助实现这一点，Effective Java 的第2项描述了如何有效地使用 [Builder 模式](https://en.wikipedia.org/wiki/Builder_pattern)。 构建这样一个对象需要很多代码， 例如下面代码示例中的营养成分对象。它有两个必需的参数（servingSize，servings）和四个可选参数（calories, fat, sodium, carbohydrates）：

```java
public class JavaNutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;

        // Optional parameters - initialized to default values
        private int calories      = 0;
        private int fat           = 0;
        private int carbohydrate  = 0;
        private int sodium        = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        public Builder calories(int val)
        { calories = val;      return this; }
        public Builder fat(int val)
        { fat = val;           return this; }
        public Builder carbohydrate(int val)
        { carbohydrate = val;  return this; }
        public Builder sodium(int val)
        { sodium = val;        return this; }

        public JavaNutritionFacts build() {
            return new JavaNutritionFacts(this);
        }
    }

    private JavaNutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

用 Java 实例化一个对象如下: 

```java
final JavaNutritionFacts cocaCola = new JavaNutritionFacts.Builder(240,8)
    .calories(100)
    .sodium(35)
    .carbohydrate(27)
    .build();
```

对于 Kotlin，你根本不需要使用构造器模式，因为默认参数的特性允许你为每个可选的构造函数参数定义默认值: 

```kotlin
class KotlinNutritionFacts(
        private val servingSize: Int,
        private val servings: Int,
        private val calories: Int = 0,
        private val fat: Int = 0,
        private val sodium: Int = 0,
        private val carbohydrates: Int = 0)
```

在 Kotlin 创建一个对象就像这样: 

```kotlin
val cocaCola = KotlinNutritionFacts(240,8,
                calories = 100,
                sodium = 35,
                carbohydrates = 27)
```

为了更好的可读性，你也可以命名所需参数 servingSize 和 servings: 

```kotlin
val cocaCola = KotlinNutritionFacts(
                servingSize = 240,
                servings = 8,
                calories = 100,
                sodium = 35,
                carbohydrates = 27)
```

和 Java 一样，这里创建的对象是不可变的。 

我们将 Java 中需要的47代码行减少到 Kotlin 的7行，从而提高了生产力。 

提示: 如果你想在 Java 中创建 KotlinNutrition 对象，你当然可以这么做，但是你必须为每个可选参数指定一个值。 幸运的是，如果添加 JvmOverloads 注解，就会生成多个构造函数。 注意，如果你想使用注解，则需要关键字 constructor: 

```kotlin
class KotlinNutritionFacts @JvmOverloads constructor(
        private val servingSize: Int,
        private val servings: Int,
        private val calories: Int = 0,
        private val fat: Int = 0,
        private val sodium: Int = 0,
        private val carbohydrates: Int = 0)
```

## 2. 容易创建单例

"Effective Java"的第3项展示了如何设计一个 Java 对象，使其像单例一样运行，这是一个只能实例化的对象。 下面的代码片段显示了一个 “monoelvistic” 的宇宙，其中只有一个 Elvis 可以存在：

```java
public class JavaElvis {

    private static JavaElvis instance;

    private JavaElvis() {}

    public static JavaElvis getInstance() {
        if (instance == null) {
            instance = new JavaElvis();
        }
        return instance;
    }

    public void leaveTheBuilding() {
    }
}
```

Kotlin 有一个[对象声明](https://kotlinlang.org/docs/reference/object-declarations.html#object-declarations)的概念，它使我们可以使用单例行为：

```kotlin
object KotlinElvis {

    fun leaveTheBuilding() {}
}
```

不需要再手动构建你的单例了！

## 3. equals（）和 hashCode（）开箱即用

一个良好的编程实践起源于函数式编程和简化代码，主要是使用不可变的值对象。 项目15提出的建议是,"除非有非常好的理由使它们变得可变，否则类别应该是不可变的 在 Java 中创建不可变的值对象是非常繁琐的，因为对于每个对象来说，你必须重写 equals ( ) 和 hashCode ( )。Joshua Bloch 用了18页的篇幅来描述如何在第8和第9项中遵守这两种方法的明确的一般合同。 例如，如果你重写 equals ( ) 你必须确保所有的反身性、对称性、传递性、一致性和 nun-nullity 性的契约都得到了履行。听起来更像是数学而不是编程。 

在 Kotlin，你只需要使用[数据类](https://kotlinlang.org/docs/reference/data-classes.html)，编译器会自动导出 equals ( )、 hashCode ( ) 等等的方法。这是可能的，因为标准功能可以机械地从你的对象的属性派生出来。 只要在你的类别前输入关键字 data ，你就完成了。 这里不需要18页的描述。 

提示: 最近，Java 的 [AutoValue](https://github.com/google/auto/tree/master/value) 开始流行起来。 该库为 Java 1.6 + 生成不可变的值类。 

## 4. 属性优于字段

```java
public class JavaPerson {

    // don't use public fields in public classes!
    public String name;
    public Integer age;
}
```

项目14建议在公共类使用访问器方法而不是公共字段。 如果你不坚持这个建议，你可能会陷入麻烦，因为这样一来，字段就可以直接访问，你就失去了封装和灵活性的所有好处。这意味着未来您将无法在不改变其公共 API 的情况下改变类的内部表示形式。例如，你不能限制一个字段的值，比如一个人的年龄。这就是为什么我们总是在 Java 中创建冗长的默认 getter 和 setter 的原因之一。 

这种最佳实践由 Kotlin 强制执行，因为它使用的属性具有自动生成的默认 getter 和 setter 而不是字段。 

```kotlin
class KotlinPerson {

    var name: String? = null
    var age: Int? = null
}
```

从语法上来说，您可以使用 person.name 或 person.age 访问这些属性。就像 Java 的公共字段一样。 您可以在不改变类 API 的情况下添加自定义 getter 和 setter: 

```kotlin
class KotlinPerson {

    var name: String? = null

    var age: Int? = null
    set(value) {
        if (value in 0..120){
            field = value
        } else{
            throw IllegalArgumentException()
        }
    }
}
```

长话短说: 有了 Kotlin 的属性，我们就可以得到更多的简洁类，并且具有更大的灵活性。 

## 5. 重写作为强制关键字而不是可选注解

1.5版本中添加了注解。最重要的是 Override，它标志着一个方法正在重写超类的方法。根据第36项，应该不断使用这个可选注解来避免恶意错误。当你认为你从一个超类重写了一个方法但是你实际上没有重写这个方法时，编译器会抛出一个错误。只要您没有忘记使用覆盖注解，此操作就会起作用。

在 Kotlin 中，重写不是可选的注解，而是强制关键字。所以这些类型的恶意错误再也没有机会了。编译器会事先提醒你。 Kotlin 坚持[让这些事情明确](https://kotlinlang.org/docs/reference/classes.html#overriding-members)。

## 结束

这是 “Effective Java” 如何影响 Kotlin 设计的第一部分。你可以在 twitter 上[关注我](https://twitter.com/lukleDev)，从我那里获取新的更新。

下次见！

 链接：[第2部分](https://medium.com/@lukleDev/how-effective-java-may-have-influenced-the-design-of-kotlin-part-2-89844d62ddf3#.o41z4tc0a)！

