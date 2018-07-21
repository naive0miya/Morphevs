# Android 任务和后台堆栈回顾

> 原文 (Medium)：[Android Task and Back Stack Review](https://medium.com/mindorks/android-task-and-back-stack-review-5017f2c18196)
>
> 作者：[Janishar Ali](https://medium.com/@janishar.ali)

[TOC]

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron66dbd6j31a70hsdii.jpg)

安卓活动是我们希望用户浏览的屏幕的逻辑构造。 每个活动与其他活动的关系对于良好的用户体验是非常重要的。 这种关系的设计应该着眼于从用户的角度来制定一个不费力的模式形成策略。 

实现上述目标最重要的一个方面是设计一个正确的向前和向后的导航。 在管理屏幕流程方面，Android 设计已经很好地提供了一个平滑的用户体验。 

> Android 开发者指南说：每个活动都应围绕用户可以执行的特定类型的操作来设计，并且可以开始其他的活动 

我们来回顾一下 Android 上面提到的缺省实现

> 将用户交互的序列定义为任务。 任务是用户在执行某项工作时进行交互的活动集合 

一个任务保存活动，安排在一个名为 backstack 的堆栈中。 堆栈具有 LIFO 结构，并按照开放顺序存储活动。 堆栈中的活动从来没有重新安排过。 后台堆栈的导航是通过后退按钮来完成的。 

## Task 和 Back Stack 的默认行为。

1. 应用程序启动器创建一个新的 Task，主 Activity 被创建并放置在后台堆栈的根目录（它还有另外一个角色，我们将在后面讨论）。
2. 当当前活动启动另一个活动时，新活动将被推送到堆栈的顶部并聚焦
3. 前一个活动在这个新的活动之下移动在后台的堆栈，并停止。 系统保留了这个活动的用户界面的当前状态，比如表单中的文本、滚动位置等等 
4. 随后的新活动继续堆积在后台的堆栈 
5. 当按下后退按钮时，当前的活动将从后面堆栈的顶部弹出。 这会破坏活动，而前一个活动会随着状态的恢复而恢复 
6. 然后，返回按钮继续弹出当前的活动和恢复以前的活动。 当最后一个活动从后台堆栈中移除时，任务将终止到任务创建前最后运行的屏幕 （在我们的例子中是启动器屏幕）。
7. 当后台堆栈变空时，任务不复存在。
8. 意图调用的不同应用程序的活动放在同一个任务中。

## 任务有一些重要的属性：

1. 当创建一个新任务或者按下 Home 按钮时，它会进入后台 
2. 然后通过点击启动器图标（这是我们之前提到的启动器图标的另一个角色）或从最近的屏幕中选择任务 
3. 当多个任务处于后台或者用户长时间离开任务时，系统为了恢复内存清除了除根活动以外的所有活动的任务。 当用户再次返回任务时，只会恢复根活动 （这个行为可以被重写，我们将在后面看到）。

## 一个重要的结果：

我们可以从上面的文字中得出一个非常重要的结果。 如果一个活动是从多个活动开始的，那么创建一个活动的新实例并将其推送到堆栈中，而不是将前一个活动的实例带到顶部。 在这种情况下，返回按钮显示同一个活动的实例多次与其状态在顺序是创建。 

> 这种特定的行为是不受欢迎的，本文的主要关注点是学习控制后面的栈，以便一次一个实例中存在一个活动，并与其发起者活动相关联。 我们将在下面的文本中看到如何做到这一点 

在很少的情况下，我们希望管理任务作为默认行为的一个偏差。 安卓开发者指南列出了以下一些情况: 

- 在启动新活动时，需要在新的任务中开始，而不是放在现有任务的后台堆栈中 

例如，Android 浏览器应用程序声明 web 浏览器活动应该始终在它自己的任务中打开。 这意味着，如果你的应用程序发出了打开 Android 浏览器的意图，那么它的活动就不会和你的应用程序放在同一个任务中。 相反，要么是浏览器的新任务开始，要么是浏览器已经在后台运行一个任务，那么这个任务就会被提出来处理新的意图。 

- 当启动一个新的活动时，需要从后台的堆栈带来现有的实例，如我们已经提到的 
- 当用户离开现有活动时，后台的堆栈应该清除到根活动 

为了解决上述与默认的偏差，我们提供了两种模式。

1. 使用属性为 android:launchMode 的 AndroidManifest \<activity> 标签。
2. 将意图传递给 startActivity ( ) 的标志包含在内。

启动模式及其等效的 startActivity 标志允许我们定义一个活动的新实例如何与当前任务关联，并指定在指定任务中启动的指令。 

下面几点介绍并调查了启动模式及其等效的 startActivity 标志类型(并非所有发射模式都有 startActivity 标志对应，反之亦然) : 

- **launchMode — standard**:

  这是活动的默认模式。 在这种模式下，创建了活动的一个新实例，并通过将意图路由到该任务，将其放入到任务中。 该模式中的活动可以实例化多次，每个实例可以属于不同的任务，一个任务可以有多个实例。 

- **launchMode — singleTop | flag — FLAG_ACTIVITY_SINGLE_TOP**:

  这种模式或者标志产生与标准启动模式完全相同的行为，如果新的活动不在后台堆栈中作为顶部存在，则该模式或标志与标准 launchMode 产生完全相同的行为。 如果它出现在顶部，那么它的行为就不同了。 在这种情况下，同样的 Activity 会继续调用 onNewIntent 方法。

- **launchMode — singleTask | flag — FLAG_ACTIVITY_NEW_TASK**:

  如果一个活动不存在于已经创建的任务中，那么它将在一个新的任务中启动活动。 只有一个活动的实例可以同时存在。 在这种情况下，后退按钮仍然能够将用户返回到前一个任务的活动中 。

- **launchMode — singleInstance**:

  只是系统没有在启动实例的任务中启动任何其他活动。 活动始终是任务的唯一成员。 由此启动的任何活动都会在一个单独的任务中打开 

> singleTask 和 singleInstance 的 launchMode 不适用于大多数应用程序。

- **flag — FLAG_ACTIVITY_CLEAR_TASK**:

  在活动启动之前清除与活动相关的任何现有任务。 这个活动随后成为任务的新根，旧的活动已经完成。 它只能与 FLAG_ACTIVITY_NEW_TASK 一起使用 

当通知必须通过完成现有的活动来启动应用程序时，这个特定的标志是有用的。例如，如果我们想从 Notification 回调处理程序服务启动 SplashActivity

```java
Intent openIntent = new Intent(getApplicationContext(), SplashActivity.class);
openIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
getApplicationContext().startActivity(openIntent);
```

- **flag — FLAG_ACTIVITY_CLEAR_TOP**:

  如果正在启动的活动已经在当前任务中运行，而不是启动该活动的新实例，那么它上面的所有其他活动都会被销毁(通过 onDestroy 方法) ，并通过 onNewIntent 方法将此意图传递到活动的实例(现在在顶部) 

**注意：**对于 FLAG_ACTIVITY_CLEAR_TOP，如果 launchMode 没有在 AndroidManifest 中定义，或者对 Activity 设置为“标准”，则会弹出 Activity 及其顶部，并将该 Activity 的新实例放在顶部。 所以， onNewIntent 方法不被调用。

这是不可取的，大多数时候我们想要重用活动并刷新它的视图状态，比如列表中的数据，当它到后面堆栈的顶部，而不是破坏，然后再创建它。 

为了实现这一点，我们将给定的 Activity 的 launchMode 定义为 singleTop，并使用标志 FLAG_ACTIVITY_CLEAR_TOP 调用 startActivity ( )。

```java
<activity
   android:name=".ActivityA"
   android:launchMode="singleTop"/>
```

并且

```java
Intent intent = new Intent(context, ActivityA.class);
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
startActivity(intent);
```

> 优先规则：如果当前活动添加了意图标志并且开始活动已经定义了启动模式，那么当前活动的请求得到遵守。

让我们来探讨几个与后退堆栈相关的重要 Activity 的清单属性。

1. **noHistory**:

   如果这个设置为 true，那么它会销毁当前的 Activity，并在任何其他 Activity 启动时将其从 Back 栈中移除。你不必显式调用完成此活动。它在 SplashActivity 中很有效。

2. **clearTaskOnLaunch**:

   如果设置为 true，则每当用户离开任务并返回时，返回堆栈就被清除到根 Activity。

3. **finishOnTaskLaunch**:

​      它与 clearTaskOnLaunch 类似，但在单个 Activity 上运行。

> 注意：不要在同一个应用程序中使用 startActivityForResult 作为活动，而应该像上面描述的那样使用 singleTop 和 FLAG_ACTIVITY_CLEAR_TOP 来处理子视图被删除时的父视图更新。

在运行时调查应用程序的任务和返回堆栈：

在开发阶段，如果我们已经实现了 launchMode 或者已经使用了意图标志，我们应该验证应用程序 Task 和 back stack。这可以通过活动信息的转储来完成。

**这个调查的步骤如下：**

1. 通过 Android Studio 在移动设备上构建和运行应用程序。
2. 在移动设备上浏览几个屏幕。
3. 在 Studio 的终端中运行 adb shell dumpsys 活动。
4. 这将给出一个长信息文本。搜索你的应用程序的包名称。
5. 在 ACTIVITY MANAGER ACTIVITIES 部分，我们可以找到有关任务和活动的所有信息。



感谢你阅读这篇文章。如果你觉得有用，请务必点击下面的❤推荐这篇文章。它会让别人得到这篇文章，并传播知识。