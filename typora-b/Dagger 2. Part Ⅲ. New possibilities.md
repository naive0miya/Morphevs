# 第三部分. 新的可能

>原文 (Medium)：[Dagger 2. Part three. New possibilities](https://proandroiddev.com/dagger-2-part-three-new-possibilities-3daff12f7ebf).
>
>作者：[Eugene Matsyuk](https://android.jlelse.eu/@eugenematsyuk?source=post_header_lockup)

[TOC]

Dagger 2 文章周期：

1. [Dagger 2. Part I. Basic principles, graph dependencies, scopes](https://android.jlelse.eu/dagger-2-part-i-basic-principles-graph-dependencies-scopes-3dfd032ccd82).
2. [Dagger 2. Part II. Custom scopes, Component dependencies, Subcomponents](https://proandroiddev.com/dagger-2-part-ii-custom-scopes-component-dependencies-subcomponents-697c1fa1cfc).
3. Dagger 2. Part three. New possibilities.

大家好！关于 Dagger 2 的第三篇文章终于到来了！

非常感谢你的阅读和评论。我很高兴这些系列文章能够帮助你深入到 Dagger 的世界。这是我继续写作的力量和动力的源泉。在第三部分中，我们将介绍你将来可能需要的各种有趣而重要的库功能。

虽然库已经存在了相当长的一段时间，但其文档仍然非常糟糕。我甚至会建议那些刚刚开始认识 Dagger 的开发者不要看官方文档，以免在这个艰难和不公正的世界中失望。

毫无疑问，有些部分或多或少地涵盖了整个过程。 然而，所有的新特性都缺乏适当的文档，因此，我不得不直接深入到生成的代码中去找出事物是如何工作的。 幸运的是，有些优秀的研究员写出了好文章，但是他们有时候也不会给出一个清晰明了的答案。 

所以，够了，让我们开始新的知识吧！ 

## 限定符注解

通常情况下，我们需要提供几个同类型的对象。 例如，我们希望系统中有两个 Executors: 一个是单线程的，另一个是 CachedThreadPool。 在这种情况下,"限定符注释"将非常方便。 这是一个有@qualifier 注释的自定义注解。 这听起来有点像"盐是咸的"，但是一切都简单多了。 

基本上，Dagger 2 已经为我们提供了一个 “限定符注解”，对于大多数情况来说，它可能已经足够了：

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {

    / The name. /
    String value() default "";
}
```

现在让我们看看它的实际效果如何：

```java
@Module
public class AppModule {

    @Provides
    @Singleton
    @Named("SingleThread")
    public Executor provideSingleThreadExecutor() {
        return Executors.newSingleThreadExecutor();
    }

    @Provides
    @Singleton
    @Named("MultiThread")
    public Executor provideMultiThreadExecutor() {
        return Executors.newCachedThreadPool();
    }

}

public class MainActivity extends AppCompatActivity {

    @Inject
    @Named("SingleThread")
    Executor singleExecutor;

    @Inject
    @Named("MultiThread")
    Executor multiExecutor;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyApplication.getInstance().getAppComponent().inject(this);
        setContentView(R.layout.activity_main);
    }
}
```

因此，我们有一个类（Executor）的两个不同的实例（singleExecutor，multiExecutor）。这正是我们需要的！请注意，具有 @Named 注释的同一类的对象也可以由完全不同的独立组件以及依赖的组件提供。

## 推迟初始化

一个常见开发问题是长时间的应用程序启动。通常有一个原因 - 我们在启动时加载过多并初始化 。此外， Dagger 2 在主线程中构建一个依赖关系图。但通常并非所有由 Dagger 提供的对象都是立即需要的。因此，库允许我们推迟对象的初始化，使用 Provider <> 和 Lazy <> 接口，直到进行第一次调用。

让我们看看下面的例子: 

```java
@Module
public class AppModule {

    @Provides
    @Named("SingleThread")
    public Executor provideSingleThreadExecutor() {
        return Executors.newSingleThreadExecutor();
    }

    @Provides
    @Named("MultiThread")
    public Executor provideMultiThreadExecutor() {
        return Executors.newCachedThreadPool();
    }

}

public class MainActivity extends AppCompatActivity {

    @Inject
    @Named("SingleThread")
    Provider<Executor> singleExecutorProvider;

    @Inject
    @Named("MultiThread")
    Lazy<Executor> multiExecutorLazy;

    @Inject
    @Named("MultiThread")
    Lazy<Executor> multiExecutorLazyCopy;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyApplication.getInstance().getAppComponent().inject(this);
        setContentView(R.layout.activity_main);
        //
        Executor singleExecutor = singleExecutorProvider.get();
        Executor singleExecutor2 = singleExecutorProvider.get();
        //
        Executor multiExecutor = multiExecutorLazy.get();
        Executor multiExecutor2 = multiExecutorLazy.get();
        Executor multiExecutor3 = multiExecutorLazyCopy.get();
    }
}
```

让我们从 Provider \<Executor> singleExecutorProvider 开始。在第一次调用 singleExecutorProvider.get ( ) 之前，Dagger 不会初始化相应的 Executor。但随着每次对 singleExecutorProvider.get ( ) 的调用，都会创建一个新实例。因此，singleExecutor 和 singleExecutor2 是两个不同的对象。此行为与非范围对象的行为基本相同。

那么在哪些情况下，提供者是相关的？  当我们提供某种可变的依赖关系时，它就会派上用场，随着时间的推移，这种依赖会改变它的状态，并且每次调用我们都需要获得当前状态。 "这个弯曲的架构是什么?" 你可能会说，我同意你的看法。 在处理遗留代码时，你经常会看到类似的东西。 

值得注意的是，库的作者也建议不要滥用 Provider 接口，因为它可能导致上面提到的"不正确的架构"和难以捕捉的错误。 

现在关于 Lazy\<Executor> multiExecutorLazy  和 Lazy\<Executor> multiExecutorLazyCopy  的问题。 2只在第一次调用 multiExecutorLazy.get ( ) 和 multiExecutorLazyCopy.get ( )  时，才初始化相应的 Executor ( )。 接下来，Dagger 缓存对每个 Lazy 都初始化了值，并将在随后调用多个 executorlazy.get ( ) 和多个 executorlazycopy.get ( ) 上返回缓存的值。 

这样，multiExecutor 和 multiExecutor2 引用一个对象，而 multiExecutor3 引用另一个对象。

但是，如果我们在 AppModule 中添加 @Singleton 注解以提供 MultipleThreadExecutor ( ) 方法，那么将为整个依赖关系树缓存该对象，并且 multiExecutor，multiExecutor2，multiExecutor3 将引用同一个对象。

小心点。 

## 异步加载

我们接近了一项非常重大的任务。如果我们想要在后台建立依赖关系图呢？听起来很有希望？是的，我在谈论生产者 ( Producers )。

老实说，这个话题值得单独考虑。这里有许多特色和细节，已经发布了足够好的材料。现在我只会介绍生产者的利弊。

优点。那么，最重要的优点是在后台下载并能够管理这个下载过程。

缺点。生产者依赖 Guava，这意味着加上 15k 方法的 apk。但最糟糕的是，使用生产者会破坏应用程序的整体架构，并使代码更加混乱。如果你已经有了 Dagger，然后决定将对象初始化移动到后台，那就需要额外的努力。

官方文件中有专门关于[这个主题的部分](https://google.github.io/dagger/producers.html)。不过，我会强烈推荐 [Miroslaw Stanek](https://medium.com/@froger_mcs) 的[文章](https://medium.com/@froger_mcs)。他有一个非常好的博客，并且有很多关于 Dagger 2 的文章。我甚至在他以前的文章中借过几张照片。 他在[这篇文章](http://frogermcs.github.io/dependency-injection-with-dagger-2-producers/)中解释生产者。

在[下一篇文章](http://frogermcs.github.io/async-injection-in-dagger-2-with-rxjava/)中，有一个非常有趣的替代方法，用于在后台加载依赖关系树，RxJava 可以帮助你。我非常喜欢他的解决方案，因为它完全否认了使用生产者的缺点，同时解决了异步加载的问题。

然而，只有一个缺陷：Miroslaw 不完全正确地应用 Observable.create（...）。但是我在文章中写了一个评论，所以请仔细注意它。 

现在我们来看看作用域对象代码是怎么样的（使用 “正确” 的 RxJava）：

```java
@Module
public class AppModule {

    @Provides
    @Singleton // or custom scope for "local" singletons
    HeavyExternalLibrary provideHeavyExternalLibrary() {
        HeavyExternalLibrary heavyExternalLibrary = new HeavyExternalLibrary();
        heavyExternalLibrary.init(); //This method takes about 500ms
        return heavyExternalLibrary;
    }

    @Provides
    @Singleton // or custom scope for "local" singletons
    Observable<HeavyExternalLibrary> provideHeavyExternalLibraryObservable(
            final Lazy<HeavyExternalLibrary> heavyExternalLibraryLazy) {
        return Observable.fromCallable(heavyExternalLibraryLazy::get)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread());
    }

}

public class MainActivity extends AppCompatActivity {

    @Inject
    Observable<HeavyExternalLibrary> heavyExternalLibraryObservable;

    //This will be injected asynchronously
    HeavyExternalLibrary heavyExternalLibrary;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyApplication.getInstance().getAppComponent().inject(this);
        setContentView(R.layout.activity_main);
        // init HeavyExternalLibrary in background thread!
        heavyExternalLibraryObservable.subscribe(
                heavyExternalLibrary1 -> heavyExternalLibrary = heavyExternalLibrary1,
                throwable -> {}
        );
    }
}
```

请注意 AppModule 中的 @Singleton 和  Lazy 接口。该接口保证重要的对象将在请求时被初始化，然后被缓存。

如果我们想每次都收到这个 “重” 对象的新副本，我们该怎么办？然后我们需要修改 AppModule：

```java
@Module
public class AppModule {

