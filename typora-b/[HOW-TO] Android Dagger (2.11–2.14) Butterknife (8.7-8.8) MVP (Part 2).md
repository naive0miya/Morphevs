# [教程] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (第2部分)

> 原文 (Medium)：[[HOW-TO] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (Part 2)](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)
>
> 作者：[Vandolf Estrellado](https://proandroiddev.com/@vestrel00?source=post_header_lockup)

[TOC]

欢迎来到制作 [Dagger.Android](https://github.com/google/dagger)（2.11 / 2.12 / 2.13 / 2.14），[ButterKnife](https://github.com/JakeWharton/butterknife)（8.7 / 8.8）和 [Model-View-Presenter](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)（MVP）的完整指南。

这是三部曲系列的第二部分: 

1. 从头开始创建一个项目， 使用新 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）支持的依赖注入框架 @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22A%3A+dagger.android%22)]
2. 使用 ButterKnife（8.7 / 8.8）来替换大量的手写样板视图绑定代码。 [[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22B%3A+butterknife%22)]
3. 将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22C%3A+mvp%22)]

链接：[第1部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-1-eb0f6b970fd)，[第3部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-3-ed5acf40eb19)

>更新
>
>本文及其 [Github 项目](https://github.com/vestrel00/android-dagger-butterknife-mvp)偶尔会更新，并与最新版本的 Dagger 2（2.11,2.12,2.13 / 2.14）和 Butterknife（8.7,8.8）兼容。
>
>Kotlin 分支现在可用：[[master-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/67)]，[[master-support-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/68)]
>
>升级到 Android Studio 3.2.0 Canary 5（从 2.3.3）。[[迁移]](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/70)

## 在我们开始之前...

本文假设你已阅读本系列的第一部分。你仍然应该能够遵循这篇文章，也许你会发现它是有用的，而不需要阅读前面的部分。 但是，建议你先阅读第一部分。 

## GitHub 项目

该系列的 GitHub 项目位于 [[PR]](https://github.com/vestrel00/android-dagger-butterknife-mvp)，这是专门为本指南而构建的。本指南的每个部分都对应于一个问题，该问题由单个请求（PR）关闭并按时间顺序标记。如果你是 Dagger 2 的经验丰富的开发人员，那么你可以跳过阅读本文并探索 GitHub 项目。

本文将为您介绍以下  [[ISSUES]](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22B%3A+butterknife%22)。

> 注意：android-dagger-butterknife-mvp 项目是一个较小的衍生项目。此项目的主要目的之一是展示/演练大型项目体系结构的特定部分。看看下面的大型项目，了解更真实的示例如何应用 Dagger Android (2.11/2.12/2.13/2.14), Butterknife (8.7/8.8), Clean Architecture, MVP, MVVM, Kotlin, Java Swing, RxJava, RxAndroid, Retrofit 2, Jackson, AutoValue, Yelp Fusion (v3) REST API, Google Maps API, 整体仓库项目管理使用 Gradle， JUnit 4，AssertJ，Mockito 2，Robolectric 3，Espresso 2 以及 Java 最佳实践和设计模式。
>
> <https://github.com/vestrel00/business-search-app-java>

## 使用 Butterknife（8.7）来替换大量的手写样板视图绑定代码

所有这些都已经过去了，现在是时候进入第二部分了，它被分成了两个步骤 : 

1. 将 Butterknife（8.7）依赖添加到 Gradle 构建脚本中。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/27)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/b.1-add%E2%80%94butterknife-dependency)]
2. 迁移到 Butterknife 视图绑定。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/28)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/b.2-migrate-to-butterknife-view-binding)]

## 1. 将 Butterknife（8.7）依赖添加到 Gradle 构建脚本中

将以下内容添加到根 build.gradle。

```groovy
dependencies {
    ...
    classpath "com.jakewharton:butterknife-gradle-plugin:8.7.0"
}
```

然后紧随着 app/build.gradle。

```groovy
dependencies {
    def butterKnifeVersion = '8.7.0'
    
    annotationProcessor "com.jakewharton:butterknife-compiler:$butterKnifeVersion"
    
    compile "com.jakewharton:butterknife:$butterKnifeVersion"
} 
```

## 2. 迁移到 Butterknife 视图绑定

是时候删除我们在片段中创建的许多模板视图绑定代码; MainFragment，Example1Fragment，Example2AFragment，Example2BFragment，Example3ParentFragment 和 Example3ChildFragment。

首先，用一些 Butterknife 代码更新 BaseFragment，使我们的剩余片段具有视图绑定功能。 

```java
public abstract class BaseFragment extends Fragment implements HasFragmentInjector {
    ...
    @Nullable
    private Unbinder viewUnbinder;
  
    @SuppressWarnings("ConstantConditions")
    @Override
    public void onViewStateRestored(Bundle savedInstanceState) {
        super.onViewStateRestored(savedInstanceState);
        // No need to check if getView() is null because this lifecycle method will
        // not get called if the view returned in onCreateView, if any, is null.
        viewUnbinder = ButterKnife.bind(this, getView());
    }
    
    @Override
    public void onDestroyView() {
        // This lifecycle method still gets called even if onCreateView returns a null view.
        if (viewUnbinder != null) {
            viewUnbinder.unbind();
        }
        super.onDestroyView();
    }
    
    ...
}
```

