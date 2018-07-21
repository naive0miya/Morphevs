# Dagger 注意事项

> 原文 (arturdryomov.online)：[A Dagger to Remember](https://arturdryomov.online/posts/a-dagger-to-remember/)
>
> 作者：[Artur Dryomov](https://arturdryomov.online/)

# 故事时间

Kotlin 和 Java 注解之间的关系很复杂。 这是从语法到工具链的过程。

有没有人记得，Kotlin 的注解在一开始就被放在方括号里吗？ 在2015年，[语法被改变为现在的形式](https://blog.jetbrains.com/kotlin/2015/04/upcoming-change-syntax-for-annotations/)。 在此之前，它是 [Inject] ，而不是 @Inject。 是的。

法在这一点上很好，但是 kapt ([Kotlin 注解处理工具](https://kotlinlang.org/docs/reference/kapt.html))来自 Kotlin 的工具链可能是... 残酷的。

## kapt : 否认

我正在开展的项目从2015年开始就开始使用 Kotlin。 在2017年初，出现了以下情况。

- 有一种增量 Kotlin 编译器。 它适用于1.1.1的每个人，相信我，那么晚是有原因的。
- kapt几乎消除了每一次增量构建的机会，因此每次更改都会进行全面重建。 这些构建被戏称为衰老，实在是太糟糕了。
- 与此同时，kapt 有机会完全破坏编译过程，需要彻底重建。 如果你看到类似 *_MemberInjector.java: error: package does not exist 的东西，你知道我在说什么。

当你在不断增长的代码库中工作时，你的构建需要5分钟，即使你只改变了一行... 这是一个可怕的经历和一个糟糕的开发环境，导致生产力下降和[潜在的财务损失](https://blog.gradle.com/quantifying-the-cost-of-builds/)。

你如何解决这个问题？ 有一个很好的方法ーー多扔点硬件进去！ 这是一个关于 [Mainframer](https://github.com/gojuno/mainframer) 是如何诞生的简短故事。

## kapt : 愤怒

使用 Mainframer 已经有一段时间了。 是的，这个问题是通过数量而不是质量来解决的，但是这是个好主意，而且规模很大。 如果你没有足够的专业知识去改变 kapt，你至少可以改变正在运行的环境。 此外，随着时间的推移，Kotlin 的工具链变得更加完善，因此获得增量构建的机会非常大。

不幸的是，Kapt 在Kotlin 1.2.21中再次发生。 将单元测试的 kapt 处理时间提高到3级是太多了，特别是如果你运行的测试比项目可执行文件本身更频繁。 我问了自己一个百万美元的问题。

> 你需要 kapt 吗？

原来在项目中只有一个注解处理依赖项。 我想你已经知道哪一个。 是的，是 [Dagger](https://google.github.io/dagger/)。 放下兔子洞，我们走了。

# 冒险时间

## 你还需要 ~~kapt~~ Dagger 吗？

我要谈谈 [Google Dagger](https://github.com/google/dagger)。 它是从 [Square Dagger](https://github.com/square/dagger) 上 forked 下来的。 Square 在运行时做了一些查找，但也使用注解处理也生成了代码，就像 Google 一样。 与此同时，谷歌版本并没有使用反射，而是提前生成所有的东西。 这真是太棒了。 无反射使用ーー更好的运行时性能和编译时上下文验证。

> 对于上下文命名，Android 开发人员可能会感到困惑。 我会把它作为一个更宽泛的术语，而不是一个框架类名称。 上下文是一个依赖容器(或者只是一个容器)。 你可以在不同的环境中观察这个命名，比如 Go 和 Spring。 你可以把它和 Google Dagger @Component 或者 Square Dagger ObjectGraph 联系起来。

不过也有不利的一面。 Square 的版本有一个小但广泛的 API。 在我看来，它几乎涵盖了你需要从依赖注入中所需的几乎所有东西。Google 的版本已经发展到了一个很大的高度，有时甚至不是一个好的方式。 这个 API 包括 Android 支持模块(让我们忘记 DaggerActivity 的存在) ，多绑定，可重用的依赖，组件和子组件，模块和生产者模块。

我正在做的项目没有使用任何神奇的东西。 见鬼，最有可能是用错了 Dagger ！

- 模型组件（如服务）具有通过构造函数传递的依赖关系。
- 演示组件，例如 ViewModel ，通过构造函数传递依赖关系作为上下文。
- 上下文是组合多个模块的结果。
- 模块包含所有依赖声明，没有任何组件对它们有 @Inject 注解。

基本上，DI 胶水与主代码库分离。

```kotlin
// Model

interface Repository {
    val content: Observable<RepositoryContent>

    class Impl : Repository {
        override val content = Observable.just(RepositoryContent())
    }
}

interface Service {
    val content: Observable<ServiceContent>

    class Impl(repository: Repository) : Service {
        override val content = repository.content.map { it.toServiceContent() }
    }
}

// Presentation

class ViewModel(context: Context) {

    @Inject lateinit var service: Service

    init {
        context.inject(this)
    }
}

// Dagger

@Module
class RepositoryModule {
    @Provides @Singleton
    fun provideRepository(): Repository = Repository.Impl()
}

@Module
class ServiceModule {
    @Provides @Singleton
    fun provideService(repository: Repository): Service = Service.Impl(repository)
}

interface Context {
    fun inject(vm: ViewModel)
}

@Singleton
@Component(modules = [RepositoryModule::class, ServiceModule::class])
interface ContextComponent : Context
```

我知道，我知道，你不是这么做的，但这只是我手头上的一个用例。

你如何使用这种结构来测试事物？ 嗯，这很简单。

- 对于模型组件，你可以模拟或存根你的依赖关系并将它们传递给构造函数。
- 对于表示组件，可以构建自己的上下文并将其传递给它。

## 决定，决定

以上所有都让我思考。

> 你需要复杂的工具来解决简单的问题吗？

正如你所看到的，设置非常简单。 是的，	潜在的 Dagger 可能带来一些好处，但是每天增加团队中每个开发人员的构建时间是否值得呢？ 并考虑到这一设置多年来一直没有任何改变的事实？

> 你是否需要继续使用为不同条件而设计的工具？

让我们面对现实吧ーー注解处理是一个不错的想法，但意味着特殊的环境。 使用 Java 来手动声明所有内容都太冗长了，所以在这里，我们有一个代码生成器可以为我们做到这一点。 如果你在整个代码库中使用 Kotlin，这是否是一个很好的选择呢？ 我不知道，这是你的代码和你的调用。 我做了我的。

## 回到原点

> 拥有一个库并不酷。 你知道什么最酷吗？ 没有库。

让我们疯狂起来，利用 Kotlin 来实现我们自己的[控制反转](https://en.wikipedia.org/wiki/Inversion_of_control)。 不是 [Koin](https://github.com/Ekito/koin) ，不是 [Kodein](https://github.com/SalomonBrys/Kodein) ，不是 [Kapsule](https://github.com/traversals/kapsule) ，只是一些模式和语言功能。

> 我强烈建议阅读马 Martin Fowler 关于控制反转容器的[文章](https://martinfowler.com/articles/injection.html)。 它几乎包含了你需要知道的关于 IoC 的所有内容，所以我将只谈论实践。

### 模块

模块是什么？ 它是依赖关系的注册表。 模块有什么属性？ 依赖于其他模块。 听起来很简单。

```kotlin
interface RepositoryModule {
    val repository: Repository

    class Impl : RepositoryModule {
        override val repository by lazy { Repository.Impl() }
    }
}

interface ServiceModule {
    val service: Service

    class Impl(repositoryModule: RepositoryModule) {
        override val service by lazy { Service.Impl(repositoryModule.repository) }
    }
}
```

注意那个 lazy 委托。 它使我们的[属性变成懒惰的单例](https://kotlinlang.org/docs/reference/delegated-properties.html#lazy)，就像 Dagger 会为你做的一样！ 换句话说，创建一个模块不会立即创建所有的依赖关系，而只会在第一次访问时进行。

### 上下文

谈论上下文... 它只是一个模块的组成，对吗？

```kotlin
interface Context :
    RepositoryModule,
    ServiceModule {

    class Impl(
        repositoryModule: RepositoryModule,
        serviceModule: ServiceModule
    ) : Context,
        RepositoryModule by repositoryModule,
        ServiceModule by serviceModule
}
```

我们在这里使用 [Kotlin 委托](https://kotlinlang.org/docs/reference/delegation.html)。 上下文将被翻译成类似这样的东西给最终用户。

```kotlin
interface Context {
    val repository: Repository
    val service: Service
}
```

你必须手工创建它，首先创建所有模块。 Dagger 可以帮你，但也没什么大不了的。

```kotlin
fun createContext(): Context {
    val repositoryModule = RepositoryModule.Impl()
    val serviceModule = ServiceModule.Impl(repositoryModule)

    return Context.Impl(repositoryModule, serviceModule)
}
```

是的，这有点冗长，但是你完全可以控制，因为这是你的代码。

### Tricks

你可以定义与模块同时创建的非懒惰依赖项。

```kotlin
interface RepositoryModule {
    val repository: Repository

    class Impl : Module {
        override val repository = Repository.Impl()
    }
}
```

你可以定义非单例依赖项。

```kotlin
interface RepositoryModule {
    val repository: Repository

    class Impl : Module {
        override val repository: Repository
            get() = Repository.Impl()
    }
}
```

你可以使用与上下文相同的委托方法来定义范围。

```kotlin
interface UserContext :
    Context,
    UserModule {

    class Impl(
        context: Context,
        userModule: UserModule
    ) : UserContext,
        Context by context,
        UserModule by userModule
}

interface Context {
    fun plus(userModule: UserModule): UserContext

    class Impl : Context {
        override fun plus(userModule: UserModule) = UserContext.Impl(this, userModule)
    }
}
```

你可以用相同的接口定义多个依赖关系。

```kotlin
interface ServiceModule {
    val yinService: Service
    val yangService: Service
}
```

你可以从 late init var 移动到 private val。

```kotlin
class ViewModel(context: Context) {
    private val service = context.service
}
```

### 结果

似乎可以在没有 Dagger 的情况下进行控制反转，谁知道？

#### 优点

- 控制反转是基于你的代码和模式而不是框架
- 因为这是你的代码，你可以用它做任何你想做的事情，而且理解它实际上是如何工作的非常简单
- 使用有意义的消息编译时验证
- 没有注解处理，即更快和更可靠的构建

#### 缺点

- 这是一个粗略的服务定位器。
  - 好吧，大部分都是真的，但是我能接受吗？ 当然。 尤其是当考虑到目标是达到控制反转而不是依赖注入的时候。

- 这是冗长的。
  - 这是，谢谢！ 你必须真正考虑你的上下文是如何制作的，而我真的喜欢它。

- 没有匕首 - 没有酷点。
- 我在 Plan 9机器上输入这个文字，所以......

不过，笑话除外ーー它在现实生活中是有效的。

- kapt 移除最终带来了团队对 Kotlin 编译器的信心。我很久没有听到关于性能或者奇怪编译错误的抱怨，这是一个好的迹象。 我已经观察到建设时间减少了25% ，并且正确的增量构建。
- 同时，我注意到人们开始关注与 IoC 相关的代码，就像他们关心主代码库一样，因为它不再是一堆没有人理解的依赖。 这也是一件好事。

# Fin

开发人员倾向于寻找一种最终解决方案。 你会解析 JSON 吗？ 你绝对必须使用这个星球上可用的性能最高的解析器，否则... 你玩 Doom 游戏了吗？ 那么，它就是这样。 哦，你只解析它一次，它是一个有两个字段的对象？ 你需要最高性能的解析器，请记住！

不要让工具成为 [MacGuffin](https://en.wikipedia.org/wiki/MacGuffin) --根据你的需求选择它，并且不要将你的需求适应于工具。我们开发事情来解决问题，而不是创造它们。









