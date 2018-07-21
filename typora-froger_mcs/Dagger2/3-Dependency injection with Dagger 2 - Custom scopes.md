# Dagger 2 的依赖注入 - 自定义范围

> 原文 (mirekstanek.online) ：[Dependency injection with Dagger 2 - Custom scopes](https://mirekstanek.online/dependency-injection-with-dagger-2-custom-scopes/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

这篇文章是在 Android 中显示依赖注入与 Dagger 2 框架的系列文章的一部分。今天我将花费一些时间来使用自定义范围，对于初学者来说，这些功能可能有点问题。

## 范围 - 它给了我们什么？
几乎每个项目都使用单例 - 对于 API 客户端，数据库帮助器，分析管理器等。但是由于我们不关心实例化（因为依赖注入框架），所以我们不应该考虑如何在代码中获取这些对象。 而是 @Inject 注释应该为我们提供理想的实例。

在 Dagger 2 范围机制中，只要存在范围，就关系到保存单个类的实例。 实际上，这意味着 @ApplicationScope 中的实例与 Application 对象一样长。 只要存在 Activity，@ActivityScope 就会保留引用（例如，我们可以在此 Activity 中托管的所有片段之间共享任何类的单个实例）。

在短的范围，我们可以看到"局部单例"，它们的寿命与范围本身一样长。

只是要清楚 - 在 Dagger 2 中默认没有提供 @ActivityScope 或/和 @ApplicationScope 注解。这是自定义范围的最常见的用法。只有 @Singleton 范围是默认可用的（由 Java 本身提供）。

## 范围 - 实际的例子
为了更好的理解 Dagger 2 的范围，我们直接看实际例子。 我们要实现一些比 Application / Activity 范围更复杂的事情。 为此，我们将使用[前一篇](http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/)文章中的 [GithubClient](https://github.com/frogermcs/GithubClient) 示例。 我们的应用程序应该有三个范围：
- @Singleton - 应用程序范围
- @UserScope - 与已选用户关联的类实例的范围（在真实应用中，可以是已登录的用户）
- @ActivityScope（活动范围） - 与活动一样长的实例的范围（在本例中为演示者）

引入 @UserScope 是今天的解决方案和前一篇文章的主要区别。 从用户体验的角度来看，它没有提供任何东西，但是从体系结构的角度来看，它有助于我们提供 User 实例而不将其作为一个意图参数。 另外在方法参数（RepositoriesManager）中需要用户数据的类可以将 User 实例作为构造函数参数（将从依赖关系图提供），可以根据需要初始化，而不是在应用程序启动时创建它。这意味着在从 Github API 获取用户之后（在呈现 RepositoriesListActivity 之前一段时间），将创建 RepositoriesManager。

这里是我们的应用程序提供的范围和组件的简单可视化。
![|center](http://frogermcs.github.io/images/15/dagger-scopes.png)
单例（应用范围）是最长的生存范围（在实践中，寿命和应用程序一样长）。UserScope 作为 Application 范围的子范围，可以访问其对象（我们总是可以从父范围获取对象）。 与 ActivityScope（寿命与 Activity 对象一样长）一样 - 它可以从 UserScope 和 ApplicationScope 中获取对象。

## 示例范围生命周期
这里是我们的应用程序范围的示例生命周期：
![](http://frogermcs.github.io/images/15/scopes-lifecycle.png)
从应用程序启动开始，所有的单例都会创建，而当我们从 Github API 获得 User 实例（真正应用程序中是在用户登录后）创建 UserScope，而当我们回到 SplashActivity 时（真正应用程序中是在用户注销后）。使用新的用户，将创建另一个 UserScope。每个 ActivityScope 都存在，只要它的 Activity 对象还存活。

## 实现
在 Dagger 2 范围内，实现归结为组件的适当配置。 一般来说，我们有两种方式来做到这一点 - 使用 @Subcomponent 注解或组件依赖。它们之间的主要区别是对象图的共享。子组件可以从其父母访问整个对象图，而组件依赖性只允许访问在 Component 接口中公开的那些对象。

我选择第一种方式 @Subcomponent 注解。如果使用 Dagger 1，它几乎与从 ObjectGraph 创建子图一样。此外，我们将使用类似的命名约定创建子图的方法（但不是强制性的）。

我们从 AppComponent 实现开始：
```java
@Singleton
@Component(
        modules = {
                AppModule.class,
                GithubApiModule.class
        }
)
public interface AppComponent {

    UserComponent plus(UserModule userModule);

    SplashActivityComponent plus(SplashActivityModule splashActivityModule);

}
```
它将成为其他子组件的根：UserComponent 和 Activities 组件。 正如您可能已经注意到的那样（特别是如果您从上一篇文章中看到 [AppComponent 实现](https://github.com/frogermcs/GithubClient/blob/1bf53a2a36c8a85435e877847b987395e482ab4a/app/src/main/java/frogermcs/io/githubclient/AppComponent.java)），从依赖关系图返回对象的所有公共方法都消失了。 由于我们有子组件，因此我们不需要公开公开依赖关系 - 子图无论如何都可以访问它们。

相反我们添加了两个方法：
- UserComponent plus(UserModule userModule);
- SplashActivityComponent plus(SplashActivityModule splashActivityModule);

他们的意思是，从我们的 AppComponent，我们将能够创建两个子组件：UserComponent 和 SplashActivityComponent。由于它们是 AppComponent 的子组件，因此它们都可以访问由 AppModule 和 GithubApiModule 生成的实例。

这种方法的命名约定是：返回类型是子组件类，方法名是任意的，参数是在这个子组件中需要的模块。

正如你可以看到 UserComponent 需要另一个模块（作为 plus（）方法的参数传递）。这样做，我们将通过新模块生成的附加对象来扩展 AppComponent 图。UserComponent 类如下所示：

```java
@UserScope
@Subcomponent(
        modules = {
                UserModule.class
        }
)
public interface UserComponent {
    RepositoriesListActivityComponent plus(RepositoriesListActivityModule repositoriesListActivityModule);

    RepositoryDetailsActivityComponent plus(RepositoryDetailsActivityModule repositoryDetailsActivityModule);
}
```
当然 @UserScope 注解是由我们创建的：
```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface UserScope {
}
```
从 UserComponent 中，我们可以创建另外两个子组件：RepositoriesListActivityComponent 和 RepositoryDetailsActivityComponent。
```java
@UserScope
@Subcomponent(
        modules = {
                UserModule.class
        }
)
public interface UserComponent {
    RepositoriesListActivityComponent plus(RepositoriesListActivityModule repositoriesListActivityModule);

    RepositoryDetailsActivityComponent plus(RepositoryDetailsActivityModule repositoryDetailsActivityModule);
}
```

更重要的是，所有的范围界定都发生在这里。从继承自 AppComponent 的 UserComponent 获取的所有实例仍然是 singletons (在应用程序范围中)。但是，由 UserModule（它是 UserComponent 的一部分）生成的将是“局部单例”，它与此 UserComponent 实例的存活时间一样长。
所以每次我们通过调用下面的方法创建另一个 UserComponent 实例：

```java
UserComponent appComponent = appComponent.plus(new UserModule(user))
```

从 UserModule 中获取的对象是不同的实例。

但重要的是，我们要负责 UserComponent 的生命周期。 因此，我们应该关注它的初始化和释放。 在我们的应用程序示例中，我创建了两个额外的方法:

```java
public class GithubClientApplication extends Application {

    private AppComponent appComponent;
    private UserComponent userComponent;

    //...

    public UserComponent createUserComponent(User user) {
        userComponent = appComponent.plus(new UserModule(user));
        return userComponent;
    }

    public void releaseUserComponent() {
        userComponent = null;
    }

    //...
}
```
当我们从 Github API（SplashActivity）获取 User 对象时，调用 createUserComponent（）。当我们来自 RepositoriesListActivity 时，调用 releaseUserComponent（）（此时我们不再需要用户范围了）。

## Dagger 2 范围 - 背后的原理

看看事情是如何运作的总是好的。 事实上，在这种情况下，确保没有魔法在 Dagger 的范围机制。

我们将从 UserModule.provideRepositoriesManager（）方法开始我们的调查。它提供了 RepositoriesManager 实例，该实例应该在 @UserScope 范围中。让我们来看看这个方法被调用的地方（第8行）

```java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class UserModule_ProvideRepositoriesManagerFactory implements Factory<RepositoriesManager> {

  //...
  
  @Override
  public RepositoriesManager get() {  
    RepositoriesManager provided = module.provideRepositoriesManager(userProvider.get(), githubApiServiceProvider.get());
    if (provided == null) {
      throw new NullPointerException("Cannot return null from a non-@Nullable @Provides method");
    }
    return provided;
  }

  public static Factory<RepositoriesManager> create(UserModule module, Provider<User> userProvider, Provider<GithubApiService> githubApiServiceProvider) {  
    return new UserModule_ProvideRepositoriesManagerFactory(module, userProvider, githubApiServiceProvider);
  }
}
```
UserModule_ProvideRepositoriesManagerFactory 是一个 Factory 模式的实现，它从 UserModule 获取 RepositoriesManager 实例。我们应该进一步挖掘。

UserModule_ProvideRepositoriesManagerFactory 用于 UserComponentImpl - 实现我们的组件（第15行）：
```java
private final class UserComponentImpl implements UserComponent {

    //...

    private UserComponentImpl(UserModule userModule) {
      if (userModule == null) {
        throw new NullPointerException();
      }
      this.userModule = userModule;
      initialize();
    }

    private void initialize() {
      this.provideUserProvider = ScopedProvider.create(UserModule_ProvideUserFactory.create(userModule));
      this.provideRepositoriesManagerProvider = ScopedProvider.create(UserModule_ProvideRepositoriesManagerFactory.create(userModule, provideUserProvider, DaggerAppComponent.this.provideGithubApiServiceProvider));
    }

    //...
    
}
```
provideRepositoriesManagerProvider 对象负责在每次请求时提供 RepositoriesManager 实例。正如我们所见，提供者是由 ScopedProvider 实现的。看看它的一部分源代码：
```java
public final class ScopedProvider<T> implements Provider<T> {
  
  //...

  private ScopedProvider(Factory<T> factory) {
    assert factory != null;
    this.factory = factory;
  }

  @SuppressWarnings("unchecked") // cast only happens when result comes from the factory
  @Override
  public T get() {
    // double-check idiom from EJ2: Item 71
    Object result = instance;
    if (result == UNINITIALIZED) {
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          instance = result = factory.get();
        }
      }
    }
    return (T) result;
  }

  //...

}
```
还能再简单点吗？在第一次调用 ScopedProvider 从工厂（在我们的例子中是UserModule_ProvideRepositoriesManagerFactory）获取实例，并像 Singleton 模式一样存储这个 object。 而我们的作用域提供者只是 UserComponentImpl 中的一个字段，简而言之，这意味着只要 Component 存在， ScopedProvider 就返回单个实例。

在这里你可以找到 [ScopedProvider](https://github.com/google/dagger/blob/master/core/src/main/java/dagger/internal/ScopedProvider.java) 的完整实现。

就是这样。 我们刚刚弄清楚 Dagger 2 中的范围是如何工作的。 现在我们知道，它们与范围注解没有任何联系。 自定义注解给我们提供了一个简单的方法来在编译时进行代码验证，并将类标记为单个/非单个实例。 所有范围界定都连接到组件的生命周期。

## 示例代码
所描述项目的完整源代码在 [Github](https://github.com/frogermcs/GithubClient) 存储库中获取。



