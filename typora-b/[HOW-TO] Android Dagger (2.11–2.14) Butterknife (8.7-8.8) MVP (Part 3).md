# [教程] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (第3部分)

> 原文 (Medium)：[[HOW-TO] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (Part 3)](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-3-ed5acf40eb19)
>
> 作者：[Vandolf Estrellado](https://proandroiddev.com/@vestrel00?source=post_header_lockup)

[TOC]

欢迎来到实现 [Dagger.Android](https://github.com/google/dagger)（2.11 / 2.12 / 2.13 / 2.14），[ButterKnife](https://github.com/JakeWharton/butterknife)（8.7 / 8.8）和 [Model-View-Presenter](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)（MVP）的完整指南。

这是三部曲系列的第二部分: 

1. 从头开始创建一个项目， 使用新 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）支持的依赖注入框架 @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22A%3A+dagger.android%22)]
2. 使用 ButterKnife（8.7 / 8.8）来替换大量的手写样板视图绑定代码。 [[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22B%3A+butterknife%22)]
3. 将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22C%3A+mvp%22)]

链接：[第1部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-1-eb0f6b970fd), [第2部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)

> 更新
>
> 本文及其 [Github 项目](https://github.com/vestrel00/android-dagger-butterknife-mvp)偶尔会更新，并与最新版本的 Dagger 2（2.11,2.12,2.13 / 2.14）和 Butterknife（8.7,8.8）兼容。
>
> Kotlin 分支现在可用：[[master-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/67)]，[[master-support-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/68)]
>
> [升级到 Android Studio 3.2.0 Canary 5（从 2.3.3）](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/70)。

## 开始之前

本文假设你已阅读本系列的第一部分和第二部分。 你仍然应该能够遵循这篇文章，也许你会发现它是有用的，而不需要阅读之前的部分。 但是，建议你先阅读第一部分和第二部分。 

## GitHub 项目

该系列的 GitHub 项目位于 [[PR]](https://github.com/vestrel00/android-dagger-butterknife-mvp)，这是专门为本指南而构建的。本指南的每个部分都对应于一个问题，该问题由单个请求（PR）关闭并按时间顺序标记。如果你是 Dagger 2 的经验丰富的开发人员，那么你可以跳过阅读本文并探索 GitHub 项目。

本文将引导你完成以下 [[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22A%3A+dagger.android%22)]。

> 注意：android-dagger-butterknife-mvp 项目是一个较小的衍生项目。此项目的主要目的之一是展示/演练大型项目体系结构的特定部分。看看下面的大型项目，了解更真实的示例如何应用 Dagger Android (2.11/2.12/2.13/2.14), Butterknife (8.7/8.8), Clean Architecture, MVP, MVVM, Kotlin, Java Swing, RxJava, RxAndroid, Retrofit 2, Jackson, AutoValue, Yelp Fusion (v3) REST API, Google Maps API, 整体仓库项目管理使用 Gradle， JUnit 4，AssertJ，Mockito 2，Robolectric 3，Espresso 2 以及 Java 最佳实践和设计模式。
>
> <https://github.com/vestrel00/business-search-app-java>

## 将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。

除此之外，现在是时候进入第3部分，该部分分为8个步骤：

1. 建立 MVP 框架。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/29)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.1-setup-mvp-framework)]
2. 将主要活动启动活动逻辑移动到导航器中。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/31)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.2-refactor-main-activity-to-navigator)]
3. 将主要活动重构为 MVP。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/32)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.3-refactor-main-activity-to-mvp)]
4. 重构示例1到 MVP。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/33)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.4-refactor-example-1-to-mvp)]
5. 重构示例2到 MVP。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/34)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.5-refactor-example-2-to-mvp)]
6. 重构示例3到 MVP。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/35)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.6-refactor-example-3-to-mvp)]
7. 重构 utils 以使用新的 MVP 模块。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/37)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.7-refactor-utils-to-use-new-mvp-modules)]
8. 清理 MVP 迁移中未使用的剩余部分。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/38)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/c.8-cleanup-unused-leftovers-from-mvp-migration)]

##  1. 建立 MVP 框架

一旦我们完成了 MVP 框架的设置，我们的 ui.common 包将如下所示。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovjhubawj307l0910sy.jpg)