    @Provides
        // No scope!
    HeavyExternalLibrary provideHeavyExternalLibrary() {
        HeavyExternalLibrary heavyExternalLibrary = new HeavyExternalLibrary();
        heavyExternalLibrary.init(); //This method takes about 500ms
        return heavyExternalLibrary;
    }

    @Provides
    @Singleton // or custom scope for "local" singletons
    Observable<HeavyExternalLibrary> provideHeavyExternalLibraryObservable(
            final Provider<HeavyExternalLibrary> heavyExternalLibraryLazy) {
        return Observable.fromCallable(heavyExternalLibraryLazy::get)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread());
    }

}

public class MainActivity extends AppCompatActivity {

    @Inject
    Observable<HeavyExternalLibrary> heavyExternalLibraryObservable;

    //This will be injected asynchronously
    HeavyExternalLibrary heavyExternalLibrary;
    HeavyExternalLibrary heavyExternalLibraryCopy;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyApplication.getInstance().getAppComponent().inject(this);
        setContentView(R.layout.activity_main);
        // init HeavyExternalLibrary and heavyExternalLibraryCopy in background thread!
        heavyExternalLibraryObservable.subscribe(
                heavyExternalLibrary1 -> heavyExternalLibrary = heavyExternalLibrary1,
                throwable -> {}
        );
        heavyExternalLibraryObservable.subscribe(
                heavyExternalLibrary1 -> heavyExternalLibraryCopy = heavyExternalLibrary1,
                throwable -> {}
        );
    }
}
```

对于 provideHeavyExternalLibrary ( ) 方法，我们删除了范围，并在ProvideHeavyExternalLibraryObservable（final Provider \<HeavyExternalLibrary> heavyExternalLibraryLazy）中使用 Provider 而不是 Lazy。因此，MainActivity 中的 heavyExternalLibrary 和 heavyExternalLibraryCopy 是不同的对象。

还可以将整个依赖树初始化过程移到后台。 你想知道如何？ 很简单。首先看看它是如何的（来自 Miroslaw 文章）

```java
public class SplashActivity extends BaseActivity {

