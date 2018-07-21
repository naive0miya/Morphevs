# Dagger 2 的依赖注入 - DI 介绍

> 原文 (mirekstanek.online) ：[Dependency injection with Dagger 2 - Introduction to DI](https://mirekstanek.online/dependency-injection-with-dagger-2-introduction-to-di/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

前段时间，[Tech Space](http://gototech.space/) 在 Cracow 的 Google I / O 2015 Extended 中，我用 Dagger 2 做了关于依赖注入的[演讲](https://speakerdeck.com/frogermcs/dependency-injection-with-dagger-2)。在准备的时候，我意识到有很多事情可以谈论，而且没有机会用十几张幻灯片来讲述所有的事情。但这可能是一个很好的切入点，可以启动新系列的安卓依赖注入。 

在这篇文章中，我想通过这篇文章来总结我的演讲。也许不是一步一步的 - 我认为是时候打破过去，而不是回到没有或不应该使用的解决方案。Jake Wharton [谈到了](https://www.parleys.com/tutorial/5471cdd1e4b065ebcfa1d557/)历史（Guice，Dagger 1），[Gregory Kick](https://www.youtube.com/watch?v=oK_XtfXPkqw)（一半演讲的几乎是关于 Spring，Guice，Dagger 1）。 我也花了几分钟谈论以前的解决方案。 但是时候从我们现在的位置开始。

## 依赖注入

依赖注入就是构造对象，并将它们传递到需要它们的地方。我不会深入理论（请参阅维基百科的 [DI 定义](http://en.wikipedia.org/wiki/Dependency_injection)）。 相反，想象一下简单的类：UserManager 有两个依赖 UserStore 和 ApiService。 没有依赖注入这个类看起来像这样：
![|center](http://frogermcs.github.io/images/13/user_manager_no_di.png)
UserStore 和 ApiService 都是在 UserManager 类中构建和提供的：

```java
class UserManager {
    
    private ApiService apiService;
    private UserStore userStore;

    //无参数构造函数。内部创建依赖关系。 
    public UserManager() {
        this.apiService = new ApiSerivce();
        this.userStore = new UserStore();
    }

    void registerUser() {/*  */}

}

class RegisterActivity extends Activity {

    private UserManager userManager;

    @Override
    protected void onCreate(Bundle b) {
        super.onCreate(b);
        this.userManager = new UserManager();
    }

    public void onRegisterClick(View v) {
        userManager.registerUser();
    }
}
```
为什么这个代码有点问题？ 让我们设想一下，你想要更改 UserStore 实现，并使用 SharedPreferences 作为存储机制。 它至少需要 Context 对象来创建一个实例，所以它应该通过构造函数传递给我们的 UserStore。 这意味着 UserManager 类也必须更新以处理新的 UserStore 构造函数。 现在想象一下使用 UserStore 的十几个类 - 所有这些类都必须被更新。

现在看看我们的 UserManager 类使用了依赖注入：
![](http://frogermcs.github.io/images/13/user_manager_di.png)
它的依赖从外部创建和提供：
```java
class UserManager {

    private ApiService apiService;
    private UserStore userStore;

    //依赖关系作为参数传递
    public UserManager(ApiService apiService, UserStore userStore) {
        this.apiService = apiService;
        this.userStore = userStore;
    }

    void registerUser() {/*  */}

}

class RegisterActivity extends Activity {

    private UserManager userManager;

    @Override
    protected void onCreate(Bundle b) {
        super.onCreate(b);
        ApiService api = ApiService.getInstance();
        UserStore store = UserStore.getInstance();
        
        this.userManager = new UserManager(api, store);
    }

    public void onRegisterClick(View v) {
        userManager.registerUser();
    }

}
```
现在在类似的情况下 - 我们正在改变它的一个依赖关系的实现 - 不需要更新 UserManager 源代码。 它的所有依赖关系都是从外部提供的，所以我们必须更新代码中的唯一位置就是我们构造 UserStore 对象的地方。

那么依赖注入使用的优点是什么？

### 构造/使用分离

我们正在构造一个类的实例 - 通常在其他地方使用这些对象。由于这种方法，我们的代码更加模块化 - 所有的依赖关系都可以用简单的方式替换（只要它们具有相同的接口），而不会影响我们应用程序的逻辑。 想要将 DatabaseUserStore 更改为 SharedPrefsUserStore？ 好的，只要注意公共 API（与 DatabaseUserStore 相同）或者只是实现相同的接口。

### 单元测试

真正的单元测试假设该类必须完全隔离的情况下测试 - 不知道它的依赖关系。实际上，基于我们的 UserManager 类，下面是我们要编写的单元测试的一个例子：

```java
public class UserManagerTests {

    UserManager userManager;

    @Mock
    ApiService apiServiceMock;
    @Mock
    UserStore userStoreMock;

    @Before
    public void setUp() {
        MockitoAnnotations.initMocks(this);
        userManager = new UserManager(apiServiceMock, userStoreMock);
    }

    @After
    public void tearDown() {
    }

    @Test
    public void testSomething() {
        // 在这里测试我们的 userManager - 它的所有依赖关系都满足了
    }
}
```
这是可能的，只使用了 DI - 这要归功于 UserManager 完全独立于 UserStore 和 ApiManager 实现。 我们可以提供这些类的模拟（简而言之，模拟是具有相同公共 API 的类，它在方法调用和/或返回我们期望的值时不做任何事情），并且将 UserManager 与其依赖关系的实际实现隔离开来。

### 独立/并行开发

由于代码模块化（UserStore 可以独立于 UserManager 实现）很容易在代码之间分割代码。 只有 UserStore 的接口必须被大家知道（特别是 UserManager 中使用的 UserStore 的公共方法）。 其余的（实现，逻辑）可以通过单元测试来测试。

## 依赖注入框架
除了优势之外，依赖注入模式也有一些缺点。其中之一是具有很大的样板。试想一下用 MVP（model-view-presenter）模式实现的简单的 LoginActivity 类。这个类可能是这样的：
![|center](http://frogermcs.github.io/images/13/login_activity_diagram.png)

只负责 LoginActivityPresenter 初始化的源代码如下所示：
```java
public class LoginActivity extends AppCompatActivity {

    LoginActivityPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        OkHttpClient okHttpClient = new OkHttpClient();
        RestAdapter.Builder builder = new RestAdapter.Builder();
        builder.setClient(new OkClient(okHttpClient));
        RestAdapter restAdapter = builder.build();
        ApiService apiService = restAdapter.create(ApiService.class);
        UserManager userManager = UserManager.getInstance(apiService);
        
        UserDataStore userDataStore = UserDataStore.getInstance(
                getSharedPreferences("prefs", MODE_PRIVATE)
        );

        //Presenter is initialized here
        presenter = new LoginActivityPresenter(this, userManager, userDataStore);
    }
}
```
看起来不那么友好，不是吗？

这就是依赖注入框架解决的问题。使用它们的相同代码看起来像这样：
```java
public class LoginActivity extends AppCompatActivity {

    @Inject
    LoginActivityPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        //满足请求的所有依赖项，使用 @Inject 注解。
        getDependenciesGraph().inject(this);
    }
}
```
简单多了，对吧？当然，DI 框架不会从任何地方获取对象 - 它们仍然需要在我们的代码中进行初始化和配置。 但是对象的构造与使用是分开的（实际上这是 DI 模式的前提）。 DI 框架关心如何将所有东西连接在一起（如何将对象发送到请求的地方）。

## 未完待续

我上面描述的一切都是 Dagger 2-依赖注入框架的一个轻微背景，可以在 Android 和 Java 开发中使用。在下一篇文章中，我将尝试通过整个 Dagger 2 API。 如果你不想等的话，可以尝试一下我的 [Github 客户端的例子](https://github.com/frogermcs/GithubClient)，这个例子是在 Dagger 2 的基础上构建的，并且和我的演讲一起使用。 只是一个提示 - @Modules 和 @Components 是构造/提供对象的地方。 @Inject 是我们的对象使用的地方。

更详细的描述 - 很快到来。

## 阅读
- [DAGGER 2 - A New Type of dependency injection ](DAGGER 2 - A New Type of dependency injection ) 出自 Gregory Kick
- [The Future of Dependency Injection with Dagger 2  ](https://www.parleys.com/tutorial/the-future-dependency-injection-dagger-2) 出自 Jake Wharton
- [Dagger 1 to 2 migration process ](http://frogermcs.github.io/dagger-1-to-2-migration/)






