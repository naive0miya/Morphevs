# Dagger 2 的依赖注入 - 减少方法数

> 原文 (mirekstanek.online) ：[Dagger 2 on production — reducing methods count](https://mirekstanek.online/dagger-2-on-production%e2%80%8a-%e2%80%8areducing-methods-count/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)

[TOC]

Dagger 2 - 完全静态的编译时依赖注入框架, 是 [Azimo](https://play.google.com/store/apps/details?id=com.azimo.sendmoney) Android 应用程序中代码体系结构的支柱。 我们已经知道，随着开发团队的简洁代码结构是每个项目中最重要的东西之一。 初始化/使用分离，更简单的测试（单元或功能），更好的可伸缩性 - 这些只是使用依赖注入框架，如 Dagger 2 的一些好处。

但是这些只是展示了如何开始使用DI。这就是为什么今天我们想要开始分享我们的经验使用 dagger2 近两年后，我们的生产应用程序。

## @Component vs @Subcomponent
在简单的教程中并不总是很清楚，但是自定义范围是 Dagger 2 中最强大的功能之一。在更复杂的应用程序中，仅使用 @Singleton 范围不会使你的生活更轻松。

让我们考虑一个简单的例子 - 我们拥有与当前登录用户严格关联的依赖关系（例如 UserProfilePresenter 类，它负责用户配置文件屏幕中的逻辑）。 在每次需要用户实体的时候，我们都可以创建 @User 范围，并将所有相关的依赖关系保存为 UserComponent 中的一个实例，只要用户登录就可以使用。

> 在生产应用程序中使用自定义范围不是这篇文章的主题，但是我们也会在不久的将来报道这个问题。

在 Dagger 2中，我们有两种方法来构建自定义范围和组件来继承和扩展对象图:

- 我们可以构建另一个 @Component，明确地显示哪个组件被扩展（在这种情况下是 AppComponent）
```java
@UserScope
@Component(
    modules = UserModule.class,
    dependencies = AppComponent.class
)
public interface UserComponent {
    UserProfileActivity inject(UserProfileActivity activity);
}
```
- 或者我们可以定义 @Subcomponent，它是用抽象工厂方法从基本组件创建的（[阅读更多](http://google.github.io/dagger/api/latest/dagger/Component.html#subcomponents)）
```java
@UserScope
@Subcomponent(
    modules = UserModule.class
)
public interface UserComponent {
    UserProfileActivity inject(UserProfileActivity activity);
}

//===== AppComponent.java =====

@Singleton
@Component(modules = {...})
public interface AppComponent {
    // Factory method to create subcomponent
    UserComponent plus(UserModule module);
}
```

## 这两种方法有什么区别？
[文档](http://google.github.io/dagger/api/latest/dagger/Component.html#component-dependencies)介绍了依赖关系可视性的差异。 @Subcomponent 可以访问父项的所有依赖项，而依赖 @Component 只能访问基本的 @Component 接口公开的依赖项。

考虑到这一点，我们最初选择依赖 @Component 的方法。我们项目中的所有组件都依赖于使用 @Singleton 作用域的 AppComponent。

## 方法计数事项...
但 @Component 和 @Subcomponent 还有另一个重要的区别。 我们来看看生成的代码。 子组件实现只是基本组件中的一个内部类。
所以对于依赖于 AppComponent 生成的代码的 UserProfileActivityComponent 看起来像这样：
```java
public final class DaggerAppComponent implements AppComponent {

    //...AppComponent code...

    private final class UserProfileActivityComponentImpl implements UserProfileActivityComponent {
        private final UserProfileActivityComponent.UserProfileActivityModule userProfileActivityModule;
        private Provider<UserProfileActivity> provideActivityProvider;
        private Provider<UserProfileActivityPresenter> userProfileActivityPresenterProvider;
        private MembersInjector<UserProfileActivity> userProfileActivityMembersInjector;

        private UserProfileActivityComponentImpl(
            UserProfileActivityComponent.UserProfileActivityModule userProfileActivityModule) {
            this.userProfileActivityModule = Preconditions.checkNotNull(userProfileActivityModule);
            initialize();
        }

        private void initialize() {
            this.provideActivityProvider = DoubleCheck.provider(BaseActivityModule_ProvideActivityFactory.create(userProfileActivityModule));

            this.userProfileActivityPresenterProvider = DoubleCheck.provider(
                UserProfileActivityPresenter_Factory.create(
                    MembersInjectors.<UserProfileActivityPresenter>noOp(),
                    provideActivityProvider,
                    DaggerAppComponent.this.logoutManagerProvider,
                    DaggerAppComponent.this.userManagerProvider)
            );

            this.userProfileActivityMembersInjector = UserProfileActivity_MembersInjector.create(
                DaggerAppComponent.this.logoutManagerProvider,
                DaggerAppComponent.this.userManagerProvider
                userProfileActivityPresenterProvider)
            );
        }

        @Override
        public UserProfileActivity inject(UserProfileActivity activity) {
            userProfileActivityMembersInjector.injectMembers(activity);
            return activity;
        }
    }
}
```
第20-31行向我们展示了 Dagger 如何提供从 AppComponent 到 UserProfileActivityComponent 所需的依赖关系。子组件可以访问基本组件提供程序字段，因此可以直接使用它们。

不同的情况是依赖 @Component。相同的结构（UserProfileActivityComponent 依赖于 AppComponent）将产生这个生成的代码：

```java
public final class DaggerUserProfileActivityComponent implements UserProfileActivityComponent {
    private Provider<LogoutManager> logoutManagerProvider;
    private Provider<UserManager> userManagerProvider;

    //...

    private DaggerUserProfileActivityComponent(Builder builder) {
        assert builder != null;
        initialize(builder);
    }

    @SuppressWarnings("unchecked")
    private void initialize(final Builder builder) {
        this.logoutManagerProvider = new Factory<LogoutManager>() {
            private final AppComponent appComponent = builder.appComponent;

            @Override
            public LogoutManager get() {
                return Preconditions.checkNotNull(
                    appComponent.getLogoutManager(),
                    "Cannot return null from a non-@Nullable component method");
            }
        };

        this.userManagerProvider = new Factory<UserManager>() {
            private final AppComponent appComponent = builder.appComponent;

            @Override
            public UserManager get() {
                return Preconditions.checkNotNull(
                    appComponent.getUserManager(),
                    "Cannot return null from a non-@Nullable component method");
            }
        };

        //... more providers ....
        
    }

    @Override
    public UserProfileActivity inject(UserProfileActivity activity) {
        userProfileActivityMembersInjector.injectMembers(activity);
        return activity;
    }

    //...
}
```
第14-34行显示，从 AppComponent 请求的每个依赖项都需要有自己的方法来为其生成 Provider <> 对象。 为什么？ 因为依赖组件只能访问基本组件中的那些依赖于接口公共的依赖项。

这对我们来说意味着什么 - Android 开发者？从基本组件中获取的每个相关组件的相关依赖将使我们更接近65k 方法的限制:

> Methods count matters? Use [@Subcomponent](https://twitter.com/subcomponent), not dependent [@Component](https://twitter.com/component) which creates additional methods for each Provider<> [#AndroidDev](https://twitter.com/hashtag/AndroidDev?src=hash) [#Dagger2](https://twitter.com/hashtag/Dagger2?src=hash)
>
> -Mirek Stanek (@froger_mcs)  [10:22 PM - Jul 16, 2016 twitter](https://twitter.com/froger_mcs/status/754320419759460352?ref_src=twsrc%5Etfw&ref_url=https%3A%2F%2Fmedium.com%2Fmedia%2F9724270dc1c60a777f4d3fcbcd0f0d1e%3FpostId%3D5a13ff671e30)

正因为如此，我们决定将我们所有依赖的 @Components 迁移到 @Subcomponent。在这个操作的结果中，我们减少了约5000个额外的方法。

## 快速分析你的 APK
最后值得一提的是，从 Android Studio 2.2 开始，有一个新功能可以让我们分析 APK 文件。它可以从 Build -> Analyze APK... 访问
![@在底部，我们可以看到由Dagger和从属@Component生成的示例匿名内部类](https://cdn-images-1.medium.com/max/800/1*LX37WOAZPGsx9W73P879zg.png)

这个工具允许我们检查特定组件（类，资源等）的大小以及对我们最有帮助的东西 - 获取有关 .dex 文件（类/方法数量）的基本信息。 这是我们依赖注入优化的出发点。

这就是今天的内容，但请保持期待。不久，我们将分享更多我们的经验关于 Dagger 2 和依赖注入我们的 [Android 应用程序](https://play.google.com/store/apps/details?id=com.azimo.sendmoney)。谢谢阅读！



