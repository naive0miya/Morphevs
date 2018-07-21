# [教程] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (第1部分)

> 原文 (Medium)：[[HOW-TO] Android Dagger (2.11–2.14) Butterknife (8.7-8.8) MVP (Part 1)](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-1-eb0f6b970fd)
>
> 作者：[Vandolf Estrellado](https://proandroiddev.com/@vestrel00?source=post_header_lockup)

[TOC]

欢迎来到制作 [Dagger.Android](https://github.com/google/dagger)（2.11 / 2.12 / 2.13 / 2.14），[ButterKnife](https://github.com/JakeWharton/butterknife)（8.7 / 8.8）和 [Model-View-Presenter](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)（MVP）的完整指南。

这是一个三部曲系列的第一部分 : 

1. 从头开始创建一个项目， 使用新 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）支持的依赖注入框架 @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22A%3A+dagger.android%22)]
2. 使用 ButterKnife（8.7 / 8.8）来替换大量的手写样板视图绑定代码。 [[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22B%3A+butterknife%22)]
3. 将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。[[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22C%3A+mvp%22)]

链接：[第2部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)，[第3部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-3-ed5acf40eb19)

> 更新
>
> 本文及其 [Github 项目](https://github.com/vestrel00/android-dagger-butterknife-mvp)偶尔会更新，并与 Dagger 2（2.11,2.12,2.13 / 2.14）和 Butterknife（8.7,8.8）的最新版本兼容。
>
> Kotlin 分支现在可用：[[master-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/67)]，[[master-support-kotlin](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/68)]
>
> 升级到 Android Studio 3.2.0 Canary 5（从 2.3.3）[[迁移]](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/70)。

## 在我们开始之前...

在我们开始之前有几件事情需要澄清。 

这是一篇指导性文章，而不是历史课。本文不会回答以下问题：

- 什么是 Dagger 2？
- Dagger 2 与 Dagger 1 有什么不同？
- 在版本2.10之前，Dagger 2项目的常见设置是什么？
- 还有其他的依赖注入框架吗？
- 什么是依赖注入？

如果你不知道上述任何问题的答案，那么 Google 就可以。这些问题已经被很多不同的人回答了很多次。为简洁起见，本指南不会涉及它们。

我将回答的唯一问题是 “在2.10版本中引入的 dagger.android 是什么？”。[Dagger 2.10](https://github.com/google/dagger/tree/dagger-2.10) 引入了 [dagger.android](https://github.com/google/dagger/tree/dagger-2.10/java/dagger/android) 框架，该框架极大地简化了 Android 中的依赖注入，并去除了以前版本中存在的大量样板依赖注入代码。

> 提示：如果你还不知道，Dagger 2可用于纯粹的 Android, 非 Android，Java 项目/模块）。

本文重点介绍使用 Dagger 2.11，其中包含一些非常有用的称为 @ContributesAndroidInjector 的东西（稍后会详细介绍）。

dagger.android 的官方 Google 用户指南位于 [[Guide]](https://google.github.io/dagger/android.html)。但是，用户指南可能很难理解，缺乏有用的示例。有几个博客介绍了新的 dagger.android 框架和一些例子，让你简单入门。然而，我还没有读到一篇试图并成功回答以下问题的文章: 

> 如何使用 Dagger Android（2.11 / 2.12 / 2.13 / 2.14），Butterknife（8.7 / 8.8）和 Model-View-Presenter（MVP）创建支持 Singleton，Activity，Fragment 和 childFragment 范围的 Android 应用程序？

读读这本指南就知道了！ 

这本指南是一本完整的指南，包括代码片段和解释。 因此，它是相当长的。然而，它会教你一切你需要知道的一个完整的 dagger.android，Butterknife，和 MVP 设置，所以泡一些咖啡（或茶），并开始阅读！

## GitHub 项目

该系列的 GitHub 项目位于 [[PR]](https://github.com/vestrel00/android-dagger-butterknife-mvp)，这是专门为本指南而构建的。本指南的每个部分都对应于一个问题，该问题由单个请求（PR）关闭并按时间顺序标记。如果你是 Dagger 2 的经验丰富的开发人员，那么你可以跳过阅读本文并探索 GitHub 项目。

本文将为你介绍以下 [[ISSUES](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues?q=is%3Aissue+is%3Aclosed+sort%3Acreated-asc+label%3A%22A%3A+dagger.android%22)]。

> 注意：android-dagger-butterknife-mvp 项目是一个较小的衍生项目。此项目的主要目的之一是展示/演练大型项目体系结构的特定部分。看看下面的大型项目，了解更真实的示例如何应用 Dagger Android (2.11/2.12/2.13/2.14), Butterknife (8.7/8.8), Clean Architecture, MVP, MVVM, Kotlin, Java Swing, RxJava, RxAndroid, Retrofit 2, Jackson, AutoValue, Yelp Fusion (v3) REST API, Google Maps API, 整体仓库项目管理使用 Gradle， JUnit 4，AssertJ，Mockito 2，Robolectric 3，Espresso 2 以及 Java 最佳实践和设计模式。
>
> <https://github.com/vestrel00/business-search-app-java>

## Gist 概述

dagger.android 2.11 的使用支持  @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围使用快速概述，请看一下这个 [[gist]](https://gist.github.com/vestrel00/64be913f954989fe52c674247e093218)。请继续阅读完整的指南，并进行适当的分步设置和说明。

## 使用新的 dagger.android（2.11）框架从头开始创建一个项目

除此之外，现在是时候进入第1部分，该部分分为9个步骤：

1. 初始化 Android Gradle 项目。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/18)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.1-initial-project-setup)]
2. 将 Dagger（2.11）依赖项添加到 Gradle 构建脚本中。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/19)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.2-add-dagger-dependency)]
3. 设置 Dagger 注入框架。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/20)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.3-setup-dagger-injection-framework)]
4. 创建范围的实用程序类。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/21)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.4-create-scoped-utils)]
5. 创建导航到其他示例活动的主活动。[PR|TAG]
6. 创建示例1; 1个活动包含1个片段。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/23)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.6-create-example-1)]
7. 创建示例2; 1个活动包含2个片段。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/24)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.7-create-example-2)]
8. 创建示例3; 一个活动包含1个片段，该片段包含1个子片段。 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/25)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.8-create-example-3)]
9. 使用 `@ContributesAndroidInjector` 注解重构子组件。[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/26)|[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.9-refactor-to-contributesandroidinjector)]

## 1. 初始化 Android Gradle 项目。

最初项目结构如下：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovfsv6p9j307w0dst9a.jpg)



目前没有什么可说的。这只是一个 Android Gradle 项目框架。

> 注意：本文将向你介绍 dagger.android，Butterknife 和 MVP 设置，项目使用的 minSdkVersion 为17的。若要支持低于17乃至14的 API 级别需要使用 AppCompatActivity，支持Fragment 和 [dagger.android.support](https://github.com/google/dagger/tree/dagger-2.11/java/dagger/android/support) API。
>
> 看看这个 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/49)] 的迁移指南，以使用支持 API。最新的支持 API 设置可在 [[master-support](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/master-support)] 分支中找到。
>
> 注意：在撰写这些文章时使用了 Android Studio 2.3.3。自3/12/18以来，该项目已升级到 Android Studio  3.2.0 Canary 5。[[迁移]](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/70) 	

