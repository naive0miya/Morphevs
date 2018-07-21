# “Effective Java”如何影响了 Kotlin 的设计ー第3部分

> 原文 (Medium)：[How “Effective Java” may have influenced the design of Kotlin — Part 3](https://medium.com/@lukleDev/how-effective-java-may-have-influenced-the-design-of-kotlin-part-3-7a01c9627e86)
>
> 作者：[Lukas Lechner](https://medium.com/@lukleDev?source=post_header_lockup)

[TOC]

你好，这个新帖子！这是我的博客文章系列的第三部分，内容是 “Effective Java 如何影响 Kotlin 的设计”。我在大约半年前写了[第1部分](https://hackernoon.com/how-effective-java-may-have-influenced-the-design-of-kotlin-part-1-45fd64c2f974)和[第2部分](https://medium.com/@lukleDev/how-effective-java-may-have-influenced-the-design-of-kotlin-part-2-89844d62ddf3)，并认为我找出了 Effective  Java 如何有效影响编程语言 Kotlin 的方方面面。 

几个月前，我买了和阅读了一本书，在我看来，Kotlin 最好的书：[Kotlin in Action](http://amzn.to/2v10gHA)。它由 [Dmitry Jemerov](https://twitter.com/intelliyole) 和 [Svetlana Isakova](https://twitter.com/sveta_isakova) 撰写，他们是在 JetBrains 工作的 Kotlin 核心开发人员。他们肯定知道他们在说什么。如果你想提高你的 Kotlin 知识到一个新的水平，我绝对推荐你阅读 Kotlin in Action📘！

在阅读本书时，我发现了新的语言特性和设计选择，这些特性和设计选择也可能受到 Effective Java 的影响。

## 11. 组合优于继承

“组合优于继承” 是软件开发领域中经常提到的原则。在第16项中，Effective Java 演示了如何用 Java 实现组合。在这本书中，显示了一个 InstrumentedHashSet 的例子。它应该计算使用 addCount 作为计数器将元素添加到集合中的次数。

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
	// The number of attempted element insertions
	private int addCount = 0;

	public InstrumentedHashSet() {
	}

	public InstrumentedHashSet(int initCap, float loadFactor) {
		super(initCap, loadFactor);
	}

	@Override
	public boolean add(E e) {
		addCount++;
		return super.add(e);
	}

	@Override
	public boolean addAll(Collection<? extends E> c) {
		addCount += c.size();
		return super.addAll(c);
	}

	public int getAddCount() {
		return addCount;
	}

	public static void main(String[] args) {
		InstrumentedHashSet<String> s = new InstrumentedHashSet<String>();
		s.addAll(Arrays.asList("Snap", "Crackle", "Pop"));
		System.out.println(s.getAddCount());
		// Prints 6 instead of 3
	}
}
```

但是，这段代码不正确，因为它输出的结果是6而不是3。这是因为 HashSet 实现在 addAll ( ) 中内部调用 add ( ) 方法三次。我们在这里看到的是，随着继承，我们总是依赖于超类。如果超类在未来发生变化，则子类的实现可能不再正常工作。在超类不是为了继承而设计的情况下尤其如此（参见项目17）。继承导致脆弱的实现。

我们可以通过创建一个 “转发类” 来解决这个问题。

```java
public class ForwardingSet<E> implements Set<E> {
	private final Set<E> s;

	public ForwardingSet(Set<E> s) { this.s = s; }

	public void clear() { s.clear(); }
	public boolean contains(Object o) { return s.contains(o); }
	public boolean isEmpty() { return s.isEmpty(); }
	public int size() { return s.size(); }
	public Iterator<E> iterator() { return s.iterator(); }

	public boolean add(E e) { return s.add(e);}
	public boolean remove(Object o) { return s.remove(o); }
	public boolean containsAll(Collection<?> c) { return s.containsAll(c);}
	public boolean addAll(Collection<? extends E> c) {return s.addAll(c);}
	public boolean removeAll(Collection<?> c) { return s.removeAll(c); }
	public boolean retainAll(Collection<?> c) { return s.retainAll(c); }
	
  	public Object[] toArray() { return s.toArray(); }
	public <T> T[] toArray(T[] a) { return s.toArray(a); }

	@Override public boolean equals(Object o) {return s.equals(o); }
	@Override public int hashCode() { return s.hashCode(); }
	@Override public String toString() { return s.toString(); }
}
```

ForwardingSet 中保存一个 Set \<E> 类型的实例，并且委托调用此实例。这种模式被称为[委托模式](https://en.wikipedia.org/wiki/Delegation_pattern)。通过增加 addCount 来执行检测的类必须重写相关的方法：

```java
public class InstrumentedSet<E> extends ForwardingSet<E> {
	private int addCount = 0;

	public InstrumentedSet(Set<E> s) {
		super(s);
	}

	@Override
	public boolean add(E e) {
		addCount++;
		return super.add(e);
	}

	@Override
	public boolean addAll(Collection<? extends E> c) {
		addCount += c.size();
		return super.addAll(c);
	}

	public int getAddCount() {
		return addCount;
	}

	public static void main(String[] args) {
		InstrumentedSet<String> s = new InstrumentedSet<String>(
				new HashSet<String>());
		s.addAll(Arrays.asList("Snap", "Crackle", "Pop"));
		System.out.println(s.getAddCount());
		// Correctly prints 3
	}
}
```

InstrumentedSet 按预期工作，独立于 Set 超类的实现细节！在 Java 中的一个缺点，就像我们在转发类 ForwardingSet 中看到的那样，一个缺点就是需要编写大量的模板代码。这绝对是 Kotlin 编译器能为我们做的工作！

Kotlin 组合

在 Kotlin 中，InstrumentedSet 可以简化为如下所示：

```kotlin
class InstrumentedSet<E>(val set: MutableSet<E>) : MutableSet<E> by set {
    var addCount = 0
    
    override fun add(element: E): Boolean {
        addCount++
        return set.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        addCount += elements.size
        return set.addAll(elements)
    }
}

fun main(args: Array<String>) {
    val s = InstrumentedSet(HashSet<String>())
    s.addAll(Arrays.asList("Snap", "Crackle", "Pop"))
    println(s.addCount)
    // Correctly prints 3
}
```

我们不需要创建一个转发类。我们可以使用 by 关键字将方法调用委托给实例 。然后编译器将生成所有将要设置的  InstrumentedSet 方法。此功能称为[类委托](https://kotlinlang.org/docs/reference/delegation.html)。

提示：如果你想了解编译器在底层生成的内容，可以使用 IntelliJ IDEA 或 Android Studio 的 “Kotlin Byte Code Viewer”。此查看器可以显示 Kotlin 文件的 Java 字节代码，并且还可以将此字节代码反编译为纯 Java 代码。当我们在 InstrumentedSet.kt 上执行此操作时，我们会看到生成的转发方法，如 clear ( ) 或 remove ( )：

```java
public final class InstrumentedSet implements Set, KMutableSet {
   private int addCount;
   @NotNull
   private final Set set;

   public final int getAddCount() {
      return this.addCount;
   }

   public final void setAddCount(int var1) {
      this.addCount = var1;
   }

   public boolean add(Object element) {
      int var2 = this.addCount++;
      return this.set.add(element);
   }

   public boolean addAll(@NotNull Collection elements) {
      Intrinsics.checkParameterIsNotNull(elements, "elements");
      this.addCount += elements.size();
      return this.set.addAll(elements);
   }

   @NotNull
   public final Set getSet() {
      return this.set;
   }

   public InstrumentedSet(@NotNull Set set) {
      Intrinsics.checkParameterIsNotNull(set, "set");
      super();
      this.set = set;
   }

   public void clear() {
      this.set.clear();
   }

   public boolean remove(Object element) {
      return this.set.remove(element);
   }

   ...

}
```

简而言之，通常最好使用组合而不是继承。我们可以通过编写大量样板代码来在 Java 中实现这一点。另一方面，Kotlin 通过类委托语言特征和委托模式实现组合，因此不需要写样板代码。

## 12. 没有原始和引用类型的混淆

在 Java 中，原始类型（例如 int，long）和引用类型（例如 String，Integer，Long）之间存在区别。他们之间基本上有三个不同点 : 

1. 原始类型包含值。 引用类型保存对象的内存位置的引用。如果要检查两个对象是否具有相同的值，那么在引用类型上使用 `==` 运算符是一个非常常见的错误。例如，new Integer（128）== new Integer（128）结果是 `false` ，因为你正在检查两个对象是否指向相同的内存地址，而它们没有，因为它们是两个不同的对象。正确的解决方案是使用 `.equals（）` 而不是 `==` 。另一方面，对于原始类型，评估等价值的正确方法是使用 `==`。
2. 原始类型只有函数值，而引用类型也具有非函数值：null。因此，通过使用引用类型，可以使用引用类型在运行时生成一个 NULLPointerException。
3. 原始类型更有时间和空间效率。通过在循环中使用引用类型而不是原始类型，你可以很容易地得到性能问题 : 

```java
public static void main(String[] args){
    Long sum = 0L;
    for(long i = 0; i <= Integer.MAX_VALUE; i++) {
      sum += i;
    }
    System.out.println(sum);
}
```

前面的示例取自 Effective Java 的第5项，其中指出 “避免创建不必要的对象”。在我的机器上执行这段代码需要大约25秒的时间。为什么需要这么长时间？原因很简单。通过使用引用类型 Long 作为 sum 的类型，每次执行 sum + = i 时，都会创建一个 Long 类型的新对象。对原始类型 long 的一个小改动将大大提高性能。在这个改变之后，代码在大约2.5秒内被执行。

Java 1.5 版包含自动装箱功能（原理是原始类型创建引用类型）和自动拆箱（原理是引用类型创建原始类型）。 这样就可以更容易地使用这两种类型。 问题在于，Java 在执行这些功能时并不明显，而这些特性常常让开发人员感到困惑，并导致令人讨厌的错误。 

长话短说：在 Java 中，你必须非常谨慎地使用哪种类型以及何时执行某种类型的装箱。 作为一般性的指南 ，你应始终使用基本类型，除非你被迫使用引用类型（例如泛型类的集合和类型参数）。第49项还建议 “原始类型优于装箱原始类型”。

Kotlin 中的类型

那么 Kotlin 的类型呢？ 原始类型和参考类型之间有区别吗？  简单的答案：No！没有不同类型的例如 Long 或 Integer，只有 Int 和 Long。

你可能会问自己，Kotlin 中的类型是相当原始类型还是相当引用类型，这里的答案是：取决于！

我们知道 Kotlin 编译器生成 Java 字节码。 因此，最终编译器是生成正确类型的编译器。 编译器很聪明，希望创建高效的字节码。 因此，大多数时候，原始类型是生成的，但并不总是这样。 

当存储在集合中并使用泛型类中的类型参数时，编译器会生成引用类型，因为这样做是没有办法的。有趣的是，当你使用 Long 这样的可空格类型时，还会产生引用类型？ . 如果你仔细想想，这背后的原因很简单。 原始类型不能保持空，而引用类型可以。 因此，当你声明: var sum: Long? = 0L 时，我们上面的代码示例在 Kotlin 也会有性能问题。	

对于平等检查，你可以始终在 Kotlin 中使用 `==` 来比较值。如果你想检查两个引用是否指向内存中的同一个对象，在 Kotlin 中称为 “结构相等”，则使用 `===`。与 Java 相比，这是一种更简单，更不容易出错的方式。

总而言之，由于 Kotlin 中的原始类型和引用类型在语言本身中没有区别，因此不可能出现  Effective Java 建议避免的一些错误。但是，你必须知道，Kotlin 从字节代码中的可空类型生成引用类型，这可能会导致性能问题。

## 13. 类之外的函数

在 Java 中，每个函数都需要在类中定义。辅助功能不包含任何状态但只包含公共静态函数的类被称为"Utility Classes"(例如 java.util.array，一种帮助例如排序数组的实用工具类)。 

在第4项中提议通过实现私有构造函数来强制非实例化这些类，以便客户端不能创建这样的 helper 类实例: 

```java
public class Arrays {

    ...

    // Suppresses default constructor, ensuring non-instantiability.
    private Arrays() {}
  
    ...
    
    public static void sort(int[] a) {
        DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
    }
    
    ...

}
```

在 Kotlin 中，函数不需要与类关联。类之外的函数称为 “顶级函数 ”，非常适合“ 实用性函数”。 Array.java 文件看起来像这样：

```kotlin
fun sort(a: IntArray) {
    DualPivotQuicksort.sort(a, 0, a.size - 1, null, 0, 0)
}

...
```

使用顶级函数，我们不需要使任何类不可实例化，因为这些函数不是任何类的一部分。 

## 14. 默认静态成员类

Effective Java 在第22项中建议 “静态成员类优于非静态成员类”。非静态成员类对它们的封闭类有一个隐含的引用，因此造成很大的麻烦（例如，它们使用大量内存空间，是内存泄漏的常见原因）。成员类在 Java 中默认是非静态的。你必须明确地提供 static 关键字来使用它们（...猜测为什么？）。

Kotlin 中的嵌套类默认不包含隐式引用：

```kotlin
class Outer {
    private val bar: Int = 1
    class Nested {
        fun foo() = 2
    }
}

val demo = Outer.Nested().foo() // == 2
```

你必须明确地添加 inner 关键字，以获得对封闭类的隐含引用: 

```kotlin
class Outer {
    private val bar: Int = 1
    inner class Inner {
        fun foo() = bar
    }
}

val demo = Outer().Inner().foo() // == 1
```

Kotlin 的设计者遵循了第15项的建议：“静态成员类优于非静态成员类”并创建成员类，与 Java 相比，默认情况下为静态成员类。

## 15. 检查输入的好方法

Effective Java 中的第7章是关于方法的。第38项建议 “检查参数的有效性”，如以下 Java 代码示例所示：

```java
 / returns the average value of an integer list, the list
    must not be null and has to contain at least 2 elements /
    public static double getAverage(List<Integer> list) {
        if (list == null) {
            throw new IllegalArgumentException("list must not be null");
        } else if (list.size() < 2) {
            throw new IllegalArgumentException("list must contain at least 2 elements");
        }

        // Do the computation and return the average
    }
```

Kotlin 团队为其[标准库](https://github.com/JetBrains/kotlin/tree/master/libraries/stdlib)中的参数检查提供了很好的实用性函数 ，更精确地说，是在文件 [Preconditions.kt](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/util/Preconditions.kt) 中。我们来看看 require ( ) 函数：

```kotlin
/
  Throws an [IllegalArgumentException] with the result of calling [lazyMessage] if the [value] is false.
 
  @sample samples.misc.Preconditions.failRequireWithLazyMessage
 /
@kotlin.internal.InlineOnly
public inline fun require(value: Boolean, lazyMessage: () -> Any): Unit {
    if (!value) {
        val message = lazyMessage()
        throw IllegalArgumentException(message.toString())
    }
}
```

 require ( ) 以参数检查的布尔结果作为第一个参数和 alazyMessage 函数类型作为第二个参数。只有在第一个参数是 false 的情况下， lazyMessage 的代码才会被执行（这就是为什么它被称为 lazy），并且 lazyMessage 的 ToString ( ) 方法将抛出 IllegalArgumentException。对于如何使用  require ( ) ，这就是验证代码的示例：

```kotlin
/ returns the average value of an integer list, the list
    must not be null and has to contain at least 2 elements /
fun getAverage(list: List<Int>): Double {

    require(list.size >= 2){
        "list must contain at least 2 elements"
    }

    // Do the computation and return the average
}
```

首先要注意的是，不再有 null 检查。 这是因为该方法只允许其客户端传递 List \<Int> 的不可为空的实例。其次，代码更具可读性。我们有一个更富有表现力的 require ( ) 方法调用，而不是 if 条件。第三，我们不需要手动抛出 IllegalArgumentException，因为 require ( ) 正在为我们做这件事。

总结：Kotlin 知道输入验证的重要性，因此在其标准库中提供了很好的实用功能。 

就是这样！ 谢谢你的阅读！ 如果你喜欢这篇文章，也许学到了一两件事，如果你给我一些掌声👏 ，我会非常感激。如果你不想错过我的文章，请随时在 twitter 上[关注我](https://twitter.com/lukleDev)😃。我经常会发布我正在学习的新东西（主要是 Kotlin 和 Android 相关的）。祝你有愉快的一天！ 🌴 