我们这一步的目标是创建新的 MVP 接口和基类，而不会破坏和更改当前现有的代码。

在这一步之后会有3个重复的类; BaseFragment，BaseFragmentModule 和 BaseChildFragmentModule。

这3个重复类完全一样。这样做是为了以细粒度和封闭的方式将迁移划分为 MVP。在我们重构/移植应用程序的其余部分以使用我们的新 MVP 框架之前，common.view 包中的重复项将不会被使用。

> 注意：有几种不同的风格/方法来实现 MVP 模式。最常见/最受欢迎的 MVP 是[被动视图](https://www.google.com/search?q=mvp+passive+view&oq=mvp+pass&aqs=chrome.0.0j69i57j0l4.1623j0j7&sourceid=chrome&ie=UTF-8)，这就是我们将要采用的视图。在我们对 MVP 的改编中，片段是包含对演示者的引用的视图。该视图是 “被动” 的，因为它只是通知演示者关于 UI 事件的信息，例如点击。演示者然后对 UI 事件作出反应并执行一些逻辑;通常进行网络调用以从 [REST API](https://www.google.com/search?q=rest+api&oq=rest+api&aqs=chrome..69i57j69i65l2j69i60j69i65j69i60.806j0j9&sourceid=chrome&ie=UTF-8) 获取数据模型，然后更新视图以显示模型数据。
>
> 该活动托管一个或多个用于片段间通信的片段。活动还可以监听片段中的事件，以便导航到不同的活动，通常通过使用 Navigator 类。

首先，创建 Presenter 接口和一个抽象实现（BasePresenter），由具体的演示者进行分类。

```java
public interface Presenter {

    /**
     * Starts the presentation. This should be called in the view's (Activity or Fragment)
     * onCreate() or onViewStatedRestored() method respectively.
     *
     * @param savedInstanceState the saved instance state that contains state saved in
     *                           {@link #onSaveInstanceState(Bundle)}
     */
    void onStart(@Nullable Bundle savedInstanceState);

    /**
     * Resumes the presentation. This should be called in the view's (Activity or Fragment)
     * onResume() method.
     */
    void onResume();

    /**
     * Pauses the presentation. This should be called in the view's Activity or Fragment)
     * onPause() method.
     */
    void onPause();

     /**
     * Save the state of the presentation (if any). This should be called in the view's
     * (Activity or Fragment) onSaveInstanceState().
     *
     * @param outState the out state to save instance state
     */
    void onSaveInstanceState(Bundle outState);

    /**
     * Ends the presentation. This should be called in the view's (Activity or Fragment)
     * onDestroy() or onDestroyView() method respectively.
     */
    void onEnd();
}
```

Presenter 接口定义了自己的生命周期，它映射到特定的 Fragment 生命周期事件，如每个接口方法的 Javadoc 中所述。Presenter 经常使用 onStart 和 onSaveInstanceState 事件来使用 Bundle 保存/恢复状态。onPause，onResume 和 onDestroy 事件通常用于暂停，恢复或销毁在单独的后台线程中完成的任何工作，以避免泄漏和崩溃。根据每个项目的需求可能需要更多的生命周期事件。

> 问题：Presenter 在生命周期中看起来很像 Activity 或 Fragment。那么为什么不让活动或片段实现演示者呢？
>
> 我们的 MVP 风格是被动视图。视图由片段实现。由于视图是 “被动” 的，意味着它不包含任何逻辑，所以我们可以跳过为视图（片段）编写测试。由于我们没有测试我们的视图（活动或片段，仅在我们的案例中是片段），所以我们可以放弃使用 [Robolectric](http://robolectric.org/) 等 Android 测试框架。我们的单元测试可能仍然是 [Mockito](http://site.mockito.org/) 的纯 [JUnit 4](http://junit.org/junit4/)测试。
>
> 本文不会涉及单元测试。这是许多其他人已经谈到过的独立话题。
>
> 问题：为什么使用 on 前缀生命周期方法？例如，为什么不使用 start 代替 onStart？
>
> 我优先选择了此生命周期事件的命名约定。“Start” 听起来像调用视图，指挥演示者做某事。“OnStart” 听起来像 View 只是通知 Presenter 发生启动事件而不是发出命令。它遵循 MVP 的规则，View 只是通知 UI 事件的 Presenter。这是一个小小的，甚至是挑剔的细节。但是，为了保持一致和可读的代码库，应遵循模式和约定（无论多小）。
>
> 侦听器也应使用相同的命名约定。侦听器监听事件。 Presenter 是一个监听器，用于监听视图中的事件。
>
> 问题：我们不测试我们的活动吗？
>
> No。活动应尽可能保持无逻辑。必须将任何必须驻留在 Activity 中的逻辑委派给一个遵循 SOLID 原则的类（“演示者”，“管理者” 或 “util”），然后对其进行测试。

如果你是一名 MVP 老手，那么在这一点上你可能会有几个问题，意见和/或疑虑;

1. 除了Bundle之外，是否还有能在重建时保持数据/状态的不同方法

首先，在回答这个问题之前，需要在数据和状态之间作出区分，以避免模棱两可。 State 是描述应用程序组件实例(如活动或片段(例如滚动位置)的当前状态的数据类型。 在更广泛的意义上，数据包含应用程序管理 / 显示的信息，而不管组件状态(如用户或内容信息)。 

数据应该存储在一个单独的数据层/数据源中，例如[存储库](https://msdn.microsoft.com/en-us/library/ff649690.aspx)，其中数据存储在内存或磁盘中，并异步检索。 如果可能，应避免将数据存储在 Bundle 中。在 Bundle 中保存大型数据集可能会导致 Activity / Fragment 重建速度减慢，因为在重新创建期间 Bundle 会同步保存和加载。一个常见的场景涉及 list / recyclerview 中显示的数据对象列表。与其在 Bundle 中保存数据对象列表，不如让数据存储库保存它。 如果数据不在缓存(磁盘或内存)中，存储库将异步地提供数据。 存储库使用异步情况下使用的相同回调方法同步提供数据，如果数据处于缓存中，则立即调用。 

> 注意：这种情况下的数据源/存储库可以用 SQLite，OkHttp / Retrofit 缓存，Firebase 等实现。

另一方面，状态需要同步保存和加载，因为它通常与由活动或片段管理的用户界面（UI）的状态有关。此外，状态需要特定于正在进行重建的活动/片段。这使得单例数据源无法成为合适的解决方案（这是一种可能的解决方案，但不合适）。在向用户显示 UI 之前，保存的状态需要可用并恢复。有不同的方法来保存/恢复状态，但有什么比使用操作系统本身提供的更好的方法？我在这里谈论 Bundle。 Android 使用 Bundle 在 onSaveInstanceState 中保存状态并在 onRestoreInstanceState 中恢复状态（Bundle 也可以通过 onCreate 等生命周期方法使用）。

2. 如何保持 Presenter 的后台操作？

演示者管理的后台操作需要活动在演示者的生命周期之外，这样才能在重建中生存。取决于用于后台操作的框架，有不同的方法来实现这一点。

对于 [RxJava](https://github.com/ReactiveX/RxJava)，每个数据请求可以由单例存储库保留 [Subjects](https://blog.mindorks.com/understanding-rxjava-subject-publish-replay-behavior-and-async-subject-224d663d452f) 。subjects 将存储在 Map \<K，V> 中，其中 K 是请求唯一的字符串，V 是 Subject 实例。	例如，Presenter P（位于 F 片段中）调用 Repository R 的 get 数据方法。然后，存储库 R 将创建一个 [ReplaySubject](http://reactivex.io/RxJava/javadoc/io/reactivex/subjects/ReplaySubject.html)，Presenter P 的 Observer O 订阅该 ReplaySubject。在 ReplaySubject 完成之前，片段 F 经历重建。然后在新的 Fragment 实例中创建一个 Presenter P 的新实例，并再次调用 Repository R 的 get 数据方法。存储库 R 检测到 ReplaySubject 已经存在（可能不完整或完整），并将同一 subjects 用于 Presenter P  的 Observer o 的新实例来订阅。 一旦 Observer O 的 onComplete 被调用，Repository R 的 ReplaySubject 就会被解除引用（从 Map \<K，V> 中删除）。为了保持简单，我忽略了其他一些细节，但这是它的要点。

在 [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html) 的情况下可以使用类似的结构/过程。将使用 AsyncTask 而不是 Subjects，并使用回调接口而不是观察者。

3. 演示者不应该不必知道框架吗？那么为什么在这里使用 Android 的 Bundle 和 @Nullable？

首先让我们弄清楚为什么 Presenter 应该与框架无关。那就是 Presenter 不应该包含特定于框架的类的引用（在这种情况下为 Android）。仅包含纯 Java 类（不包括 Android 类）的演示者只需使用 JUnit 和 Mockito 进行单元测试。这允许绕过使用 Robolectric。这很重要，因为 Robolectric 测试比纯粹的 JUnit 测试慢得多，因为 Robolectric 必须模拟整个 Android 环境。

> 注意：有些人可能会认为，与框架无关的 Presenter 的另一个原因是跨不同框架的可重用性。例如，纯 Java 应用程序中使用的 Presenter 将用于 Android 和 Java Swing 应用程序中。但是，在实践中，这通常不起作用，因为框架差异（以及可能的应用差异）。尝试将 [Swing Window](http://docs.oracle.com/javase/7/docs/api/java/awt/event/WindowListener.html) 生命周期映射到 [Android Activity / Fragment](https://developer.android.com/guide/components/activities/activity-lifecycle.html) 生命周期。它可能适用于一些简单的情况，但两者是不同的兽类。

这里要理解的关键是，这不是一个全或无的情景。在 MVP 的这种改编中，演示者被允许使用2个 Android 实体 - Bundle 类和 @Nullable 注解。这些 Android 实体都不会强制使用 Robolectric。 Bundle 可以简单地用 Mockito 2 来模拟，而 @Nullable 注解根本不影响测试。

> 问题：Bundle 是 final 类，所以它不能被 Mockito 模拟。 [PowerMockito](https://github.com/powermock/powermock/wiki/Mockito) 必须在这里使用吗？
>
> No。[Mockito 2支持模拟 final 的类/方法](https://github.com/mockito/mockito/wiki/What%27s-new-in-Mockito-2#unmockable)。
>
> 注意：为简洁起见，我没有在这里列出关于状态重建的实例。欲了解更多信息，请阅读我在 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/41#issuecomment-329583039)] 中的评论。

如果你真的想让演示者免于使用 Android 实体，那么你将不得不使用类似 Guava 或 RxJava 的注解替换 Android 支持的 @Nullable 注解。然后在一个接口后面抽象 Bundle，这个接口将由使用 Presenter 的每个框架实现。 然而，正如我在上面提到的，我不认为这样做有什么意义，因为不管怎样，这些内容不能跨框架使用。 

```java
public abstract class BasePresenter<T extends MVPView> implements Presenter {

    protected final T view;

    protected BasePresenter(T view) {
        this.view = view;
    }

    @Override
    public void onStart(@Nullable Bundle savedInstanceState) {
    }

    @Override
    public void onResume() {
    }

    @Override
    public void onPause() {
    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
    }

    @Override
    public void onEnd() {
    }
}
```

BasePresenter 是一个抽象类，它实现 Presenter 并接受一个扩展 MVPView 的类型 T. 这个抽象类由我们的所有演示者分类，以获得对受保护的 T 视图的访问，并且可以选择不实现所有 Presenter 接口方法。

接下来， 创建 MVPView 接口和一个抽象的 Fragment 实现（BaseViewFragment），以便通过其他片段进行扩展。

```java
public interface MVPView {
}
```

MVPView 不过是一个用于类型安全和类型解析的空接口。

> 注意：这不能命名为 View，因为它与 Android的android.view.View 相冲突，这是 Android 应用程序中常用的一个类。通过避免这种碰撞，我们也避免了混淆。

 ```java
public abstract class BaseViewFragment<T extends Presenter> extends BaseFragment
        implements MVPView {

    @Inject
    protected T presenter;

    @Override
    public void onViewStateRestored(Bundle savedInstanceState) {
        super.onViewStateRestored(savedInstanceState);
        // Only start the presenter when the views have been bound.
        // See BaseFragment.onViewStateRestored
        presenter.onStart(savedInstanceState);
    }

    @Override
    public void onResume() {
        super.onResume();
        presenter.onResume();
    }

    @Override
    public void onPause() {
        super.onPause();
        presenter.onPause();
    }

    @CallSuper
    @Override
    public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        presenter.onSaveInstanceState(outState);
    }

    @Override
    public void onDestroyView() {
        presenter.onEnd();
        super.onDestroyView();
    }
}
 ```

BaseViewFragment 是我们 view.BaseFragment 的子类，实现了 MVPView 接口。它包含一个扩展 Presenter 的类型 T.  ` T presenter` 在这里注入，以便调用它的生命周期方法。这为 T 提供者提供了免费的子类，而不必担心调用 Presenter 的生命周期方法。

> 问题：为什么有2个不同的基片段类; BaseFragment 和 BaseViewFragment？为什么不把它们合并成一个基类？
>
> 一些片段，比如我们的 MainFragment，不包含任何逻辑来保证演示者的需要。无逻辑片段扩展了 BaseFragment，因为它们不需要有 Presenter-View 对。对于具有逻辑且需要演示者来承载所述逻辑的片段，使用 BaseViewFragment。
>
> 注意：在 onViewStateRestored 中调用 Presenter.onStart 方法，以便在演示者开始之前将 Fragment 的视图绑定。这确保了如果 Presenter 调用使用绑定视图的 MVPView 方法，则不会发生 NullPointerException。
>
> 此外，在 onCreateView 中返回 null View 的碎片将导致 onViewStateRestored 不被调用。这导致 Presenter.onStart 不被调用。因此，no-UI 片段不支持 Presenter-View 对。如果需要，我们可以修改我们的代码以支持 no-UI 片段中的 Presenter-View 对。不过，我会保持原样，因为我认为不适合在 no-UI  片段中使用 Presenter-View 对。不同意请随意重构。
>
> 在 onViewStateRestored 而不是 onViewCreated 持行演示者绑定。 “为什么在 onViewStateRestored 中执行绑定” 问题以在[第2部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)中进行了讨论。

最后，我们只需将 ui.common 包中的 BaseFragment，BaseFragmentModule 和 BaseChildFragmentModule 的副本（不移动）粘贴到 ui.common.view 包。重申一下，这是为了以细粒度和封闭的方式将迁移划分为 MVP。在我们重构/移植应用程序的其余部分以使用我们的新 MVP 框架之前， common.view 包中的重复项将不会被使用。

## 2. 将主活动启动活动逻辑移动到导航器中

我们的 MainActivity 包含一些处理其他活动的逻辑。我们将把这个逻辑放到 Navigator 类中，以保持我们的活动无逻辑。此外，我们将能够在其他活动中重新使用导航器来启动其他活动，而不是再次编写相同的开始活动代码。

```java
@Singleton
public final class Navigator {

    @Inject
    Navigator() {
    }

    public void toExample1(Context context) {
        Intent intent = new Intent(context, Example1Activity.class);
        context.startActivity(intent);
    }

    public void toExample2(Context context) {
        Intent intent = new Intent(context, Example2Activity.class);
        context.startActivity(intent);
    }

    public void toExample3(Context context) {
        Intent intent = new Intent(context, Example3Activity.class);
        context.startActivity(intent);
    }
}
```

Navigator 是一个单例，包含以前在 MainActivity 中的开始活动逻辑，如下面的变更集所示。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovjo857qj30cm0c6wfo.jpg)

MainActivity 现在使用已在 BaseActivity 中注入的导航器对象，如下所示。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovjr58ruj308m02mq2w.jpg)

> 问题：如果 Activity 有额外的参数可以添加到意图中呢？
>
> 阅读这个[讨论](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/13#issuecomment-330789328)来寻找答案。
>
> 注意：如果需要，我们现在可以在我们的测试中模拟和验证导航器。活动和片段应该尽可能被动，因此不需要测试。但是，像任何规则一样，有时它们会被破坏（无论出于何种原因）。在 Activity 类中有关于启动活动的逻辑的情况下，我们可以通过 Navigator 轻松模拟和验证启动活动调用。

## 3. 将主要活动重构为 MVP

这一步只需要将 MainFragment，MainFragmentListener 和 MainFragmentModule 从 ui.main 包移动到 ui.main.view 。然后 MainFragment 导入 ui.common.view.BaseFragment 而不是 ui.common.BaseFragment。而已！我们刚刚将我们的无逻辑 MainFragment 转换为 MVP。

## 4. 重构示例1到 MVP

首先，在 ui.example_1.presenter 包中创建 Example1Presenter 接口和 Example1PresenterImpl 实现，并在 Example1PresenterModule 中提供它。

```java
public interface Example1Presenter extends Presenter {
    void onDoSomething();
}
```

Example1Presenter 扩展了我们的 Presenter 接口并声明了一个名为 onDoSomething（）的方法，当 do_something 按钮被点击时，我们的 Example1Fragment 将调用它。

```java
@PerFragment
final class Example1PresenterImpl extends BasePresenter<Example1View> implements Example1Presenter {

    private final SingletonUtil singletonUtil;
    private final PerActivityUtil perActivityUtil;
    private final PerFragmentUtil perFragmentUtil;

    @Inject
    Example1PresenterImpl(Example1View view, SingletonUtil singletonUtil,
                          PerActivityUtil perActivityUtil, PerFragmentUtil perFragmentUtil) {
        super(view);
        this.singletonUtil = singletonUtil;
        this.perActivityUtil = perActivityUtil;
        this.perFragmentUtil = perFragmentUtil;
    }

    @Override
    public void onDoSomething() {
        // Do something here. Maybe make an asynchronous call to fetch data...
        String something = singletonUtil.doSomething();
        something += "\n" + perActivityUtil.doSomething();
        something += "\n" + perFragmentUtil.doSomething();
        view.showSomething(something);
    }
}
```

Example1PresenterImpl 实现了 Example1Presenter 接口，并使用即将创建的 View1 类型的 Example1View 扩展了我们的 BasePresenter。这个类的作用域是 @PerFragment，表示在托管片段的整个生命周期（Example1Fragment）中只有该类的一个实例可用。该实现注入了 Example1Fragment 中的依赖关系; SingletonUtil，PerActivityUtil 和 PerFragmentUtil。

onDoSomething ( ) 方法包含与 Example1Fragment.onDoSomethingClicked ( ) 中完全相同的代码。然后使用 view.showSomething（稍后更多）方法显示 String。

> 注意：我们现在可以在片段之外的 onDoSomething（）方法中测试代码，并且只需在纯 Java 类中进行测试。此外，我们将能够减少 Example1Fragment 中的代码行数，并保持逻辑自由，因为所有视图都应该是！

```java
@Module
public abstract class Example1PresenterModule {
    @Binds
    @PerFragment
    abstract Example1Presenter example1Presenter(Example1PresenterImpl example1PresenterImpl);
}
```

Example1PresenterModule 使用我们的 Example1PresenterImpl 提供/绑定 Example1Presenter 的 @PerFragment 实例。

> 注意：我们使用 Dagger 的 @Binds 注解来提供具有具体类的实例的接口。有关 @Binds 和 @Provides 注解的更深入的解释，[请阅读本文](https://medium.com/@vestrel00/giannis-tsironis-this-looks-like-the-same-issue-that-mohamed-alouane-posted-in-https-github-com-a2dbdc55be2c)。

其次，将 Example1Fragment 和 Example1FragmentModule 从 ui.example_1 包移到 ui.example_1.view。然后，创建 Example1Fragment 实现 Example1View 接口。

```java
public interface Example1View extends MVPView {
    void showSomething(String something);
}
```

Example1View 扩展了我们的 MVPView 并声明了一个名为 showSomething 的方法，它被 Example1PresenterImpl 用来显示一个 String 对象。

然后我们继续重构 Example1Fragment 来扩展我们的 BaseViewFragment 并使用 Example1Presenter。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovjwzzdyj30gw0q1djt.jpg)

Example1Fragment 现在扩展了 BaseViewFragment \<Example1Presenter> 并实现了 Example1View。 SingletonUtil，PerActivityUtil 和 PerFragmentUtil 被删除。现在，onDoSomethingClicked ( ) 方法简单地调用 presenter.onDoSomething ( )，其中 Example1PresenterImpl 执行逻辑并调用此处实现的 Example1View 的 showSomething 接口方法。

最后，Example1FragmentModule 现在包含 Example1PresenterModule，并提供 @PerFragment 范围的  Example1Fragment 作为 Example1View 的实现。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovk09etsj30e909d755.jpg)

ui.example_1 包现在看起来如下所示。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovk2t64nj307p06amxc.jpg)

然后！我们的示例1现在已转换为 MVP 模式！

> 问题：为什么要将演示者和视图实体放入单独的包中？为什么不把所有的 Presenter 和 View 类放在同一个包中，以便更多的类可以是包私有的并且可以使用1个 Dagger 模块提供？由于演示者和视图之间存在1对1映射，为什么不在相同的类/接口中定义 Presenter-View  合约？
>
> 答案是偏好。定义演示者视图对时，我更喜欢具有更高的粒度级别。此外，它打破了“一半”的极大特点。它使我能够查看哪些演示者类在视图中使用，反之亦然，通过使某些类为包私有和一些公共。看看下面更大规模的真实世界的例子。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovk5ahapj308g08gjrl.jpg)

> 在上图中，我们看到 BusinessListPresenter 是公开的，因为它在视图包中被引用。我们也看到 BusinessListView 是公开的，因为它在演示程序包中被引用。其余的公共类是公开的依赖注入的目的。其他一切都是封装私有的。
>
> 这种额外的粒度级别值得怀疑，但我更喜欢它。无论如何，你可以摆脱演示者/视图包，并将所有内容放在一个功能包下。这样，只需要一个 Dagger 模块，而更多的类将是包私有的。此外，你可以在一个类/接口下声明 Presenter-View 合同，而不是单独进行。
>
> 一般人可能会同意并且更喜欢将一个功能的所有类放置在一个包下（而不是具有演示者/视图包）。如果我必须选择最 “正确” 的方法，那么我会遵循普遍的共识😏

## 5. 重构示例2到 MVP

这一步涉及到与上一步相同的过程。因此，为了简洁起见，我将跳过这一步中的更改。到此步骤结束时， ui.example_2 包将如下所示。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovk8ollpj30860b4js1.jpg)

你可以查看 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/34/files#diff-bdf78f657fdb3fee025cd5717fb24ead)] 中的完整更改。

## 6. 重构示例3到 MVP

这一步涉及到与上一步相同的过程。因此，为了简洁起见，我将跳过这一步中的更改。到此步骤结束时， ui.example_3 包将如下所示。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovkbfctwj308q0b13z6.jpg)

你可以查看  [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/35)]中的完整更改。

## 7. 重构 utils 以使用新的 MVP 模块

在这一步中，我们简单地将 ui.common.BaseFragmentModule 的导入更改为 PerFragmentUtil 中的 ui.common.view.BaseFragmentModule。我们还将 ui.common.BaseChildFragmentModule 的导入更改为 PerChildFragmentUtil 中的 ui.common.view.BaseChildFragmentModule。

## 8. 清理 MVP 迁移中未使用的剩余部分

我们通过删除3个旧的（现在未使用的）重复的基本片段类来完成; ui.common.BaseFragment，ui.common.BaseFragmentModule 和 ui.common.BaseChildFragmentModule。

是时候庆祝了！我们的代码库现在已完全迁移到 MVP 被动视图。干杯。 🍻

## 结束

在这个三部分系列的这一部分，我们将代码重新设置为 Model-View-Presenter (MVP) ，以提高可测性、可维护性和可伸缩性。 

在这一点上，我们已经完成了我们的任务，从第一部分回答我们原来的问题;

>如何使用 Dagger Android（2.11 / 2.12 / 2.13 / 2.14），Butterknife（8.7 / 8.8）和 Model-View-Presenter（MVP）创建支持 Singleton，Activity，Fragment 和 PerChildFragment  范围的 Android 应用程序？

重温[第1部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-1-eb0f6b970fd)，看看我们是如何使用支持 @Singleton，@PerActivity，@PerFragment 和@PerChildFragment 范围的新 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）依赖注入（DI）框架从头开始创建项目。

重温[第2部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)，看看我们如何使用 ButterKnife（8.7 / 8.8）来替换大量的手写样板视图绑定代码。