## 2. 将 Dagger（2.11）依赖项添加到 Gradle 构建脚本中

在 app/build.gradle 添加。

```java
dependencies {
    def daggerVersion = '2.11'

    annotationProcessor "com.google.dagger:dagger-compiler:$daggerVersion"
    annotationProcessor "com.google.dagger:dagger-android-processor:$daggerVersion"

    compile "com.google.dagger:dagger:$daggerVersion"
    compile "com.google.dagger:dagger-android:$daggerVersion"
}
```

>注意：AnnotationProcessor 已经在 Android Gradle 插件2.2版本中引入，替换了包括 apt / android-apt 在内的第三方插件。

## 3. 设置 Dagger 注入框架

依赖注入流程如下。

1. 应用程序注入活动
2. 活动注入片段
3. 片段注入子片段

依赖关系是从上到下共享的。 活动可以访问 Application@Singleton 依赖项。 片段可以访问 Application@Singleton 和 Activity@PerActivity 依赖关系。 子片段可以访问 Application@Singleton，Activity@PerActivity，和 (parent) Fragment@PerFragment。 

> 注意：本指南没有说明如何注入 Services，IntentServices，BroadcastReceivers 和 ContentProviders 为保持指南合理的大小。阅读本指南后，读者应该能够轻松地自行理解这一点。
>
> 要开始上述内容，请查看[基础框架类型](https://google.github.io/dagger/android.html#base-framework-types); [DaggerService](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/DaggerService.java)，[DaggerIntentService](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/DaggerIntentService.java)，[DaggerBroadcastReceiver](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/DaggerBroadcastReceiver.java) 和 [DaggerContentProvider](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/DaggerContentProvider.java)。

实现这一流程非常简单。

首先，创建自定义范围注解; @PerActivity，@PerFragment 和 @PerChildFragment。

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface PerActivity {
}
```

@PerActivity 范围注解指定依赖项的生命周期与 Activity 的生命周期相同。这用于注释在 Activity，Fragment  和 childFragments 的生命周期中表现为单例类似的依赖关系，而不是整个应用程序。

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface PerFragment {
}
```

@PerFragment 自定义范围注解指定依赖项的生命周期与 Fragment 的生命周期相同。这用于注释在 Fragment 和 childFragments 的生命周期中表现为单例类似的依赖关系，而不是整个应用程序或活动。

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface PerChildFragment {
}
```

@PerChildFragment 自定义范围注释指定依赖项的生命周期与子 Fragment（使用 Fragment.getChildFragmentManager ( ) 添加的片段内的片段）的生命周期相同。这用于注释在子 Fragment 的生命周期中表现类似于单例的依赖关系，而不是整个 Application，Activity 或父 Fragment。

> 注意：此设置不支持子片段内的子片段，因为会发生冲突的范围（编译时错误）。通常应避免子片段内的子片段。但是，如果需要其他级别的子片段，则需要创建另一个范围（可能是 @PerGrandChild 自定义范围注解）。

没有 @PerApplication 自定义作用域注解。 @Singleton 用于指定依赖项的生命周期与应用程序的生命周期相同。

>问题：为什么不创建 @PerApplication 自定义作用域而是使用 @Singleton？
>
>如果第三方库/依赖项使用依赖注入，则使用 @Singleton。如果你使用 @PerApplication 而不是标准 @Singleton 范围，则 Dagger 将无法自动注入 @Singleton 范围依赖项。

接下来，创建 App，AppModule 和 AppComponent，这是整个依赖注入设置的入口点。

```java
public class App extends Application implements HasActivityInjector {

    @Inject
    DispatchingAndroidInjector<Activity> activityInjector;

    @Override
    public void onCreate() {
        super.onCreate();
        DaggerAppComponent.create().inject(this);
    }

    @Override
    public AndroidInjector<Activity> activityInjector() {
        return activityInjector;
    }
}
```

所有依赖注入的入口点是 App，它实现了 [HasActivityInjector](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/HasActivityInjector.java)，它提供了一个 dagger 注入 的 DisagchingAndroidInjector \<Activity> 。这表明活动是参与 dagger.android 注入。

最高级别的注入发生在 onCreate 中使用 DaggerAppComponent.create ( ).inject (this)，这是一个由 Dagger 在编译时根据 AppComponent 生成的类。

> 注意：应用程序也可以用扩展 [DaggerApplication](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/DaggerApplication.java) 替代实现 [HasActivityInjector](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/HasActivityInjector.java)。但是，应该避免继承，以便稍后继承其他内容的选项是打开的。例如。应用程序需要扩展 [MultiDexApplication](https://developer.android.com/reference/android/support/multidex/MultiDexApplication.html)（multidex 并不是一个很好的例子，因为应用程序可以安装它，而不必扩展它 - 这只是一个假设）。
>
> 基本框架类型 DaggerApplication 包含比我们所拥有的更多的代码，除非我们需要注入 Service，IntentService，BroadcastReceiver 或 ContentProvider（尤其是 ContentProvider），否则这不是必需的。如果我们需要注入除 Activity 和 Fragment 之外的其他类型，或者如果你知道你的应用程序不需要扩展其他类型，那么它可能只是扩展 DaggerApplication 而不是自己编写更多的 dagger 代码。

```java
@Module(includes = AndroidInjectionModule.class)
abstract class AppModule {
}
```

AppModule 是一个抽象类，使用 @Module 注解进行注释，并包含 [AndroidInjectionModule](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/AndroidInjectionModule.java)，其中包含可用性的 bindings 以确保 dagger.android 框架类。 AppModule 现在是空的，但我们稍后会添加内容。

```java
@Singleton
@Component(modules = AppModule.class)
interface AppComponent {
    void inject(App app);
}
```

AppComponent 使用 @Component 和 @Singleton 进行注释，以指示其模块（AppModule）将提供 @Singleton 范围或无范围的依赖关系。

接下来是创建在整个应用程序中使用的基类; BaseActivity，BaseActivityModule，BaseFragment，BaseFragmentModule 和 BaseChildFragmentModule。

```java
public abstract class BaseActivity extends Activity implements HasFragmentInjector {

    @Inject
    @Named(BaseActivityModule.ACTIVITY_FRAGMENT_MANAGER)
    protected FragmentManager fragmentManager;

    @Inject
    DispatchingAndroidInjector<Fragment> fragmentInjector;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        AndroidInjection.inject(this);
        super.onCreate(savedInstanceState);
    }

    @Override
    public final AndroidInjector<Fragment> fragmentInjector() {
        return fragmentInjector;
    }

    protected final void addFragment(@IdRes int containerViewId, Fragment fragment) {
        fragmentManager.beginTransaction()
                .add(containerViewId, fragment)
                .commit();
    }
}
```

BaseActivity 包含与 App 中的代码非常相似的 dagger.android 代码。唯一的区别是 BaseActivity 实现了[HasFragmentInjector](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/android/HasFragmentInjector.java)，表明片段要参与 dagger.android 注入。

> 注意：BaseActivity 也可以用扩展 DaggerActivity 代替实现 HasFragmentInjector。但是，应该避免继承，以便在后来从其他事物继承的选项是开放的。
>
> 注意：对于支持 Fragment 和 AppCompatActivity 用户，请查看此 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/49)] 以获取迁移指南以使用支持 API。最新的支持 API 设置可在 [[master-support](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/master-support)] 分支中找到。
>
> 问题：为什么将 FragmentManager 注入到 BaseActivity 中？为什么不使用 getFragmentManager ( ) 方法？
>
> 简而言之，就是在测试中轻松进行模拟和验证。阅读此 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/52)] 以获得更详细的答案。

注入发生在 onCreate 中且在 super 调用之前。

BaseActivityModule.ACTIVITY_FRAGMENT_MANAGER 的 FragmentManager 也被注入。此名称在此处是必要的，以避免注入期间活动的 FragmentManager 与片段的子 FragmentManager 之间发生冲突。

> 注意：我们也可以创建自己的自定义 [@Qualifier](https://google.github.io/dagger/users-guide.html#qualifiers)，而不是使用 @Named 来区分活动和片段（子） FragmentManager。我写了一篇[长篇大论](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/20/files#r153559377)，讨论 @Qualifer 和 @Named 以及各自的优点/缺点。

addFragment 方法为子类提供了添加片段的功能。现在未使用，但稍后会使用。

> 问题：int containerViewId 的 @IdRes 注解是什么？
>
> [@IdRes](https://developer.android.com/reference/android/support/annotation/IdRes.html) 是 Android 支持注解库的一部分，它表示 “整型参数，字段或方法返回值应该是 id 资源引用” 。有关支持注解的完整列表，请[单击此处](https://developer.android.com/reference/android/support/annotation/package-summary.html)。
>
> Android 支持注解库附带了我们的 Dagger 2.11和 Butterknife 8.7依赖项。但是，你应该将支持注解库声明为单独的依赖项。访问[官方文档](https://developer.android.com/studio/write/annotations.html)以了解更多信息。

```java
@Module
public abstract class BaseActivityModule {