我们只需将上面的代码添加到我们的 BaseFragment，就是这样！现在我们其余的片段都有能力使用 Butterknife 的力量。

视图绑定发生在 onViewStateRestored 中调用 Butterknife.bind（this，view），它将 BaseFragment 及其子类与视图，侦听器和资源（字符串，颜色，drawables 等）从 getView ( ) 提供的视图中获取绑定。Butterknife.bind 方法返回一个对 Unbinder 对象的引用，然后我们使用该对象在 onDestroyView ( ) 中解除绑定（unbind( )）我们的视图。解除绑定将视图引用设置为 null。这对于防止片段中的视图泄漏非常有用，因为片段的视图生命周期不同于活动。 

> 注意：有关 Butterknife 的完整指南，请阅读[用户指南](http://jakewharton.github.io/butterknife/)。

某些片段可能不提供 UI，这就是为什么我们在执行绑定之前检查视图是否为空的原因。

> 问题：为什么要在 onViewStateRestored 中执行绑定？
>
> 我们在 onViewStateRestored 中而不是在 onCreateView 或 onViewCreated 中绑定视图，在没有用户交互的情况下如果侦听器发生更改不会自动调用视图状态。如果我们在此方法之前绑定（例如onViewCreated），那么任何由 Butterknife（或没有 Butterknife）绑定的OnCheckedChangeListener（和其他这样的监听器）将在片段重新创建时被调用（因为 Android 本身保存和恢复视图的状态），这可能会产生不想要的（或想要的）副作用。请看一下这个 [gist](https://gist.github.com/vestrel00/982d585144423f728342787341fa001d) 作为具体的例子。
>
> 生命周期顺序如下（如果通过 xml 或 java 添加，或者 retain 实例为 true，则相同）：onAttach - > onCreateView - > onViewCreated - > onActivityCreated - > onViewStateRestored - > onResume。请注意，onCreate（和其他生命周期事件）被故意忽略。
>
> 这种方法的要点是，在 onViewStatedRestored 之前，由 Butterknife 绑定的视图，监听器和资源将为null。但是，这不会对项目的当前状态以及我们执行 MVP 的重构时造成任何问题。在 onViewStateRestored 之前，请小心不要使用任何使用 Butterknife 绑定的对象。此方法的另一个警告是 ListView 或 RecyclerView 的滚动位置不会自动保留。原因是 list / recycler view 适配器必须在 onViewStateRestored 中设置，该操作在 OS 尝试恢复滚动位置之后发生。滚动位置必须手动保存并恢复。
>
> 将数据绑定到 onViewStateRestored 中的视图没有任何问题。等待 onViewStateRestored 绑定数据并执行其他表示逻辑具有的优点是可以在演示期间查看视图的恢复状态。如果你必须在 onViewStateRestored 之前（例如在 onCreateView，onViewCreated 和 onActivityCreated 中）出于任何原因绑定你的视图，请执行此操作。只需注意到副作用（你甚至可以故意使用，通过设计）。
>
> 注意：在 onCreateView 中返回 null View 的片段会导致 onViewCreated 和 onViewStateRestored 不被调用。这意味着 Butterknife.bind 不会被调用，这是完全正确的，因为没有视图绑定。虽然很奇怪，但在这种情况下仍然会调用 onDestroyView。
>
> 注意：onViewStateRestored（Bundle）生命周期方法仅在 API 级别17开始可用。支持低于17乃至14的 API 级别需要使用 AppCompatActivity，支持 Fragment 和 dagger.android.support API。
>
> 看看这个 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/49)] 的迁移指南，以使用支持 API。最新的支持 API 设置可在 [[master-support](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/master-support)] 分支中找到。

接下来，让我们来看看 MainFragment 的变化。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovj649r9j30g70x9n0j.jpg)

这是很多删除的代码！ MainFragment 不再实现 View.OnClickListener。设置 onViewCreated 中3个按钮的点击侦听器已被删除。带有大量 switch 语句的丑陋的 onClick 方法也被删除。3个按钮的 OnClickListener（example_1，example_2 和 example_3）现在使用 Butterknife 的 @OnClick 方法注释进行设置。

> 注意：就像 Dagger 2一样，用 Butterknife 注解注释的类成员变量或方法不能是私有的

最后，我们来看看 Example1Fragment 的更改。其他示例片段的更改与 Example1Fragment 中的更改完全相同，因此为了简洁起见，将其从本文中排除。你可以查看 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/28/files)] 中的其他更改。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovjb4y13j30h00rpdiu.jpg)

与 MainFragment 相同的更改发生在这里，即删除点击侦听器代码和使用 @OnClick。这里唯一另外的改变是使用 @BindView 来绑定 TextView 的 someText，它取代了 onViewCreated 中的 view.findViewById 代码。

## 结束

在这个由3部分组成的系列的这一部分中，我们用 ButterKnife（8.7 / 8.8）替换了大量的手写样板视图绑定代码。

继续阅读[第3部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-3-ed5acf40eb19)，了解如何将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。

重温第一部分，看看我们是如何使用支持 @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围的新 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）依赖注入（DI）框架从头开始创建项目。

