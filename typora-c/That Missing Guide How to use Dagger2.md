# 指南: 如何使用 Dagger2

> 原文 (Medium)：[That Missing Guide: How to use Dagger2](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97)
>
> 作者：[Gabor Varadi](https://medium.com/@Zhuinden?source=post_header_lockup)

[TOC]

大约2.5年前，当 Dagger 2发布的时候，我很兴奋谷歌创造了一个 fork  从 Dagger (由 Square 创建)的分支。 

原来的 dagger 是部分反射的，并没有很好地与 Proguard 一起使用。 然而，Dagger2 完全基于注解处理，所以它在编译时起到了魔力。 它不需要额外的 Proguard 配置，而且通常更快。 

虽然文件越来越好，他们仍然主要是用恒温器和加热器冲泡咖啡。 我从来没有在一个真正的应用程序中拥有这两者。 

所以 Koin 和 Kodein 以及其他随机出现的库试图声称他们是 “最简单的 DI 解决方案”，使得 “依赖注入无痛苦”，我想是时候写一个(或者另一个?) 快速指南告诉你使用 Dagger 2是多么的简单。 

### 步骤:

- 忽略 dagger-android（添加在版本2.10），你完全不需要它
- 使用 @Inject 注解的构造函数，你可以在任何地方（以及它自带的范围）
- 使用 @Component 和 @Module（当你实际需要的时候）
- 理解 @Singleton 范围（基本上“只允许创建这个东西的一个实例”），并且所有的范围都是一样的
- 只有实际上需求时才开始使用 subscopes 。  没有他们你通常可以逃脱。 不管怎样，不要因为他们而感到压力 

### 如何在你的项目中添加 Dagger

像往常一样进入你的 build.gradle 依赖关系

```groovy
annotationProcessor ‘com.google.dagger:dagger-compiler:2.11’
implementation ‘com.google.dagger:dagger:2.11’
compileOnly ‘org.glassfish:javax.annotation:10.0-b28’
```

当然，在 Kotlin 中，用 kapt 替换 annotationProcessor。

### 如何判断你是否做错了

我其实很早以前就犯了这个错误，很多指导还是犯了同样的错误。事实上，我只知道这是错的，因为有一篇[日文文章](http://blog.tai2.net/android_mvp.html)提到它，所以就是这样。

我们假设你有这个：

```java
public class MyServiceImpl implements MyService {
    private final ApiService apiService;
    public MyServiceImpl(ApiService apiService) {
        this.apiService = apiService;
    }
}
```

这就是你不要这样做：

```java
@Module
public class ServiceModule {
    @Provides
    @Singleton
    public MyService myService(ApiService apiService) {
        return new MyServiceImpl(apiService); // <-- NO
    }
}
```

假设你真的需要将一个实现绑定到一个接口上，那么这就是你要做的: 

```java
@Module
public class ServiceModule {
    @Provides
    MyService myService(MyServiceImpl service) {
        return service; // <-- YES
    }
}
@Singleton // <-- YES
public class MyServiceImpl implements MyService {
    private final ApiService apiService;
    @Inject // <-- YES
    public MyServiceImpl(ApiService apiService) { // <-- YES
        this.apiService = apiService;
    }
}
```

好的，所以还有一件事情可以轻松简化整个事情。不，我不是说使用 @Binds（它做的事情完全相同，只不过它是一个抽象的方法 ）

如果我不需要一个类的接口，因为在运行时执行时没有它的多个子类（并且可以在测试时模拟它），那么我不需要这个接口。然后我不再需要模块。

```java
@Singleton
public class MyService {
    private final ApiService apiService;
    @Inject
    public MyService(ApiService apiService) {
        this.apiService = apiService;
    }
}
```

Boom，这是 MyService 的 Dagger 配置。添加 @Singleton 和添加 @Inject，这样就可以了。 

我会说这比 Kodein 的 instance ( ) 链更好，而且是使用 provider ，对吗？

#### 一个关于接口的警告

作为对一些有效批评的回应，我必须做一个说明ーー我并不是说你应该删除你所写的每一个接口，但是你应该考虑是否真的需要这个接口。 

如果它有助于你的测试，例如配置不同的数据源，那么无论如何都要使用一个接口！ 无论如何，Kotlin 把绑定方法做成了一个单行。 

```kotlin
@Provides fun myService(impl: MyServiceImpl) : MyService = impl;
```

### 让我们来看一个真实的例子

我想下载一批猫，并在屏幕上显示一个 RecyclerView。当我向下滚动时，我想要下载更多的猫，并将它们保存在一个数据库中，至少跨越轮换。 

#### 问题: 在没有 DI 框架的情况下，一个拥有手工 DI 的世界

只要有足够的冲击力，我们就可以得到以下类别: 

```java
Context appContext;     
CatTable catTable;     
CatDao catDao;     
CatMapper catMapper;     
CatRepository catRepository;     
CatService catService;     
Retrofit retrofit;     
DatabaseManager databaseManager;     
Scheduler mainThreadScheduler;     
Scheduler backgroundThreadScheduler;
```

在我们的应用程序中，我们可以自己手动设置这些类: 

```java
this.appContext = this;
this.mainThreadScheduler = new MainThreadScheduler();
this.backgroundThreadScheduler = new BackgroundScheduler();
this.catTable = Tables.CAT;
this.databaseManager = new DatabaseManager(
                         appContext, Tables.getTables());
this.catMapper = new CatMapper();
this.retrofit = new Retrofit.Builder()
       .baseUrl("http://thecatapi.com/")
       .addConverterFactory(SimpleXmlConverterFactory.create())
       .build();
this.catService = retrofit.create(CatService.class);
this.catDao = new CatDao(catTable, catMapper, databaseManager);
this.catRepository = new CatRepository(catDao, catService, 
    catMapper, backgroundThreadScheduler, mainThreadScheduler);
```

这些只是一些简单的类来获得一批猫... 然而，这是一个很多手工配置。 

这些实例化的顺序也很重要！ 如果你在创建 dao，或者服务，或者 mapper 之前不小心尝试创建它，那么该存储库将会抛出一个 NPE。 但是只有在运行时！ 

### 使用构造函数注入 + 范围

如果我们只使用带有构造函数注入的 Dagger 2，我们实际上可以使它变得更容易。 简而言之，让我们假设 Kotlin。 

```kotlin
@Singleton
class CatMapper @Inject constructor() {
}
@Singleton
class CatDao @Inject constructor(
    private val catTable: CatTable,
    private val catMapper: CatMapper,
    private val databaseManager: DatabaseManager) {
}
@Singleton
class CatRepository @Inject constructor(
    private val catDao: CatDao,
    private val catService: CatService,
    private val catMapper: CatMapper,
    private @Named("BACKGROUND") val backgroundThread: Scheduler, 
    private @Named("MAIN") val mainThread: Scheduler) {
}
```

我们可以自由地使用我们在构造函数中收到的所有东西，它们都是可用的，并且是由外部提供的。 Dagger 会想出"怎么做"的。 

### 使用 @Module 为了那些 Dagger 搞不清楚的事情 

#### 提供没有 @Inject 注入构造函数的东西

我们可以使用构造函数注入我们自己的东西是很好的，但是那些用生成器或者其他地方建造的东西呢？ 

这就是模块的作用。

在这种情况下，Retrofit 和由 Retrofit 创建的实现。

```kotlin
@Module
class ServiceModule {
    @Provides
    @Singleton
    fun retrofit(): Retrofit = Retrofit.Builder()
       .baseUrl("http://thecatapi.com/")
       .addConverterFactory(SimpleXmlConverterFactory.create())
       .build();
    @Provides
    @Singleton
    fun catService(retrofit: Retrofit): CatService = 
         retrofit.create(CatService::class.java);
}
```

好的，现在 Dagger 知道如何提供这些东西，我们可以在构造函数注入中直接使用它们。 干净利落。 

#### 提供现有实例

假设我们也想要应用程序上下文。 我们想要提供这些，而我们并不拥有它，对吗？ 所以我们需要一个模块。 

```kotlin
@Module
class ContextModule(private val appContext: Context) {
   @Provides
   fun appContext(): Context = appContext;
}
```

还有 

```kotlin
val contextModule = ContextModule(this); // this === app
```

Boom，搞定。 

你可以使用 @BindsInstance 并手动定义你自己的 @Component.Builder 以完全相同的方式（提供外部提供的 Context）完成配置的三倍，但由于它做同样的事情并且它更复杂，所以我们不需要关心。 只需使用具有构造函数参数的模块。

#### 提供命名的实现

在我们的例子中，我们不能避免使用 BackgroundScheduler 和 MainThreadScheduler 作为 Scheduler 的实例（允许在  ImmediateScheduler 传递），所以我们也必须为它添加一个模块。

```java
@Module
class SchedulerModule {
    @Provides
    @Singleton
    @Named("BACKGROUND")
    fun backgroundScheduler(): Scheduler = BackgroundScheduler();
    @Provide
    @Singleton
    @Named("MAIN")
    fun mainThreadScheduler(): Scheduler = MainThreadScheduler();
}
```

## @Component: 让依赖成为我们关系图的东西

我们已经用构造函数注入和模块描述了如何创建事物。 现在我们需要能够创造事物的东西。 一个供应商，工厂的集合，你的名字。 它被称为 @Component，但你也可以把它称为 ObjectGraph，它会是同样的事情。

我绝对不会把它称为 [NetComponent](https://github.com/codepath/dagger2-example/blob/b2804f2c892655fff44a29fa5895b3ad7e79e711/app/src/main/java/com/codepath/daggerexample/di/components/NetComponent.java)（因为这没有任何意义），我将它称为 SingletonComponent。

它看起来像这样：

```java
@Component(modules={ServiceModule.class, 
                    ContextModule.class,
                    SchedulerModule.class})
@Singleton
public interface SingletonComponent {
    // nothing here initially
}
```

超级复杂，对吗？问题是，除非我们把任何一种方法都允许字段注入到具体类型或者提供方法中，否则我们不能从这个组件中获得任何东西。

所以你可以添加

```java
void inject(MyActivity myActivity);
```

或者

```java
Context appContext();
CatDao catDao();
...
```

就我个人而言，我认为现场注入规模相当严重，所以我更愿意为每个我需要的依赖项提供一个提供方法，然后传递我的组件。所以它看起来像这样。

```java
@Component(modules={ServiceModule.class, 
                    ContextModule.class,
                    SchedulerModule.class})
@Singleton
public interface SingletonComponent {
    Context appContext();
    CatTable catTable();
    CatDao catDao();
    CatMapper catMapper();
    CatRepository catRepository();
    CatService catService();
    Retrofit retrofit();
    DatabaseManager databaseManager();
    @Named("MAIN") Scheduler mainThreadScheduler();
    @Named("BACKGROUND") Scheduler backgroundThreadScheduler();
}
```

这可能看起来像一堆方法，但是这就是为什么 Dagger 能够在我们需要的时候找到我们的依赖关系的原因，并且仍然确保只有它们的一个实例（如果它们是有作用域的）无论如何什么。

其实还挺方便的。 

### 创建组件的实例

这是一个单例组件，为了创建一个进程级别的单例，我们可以使用 Application.onCreate ( ) 作为我们的选择。

```kotlin
class CustomApplication: Application() {
    lateinit var component: SingletonComponent;
    override fun onCreate() {
        super.onCreate();
        component = DaggerSingletonComponent.builder()
                       .contextModule(ContextModule(this))
                       .build();
    }
}
```

DaggerSingletonComponent 是由 Dagger 自己创建的，它是我们需要为适当的 DI 编写的一堆代码。该框架“写”给我们。

现在，如果我们有一个对 Application 类的引用，我们可以检索任何类，完全注入 - 或者用它来注入我们的活动/片段（由系统使用它们的无参数构造函数创建）。

#### 访问组件，并注入活动 / 片段

我喜欢做的一个技巧就是让 injector 可以从任何地方访问，即使没有应用程序实例。

```kotlin
class CustomApplication: Application() {
    lateinit var component: SingletonComponent
        private set
    override fun onCreate() {
        super.onCreate();
        INSTANCE = this;
        component = DaggerSingletonComponent.builder()
                       .contextModule(ContextModule(this))
                       .build();
    }
    companion object {
       @JvmField lateinit var INSTANCE : CustomApplication
          private set
    }
}
```

然后我可以做

```kotlin
class Injector private constructor {
    companion object {
        fun get() : SingletonComponent =
           CustomApplication.INSTANCE.component;
    }
}
```

所以现在在我的活动，我可以做

```kotlin
class CatActivity: AppCompatActivity() {
    lateinit var catRepository: CatRepository;
    init {
        this.catRepository = Injector.get().catRepository();
    }
}
```

有了这个，我们可以在我们的活动中得到任何我们想要的东西。 有人可能会说：“嘿，等待，你是依靠服务查找”，但我认为这是你可以为活动/片段做的最好/最稳定的事情。

对于你实际拥有的类来说，这不是最好的事情，那么为什么我会使用服务查找呢？ 只需使用 @Inject！

### 但是我想要一个不是单例的演讲者

那我们有三个选择：

- 创建一个无作用域的演示者，让它随着配置更改的活动/片段而死亡 
- 创建一个无作用域的演示者，并将其保留在 Activity / Fragment 的范围内 
- 创建一个有作用域的子组件和一个子作用域的演示者

有人可能会认为， 不需要在配置更改中保留演示者，只需观察由单一业务逻辑中跟踪的网络调用引起的更改，而不是演示者本身。

他们是对的！

因此，我甚至不需要深入讨论子范围界定ーー虽然我喜欢这个概念，但实际上我们可以避免使用子组件 / 组件依赖关系。 一般来说，你可以混合使用单例 scoped 和 unscoped 的东西，你可以从同一个注射器得到它们，而不需要在它们的作用范围内处理子范围的组件。 

我必须承认，使用正确的生命周期创建子范围的组件是很容易的，你只需要在 ViewModelProviders.Factory 中使用 ViewModel。

无论如何，这是一个无作用域的演示者：

```kotlin
// UNSCOPED
class CatPresenter @Inject constructor(
    private val catRepository: CatRepository 
) {
   ...
}
```

这样，我们的活动可以得到这个实例...如果我们想保留它在配置更改，这也不是很难：

```kotlin
class CatActivity: AppCompatActivity() {
    private lateinit var catPresenter: CatPresenter;
    override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState);
         val nonConfig = getLastCustomNonConfigurationInstance();
         if(nonConfig != null) {
             this.catPresenter = nonConfig as CatPresenter;
         } else {
             this.catPresenter = Injector.get().catPresenter();
         }
    }
    override fun onRetainCustomNonConfigurationInstance() =  
        catPresenter;
}
```

通过这种方式，演示者本身是没有范围的，但只有在实际需要的时候才会创建。 我们也可以使用新发布的架构组件的 ViewModel。 我们不需要为它创建一个子组件。 

### 什么是 subscopes?

如果你仍然对使用 subscope 和 @Subcomponent 以及所有这些，感兴趣的话，请随时[用 Dagger2 查看这个修改过的 Mortar 示例](https://github.com/Zhuinden/simple-stack/tree/d18f9ba44b6851dd14d5fb7bfd697072d08e6c9a/simple-stack-mortar-sample/src/main/java/com/example/mortar)。

以 Observable \<T> 的形式提供数据作为子范围的依赖关系只是非常酷。Jake Wharton 已经这样做了很多年了，我们还只是在叙旧。 

一般来说，我们可以很容易地避免使用子范围，通过保持未作用的依赖项，而不是对组件进行分析，然后保持子范围内组件的活动。 

但是对于子范围界定 ，你可以使用 @Subcomponent 和 @Component（dependencies = {...}）来进行实验，它们做同样的事情：从父级继承提供程序。

### 什么是 Dagger-Android?

虽然它允许你通过分层查找来[自动注入 Activities / Fragments](https://github.com/googlesamples/android-architecture-components/blob/6e0cd31d34782623add614703fa1abe4bb7bd575/GithubBrowserSample/app/src/main/java/com/android/example/github/di/AppInjector.java)，但它的设置非常神奇。[这不是可配置的](https://github.com/googlesamples/android-architecture-components/blob/6e0cd31d34782623add614703fa1abe4bb7bd575/GithubBrowserSample/app/src/main/java/com/android/example/github/di/FragmentBuildersModul);如果我真的想做范围确定，我个人似乎无法弄清楚如何创建范围。

所以我不使用它，我个人不建议使用它。虽然有一些教程可以使用 @ContributesXInjector 工作。

## 结束

希望这能帮助你看到使用 Dagger 2比人们谈论的噩梦般的"学习曲线"简单得多。 

这真的只是：

应用 @Singleton 和 @Inject 

创建 @Module，你不能用 @Inject 做到的

创建 @Component 并使用组件