    static final String ACTIVITY_FRAGMENT_MANAGER = "BaseActivityModule.activityFragmentManager";

    @Binds
    @PerActivity
    abstract Context activityContext(Activity activity);

    @Provides
    @Named(ACTIVITY_FRAGMENT_MANAGER)
    @PerActivity
    static FragmentManager activityFragmentManager(Activity activity) {
        return activity.getFragmentManager();
    }
}
```

BaseActivityModule 提供了基本的活动依赖关系; 活动上下文和名为 ACTIVITY_FRAGMENT_MANAGER 的活动 FragmentManager。BaseActivity 的子类模块需要包含 BaseActivityModule 并提供 Activity 的具体实现。稍后会显示这方面的一个例子。

> 问题：我得到一个运行时错误，IllegalStateException 模块必须在对这个项目做一些修改后设置。我如何解决它？什么是 @Binds？它与 @Provides 有什么不同？
>
> 最可能的问题是你正尝试在抽象模块的非静态方法上使用 @Provides。[在这里阅读更多](https://medium.com/@vestrel00/giannis-tsironis-this-looks-like-the-same-issue-that-mohamed-alouane-posted-in-https-github-com-a2dbdc55be2c)。在那篇文章中，我还解释了 @Binds 是什么以及它与 @Provides 的不同之处。
>
> 注意：使用 @PerActivity 将上下文 activityContext（Activity 活动）作为范围是没有必要的，因为 Activity 实例将始终是唯一的（即使没有任何范围，也不会创建它的新实例）。通常，提供应用程序，活动，片段，服务等不需要范围注解，因为它们是被注入的组件，并且它们的实例是唯一的。
>
> 同样的事情适用于 static FragmentManager activityFragmentManager(Activity activity)。Activity 实例是唯一的，所以它返回的 FragmentManager 总是来自同一个 Activity。因此，@PerActivity 在这里不是必要的，因为范围是隐式地每个活动（字面上）。
>
> 但是，在这些情况下使用范围注释会使模块更易于阅读。为了理解它的范围，我们不必看看提供了什么。[我这里选择可读性和可靠性（可忽略）“性能/优化”](https://github.com/google/dagger/issues/832#issuecomment-320510038)。 （是的，我阅读了关于作用域依赖关系和[一次性同步块](https://github.com/google/dagger/blob/dagger-2.11/java/dagger/internal/DoubleCheck.java#L44)的 DoubleCheck 包装。鉴于以下支持材料，我仍然遵守我的声明;  [[1]](https://stackoverflow.com/questions/1671089/why-are-synchronize-expensive-in-java/1671097#1671097), [[2]](https://coderanch.com/t/462449/java/synchronization-expensive)。我不想进一步跑题，但实际上只有当多线程同时尝试访问相同的锁定资源时才会发挥同步成本。大多数依赖注入设置不会产生这种同步损失，因为注入通常只发生在一个线程中。) 	

```java
public abstract class BaseFragment extends Fragment implements HasFragmentInjector {

    @Inject
    protected Context activityContext;

    // Note that this should not be used within a child fragment.
    @Inject
    @Named(BaseFragmentModule.CHILD_FRAGMENT_MANAGER)
    protected FragmentManager childFragmentManager;

    @Inject
    DispatchingAndroidInjector<Fragment> childFragmentInjector;

