# Dagger 2 : Component Relationships & Custom Scopes

> 原文 (Medium)：[Dagger 2 : Component Relationships & Custom Scopes](https://proandroiddev.com/dagger-2-component-relationships-custom-scopes-8d7e05e70a37?source=user_profile---------1----------------)
>
> 作者：[Garima Jain](https://proandroiddev.com/@ragdroid?source=post_header_lockup)

[TOC]

在之前的一篇文章中，我们看到了 @singleton 和 @reusable 范围。 在这篇文章中，让我们尝试通过在生成的类的海洋中多潜水一点来面对野兽，并尝试学习所涉及的模式。 做好准备释放野兽，甚至在这个过程中失去理智 

> 很久很久以前，有一个叫做亚洲机器人的开发商，他很想看到依赖的面孔。 她从最简单的实现开始，最后干预了生成的代码，被"The Dahaka"追逐 。
>
> 从第一天起，她遇到了一个名为 “子组件”（Subcomponent）的术语，她一直拖延着去了解组件的依赖关系，直到有一天 Google 释放了沙子，并引入了 Dagger-Android 。 她仔细查阅了它的文件，里面全是子组件。 经过几次谷歌搜索和 Stack Overflow 的阅读之后，她意识到，她试图获得的理解不会没有代价。 她知道她需要开始她与组件依赖的冒险，她试图解码所有相关的 Dagger-Android 神秘的东西。 这就是一切开始的地方。“

首先，让我们从一些可以描述而又不太疯狂的事情开始: 

当你第一次使用 Dagger 时，通常首先使用一个 AppComponent 和一个 @singleton 范围的 AppModule。 在这种情况下，Dagger 生成了一个 DaggerAppComponent 组件，它与 AppModule 有一个 HAS-A 关系。 有一个 appModule 实例。 

但是随着应用的规模的增长，你很快就会意识到你的 AppModule 开始变成一个有各种依赖性的上帝模块。 这时你就开始寻找解决这个问题的不同方法。 

我主要看到三种不同类型的组件依赖关系：

## 1. 单组件，多模块: 

当你的主要上帝组件变得非常大，并在 AppModule 本身内部声明各种依赖关系时，你就可以把它分解成 AppModule，ApiModule 等等。 请记住，所有这些依赖依赖关系仍然具有 Singleton 范围(如果指定的话)。 下面提到的方法大部分都是相同的，所生成的 DaggerAppComponent 组件将与你的两个模块都有一个 HAS-A 关系。 

DaggerAppComponent HAS-A apiModule.
有两种方法可以实现这一点：

1. AppComponent 声明对 AppModule 和 ApiModule 的依赖。

```java
@Component(modules = {AppModule.class, ApiModule.class})
public interface AppComponent 
```

2. AppComponent 声明只依赖 AppModule ，而 AppModule 则包含 ApiModule。

```java
@Component(modules = AppModule.class)
public interface AppComponent

@Module(includes = ApiModule.class)
public class AppModule
```

但是，这种方法存在一个问题。在这里仍然没有办法创建自定义范围，让一些依赖关系像 @ActivityScope，@FragmentScope，@UserScope 等只能在一段时间存在。

## 2. 子组件依赖 

当你意识到你的依赖关系不一定要像你的应用程序一样长时间存活，你可以创建一个子组件。 子组件可以访问在超级组件中提供的任何依赖，而无需明确声明它。 例如: @ActivityScope 或 @FragmentScope，此类依赖关系的范围与活动或片段的范围有关，比应用程序的范围窄。 我们将在下一节中详细研究子组件。 

## 3. 依赖组件 

创建依赖组件是让依赖项更短的生存的另一种方式。 例如: 一些依赖项，比如用户对象本身，只有当用户登录到应用程序并且直到他 / 她登陆之后才会存在。 在这种情况下，我们可以创建一个名为 UserComponent 的组件，该组件依赖于 AppComponent 和自定义作用域 @userscope。 在这种情况下，依赖组件只能通过接口从其他组件访问显式暴露的依赖项。 依赖组件作为一个独立对象存在。 

## 子组件vs依赖组件

UserScope :

> 她是亚洲的机器人，曾多次听过 UserScope 这个词，但直到她偶然发现了 Miroslaw Stanek 的一篇 [关于 UserScope 的很棒文章](http://frogermcs.github.io/building-userscope-with-dagger2/)后。 这篇文章为她指明了方向，使她能够继续进行 Dagger 2 的冒险。她开始一个接一个地潜入生成的类，并在过程中被 “Dahaka” 追逐。

前面关于范围的文章中提到的范围是任何对象/依赖关系的生命周期。 所以，UserScope 是与任何用户相关的所有依赖的范围。 我们来构建一个示例应用程序，使用 UserScope 的概念演示各种组件的依赖关系。

我们将使用 Pokemon 应用程序的示例来了解子组件和依赖组件之间的区别。你可以在这里找到所有的[演示代码](https://github.com/ragdroid/Dahaka/tree/dependent-component)。 Kotlin 版本在[这里](https://github.com/ragdroid/Dahaka/tree/dependent-component-kotlin)。

> 我们打算用可爱的宠物小精灵击败 Dahaka！ ：P

让我们看看下面的图表来理解我们正在尝试构建的东西：
![Scoping Architecute for the Pokemon app](https://ws4.sinaimg.cn/large/006tKfTcgy1frot76h72uj30m80gogna.jpg)

让我们开始阅读我们的图表/源代码。

1. AppComponent（@Singleton）: 单例范围组件，应用程序的主要组件。通常创建并持久状态在 Application 类中。
2. LoginComponent @ActivityScope : AppComponent 的子组件。这包含特定于 LoginActivity 的任何依赖关系。
3. UserComponent @UserScope：依赖 AppComponent 。包含与用户（Pokemon）相关的任何依赖关系。这个组件还包含两个 @ActvityScope 子组件：HomeComponent 和 ItemsComponent。
4. HomeComponent @ActivityScope：UserComponent 的子组件。它也由三个 Fragments 组成：ProfileFragment，StatsFragment 和 MovesFragment。
5. ItemsComponent @ActivityScope：UserComponent 的子组件。

下面是一个运行的例子，说明我们的 UserScope 是什么样的: 
![@UserScope|center](https://ws4.sinaimg.cn/large/006tKfTcgy1frot8ioqryg308w0fdqv5.gif)

在这篇文章中，我们将重点讨论两个主要的案例。 ：

1. 依赖组件：依赖于 AppComponent 的 UserComponent。
2. 子组件：LoginComponent 是 AppComponent 的一个子组件

![@Subcomponent vs Dependent Component](https://ws3.sinaimg.cn/large/006tKfTcgy1frotbivq96j30m80h5wid.jpg)

我们的 AppComponent，它公开了一个名为 schedulerProvider 的依赖项。 Dagger 将生成一个名为 DaggerAppComponent 的 AppComponent 的实现。UserComponent 依赖于 AppComponent 并使用 AppComponent 接口访问 schedulerProvider。 LoginComponentImpl 是我们 AppComponent 的一个子组件。 我们来看看上面结构的代码。

AppComponent

```java
@Singleton
@Component(modules = arrayOf(AppModule::class))
interface AppComponent {
    //exposes the builder for LoginComponent    
    fun loginBuilder(): LoginComponent.Builder

    //exposes schedulerProvider dependency
    fun schedulerProvider(): BaseSchedulerProvider
}
```

我们有一个 @Singleton AppComponent，它公开了 schedulerProvider ( )。它也为 LoginComponent 公开了一个 Builder（我们稍后会用到它）。

DahakaApplication

```java
class DahakaApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        appComponent = DaggerAppComponent
				.builder()
				.application(this)
				.build()
    }
}
```

我们的 DahakaApplication 负责创建 AppComponent 。

### 依赖组件

UserComponent

```java
@Component(dependencies = arrayOf(AppComponent::class), 
			modules = arrayOf(UserModule::class))
@UserScope
interface UserComponent {

    @Component.Builder
    interface Builder {
       
	@BindsInstance
        fun pokeMon(pokemon: Pokemon): Builder

        …
    }
}
```

1. dependencies  = arrayOf (AppComponent :: class )：这是我们如何告诉 dagger UserComponent 依赖于 AppComponent。
2. 我们在 UserComponent 中声明了 @Component.Builder。这告诉 dagger 根据我们的规格生成 Builder 实现。更多关于 [Component.Builder](https://proandroiddev.com/dagger-2-component-builder-1f2b91237856) 的信息
3. @BindsInstance 绑定我们的用户 : Pokemon 到对象图。更多关于 [@BindsInstance](https://proandroiddev.com/dagger-2-component-builder-1f2b91237856) 的信息

UserManager

```java
@Singleton
class UserManager @Inject
constructor(private val service: PokemonService) {

    var userComponent: UserComponent? = null
        private set

    public fun createUserSession(pokemon: Pokemon) {
        userComponent = DaggerUserComponent.builder()
                .appComponent(DahakaApplication.app.appComponent)
                .pokeMon(pokemon)
                .build()
    }

	…
}
```

UserManager 是一个单例依赖项。 我们登录后，从我们的用户服务器上得到一个 Pokemon 实例。我们通过构建 UserComponent 来开始会话。请注意，我们需要向 UserComponent 提供 appComponent 的一个实例，以访问诸如 schedulerProvider 之类的依赖项。Pokemon 将被绑定到对象图上，以便它可以在 @UserScope 中使用。

让我们来看看 “Dahaka 的面孔”，通过生成的 DaggerUserComponent 进行浏览。

DaggerUserComponent（Generated）

```java
public final class DaggerUserComponent implements UserComponent {

    private Provider<BaseSchedulerProvider> schedulerProvider;

    private void initialize(final Builder builder) {
	
	    this.schedulerProvider =
    	    		new AppComponent_schedulerProvider(builder.appComponent);
	    …
    }

    private static class AppComponent_schedulerProvider 
					implements Provider<BaseSchedulerProvider> {
        private final AppComponent appComponent;
	
            @Override
	    public BaseSchedulerProvider get() {
 		    return appComponent.schedulerProvider();
	    }
	    …
    }
}
```

我们在这里有几点要注意: 

1. 每当我们声明一个依赖的 SomeDependency 时，Dagger 就在组件实现中存储一个 Provider \<SomeDependency>。因此，在上面的情况下，Dagger 存储一个 Provider \<BaseSchedulerProvider> schedulerProvider。
2. 在 DaggerUserComponent 的 initialize ( ) 函数内部，Dagger 使用我们先前在构建器中传递的 appComponent 来初始化 schedulerProvider，同时创建 UserComponent。
3. 每当任何依赖项需要 schedulerProvider 时，dagger 将调用 schedulerProvider.get ( )，最终将使用 AppComponent 实例获取依赖项。 我们不能从 UserComponent 中访问 schedulerProvider 依赖。 我们使用 AppComponent 接口来访问 @Singleton 范围中的任何依赖项。 这就是为什么我们不得不在 AppComponent 中暴露我们的 schedulerProvider ( ) 依赖。

### 子组件

Login Subcomponent

```java
@ActivityScope
@Subcomponent(modules = arrayOf(LoginModule::class))
interface LoginComponent {

    @Subcomponent.Builder
    interface Builder {

        @BindsInstance
        fun loginActivity(loginActivity: LoginActivity): Builder
    }

}
```

1. LoginComponent 具有 @ActivityScope，因为它绑定到 LoginActivity。
2. 我们告诉 dagger，这是一个注解了 @Subcomponent 的子组件。
3. 就像 @Component.Builder 一样，我们也可以告诉 Dagger 这个子组件的 Builder 应该是什么样的。
4. 我们告诉 Dagger 使用 @BindsInstance 注解将 LoginActivity 的实例绑定到依赖关系图上。

LoginActivity

```java
class LoginActivity {


    private lateinit var loginComponent: LoginComponent

    override fun initDagger(appComponent: AppComponent) {
        loginComponent = appComponent
                .loginBuilder()
                .loginActivity(this)
                .build()
   }

	…
}
```

我们使用之前在 LoginComponent 中声明的 Builder 构建 LoginActivity 中的 LoginComponent。我们可以从AppComponent 获取暴露的 Builder appComponent.loginbuilder ( )。

让我们来看看 “Dahaka 的面孔”，通过生成的 DaggerAppComponent 进行浏览。

DaggerAppComponent (Generated)

```java
public final class DaggerAppComponent implements AppComponent {

  	this.loginBuilderProvider =
      		new Factory<LoginComponent.Builder>() {
        		@Override
        		public LoginComponent.Builder get() {
         	 		return new LoginComponentBuilder();
        		}
    	  	};
	}


	private final class LoginComponentBuilder implements LoginComponent.Builder {
		…
	}

	private final class LoginComponentImpl implements LoginComponent {

		private LoginComponentImpl(LoginComponentBuilder builder) {
  			assert builder != null;
			initialize(builder);
		}
  		…
	}
}
```

这里有几点需要注意：

1. Dagger 生成一个 LoginComponentBuilder，它实现了我们在 LoginComponent 中声明的 LoginComponent.Builder。
2. Dagger 实例化一个 Provider \<LoginComponent.Builder>。
3. 最后，最重要的是要注意：Dagger 生成一个 LoginComponentImpl，它是 DaggerAppComponent 的一个内部类。 任何由 LoginComponentImpl 所需的 AppComponent 依赖都可以直接从 AppComponent 访问，而不需要显式公开这些依赖，因为内部类可以访问它的外部类的成员。

## 依赖组件与子组件

现在让我们通过下面的图表来连接这些点: 
![@Dependent Component vs Sub component](https://ws1.sinaimg.cn/large/006tKfTcgy1frotg7qxt6j30m80h5wid.jpg)

**DaggerUserComponent**

- DaggerUserComponent 依赖于 AppComponent。
- DaggerUserComponent 不知道 AppComponent 的实现，只知道通过 AppComponent 接口公开的依赖关系。
- DaggerUserComponent 使用 AppComponent 接口访问 schedulerProvider。

## 结束

使用依赖组件，当你希望保持两个组件的独立性，并保持两者之间的耦合性更小。 

当两个组件相互耦合时使用子组件，就像应用程序和活动一样。 此外，dagger-android 系统能很好地处理子组件，并且可以减少活动、片段、服务等安卓框架类的模板。 更多关于 dagger-android  在即将到来的帖子。 

感谢 [Lucia Payo](https://medium.com/@LuciaP_86) 的评论:)

