# 现代 Android Kotlin 开发 - 第4部分

> 原文 (Medium)：[Modern Android development with Kotlin Part 4](https://proandroiddev.com/modern-android-development-with-kotlin-part-4-4ac18e9868cb?source=bookmarks---------5----------------)
>
> 作者：[Mladen Rakonjac](https://proandroiddev.com/@mladenrakonjac?source=bookmarks---------5----------------)

[TOC]

> 很难找到一个覆盖 Android 开发中所有新功能的项目，所以我决定写一个。

1. Android Studio 3 [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
2. Kotlin language [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
3. Build Variants [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
4. ConstraintLayout [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
5. Data binding library [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
6. MVVM architecture + repository pattern + Android Manager Wrappers [Part 2](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-2-17444fcdbe86)
7. RxJava2 and how it helps us in architecture  [Part 3](https://proandroiddev.com/modern-android-development-with-kotlin-part-3-8721fb843d1b)
8. Dagger 2.11, what is Dependency Injection, why you should use it. 
9. Retrofit (with Rx Java2)
10. Room (with Rx Java2)

## 什么是依赖注入？

我们来看看我们的 GitRepoRepository 类：

```kotlin
class GitRepoRepository(private val netManager: NetManager) {

    private val localDataSource = GitRepoLocalDataSource()
    private val remoteDataSource = GitRepoRemoteDataSource()

    fun getRepositories(): Observable<ArrayList<Repository>> {
      ...
    }
}
```

我们可以说我们的 GitRepoRepository 类依赖于三个对象：netManager，localDataSource 和 remoteDataSource。数据源在 GitRepoRepository 中初始化，而 netManager 通过构造函数提供。换句话说，我们将 netManager 注入到 GitRepoRepository 中。

依赖注入是一个非常简单的概念：说出你需要什么和别人将负责提供它。

让我们看看我们构建我们的 GitRepoRepository 类（cmd + B for Mac，alt + B for Windows）的位置：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovaqx2ijj30k003wdgz.jpg)

正如你所看到的，GitRepoRepository 是在 MainViewModel 中构造的。另外，NetManager 也是在那里构建。他们是否应该注入 ViewModel？当然。GitRepoRepository 实例应该提供给 ViewModel，因为 GitRepoRepository 可以在其他 ViewModel 中使用。另一方面，我们确信只有一个 NetManager 实例可以用于整个应用程序。让我们通过构造函数来提供它。我们希望有这样的东西: 

```kotlin
class MainViewModel(private var gitRepoRepository: GitRepoRepository) : ViewModel() {
  ...
}
```

如果你可以回想一下，我们并没有在 MainActivity 中创建 MainViewModel。我们调用  	ViewModelProviders：

```kotlin
class MainActivity : AppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        val viewModel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        ...
    }
    
  ...
}
```

如前所述，ViewModelProvider 将创建新的 ViewModel 或返回现有的 ViewModel。现在我们必须将 GitRepoRepository 作为参数。怎么做？

我们需要为 MainViewModel 制作特殊的 Factory 类，因为我们不能使用标准的模型：

```kotlin
class MainViewModelFactory(private val repository: GitRepoRepository) 
         : ViewModelProvider.Factory {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(MainViewModel::class.java)) {
            return MainViewModel(repository) as T
        }

        throw IllegalArgumentException("Unknown ViewModel Class")
    }

}
```

所以，现在我们可以在我们想要构造参数时加入参数：

```kotlin
class MainActivity : AppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {
    ....

    override fun onCreate(savedInstanceState: Bundle?) {
        ...

        val repository = GitRepoRepository(NetManager(applicationContext))
        val mainViewModelFactory = MainViewModelFactory(repository)
        val viewModel = ViewModelProviders.of(this, mainViewModelFactory)
                            .get(MainViewModel::class.java)

        ...
    }

    ...
}
```

等等，我们仍然没有解决问题。我们是否应该在活动中创建 MainViewModelFactory  的实例？NO。它应该注入那里。让我们来创建一个 Injection 类，它将提供我们需要的实例的方法：

```kotlin
object Injection {

    fun provideMainViewModelFactory(context: Context) : MainViewModelFactory{
        val netManager = NetManager(context)
        val repository = GitRepoRepository(netManager)
        return MainViewModelFactory(repository)
    }
}
```

现在，我们可以从这个类注入 MainActivity.kt：

```kotlin
class MainActivity : AppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {

    private lateinit var mainViewModelFactory: MainViewModelFactory

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
      
        mainViewModelFactory = Injection.provideMainViewModelFactory(applicationContext)
        val viewModel = ViewModelProviders.of(this, mainViewModelFactory)
                            .get(MainViewModel::class.java)
        
        ...

    }
    ...
}
```

因此，现在我们的 Activity 不知道来自应用程序数据层的存储库。代码的这种组织对我们有很大的帮助，特别是对应用程序的测试。这样我们将 UI 逻辑与业务逻辑分开。

我们可以在 Injection.kt 中应用依赖注入概念：

```kotlin
object Injection {
    
    private fun provideNetManager(context: Context) : NetManager {
        return NetManager(context)
    }

    private fun gitRepoRepository(netManager: NetManager) :GitRepoRepository {
        return GitRepoRepository(netManager)
    }

    fun provideMainViewModelFactory(context: Context): MainViewModelFactory {
        val netManager = provideNetManager(context)
        val repository = gitRepoRepository(netManager)
        return MainViewModelFactory(repository)
    }
    
}
```

现在每个类都有自己的方法来提供实例。 如果你看得更清楚，这些方法每次调用它们时都会返回新实例。 应该是这样吗？ 当我们在某个存储库类中需要 NetManager 的时候，我们是不是应该创建一个 NetManager 的新实例？ 没有。 我们每个应用程序只需要一个 NetManager 实例。 我们可以说 NetManager 应该是一个 singleton。 

> 在[软件工程](https://en.wikipedia.org/wiki/Software_engineering)中，单例模式是一种[软件设计模式](https://en.wikipedia.org/wiki/Software_design_pattern)，它将[类](https://en.wikipedia.org/wiki/Class_%28computer_programming%29)的[实例化](https://en.wikipedia.org/wiki/Instantiation_%28computer_science%29)限制为一个[对象](https://en.wikipedia.org/wiki/Object_%28computer_science%29)。

让我们来实现它：

```kotlin
object Injection {

    private var NET_MANAGER: NetManager? = null

    private fun provideNetManager(context: Context): NetManager {
        if (NET_MANAGER == null) {
            NET_MANAGER = NetManager(context)
        }
        return NET_MANAGER!!
    }

    private fun gitRepoRepository(netManager: NetManager): GitRepoRepository {
        return GitRepoRepository(netManager)
    }

    fun provideMainViewModelFactory(context: Context): MainViewModelFactory {
        val netManager = provideNetManager(context)
        val repository = gitRepoRepository(netManager)
        return MainViewModelFactory(repository)
    }
}
```

> 注意：这不是在 Kotlin 中用参数实现单例的最佳解决方案。我强烈推荐这篇[文章](https://medium.com/@BladeCoder/kotlin-singletons-with-argument-194ef06edd9e)阅读更多。

这样我们确保每个应用只有一个实例。换句话说，我们可以说 NetManager 实例具有 Application 范围。

我们来看一个依赖关系图：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovawk50bj30ha0jq755.jpg)

## 我们为什么要用 Dagger？

如果你看一下 Injection，你会发现如果我们在图中有很多的依赖关系，我们需要做很多工作。Dagger 帮助我们以简单的方式管理我们的依赖和范围。

让我们导入 dagger：

```kotlin
...
dependencies {
    ...
    
    implementation "com.google.dagger:dagger:2.14.1"
    implementation "com.google.dagger:dagger-android:2.14.1"
    implementation "com.google.dagger:dagger-android-support:2.14.1"
    kapt "com.google.dagger:dagger-compiler:2.14.1"
    kapt "com.google.dagger:dagger-android-processor:2.14.1"
    
    ...
}
```

要使用 Dagger，我们需要具有扩展 DaggerApplication 类的[应用程序](https://developer.android.com/reference/android/app/Application.html)类。让我们来写 ModernApplication：

```
class ModernApplication : DaggerApplication(){

    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        TODO("not implemented")
    }

}
```

由于它扩展了 DaggerApplication ( )，因此需要实现 applicationInjector ( ) 方法，该方法应返回 AndroidInjector 的实现。稍后我会介绍 AndroidInjector。

不要忘记在 AndroidManifest.xml 文件中注册它：

```groovy
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="me.fleka.modernandroidapp">
  
    ...

    <application
        android:name=".ModernApplication"
        ...>
       ...
    </application>

</manifest>
```

首先，创建 AppModule dagger 模块。模块是具有 @Provides 注解功能的类。我们说，对于这些方法，他们是提供者，因为他们提供实例。为了使某些类作为模块，我们需要使用 @module 注解对该类进行注释。 这些注解有助于 Dagger 制作和验证图形。我们的 AppModule 将只提供一个提供应用程序上下文的函数：

```kotlin
@Module
class AppModule{

    @Provides
    fun providesContext(application: ModernApplication): Context {
        return application.applicationContext
    }
}
```

现在我们来写一个组件：

```kotlin
@Singleton
@Component(
        modules = [AndroidSupportInjectionModule::class,
            AppModule::class])
interface AppComponent : AndroidInjector<ModernApplication> {

    @Component.Builder
    abstract class Builder : AndroidInjector.Builder<ModernApplication>()
}
```

组件是一个接口，我们在其中指定应该从哪些模块实例中注入哪些类的接口。 在这种情况下，我们指定 AppModule 和 AndroidSupportInjectionModule。

AndroidSupportInjectionModule 是帮助我们将实例注入 Android 生态系统类的模块，生态系统类例如：Activities，Fragments，Services，BroadcastReceivers 或 ContentProviders。

因为我们想要使用我们的组件来注入我们的 AppComponent 必须扩展 AndroidInjector \<T> 的类。对于 T  我们使用我们的 ModernApplication 类。如果你打开 AndroidInjector 接口，你可以看到它有：

```kotlin
abstract class Builder<T> implements AndroidInjector.Factory<T> {
    @Override
    public final AndroidInjector<T> create(T instance) { ... }
    public abstract void seedInstance(T instance);
    ...
  }
}
```

Builder 类有两种方法：create(T instance) 创建 AndroidInjector 和提供该实例的 seedInsance(T instance)。在我们的例子中，我们将创建具有 ModernApplication 实例的 AndroidInjector，并将它提供给需要的实例。

要指定我们实例的类型，我们需要添加：

```kotlin
@Component.Builder
abstract class Builder : AndroidInjector.Builder<ModernApplication>()
```

到我们的 AppComponent.

总结：

- 我们有 AppComponent，它是扩展 AndroidInjector 的应用程序的主要组件
- 当我们想要构建组件时，我们需要使用一个 ModernApplication 类的实例作为参数
- ModernApplication 的实例将被提供给 AppComponent 中使用的模型中的所有其他 @Provides 方法。例如，ModernApplication 的实例将被提供给 AppModule 中的 provideContext（application：ModernApplication）方法。

现在，我们应该 Make Project。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovb0d791j301k01g0p0.jpg)

完成后，Dagger 会自动生成一些新类。对于我们的 AppComponent，Dagger 会制作 DaggerAppComponent 类。

让我们回到 ModernApplication 并创建应用程序的主要组件。该创建的组件应该在 applicationInjector（）方法中返回。

```kotlin
class ModernApplication : DaggerApplication(){

    override fun applicationInjector(): AndroidInjector<out DaggerApplication> {
        return DaggerAppComponent.builder().create(this)
    }

}
```

现在我们完成了 Dagger 所需的标准配置。

因为我们想要将实例注入到 MainActivity 类中，所以我们需要创建 MainActivityModule。

```kotlin
@Module
internal abstract class MainActivityModule {

    @ContributesAndroidInjector()
    internal abstract fun mainActivity(): MainActivity

}
```

@ContributesAndroidInjector 注解帮助 Dagger 连接需要的东西，以便我们可以在指定的活动中注入实例。

如果你回到我们的活动，你可以看到我们使用 Injection 类注入了 MainViewModelProvider。所以，我们需要在我们的 MainActivityModule 中提供 MainViewModelProvider 的提供者方法：

```kotlin
@Module
internal abstract class MainActivityModule {
    
    @Module
    companion object {

       @JvmStatic
       @Provides
       internal fun providesMainViewModelFactory(gitRepoRepository: GitRepoRepository)
        : MainViewModelFactory {
          return MainViewModelFactory(gitRepoRepository)
       }
     }

    @ContributesAndroidInjector()
    internal abstract fun mainActivity(): MainActivity

}
```

但是什么会提供 GitRepoRepository 来提供 MainViewModelFactoty ( ) 方法？

有两种选择：我们可以为它提供方法并返回新实例，或者我们可以使用 @Inject 注释构造函数。

让我们回到我们的 GitRepoRepository 并使用 @Inject 注释来注释它的构造函数：

```kotlin
class GitRepoRepository @Inject constructor(var netManager: NetManager) {
  ...
}
```

这话导致我们的 GitRepoRepository 需要 NetManager 实例，让我们对 NetManager 执行相同的操作：

```kotlin
@Singleton
class NetManager @Inject constructor(var applicationContext: Context) {
   ...
}
```

我们使用 @Singleton 注解来设置 NetManager 的范围。另外，NetManager 需要 applicationContext。 AppModule 中有一个方法提供它。

不要忘记将 MainActivityModule 添加到 AppComponent.kt 中的模块列表中：

```kotlin
@Component(
        modules = [AndroidSupportInjectionModule::class,
            AppModule::class,
            MainActivityModule::class])
interface AppComponent : AndroidInjector<ModernApplication> {

    @Component.Builder
    abstract class Builder : AndroidInjector.Builder<ModernApplication>()
}
```

 最后，我们需要将它注入到我们的 MainActivity 中。为了使 Dagger 能够在那里工作，我们的MainActivity 需要扩展 DaggerAppCompatActivity。

```
class MainActivity : DaggerAppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {
    ...
    @Inject lateinit var mainViewModelFactory: MainViewModelFactory

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val viewModel = ViewModelProviders.of(this, mainViewModelFactory)
                .get(MainViewModel::class.java)
        ...
       }

    ...
}
```

要注入 MainViewModelFactory 实例，我们需要使用 @Inject 注解。

重要提示：mainViewModelFactory 变量必须是公共的。

不需要 Injection 类的注入：

```kotlin
mainViewModelFactory = Injection.provideMainViewModelFactory(applicationContext)
```

实际上，我们现在可以删除 Injection 类，因为我们正在使用 Dagger。

## 步骤

- 我们要将 MainViewModelFactory 注入到 MainActivity 中
- 为了让 Dagger 在 MainActivity 中工作，MainActivity 必须扩展 DaggerAppCompatActivity
- 我们需要使用 @Inject 注解来注释 mainViewModelFactory
- Dagger 搜索具有返回 MainActivity 的 @ContributesAndroidInjector 注释的方法的模块。
- Dagger 搜索返回 MainViewModelFactory 实例的 provider 或 @Inject 注释构造函数。
- provideMainViewModelFactory（）返回该实例，但为了实现它，它需要 GitRepoRepository 实例
- Dagger 搜索返回 GitRepoRepository 实例的 provider 或 @Inject 注释构造函数。
- GitRepoRepository 类具有带 @Inject 注解的构造函数。但是那个构造函数需要 NetManager 实例。
- Dagger 搜索返回NetManager 实例的 provider 或 @Inject 注释的构造函数。
- NetManager 类具有带 @Inject 注释的构造函数。但是该构造函数需要应用程序上下文实例。
- Dagger 搜索返回应用程序上下文实例的 provider 。
- AppModule 具有返回应用程序上下文的提供者方法。但是那个构造函数需要 ModernApplication 实例。
- AndroidInjector 有这个提供者。

这就是它的工作原理。

## 有一种更好的自动化方式来提供 ViewModelFactory

问题：对于每个具有参数的 ViewModel，我们需要创建 ViewModelFactory 类。在 [Chris Banes](https://medium.com/@chrisbanes) 的 [Tivi](https://github.com/chrisbanes/tivi) 应用程序源代码中，我发现了很好的自动化方法。

创建 ViewModelKey.kt：

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.PROPERTY_SETTER)
@Retention(AnnotationRetention.RUNTIME)
@MapKey
annotation class ViewModelKey(val value: KClass<out ViewModel>)
```

让我们来写 DaggerAwareViewModelFactory：

```kotlin
class DaggerAwareViewModelFactory @Inject constructor(
        private val creators: @JvmSuppressWildcards Map<Class<out ViewModel>, Provider<ViewModel>>
) : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        var creator: Provider<out ViewModel>? = creators[modelClass]
        if (creator == null) {
            for ((key, value) in creators) {
                if (modelClass.isAssignableFrom(key)) {
                    creator = value
                    break
                }
            }
        }
        if (creator == null) {
            throw IllegalArgumentException("unknown model class " + modelClass)
        }
        try {
            @Suppress("UNCHECKED_CAST")
            return creator.get() as T
        } catch (e: Exception) {
            throw RuntimeException(e)
        }
    }
}
```

创建 ViewModelBuilder 模块：

```kotlin
@Module
internal abstract class ViewModelBuilder {

    @Binds
    internal abstract fun bindViewModelFactory(factory: DaggerAwareViewModelFactory):
            ViewModelProvider.Factory
}
```

将 ViewModelBuilder 添加到 AppComponent：

```kotlin
@Singleton
@Component(
        modules = [AndroidSupportInjectionModule::class,
            AppModule::class,
            ViewModelBuilder::class,
            MainActivityModule::class])
interface AppComponent : AndroidInjector<ModernApplication> {

    @Component.Builder
    abstract class Builder : AndroidInjector.Builder<ModernApplication>()
}
```

在类 MainViewModel 中添加 @Inject：

```kotlin
class MainViewModel @Inject constructor(var gitRepoRepository: GitRepoRepository) : ViewModel() {
  ...
}
```

从现在开始，我们只需要将它绑定到 Activity 的模块：

```kotlin
@Module
internal abstract class MainActivityModule {

    @ContributesAndroidInjector
    internal abstract fun mainActivity(): MainActivity

    @Binds
    @IntoMap
    @ViewModelKey(MainViewModel::class)
    abstract fun bindMainViewModel(viewModel: MainViewModel): ViewModel


}
```

不需要 MainViewModelFactory 提供程序。实际上，根本不需要 MainViewModelFactory.kt，因此你可以将其删除。

最后，在 MainActivity.kt 中改变它，所以我们有 ViewModel.Factory 类型而不是MainViewModelFactory：

```kotlin
class MainActivity : DaggerAppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {
  
    @Inject lateinit var viewModelFactory: ViewModelProvider.Factory

    override fun onCreate(savedInstanceState: Bundle?) {
      ...

        val viewModel = ViewModelProviders.of(this, viewModelFactory)
                .get(MainViewModel::class.java)
       ...
    }
    ...
}
```

感谢 [Chris Banes](https://medium.com/@chrisbanes) 提供这个优雅的解决方案。

我希望你喜欢这篇文章。关注我获得更多。在下一部分中，我们将使用 Retrofit 从 GitHub 获取真实存储库。在下下一部分中，我们将使用 Room 在本地保存它们，以便我们的应用可以离线工作。

我也在考虑编写更多的部分，我将解释如何使用 Espresso 测试我们的 UI 层，或使用单元测试来测试我们的数据层。可能还会有更多关于使用 Fastlane，Circle.ci 和 Fabric 持续交付的信息。

谢谢大家的阅读！ 😊