    @SuppressWarnings("deprecation")
    @Override
    public void onAttach(Activity activity) {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
            // Perform injection here before M, L (API 22) and below because onAttach(Context)
            // is not yet available at L.
            AndroidInjection.inject(this);
        }
        super.onAttach(activity);
    }

    @Override
    public void onAttach(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            // Perform injection here for M (API 23) due to deprecation of onAttach(Activity).
            AndroidInjection.inject(this);
        }
        super.onAttach(context);
    }

    @Override
    public final AndroidInjector<Fragment> fragmentInjector() {
        return childFragmentInjector;
    }

    protected final void addChildFragment(@IdRes int containerViewId, Fragment fragment) {
        childFragmentManager.beginTransaction()
                .add(containerViewId, fragment)
                .commit();
    }
}
```

与 BaseActivity 一样，BaseFragment 实现了 HasFragmentInjector，指示子片段将参与 dagger.android 注入。

> 注意：BaseFragment 也可以用扩展 DaggerFragment 代替实现 HasFragmentInjector。但是，应该避免继承，以便稍后继承其他内容的选项是打开的。
>
> 问题：如果是 DialogFragments 怎么办？他们如何注入？
>
> 有人在这里提出了这个[问题](https://medium.com/@gtsironis8/is-there-a-way-to-inject-dialogfragments-i-have-tried-to-do-this-the-way-a-simple-fragment-is-897d42777096)，我在这里已经[回答](https://medium.com/@vestrel00/giannis-tsironis-injecting-dialogfragments-are-no-different-than-injecting-regular-or-support-270d8aa67622)了。
>
> 问题：为什么活动 Context 和子 FragmentManager 注入到 BaseFragment 中？为什么不分别使用getContext ( ) 和 getChildFragmentManager ( ) 方法？
>
> 简而言之，就是在测试中轻松进行模拟和验证。阅读此 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/52)] 以获得更详细的答案。

注入发生在 onAttach 中且在 super 调用之前。

> 注意：我们将 AndroidInjection.inject（this）放置在 Android 版本 M（API 级别23）及更高版本的 onAttach（Context）中，并且还放在 onAttach（Activity） 适用于 Android 版本 L（API 级别22）及以下。原因是 onAttach（Activity）从 API 级别23开始已被弃用。不仅在 onAttach（Context）中执行注入，因为它不会被运行 Lollipop（API 级别22）和更低级别的设备调用，这会在尝试访问 Fragment 依赖关系时导致 NullPointerException。我已经学会了这个难题。看看这个  [[BUG](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/46)]  和 [[BUGFIX](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/47)]。
>
> 另外需要注意的是，使用支持 Fragment 不需要上面的 API 级别检查。我们只需要在 onAttach（Context）中调用 AndroidSupportInjection.inject（this），因为即使对于 API level 22及更低版本，支持 Fragments 也会调用 onAttach（Context） ，这是有道理的，因为支持库的整个目标是给旧版本的 Android 代码提供更新的 API 级别。
>
> 看看这个[[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/49)]  的迁移指南，以使用支持 API。最新的支持 API 设置可在 [[master-support](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/master-support)] 分支中找到。

由 BaseActivityModule 提供的 Activity Context 以及名为 BaseFragmentModule.CHILD_FRAGMENT_MANAGER 的 FragmentManager。正如前面在 BaseActivity 中提到的那样，这个名字在这里是必要的，以避免注入期间 activity 的 FragmentManager 和 fragment 的子 FragmentManager 之间的冲突。

addChildFragment 方法为子类提供了添加子片段的能力。现在未使用，但稍后会使用。

子片段还扩展了 BaseFragment 以避免代码重复，稍后重构为 Model-View-Presenter（MVP）体系结构时，这将变得更加明显。权衡是子片段将有权访问 childFragmentManager 和 addChildFragment，除非支持 grandchildren（子片段内的子片段），否则子片段不应使用它。

```java
@Module
public abstract class BaseFragmentModule {

    public static final String FRAGMENT = "BaseFragmentModule.fragment";

    static final String CHILD_FRAGMENT_MANAGER = "BaseFragmentModule.childFragmentManager";

    @Provides
    @Named(CHILD_FRAGMENT_MANAGER)
    @PerFragment
    static FragmentManager childFragmentManager(@Named(FRAGMENT) Fragment fragment) {
        return fragment.getChildFragmentManager();
    }
}
```

BaseFragmentModule 提供了基本的片段依赖关系; 到目前为止只有子 FragmentManager 命名为 CHILD_FRAGMENT_MANAGER。BaseFragment 的子类模块需要包含 BaseFragmentModule 并提供名为 FRAGMENT 的 Fragment 的具体实现。稍后会显示这方面的一个例子。

> 注意：与 BaseActivity 和 BaseFragment FragmentManager 名称类似，为了消除有关在子片段中提供哪个片段的任何歧义，需要使用 Fragment 依赖项的不同名称。
>
> 例如，我们有父代片段 P 和子代片段 C.父片段 P 将使用 BaseFragmentModule.FRAGMENT 名称提供片段引用，而子片段 C 将使用 BaseChildFragmentModule.CHILD_FRAGMENT 名称提供片段引用。
>
> 如果父子片段依赖项没有唯一命名，那么子片段及其依赖关系将不知道提供了哪个片段，因为这两个依赖项具有相同类型的片段。它可能是父代片段或子代片段。因此，导致 “android.app.Fragment 绑定多次” 的编译错误的模糊性。
>
> 注意：Fragment.getChildFragment ( ) 方法仅在 API 级别17开始才可用。支持低于17乃至14的 API 级别需要使用 AppCompatActivity，支持 Fragment 和 dagger.android.supportAPIs。
>
> 看看这个  [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/49)] 的迁移指南，以使用支持 API。最新的支持 API 设置可在 [[master-support](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/master-support)] 分支中找到。

```java
@Module
public abstract class BaseChildFragmentModule {
  
    public static final String CHILD_FRAGMENT = "BaseChildFragmentModule.childFragment";
}
```

BaseChildFragmentModule 提供了基本的子片段依赖关系。BaseChildFragment 的子类模块需要包含 BaseChildFragmentModule，并提供名为 CHILD_FRAGMENT 的 Fragment 的具体实现。稍后会显示这方面的一个例子。

## 4. 创建有范围的实用程序类 

为了测试不同的范围（@Singleton，@PerActivity，@PerFragment 和 @PerChildFragment），我们将创建一个使用每个范围提供的“实用程序”类; SingletonUtil，PerActivityUtil，PerFragmentUtil 和 PerChildFragmentUtil。

```java
@Singleton
public final class SingletonUtil {

    @Inject
    SingletonUtil() {
    }
  
    public String doSomething() {
        return "SingletonUtil: " + hashCode();
    }
}
```

SingletonUtil 的范围是 @Singleton。这意味着应用程序和所有活动，片段和子片段及其依赖关系将共享此类的同一个单一实例。

为了自动提供这个类的一个实例，不需要手动创建它的一个新实例，提供了一个默认的包私有构造函数并使用 @Inject 注释。

doSomething ( ) 方法返回实例的 hashCode。这将稍后用于验证在所有活动，片段和子片段中使用相同的实例。

> 问题：如果我想让 SingletonUtil 持有对应用程序/应用程序上下文的引用，该怎么办？换句话说，我如何向 Singleton 依赖关系提供应用程序/应用程序上下文？
>
> 我为这个问题创建了一个[[ISSUE](https://github.com/vestrel00/android-dagger-butterknife-mvp/issues/42)] ，并有一个工作解决方案，如  [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/44)] 所示。答案是在 AppComponent  @Component.Builder 中使用 @BindsIntance 来绑定/提供应用程序实例。
>
> 更新：找到了提供应用程序上下文的更好方法。看看这个 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/45#discussion_r131630787)]。这个改进版本使用 AndroidInjector \<App> 和 AndroidInjector.Builder \<App> 为我们提供 inject（App app）方法和 App 实例，而不像我们在使用 @BindsInstance 方法时所做的那样。这种方法更符合 dagger.android 的注入，正如我们将在本文后面的章节中看到的那样。

```java
@PerActivity
public final class PerActivityUtil {

    private final Activity activity;

    @Inject
    PerActivityUtil(Activity activity) {
        this.activity = activity;
    }

    public String doSomething() {
        return "PerActivityUtil: " + hashCode() + ", Activity: " + activity.hashCode();
    }
}
```

PerActivityUtil 的作用域是 @PerActivity。这意味着 Activity 及其所有片段和子片段及其依赖关系将共享此类的相同实例。但是，不同的活动实例将拥有自己的实例。这在应用程序级别上不可用。

doSomething ( ) 方法返回实例和活动 hashCode。稍后将用它来验证每个 Activity 的所有片段和子片段中使用了相同的实例，但每个 Activity 都有自己的实例。

```java
@PerFragment
public final class PerFragmentUtil {

    private final Fragment fragment;

    @Inject
    PerFragmentUtil(@Named(BaseFragmentModule.FRAGMENT) Fragment fragment) {
        this.fragment = fragment;
    }
  
