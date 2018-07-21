# 测量、布局、绘制、可复用: 自定义视图和视图组

> 原文 (Medium)：[Measure, Layout, Draw, Repeat: Custom Views and ViewGroups](https://academy.realm.io/posts/360andev-huyen-tue-dao-measure-layout-draw-repeat-custom-views-and-viewgroups-android/)
>
> 作者：[Huyen Tue Dao](https://twitter.com/queencodemonkey)

[TOC]

有时候，你所需要的就是安卓平台的布局和小工具。有时候，你需要更多的控制设计和交互，或者一些性能帮助。定制视图和视图组是强大的工具，但是拥有强大的能力会带来极大的复杂性。 在这个 [360AnDev](https://360andev.com/) 的演讲中，为了帮助你开始，我们将首先构建一个简单的自定义视图，然后添加布局、绘图和交互。在这个过程中，我们将讨论何时和何时不去定制和谈论最佳实践 

## 引言（00:00）

我叫 Huyen Tue Dao。我是安卓开发者，已经有六七年的时间了。我目前在 Trello 的 Android 团队，我喜欢自定义视图。我想用现代的说法，我是一个自定义视图的粉丝。 它们是我最喜欢的 Android 部分之一。这很奇怪，因为这个平台有很多很棒的小工具和视图，比如按钮、文本视图、视图页面、布局、线性布局和直线布局 ，...为什么你会想要写自己的自定义视图？ 有几个很好的理由可以解释为什么 。

## 自定义视图。为什么？为什么不？ (00:37)

以下是一些你可以考虑使用自定义视图的原因: 

- 假设你正在构建一个应用程序，而组件并不是按照你想要的方式工作的。 
- 你想要创建一个新的、创新的、酷的交互或者动画或者用户界面，而且你不能使它与当前的 Android 组件一起工作。 
- 如果你有一些很酷的逻辑，并且想在应用程序的不同部分使用，那么可以使代码更可重用 。
- 为了使你的应用更加模块化，你有一个复杂的用户界面，你希望拥有良好的软件工程，良好的软件设计，并且把事情分开。
- 性能：如果你有嵌套的视图，或者如果期间你有很多视图，拖累你的性能 - 自定义视图可以帮助你。

还有一些原因可能导致你不想使用自定义视图。它们费时，困难。 使用 Android 的平台小部件和视图的好处在于，你有很多东西：功能，可访问性，不同的交互，动画，样式，以及 Android 提供的所有美妙的东西。如果你做自定义的视图，你必须自己做很多事情。 这是一个缺点，自定义视图并不总是你的解决方案。

## 安卓系统如何绘制视图 （03:24）

你将开始使用 XML 布局。这些 XML 布局必须实例化;  它们将会被 inflated，并且你在 XML 布局中指定的每个视图都将被分配并生效。最终，你的视图层次结构将与你的视图和其他一切相结合。 

一旦这个视图层次结构出现，在它最终出现在屏幕上之前必须发生三件事情：测量，布局和绘制。 每一个都是视图层次结构的遍历，每个层次都是从父母到孩，再到孩子的孩子，到孩子的孩子，孩子，孩子，孩子，一直往下。 

 安卓系统绘制视图有三个阶段，作为开发人员，在这个过程中有三个切入点。对于 “测量” 阶段，有 onMeasure ( )，对于 “布局”，有 onLayout ( )，对于 “绘制”，有 onDraw ( )：

1. 测量。从根部开始，再从父母下降到孩子，父母会为每个孩子找出一些限制，让他们知道他们能有多大。那个孩子会接受这些限制，并且想出，”考虑到父母告诉我的我能有多大，这就是我想要成为的大小“。然后它自己保留这个信息，直到后来父母准备好读取这些信息。在计算的过程中，它有自己的宽度和高度，这个孩子实际上可以测量任何它自己的孩子。同样， 它是一个递归过程。
2. 测量之后就是布局。布局是指父母给孩子安排位置; 它会在屏幕上找到最终的位置和大小。
3.  最后，绘制。一旦所有的东西都被精确地测量和定位，他们就会开始绘制。 绘制从父母开始，父母绘制自己，然后基本上要求每个孩子也做同样的事情。需要注意的是，无论谁先绘制，最后在屏幕上都会显得很低。 父母会绘制自己，然后当孩子绘制时，他们会绘制在父母的上方。

## 不同的方式去自定义（07:32）

现在我们现在已经介绍了安卓系统如何绘制视图的三个不同阶段，那么我们如何能够将其转化为自定义视图呢？

有几种不同的方法可以解决这个问题。 Lucas Rocha 写了一篇很棒的文章 [Custom Layouts on Android](http://lucasr.org/2014/05/12/custom-layouts-on-android/)，在那里他探索了这些不同的方法，并且给了他们不同的名字（我加了一个名字，这个名字不如 Lucas 的好）。

两种创建自定义视图的通用方法: 扩展现有类，或扩展基本视图类 / 组类。 

视图自定义扩展： 最简单的方法是扩展现有的视图类。可以说，这是从视图中获取一些自定义逻辑或自定义行为的最简单方法。

视图平面自定义： 如果你真的喜欢冒险，你可以做 Lucas 所谓的平面自定义视图。扩展基础 U-class，而不是扩展更高级别的 Widget。通常情况下，如果你想完全自定义绘制逻辑，那么你会这样做，如果你有一个很酷的行为，你想做所有你自己想做的事情。这就是你使用平面自定义组的时候。平面自定义组也有利于性能。 如果你有一个有很多内容的布局，而不是使用一堆不同的视图来实现 UI 或者那个行为，你可以使用一个单一的视图来做同样的事情。通过这种方法，你可以减少视图树中的层级，并减少正在使用的视图数量。 

在 ViewGroup 方面，Lucas 谈到了一些不同的策略。

ViewGroup 组合：我有一组组件，按钮和小部件，我只想对它们做些什么。例如，将它们分离出来，使我的 UI 更加模块化; 也许我只是有一些特定的代码或者业务逻辑，我想应用到这些，也许我有一套控件。你可以做一个所谓的 “合成视图组”：可以把你的小部件和视图集放在一个平台布局中(例如线性布局或相对布局) ，这就是你的自定义视图组，这就是你的逻辑集中点。 

ViewGroup 自定义组合：（如果你想冒险的话!）：与其接受你的观点集，并将它们放在平台的视图组中，不如创建自己的视图组。如果你想做自己的自定义布局逻辑(例如，在一个圈中排列所有视图的视图组) ，那么这是很好的。自定义复合视图也可以对性能有好处。 你可能听说过相对布局有时不利于性能，因为它确实测量布局两次。如果你在这方面遇到了问题，并且你说,"我不想处理这个双重布局路径，测量布局路径的事情,"你可以创建一个自定义的复合视图组: 你特别有效地展示了所有的东西，并且以这种方式帮助你的性能。 

在这个演讲中，我们将大胆探讨平面的自定义或平面自定义视图，以及自定义的组合视图。

## 你需要实现什么？ （10:58）

测量，布局和绘制：这些是视图遍历的入口点。秘诀在于，你不必实现所有三个实现自定义视图和自定义视图组。根据你正在做什么和你需要什么，你只需要实现其中的一个子集即可。 

我要做一个简单的自定义视图示例。如果你正在做一个平面自定义视图，你唯一需要的东西是 onDraw。你需要在屏幕上呈现该视图，这就是你所需要的。你不需要 “布局”，因为平面自定义视图不会有孩子。你在技术上不需要 onMeasure，但我认为这是一个好主意：虽然在基础 U-class 中有默认测量逻辑，但它并不总是最好的情况。如果你想创建一个可用的酷炫自定义视图，你可以与你的队友或者全世界分享，那么实现 onMeasure 会使得这个视图更加可重用，渲染效果更好。 

那么，我今天要为你们建造什么？ 计数器！ (见图11) 。它是如此的神奇，一个奇妙的方块，里面有数字，这正是我需要的定制视图。 因为我过度思考事情(我是一个工程师) ，我有这个接口，所以这些是我将要提到的方法: 

```java

package randomlytyping.widget

 /**
  * An interface for a view implementing a tally counter.
  */
public interface TallyCounter {

    /**
     * Reset the counter.
     */
    void reset();

    /**
     * Increment the counter.
     */
    void increment();

    /**
     * @return The current count of the counter.
     */
    int getCount();

    /**
     * Set the counter value.
     */
    void setCount(int count);

}
```

## 视图构造函数（12:40）

当你正在构建自定义视图时 ，你首先需要的是构造函数。你必须将自定义视图带入生活，并重写基本 U-class 中的构造函数。有四个，但你只需要担心前两个：

1. 从代码创建新的视图 `View(Context context)`
2. 从 XML 视图创建 `View(Context context, AttributeSet attrs)`
3. 从 XML  创建具有主题属性视图  `View(Context context, AttributeSet attrs, int defStyleAttr)`
4. 从 XML  创建具有主题属性或样式资源视图  `View(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)`

这两个构造函数用于不同的情况：

1.  `View (Context context)` 将采用一个参数，上下文 - 视图所处的当前上下文，视图将如何访问资源，主题等。当你使用新运算符构建视图时，你将调用此构造函数。
2.  `View(Context context, AttributeSet attrs)` 类似于第一个，但它有一个额外的参数：它有相同的上下文，但它也有一个 AttributeSet。这个构造函数用于当你的视图从 XML  inflated  的时候，而且包含所有这些库布局属性集的属性 ，或者你可以在 XML 中指定的属性(例如填充和文本颜色和背景) ，所有这些属性最终都会出现在属性集中，这就是你的视图可以访问这些属性集的方式 。

如果你有兴趣，我的同事 [Dan](https://twitter.com/DANLEW42) 写了一篇名为 [“A Deep Dive Into Android View Constructors.”](http://blog.danlew.net/2016/07/19/a-deep-dive-into-android-view-constructors/) 的博客。

## 示例1：TallyCounterView（14:00）

这是我的 TallyCounterView 和我实现的两个构造函数：

```java
public TallyCounterView(Context context) {
    this(context, null);
}

public TallyCounterView(Context context, AttributeSet attrs) {
    super(context, attrs);
    
    // Set up points for canvas drawing.
    backgroundPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    backgroundPaint.setColor(ContextCompat.getColor(context, R.color.colorPrimary));
    linePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    linePaint.setColor(ContextCompat.getColor(context, R.color.colorAccent));
    linePaint.setStrokeWidth(1f);
    numberPaint = new TextPaint(Paint.ANTI_ALIAS_FLAG);
    numberPaint.setColor(ContextCompat.getColor(context, android.R.color.white));
    // Set the number text size to be 64sp.
    // Translate 64sp
    numberPaint.setTextSize(Math.round(64f  getResources().getDisplayMetrics().scaledDensity));
    
    // Allocate objects needed for canvas drawing here.
    backgroundRect = new RectF();
    
    // Initialize drawing measurements.
    cornerRadius = Math.round(2f  getResources().getDisplayMetrics().density);
    
    // Do initial count setup.
    setCount(0);
}

```

当你正在做自定义绘图时，你将需要很多的对象。当你在视图中进行绘制时，你正在绘制一个画布，并且你将使用绘画进行绘制。 有一个绘画对象和一个文本绘画对象：这些对象拥有可以应用于绘画操作的属性和样式（例如笔画宽度，填充颜色，填充模式）。 我们将需要一些绘画对象来完成绘制。

### TallyCounterView(Context context)

在我的构造函数中，我正在实例化它们，设置填充颜色，并且我有我的数字颜料。我的 numberPaint 是一个 TextPaint 对象（我将用它来绘制性感的小计数器），除了设置填充颜色之外，我还将设置文本大小。

请记住，一切都是相对于像素（你所做的任何绘图操作都是以像素为单位）。在进行自定义视图时，请根据 Dp 和 Sp 来考虑布局的灵活性和适应性。 当你做自定义的视图绘制时，从 Dp 和 Sp 转换为像素 - 你可以通过 Sp 中的值(这里，我希望我的文本是64 Sp)。我要乘以64，然后乘以显示器的密度。 我将通过资源访问这些数据，获取显示度量，并获得技能密度(比如比率)。 然后，我需要乘以我的64 sp，使它达到屏幕上需要的实际像素大小。 

你要做的大部分绘图操作都采用了几何对象、点和矩形，你可能在构造函数中应该这样做。 

最后，你需要的任何其他属性。 例如，我要画一个圆角矩形，我有一个角半径。 如果我想让我的角落半径为2 Dps，我会拿出显示度量，得到密度值，然后乘以我在 Dps 中的值。 然后我将 set count 调用到计数器上，以便初始化它并准备点击。 

### onDraw(Canvas canvas)

它的主题是 onDraw。我在粉红色的背景上绘制一些浅色的文字，下面有一行，文本位于画布中央：

```java
@Override
protected void onDraw(Canvas canvas) {
    // Grab canvas dimensions.
    final int canvasWidth = canvas.getWidth();
    final int canvasHeight = canvas.getHeight();
    
    // Calculate horizontal center.
    final float centerX = canvasWidth  0.5f;
    
    // Draw the background.
    backgroundRect.set(0f, 0f, canvasWidth, canvasHeight);
    canvas.drawRoundRect(backgroundRect, cornerRadius, cornerRadius, backgroundPaint);
    
    // Draw baseline.
    final float baselineY = Math.round(canvasHeight  0.6f);
    canvas.drawLine(0, baselineY, canvasWidth, baselineY, linePaint);
    
    // Draw text.
    
    // Measure the width of text to display.
    final float textWidth = numberPaint.measureText(displayedCount);
    // Figure out an x-coordinate that will center the text in the canvas.
    final float textX = Math.round(centerX - textWidth  0.5f);
    // Draw.
    canvas.drawText(displayedCount, textX, baselineY, numberPaint);
}
```

首先（为了方便起见），我获取画布宽度和画布高度。在 onDraw 中传递给你的画布就是你将要做的绘图操作。

为了得到文本的中心，我将计算我的画布的中心。 首先，我要绘制背景。 我将调用 drawRoundRect (这个方法可以在画布对象上调用)。 我将计算矩形的边界(画布的边界) ，然后将这个矩形重新定义为界限，我的角辐射，以及我早些时候实例化的颜色。 这应该能勾勒出我的背景。 

类似地，对于我正在绘制的线条，我使用了 drawLine 方法，传递一个我计算出来的坐标，然后绘制文本。 绘制文字更有趣: 对于我来说，把文字放在我的画布上，我必须知道那个文字有多宽。 我正在利用 TextPaint 测量文本方法，它将采用一个字符串，并且，基于文本颜料的属性，将让你知道该文本将会有多宽。 然后我可以计算出在哪里我需要定位我的文本以使它集中，最后，我绘制文本。 

## 反映状态的变化（18:42）

这就是我需要做的，以绘制这个绝对华丽的计数器（见幻灯片16）。 这就是你如何绘制出一个视图并让它出现在屏幕上的方法。我有一个点击按钮，那个点击按钮应该增加我的计数器... 但是它没有。只要点击那个按钮，我就会在我的计数器上调用 set count。 怎么了？ 

- 当我设置计数器时，我正在改变状态。 当我改变状态时，我没有再次绘制计数器来反映这种状态(这是你在做自定义视图时必须记住的)。 当对象 / 视图的状态发生变化时，你必须能够在绘图中反映出来。 要做到这一点，你需要使用 invalidate ( ) 方法。这个方法对安卓系统说, "这个视图是脏的," 注意到绘图不再有效，不再正确地反映了该视图的状态。它会让安卓系统知道这个需要被绘制出来 
- 安卓系统将在"未来某个时间点"再次绘制它。当你将一个视图标记为肮脏的、需要绘制的时候，任何与屏幕上的视图相交的视图都会被重新绘制。 这是很棒的，这是你如何要求你的视图重新绘制的方法。
- 另外，看看另一种情况也很有趣 invalidate ( ) 方法，它需要四个整数值: 左，右，顶，底，这使你可以指定一个特定的脏区域。 如果你看着我的计数器，任何只会改变的东西就是中间的文字。 如果我想更准确地说，我可以为那个文本找到一个包围框，然后通过它来使文本失效 。

为了让我的计数器成为一个计数器，我必须将这个 invalidate ( ) 调用添加到我的 setCount 方法中：

```java
//
// TallyCounter interface
//

@Override
public void reset() { setCount(0); }

@Override
public void increment() { setCount(count + 1); }

@Override
public void setCount(int count) {
    count = Math.min(count, MAX_COUNT);
    this.count = count;
    this.displayedCount = String.format(Locale.getDefault(), "%04d", count);
    invalidate()
}

@Override
public int getCount() { return count; }

//
// View overrides
//

@Override
protected void onDraw(Canvas canvas) {
...
```

然后，我点击我的按钮：它正在计数！

## 绘制时要记住的事情（20:29）

有些事情要记住，一些我做错了的事情，我很乐意与你分享，所以你不这样做：

- 不要在 onDraw 中分配对象。永远，永远，否则 onDraw 将被调用很多次。每次需要绘制视图时都会调用它。如果你在那里分配对象，你可以想象如果你继续在 onDraw 中分配对象，你将使用多少内存。
- Invalidate( ) 只在需要的时候调用。 绘制和无效过程是昂贵的。如果你可以通过使用与其他视图不相交的四个坐标来保存被绘制的其他视图(其他视图可以自行冷却并保持冷却) ，那么你就可以节省处理和抽出时间 。
- 文本位于基线。 文本位于基线位置。文本的基线就是文本所处的位置，但有时会有一些文字在这条线下。如果你画一个 w，那么 w 就位于基线上，但是小阀杆会突出来。当我第一次开始绘制所有的文本时，似乎总是高于它的本意——那是因为当你定位文本时，你将文本定位为(0,0)。 它不是文本的底部在(0,0)-这是那个基线 。

再次说明：你绘制像素时，请用 dp 和 sp 来考虑，以确保你的自定义视图非常好，并且可以适应不同的布局和大小。

这个布局的秘密在于计数器下面有一个按钮，你看不到它。因为它没有实现 onMeasure，计数器不知道它应该有多大。

这就是为什么我赞成实施 onMeasure。在这个布局中坚持这个视图（参见幻灯片21）意味着那个可怜的按钮这片荒芜的地方消失了。 实施 onMeasure 总是很好 - 你可以使你的视图更加灵活和 “布局能力”。

## 测量是如何进行的 （23:00）

### Child 在 XML 或 Java 定义 LayoutParams

它类似于这个视图与它的父视图之间的谈判。孩子和父母以不同的方式交流他们想要的东西。孩子通过 LayoutParams ( ) 与父视图进行通信 (它想要有多宽 / 多高) ，我们通过在 XML 中调用 `layout_width`  和  `layout_height`  来使用布局参数。 

你可以用 Java 代码来实现，但是这就是孩子如何告诉父母它想要如何布局的。无论你的布局是否正在从 XML 中 inflated，或者你是否在 Java 中手工处理所有的事情，在某个时候你都会把 setLayoutParams 调用到这个孩子上。 在稍后的某个时间点，父类将调用 getLayoutParams 来检索这些布局参数。 

### Parent 计算 MeasureSpecs 然后传递给 child.measure（）

过了一段时间，在父母 onMeasure 中，父母会查看孩子的布局参数，并计算出孩子应该有多大。 

如果还有其他孩子，父母需要根据孩子要求的可用空间和布局参数，弄清楚这个孩子会有多大。 在父的 onMeasure 中，它将解决所有这些问题，并且它编码这些约束，每个视图可以通过 MeasureSpec 值来实现。 测量值是一个整数，它编码两个事物，一个模式和一个大小。这两者的结合可以告诉一个孩子:"你最多可以是300 dp,"或者"你可以准确地说是500 dp,"或者"你的尺寸没有明确说明，你想要多大就有多大。" 

这些测量是由父母计算出来的，一个是宽度的，一个是高度的，当父母调用测量值时，它们就会通过。 

### 孩子计算宽度/高度，setMeasuredDimension（）

当 onMeasure 执行时，它会得到这两个 measures。 然后，由孩子来看这些约束，这些 measures，然后看它的内容，并弄清楚它想要多大(或者它可以是多大) ，但仍然遵循这些父母约束。 接下来，它将调用 setMeasuredDimension ( ) ，该维度存储了测量宽度和高度，即所需宽度和高度。 记住 setMeasuredDimension ( ) 在实现 onMeasure 时是强制性的。 如果你在某个时候没有调用它，那么你将会得到一个运行时异常(你将会感到悲伤)。 

### 父调用 child.layout（），最终的子尺寸/位置

一旦这个孩子已经知道它测量的宽度和高度，该父母将开始布局孩子。测量那个孩子的最后阶段是确定一个最终的大小和位置。 当它调用孩子的布局时，父母会这样做。

在某种程度上，孩子有两个维度: 测量尺寸，测量高度，或者测量宽度和测量高度。 你调用 getMeasuredWidth 和 getMeasuredHeight 来获取屏幕上的值、最终宽度和高度，然后调用 getWidth 和 getHeight 来获取这些值。 

### 视图覆盖（26:11）

当孩子从父母那里得到这些约束时，我会专注于孩子需要做什么。 

我的计算器视图。让我们先从高层考虑这个问题。 我希望计数器足够大，以适应这个文本，所以我将测量文本。我也想有填充。我希望人们使用这些填充属性，并让它在视图的左上角和右上方增加一点空间。 

```java

...

//
// View overrides
//

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    final Paint.FontMetrics fontMetrics = numberPaint.getFontMetrics();
    
    // Measure maximum possible width of text.
    final float maxTextWidth = numberPaint.measureText(MAX_COUNT_STRING);
    // Estimate maximum possible height of text.
    final float maxTextHeight = -fontMetrics.top + fontMetrics.bottom;
    
    // Add padding to maximum width calculation.
    final int desiredWidth = Math.round(maxTextWidth + getPaddingLeft() + getPaddingRight());
    
    // Add padding to maximum height calculation.
    final int desiredHeight = Math.round(maxTextHeight  2f + getPaddingTop()  + getPaddingBottom());
    
    // Reconcile size that this view wants to be with the size the parent will let it be.
    final int measureWidth = reconcileSize(desiredWidth, widthMeasureSpec);
    final int measuredHeight = reconcileSize(desiredHeight, heightMeasureSpec);
    
    // Store the final measured dimensions.
    setMeasuredDimension(measuredWidth, measuredHeight);
}

```

首先，我将测量文本，如果你经常使用文本和文字，你就会对字体度量方法变得友好。 字体度量是一个生活在许多不同地方的对象，但是每个文本绘图对象都能给你一个字体度量对象。 字体指标具有与文本相关的所有属性; ascent, decent, top, bottom，这些都是关于文本高于基线的概念，以及它可能低于基线的概念。 

我要测量一下文字。 当你和 onMeasure 一起工作的时候，你正在尝试计算内容，或者你的内容的大小，真的是很好的考虑边缘情况的上限。 在这种情况下，我不仅仅是测量我所有正确的文本，而是要测量我可以呈现的最大文本值，我可以渲染的最大值(例如这里 -- 9,999) ，然后用它创建一个字符串。 我不会像现在这样再次测量这个数字，而是要测量一下，以确保我一直都适应最大的价值。 我要测量一下，然后再用文本宽度来测量文字的宽度，使用这个测量文本方法在文本涂料，我不会测量高度。 

在安卓系统中测量文本高度有点奇怪，这是在低估它。有时候，它不起作用; 很多时候你不得不使用你最好的猜测和估计。在这里，我不会测量文本，而是使用 fontMetrics，一些字体属性。我将使用顶部，这是基线上方的最大距离，字体中的任何字形都可以是，我将使用底部，这是相反的值。 我要把这些结合起来说，这可能是文本中最大的一部分。 

接下来，对于这两个值，我将添加一些填充。我将采用我计算的那两个文本测量; 我将添加填充。然后，我将调用左侧填充，右侧填充，顶部填充和底部填充，并将其添加到测量的文本宽度和高度。

一旦我得到了这个，我需要思考一分钟，因为父母说我只能有一定的宽度和高度，但是我想做这个宽度和高度。 我们如何调和这一点呢？ 在视图类中有一个叫做 reconcileSize 的方法，并且解析大小需要一个内容大小，你想成为多大的内容，它将有一个 MeasureSpec 参数，父母告诉孩子它可以有多大，然后协调他们。 这需要你的努力，而且如果你正在执行 onMeasure 的实现，这是一个你必须使用的东西。 

### 协调大小（29:33）

我作弊了: 我实际上重新实现了它(因为解析的大小有点复杂)。 为了这次谈话的目的，我重新执行并命名为 reconcileSize : 

```java
/
  Reconcile a desired size for the view contents with a {@link android.view.View.MeasureSpec}
  constraint passed by the parent.
 
  This is a simplified version of {@link View#resolveSize(int, int)}
 
  @param contentSize Size of the view's contents.
  @param measureSpec A {@link android.view.View.MeasureSpec} passed by the parent.
  @return A size that best fits {@code contentSize} while respecting the parent's constraints.
 /
private int reconcileSize(int contentSize, int measureSpec) {
    final int mode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    switch(mode) {
        case MeasureSpec.EXACTLY:
            return specSize;
        case MeasureSpec.AT_MOST:
            if (contentSize < specSize) {
                return contentSize;
            } else {
                return specSize;
            }
        case MeasureSpec.UNSPECIFIED:
        default:
            return contentSize;
    }
}

...
```

它看着父母给你的那种模式，如果这种模式是 EXACTLY，那么方法就会说：“你将会和父母一样大小”。 如果是 AT_MOST，那么它会检查视图想要的大小，而不是父母所说的 “你可以做任何更小的事情” 的大小。如果这个模式是未知 UNSPECIFIED 的，那么方法就会说：“你可以是任何大小，你想要的大小，就是内容的大小。“

这是我测量的计数器，它看起来更好（参见幻灯片29）。 它很好地适合文本，它没有霸占所有的屏幕，而且下面有一个按钮现在已经出现了。我确定该按钮很高兴我们实施了 onMeasure，并且你也会如此。现在这个视图对它的大小有了更好的概念; 我可能可以在不同的地方使用它，而不需要它占据整个布局。 

### 你需要实现什么？ （30:36）

我们来谈谈自定义 ViewGroups。视图组也是视图，但是当你实现一个自定义视图组时，它的工作方式是不同的：视图组有子视图，而平面自定义视图可能是自私的，只用担心自己。

自定义视图组必须担心其子项 - 这是视图组的全部要点：分组和放置。你必须执行的方法才能使你的自定义视图组不同于：

- 你必须实施 onMeasure ( )，因为你必须能够调用测量你的每个孩子 。
- 你必须实现 onLayout ( )，因为你必须调用每个孩子的布局才能将其定位在屏幕上。如果你不调用子项的布局方法，他们会变成零宽度，零高度和零零的大小：他们消失了。
- 你不必实现 onDraw ( )。实际上，视图组中的默认行为是不绘制的。有一个名为 setWillNotDraw 的标志，默认情况下是 true。我不知道为什么你真的需要在视图组的顶部绘制，因为视图组的重点是分组和放置。如果这样做，请确保将 WillNotDraw 设置为 false。

## 示例＃2：自定义列表（33:09）

我要切换齿轮：我有一个包含一些项目的列表（请参见幻灯片31），并且此列表项结构在 Android 中很熟悉。图像视图，文本和更多文本。你可以使用相对布局来做到这一点，但也许你遇到了一些性能问题，或者你可能只想使用自定义视图和视图组。

除此之外，当你深入了解自定义视图和视图组时，你会感到害怕，最好从直截了当的结构 / 布局开始。 如果你有一个有静态内容的布局，或者如果你有一个布局，你可以很简单的描述。 这里我有一个图片，左边的一些文字，下面的一些文字: 这是一个非常简单的布局，它是一个良好的方式实践和使用和实施 onMeasure 和 onLayout。 

以下是我的自定义列表项目的 XML 布局，以及它的类: 

```xml
<randomlytyping.widget.SimpleListItem>
...
android:paddingStart="0dp"
android:paddingEnd="@dimen/activity_horizontal_margin"
android:background="?attr/selectableItemBackground"
android:clickable="true">

<ImageView
    android:id="@+id/icon"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_marginStart="@dimen/list_item_icon_margin_start"
    android:layout_marginEnd="@dimen/list_item_icon_margin_end"
    android:contentDescription="@null" />
    
<TextView
    android:id="@+id/title"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    style="@style/TextAppearance.AppCompat.Subhead" />
    
<TextView
    android:id="@+id/subtitle"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    style="@style/TextAppearance.AppCompat.Body1"
    android:textColor="?android:attr/textColorSecondary" />

</randomlytyping.widget.SimpleListItem>
```

```java

/
  Custom {@link ViewGroup} for displaying the example list items from the launch screen but
  without the extra measure/layout pass from the {@link android.widget.RelativeLayout}.
 /
public class SimpleListItem extends ViewGroup {
    //
    // Fields
    //
    
    @BindView(R.id.icon)
    ImageView icon;
    
    @BindView(R.id.title)
    TextView titleView;
    
    @BindView(R.id.subtitle)
    TextView subtitleView;
    
    //
    // Constructors
    //
    
    /
      Constructor.
     
      @param context The current context.
      @param attrs The attributes of the XML tag that is inflating the view.
     /
    public SimpleListItem(Context context, AttributeSet attrs) { super(context, attrs); }
    ...
}
```

我有图像视图，文本视图和另一个文本视图。它看起来像你可以做一个相对的布局，除了路线标签是 SimpleListItem，这是我的自定义 ViewGroup 扩展的名称。 这是一个组合视图组。 我在类上给我的布局有非常具体的视图。

### 自定义布局参数（33:35）

父母和视图组的 `onMeasure` 有些不同。他们更复杂，因为他们不得不担心孩子，以及他们自己 。我想与你谈谈几件不同的事情，当你在做测量视图组的时候，你可能需要注意一下。这是不需要的，但是它真的很好做: 布局参数。 有一个 ViewGroup.LayoutParams 类，它编码一个宽度和高度(这是你使用直观视图组使用的默认布局参数)。 它不是线性布局——线性布局具有宽度和高度，但它也有一些很酷的其他东西(例如重量和重力) ，这些东西不属于 LayoutParams 类。 如果你正在创建一个自定义视图组，因为你想用一个布局做一些很酷的事情(例如，你想要做一个非常有趣的算法来展示你的视图) ，可以考虑创建一个自定义的 LayoutParams 类——你可以编码用户可以设置的不同模式和属性。 

有趣的是边距。我们一直使用边距来使所有的东西都处于良好的位置。 边距不在基本的 LayoutParams 类中; 如果你使用基本视图组并且直接使用它，边距就不会工作。 但是在 ViewGroup 中有一个 MarginLayoutParams 子类，可以在自定义视图中使用边距。你必须开始让视图组知道这个布局参数将在你实例化时使用。 

有四种方法可以让你的定制 ViewGroup 使用 LayoutParams 或者你自己定制的 LayoutParams: 

1. checkLayoutParams（）验证方法，它检查特定的 ViewGroup.LayoutParams 距离是否有效。 如果我将相对布局参数传递到线性布局中，checkLayoutParams（）会说，“不能在这里使用” 。有三种方法可以生成布局参数。
2. 当 ViewGroup 获取一个没有布局参数的视图时调用 generateDefaultLayoutParams（），它可以有一些默认的布局参数。
3. generateDefaultLayoutParams（ViewGroup.LayoutParams p） - 以 ViewGroup.LayoutParams 作为参数。当视图获得一个错误的布局参数(例如将一些相对布局参数传递到线性布局中)。这个方法将给你一个机会来获取布局参数(例如无效的布局参数对象) ，然后拿出你可以使用的东西。 你可以在特定的布局参数对象中提取出宽度、高度和任何其他属性，并使用这些属性中的某些属性创建一个新的、有效的版本 
4. generateDefaultLayoutParams（AttributeSet attrs），一个带有属性集的生成布局参数。 AttributeSet 是我们的 XML inflated 的朋友，它是基于 XML 中指定的属性生成一组布局参数。

如果你想在你的自定义视图组中使用 MarginLayoutParams，或者如果你想创建你自己的很酷的布局参数，你需要实现这四个方法。

### 有用的测量方法（37:05）

实现父母的 onMeasure 可能很困难：你有这些孩子，他们想要不同的尺寸 / 被设置不同的方式，你只有这么多的空间 / 填充，你需要确保每个人的布局参数都能满足你的最大能力，同时还要遵循父母自己的宽度测量和高度测量。 这听起来很复杂，但是在视图组中有一些方法可以帮助你。 

1）测量 Spec + Padding 有两种方法可以对父方进行宽度测量，对父方进行高度测量，观察，读取它的布局参数，并根据该视图组找出该视图组自身的约束条件，并考虑到这一点，但仍然要尽其所能给孩子那些布局参数，这就是所谓的 measureChild (...)。用来测量孩子，并把它调用到视图组中的所有孩子上。 但是，如果你想要添加边距，那么前两种方法就不会考虑到边距。 

2）测量 Spec+填充+边距 大部分是由于边距不是默认处理的。有一种叫做 measureChildWithMargins（...）的方法，该方法不仅计算和协调父母可以是什么与孩子想成为什么，而且考虑边距。它将从该孩子中提取，并将其添加到计算。

3）测量 Spec +子布局参数 你可能甚至不需要使用这个方法，但是在你想做自己的事情时很方便 ：可以使用 getChildMeasureSpec（...）。所有关于协调父母可以有多大，孩子想要多大的复杂逻辑都包含在 getChildMeasureSpec（...）。它根据视图组的内容以及孩子想要成为什么来制定一个 MeasureSpec 来帮助那些孩子。这被称为测量孩子和测量孩子的参数，但如果你想做自己的东西的话，你可以调用这种方法。

### Example 2

回到我的列表项目：这里是我的布局参数方法的限制。我想在我最简单的项目上使用边距 - 这是我必须要做的。 这是简单的，不是太多的代码，能够在这四种不同的情况下生成边界布局参数的对象：

```java

/
  Validates if a set of layout parameters is valid for a child this ViewGroup.
 /
@Override
protected boolean checkLayoutParams(ViewGroup.LayoutParams p) {
    return p instanceof MarginLayoutParams;
}

/
  @return A set of default layout parameters when given a child with no layout parameters.
 /
@Override
protected LayoutParams generateDefaultLayoutParams() {
    return new MarginLayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}

/
  @return A set of layout parameters created from attributes passed in XML.
 /
@Override
public LayoutParams generateLayoutParams(AttributeSet attrs) {
    return new MarginLayoutParams(getContext(), attrs);
}

/
  Called when {@link #checkLayoutParams(LayoutParams)} fails.
 
  @return A set of valid layout parameters for this ViewGroup that copies appropriate/valid
  attributes from the supplied, not-so-good-parameters.
 /
@Override
protected LayoutParams generateLayoutParams(ViewGroup.LayoutParams p) {
    return generateDefaultLayoutParams();
}
```

它的主旨是 onMeasure。我有一个图标，一个标题，一个副标题：我将要测量它们，并使用具有边距的孩子来计算这些边距。正如你所看到的，这并不算太坏：

```java

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

    // Measure icon.
    measureChildWithMargins(icon, widthMeasureSpec, 0, heightMeasureSpec, 0);
    
    // Figure out how much width the icon used.
    MarginLayoutParams lp = (MarginLayoutParams) icon.getLayoutParams();
    int widthUsed = icon.getMeasuredWidth() + lp.leftMargin + lp.rightMargin;
    
    int heightUsed = 0;
    
    // Measure title
    measureChildWithMargins(
        titleView,
        // Pass width constraints and width already used.
        widthMeasureSpec, widthUsed,
        // Pass height constraints and height already used.
        heightMeasureSpec, heightUsed
    );
    
    // Measure the Subtitle.
    measureChildWithMargin(
        subtitleView,
        // Pass width constraints and width already used.
        widthMeasureSpec, widthUsed,
        // Pass height constraints and height already used.
        heightMeasureSpec, titleView.getMeasuredHeight());

    // Calculate this view's measured width and height.

...
```

我想要做的第一件事是，因为图标，我的图像视图坐在这里，标题和副标题坐在旁边，这个图标在测量方面是第一位的。我将首先测量该图标。当你调用 measureChild 时，或者你使用边距调用 measure 孩子时，你会传入视图和父级的 measures，其他整数值。一个整数值叫做 widthUsed，另一个叫做 heightUsed。当你测量你的孩子时，你会开始占用空间，父母就会有越来越少的空间去充分利用，widthUsed 和 heightUsed 是一种方式跟踪和提供信息给测量孩子和测量孩子边距的方法，它可以把这些考虑在内。 

图标在所有空间中都是自由的; 它通过(0,0)的宽度使用和高度使用和通过父母的宽度和高度 measuresrespec。 一旦我们测量了这个图标，我们就想知道它使用了多少宽度。 我们将调用 getMeasuredWidth，获取图标的宽度并添加到它的边缘。 测量宽度考虑了内容和填充。 现在我正在左边和右边的空白处添加，看看图标在这个布局中需要多少空间，包括它本身和它的边距。 

现在我不得不量度标题。 标题位于图标的右边; 它有一个小小的水平空间; 我将通过宽度使用的图标。 但是这个标题可以自由控制高度; 我将传递(0,0)的高度。 我将调用 measureChildWithMargins，我可以考虑标题的空白，并且我将拥有(一旦这样做了) ，getMeasuredWidth 和 getMeasuredHeight 标题。 最后，副标题。 可怜的副标题，它是最后一个，它只是得到剩下的东西。 我要计算图标使用的宽度，标题所使用的高度，然后我将这些输入子标题 childwithmargins 的调用中，然后，在这些完成之后，我终于有 getMeasuredWidth 和 孩子的高度、副标题的高度。 

孩子们知道，或者这个视图组的孩子知道他们想要多大。父母仍然需要计算，父亲或母亲仍然是视图，所以它仍然需要计算出它的宽度和高度是多少。 所有这些代码线将我做的所有事情都加在一起，图标的宽度和标题的宽度，计算出标题是否比副标题更大，这个标题的宽度应该是多少，并且把所有这些放到一起，形成一个宽度和高度。 

```java

@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    MarginLayoutParams layoutParams = (MarginLayoutParams) icon.getLayoutParams();
    
    // Figure out the x-coordinate and y-coordinate of the icon.
    int x = getPaddingLeft() + layoutParams.leftMargin;
    int y = getPaddingTop() + layoutParams.topMargin;
    
    // Layout the icon.
    icon.layout(x, y, x + icon.getMeasuredWidth(), y + icon.getMeasuredHeight());
    
    // Calculate the x-coordinate of the title: icon's right coordinate +
    // the icon's right margin.
    x += icon.getMeasuredWidth() + layoutParams.rightMargin;
    
    // Add in the title's left margin.
    layoutParams = (MarginLayoutParams) titleView.getLayoutParams();
    x += layoutParams.leftMargin;
    
    // Calculate the y-coordinate of the title: this ViewGroup's top padding +
    // the title's top margin
    y = getPaddingTop() + layoutParams.topMargin;
    
    // Layout the title.
    titleView.layout(x, y, x + titleView.getMeasuredWidth(), y + titleView.getMeasuredHeight());
    
    // The subtitle has the same x-coordinate as the title. So no more calculating there.
    
    // Calculate the y-coordinate of the subtitle: the title's bottom coordinate +
    // the title's bottom margin.

...
```

最后，我将称之为分辨率大小，平台上的实际解析大小，以获取测量宽度，所有这些增加了图标的宽度和高度以及标题和副标题。 我将通过所有这些来解决大小，传递父的实际宽度和高度 MeasureSpec，然后它将吐出这些，我可以通过它来设置度量维度。 现在我知道父母想要有多大了。 

这是结果（参见幻灯片39）。在一些代码行中，我得到一个漂亮的，简单的列表项。 如果我使用相对布局，我可以做得更快，但也不是太糟糕。在一个视图组中测量和布置子项可以是冗长的，它可以是长的，很多的代码，但它是直接的。其中大部分是能够阐明如何布置事情并将其转化为代码; 把它变成可重用的视图组，这些视图组是高性能的，可以做很酷的事情。

### 从这开始（47:11）

真正的无障碍专家是 Kelly Schuster。与自定义视图很酷的东西是自定义可绘制和可绘制状态。 Ryan Harter 在 [custom drawables](http://ryanharter.com/blog/2015/04/03/custom-drawables/) 方面做得很好 ，Marcos Paulo Damasceno 做了一个[很好的演讲](http://devforfun.ghost.io/magic-world-of-drawables/)。如果你有兴趣做自定义的图形工作，自定义绘图，以及制作一些在不同地方可以重复使用的东西，可以查看一下。 

我还建议探索所有不同的绘图操作。你可以用画布做很多事情。[Romain Guy](https://medium.com/@romainguy) 非常了解画布以及绘画/建筑如何运作。我不断阅读他的旧博客帖子。此外，Chuki 还介绍了如何使用滤镜和着色器 - 这些是在绘制时添加不同风格和着色的不同方法。

如果你担心性能，你可以开始寻求硬件加速并了解硬件层和显示列表。

[Eugenio Marletti](https://speakerdeck.com/takhion/custom-views) 对自定义视图和显示列表知道很多，并使事情变得高效。他在 [Clue](https://www.helloclue.com/) 团队工作，这是一个自定义视图工作的应用程序（如果你想要进入大量的自定义视图工作，那么这个应用程序是非常棒的）。

我强烈建议你开始使用自定义视图。 它将教你如何使用 Android 平台，如何使用视图和布局，并且它会给你机会在平台上做一些很酷很有趣的事情。 我可能会在不必要的情况下使用它们，但是我鼓励你尝试一下，因为这是一个很酷的方式来探索 Android 的可能性。 