    @Inject
    SplashActivityPresenter presenter;
    @Inject
    AnalyticsManager analyticsManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setupActivityComponent();
    }

    @Override
    protected void setupActivityComponent() {
        final SplashActivityComponent splashActivityComponent =
                GithubClientApplication.get(SplashActivity.this)
                        .getAppComponent()
                        .plus(new SplashActivityModule(SplashActivity.this));
        splashActivityComponent.inject(SplashActivity.this);
    }

}
```

现在看看更新的方法 void setupActivityComponent（）（在 RxJava 上使用我的编辑）：

```java
@Override
protected void setupActivityComponent() {
    Completable.fromAction(() -> {
                final SplashActivityComponent splashActivityComponent =
                        GithubClientApplication.get(SplashActivity.this)
                                .getAppComponent()
                                .plus(new SplashActivityModule(SplashActivity.this));
                splashActivityComponent.inject(SplashActivity.this);
            })
            .doOnCompleted(() -> {
                //Here is the moment when injection is done.
                analyticsManager.logScreenView(getClass().getName());
                presenter.callAnyMethod();
            })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(() -> {}, throwable -> {});
}
```

## 测量

在最后一节中，我们讨论了应用程序的启动性能。然而，我们知道，如果问题是关于性能和速度，我们需要测量！ 我们不应该依赖直觉，而且"它似乎变得更快"的感觉。 而且 Miroslaw 在[这篇文章](http://frogermcs.github.io/dagger-graph-creation-performance/)和[这篇文章](http://frogermcs.github.io/dagger2metrics-measure-performance-of-graph-initialization/)中再次帮助我们。没有他，我们会怎么做，我无法想象。

## 新的有趣的机会

在 Dagger 中出现了一些新的有趣特征，保证让我们的生活更轻松。但是要理解一切是如何运作的以及它给我们带来了什么并不是一件容易的事。 好了，让我们开始吧！ 

## @Reusable范围

这是一个有趣的注解。 它可以保存内存，但事实上它不受任何范围的限制，这使得在任何组件中重用依赖性非常方便。 也就是说，它是一个介于范围和未范围之间的东西。 

在文档中有一个非常重要的点，这个点最初并没有引起注意:"对于使用 @reusable 依赖项的每个组件，这个依赖项分别缓存。 ” 我的补充说明：“与范围注释不同，在创建时缓存对象，其实例被子组件和依赖组件使用。”

让我们马上来看看这个例子: 

我们的主要组成部分: 

```java
@Component(modules = {AppModule.class, UtilsModule.class})
@Singleton
public interface AppComponent {