    public String doSomething() {
        return "PerFragmentUtil: " + hashCode() + ", Fragment: " + fragment.hashCode();
    }
}
```

PerFragmentUtil 的作用域是 @PerFragment。这意味着片段及其所有子片段及其依赖关系将共享此类的同一个实例。但是，不同的片段实例将拥有自己的实例。这在活动和应用程序级别上不可用。

> 注意：注意构造函数中片段的名称是 BaseFragmentModule.FRAGMENT。向上滚动阅读关于此名称的解释。

doSomething ( ) 法返回实例和片段 hashCode。稍后将用它来验证每个（父）片段的所有子片段中都使用了相同的实例，尽管每个（父）片段都有自己的实例。

```java
@PerChildFragment
public final class PerChildFragmentUtil {

    private final Fragment childFragment;

    @Inject
    PerChildFragmentUtil(@Named(BaseChildFragmentModule.CHILD_FRAGMENT) Fragment childFragment) {
        this.childFragment = childFragment;
    }

    public String doSomething() {
        return "PerChildFragmentUtil: " + hashCode()
                + ", child Fragment: " + childFragment.hashCode();
    }
}
```

PerChildFragmentUtil 的作用域是 @PerChildFragment。这意味着子 Fragment（使用 Fragment.getChildFragmentManager ( ) 添加的片段中的片段）及其所有依赖项将共享此类的同一个实例。但是，不同的子片段实例将拥有它们自己的此类实例。这在（父级）片段，活动和应用程序级别不可用。

> 注意：注意构造函数中片段的名称是 BaseChildFragmentModule.CHILD_FRAGMENT。向上滚动阅读关于此名称的解释。

doSomething ( ) 方法返回实例和子片段 hashCode。这将稍后用于验证在不同的子片段中没有使用相同的实例。

## 5. 创建导航到其他示例活动的主活动

现在我们将创建一个活动，提供一种方式来导航到其他3个示例活动。MainActivity 将包含一个片段 MainFragment，它包含3个按钮; 3个示例活动中的每一个各有1个按钮。MainActivity 只是托管 MainFragment 并通过侦听器接口监听 MainFragment 中的按钮点击。

首先，创建 MainFragment，MainFragmentListener，MainFragmentModule 和 MainFragmentSubcomponent。

```java
public final class MainFragment extends BaseFragment implements View.OnClickListener {

    @Inject
    MainFragmentListener listener;

    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
                             Bundle savedInstanceState) {
        return inflater.inflate(R.layout.main_fragment, container, false);
    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        // TODO (Butterknife) replace with butterknife view binding
        view.findViewById(R.id.example_1).setOnClickListener(this);
        view.findViewById(R.id.example_2).setOnClickListener(this);
        view.findViewById(R.id.example_3).setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.example_1:
                onExample1Clicked();
                break;
            case R.id.example_2:
                onExample2Clicked();
                break;
            case R.id.example_3:
                onExample3Clicked();
                break;
            default:
                throw new IllegalArgumentException("Unhandled view " + v.getId());
        }
    }

    private void onExample1Clicked() {
        listener.onExample1Clicked();
    }

    private void onExample2Clicked() {
        listener.onExample2Clicked();
    }

    private void onExample3Clicked() {
        listener.onExample3Clicked();
    }
}
```

MainFragment 扩展了我们的 BaseFragment，并监听来自 main_fragment 布局中声明的3个按钮的点击事件。 MainFragmentListener 方法根据点击事件被调用。为什么不把这个片段的布局放入 Activity 本身？我们可以做到这一点。但是，如果我们想要，我们将失去在不同活动中重复使用此片段和其他片段的能力。该活动仅充当用于片段间通信的1个或更多片段的主机。所有视图和逻辑都是碎片。这导致了模块化和可重用的架构以及更简单的 MVP 架构，我们将在后面做。

> 注意：这里有很多视图绑定代码。在 onViewCreated 中查找视图以设置它们的点击侦听器并用一个大的 switch 语句实现 View.OnClickListener。所有这些代码在使用 Butterknife 后将会大大简化。

```java
interface MainFragmentListener {

    void onExample1Clicked();

    void onExample2Clicked();

    void onExample3Clicked();
}
```

MainFragmentListener 只是定义了单击按钮时调用的方法。

```java
@Module(includes = BaseFragmentModule.class)
abstract class MainFragmentModule {

