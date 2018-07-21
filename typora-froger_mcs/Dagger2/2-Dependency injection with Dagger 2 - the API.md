#  Dagger 2 的依赖注入 - API

> 原文 (mirekstanek.online ：[Dependency injection with Dagger 2 - the API](https://mirekstanek.online/dependency-injection-with-dagger-2-the-api/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

这篇文章是一系列文章的一部分，它们展示了安卓系统和 Dagger 2 的依赖注入。 今天，我想深入探讨一下 Dagger 2 的基本原理，并通过这个依赖注入框架的整个 API。

## Dagger 2
在我的上一篇文章中，我提到了 DI 框架给我们提供了什么——他们把所有的内容使用尽可能少的代码一起连接起来。Dagger 2 是 DI 框架的一个例子，它为我们生成了大量的模板。但是为什么它比其他的更好呢？现在它是唯一一个可以生成完全可追踪的源代码的 DI 框架，它模仿了用户可以手写的代码。这意味着在依赖关系图构造中没有魔法。 Dagger 2 的动态性不如其他的(完全没有反射使用) ，但生成的代码的简单性和性能与手写代码处于同一水平。


### Dagger 2 基本原理
这是 Dagger 2 的 API：
```java
public @interface Component {
    Class<?>[] modules() default {};
    Class<?>[] dependencies() default {};
}

public @interface Subcomponent {
    Class<?>[] modules() default {};
}

public @interface Module {
    Class<?>[] includes() default {};
}

public @interface Provides {
}

public @interface MapKey {
    boolean unwrapValue() default true;
}

public interface Lazy<T> {
    T get();
}
```
此外，[JSR-330](https://jcp.org/en/jsr/detail?id=330) (Java 中的依赖注入标准)还定义了其他元素，这些元素被用于 Dagger 2 :

```java
public @interface Inject {
}

public @interface Scope {
}

public @interface Qualifier {
}
```
们把它们全部检查一遍。

## @Inject 注解
首先，对于 DI 最重要的是 @Inject 注解。 JSR-330 标准的一部分，标记了应该由依赖注入框架提供的那些依赖关系。 在 Dagger 2 中有三种提供依赖关系的方法:

#### 构造函数注入

@Inject 用于构造函数：
```java
public class LoginActivityPresenter {
    
    private LoginActivity loginActivity;
    private UserDataStore userDataStore;
    private UserManager userManager;
    
    @Inject
    public LoginActivityPresenter(LoginActivity loginActivity,
                                  UserDataStore userDataStore,
                                  UserManager userManager) {
        this.loginActivity = loginActivity;
        this.userDataStore = userDataStore;
        this.userManager = userManager;
    }
}
```
所有参数都取自依赖关系图。 在结构体中使用的 @Inject 注解也使这个类成为依赖关系图的一部分。 这意味着它也可以在需要的时候被注入，比如:

```java
public class LoginActivity extends BaseActivity {

    @Inject
    LoginActivityPresenter presenter;
    
    //...
}
```
这种情况下的局限性在于我们不能用 @Inject 在类中注解多个构造函数。

#### 字段注入

另一种选择是使用 @Inject 注释特定的字段：

```java
public class SplashActivity extends AppCompatActivity {
    
    @Inject
    LoginActivityPresenter presenter;
    @Inject
    AnalyticsManager analyticsManager;
    
    @Override
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        getAppComponent().inject(this);
    }
}
```
但在这种情况下，注入过程必须“手工”调用，在我们类的某个地方：
```java
public class SplashActivity extends AppCompatActivity {
    
    //...
    
    @Override 
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        getAppComponent().inject(this);    //在这个时候注入要求的依赖
    }
}
```
在这个调用之前，我们的依赖项是空值。
字段注入的局限性在于它们不能私有。为什么？简而言之，生成的代码通过显式调用填充字段，如下所示：

```java
//这个类是由 Dagger 2 自动生成的
public final class SplashActivity_MembersInjector implements MembersInjector<SplashActivity> {

    //...

    @Override
    public void injectMembers(SplashActivity splashActivity) {
        if (splashActivity == null) {
            throw new NullPointerException("Cannot inject members into a null reference");
        }
        supertypeInjector.injectMembers(splashActivity);
        splashActivity.presenter = presenterProvider.get();
        splashActivity.analyticsManager = analyticsManagerProvider.get();
    }
}
```

#### 方法注入
用 @Inject 提供依赖的最后一种方法是对该类的公共方法进行注释:

```java
public class LoginActivityPresenter {
    
    private LoginActivity loginActivity;
    
    @Inject 
    public LoginActivityPresenter(LoginActivity loginActivity) {
        this.loginActivity = loginActivity;
    }

    @Inject
    public void enableWatches(Watches watches) {
        watches.register(this);    //Watches 实例需要完全构建的 LoginActivityPresenter
    }
}
```
所有的方法参数都是从依赖关系图提供。 但为什么我们需要方法注入？ 当我们希望将类实例本身(this 引用)传递给被注入的依赖项时，它将起作用。方法注入是在构造函数调用之后立即调用的，所以这意味着我们正在完全构造 this。

## @Module 注解
@Module 是 Dagger 2 API 的一部分。这个注解用来标记提供依赖关系的类 - 由于这个 Dagger 会知道在哪个地方构建了需要的对象。

```java
@Module
public class GithubApiModule {
    
    @Provides
    @Singleton
    OkHttpClient provideOkHttpClient() {
        OkHttpClient okHttpClient = new OkHttpClient();
        okHttpClient.setConnectTimeout(60 * 1000, TimeUnit.MILLISECONDS);
        okHttpClient.setReadTimeout(60 * 1000, TimeUnit.MILLISECONDS);
        return okHttpClient;
    }

    @Provides
    @Singleton
    RestAdapter provideRestAdapter(Application application, OkHttpClient okHttpClient) {
        RestAdapter.Builder builder = new RestAdapter.Builder();
        builder.setClient(new OkClient(okHttpClient))
               .setEndpoint(application.getString(R.string.endpoint));
        return builder.build();
    }
}
```

## @Provides 注解
这个注解被用在 @Module 类中。 @Provides 将在返回依赖项的模块中标记这些方法。

```java
@Module
public class GithubApiModule {
    
    //...
    
    @Provides   //This annotation means that method below provides dependency
    @Singleton
    RestAdapter provideRestAdapter(Application application, OkHttpClient okHttpClient) {
        RestAdapter.Builder builder = new RestAdapter.Builder();
        builder.setClient(new OkClient(okHttpClient))
               .setEndpoint(application.getString(R.string.endpoint));
        return builder.build();
    }
}
```

## @Component 注解
这个注解用来构建将所有东西连接在一起的接口。 在这个地方，我们定义从哪个模块（或其他组件）中取出依赖关系。 此处还有定义关系图的哪些依赖性应公开可见（可以被注入）以及组件可以注入对象的地方。一般来说 @Component 是 @Module 和 @Inject 之间的桥梁。

使用两个模块的 @Component 示例源代码可以将依赖关系注入到 GithubClientApplication 中，并公开可见了三个依赖关系：
```java
@Singleton
@Component(
    modules = {
        AppModule.class,
        GithubApiModule.class
    }
)
public interface AppComponent {

    void inject(GithubClientApplication githubClientApplication);

    Application getApplication();

    AnalyticsManager getAnalyticsManager();

    UserManager getUserManager();
}
```
另外 @Component 可以依赖于另一个组件，并且已经定义了生命周期（我将在未来的一个文章中写关于范围界定的文章）：

```java
@ActivityScope
@Component(      
    modules = SplashActivityModule.class,
    dependencies = AppComponent.class
)
public interface SplashActivityComponent {
    SplashActivity inject(SplashActivity splashActivity);

    SplashActivityPresenter presenter();
}
```

## @Scope 注解
```java
@Scope
public @interface ActivityScope {
}
```
JSR-330 标准的另一部分。 在 Dagger2 @Scope 是用来定义自定义范围注解。 简而言之，它们使依赖关系与单例类似。带注解的依赖关系是单个实例，但与组件生命周期（而不是整个应用程序）相关。 但正如我所说的那样，我们将在下一篇文章中深入研究。 现在值得一提的是，所有的自定义范围（从代码角度来看）都是一样的 - 它们保存对象的单个实例。同时，它们也被用于关系图验证过程，有助于尽快捕捉关系图结构问题。

>现在有一些不那么重要的，并不总是使用的东西:

## @MapKey 注解
这个注解用来定义依赖集合（现在的 map 和 set）。示例代码应该自我解释:
**定义**

```java
@MapKey(unwrapValue = true)
@interface TestKey {
    String value();
}
```
**提供依赖关系**
```java
@Provides(type = Type.MAP)
@TestKey("foo")
String provideFooKey() {
    return "foo value";
}

@Provides(type = Type.MAP)
@TestKey("bar")
String provideBarKey() {
    return "bar value";
}
```
**用法**
```java
@Inject
Map<String, String> map;

map.toString() // => „{foo=foo value, bar=bar value}”
```
@MapKey 注解现在只支持两种类型的键 - 字符串和枚举。

## @Qualifier 注解
@Qualifier 注解帮助我们为具有相同接口的依赖项创建 “标签”。 想象一下，你需要提供两个 RestAdapter 对象 - 一个用于 Github API，另一个用于 Facebook API。 限定符将帮助你确定正确的一个：
**命名依赖关系**
```java
@Provides
@Singleton
@GithubRestAdapter  //Qualifier
RestAdapter provideRestAdapter() {
    return new RestAdapter.Builder()
        .setEndpoint("https://api.github.com")
        .build();
}

@Provides
@Singleton
@FacebookRestAdapter  //Qualifier
RestAdapter provideRestAdapter() {
    return new RestAdapter.Builder()
        .setEndpoint("https://api.facebook.com")
        .build();
}
```
**注入依赖关系**
```java
@Inject
@GithubRestAdapter
RestAdapter githubRestAdapter;

@Inject
@FacebookRestAdapter
RestAdapter facebookRestAdapter;
```
就这样。我们刚刚了解了 Dagger 2 API 的所有重要元素。

## 应用示例 
现在是时候在实践中检查我们的知识了。我们将实现简单的 Github 客户端应用程序，它建立在 Dagger 2 之上。

**这个想法**
我们的 Github 客户端有三个活动和非常简单的用例。其整个流程： 
- 输入 Github 用户名 
- 如果用户存在显示公共存储库的整个列表 
- 在用户按下列表项目后显示资料库详细信息。

以下是我们的应用程序的样子：
![](http://frogermcs.github.io/images/14/app_flow.png)
从 DI 角度来看，我们的应用程序架构如下所示：
![](http://frogermcs.github.io/images/14/local_components.png)简而言之 - 每个活动都有自己的依赖关系图。 每个图（_Component 类）有两个对象 - _Presenter 和 _Activity。 此外，每个组件都从全局组件 AppComponent 中获取依赖关系，其中包含 Application，UserManager，RepositoriesManager 等。
![](http://frogermcs.github.io/images/14/app_component.png)
说到 AppComponent - 只要看看这个接口。 它包含两个模块：AppModule 和 GithubApiModule。
GithubApiModule 提供了一些依赖，比如 OkHttpClient 或 RestAdapter，它们只在这个模块的其他依赖项中使用。在 Dagger 2 中，我们可以控制哪些对象在组件外部可见。在我们的例子中，我们不想暴露提到的对象。相反，我们只是公开 UserManager 和 RepositoriesManager，因为在我们的活动中只使用那些对象。所有这些都是由公共方法定义的，这些方法返回非 void 类型，并且没有参数。

**文档中的例子：**
提供方法

```java
SomeType getSomeType();
Set<SomeType> getSomeTypes();
@PortNumber int getPortNumber();
```
此外，我们还必须定义要在哪里注入依赖关系（通过成员注入）。 在我们的例子中，AppComponent 没有注入任何地方，因为它只是作为我们的 scoped 组件的依赖。并且他们每个都定义了  `inject(_Activity activity)` 方法。此外，我们还有一个简单的规则 - 注入被定义为具有单参数的方法(定义我们想要注入依赖项的实例) ，不管名称，但必须返回 void 或传递的参数类型。

**文档中的例子：**
成员注入方法

```java
SomeType getSomeType();
Provider<SomeType> getSomeTypeProvider();
Lazy<SomeType> getLazySomeType();
```

## 实现
我不会深究什么源代码。相反，只需克隆 [GithubClient](https://github.com/frogermcs/GithubClient) 代码并将其导入到最新的 Android Studio 中即可。这里有一些提示，从哪里开始：

## Dagger 2 安装
只要检查 /app/build.gradle 文件。我们所需要做的就是添加 Dagger 2 依赖，并使用 [android-apt](https://bitbucket.org/hvisser/android-apt) 插件在生成的代码和 Android Studio IDE 之间进行一些绑定。

**AppComponent**
从 GithubClientApplication 类开始探索 GithubClient 项目。 在这个地方 AppComponent 被创建和存储。  这意味着只要应用程序对象存在，所有单实例对象都将存在。

由 Dagger 2 生成的代码提供了 AppComponent 实现（可以从 builder 模式创建对象： `Dagger{ComponentName}.builder()`)。 同时，这也是我们必须将所有组件的依赖(模块和其他组件)放在一起。

**Scoped components**

要检查如何创建 Activies 组件，只需从 SplashActivity 开始探索。 它覆盖 setupActivityComponent（AppComponent）创建他自己的组件（SplashActivityComponent）并注入所有 @Inject 注释的依赖关系（在这种情况下为 SplashActivityPresenter 和 AnalyticsManager）。

同样在这个地方，我们提供了 AppComponent 实例（因为 SplashActivityComponent 依赖于它）以及 SplashActivityModule（它提供了 Presenter 和 Activity 实例）。

剩下的就看你自己了。 说真的——试着弄清楚所有事情是如何相互配合的。 在下一篇文章中，我们将试图仔细研究 Dagger 2 元素(它们如何在幕后工作)。

## 示例代码
所描述项目的完整源代码在 [Github](https://github.com/frogermcs/GithubClient/tree/1bf53a2a36c8a85435e877847b987395e482ab4a) 存储库中获取。