    FirstComponent.Builder firstComponentBuilder();
    SecondComponent.Builder secondComponentBuilder();

}
```

AppComponent 有两个子组件。你有没有注意到 FirstComponent.Builder 的构造？我们稍后再谈。

现在我们来看看 UtilsModule：

```java
@Module
public class UtilsModule {

    @Provides
    @NonNull
    @Reusable
    public NumberUtils provideNumberUtils() {
        return new NumberUtils();
    }

    @Provides
    @NonNull
    public StringUtils provideStringUtils() {
        return new StringUtils();
    }

}
```

NumberUtils 带有注解 @Reusable，并且 StringUtils 没有范围。

接下来，我们有两个 Subcomponens：

```java
@FirstScope
@Subcomponent(modules = FirstModule.class)
public interface FirstComponent {

    @Subcomponent.Builder
    interface Builder {
        FirstComponent.Builder firstModule(FirstModule firstModule);
        FirstComponent build();
    }

    void inject(MainActivity mainActivity);

}

@SecondScope
@Subcomponent(modules = {SecondModule.class})
public interface SecondComponent {

    @Subcomponent.Builder
    interface Builder {
        SecondComponent.Builder secondModule(SecondModule secondModule);
        SecondComponent build();
    }

    void inject(SecondActivity secondActivity);
    void inject(ThirdActivity thirdActivity);

}
```

FirstComponent 仅在 MainActivity 中注入，SecondComponent 在 SecondActivity 和 ThirdActivity 中注入。

让我们看看代码：

```java
public class MainActivity extends AppCompatActivity {

    @Inject
    NumberUtils numberUtils;

    @Inject
    StringUtils stringUtils;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        MyApplication.getInstance().getFirstComponent()
                .inject(this);
        // other...
    }

}

public class SecondActivity extends AppCompatActivity {

    @Inject
    NumberUtils numberUtils;

    @Inject
    StringUtils stringUtils;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        MyApplication.getInstance().getSecondComponent()
                .inject(this);
        // other...
    }

}

public class ThirdActivity extends AppCompatActivity {

