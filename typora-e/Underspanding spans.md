# 详解 Spans

> 原文 (Medium) ：[Underspanding spans](https://medium.com/google-developers/underspanding-spans-1b91008b97e4)
>
> 作者 ：[Florina Muntenescu](https://medium.com/@florina.muntenescu?source=post_header_lockup)

Spans 是一个强大的概念，通过提供对 TextPaint 和 Canvas 等组件的访问，从而在字符或段落级别上使用样式文本。 我们讨论了如何使用 Spans，提供了哪些Spans开箱即用，如何	轻松创建你自己的，以及如何在之前的文章中测试它们。

[Spantastic text styling with Spans | medium.com](https://medium.com/google-developers/spantastic-text-styling-with-spans-17b0c16b4568)

让我们看看在使用文本时，可以使用哪些API来确保特定用例的最佳性能。 我们将探索更多关于 Spans 的内幕以及框架如何使用它们。 最后，我们将看到我们如何在同一进程或跨进程传递 Spans，并在此基础上，决定创建自己的自定义 Spans 时需要注意哪些类型的警告。

### 幕后：Spans 如何工作

Android 框架处理文本样式，并在几个类中使用 Spans : TextView，EditText，布局类(Layout，StaticLayout，DynamicLayout) 和 TextLine (在布局中使用的一个私有类) ，它依赖于几个参数:

- 文本类型: 可选的、可编辑的或不可选择的
- [BufferType](https://developer.android.com/reference/android/widget/TextView.BufferType.html)
- TextView 的  LayoutParams 类型
- 等等

框架检查 Spanned 对象是否包含不同框架 Spans 的实例，并触发不同的操作。

文本布局和绘制背后的逻辑是复杂的，并且遍布不同的类; 在本节中，我们只能对文本的处理方式提出一种简单的看法，而且只针对某些情况。

每次 span 发生变化时，TextView.spanChange 会检查跨度是否为 [UpdateAppearance](https://developer.android.com/reference/android/text/style/UpdateAppearance.html)，[ParagraphStyle](https://developer.android.com/reference/android/text/style/ParagraphStyle.html) 或 [CharacterStyle](https://developer.android.com/reference/android/text/style/CharacterStyle.html) 的实例，如果是，则自行失效，触发新视图的绘制。

[TextLine](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/text/TextLine.java) 类表示一行样式文本，它特别适用于扩展 CharacterStyle，[MetricAffectingSpan](https://developer.android.com/reference/android/text/style/MetricAffectingSpan.html) 和 [ReplacementSpan](https://developer.android.com/reference/android/text/style/ReplacementSpan.html) 的 spans。 这是触发 [MetricAffectingSpan.updateMeasureState](https://developer.android.com/reference/android/text/style/MetricAffectingSpan.html#updateMeasureState%28android.text.TextPaint%29) 和 [CharacterStyle.updateDrawState](https://developer.android.com/reference/android/text/style/CharacterStyle.html#updateDrawState%28android.text.TextPaint%29) 的类。

管理屏幕上可视元素中文本布局的基类是 [android.text.Layout](https://developer.android.com/reference/android/text/Layout.html)。 布局及其两个子类 [StaticLayout](https://developer.android.com/reference/android/text/StaticLayout.html) 和 [DynamicLayout](https://developer.android.com/reference/android/text/DynamicLayout.html) 检查文本上设置的 span，以计算行高度和布局边距。 除此之外，每当 DynamicLayout 中显示的 span 被更新时，布局都会检查 span 是否为 UpdateLayout span，并为受影响的文本生成新布局。

### 设置文本以达到最大性能

根据你的需要，有几种内存高效的方式在文本视图中设置文本。

### 1. 在 TextView 上设置的文本永远不会改变

如果你只是在TextView上设置文本而不更新它，你可以创建一个新的 SpannableString 或 SpannableStringBuilder 实例，设置所需的 span，然后调用 [textView.setText(spannable)](https://developer.android.com/reference/android/widget/TextView.html#setText%28java.lang.CharSequence%29)。 由于你不再处理文本，所以没有更多的性能需要改进。

### 2. 通过添加 / 移除 span 来改变文本样式

让我们考虑一下情况，即文本不会改变，但附加的 span 可以。 例如，假设每当点击一个按钮时，你希望文本中的单词变为灰色。 所以，我们需要在文本中添加一个新的 span。 要做到这一点，很有可能你会试图调用两次[textView.setText (CharSequence)](https://developer.android.com/reference/android/widget/TextView.html#setText%28java.lang.CharSequence%29) : 首先设置初始文本，然后再次点击按钮。 一个更优化的解决方案是调用 [textView.setText (CharSequence，BufferType)](https://developer.android.com/reference/android/widget/TextView.html#setText%28java.lang.CharSequence,%20android.widget.TextView.BufferType%29) ，并在按钮被单击时更新 Spannable 对象的 span。

以下是这些选项下面发生的事情:

**选项1: 调用 [textView.setText (CharSequence)](https://developer.android.com/reference/android/widget/TextView.html#setText%28java.lang.CharSequence%29) 多次ー次优**

当调用 textView.setText (CharSequence) 时，在幕后 TextView 下创建了一个你的 Spannable 作为一个 SpannedString 的副本，并将它作为 aCharSequence 保存在内存中。这样做的结果就是你的文本和 span 是不可变的。 所以，当你需要更新文本风格时，你将不得不创建一个新的 Spannable，包括文本和 span，再次调用 textView.setText，这样反过来又会创建一个新的对象副本。

**选项2: 调用 [textView.setText (CharSequence，BufferType)](https://developer.android.com/reference/android/widget/TextView.html#setText%28java.lang.CharSequence,%20android.widget.TextView.BufferType%29) ，并更新一个可以访问的对象ー最优**

当调用 textView.setText (CharSequence，BufferType) 时，BufferType 参数告诉文本视图什么类型的文本被设置: 静态(在调用 textView.setText (CharSequence) 时的默认类型)、 styleable / spannable text 或 editable (将由 EditText 使用)。

既然我们正在处理可以风格化的文本，我们可以调用:

```kotlin
textView.setText(spannableObject, BufferType.SPANNABLE)
```

在这种情况下，TextView 不再创建 SpannedString，但是它将在 Spannable.Factory 的一个成员对象的帮助下创建一个 SpannableString。 因此，由 TextView 保存的 CharSequence 副本有**可变的标记和不可变的文本**。

为了更新 span，我们首先将文本作为 Spannable，然后根据需要更新 span。

```kotlin
// if setText was called with BufferType.SPANNABLE
textView.setText(spannable, BufferType.SPANNABLE)
// the text can be cast to Spannable
val spannableText = textView.text as Spannable
// now we can set or remove spans
spannableText.setSpan(
     ForegroundColorSpan(color), 
     8, spannableText.length, 
     SPAN_INCLUSIVE_INCLUSIVE)
```

有了这个选项，我们只能创建最初的 Spannable 对象。 将持有一个副本，但是当我们需要修改它的时候，我们不需要创建任何其他对象，因为我们将直接使用 TextView 保存的 Spannable 文本。 然而，TextView 只会被告知 span 的增加 / 移除 / 重新定位。 如果你更改了 span 内部属性，你将不得不调用 invalidate ( ) 或 requestLayout ( ) ，这取决于更改的类型。 详情请参阅以下「奖赏优化小贴士」。

### 3. 文本更改(重用 TextView)

假设我们想重用一个 TextView，并且多次设置文本，就像 RecyclerView.ViewHolder。 默认情况下，独立于 BufferType 集合，TextView 创建了一个 CharSequence 对象的副本并将其保存在内存中。 这样可以确保所有的 TextView 更新都是经过深思熟虑的，而不是偶然的，因为开发人员为了其他原因而改变 CharSequence 值。

在上面的选项2中，我们通过 textView.setText (spannableObject BufferType.SPANNABLE) 设置文本时，TextView 通过使用 [Spannable.Factory](https://developer.android.com/reference/android/text/Spannable.Factory.html) 实例创建了一个新的 SpannableString。 因此，每次我们设置一个新的文本，它将创建一个新的对象。 如果你想对这个过程采取更多的控制，避免额外的对象创建实现你自己的 Spannable.Factory，重写 [newSpannable(CharSequence)](https://developer.android.com/reference/android/text/Spannable.Factory.html#newSpannable%28java.lang.CharSequence%29)，并将工厂设置为 TextView。

在我们自己的实现中，我们希望避免新对象的创建，因此我们可以只是返回 CharSequence 作为一个 Spannable。 请记住，为了做到这一点，你必须调用 textView.setText (spannableObject BufferType.SPANNABLE) ，否则源代码 CharSequence 将是一个 Spanned 的实例，它不能被投射到 Spannable，从而导致 ClassCastException。

```kotlin
val spannableFactory = object : Spannable.Factory() {
    override fun newSpannable(source: CharSequence?): Spannable {
        return source as Spannable
    }
}
```

设置 Spannable.Factory 对象一旦获得对文本视图的引用即可。 如果你正在使用 RecyclerView，那么在你第一次 inflate 你的视图的时候就这样做。

```kotlin
textView.setSpannableFactory(spannableFactory)
```

有了这个，你就可以避免每次你的 RecyclerView 将一个新项目绑定到你的 ViewHolder 时，你都在避免额外的对象创建。

为了在处理文本和 RecyclerViews 时获得更多的性能，而不是在 ViewHolder 中从字符串中创建你的 Spannable 对象，**请在将列表传递给 Adapter 之前执行此操作。** 这允许你在一个后台线程上构建 Spannable 对象，以及与列表元素一起完成的任何其他工作。 然后你的适配器可以保存一个 List\<Spannable> 的引用。

### 奖励优化小贴士

如果你只需要更改一个 span 的内部属性(例如，自定义项目符号 span 的项目符号颜色) ，你不需要再次调用 TextView.setText，而只需要调用 invalidate ( ) 或 requestLayout ( )。 当视图只需要重绘或重新测量时，再次调用 setText 将会导致不必要的逻辑被触发和对象被创建。

你需要做的是保持一个引用到你的可变 span，并且，根据你在视图中更改的属性类型，调用:

- **TextView.invalidate()** 如果你只是在改变**文字外观**，触发一个重绘，跳过重新布局
- **TextView.requestLayout()** 如果你做出改变**会影响尺寸**，所以视图可以处理**测量，布局和绘制**

假设你有自己的自定义项目符号实现，默认项目符号点颜色为红色。 无论何时按下按钮，你都希望将项目符号点的颜色更改为灰色。 实现将如下所示：

```kotlin
class MainActivity : AppCompatActivity() {
    // keeping the span as a field
    val bulletSpan = BulletPointSpan(color = Color.RED)
    override fun onCreate(savedInstanceState: Bundle?) {
        …
        val spannable = SpannableString(“Text is spantastic”)
        // setting the span to the bulletSpan field
        spannable.setSpan(
            bulletSpan, 
            0, 4, 
            Spanned.SPAN_INCLUSIVE_INCLUSIVE)
        styledText.setText(spannable)
        button.setOnClickListener( {
            // change the color of our mutable span
            bulletSpan.color = Color.GRAY
            // color won’t be changed until invalidate is called
            styledText.invalidate()
        }
    }
```

### 幕后: 传递具有跨进程和相同进程的文本

当 Spanned 对象通过帧内或进程间时，不会使用自定义跨度属性。 如果所需的样式可以通过框架 span 来实现，那么更喜欢使用多个框架 span 来实现你自己的 span。 否则，更喜欢实现扩展一些基本接口或抽象类的自定义 span。

在安卓系统中，文本可以在相同的进程(intra-process)中传递，例如通过 Intents 从一个活动传递到另一个活动，以及当文本从一个应用复制到另一个应用程序时(进程间)之间传递。

自定义 span实现不能跨过相同进程边界，因为其他进程不知道它们，也不知道如何处理它们。 框架 span 是全局对象，但是只有从 ParcelableSpan 扩展的 span 才能在内部和之间传递。 这一功能允许对框架中定义的所有 span 的所有属性进行分割和不分割。 [TextUtils.writeToParcel](https://developer.android.com/reference/android/text/TextUtils.html#writeToParcel%28java.lang.CharSequence,%20android.os.Parcel,%20int%29)  方法负责保存 span 信息到 Parcel。

例如，你可以通过相同的进程，在活动之间通过一个意图:

```kotlin
// start Activity with text with spans
val intent = Intent(this, MainActivity::class.java)
intent.putExtra(TEXT_EXTRA, mySpannableString)
startActivity(intent)
// read text with Spans
val intentCharSequence = intent.getCharSequenceExtra(TEXT_EXTRA)
```

所以，即使你在同一个进程中传递 span，只有框架 ParcelableSpans 才能通过 Intent 传递。

ParcelableSpans 还允许将文本和 span 从一个进程复制到另一个进程。 复制和粘贴文本的过程通过ClipboardService，在底层使用相同的 TextUtil.writeToParcel 方法。 因此，即使你从应用复制 span 并将它们粘贴到同一应用中，这也是一个进程间操作，并且需要分包，因为文本通过 ClipboardService。

默认情况下，任何实现 Parcelable 的类都可以从 Parcel 写入和恢复。 当在进程之间传递一个 Parcelable 对象时，唯一保证正确恢复的类是框架类。 如果尝试从 Parcel 恢复数据的进程无法构造对象，因为数据类型是在其他应用程序中定义的，则该进程将崩溃。

这里有两个重要的警告:

- 当具有跨度的文本传递时，无论是在相同的过程中，还是在进程之间，只有框架的 parcelablespan 引用被保留。 因此，自定义 span 样式不会被传播
- 你不能创造你自己的 ParcelableSpan。 为了避免由于未知数据类型导致的崩溃，框架不允许实现自定义的 ParcelableSpan，通过定义两种方法，[getSpanTypeIdInternal](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/text/ParcelableSpan.java#L39) 和 [writeToParcelInternal](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/text/ParcelableSpan.java#L47)，作为隐藏。 它们都是由 TextUtils.writeToParcel 使用的

假设你想要定义一个 BulletSpan，允许定制大小的子弹，因为现有的子弹跨定义了一个固定的半径大小为4px。 下面是你如何实现它，以及每种方式的后果是什么:

1. 创建一个扩展 BulletSpan 的 CustomBulletSpan，但也允许为子弹的大小设置一个参数。 当 span 从一个活动传递到另一个活动或者通过复制文本来传递时，附在文本上的 span 将是 BulletSpan。 这意味着，当文本被绘制时，它将有框架的默认子弹半径，而不是在 CustomBulletSpan 中设置的那个
2. 创建一个只从 LeadingMarginSpan 扩展并重新实现项目符号点功能的 CustomBulletSpan。 当 span 从一个活动传递到另一个活动或通过复制文本时，连接到文本的 span 将为LeadingMarginSpan。 这意味着当文本被绘制时，它将失去所有样式。

如果仅仅通过框架 span 就可以实现所需的样式，则倾向于应用多个框架 span 来实现你自己的。 否则，更喜欢实现扩展一些基本接口或抽象类的自定义 span。 像这样，当对象在内部或者进程间传递时，你可以避免将框架的实现应用到可编写的框架中。

通过了解 Android 如何使用 span 呈现文本，希望你可以在应用中有效且高效地使用它。 下一次你需要设置文本样式时，请根据你对该文本所做的进一步工作，决定是应该应用多个框架 span 还是创建自己的自定义 span。

在 Android 中使用文本是一个很普遍的任务，所以调用正确的 TextView.setText 方法可以帮助你减少应用的内存使用量并提高其性能。

> *非常感谢* [*Siyamed Sinir*](https://twitter.com/siyamed)*, Clara Bayarri 和* [*Nick Butcher*](https://medium.com/@crafty)*.*







