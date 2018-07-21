# Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes

> 原文 (Medium)：[Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes](https://proandroiddev.com/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736?source=user_profile---------2----------------)
>
> 作者：[Garima Jain](https://proandroiddev.com/@ragdroid?source=post_header_lockup)

[TOC]

这篇文章是 [**Dagger and the Dahaka**](https://medium.com/@ragdroid/dagger-2-android-defeat-the-dahaka-b1c542233efc) 系列的一部分。如果你想了解更多关于 “Dahaka” 的文章，请查看关于这个系列的[主文章](https://medium.com/@ragdroid/dagger-2-android-defeat-the-dahaka-b1c542233efc)。

在这篇文章中，我们看看 Singleton 注解以及它如何影响生成的 DaggerComponent 类。然后，我们将看看另一个名为 Reusable 的注解，并尝试了解如何更好地使用这些范围。

想知道更多关于 “Dahaka”？本文使用 “Dahaka” 的视角。
![](https://ws4.sinaimg.cn/large/006tKfTcgy1frot6gpe6fj30m80c8ac6.jpg)
所以，让我们直接进入主题！ 😎

## [Scope](http://docs.oracle.com/javaee/6/api/javax/inject/Scope.html)

这是来自 javax.inject 包的注解。引入这个包是为了在不同的框架中[标准化 DI](http://www.java-tv.com/2012/08/06/standardized-dependency-injection-in-java/)（依赖注入）。

就像 Donn Felker 在他的 [Caster IO 讲座中提到的 Scopes](https://caster.io/episodes/dagger-2-scopes-part-1/) 一样。范围就像一个变量的范围。 它定义了一个范围内实例的存在时间。 这里有一些重要的事情需要注意: 

> 默认情况下，如果没有范围注解，注入器创建一个实例，使用一个注入的实例，然后忘记它 。
>
> 如果存在范围注解，则注入器可以保留该实例，以便在稍后的注入中可能重用。  - [javax.inject javadocs](http://docs.oracle.com/javaee/6/api/javax/inject/Scope.html)

也就是说。 如果我们不指定任何范围注解，组件将在每次注入依赖项时创建一个新实例，而如果我们指定一个 范围，组件可以保留该实例以供将来使用。 稍后再谈这个问题。 

我们从 javax.inject 包中的另一个标准注解开始：

## @Singleton

javax.inject 包定义了另一个在各种 DI 框架中通用的注解。 这个注解的依赖范围只被实例化一次。 或者你可以说，这个依赖的实例范围遍及整个应用程序。

在 dagger 2 中，组件与范围绑定。 我们可以用一个范围对一个组件接口进行注释，这将使我们能够告诉 dagger 保留一个特定依赖的实例，然后重新使用它以备将来的注入。 我们可以通过两种方式重用依赖关系: 

- 使用与我们的组件相同的注解， 在我们的模块中注解 Provider 方法。
- 注解依赖类本身，再次使用与我们的组件相同的注解。

举个例子：
我们有一个简单的 AppComponent，显示了 Application 依赖关系：

```java
@Component(modules = AppModule.class)
 @Singleton
 public interface AppComponent {
     Application application();
 }
```

请注意，没有必要在你的 AppComponent 中公开 Application。这只是为了演示目的而完成的。

## 没有 @Singleton 范围：

我们有一个提供这个 Application 依赖的模块

```java
@Module
 public class AppModule {
 
     private final Application application;
 
     public AppModule(Application application) {
         this.application = application;
     }
 
     @Provides //No @Singleton annotation
     Application applicationProvider() {
         return application;
     }
 
 }
```

请注意，我们不在这里使用 Singleton 注解，这是有道理的，因为 application 本身总是 @Singleton 。

对于上面的代码，Dagger 创建一个 DaggerAppComponent，它将有一个初始化方法，如下所示：

```java
@Generated
public final class DaggerAppComponent implements AppComponent {
  private Provider<Application> applicationProvider;  
  
  /** other code **/
  private void initialize(final Builder builder) { 
     this.applicationProvider =
        AppModule_ApplicationFactory.create(builder.appModule);
  }
}
```

还有一些我们还不了解的东西。让我们一个接一个地说：

- **Provider \<Application>：** Provider \<T> 也是 javax.inject 包中的标准接口。 Dagger2 为我们的依赖项创建 providers ，这些 providers 是在我们的模块中使用 @Provides 注解提供的。 它初始化每个依赖项的 Provider \<T>，并在 Component 实例中保持对 provider 的引用。 随后可以使用 provider.get ( ) 方法获取依赖关系。 我们将在稍后的帖子中说明 provider 注入。
- **AppModule_ApplicationFactory：** 对于我们的依赖关系，Dagger2 创建一个 Factory \<T> 实现，它负责使用模块或 Inject 构造函数中的提供者方法创建我们的依赖项。 工厂与提供者相似，略有不同。 当我们调用 factory.get ( ) 时，保证调用我们的 module.providerMethod ( ) 或注入构造函数来获得依赖关系。 然而，当我们调用 provider.get ( ) 时，提供者也可能会应用一些作用域语义（如 DoubleCheck 等）。示例：在上述情况下，调用 applicationFactory.get ( ) 将调用 AppModule 的 applicationProvider ( ) 方法。 这里，Factory \<Application> 和 Provider \<Application> 之间没有区别，因为提供者方法没有使用 Singleton 注解。

## 带有 @Singleton 作用域：

让我们将 Singleton 作用域应用于 AppModule 中的提供者方法，看有什么不同 : 

```java
@Module
public class AppModule {

    private final Application application;

    public AppModule(Application application) {
        this.application = application;
    }

    @Provides
    @Singleton
    Application applicationProvider() {
        return application;
    }
}
```

我们用 @Singleton 范围标记了我们的 Application 提供者方法。现在 Dagger 会为我们的 DaggerAppComponent 创建 initialize ( ) 方法，如下所示：

```java
private void initialize(final Builder builder) {
   this.applicationProvider =
       DoubleCheck.provider(
            AppModule_ApplicationFactory.create(builder.appModule));
 }
```

请注意，我们的 Factory \<Application> 包装在 DoubleCheck.provider ( ) 调用中。 在这种情况下，Factory \<Application> 与 Provider \<Application> 不同。 Factory \<Application> 将直接调用 appModule.application ( )，而 Provider 是一个 DoubleCheck 提供程序。

## DoubleCheck

这里来自 [DoubleCheck](https://github.com/google/dagger/blob/master/java/dagger/internal/DoubleCheck.java) 的 java-doc：

> 一个 Lazy 和 Provider 实现，使用双重检查语句记忆从委托返回的值正如 Effective Java 2 的 71 项目中的描述。

记忆( Memoization ) 对于像我这样的人来说意味着那里可能有一个拼写错误。

> 在计算机领域中，memoization 或 memoisation 是一种优化技术，主要用于加速计算机程序，通过存储昂贵的函数调用的结果，当相同的输入再次发生，返回缓存的结果。 - [维基百科](https://en.wikipedia.org/wiki/Memoization)

那么，在我们的情况下，我们可以没有 DoubleCheck 锁。看看 DoubleCheck 的[源代码](https://github.com/google/dagger/blob/master/java/dagger/internal/DoubleCheck.java)，以获得更好的描述。

## 补充 #1

我们大多数人都习惯于对我们的所有提供者方法进行范围界定（Scope All Things）！ 范围注解使我们能够拥有每个组件的依赖性的单个实例。 这里的要点是，我们应该只在需要的时候使用 Scope， 比如沉重的可变对象。 不必要的范围界定只会导致开销。 默认情况下，要么范围要么只在需要时添加范围:)

这带着我们进入下一个注解。

## @Reusable

可重用范围介于无范围和范围实例之间。根据 [Dagger 用户指南](https://google.github.io/dagger/users-guide.html#reusable-scope)，这个范围可以在以下情况下使用: 

> 有时候，你想要限制一个 @Inject 构造类被实例化的次数，或者一个 @Provides 方法被调用的次数，但是你不需要保证在任何特定组件或子组件的生命周期中使用完全相同的实例。 这在分配可能很昂贵的 Android 环境中会很有用。

@Reusable 不与任何单个组件或子组件绑定 。 如果你的组件中有一个可重用的依赖项，但实际上它只在子组件中使用，那么你的子组件将缓存该实例。 因此，多个子组件可以最终缓存你的依赖。 

因此，它的意思是用于那些不是严格的 Singleton 实例，但是重新使用它们是很好的，因为初始化它们的成本相对较高。 而且，使用不可变对象使用它们是安全的，我们可以通过将可变对象标记为 @Reusable 来破坏可变对象的状态。 

举个例子：

```java
@Component(modules = ReusableModule.class)
public interface ReusableComponent {
    SomeObject someObject();
}
```

这是我们的 SomeObject 类：

```java
@Reusable
public class SomeObject {
  
    @Inject
    public SomeObject() {
    }
}
```

这里是 DaggerAppComponent 的初始化方法：

```java
Provider<SomeObject> someObjectProvider;

private void initialize(final Builder builder) {

  this.someObjectProvider = 
        SingleCheck.provider(SomeObject_Factory.create());
}
```

这次 Dagger 包装 SingleCheck 提供程序中的 Factory \<SomeObject>。 我们仍然可以在任何可能的情况下重新使用我们的 SomeObject，同时也避免了 @Singletons 的开销，而且这些依赖可以在内存不足时被收集。

这里有一个很好的 Android [对话集](https://www.youtube.com/watch?v=KwRXQ6nT7jQ)， 是 Mike 关于 Dagger2 的优化技巧，他触及 @Reusable 。感兴趣可以一探究竟！

## Scopes 和 Components （释放野兽）

上述概念不仅适用于组件，还适用于子组件和依赖组件。 每当我们创建一个新的子组件或者一个依赖组件时，我们都必须创建一个新的范围，或者我们可以说当我们想要限制我们的一些依赖的范围时，我们就创建一个子组件或者一个依赖组件。 范围规则仍然适用于那里。 如果我们在子组件中调用依赖项，它将在该组件内部成为"单例"。 如果我们使用相同的组件实例来注入它，我们将得到相同的缓存实例。 要了解更多关于范围和组件关系的信息，请检查文章 [Dagger 2 : Component Relationships & Custom Scopes](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37) ) 

## 结束

- 仔细使用范围，不要盲目地用界定所有的依赖关系。 （就像我不久前做的一样）
-  使用 Singleton ，只有当你真的需要在你的整个生命周期中缓存 Singleton 的时候，就像那些可变的沉重对象一样。
- 否则，更推荐用 Reusable 界定不可变沉重对象。

非常感谢 [Mike Nakhimovich](https://medium.com/@theMikhail) 审查这篇文章！还有 [Ritesh Gupta](https://medium.com/@_riteshhh) 和 [Sergii Zhuk](https://medium.com/@sergii)，让这一切变得不那么疯狂。 

## TL; DR

系列的文章列表：这里是文章的链接：

- [Dagger 2 : Component.Builder](https://medium.com/@ragdroid/dagger-2-component-builder-1f2b91237856)
- [Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)
- [Dagger 2 : Component Relationships & Custom Scopes](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37)
- [Dagger 2 Annotations : @Binds & @ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f)

可以观看视频：[[Video]](https://www.youtube.com/watch?v=iczp_toHxmA)