    @Inject
    NumberUtils numberUtils;

    @Inject
    StringUtils stringUtils;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_third);
        MyApplication.getInstance().getSecondComponent()
                .inject(this);
        // other...
    }

}
```

简要介绍一下导航。从 MainActivity 我们进入 SecondActivity，然后进入 ThirdActivity。现在是问题。当我们已经在第三个屏幕上时，将创建多少个 NumberUtils 和 StringUtil 对象？

由于 StringUtils 被解除了范围，将创建三个实例，即每次注入都会创建一个新对象。 我们知道。 

但是会有两个 NumberUtils 对象 - 一个用于 FirstComponent，另一个用于 SecondComponent。在这里我将再次从文档中提供有关 @Reusable 的基本概念：“对于使用 @Reusable 依赖项的每个组件，这个依赖项会被分开缓存！”，与创建时缓存对象的范围注解形成对比的是，其实例由子组件和相关组件使用。

但 Google 员工自己警告说，如果你需要一个独一无二的可变对象，那么只需使用有范围的注解。 

我还会给你一个关于比较 @Singleton 和 @Reusable 的问题的[链接](http://stackoverflow.com/questions/39136042/dagger-reusable-scope-vs-singleton)。

## @Subcomponent.Builder

一个使代码更加美丽的特性。 以前，要创建 @Subcomponent，我们必须写下这样的东西: 

```java
@Component(modules = {AppModule.class, UtilsModule.class})
@Singleton
public interface AppComponent {
    FirstComponent plusFirstComponent(FirstModule firstModule, SpecialModule specialModule);
}

@FirstScope
@Subcomponent(modules = {FirstModule.class, SpecialModule.class})
public interface FirstComponent {
    void inject(MainActivity mainActivity);
}

//FirstComponent creation:
FirstComponent firstComponent = appComponent.plusFirstComponent(new FirstModule(), new SpecialModule());
```

我不喜欢这种方法，因为父组件载有关于子组件使用的模块的不必要的知识。 再加上传递大量的参数看起来并不好，因为有一个模式生成器用于此目的。 如今，它很优雅: 

```java
@Component(modules = {AppModule.class, UtilsModule.class})
@Singleton
public interface AppComponent {
    FirstComponent.Builder firstComponentBuilder();
}

@FirstScope
@Subcomponent(modules = {FirstModule.class, SpecialModule.class})
public interface FirstComponent {

    @Subcomponent.Builder
    interface Builder {
        FirstComponent.Builder firstModule(FirstModule firstModule);
        FirstComponent.Builder specialModule(SpecialModule specialModule);
        FirstComponent build();
    }

    void inject(MainActivity mainActivity);

}

// FirstComponent creation:
FirstComponent firstComponent = appComponent
        .firstComponentBuilder()
        .firstModule(new FirstModule())
        .specialModule(new SpecialModule())
        .build();
```

现在，它好得多=）

## 静态

现在我们有机会做这样的事情：

```java
@Provides static User currentUser(AuthManager authManager) {
    return authManager.currentUser();
}
```

也就是说，在模块中提供依赖关系的方法可以进行静态化。 起初我不太明白，为什么我需要它？ 事实证明，这种功能的请求已经存在了很长一段时间，而且在某些情况下它是有用的。 

在 SOF 上就这个话题提出了一个[很好的问题](https://stackoverflow.com/questions/38933560/different-singleton-static-provides-in-dagger2)：“@Provide static 的 @Singleton 实际上与静态有什么不同？”。为了更好地理解这个差异，你需要阅读问题的答案，同时并行地实验和查看生成的代码。 

假设我们在模块中有相同方法的三个不同版本：

```java
@Provides User currentUser(AuthManager authManager) {
    return authManager.currentUser();
}

@Provides @Singleton User currentUser(AuthManager authManager) {
    return authManager.currentUser();
}