    @Binds
    @Named(BaseFragmentModule.FRAGMENT)
    @PerFragment
    abstract Fragment fragment(MainFragment mainFragment);
}
```

MainFragmentModule 包含 BaseFragmentModule 并根据 BaseFragmentModule 中指定的合同提供一个具体片段，在这个例子中为 MainFragment。

```java
// TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
@PerFragment
@Subcomponent(modules = MainFragmentModule.class)
public interface MainFragmentSubcomponent extends AndroidInjector<MainFragment> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainFragment> {
    }
}
```

MainFragmentSubcomponent 扩展了 AndroidInjector \<MainFragment> 并指定使用 MainFragmentModule 来提供它的依赖关系，并且 Builder 构建子组件实例将这些依赖注入到 MainFragment 中。子组件用 @PerFragment 注释，表明指定的模块提供 @PerFragment 范围和非范围的依赖关系。

这个 MainFragmentSubcomponent 接口被 Dagger 用来向 MainFragment 注入依赖。注意 Builder 是真正的准系统。这将允许稍后使用 @ContributesAndroidInjector 自动提供子组件注入器，从而可以不需要拥有 @Subcomponent 类。

> 注意：Builder 在我们的使用中是空的。如果需要在运行时注入依赖项而不是编译时，通常会使用它们。阅读[官方用户指南](https://google.github.io/dagger//users-guide.html#binding-instances)了解更多信息。

其次，创建 MainActivity，MainActivityModule 和 MainActivitySubcomponent。

```java
public final class MainActivity extends BaseActivity implements MainFragmentListener {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main_activity);

        if (savedInstanceState == null) {
            addFragment(R.id.fragment_container, new MainFragment());
        }
    }

    @Override
    public void onExample1Clicked() {
        // TODO start example 1 activity
        Toast.makeText(this, "Launch example 1", Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onExample2Clicked() {
        // TODO start example 2 activity
        Toast.makeText(this, "Launch example 2", Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onExample3Clicked() {
        // TODO start example 3 activity
        Toast.makeText(this, "Launch example 3", Toast.LENGTH_SHORT).show();
    }
}
```

MainActivity 扩展了我们的 BaseActivity 并实现了 MainFragmentListener 接口，其中 Toast 发送到每个接口方法调用。我们将用代码替换这些占位符，以便稍后开始我们的示例活动。

MainFragment 在 onCreate 中添加到 fragment_container 中。

> 提示：添加片段也可以通过使用 \<fragment> 标记声明片段类在 xml 布局中完成。

```java
@Module(includes = BaseActivityModule.class,
        subcomponents = MainFragmentSubcomponent.class)
abstract class MainActivityModule {

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @FragmentKey(MainFragment.class)
    abstract AndroidInjector.Factory<? extends Fragment>
    mainFragmentInjectorFactory(MainFragmentSubcomponent.Builder builder);

    @Binds
    @PerActivity
    abstract Activity activity(MainActivity mainActivity);

    @Binds
    @PerActivity
    abstract MainFragmentListener mainFragmentListener(MainActivity mainActivity);

}
```

MainActivityModule 包含 BaseActivityModule，并指定 MainFragmentSubcomponent 是此模块的一个子组件（从而获得对此活动的访问权限，进而访问应用程序的依赖项）。

mainFragmentInjectorFactory 方法接受 MainFragmentSubcomponent.Builder 并返回AndroidInjectorFactory。这为 MainFragment 提供了注入器。

活动方法根据 BaseActivityModule 中指定的协定接受具体的活动，在这种情况下为 MainActivity。

mainFragmentListener 接受实现 MainFragmentListener 的 MainActivity，并将其绑定到需要注入 MainFragmentListener 的 MainFragment 中。

```java
// TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
@PerActivity
@Subcomponent(modules = MainActivityModule.class)
public interface MainActivitySubcomponent extends AndroidInjector<MainActivity> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainActivity> {
    }
}
```

在结构上 MainActivitySubcomponent 与 MainFragmentSubcomponent 非常相似。它扩展了 AndroidInjector \<MainActivity> 并指定使用 MainActivityModule 提供它的依赖关系，并且 Builder 构建将这些依赖关系注入到 MainActivity 中的子组件实例。子组件用 @PerActivity 注释，表明指定的模块提供 @PerActivity 范围和非范围的依赖关系。

此子组件以及前面提到的所有其他子组件将在稍后用 @ContributesAndroidInjector 替换。

> 注意：为简洁起见，诸如在 AndroidManifest.xml 中添加活动，创建 main_activity.xml 和 main_fragment.xml 布局等内容已被忽略。如果你想看到这些排除的东西，那么看看 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/22)]。你也可以通过检查  [[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.5-create-main-activity)] 使项目在此状态下旋转。一般来说，本文的每个部分都有相应的 PR 和 TAG 供你方便使用。

再次，更新 AppModule 以提供 MainActivity 的注入器。

```java
@Module(includes = AndroidInjectionModule.class,
        subcomponents = MainActivitySubcomponent.class)
abstract class AppModule {

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @ActivityKey(MainActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    mainActivityInjectorFactory(MainActivitySubcomponent.Builder builder);
}
```

mainActivityInjectorFactory 为 MainActivity 提供注入器。 MainActivitySubcomponent 也被添加为 @Module 子组件字段中的子组件。

> 问题：为什么不将注入器放入 “Builders” 模块？
>
> 我读过一些展示 dagger.android 注入的博客。这些博客将注入器放置在名为 [BuildersModule](https://github.com/Nimrodda/dagger-androidinjector/blob/master/app/src/main/java/org/codepond/daggersample/BuildersModule.java) 的模块中，它包含在应用程序模块中。这只有在 BuildersModule 中提供的注入器具有相同的作用域（例如@PerActivity）或没有作用域时才有效。
>
> 因此，我们将注入器放置在适当的地方。 AppModule 中的活动注入器，活动模块中的片段注入器以及（父）片段模块中的子片段注入器。

统一应用程序显示主要活动，显示3个按钮（稍后）启动示例活动。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovg7d9w0j30dc07i74h.jpg)





## 6. 创建示例1; 1个活动包含1个片段

对于我们的第一个示例，我们将创建一个包含单个片段的活动。这个例子与我们为 MainActivity 和 MainFragment 所做的非常相似，所以为了简洁起见，我只会解释尚未解释的部分。Example1Activity 托管 Example1Fragment，它使用 SingletonUtil， PerActivityUtil 和 PerFragmentUtil 依赖关系注入。当do_something 按钮被点击时，每个 util 对象的哈希代码就会显示出来。这将允许我们验证范围依赖关系已被提供并正确注入。

首先，创建 Example1Fragment，Example1FragmentModule 和 Example1FragmentSubcomponent。

```java
public final class Example1Fragment extends BaseFragment implements View.OnClickListener {

    @Inject
    SingletonUtil singletonUtil;

    @Inject
    PerActivityUtil perActivityUtil;

    @Inject
    PerFragmentUtil perFragmentUtil;

    private TextView someText;
    
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        return inflater.inflate(R.layout.example_1_fragment, container, false);
    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        // TODO (Butterknife) replace with butterknife view binding
        someText = (TextView) view.findViewById(R.id.some_text);
        view.findViewById(R.id.do_something).setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.do_something:
                onDoSomethingClicked();
                break;
            default:
                throw new IllegalArgumentException("Unhandled view " + v.getId());
        }
    }

    private void onDoSomethingClicked() {
        String something = singletonUtil.doSomething();
        something += "\n" + perActivityUtil.doSomething();
        something += "\n" + perFragmentUtil.doSomething();
        showSomething(something);
    }

    private void showSomething(String something) {
        someText.setText(something);
    }
}
```

SingletonUtil，PerActivityUtil 和 PerFragmentUtil 被注入并在 onDoSomethingClicked ( ) 方法中使用，该方法在单击 do_something 按钮时被调用。然后使用每个 util 对象的 doSomething ( ) 方法用于连接 SingletonUtil，PerActivityUtil 和 PerFragmentUtil 对象的哈希代码以及活动和片段的哈希代码。这些哈希码然后显示在 someText TextView 中。

Example1FragmentModule 和 Example1FragmentComponent 与 MainFragmentModule 非常相似，因此为了简洁起见，MainFragmentComponent 因此被排除在本文之外。

> 注意：要重申，你可以查看完整更改集的  [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/23/files)] ，并签出完整代码的  [[TAG](https://github.com/vestrel00/android-dagger-butterknife-mvp/tree/a.6-create-example-1)]。这是我对本文其余部分重申的最后一次。

其次，创建 Example1Activity，Example1ActivityModule 和 Example1ActivitySubcomponent。

```java
public final class Example1Activity extends BaseActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.example_1_activity);

        if (savedInstanceState == null) {
            addFragment(R.id.fragment_container, new Example1Fragment());
        }
    }
}
```

如上所示，Example1Activity 只是托管 Example1Fragment。

Example1ActivityModule 和 Example1ActivityComponent 与 MainActivityModule 非常相似，因此为了简洁起见，MainActivityComponent 因此被排除在本文之外。

再次，更新 AppModule 以提供 Example1Activity 的注入器。

```java
@Module(includes = AndroidInjectionModule.class,
        subcomponents = {
                MainActivitySubcomponent.class,
                Example1ActivitySubcomponent.class
        })
abstract class AppModule {

