# 在 Android 上写一个模块化的项目

> 原文 (Medium)：[Writing a modular project on Android](https://medium.com/mindorks/writing-a-modular-project-on-android-304f3b09cb37)
>
> 作者：[karntrehan](https://medium.com/@karntrehan?source=post_header_lockup)

[TOC]

当我们在 Android Studio 上创建一个新项目时，它会为我们提供一个模块，即应用程序模块。这是我们大多数人编写我们整个应用程序的地方。每次点击运行按钮都会触发我们整个模块上的 gradle build，并检查其中的所有文件是否有变化。这就是为什么 gradle build 可能需要[10分钟](https://eng.uber.com/android-monorepo/)才能完成更大的应用程序并且会减慢开发者的[输出](https://imgs.xkcd.com/comics/compiling.png)。

为了解决这个问题，像 Uber 这样的复杂应用程序决定对它们的应用程序进行模块化，并且从中[获得了很多](https://www.youtube.com/watch?v=j6CiHlapado)。以下是有一个模块化项目的一些优势。

- [更快](https://proandroiddev.com/modular-architecture-for-faster-build-time-d58397cb7bfe)的 gradle builds。
- 通过应用程序 / 模块重新使用公共功能 
- 轻松插入[即时应用程序](https://developer.android.com/topic/instant-apps/overview.html#features)。
- 更好的团队合作，因为一个人可以单独负责一个模块。
- 更流畅的 git 流程。

由于上述优点，当我开始实现 [Posts](https://github.com/karntrehan/Posts/) 应用程序时，我始终记住从第一天开始使用模块化方法。 安卓团队为我们提供了一些[工具](https://developer.android.com/google/play/publishing/multiple-apks.html#CreatingApks)，但我确实遇到了一些障碍。 以下是我的一些经验教训: 

## 我该如何分割我的模块？

你的应用程序有一系列流程，例如，谷歌 Play 有应用程序的详细流程，其中包含了摘要、描述细节、屏幕截图和评论活动。 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frotsq4airj30m8091abp.jpg)

所有这些都可以归入相同的模块 - app-details。

你的应用程序可以包含多个这样的流程模块，如身份验证、设置、登陆等。 还有一些模块不需要用户界面元素来呈现，例如通知、分析、第一次提取等等。 这些模块包含与该流程相关的活动、视图、存储库、实体和依赖注入。 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frotsuvkkcj308o0e70t3.jpg)

但是这些模块总是想要共享一些常见的功能和实用工具。 这就是为什么你需要一个核心模块。 

## 核心模块是什么？

核心模块是你项目中的一个简单的库模块。 核心模块可以，除其他外， 

- 为依赖注入框架提供全局依赖，如 [Retrofit](https://github.com/karntrehan/Posts/blob/master/core/src/main/java/com/karntrehan/posts/core/di/NetworkModule.kt)，[SharedPreferences](https://github.com/karntrehan/Posts/blob/master/core/src/main/java/com/karntrehan/posts/core/di/StorageModule.kt) 等。
- 包含实用工具类和[扩展功能](https://github.com/karntrehan/Posts/tree/master/core/src/main/java/com/karntrehan/posts/core/extensions)。
- 包含[全局类](https://github.com/karntrehan/Posts/blob/master/core/src/main/java/com/karntrehan/posts/core/networking/Outcome.kt)和回调。
- 在应用程序类中[实例化库](https://github.com/karntrehan/Posts/blob/master/core/src/main/java/com/karntrehan/posts/core/application/CoreApp.kt) Firebase Analytics，Crashlytics，LeakCanary，Stetho 等库。

## 如何使用第三方库？

核心模块的主要职责之一是为你的特性模块提供外部依赖。 这使得在所有功能中分享同一个库的版本变得很容易。 只要在你的核心模块中使用 api 来标记依赖关系，你所有依赖的功能模块都可以接收到它们。 

```groovy
dependencies {
    api fileTree(include: ['*.jar'], dir: 'libs')
    api deps.support.appCompat
    api deps.support.recyclerView
    api deps.support.cardView
    api deps.support.support
    api deps.support.designSupport

    api deps.android.lifecycleExt
    api deps.android.lifecycleCommon
    api deps.android.roomRuntime
    api deps.android.roomRx

    api deps.kotlin.stdlib

    api deps.reactivex.rxJava
    api deps.reactivex.rxAndroid

    api deps.google.dagger
    kapt deps.google.daggerProcessor

    api deps.square.picasso
    api deps.square.okhttpDownloader

    api deps.square.retrofit
    api deps.square.okhttp
    api deps.square.gsonConverter
    api deps.square.retrofitRxAdapter

    implementation deps.facebook.stetho
    implementation deps.facebook.networkInterceptor

    testApi deps.test.junit
    androidTestApi deps.test.testRunner
    androidTestApi deps.test.espressoCore
}
```

有依赖性的可能性只在 feature-a 模块中有用，但不在 feature-b 中，这两者都依赖于核心。在这种情况下，我会建议使用 api 在核心中定义你的依赖关系，因为 proguard 会考虑不将其包含在 feature-b 即时应用程序中。

## 我如何使用 Room？

这个问题困扰了我很长时间。 我们希望将我们的数据库定义为核心模块，因为它是我们的应用程序希望共享的一个共同功能。 为了让 Room 工作，你需要一个数据库文件，其中包含所有提到的实体类。 

```kotlin
@Database(entities = [Post::class, User::class, Comment::class], version = 1,exportSchema = false)
abstract class PostDb : RoomDatabase() {
    abstract fun postDao(): PostDao
    abstract fun userDao(): UserDao
    abstract fun commentDao(): CommentDao
}
```

但是，如上所述，我们的实体类在依赖  feature 模块中定义，而 core 模块无法访问它们。 这就是我遇到障碍的地方，经过一番思考，做了你能做的最好的事情，向 [Yigit](https://github.com/yigit) 寻求帮助。 

Yigit 阐明，你必须在每个特征模块中创建一个新的数据库文件，并且每个模块都有一个数据库。 

这有一些优点：

- [迁移](https://medium.com/google-developers/understanding-migrations-with-room-f01e04b07929)是模块化的。
- 即时应用只包含他们需要的表格。
- 查询速度会更快。

缺点：

- 跨模块数据关系将不可能。

注意：不要忘记将以下依赖关系添加到你的 feature模块中，以使 Room 的注解生效。

```groovy
kapt "android.arch.persistence.room:compiler:${versions.room}"
```

## 我如何使用 Dagger 2？

与 Room 同样的问题也被 Dagger 击中。核心模块中的我的应用程序类将无法访问和初始化我的特征模块组件。这是依赖组件的完美用例。 

你的核心组件提到它希望向依赖组件公开的依赖关系 

```kotlin
@Singleton
@Component(modules = [AppModule::class, NetworkModule::class, StorageModule::class, ImageModule::class])
interface CoreComponent {

    fun context(): Context

    fun retrofit(): Retrofit

    fun picasso(): Picasso

    fun sharedPreferences(): SharedPreferences

    fun scheduler(): Scheduler
}
```

模块组件将 CoreComponent 定义为依赖项并使用传递的依赖项 

```kotlin
@ListScope
@Component(dependencies = [CoreComponent::class], modules = [ListModule::class])
interface ListComponent {
    fun inject(listActivity: ListActivity)
}

@Module
@ListScope
class ListModule {

    /*Uses parent's provided dependencies like Picasso, Context and Retrofit*/
    @Provides
    @ListScope
    fun adapter(picasso: Picasso): ListAdapter = ListAdapter(picasso)

    @Provides
    @ListScope
    fun postDb(context: Context): PostDb = Room.databaseBuilder(context, PostDb::class.java, Constants.Posts.DB_NAME).build()

    @Provides
    @ListScope
    fun postService(retrofit: Retrofit): PostService = retrofit.create(PostService::class.java)
}
```

## 我在哪里初始化我的组件？

我为特性的所有组件创建了一个 singleton 持有器。 此持有器用于创建、维护和销毁组件实例。 

```kotlin
@Singleton
object PostDH {
    private var listComponent: ListComponent? = null

    fun listComponent(): ListComponent {
        if (listComponent == null)
            listComponent = DaggerListComponent.builder().coreComponent(CoreApp.coreComponent).build()
        return listComponent as ListComponent
    }

    fun destroyListComponent() {
        listComponent = null
    }
}
```

注意：不要忘记将以下依赖项添加到你的 feature 模块中，使 Dagger 的注解过程发挥作用。

```groovy
kapt "com.google.dagger:dagger-compiler:${versions.dagger}"
```

## 结束

虽然有一些棘手的部分可以将你的应用转换成模块，其中一些我试图在上面解决，但优势是深远的。 如果你用你的模块碰到任何障碍，请在下面提到它们，我们可以共同努力解决问题。 

谢谢阅读！