@Provides static User currentUser(AuthManager authManager) {
    return authManager.currentUser();
}
```

同时，authManager.currentUser ( ) 可以在不同的时间提供不同的实例。

逻辑问题是: 这些方法有何不同。 

在第一种情况下，我们有一个经典的无范围 。每个请求都会被赋予一个authManager.currentUser ( ) 新实例（更确切地说，是一个到 currentUser 的新链接）。

在第二种情况下，与 currentUser 的链接将在第一次请求时被缓存，并且在随后的每个请求中都会给出。 例如，如果 currentUser 在 AuthManager 中已经改变，那么将给出与无效实例的旧链接。 

第三种情况更有趣。该方法的行为类似于无范围，即对于每个请求将给予一个新的链接。 这是@singleton 的第一个区别，它缓存对象。 因此，在 @Provide static 方法中放置对象初始化并不完全合适。 

那么 @Provide static 和 unscope 有什么区别？假设我们有以下模块：

```java
@Module
public class AuthModule {
    @Provides
    User currentUser(AuthManager authManager) {
        return authManager.currentUser();
    }
}
```

AuthManager 由另一个模块作为 Singleton 提供。现在快速查看 AuthModule_CurrentUserFactory 生成的代码（在 Android studio 中，只需将光标放在 currentUser 上并按 Ctrl + B）：

```java
@Generated(
        value = "dagger.internal.codegen.ComponentProcessor",
        comments = "https://google.github.io/dagger"
)
public final class AuthModule_CurrentUserFactory implements Factory<User> {
    private final AuthModule module;

    private final Provider<AuthManager> authManagerProvider;

    public AuthModule_CurrentUserFactory(
            AuthModule module, Provider<AuthManager> authManagerProvider) {
        assert module != null;
        this.module = module;
        assert authManagerProvider != null;
        this.authManagerProvider = authManagerProvider;
    }

    @Override
    public User get() {
        return Preconditions.checkNotNull(
                module.currentUser(authManagerProvider.get()),
                "Cannot return null from a non-@Nullable @Provides method");
    }

    public static Factory<User> create(AuthModule module, Provider<AuthManager> authManagerProvider) {
        return new AuthModule_CurrentUserFactory(module, authManagerProvider);
    }

    / Proxies {@link AuthModule#currentUser(AuthManager)}. /
    public static User proxyCurrentUser(AuthModule instance, AuthManager authManager) {
        return instance.currentUser(authManager);
    }
}
```

如果你添加 static 到 currentUser：

```java
@Module
public class AuthModule {
    @Provides
    static User currentUser(AuthManager authManager) {
        return authManager.currentUser();
    }
}
```

然后我们得到：

```java
@Generated(
        value = "dagger.internal.codegen.ComponentProcessor",
        comments = "https://google.github.io/dagger"
)
public final class AuthModule_CurrentUserFactory implements Factory<User> {
    private final Provider<AuthManager> authManagerProvider;

    public AuthModule_CurrentUserFactory(Provider<AuthManager> authManagerProvider) {
        assert authManagerProvider != null;
        this.authManagerProvider = authManagerProvider;
    }

    @Override
    public User get() {
        return Preconditions.checkNotNull(
                AuthModule.currentUser(authManagerProvider.get()),
                "Cannot return null from a non-@Nullable @Provides method");
    }

    public static Factory<User> create(Provider<AuthManager> authManagerProvider) {
        return new AuthModule_CurrentUserFactory(authManagerProvider);
    }

    / Proxies {@link AuthModule#currentUser(AuthManager)}. /
    public static User proxyCurrentUser(AuthManager authManager) {
        return AuthModule.currentUser(authManager);
    }
}
```

注意，静态版本中没有 AuthModule。 因此，静态方法由组件直接调用，绕过模块。 如果模块中只有静态方法，则甚至没有创建模块实例。 

这就是效率， 没有不必要的调用。实际上，我们在这里有一个性能增益。 我们也知道，调用静态方法比非静态方法快15-20% 。 如果我错了，Alexander Efremenkov 会纠正我。他肯定知道，如果有必要，他会做适当的测量。

## @Binds +构造函数注入

超级方便的联合，可以显著地减少模板代码。 当我刚开始学习 Dagger 时，我不明白为什么我们需要构造函数注入。对象来自哪里？然后有 @Binds。事实上一切其实都很简单。感谢  Vladimir Tagakov 和[这篇文章](https://android.jlelse.eu/inject-interfaces-without-providing-in-dagger-2-618cce9b1e29#.p0xkh59pg)的帮助。

我们来考虑一个典型的情况。有一个 Presenter 接口及其实现：

```java
public interface IFirstPresenter {
    void foo();
}

public class FirstPresenter implements IFirstPresenter {

