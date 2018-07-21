#  Dagger 2 的依赖注入 - 关系图创建优化

> 原文 (mirekstanek.online) ：[Dagger 2 - graph creation performance](https://mirekstanek.online/dagger-2-graph-creation-performance/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

[PerfMatters](https://twitter.com/search?q=%23perfmatters) - 最近非常流行的标签，特别是在 Android 世界。不管怎样，应用程序刚刚做了一些事情的时间已经过去了。现在一切都应该是愉快的，光滑的，快速的。 例如，Instagram [花了将近半年的时间](http://instagram-engineering.tumblr.com/post/97740520316/betterandroid)来让它的应用程序更快、更漂亮、屏幕尺寸更大。

这就是为什么今天我想和大家分享一些对你的应用启动时间有很大影响的小提示(尤其是当它使用一些外部库时)。

## 对象图的创建
在大多数情况下，在应用开发过程中，它的启动时间或多或少地增加了。有时候很难在日常发展中发现，但是当你把你的第一个应用程序版本和最新的应用程序版本进行比较时，你会发现这个差别相对较大。

而原因可能在于 dagger 对象图的创建过程。

你可能会问 Dagger 2？ 确切地说，即使你刚刚将实现从其他基于反射的解决方案中移动，并且你的代码是在编译时生成的，不要忘记对象创建仍然发生在运行时。

对象（及其依赖关系）在注入时第一次创建。Jake Wharton 在他关于 Dagger 2 的演示幻灯片中清楚地展示了这一点:

下面是我们的 GithubClient 示例应用程序：
- 应用程序首次启动（在被杀害之后）。应用程序对象没有 @Inject 字段，所以只创建 AppComponent 对象。
- 应用程序创建 SplashActivity - 它有两个 @Inject 字段：AnalyticsManager 和 SplashActivityPresenter。
- AnalyticsManager 依赖于已经提供的 Application 对象。所以只有 AnalyticsManager 构造函数被调用。
- SplashSctivityPresenter 依赖于：SplashActivity，Validator 和 UserManager。提供了 SplashActivity，应该创建 Validator 和 UserManager。
- UserManager 依赖于也应该创建的 GithubApiService。在它创建 UserManager 之后。
- 现在，当我们有所有的依赖，SplashSctivityPresenter 创建。
  ![|center](http://frogermcs.github.io/images/18/graph.png)

有点纠结，但结果是，在创建 SplashActivity 之前（让我们假设对象注入是 onCreate（）方法中完成的唯一操作），我们必须等待构造（和可选配置）：

- GithubApiService（它也使用一些依赖，如 OkHttpClient 和 RestAdapter）
- UserManager
- Validator
- SplashActivityPresenter
- AnalyticsManager

一个接一个创建。

嘿，但不用担心，很多更加丰富的图几乎可以立即创建。

## 问题
现在让我们想象一下，我们必须添加外部库，这些库应该在应用开始时初始化(如 Crashlytics，Mixpanel，Google Analytics，Parse 等)。 让我们想象一下，它是我们的重量级外部库(HeavyExternalLibrary)，看起来是这样的:

```java
public class HeavyExternalLibrary {

    private boolean initialized = false;

    public HeavyExternalLibrary() {
    }

    public void init() {
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        initialized = true;
    }

    public void callMethod() {
        if (!initialized) throw new RuntimeException("Call init() before you use this library");
    }
}
```
简而言之，构造函数是空的，它的调用几乎没有成本。 但是我们有 init ( ) 方法需要大约 500ms，在我们能够使用这个库之前必须调用它。 为了确保我们在模块中的某个地方创建对象的时刻调用 init ( ) :

```java
//AppModule

@Provides
@Singleton
HeavyExternalLibrary provideHeavyExternalLibrary() {
    HeavyExternalLibrary heavyExternalLibrary = new HeavyExternalLibrary();
    heavyExternalLibrary.init();
    return heavyExternalLibrary;
}
```
现在我们的 HeavyExternalLibrary 成为 SplashActivityPresenter 的一部分：
```java
//SplashActivityModule

@Provides
@ActivityScope
SplashActivityPresenter
provideSplashActivityPresenter(Validator validator, UserManager userManager, HeavyExternalLibrary heavyExternalLibrary) {
    return new SplashActivityPresenter(splashActivity, validator, userManager, heavyExternalLibrary);
}
```
会发生什么？我们的应用程序启动大约500毫秒，仅仅是因为在创建 SplashActivityPresenter 依赖图的过程中必须进行的重量级外部库初始化。

## 测量
Android SDK（和 Android Studio 本身）为我们提供了一个可视化应用程序执行的工具 - [Traceview](http://tools.android.com/tips/traceview)。由于这一点，我们可以看到每个方法需要花费多少时间，并在注入过程中找到瓶颈。

顺便说一句，如果你以前没有看到它，请看看[Udi Cohen 关于 Android 性能优化的博客文章](http://blog.udinic.com/2015/09/15/speed-up-your-app/)。

从 Android Studio（Android 监视器选项卡 - > CPU - >启动/停止方法跟踪）直接启动的 Traceview 有时并不是那么精确，特别是当我们尝试在应用程序启动时点击“开始”。

对我们来说幸运的是，当我们知道应该测量的代码的确切位置时，可以使用这种方法。 可以使用 [Debug.startMethodTracing ()](http://developer.android.com/reference/android/os/Debug.html#startMethodTracing(java.lang.String) ) 指出我们的代码中应该开始测量的地方。 Debug.stopMethodTracing () 停止跟踪并创建新文件。

为了实践，让我们测量我们的 SplashActivity 注入过程：
```java
@Override
protected void setupActivityComponent() {
    Debug.startMethodTracing("SplashTrace");
    GithubClientApplication.get(this)
            .getAppComponent()
            .plus(new SplashActivityModule(this))
            .inject(this);
    Debug.stopMethodTracing();
}
```
setupActivityComponent () 在 onCreate () 中被调用。

根据文档结果保存在 `/sdcard/SplashTrace.trace`。

要拉取他们可以在你的终端使用这个：` $ adb pull /sdcard/SplashTrace.trace`

现在，你只需将其拖到 Android Studio 中即可阅读此文件：
![|center](http://frogermcs.github.io/images/18/trace.png)

你应该看到类似的东西：
![](http://frogermcs.github.io/images/18/traceview.png)
当然在这种情况下的结果是非常清楚的：AppModule_ProvideHeavyExternalLibraryFactory.get（）（我们的 HeavyExternalLibrary 被创建的地方）大约需要 500ms。

从纯粹的好奇心，让我们最后放大一小段追踪：
![](http://frogermcs.github.io/images/18/traceview2.png)你看出区别了吗？我们的示例类的构造：AnalyticsManager 需要不到1ms的时间。 如果你想看到它，这里是在这个例子中使用的 [SplashTrace.trace](https://github.com/frogermcs/frogermcs.github.io/raw/master/files/18/SplashTrace.trace) 文件。

## 解决

不幸的是，对于这种性能问题，有时没有明确的答案。 这里有两个对我帮助很大:

### 惰性加载(临时解决方案)

> 提示2: 使用 [# dagger2](https://twitter.com/hashtag/Dagger2?src=hash) Lazy<>，特别是在 Application 中初始化但稍后使用的依赖项。 
>
> [# perfiddev](https://twitter.com/hashtag/AndroidDev?src=hash) [# perfmatters](https://twitter.com/hashtag/perfmatters?src=hash)
>
> — Miroslaw Stanek (@froger_mcs) [2015年9月22日 twitter](https://twitter.com/froger_mcs/status/646323343851917312?ref_src=twsrc%5Etfw&ref_url=http%3A%2F%2Ffrogermcs.github.io%2Fdagger-graph-creation-performance%2F)

首先，如果你需要所有注入的依赖关系，我们先来考虑一下。也许他们中的一些可以在稍后的时间里进行懒惰加载？ 当然这并不能解决真正的问题（UI 线程会在第一次调用 Lazy <>.get（）方法时被阻塞）。但在某些情况下，它可以帮助启动时间（特别是在对象很少被使用的情况下）。 检查 [Lazy <>](http://google.github.io/dagger/api/2.0/dagger/Lazy.html) 接口文档以获取更多详细信息和代码示例。

简而言之，你使用 @Inject SomeClass someClass 的每个地方都可以用 @Inject Lazy \<SomeClass> someClassLazy （也可以在构造函数注入中）替换。而要获得 someClass 的实例，你必须调用 someClassLazy.get（）。

### 异步对象创建

第二个选项（更多的是想法而不是最终解决方案）是在后台线程中的某个地方初始化对象，缓存所有调用的方法，然后在初始化对象上调用它们。

这个解决方案的缺点是它必须为我们想要包装的所有类别独立地进行准备。 而且它只适用于将来可以执行方法调用的那些对象（就像任何分析一样——如果一些事件稍后被记录下来也没关系）。

这个解决方案可以找到我们的 HeavyExternalLibrary：
```java
public class HeavyLibraryWrapper {

    private HeavyExternalLibrary heavyExternalLibrary;

    private boolean isInitialized = false;

    ConnectableObservable<HeavyExternalLibrary> initObservable;

    public HeavyLibraryWrapper() {
        initObservable = Observable.create(new Observable.OnSubscribe<HeavyExternalLibrary>() {
            @Override
            public void call(Subscriber<? super HeavyExternalLibrary> subscriber) {
                HeavyLibraryWrapper.this.heavyExternalLibrary = new HeavyExternalLibrary();
                HeavyLibraryWrapper.this.heavyExternalLibrary.init();
                subscriber.onNext(heavyExternalLibrary);
                subscriber.onCompleted();
            }
        }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).publish();

        initObservable.subscribe(new SimpleObserver<HeavyExternalLibrary>() {
            @Override
            public void onNext(HeavyExternalLibrary heavyExternalLibrary) {
                isInitialized = true;
            }
        });

        initObservable.connect();
    }

    public void callMethod() {
        if (isInitialized) {
            HeavyExternalLibrary.callMethod();
        } else {
            initObservable.subscribe(new SimpleObserver<HeavyExternalLibrary>() {
                @Override
                public void onNext(HeavyExternalLibrary heavyExternalLibrary) {
                    heavyExternalLibrary.callMethod();
                }
            });
        }
    }
}
```
当调用 HeavyLibraryWrapper 构造函数时，库初始化发生在后台线程（Schedulers.io () ）中。 与此同时，当用户调用 callMethod () 时，它会将新订阅添加到我们的初始化过程中。 当它完成（onNext () 方法返回初始化 HeavyExternalLibrary 对象）缓存调用传递给此对象。

目前这个想法很简单，还在开发中。有可能导致内存泄漏（例如，如果我们必须通过 callMethod () )中的任何参数），但总的来说，对于最简单的情况来说是可行的。

**任何其他解决方案**

性能提升过程是非常独特的。 但是如果你想分享你的想法，请在这里评论。

谢谢你的阅读！

## 示例代码
所描述项目的完整源代码在 [Github](https://github.com/frogermcs/GithubClient) 存储库中获取。




