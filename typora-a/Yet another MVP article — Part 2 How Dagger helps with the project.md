# 另一篇 MVP 文章 - 第2部分：Dagger 如何帮助项目

> 原文 (Medium)：[Yet another MVP article — Part 2: How Dagger helps with the project](https://hackernoon.com/yet-another-mvp-article-part-2-how-dagger-helps-with-the-project-90d049a45e00)
>
> 作者： [Mohsen Mirhoseini](https://hackernoon.com/@mirhoseini)

[TOC]

## 你能解释一下 Dagger 是如何连接核心中的模块和 MVP 层的吗？

![@Dagger diagram](https://ws4.sinaimg.cn/large/006tNc79gy1froo46fmddj30f50cxq94.jpg)
Dagger2 使用模块，组件和子组件知道什么如何注入依赖关系。你可以找到更多的文章来解释 Dagger 里面发生的事情。

从 Dagger 图底部到顶部我可以提到 CacheModule 和 SearchModule，它们提供 View 和 Presenter：

```java
package com.mirhoseini.marvel.character.cache;

import dagger.Module;
import dagger.Provides;

@Module
class CacheModule {

    private CacheView view;

    CacheModule(CacheView view) {
        this.view = view;
    }

    @Provides
    public CacheView provideView() {
        return view;
    }

    @Provides
    @Cache
    public CachePresenter providePresenter(CachePresenterImpl presenter) {
        presenter.bind(view);
        return presenter;
    }

}
```

```java
package com.mirhoseini.marvel.character.search;

import dagger.Module;
import dagger.Provides;

@Module
class SearchModule {

    private SearchView view;

    SearchModule(SearchView view) {
        this.view = view;
    }

    @Provides
    public SearchView provideView() {
        return view;
    }

    @Provides
    @Search
    public SearchInteractor provideInteractor(SearchInteractorImpl interactor) {
        return interactor;
    }

    @Provides
    @Search
    public SearchPresenter providePresenter(SearchPresenterImpl presenter) {
        presenter.bind(view);
        return presenter;
    }

}
```

首先，注意提供方法的返回类型，它们都返回接口，而不是接口的实现！ 这就是我们如何将实现替换为进一步测试的方法，而且我必须提到这是 [**SOLID**](https://www.novoda.com/blog/designing-something-solid/) 原则中 open / closed，Liskov 替换和依赖倒置原则的混合。 

其次，不要忘记我们仍然在核心模块中，我们不知道它将如何被视图使用。 

最后，这两个模块被 CacheSubComponent 和 SearchSubComponent 使用，它们是应用组件的子组件，我将在 app 模块中解释。 

我们在核心部分还有两个模块，可以帮助我们提供 retrofit 依赖关系，ApiModule 和 ClientModule。 熟悉了  retrofit2 会帮助你理解内在的东西，然而，我将在本文的下一部分解释所有有关它的事情。 

这两个模块和 DatabaseModule 都被 ApplicationComponent 使用，我将在 app 模块中进行解释。

## Dagger 在 app 模块里起什么作用？

应用程序模块最重要的部分是应用组件，它使用 AndroidModule 和 ApplicationModule 来提供所有的应用注入。 

## ApplicationComponent 内部是什么：

### 1. AndroidModule：

这个模块提供了应用程序上下文和应用程序资源，这些资源在开发安卓系统时总是很方便的，例如，服务或不同的资源。 一些开发人员在 ApplicationModule 中实现了这个模块，但是我更喜欢为了更多的权限而把它分开。 

该模块将在扩展 Application 类的 [MarvelApplication](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/MarvelApplication.java) 类中创建：

```java
package com.mirhoseini.marvel;

import android.app.Application;

public abstract class MarvelApplication extends Application {

    private static ApplicationComponent component;

    public static ApplicationComponent getComponent() {
        return component;
    }

    @Override
    public void onCreate() {
        super.onCreate();

        initApplication();

        component = createComponent();
    }

    public ApplicationComponent createComponent() {
        return DaggerApplicationComponent.builder()
                .androidModule(new AndroidModule(this))
                .build();
    }

    public abstract void initApplication();

}
```

正如你可能已经提到的那样，这个类是抽象的，它通过实现 initApplication ( ) 方法来调试或发布 buildType，这些方法对于发布版本或调试版本来说是非常方便的。

在这个示例中，我使用了调试 MarvelApplicationImpl 来安装 [Timber](https://github.com/mirhoseini/marvel/blob/master/app/src/debug/java/com/mirhoseini/marvel/MarvelApplicationImpl.java) 库，以避免在发布版本中使用 Timber 日志：

```java
package com.mirhoseini.marvel;

import timber.log.Timber;

public class MarvelApplicationImpl extends MarvelApplication {

    @Override
    public void initApplication() {

        // initialize Timber in debug version to log
        Timber.plant(new Timber.DebugTree() {
            @Override
            protected String createStackElementTag(StackTraceElement element) {
                // adding line number to logs
                return super.createStackElementTag(element) + ":" + element.getLineNumber();
            }
        });

    }
}
```

### 2. ApplicationModule:

几乎所有的应用需求都在这个模块中提供，

- isDebug：使用 BuildConfig.DEBUG 布尔变量来检查应用程序的运行实例是否处于调试模式，并使用它来记录 logcat 中的网络 API 调用。
- networkTimeoutInSeconds，cacheSize，cacheMaxAge，cacheMaxStale，cacheDir：提供创建 OkHttp 客户端进行 Retrofit 时使用的网络参数。
- 端点：为 retrofit 提供 API 的端点。
- appScheduler：提供 RxAndroid 调度程序，我将在 RxJava 部分解释。
- isConnect：提供处理脱机情况的网络状态。

### 3. core 模块的 [ApiModule](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/domain/ApiModule.java)，[ClientModule ](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/domain/ClientModule.java)

### 4. core 模块的 [DatabaseModule](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/database/DatabaseModule.java)

### 5. SubComponents :

关于 Dagger 的好处是子组件，你可以在你的主要应用组件中添加子组件，并使用内部的所有提供程序并添加你的。 

在这个示例中，[SearchSubComponent](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/character/search/SearchSubComponent.java) 和 [CacheSubComponent](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/character/cache/CacheSubComponent.java) 加起来就是 ApplicationComponent，使其更加精彩。 

## SearchSubComponent 和 CacheSubComponent：

让我们再看看 ApplicationComponent：

```java
package com.mirhoseini.marvel;
/*...*/

@Singleton
@Component(modules = {
        AndroidModule.class,
        ApplicationModule.class,
        ApiModule.class,
        DatabaseModule.class,
        ClientModule.class
})
public interface ApplicationComponent {
        
    void inject(MainActivity activity);
    SearchSubComponent plus(AppSearchModule module);
    CacheSubComponent plus(AppCacheModule module);
        
}
```

正如你所看到的，我们有注射方法帮助注射，但是这些加方法是什么？

首先，我不得不提及 inject 和 plus 这个名字！ 你可以把它重新命名为你喜欢的任何东西，而 Dagger 负责方法的输入和输出以知道它们的用途。 但是请不要乱动这个名字，因为你的团队成员稍后必须阅读此代码。

使用 plus 方法，你可以要求 Dagger 将一个 SubComponent 添加到 ApplicationComponent，并在该SubComponent 中执行 inject 方法。

plus 方法使用模块并返回使用该模块的 SubComponent。

```java
package com.mirhoseini.marvel.character.search;

import dagger.Subcomponent;

@Search
@Subcomponent(modules = {
        AppSearchModule.class
})
public interface SearchSubComponent {

    void inject(CharacterSearchFragment fragment);

}
```

## 那么好吧... Dagger  整合开始了吗？

通过[清单](https://github.com/mirhoseini/marvel/blob/master/app/src/main/AndroidManifest.xml)中的一些变化，我们可以要求 Android 使用 MarvelApplication 实现作为我们的应用程序的起点：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.mirhoseini.marvel">

    <!-- *** -->

    <application
        android:name=".MarvelApplicationImpl"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
       
        <!-- *** -->
       
    </application>

</manifest>
```

在 MarvelApplication 类中，我们曾经通过扩展 BaseActivity 和 BaseFragment 类来创建应用程序组件，并在活动和片段中使用它。 它们都是抽象的，注入依赖方法必须在活动或片段中实现。 

## 如何在 dagger 里遨游？

如果你喜欢，你可以做更多的 dagger ！

看看这篇文章，它解释了如何 [Dagger 风格](http://frogermcs.github.io/inject-everything-viewholder-and-dagger-2-example/) 的一切...我已经完成了这个示例项目，通过在 AppModule 中使用 [AppSearchModule](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/character/search/AppSearchModule.java) 和 [AppCacheModule](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/character/cache/AppCacheModule.java) 从核心模块中扩展 SearchModule 和 CacheModule。