    public FirstPresenter() {}

    @Override
    public void foo() {}

}
```

我们提供模块中的所有内容，并将 Presenter 的接口注入活动中：

```java
@Module
public class FirstModule {

    @Provides
    @FirstScope
    public IFirstPresenter provideFirstPresenter() {
        return new FirstPresenter();
    }

}

public class MainActivity extends AppCompatActivity {

    @Inject
    IFirstPresenter firstPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        MyApplication.getInstance().getFirstComponent()
                .inject(this);
        // others
    }

}
```

假设我们的 FirstPresenter 需要帮助程序类，它将部分工作委托给它们。为此，我们需要在模块中创建两个更多的方法来提供新的类， 然后更改 FirstPresenter 构造函数，从而在模块中更新相应的方法。 

该模块内容将如下所示：

```java
@Module
public class FirstModule {

    @Provides
    @FirstScope
    public HelperClass1 provideHelperClass1() {
        return new HelperClass1();
    }

    @Provides
    @FirstScope
    public HelperClass2 provideHelperClass2() {
        return new HelperClass2();
    }

    @Provides
    @FirstScope
    public IFirstPresenter provideFirstPresenter(
            HelperClass1 helperClass1, HelperClass2 helperClass2) {
        return new FirstPresenter(helperClass1, helperClass2);
    }

}
```

每次你想要添加一些类并与其他类共享时，同样的事情也会发生。 这个模块很快就变脏了。 而且代码太多了，不是吗？ 但是有一个解决方案可以显著地减少代码。 

首先，如果我们需要创建一个依赖项并给出一个已经完成的类，而不是一个接口( helperal1 和 HelperClass2) ，我们可以使用构造函数注入。 它看起来是这样的: 

```java
@FirstScope
public class HelperClass1 {
    @Inject
    public HelperClass1() {
    }
}

@FirstScope
public class HelperClass2{
    @Inject
    public HelperClass2() {
    }
}
```

请注意，@FirstScope 注解已添加到类中，这样 Dagger 就可以理解将这些类分配给哪个依赖项树。 

现在我们可以安全地从模块中移除 helpera 和 HelperClass2的提供者: 

```java
@Module
public class FirstModule {

    @Provides
    @FirstScope
    public IFirstPresenter provideFirstPresenter(
            HelperClass1 helperClass1, HelperClass2 helperClass2) {
        return new FirstPresenter(helperClass1, helperClass2);
    }

}
```

如何才能减少模块中的代码？ 这里我们应用 @Binds：

```java
@Module
public abstract class FirstModule {

    @FirstScope
    @Binds
    public abstract IFirstPresenter provideFirstPresenter(FirstPresenter firstPresenter);

}
```

并且在 FirstPresenter 中添加构造函数注入：

```java
@FirstScope
public class FirstPresenter implements IFirstPresenter {

    private HelperClass1 helperClass1;
    private HelperClass2 helperClass2;

    @Inject
    public FirstPresenter(HelperClass1 helperClass1, HelperClass2 helperClass2) {
        this.helperClass1 = helperClass1;
        this.helperClass2 = helperClass2;
    }

    @Override
    public void foo() {}

}
```

这里有什么新东西？ FirstModule 和 ProvideFirstPresenter 已经变得抽象。 @Provide 注解已被 @Binds 替换。我们传递的不是必要的依赖，而是对参数的一个具体实现!  

范围注解已添加到 FirstPresenter 中 - @FirstScope ，Dagger 通过该注解，了解该类的位置。此外，@Inject 注解已添加到构造函数中。它变得更清洁了，现在添加新的依赖关系变得更加容易！

来自 [Andrei Zajac](https://habrahabr.ru/users/mujahit/) 的抽象模块的一些有价值的补充。 让我们记住 FirstModule 引用 FirstComponent，它又是 AppComponent 的一个子组件。为了创建 FirstComponent，我们这样做了：

```java
appComponent
    .firstComponentBuilder()
    .firstModule(new FirstModule())
    .specialModule(new SpecialModule())
    .build();
```

但如果是抽象的，我们如何创建 FirstModule 实例呢？ 在前面的文章中，我提到，如果我们没有向模块构造函数传递任何东西(即我们使用缺省构造函数) ，那么我们可以在创建组件时省略这些模块初始化: 

```java
appComponent
    .firstComponentBuilder()
    .build();
```

而 Dagger 自己决定如何处理抽象和非抽象模块，以及如何提供所有这些必要的依赖。 

还要注意，如果一个模块只有抽象方法，那么它可以通过一个接口来实现：

```java
@Module
public interface FirstModule {

