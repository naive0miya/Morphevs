# Dagger 2 的依赖注入- 生产者

> 原文 (mirekstanek.online) ：[Dependency injection with Dagger 2 — Producers](https://mirekstanek.online/dependency-injection-with-dagger-2%e2%80%8a-%e2%80%8aproducers/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

这篇文章的第一个版本最初写在我的开发博客上：http://frogermcs.github.io/

这篇文章是在 Android 中显示依赖注入与 Dagger 2 框架的系列文章的一部分。今天我们来看看 Dagger Producers - Dagger 2 的扩展，它实现了 Java 中的异步依赖注入。

## 初始化性能问题
我们都知道 Dagger 2 是非常好的依赖注入框架。但即使是所有这些微观优化，依然存在注入依赖关系 - “重”的第三方和/或我们的主线程阻塞代码的性能问题。依赖注入是在尽可能短的时间内在正确的地方交付请求的依赖关系的过程 - 这就是 Dagger 2 所做的很好的一点。但是 DI 也是关于创建这些依赖关系的。如果我们需要花费数百毫秒的时间来创建它们，那么在纳秒内传递依赖关系有什么意义呢？

当我们的应用程序创建了一堆严重的单例（现在由 Dagger 2 立即提供服务）时，也许这并不是那么重要。但是我们仍然需要一个时间窗口来创建它们 - 在大多数情况下，它是应用程序的启动时间。

这个问题（并提示如何调试）在我之前的一篇博文中被描述得更为广泛：[Dagger 2 - 图形创建性能](http://frogermcs.github.io/dagger-graph-creation-performance/)。

在很短的时间内，让我们想象一下这种情况 - 您的应用程序具有初始屏幕（SplashScreen），在应用程序启动后立即执行所有需要的操作：

- 初始化所有跟踪库（Google Analytics，Crashlytics），并向其发送第一批数据
- 为 API 和/或数据库通信创建整个堆栈
- 为我们的视图提供逻辑（MVP 中的演示者，MVVM 中的 ViewModels 等）

即使我们的代码被很好地优化，仍然有一些外部库需要几十或几百毫秒来初始化。 在我们的启动屏幕将被呈现之前，所有请求的依赖关系（及其依赖关系）必须被初始化和交付。 这意味着启动时间是每个启动时间的总和。

[AndroidDevMetrics](https://github.com/frogermcs/androiddevmetrics) 测量的示例堆栈如下所示：
![|center](https://cdn-images-1.medium.com/max/800/0*FI18ZzjCho5-5fNC.png)
用户将在大约 600ms 内看到 SplashActivity（+附加系统工作） - 所有初始化时间的总和。

## 生产者 - 异步依赖注入
Dagger 2 有个扩展名为生产者，或多或少地为我们解决这些问题。

这个想法很简单 - 整个初始化过程可以在后台线程上执行，然后传递给应用程序的主线程。

与 Dagger 同步注入类似，我们有几个注解用于注入过程:

### @ProducerModule
和 @Module 一样，这个用来标记传递依赖关系的类。感谢这一点，Dagger 将知道在哪里找到所需的依赖关系。

### @Produces
与 @Provide 类似，这个注解用于在 @ProducerModule 注释类中标记返回依赖关系的方法。 @Produces 注释的方法可以返回 ListenableFuture \<T> 或对象本身（它也将在给定的后台线程中初始化）。

### @ProductionComponent
像 @Component 一样，它负责依赖性传递。 这是我们的代码和 @ProducerModule 之间的桥梁。 与 @Component 唯一的区别是我们不能决定依赖关系的范围。 这意味着，每个组件实例贡献的每个 Produces 方法在每个组件实例中最多被调用一次，不管这个绑定被用作依赖项的次数。

这意味着每个由 @ProductionComponent 提供服务的对象将是一个单一的实例（只要我们从这个特定的组件获取它）。

生产者的文件是相当不错的，所以没有意义在这里复制它。相反，只要看看：[Dagger 2 Producers 文档](http://google.github.io/dagger/producers.html)。


## 生产者的价值
在我们开始实施之前，有一些事情值得一提。 生产者比 Dagger 2 本身更复杂一些。看起来，移动应用程序不是他们主要的使用目的，重要的是要知道几件事：

- 生产者使用 Guava 库，并建立在 ListenableFuture 类的顶部。这意味着你必须在你的应用程序中处理 15K 的其他方法。可能意味着你也需要处理 Proguard 和更长的编译时间。
- 正如您稍后将看到的，创建 ListenableFutures 不会花费任何东西。 所以，如果你指望生产者将帮助你从10毫秒到0毫秒的优化，你可能是错的。 但是如果规模更大（100ms - 10ms），你会发现能寻找到。
- 现在没有办法使用 @Inject 注解，所以你必须手工处理 ProductionComponents。它可以使你的标准化，干净的代码混乱。 在[这里](http://stackoverflow.com/questions/35617378/injects-after-produces)你可以找到一个很好的尝试与 @Inject 注释的间接解决方案。

## 示例应用
如果你仍然想与生产者打交道，让我们更新我们的 [GithubClient](https://github.com/frogermcs/GithubClient/) 应用程序，以在注射过程中使用它们。在实施之前和之后，我们将使用 [AndroidDevMetrics](https://github.com/frogermcs/androiddevmetrics) 来衡量应用的启动时间，并比较结果。

[GithubClient](https://github.com/frogermcs/GithubClient/tree/1c14683691e0e7af17b26055a0fd041d4a7df424) 应用程序的版本就在生产者更新之前。其测量的平均发射时间如下所示：
![|center](https://cdn-images-1.medium.com/max/800/0*OqbY3SjjOd8wvRLk.png)
我们的计划是处理 UserManager 及其所有依赖关系使用生产者。

## 配置
我们将尝试 Dagger v2.1（但生产者也可以使用当前版本2.0）。

让我们将新版本的 Dagger 添加到我们的项目中：
app/build.gradle:
```java
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
apply plugin: 'com.frogermcs.androiddevmetrics'

repositories {
    maven {
        url "https://oss.sonatype.org/content/repositories/snapshots"
    }
}
//...

dependencies {
    //...

    //Dagger 2
    compile 'com.google.dagger:dagger:2.1-SNAPSHOT'
    compile 'com.google.dagger:dagger-producers:2.1-SNAPSHOT'
    apt 'com.google.dagger:dagger-compiler:2.1-SNAPSHOT'

    //...
}
```
正如你所看到的，生产者作为一个新的依赖关系被提供，在 Dagger 2 库旁边。另外值得一提的是，Dagger v2.1 最后并不需要 org.glassfish:javax.annotation:10.0-b28  依赖。

## 生产者模块
现在让我们把代码从 GithubApiModule 移动到新创建的 GithubApiProducerModule。原始的源代码可以在这里找到：[GithubApiModule](https://github.com/frogermcs/GithubClient/blob/1c14683691e0e7af17b26055a0fd041d4a7df424/app/src/main/java/frogermcs/io/githubclient/data/api/GithubApiModule.java)。
GithubApiProducerModule.java
```java
@ProducerModule
public class GithubApiProducerModule {

    @Produces
    static OkHttpClient produceOkHttpClient() {
        final OkHttpClient.Builder builder = new OkHttpClient.Builder();
        if (BuildConfig.DEBUG) {
            HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
            logging.setLevel(HttpLoggingInterceptor.Level.BODY);
            builder.addInterceptor(logging);
        }

        builder.connectTimeout(60 * 1000, TimeUnit.MILLISECONDS)
                .readTimeout(60 * 1000, TimeUnit.MILLISECONDS);

        return builder.build();
    }

    @Produces
    public Retrofit produceRestAdapter(Application application, OkHttpClient okHttpClient) {
        Retrofit.Builder builder = new Retrofit.Builder();
        builder.client(okHttpClient)
                .baseUrl(application.getString(R.string.endpoint))
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create());
        return builder.build();
    }

    @Produces
    public GithubApiService produceGithubApiService(Retrofit restAdapter) {
        return restAdapter.create(GithubApiService.class);
    }

    @Produces
    public UserManager produceUserManager(GithubApiService githubApiService) {
        return new UserManager(githubApiService);
    }

    @Produces
    public UserModule.Factory produceUserModuleFactory(GithubApiService githubApiService) {
        return new UserModule.Factory(githubApiService);
    }
}
```
看起来像？这很好，我们只更新了：
- @Module 到 @ProducerModule
- @Provides @Singleton到 @Produces。你还记得吗？在生产者中，我们默认有单个实例。

UserModule.Factory 依赖关系只是在应用程序的逻辑原因下才添加的。

## 生产者组件
现在让我们来创建一个将提供 UserManager 实例的 @ProductionComponent：
```java
@ProductionComponent(
        dependencies = AppComponent.class,
        modules = GithubApiProducerModule.class
)
public interface AppProductionComponent {
    ListenableFuture<UserManager> userManager();

    ListenableFuture<UserModule.Factory> userModuleFactory();
}
```
再次，非常类似于普通的 [Dagger 的 @Component](https://github.com/frogermcs/GithubClient/blob/master/app/src/main/java/frogermcs/io/githubclient/AppComponent.java)。
ProductionComponent 也是以与标准组件非常类似的方式构建的：
```java
AppProductionComponent appProductionComponent = DaggerAppProductionComponent.builder()
    .executor(Executors.newSingleThreadExecutor())
    .appComponent(appComponent)
    .build();
```
额外的参数是 Executor 实例，它告诉 ProductionComponent 应该创建哪个线程的依赖关系。 在我们的例子中，我们使用单线程执行程序，但是当然，提高并行性和使用多线程执行程序并不是问题。

## 获取依赖
正如我所说，目前没有办法使用 @Inject 注释。相反，我们必须直接询问 ProductionComponent（你可以在 [SplashActivityPresenter](https://github.com/frogermcs/GithubClient/blob/master/app/src/main/java/frogermcs/io/githubclient/ui/activity/presenter/SplashActivityPresenter.java) 中找到这个代码）：
这里重要的是，当你第一次调用 appProductionComponent.userManager () 时，对象初始化就开始了。 这个 UserManager 对象将被缓存。 这意味着每个绑定具有与其封闭组件实例相同的生命周期。

几乎就是这一切。当然，你应该记住，在调用 Future.onSuccess () 之前，userManager 实例将是 null。

## 性能
最后让我们来看看现在注入性能如何：
![|center](https://cdn-images-1.medium.com/max/800/0*_zHVD_R7wlBG-nFR.png)
是的，这是正确的 - 在这种情况下，平均值约为15毫秒。它仍然少于同步注入（平均25毫秒），但没有你想象的那么少。这是因为生产者不像 Dagger 那样轻便。

所以现在是你的决定 - 如果值得处理 Guava，Proguard 和代码复杂性，那么这些优化。

请记住，如果您认为生产者不适合您的应用程序，那么您可以尝试使用 RxJava 或您应用程序中使用的其他异步代码来打包注入。

## 示例代码 
所描述项目的完整源代码在 [Github](https://github.com/frogermcs/GithubClient) 存储库中获取。

