

## 详解 Dagger2

> - 原文链接 : [Tasting Dagger 2 on Android](http://fernandocejas.com/2015/04/11/tasting-dagger-2-on-android/)
> - 原文作者 : [Fernando Cejas](http://fernandocejas.com/)
> - [译文出自 : 开发技术前线 www.devtf.cn](http://www.devtf.cn/)
> - 译者 : [xianjiajun](https://github.com/xianjiajun)
> - 校对者: [chaossss](https://github.com/chaossss)
> - 状态 : 完成

##为什么使用依赖注入

首先我们需要知道，人们在很长的一段时间里都是利用控制反转原则规定：应用程序的流程取决于在程序运行时对象图的建立。通过抽象定义的对象交互可以实现这样的动态流程。而使用依赖注入技术或者服务定位器便可以完成运行时绑定。

使用依赖注入可以带来以下好处：

- 依赖的注入和配置独立于组件之外。
- 因为对象是在一个独立、不耦合的地方初始化，所以当注入抽象方法的时候，我们只需要修改对象的实现方法，而不用大改代码库。
- 依赖可以注入到一个组件中：我们可以注入这些依赖的模拟实现，这样使得测试更加简单。

可以看到，能够管理创建实例的范围是一件非常棒的事情。按我的观点，你 app 中的所有对象或者协作者都不应该知道有关实例创建和生命周期的任何事情，这些都应该由我们的依赖注入框架管理的。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbuyqgj05j30ej068t91.jpg)

#什么是 JSR-330？

 为了最大程度的提高代码的复用性、测试性和维护性，java 的依赖注入为注入类中的使用定义了一整套注解（和接口）标准。Dagger1 和 Dagger2（还有 Guice）都是基于这套标准，给程序带来了稳定性和标准的依赖注入方法。

#Dagger1 

这个版本不是这篇文章的重点，所以我只是简略地说一下。不管怎样，Dagger1 还是做了很多的贡献，可以说是如今 Android 上最流行的依赖注入框架。它是由 Square 公司受到 Guice 启发创建的。

基本特点：

- 多个注入点：依赖，通过 injected
- 多种绑定方法：依赖，通过 provided
- 多个 modules：实现某种功能的绑定集合
- 多个对象图： 实现一个范围的 modules 集合

Dagger1 是在编译的时候实行绑定，不过也用到了反射机制。但这个反射不是用来实例化对象的，而是用于图的构成。Dagger 会在运行的时候去检测是否一切都正常工作，所以使用的时候会付出一些代价：偶尔会无效和调试困难。

#Dagger2

Dagger2 是 Dagger1 的分支，由谷歌公司接手开发，目前的版本是2.0。Dagger2是受到 [AutoValue项目](https://github.com/google/auto)的启发。 刚开始，Dagger2 解决问题的基本思想是：利用生成和写的代码混合达到看似所有的产生和提供依赖的代码都是手写的样子。

如果我们将 Dagger2 和1比较，他们两个在很多方面都非常相似，但也有很重要的区别，如下：

- 再也没有使用反射：图的验证、配置和预先设置都在编译的时候执行。
- 容易调试和可跟踪：完全具体地调用提供和创建的堆栈
- 更好的性能：谷歌声称他们提高了13%的处理性能
- 代码混淆：使用派遣方法，就如同自己写的代码一样

当然所有这些很棒的特点都需要付出一个代价，那就是缺乏灵活性，例如：Dagger2 没用反射所以没有动态机制。

#深入研究

想要了解Dagger2，就必须要知道依赖注入的基础和这其中的每一个概念：

- @Inject: 通常在需要依赖的地方使用这个注解。换句话说，你用它告诉Dagger这个类或者字段需要依赖注入。这样，Dagger就会构造一个这个类的实例并满足他们的依赖。
- @Module: Modules类里面的方法专门提供依赖，所以我们定义一个类，用@Module注解，这样Dagger在构造类的实例的时候，就知道从哪里去找到需要的依赖。modules的一个重要特征是它们设计为分区并组合在一起（比如说，在我们的app中可以有多个组成在一起的modules）。
- @Provide: 在modules中，我们定义的方法是用这个注解，以此来告诉Dagger我们想要构造对象并提供这些依赖。
- @Component: Components从根本上来说就是一个注入器，也可以说是@Inject和@Module的桥梁，它的主要作用就是连接这两个部分。Components可以提供所有定义了的类型的实例，比如：我们必须用@Component注解一个接口然后列出所有的@Modules组成该组件，如果缺失了任何一块都会在编译的时候报错。所有的组件都可以通过它的modules知道依赖的范围。
- @Scope: Scopes可是非常的有用，Dagger2可以通过自定义注解限定注解作用域。后面会演示一个例子，这是一个非常强大的特点，因为就如前面说的一样，没必要让每个对象都去了解如何管理他们的实例。在scope的例子中，我们用自定义的@PerActivity注解一个类，所以这个对象存活时间就和activity的一样。简单来说就是我们可以定义所有范围的粒度(@PerFragment, @PerUser, 等等)。
- Qualifier: 当类的类型不足以鉴别一个依赖的时候，我们就可以使用这个注解标示。例如：在Android中，我们会需要不同类型的context，所以我们就可以定义qualifier注解“@ForApplication”和“@ForActivity”，这样当注入一个context的时候，我们就可以告诉Dagger我们想要哪种类型的context。

#不废话上代码 

前面已经讲了很多理论了，所以接下来让我们看看如何使用 Dagger2。首先还是要在我们的 build.gradle 文件中如下配置： 

```groovy
apply plugin: 'com.neenbedankt.android-apt'
 
buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.4'
  }
}
 
android {
  ...
}
 
dependencies {
  apt 'com.google.dagger:dagger-compiler:2.0'
  compile 'com.google.dagger:dagger:2.0'
  
  ...
}
```

如上所示，我们添加了编译和运行库，还有必不可少的 apt 插件，没有这插件，dagger 可能不会正常工作，特别是在 Android studio 中。

#例子

几个月前，我写了一篇关于如何在 Android 上实现 bob 叔叔的简洁架构的文章，强烈建议大家去看一下，看完之后，你将会对我们现在做的事情有更好的理解。言归正传，在我以前的方案中，构造和提供大多数对象的依赖的时候，会遇到问题，具体如下（见评注）：

```Java
 @Override void initializePresenter() {
    // All this dependency initialization could have been avoided by using a
    // dependency injection framework. But in this case this is used this way for
    // LEARNING EXAMPLE PURPOSE.
    ThreadExecutor threadExecutor = JobExecutor.getInstance();
    PostExecutionThread postExecutionThread = UIThread.getInstance();

    JsonSerializer userCacheSerializer = new JsonSerializer();
    UserCache userCache = UserCacheImpl.getInstance(getActivity(), userCacheSerializer,
        FileManager.getInstance(), threadExecutor);
    UserDataStoreFactory userDataStoreFactory =
        new UserDataStoreFactory(this.getContext(), userCache);
    UserEntityDataMapper userEntityDataMapper = new UserEntityDataMapper();
    UserRepository userRepository = UserDataRepository.getInstance(userDataStoreFactory,
        userEntityDataMapper);

    GetUserDetailsUseCase getUserDetailsUseCase = new GetUserDetailsUseCaseImpl(userRepository,
        threadExecutor, postExecutionThread);
    UserModelDataMapper userModelDataMapper = new UserModelDataMapper();

    this.userDetailsPresenter =
        new UserDetailsPresenter(this, getUserDetailsUseCase, userModelDataMapper);
  }
```

可以看出，解决这个问题的办法是使用依赖注入框架。我们要避免像上面这样引用代码：这个类不能涉及对象的创建和依赖的提供。 那我们该怎么做呢，当然是使用 Dagger2，我们先看看结构图：

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbw1stkn2j30e90bpmxv.jpg)

接下来我们会分解这张图，并解释各个部分还有代码。

Application Component: 生命周期跟 Application 一样的组件。可注入到 AndroidApplication 和 BaseActivity 中类中。

```Java
@Singleton // Constraints this component to one-per-application or unscoped bindings.
@Component(modules = ApplicationModule.class)
public interface ApplicationComponent {
  void inject(BaseActivity baseActivity);

  //Exposed to sub-graphs.
  Context context();
  ThreadExecutor threadExecutor();
  PostExecutionThread postExecutionThread();
  UserRepository userRepository();
}
```

我为这个组件使用了 @Singleton 注解，使其保证唯一性。也许你会问为什么我要将 context 和其他成员暴露出去。这正是 Dagger 中 components 工作的重要性质：如果你不想把 modules 的类型暴露出来，那么你就只能显示地使用它们。在这个例子中，我把这些元素暴露给子图，如果你把他们删掉，编译的时候就会报错。

Application Module: module 的作用是提供在应用的生命周期中存活的对象。这也是为什么 @Provide 注解的方法要用 @Singleton 限定。

```Java
@Module
public class ApplicationModule {
  private final AndroidApplication application;

  public ApplicationModule(AndroidApplication application) {
    this.application = application;
  }

  @Provides @Singleton Context provideApplicationContext() {
    return this.application;
  }

  @Provides @Singleton Navigator provideNavigator() {
    return new Navigator();
  }

  @Provides @Singleton ThreadExecutor provideThreadExecutor(JobExecutor jobExecutor) {
    return jobExecutor;
  }

  @Provides @Singleton PostExecutionThread providePostExecutionThread(UIThread uiThread) {
    return uiThread;
  }

  @Provides @Singleton UserCache provideUserCache(UserCacheImpl userCache) {
    return userCache;
  }

  @Provides @Singleton UserRepository provideUserRepository(UserDataRepository userDataRepository) {
    return userDataRepository;
  }
}
```

Activity Component: 生命周期跟 Activity 一样的组件。

```Java
@PerActivity
@Component(dependencies = ApplicationComponent.class, modules = ActivityModule.class)
public interface ActivityComponent {
  //Exposed to sub-graphs.
  Activity activity();
}
```

@PerActivity是一个自定义的范围注解，作用是允许对象被记录在正确的组件中，当然这些对象的生命周期应该遵循activity的生命周期。这是一个很好的练习，我建议你们都做一下，有以下好处：

- 注入对象到构造方法需要的activity。
- 在一个per-activity基础上的单例使用。
- 只能在activity中使用使得全局的对象图保持清晰。

看下代码：

```Java
@Scope
@Retention(RUNTIME)
public @interface PerActivity {}
```

Activity Module: 在对象图中，这个 module 把 activity 暴露给相关联的类。比如在 fragment 中使用 activity 的 context。

```Java
@Module
public class ActivityModule {
  private final Activity activity;

  public ActivityModule(Activity activity) {
    this.activity = activity;
  }

  @Provides @PerActivity Activity activity() {
    return this.activity;
  }
}
```

User Component: 继承于 ActivityComponent 的组件，并用 @PerActivity 注解。我通常会在注入用户相关的 fragment 中使用。因为 ActivityModule 把 activity 暴露给图了，所以在任何需要一个 activity 的 context 的时候，Dagger 都可以提供注入，没必要再在子 modules 中定义了。

```Java
@PerActivity
@Component(dependencies = ApplicationComponent.class, modules = {ActivityModule.class, UserModule.class})
public interface UserComponent extends ActivityComponent {
  void inject(UserListFragment userListFragment);
  void inject(UserDetailsFragment userDetailsFragment);
}
```

User Module: 提供跟用户相关的实例。基于我们的例子，它可以提供用户用例。

```Java
@Module
public class UserModule {
  @Provides @PerActivity GetUserListUseCase provideGetUserListUseCase(GetUserListUseCaseImpl getUserListUseCase) {
    return getUserListUseCase;
  }

  @Provides @PerActivity GetUserDetailsUseCase provideGetUserDetailsUseCase(GetUserDetailsUseCaseImpl getUserDetailsUseCase) {
    return getUserDetailsUseCase;
  }
}
```

#整合到一起

现在我们已经实现了依赖注入图，但是我该如何注入？我们需要知道，Dagger 给了我们一堆选择用来注入依赖：

1. 构造方法注入：在类的构造方法前面注释 @Inject
2. 成员变量注入：在类的成员变量（非私有）前面注释 @Inject
3. 函数方法注入：在函数前面注释 @Inject

这个顺序是 Dagger 建议使用的，因为在运行的过程中，总会有一些奇怪的问题甚至是空指针，这也意味着你的依赖在对象创建的时候可能还没有初始化完成。这在 Android 的 activity 或者 fragment 中使用成员变量注入会经常遇到，因为我们没有在它们的构造方法中使用。

回到我们的例子中，看一下我们是如何在 BaseActivity 中注入一个成员变量。在这个例子中，我们注入了一个叫 Navigator 的类，它是我们应用中负责管理导航的类。

```Java
public abstract class BaseActivity extends Activity {

  @Inject Navigator navigator;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    this.getApplicationComponent().inject(this);
  }

  protected ApplicationComponent getApplicationComponent() {
    return ((AndroidApplication)getApplication()).getApplicationComponent();
  }

  protected ActivityModule getActivityModule() {
    return new ActivityModule(this);
  }
}
```

Navigator 类是成员变量注入的，由 ApplicationModule 里面 @Provide 注解显示提供的。最终我们初始化 component 然后调用 inject ( ) 方法注入成员变量。我们通过在 Activity的onCreate ( ) 方法中调用 getApplicationComponent( )，完成这些操作。getApplicationComponent( ) 方法放在这儿是为了复用性，它的主要作用是为了获取实例化的 ApplicationComponent 对象。

在 Fragment 的 presenter 中我们也做了同样的事情，这儿的获取方法有一点不一样，因为问我们使用的是 per-activity 范围限定的 component。所以我们注入到 UserDetailsFragment 中的 UserComponent 其实是驻留在 UserDetailsActivity 中的。

```Java
private UserComponent userComponent;
```

我们必须在 activity 的 onCreate ( ) 方法中用下面的方式初始化。

```Java
private void initializeInjector() {
  this.userComponent = DaggerUserComponent.builder()
      .applicationComponent(getApplicationComponent())
      .activityModule(getActivityModule())
      .build();
}
```

Dagger会处理我们的注解，为 components 生成实现并重命名加上 “Dagger” 前缀。因为这个是一个组合的 component，所以在构建的时候，我们必须把所有的依赖的传进去（components 和 modules）。现在我们的 component 已经准备好了，接着为了可以满足 fragment 的依赖需求，我们写一个获取方法：

```Java
@Override public UserComponent getComponent() {
  return userComponent;
}
```

我们现在可以利用 get 方法获取创建的 component，然后调用 inject ( ) 方法将 Fragment 作为参数传进去，这样就完成了绑定 UserDetailsFragment 依赖。

```Java
@Override public void onActivityCreated(Bundle savedInstanceState) {
  super.onActivityCreated(savedInstanceState);
  this.getComponent.inject(this);
}
```

[想要查看完整的例子，可以去我的github](https://github.com/android10/Android-CleanArchitecture).这里面有一些地方重构了的，我可以告诉你一个重要的思想（来自官方的例子）是：

```Java
public interface HasComponent<C> {
  C getComponent();
}
```

因此，客户端（例如 fragment）可以获取并且使用 component（来自 activity）：

```Java
@SuppressWarnings("unchecked")
protected <C> C getComponent(Class<C> componentType) {
  return componentType.cast(((HasComponent<C>)getActivity()).getComponent());
}
```

这儿使用了强制转换，不论这个客户端不能获取到能用的component，但是至少很快就会失败。如果你有任何想法能够更好地解决这个问题，请告诉我。

 #Dagger2生成的代码 

在了解 Dagger 的主要特征之后，我们再来看看内部构造。为了举例说明，我们还是用 Navigator 类，看看它是如何创建和注入的。首先我们看一下我们的 DaggerApplicationComponent。

```Java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class DaggerApplicationComponent implements ApplicationComponent {
  private Provider<Navigator> provideNavigatorProvider;
  private MembersInjector<BaseActivity> baseActivityMembersInjector;

  private DaggerApplicationComponent(Builder builder) {  
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {  
    return new Builder();
  }

  private void initialize(final Builder builder) {  
    this.provideNavigatorProvider = ScopedProvider.create(ApplicationModule_ProvideNavigatorFactory.create(builder.applicationModule));
    this.baseActivityMembersInjector = BaseActivity_MembersInjector.create((MembersInjector) MembersInjectors.noOp(), provideNavigatorProvider);
  }

  @Override
  public void inject(BaseActivity baseActivity) {  
    baseActivityMembersInjector.injectMembers(baseActivity);
  }

  public static final class Builder {
    private ApplicationModule applicationModule;
  
    private Builder() {  
    }
  
    public ApplicationComponent build() {  
      if (applicationModule == null) {
        throw new IllegalStateException("applicationModule must be set");
      }
      return new DaggerApplicationComponent(this);
    }
  
    public Builder applicationModule(ApplicationModule applicationModule) {  
      if (applicationModule == null) {
        throw new NullPointerException("applicationModule");
      }
      this.applicationModule = applicationModule;
      return this;
    }
  }
}
```

有两个重点需要注意。第一个：由于我们要将依赖注入到 activity 中，所以会得到一个注入这个比成员的注入器（由 Dagger 生成的 BaseActivity_MembersInjector）： 

```Java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class BaseActivity_MembersInjector implements MembersInjector<BaseActivity> {
  private final MembersInjector<Activity> supertypeInjector;
  private final Provider<Navigator> navigatorProvider;

  public BaseActivity_MembersInjector(MembersInjector<Activity> supertypeInjector, Provider<Navigator> navigatorProvider) {  
    assert supertypeInjector != null;
    this.supertypeInjector = supertypeInjector;
    assert navigatorProvider != null;
    this.navigatorProvider = navigatorProvider;
  }

  @Override
  public void injectMembers(BaseActivity instance) {  
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    supertypeInjector.injectMembers(instance);
    instance.navigator = navigatorProvider.get();
  }

  public static MembersInjector<BaseActivity> create(MembersInjector<Activity> supertypeInjector, Provider<Navigator> navigatorProvider) {  
      return new BaseActivity_MembersInjector(supertypeInjector, navigatorProvider);
  }
}
```

这个注入器一般都会为所有 activity 的注入成员提供依赖，只要我们一调用 inject ( ) 方法，就可以获取需要的字段和依赖。

第二个重点：关于我们的 DaggerApplicationComponent 类，我们有一个 Provider，它不仅仅是一个提供实例的接口，它还是被 ScopedProvider 构造出来的，可以记录创建实例的范围。

Dagger 还会为我们的 Navigator 类生成一个名叫 ApplicationModule_ProvideNavigatorFactory 的工厂，这个工厂可以传递上面提到的范围参数然后得到这个范围内的类的实例。

```Java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class ApplicationModule_ProvideNavigatorFactory implements Factory<Navigator> {
  private final ApplicationModule module;

  public ApplicationModule_ProvideNavigatorFactory(ApplicationModule module) {  
    assert module != null;
    this.module = module;
  }

  @Override
  public Navigator get() {  
    Navigator provided = module.provideNavigator();
    if (provided == null) {
      throw new NullPointerException("Cannot return null from a non-@Nullable @Provides method");
    }
    return provided;
  }

  public static Factory<Navigator> create(ApplicationModule module) {  
    return new ApplicationModule_ProvideNavigatorFactory(module);
  }
}
```

这个类非常简单，它代表我们的 ApplicationModule（包含 @Provide 方法）创建了 Navigator 类。

总之，上面的代码看起来就像是手敲出来的，而且非常好理解，便于调试。其余还有很多可以去探索，你们可以通过调试去看看 Dagger 如何完成依赖绑定的。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbuzd4lrmj30sg0h37as.jpg)

#源码:

例子: [android10/Android-CleanArchitecture | github.com](https://github.com/android10/Android-CleanArchitecture)

#相关文章: 

- [Architecting Android…The clean way?.](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)
- [Dagger 2, A New Type of Dependency Injection.](https://www.youtube.com/watch?v=oK_XtfXPkqw)
- [Dependency Injection with Dagger 2.](https://speakerdeck.com/jakewharton/dependency-injection-with-dagger-2-devoxx-2014)
- [Dagger 2 has Components.](https://publicobject.com/2014/11/15/dagger-2-has-components/)
- [Dagger 2 Official Documentation.](http://google.github.io/dagger/)