    @FirstScope
    @Binds
    IFirstPresenter provideFirstPresenter(FirstPresenter firstPresenter);

}
```

此外，我们只能在抽象模块中添加静态方法，但不能添加"正常"模块: 

```java
@Module
public abstract class FirstModule {

    @FirstScope
    @Binds
    public IFirstPresenter provideFirstPresenter(FirstPresenter firstPresenter);

    @FirstScope
    @Provides
    public HelperClass3 provideHelperClass3() {  // <- Incorrect!
        return new HelperClass3();
    }

    @FirstScope
    @Provides
    public static HelperClass3 provideHelperClass3() {  // <- Ok!
        return new HelperClass3();
    }

}
```

## 还有什么没有被覆盖

接下来，我会给你一个简短的描述功能和链接到资源的列表：

1. Muitibindings 允许绑定集合中的对象（Set 和 Map）。适合实施 “插件架构”。对于初学者，我强烈推荐[这篇非常详细的文章](https://medium.com/@hamidgh/dagger-multibinding-sets-and-maps-713254b7f734#.8cmale9uw)。更多有趣的多重绑定应用案例可以看 Miroslav 的[文章1](http://frogermcs.github.io/inject-everything-viewholder-and-dagger-2-example/) , [文章2](http://frogermcs.github.io/activities-multibinding-in-dagger-2/)。另外还有[官方文档的链接](https://google.github.io/dagger/multibindings.html)。
2. Releasable references 在你遇到内存不足问题时非常有用。在相应的注解的帮助下，我们标记可以在出现内存不足时释放的对象。它可以被视为黑科技。在[文档](https://google.github.io/dagger/users-guide.html)（Releasable references 小节）中，一切都很清楚地描述。
3. Testing 当然，单元测试 Dagger 是不需要的。但是对于功能，集成和 UI 测试，替换某些模块可能会很有用。[Artem Zinnatullin](https://artemzin.com/blog/author/artem/) 在他的[文章](https://artemzin.com/blog/android-development-culture-the-document-qualitymatters/)和[例子](https://github.com/artem-zinnatullin/qualitymatters)中极大地解释了这个问题。有关[测试的部分](https://google.github.io/dagger/testing.html)在文档中突出显示。 但是，Google 也不能正确地描述如何替代这个组件。 如何正确创建模拟模块并替换它们。为了替换组件（单个模块），我使用 [Artem 的方式](https://github.com/artem-zinnatullin/qualitymatters/blob/master/app/src/functionalTests/java/com/artemzin/qualitymatters/functional_tests/QualityMattersFunctionalTestApp.java)。是的，如果可以在单独的类中创建测试组件和模块，并将它们全部包含在测试应用程序文件中，那将很酷。也许有人知道？
4. @BindsOptionalOf 适用于 Java 8或 Guava 的 Optional，这使得我们很难访问此功能。如果感兴趣，你可以在[文档末尾](https://google.github.io/dagger/users-guide.html)找到说明。
5. @BindsInstance 其主要信息是停止通过模块的构造函数传输任何对象。 一个常见的例子是，当全局上下文通过 AppComponent 构造函数传递时。 所以有了这个注解，你就不需要这么做了。 [在文档结尾处](https://google.github.io/dagger/users-guide.html)有一个例子。

就是这样！ 似乎所有的都被掩盖了。如果有什么东西不见了或者没有被充分描述，那就写给我！ 我会解决这个问题。 

Authors:
[Eugene Matsyuk](https://medium.com/@eugenematsyuk)
[Roman Yatsina](https://medium.com/@yatsinar)
[Atabek Murtazaev](https://medium.com/@atabekm)