    ...
      
    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @ActivityKey(Example1Activity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    example1ActivityInjectorFactory(Example1ActivitySubcomponent.Builder builder);
}
```

统一应用程序的示例1显示了 “Do Something” 按钮。点击按钮显示我们的 util 对象，活动和片段的哈希代码。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovgdvrkgj30dc07i74h.jpg)

这里没有太多可以说的，除了应用程序按预期工作以及依赖注入工作的事实。每个 util 对象的哈希代码都是不同的，但这是可以预料的，因为它们都属于不同的类。活动和片段的哈希码当然是不同的。

> 注意：使用散列码来确定对象的身份和唯一性对于这个小示例应用程序就足够了。正如 [Object.hashCode ( )](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#hashCode%28%29) javadoc 所述，“尽可能合理实用，由类 Object 定义的 hashCode 方法确实为不同的对象返回不同的整数”。但是，如果内存中有足够的对象（数千个），则可能会有一些哈希代码发生冲突。查看[散列码唯一性](https://stackoverflow.com/questions/1381060/hashcode-uniqueness)。

一个更有趣的例子，以及证明我们的依赖注入设置正常工作的例子将在例子2和例子3中提供，所以请继续阅读！

## 7. 创建示例2; 1个活动包含2个片段

对于我们的第二个例子，我们将创建一个包含2个片段的 Activity。这个例子与我们前面的例子非常相似。Example2Activity 托管 Example2AFragment 和 Example2BFragment，它们都使用 SingletonUtil，PerActivityUtil 和 PerFragmentUtil 依赖关系注入。然后在每个片段的 do_something 按钮被点击时显示每个 util 对象的哈希代码。

首先，创建2个片段（Example2AFragment 和 Example2BFragment）及其各自的模块和子组件。然后创建活动（Example2Activity）及其模块和子组件。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovgmgqonj308n06c74l.jpg)

> 问题：为什么不将 Dagger 模块和子组件放在 di 软件包中？或者把它们全部放在名为 internal / di 的包中？
>
> 我见过几个项目这样做，我甚至自己做了。这让我怀疑这是否是标准做法。无论如何，我们并没有将所有的 Dagger 模块和子组件放入这样的包中，因为那时必须公开所有必须提供的依赖关系。
>
> 将模块和子组件放置在与它们注入/提供的依赖关系相同的包中，可以使类成为包专用的，从而增加了包的模块性。它还使你可以策略性地考虑哪些类属于哪个包，并帮助你保持包的重点。

活动和2片段与第一个例子中的活性和片段非常相似。除了类名以外，其中很多是前一个示例的复制和粘贴。为了清楚起见，这是为了达到目的。

> 问题：为什么要复制和粘贴代码而不是使用基类共享代码？为什么不把共享示例片段代码放在基类中？
>
> 复制和粘贴代码仅适用于本指南，通过使每个示例相互独立来提高清晰度。通过一切手段，你可以采取代码，并自己重构，按照你所想。本指南仅为读者容易理解而违反 [DRY 原则](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。

```java
public final class Example2Activity extends BaseActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.example_2_activity);

        if (savedInstanceState == null) {
            addFragment(R.id.fragment_a_container, new Example2AFragment());
            addFragment(R.id.fragment_b_container, new Example2BFragment());
        }
    }
}
```

Example2Activity 托管 Example2AFragment 和 Example2BFragment。

```java
@Module(includes = BaseActivityModule.class,
        subcomponents = {
                Example2AFragmentSubcomponent.class,
                Example2BFragmentSubcomponent.class
        })
abstract class Example2ActivityModule {

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @FragmentKey(Example2AFragment.class)
    abstract AndroidInjector.Factory<? extends Fragment>
    example2AFragmentInjectorFactory(Example2AFragmentSubcomponent.Builder builder);

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @FragmentKey(Example2BFragment.class)
    abstract AndroidInjector.Factory<? extends Fragment>
    example2BFragmentInjectorFactory(Example2BFragmentSubcomponent.Builder builder);

    @Binds
    @PerActivity
    abstract Activity activity(Example2Activity example2Activity);
}
```

Example2ActivityModule 根据 BaseActivityModule 中指定的契约为2个片段提供注入器以及 Activity 的具体实现。

其次，我们将 Example2Activity 注入器添加到我们的 AppModule。这和前面的例子几乎是一样的。

```java
@Module(includes = AndroidInjectionModule.class,
        subcomponents = {
                MainActivitySubcomponent.class,
                Example1ActivitySubcomponent.class,
                Example2ActivitySubcomponent.class
        })
abstract class AppModule {

    ...

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @ActivityKey(Example2Activity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    example2ActivityInjectorFactory(Example2ActivitySubcomponent.Builder builder);
}
```

统一应用程序的示例2显示了这两个片段都包含 “Do Something” 按钮。点击按钮显示我们的 util 对象，活动和片段的哈希代码。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovh7b2wgj30io0aigmt.jpg)

在这两个片段中注入的 SingletonUtil 是相同的（通过它们的哈希码可以看出）。这两个片段都存在于相同的活动中，因此它们都具有相同的 PerActivityUtil 和 Activity。由于这两个片段不同，它们的 PerFragmentUtil 和 Fragment 实例是不同的。

> 注意：将同一个片段类的多个不同实例添加到同一活动中会为每个片段实例生成不同的 @PerFragment 依赖关系集。

## 8. 创建示例3; 一个活动包含1个片段，该片段包含1个子片段

对于我们的第三个示例，我们将创建一个包含1个片段的 Activity，且该片段包含一个子片段。Example3Activity 托管  Example3ParentFragment，Example3ParentFragment 托管   Example3ChildFragment。父代和子代片段都使用 SingletonUtil，PerActivityUtil 和 PerFragmentUtil 依赖关系注入。Example3ChildFragment 另外还注入了 PerChildFragmentUtil 依赖项。然后在每个片段的 do_something 按钮被点击时显示每个 util 对象的哈希代码。

首先，创建父级和子级片段以及它们各自的模块和子组件。然后创建活动及其模块和子组件。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovhbokscj309b069glx.jpg)

该活动和2个片段与前面实例中的活动和片段非常相似。除了类名以外，其中很多是前一个示例的复制和粘贴。虽然有一些差异。

```java
public final class Example3ParentFragment extends BaseFragment implements View.OnClickListener {

    ...
      
    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        if (savedInstanceState == null) {
            addChildFragment(R.id.child_fragment_container, new Example3ChildFragment());
        }

        ...
    }
}
```

Example3ParentFragment 将 Example3ChildFragment 添加为子片段。

```java
@Module(includes = {
        BaseFragmentModule.class
},
        subcomponents = Example3ChildFragmentSubcomponent.class)
abstract class Example3ParentFragmentModule {

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @FragmentKey(Example3ChildFragment.class)
    abstract AndroidInjector.Factory<? extends Fragment>
    example3ChildFragmentInjectorFactory(Example3ChildFragmentSubcomponent.Builder builder);

    ...
}
```

Example3ParentFragmentModule 提供了 Example3ChildFragment 的注入器。

```java
public final class Example3ChildFragment extends BaseFragment implements View.OnClickListener {

    ...
      
    @Inject
    PerChildFragmentUtil perChildFragmentUtil;

    ...

    private void onDoSomethingClicked() {
        String something = singletonUtil.doSomething();
        something += "\n" + perActivityUtil.doSomething();
        something += "\n" + perFragmentUtil.doSomething();
        something += "\n" + perChildFragmentUtil.doSomething();
        showSomething(something);
    }
    
    ...
}
```

Example3ChildFragment 注入一个 PerChildFragmentUtil 对象。

```java
@Module(includes = {
        BaseChildFragmentModule.class,
})
abstract class Example3ChildFragmentModule {
    
