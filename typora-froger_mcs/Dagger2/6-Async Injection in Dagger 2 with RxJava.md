# Dagger 2 的依赖注入- 使用 RxJava 进行异步注入

> 原文 (mirekstanek.online) ：[Async Injection in Dagger 2 with RxJava](https://mirekstanek.online/async-injection-in-dagger-2-with-rxjava/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

这篇文章的第一个版本最初写在我的开发博客上：http://frogermcs.github.io/

这篇文章是在 Android 中显示依赖注入与 Dagger 2 框架的系列文章的一部分。今天我们来看看 RxJava 的异步注入2 - 替代 Dagger 2 Producers 。

几个星期前，我在 “ Dagger 2 ” 中用生产者写了[一篇](https://medium.com/@froger_mcs/dependency-injection-with-dagger-2-producers-c424ddc60ba3)关于异步依赖注入的文章。 在后台线程上执行对象初始化有一个很大的好处 - 它不会阻塞负责实时绘制用户界面的主线程（保持平滑界面[每秒60帧](https://www.youtube.com/watch?v=CaMTIgxCSqU)）。

值得指出的是，缓慢的初始化并不是每个人的问题。但是，即使你在代码中真正关心它，也总是有可能任何外部库在构造函数或任何 init () 方法中执行光盘 / 网络操作。如果你不确定，我建议你尝试 [AndroidDevMetrics](https://github.com/frogermcs/AndroidDevMetrics) —— Android 的性能指标库。 它可以告诉你在应用程序中显示特定屏幕需要多少时间，(如果你使用 Dagger 2)在依赖图中提供每个对象需要消耗多少时间。

不幸的是生产者不是为 Android 设计的，有一些缺陷：
- 使用 Guava 作为依赖项（可能导致 64k 方法问题并增加构建时间）
- 不是非常快（注入机制可以根据设备阻塞几毫秒至几十毫秒的主线程）
- 不要使用 @Inject 注解（代码变得更混乱一些）

虽然我们对后两者不能做太多的事情，但是第一个可以在 Android 项目中让 Producers 妥协。

## 使用 RxJava 的异步注入
幸运的是，很多 Android 开发者使用 RxJava（和 [RxAndroid](https://github.com/ReactiveX/RxAndroid)）在我们的应用程序中创建异步代码。让我们尝试在 Dagger 2 异步注入中使用它。

## 异步 @Singleton 注入
这是我们沉重的对象：
```java
@Provides
@Singleton
HeavyExternalLibrary provideHeavyExternalLibrary() {
    HeavyExternalLibrary heavyExternalLibrary = new HeavyExternalLibrary();
    heavyExternalLibrary.init(); //This method takes about 500ms
    return heavyExternalLibrary;
}
```
现在让我们创建另外的 provide...() 方法，它返回 Observable \<HeavyExternalLibrary> 对象，它将异步调用这个代码：

```java
@Singleton
@Provides
Observable<HeavyExternalLibrary> provideHeavyExternalLibraryObservable(final Lazy<HeavyExternalLibrary> heavyExternalLibraryLazy) {
    return Observable.create(new Observable.OnSubscribe<HeavyExternalLibrary>() {
        @Override
        public void call(Subscriber<? super HeavyExternalLibrary> subscriber) {
            subscriber.onNext(heavyExternalLibraryLazy.get());
        }
    }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());
}
```
让我们逐行分析一下：
- @Singleton - 重要的是要记住，它将是 Observable 对象的单个实例，而不是 HeavyExternalLibrary。 Singleton 也阻止了创建额外的 Observable 对象。
- @Provides - 因为这个方法也是 @Module 注解类的一部分。你还记得 [Dagger 2 API](http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/) 吗？
  - Lazy \<HeavyExternalLibrary> heavyExternalLibraryLazy 对象可防止由Dagger内部初始化 HeavyExternalLibrary 对象（否则在调用 provideHeavyExternalLibraryObservable（）对象的时刻已经创建）。
- Observable.create（...）代码 - 每当此 Observable 将被订阅时，它将通过调用 heavyExternalLibraryLazy.get（）来返回 heavyExternalLibrary 对象。
- .subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())。 - 默认情况下， RxJava 代码在创建 Observable 的线程上执行。 这就是为什么我们应该将执行移动到后台线程（在这种情况下是 Schedulers.io（）），并且只是注意主线程（AndroidSchedulers.mainThread（））上的结果。

我们的 Observable 像图中的任何其他对象一样被注入。但是对象 heavyExternalLibrary 本身稍后可用：
```java
public class SplashActivity {

	@Inject
	Observable<HeavyExternalLibrary> heavyExternalLibraryObservable;

	//This will be injected asynchronously
	HeavyExternalLibrary heavyExternalLibrary; 

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate();
		//...
		heavyExternalLibraryObservable.subscribe(new SimpleObserver<HeavyExternalLibrary>() {
            @Override
            public void onNext(HeavyExternalLibrary heavyExternalLibrary) {
	            // 我们的依赖将从此刻起可用
	            SplashActivity.this.heavyExternalLibrary = heavyExternalLibrary;
            }
        });
	}
}
```

## 异步新实例注入

上面的代码介绍了如何注入单例对象。如果我们想要异步注入新的实例呢？

确保我们的对象不是 @Singleton 注释了：
```java
@Provides
HeavyExternalLibrary provideHeavyExternalLibrary() {
    HeavyExternalLibrary heavyExternalLibrary = new HeavyExternalLibrary();
    heavyExternalLibrary.init(); //This method takes about 500ms
    return heavyExternalLibrary;
}
```
另外我们的 Observable \<HeavyExternalLibrary> 提供者方法应该稍微改变一下。 我们不能使用 Lazy \<HeavyExternalLibrary>，因为它仅在调用 get（）方法时第一次创建新实例（有关更多详细信息，请参见 [Lazy documentation](http://google.github.io/dagger/api/latest/dagger/Lazy.html)）。

这里是更新的代码：
```java
@Singleton
@Provides
Observable<HeavyExternalLibrary> provideHeavyExternalLibraryObservable(final Provider<HeavyExternalLibrary> heavyExternalLibraryProvider) {
    return Observable.create(new Observable.OnSubscribe<HeavyExternalLibrary>() {
        @Override
        public void call(Subscriber<? super HeavyExternalLibrary> subscriber) {
            subscriber.onNext(heavyExternalLibraryProvider.get());
        }
    }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());
}
```
我们的 Observable \<HeavyExternalLibrary> 可以是一个 singleton，但每当我们调用 subscribe（）时，我们都会在 onNext（）调用中获得 HeavyExternalLibrary 的新实例：
```java
heavyExternalLibraryObservable.subscribe(new SimpleObserver<HeavyExternalLibrary>() {
    @Override
    public void onNext(HeavyExternalLibrary heavyExternalLibrary) {
        //New instance of HeavyExternalLibrary
    }
});
```

## 完全异步注入
还有另外一种方法可以在 Dagger 2 中用 RxJava 进行异步注入。我们可以简单地用 Observable 来包装整个注入过程。

假设我们的注入是以这种方式执行的（代码来自 [GithubClient](https://github.com/frogermcs/GithubClient/) 示例项目）：
```java
public class SplashActivity extends BaseActivity {

    @Inject
    SplashActivityPresenter presenter;
    @Inject
    AnalyticsManager analyticsManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }

    //This method is called in super.onCreate() method
    @Override
    protected void setupActivityComponent() {
        final SplashActivityComponent splashActivityComponent = GithubClientApplication.get(SplashActivity.this)
                .getAppComponent()
                .plus(new SplashActivityModule(SplashActivity.this));
        splashActivityComponent.inject(SplashActivity.this);
    }
}
```
为了使它异步，我们只需要用 Observable 包装 setupActivityComponent（）方法：
```java
@Override
protected void setupActivityComponent() {
    Observable.create(new Observable.OnSubscribe<Object>() {
        @Override
        public void call(Subscriber<? super Object> subscriber) {
            final SplashActivityComponent splashActivityComponent = GithubClientApplication.get(SplashActivity.this)
                    .getAppComponent()
                    .plus(new SplashActivityModule(SplashActivity.this));
            splashActivityComponent.inject(SplashActivity.this);
            subscriber.onCompleted();
        }
    })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new SimpleObserver<Object>() {
                @Override
                public void onCompleted() {
                    //Here is the moment when injection is done.
                    analyticsManager.logScreenView(getClass().getName());
                    presenter.callAnyMethod();
                }
            });
}
```
正如所注释的，所有 @Inject 注释对象将在未来一段时间注入。在返回注入过程中会是异步的，对主线程不会有太大的影响。

当然，为 subscribeOn（）创建 Observable 对象和附加线程并不是完全免费的 - 这也需要一些时间。这与生产者代码产生的影响非常相似。
![|center](https://cdn-images-1.medium.com/max/800/0*ufeyAoFyb3fTPo5X.png)
应用程序启动时间减少到 253ms（onCreate 方法在启动活动中为 119ms）

