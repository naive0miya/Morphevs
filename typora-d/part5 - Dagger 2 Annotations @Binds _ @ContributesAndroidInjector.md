# Dagger 2 Annotations: @Binds & @ContributesAndroidInjector

> 原文 (Medium)：[Dagger 2 Annotations: @Binds & @ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f?source=user_profile---------0----------------)
>
> 作者：[Garima Jain](https://proandroiddev.com/@ragdroid?source=post_header_lockup)

[TOC]

在本文中，我们将简要介绍两个注解：@Binds 和 @ContributesAndroidInjector 。 阅读以前的文章并不一定要理解这一点，但对 dagger 的基本理解是必须的。

dagger 有许多有用的注释，不仅可以减少你编写的代码，还可以使你的生成的代码更加优化，其中一个注解是 @Binds。

## [Binds](https://google.github.io/dagger/api/2.11/dagger/Binds.html)

提供这个注解来替换 @Provides 方法，它们只是返回注入的参数。举个例子

我们有一个实现 LoginContract.Presenter 的 LoginPresenter。没有 @Binds，它的提供者方法将如下所示：

```java
@Provides
public LoginContract.Presenter 
  provideLoginPresenter(LoginPresenter loginPresenter) {
    return loginPresenter;
}
```

在上面的例子中，我们可以使用 @Binds 注解并使上面的方法 abstract：

```java
@Binds
public abstract LoginContract.Presenter
  provideLoginPresenter(LoginPresenter loginPresenter);
```

当然，在这种情况下，我们还需要将我们的模块标记为抽象，这比具体的模块更有效率，从而使得 @Binds 更高效。

## 抽象模块为什么有更高性能？

@Provides 方法是实例方法，他们需要我们模块的实例才能被调用。 如果我们的模块是抽象的并且包含 @Binds 方法，那么 Dagger 将不会实例化我们的模块，而是直接使用我们的注入参数（在上面的例子中为 LoginPresenter）的提供者。

## 如果你的模块同时包含 @Provides 和 @Binds 方法呢？

Dagger 文档在 @Binds vs @Provides 方法上有很多常见问题。如果你的模块有 @Provides 和 @Binds 方法，你有两个选择：

- 最简单的方法是将你的 @Provides 实例方法标记为静态。
- 如果需要将它们保存为实例方法，那么可以将模块拆分为两个，并将所有的 @Binds 方法抽取到一个抽象模块中。

要了解更多这样的优化，甚至是最近的2.12版 dagger 带来的其他优化。我建议你观看 Google 的 Ron Shapiro 的[“优化 Android 上的 Dagger”](https://www.youtube.com/watch?v=PBrhRvhF00k)。

## @ContributesAndroidInjector

如果你已经看到了 dagger-android 的基本实现，你会知道它引入了一些样板代码。 当你试图切换到  android dagger 或转换你的当前应用程序使用它。 我认为最麻烦的是必须为每个 activity，fragment ，service 等创建单独的子组件，并将其添加到 DispatchingAndroidInjector 的 injectorFactor 中（通过使用 @IntoSet）

Dagger Android 引入了一个注解，可以为你减少 Component ，Component ，Binds ， IntoSet ，Subcomponent， ActivityKey， FragmentKey 等样板。

如果你有一个像下面这样简单的模块，你可以让 Dagger 处理剩下的事情。

```java
@Module
public abstract class LoginModule {

    @Binds
    public abstract LoginContract.Presenter
  provideLoginPresenter(LoginPresenter loginPresenter);

}
```

所有你需要做的就是在组件的 Module 中写下面的代码片段，它将成为生成的 LoginComponent 的超级组件。 例如，如果你有一个 AppComponent，并且你想让 dagger 为你的 LoginActivity 生成一个 LoginSubcomponent，你会在你的 AppModule 中写下如下代码片段。

```java
@ContributesAndroidInjector(modules = LoginModule.class)
abstract LoginActivity loginActivity();
```

为所有这些绑定提取一个单独的 ActivityBindingModule，并将它包含在 AppComponent 的模块列表中: 

```java
@Singleton
@Component(modules = {
        AppModule.class,
        ActivityBindingModule.class})
public interface AppComponent
```

现在，让我们参考 @ContributesAndroidInjector 的文档，看看这里发生了什么 : 

- “为此方法的返回类型生成一个 AndroidInjector ”：它将生成 AndroidInjector \<LoginActivity>。 （更多关于它在即将到来的文章中）
- “这个注解必须应用于返回一个具体的 Android 框架类型的模块的抽象方法”，即 Activity / Fragment 等。

以下是使用 @ContributesAndroidInjector 为你生成的样板：

```java
//Simplified Generated File
@Module(subcomponents =
           Module_LoginActivity.LoginActivitySubcomponent.class)
public abstract class Module_LoginActivity {
  private Module_LoginActivity() {}

  @Binds
  @IntoMap
  @ActivityKey(LoginActivity.class)
  abstract AndroidInjector.Factory<? extends Activity>
      bindAndroidInjectorFactory(
            LoginActivitySubcomponent.Builder builder);

  @Subcomponent(modules = LoginModule.class)
  public interface LoginActivitySubcomponent extends
                                 AndroidInjector<LoginActivity> {
    @Subcomponent.Builder
    abstract class Builder extends
                         AndroidInjector.Builder<LoginActivity> {}
  }
}
```

1. 它生成 LoginActivitySubcomponent。
2. 它为我们添加了必要的 @Subcomponent 注解。
3. 它将（LoginActivity.class，LoginActivitySubcomponent.Builder）的条目添加到 DispatchingAndroidInjector 使用的注入器工厂映射中。 Dagger-Android 使用这个条目来构建我们的 LoginActivitySubcomponent 并为 LoginActivity 执行注入。
4. 另外，将 LoginActivity 绑定到对象图。 （请参阅 AndroidInjector 的源代码知道如何，提示：[@BindsInstance](https://proandroiddev.com/dagger-2-component-builder-1f2b91237856)）

同样，你也可以使用 @ContributesAndroidInjector 和任何 android 框架类型，如 Fragments，Services 等。下面是一个使用 [Fragments 的例子](https://github.com/ragdroid/Dahaka/blob/dagger-android/app/src/main/java/com/ragdroid/dahaka/activity/home/HomeModule.java#L23)。

## 范围界定 @ContributesAndroidInjector

如果你想在你的 LoginModule 范围内确定依赖项的范围，那么你还需要将生成的 LoginActivitySubcomponent 作为范围。 在这里了解更多关于范围。 要做到这一点，你可以指定一个范围以及 @ContributesAndroidInjector 注解，这样你可以在你的 LoginModule 中包含任何东西，如下所示。

```java
@ContributesAndroidInjector(modules = LoginModule.class)
@ActivityScope
abstract LoginActivity loginActivity();
```

这将生成用 ActivityScope 注释的 LoginActivitySubcomponent。

```java
//Simplified Generated File
@Module(subcomponents =
           Module_LoginActivity.LoginActivitySubcomponent.class)
public abstract class Module_LoginActivity {
  ...

  @Subcomponent(modules = LoginModule.class)
  @ActivityScope
  public interface LoginActivitySubcomponent extends
                                 AndroidInjector<LoginActivity> {
    @Subcomponent.Builder
    abstract class Builder extends
                         AndroidInjector.Builder<LoginActivity> {}
  }
}
```

既然组件是注解了作用域的，你可以应用一般的作用域规则，并用 @ActivityScope 标记你的任何 @Binds 方法：

```java
@Binds
@ActivityScope
public abstract LoginContract.Presenter provideLoginPresenter(LoginPresenter loginPresenter);
```

结束:)

正如你在上面的例子中看到的那样，通过了解背后发生了什么，我们正在尝试与生成代码的野兽(Dahaka)成为朋友，这个代码反过来帮助我们优化我们生成的代码和减少模板。 

在即将发布的文章中，我们将尝试了解更多关于匕首的信息，以及幕后的内容。如果你想看到 dagger-android 的实现，你可以看看这里的 Dahaka [github](https://github.com/ragdroid/Dahaka) 仓库。