    @Binds
    @Named(BaseChildFragmentModule.CHILD_FRAGMENT)
    @PerChildFragment
    abstract Fragment fragment(Example3ChildFragment example3ChildFragment);
}
```

Example3ChildFragmentModule 使用 @PerChildFragment 作用域提供它的依赖关系。

```java
// TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
@PerChildFragment
@Subcomponent(modules = Example3ChildFragmentModule.class)
public interface Example3ChildFragmentSubcomponent extends AndroidInjector<Example3ChildFragment> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<Example3ChildFragment> {
    }
}
```

Example3ChildFragmentSubcomponent 用 @PerChildFragment 作用域进行注释，指示其模块将仅提供 @PerChildFragment 作用域或无限范围依赖关系的依赖关系。

```java
@Module(includes = BaseActivityModule.class,
        subcomponents = Example3ParentFragmentSubcomponent.class)
abstract class Example3ActivityModule {

    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @FragmentKey(Example3ParentFragment.class)
    abstract AndroidInjector.Factory<? extends Fragment>
    example3ParentFragmentInjectorFactory(Example3ParentFragmentSubcomponent.Builder builder);
}
```

Example3ActivityModule 提供了 Example3ParentFragment 的注入器。

```java
@Module(includes = AndroidInjectionModule.class,
        subcomponents = {
                MainActivitySubcomponent.class,
                Example1ActivitySubcomponent.class,
                Example2ActivitySubcomponent.class,
                Example3ActivitySubcomponent.class
        })
abstract class AppModule {
    ...
    
    // TODO (ContributesAndroidInjector) remove this in favor of @ContributesAndroidInjector
    @Binds
    @IntoMap
    @ActivityKey(Example3Activity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    example3ActivityInjectorFactory(Example3ActivitySubcomponent.Builder builder);
}
```

像往常一样，AppModule 为活动提供注入器（Example3Activity）。

统一应用程序的示例3显示父代和子代片段，其中都包含 “Do Something” 按钮。点击按钮显示我们的 util 对象，活动，父代片段和子片段的散列代码。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovhhskvwj30io0ai400.jpg)

与前面的例子类似，在这两个片段中注入的 SingletonUtil 都是相同的（通过它们的散列码可以看出）。这两个片段都存在于同一个活动中，因此它们都具有相同的 PerActivityUtil 和 Activity 哈希码。

PerFragmentUtil 和 Fragment 实例是相同的，因为子片段位于父片段内。PerChildFragmentUtil 仅在子片段中可用。当然，子片段与（父）片段有不同的哈希码。

在这一点上，我们已经证明，我们的设置工作！

## 9. 使用 @ContributesAndroidInjector 注解重构子组件

如前所述，我们的 Dagger 子组件是准系统，并且是相当重复的，违反了 DRY 原则！为了解决这个问题， Dagger 2.11为我们提供了 @ContributesAndroidInjector，它自动生成 @Subcomponent，否则我们必须编写自己的代码。这使我们可以删除我们所有的子组件类。

> 问题：为什么要显示 AndroidInjector，AndroidInjector.Factory，@Subcomponent 和 @ Subcomponent.Builder 的用法？我们不能在所有情况下都使用 @ContributesAndroidInjector 吗？
>
> 在某些情况下，我们将无法使用 @ContributesAndroidInjector。这种情况包括你需要使用 @BindsInstance 绑定组件实例的时候，如下所示。在这种情况下，你将需要使用 @ Subcomponent.Builder。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovhor1yrj30k0091tan.jpg)

> 未来版本的 dagger-android 可能（也可能不会）在 @ContributesAndroidInjector 中添加诸如此类的附加功能，但截至目前，版本2.11-2.14不支持它。
>
> 问题：确切地说，我们什么时候可以使用 @ContributesAndroidInjector？ 这是在正式的 dagger-android 集成页面中的小型 pro-tip 中回答的。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovhssreuj30k001174v.jpg)

> 本文中的示例符合使用 @ContributesAndroidInjector 的条件。

迁移到 @ContributesAndroidInjector 只需3个简单的步骤;

1. 用 @ContributesAndroidInjector 替换所有模块中的 AndroidInjector.Factory 用法。
2. 删除每个模块中的所有子组件包含。
3. 删除使用 @Subcomponent 注解的所有类

这里有很多（类似的）改变，所以我不会显示所有的改变。我将只演示如何重构示例3，因为它使用了所有不同的作用域。你可以查看完整 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/26/files)] 中的其他更改。

首先，用 @ContributesAndroidInjector 替换所有模块中的 AndroidInjector.Factory 用法。

AppModule.java

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovhye73ej30hg05hmya.jpg)

Example3ActivityModule.java

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovi1oqw2j30ho05fdh1.jpg)

Example3ParentFragmentModule.java

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovi5gpukj30hg05cdh2.jpg)

请注意，模块中提供的注入器及其模块的范围现在在模块提供方法中提供，而不是在子组件类中提供。

其次，删除每个模块中的所有子组件。

AppModule.java

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovi8yilcj309u05iq3d.jpg)

请注意，对于此示例，只需要删除 Example3ActivitySubcomponent。

Example3ActivityModule.java

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovic7uvqj30cf02zdgb.jpg)

Example3ParentFragmentModule.java

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovifzfvhj30c304bjrx.jpg)

最后，删除所有使用 @Subcomponent 注解的类。这部分需要删除 Example3ActivitySubcomponent，Example3ParentFragmentSubcomponent 和 Example3ChildFragmentSubcomponent。

示例3此时不再使用子组件！再次，你可以查看完整 [[PR](https://github.com/vestrel00/android-dagger-butterknife-mvp/pull/26/files)] 中的主要活动和其他示例的其他更改。

### 结束

在这个三部分系列的这一部分中， 我们创建了一个项目，使用新的 Dagger.Android（2.11 / 2.12 / 2.13 / 2.14）依赖注入框架，支持 @Singleton，@PerActivity，@PerFragment 和 @PerChildFragment 范围。如果你使用过以前版本的 Dagger 2，你应该能够看到这个新的 dagger.android 设置涉及的 Dagger 代码少了多少！

> 问题：使用 dagger-android 和 MVP 有一个[谷歌示例](https://github.com/googlesamples/android-architecture/tree/todo-mvp-dagger/)。本指南与 Google 示例有不同的结构。我信任/遵循哪个？
>
> 这完全取决于你。我写了本指南，向其他人展示了使用 dagger-android，Butterknife 和 MVP 的架构。至于任何事情，都不会只有一种做法。选择一个满足你需求的架构/结构，并选择一个对你有意义且适合你的架构/结构。我建议你完整阅读本指南。我尽力解释每一行代码，这样一切都变得有意义，这是谷歌示例可能不提供的东西（也许是其他事情）。

阅读[第2部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-2-6eaf60965df7)，了解如何替换我们创建的大量手写样板视图绑定代码使用 ButterKnife（8.7 / 8.8）。

阅读[第3部分](https://proandroiddev.com/how-to-android-dagger-2-10-2-11-butterknife-mvp-part-3-ed5acf40eb19)，了解如何将代码重构为 Model-View-Presenter（MVP）以提高可测试性，可维护性和可伸缩性。

