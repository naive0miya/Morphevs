# Dagger 2 的依赖注入 - UserScope

> 原文 (mirekstanek.online) ：[Building UserScope with Dagger2](https://mirekstanek.online/building-userscope-with-dagger2/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs) 

[TOC]

在 [Azimo Android](https://play.google.com/store/apps/details?id=com.azimo.sendmoney) 应用程序中，我们使用 Dagger 2 作为依赖注入的框架，使我们的代码架构变得简洁而且易于扩展。 就像我们[以前的文章](https://medium.com/azimolabs/dagger-2-on-production-reducing-methods-count-5a13ff671e30#.60fkulphh)，我们想分享我们的经验，所以今天我将展示如何避免问题和构建用于生产应用程序的自定义 UserScope。

**TL;DR**
Dagger 2 中的自定义范围可以更好地控制依赖关系，这些依赖关系应该存在不寻常的时间长度(不同于应用程序和屏幕生存时间)。但要在 Android 应用程序中正确实现它，我们需要记住以下事情：范围不能比应用程序长，进程可以被系统杀死，并在用户流中用新对象实例进行恢复。

示例源代码在这里可用：[Dagger 2 recipes: UserScope](https://github.com/frogermcs/Dagger2Recipes-UserScope)。

在我的一篇关于 Dagger 2 的博客[文章](http://frogermcs.github.io/dependency-injection-with-dagger-2-custom-scopes/)中，我写了关于构建自定义范围和子组件的信息。作为一个例子，我使用了 UserScope，只要用户登录，它就应该存在。 范围生命周期的例子：
![@两个提出的用户范围是不一样的](https://cdn-images-1.medium.com/max/800/1*uib-RZ30N07zYszvH1yiXA.png)

虽然看起来相当简单，但是它的实现表明，这个帖子远离可以生产的代码。 这就是为什么我想再次讨论这个问题 - 但更多的是在实施的情况下。

## Example app
我们将构建3屏应用程序，它将能够从 [Github API](https://developer.github.com/v3/) 获取用户详细信息（让我们假设这将是生产应用程序中的认证事件）。

**应用程序将有3个范围：**
- 应用程序范围（@Singleton） - 与应用程序一样长的依赖关系。
- @UserScope - 只要用户会话处于活动状态（在单个应用程序启动期间）就存在的依赖关系。**重要的是：**这个范围不会比应用程序本身更长。每个新的应用程序实例都将创建新的 @UserScope（即使用户会话没有在应用程序启动之间完成）。
- @ActivityScope - 只要活动屏幕存活的依赖关系。

#### 它是如何工作的？

当应用程序从给定的用户名从 Github API 获取数据时，将打开新的屏幕（UserDetails）。 幕后 LoginActivityPresenter 要求 UserManager 启动会话（从 Github API 获取数据）。当操作成功时，用户被保存到 UserDataStore，UserManager 创建 UserComponent。

```java
//UserManager
public Observable<User> startSessionForUser(String username) {
    return githubApiService.getUser(username)
            .map(User.UserResponseToUser())
            .doOnNext(new Action1<User>() {
                @Override
                public void call(User user) {
                    userDataStore.createUser(user);
                    startUserSession();
                }
            })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread());
}

private boolean startUserSession() {
    User user = userDataStore.getUser();
    if (user != null) {
        Timber.i("Session started, user: %s", user);
        userComponent = userComponentBuilder.sessionModule(new UserModule(user)).build();
        return true;
    }

    return false;
}
```
UserManager 是一个 singleton 对象，所以它的存活时间和 Application 一样长，并且负责 UserComponent，只要用户会话存在，该引用就被保留。

当用户决定关闭会话时，将移除组件，所有注释了 @UserScope 的对象都应该准备好用于 GC。

```java
//UserManager
public void closeUserSession() {
    Timber.i("Close session for user: %s", userDataStore.getUser());
    userComponent.logoutManager().startLogoutProcess();
    userDataStore.clearUser();
    userComponent = null;
}
```

## 组件层次结构
在我们的应用程序中，所有的子组件都使用 AppComponent 作为根组件。用于显示与用户相关的内容的屏幕的子组件使用保存 @UserScope 注释对象的 UserComponent（每个用户会话一个实例）。

![](https://cdn-images-1.medium.com/max/800/1*MC8EMl8t0esUUmvrBg5mIA.png)

UserDetailsActivityComponent 的示例组件层次结构如下所示：
```java
// AppComponent.java 

@Singleton
@Component(modules = {
        AppModule.class,
        GithubApiModule.class
})
public interface AppComponent {
    UserComponent.Builder userComponentBuilder();
}


// UserComponent.java 

@UserScope
@Subcomponent(modules = UserModule.class)
public interface UserComponent {
    
    @Subcomponent.Builder
    interface Builder {
        UserComponent.Builder sessionModule(UserModule userModule);
        UserComponent build();
    }

    UserDetailsActivityComponent plus(UserDetailsActivityComponent.UserDetailsActivityModule module);
}

// UserDetailsActivityComponent.java 

@ActivityScope
@Subcomponent(modules = UserDetailsActivityComponent.UserDetailsActivityModule.class)
public interface UserDetailsActivityComponent {
    UserDetailsActivity inject(UserDetailsActivity activity);
}
```
对于 UserComponent，我们使用 [Subcomponent 构建器](http://google.github.io/dagger/subcomponents.html#subcomponent-builders)模式来使我们的代码更清晰，并有可能将此构建器注入到 UserModule（UserComponent.Builder 用于创建上面显示的组件 UserModule.startUserSession 方法）。

## 在应用程序启动之间恢复 UserScope
正如我之前提到的，UserScope 无法延长应用程序的运行时间。 所以如果我们假设用户会话可以在应用程序启动之间存储，我们需要为 UserScope 处理状态恢复操作。 有两种情况我们需要记住：

**用户从头开始应用程序**

这是最常见的情况，应该是相当简单的处理。 用户从头开始应用（例如，通过点击应用图标）。 应用程序对象被创建，然后第一个活动（LAUNCHER）被启动。如果我们的数据存储中有任何保存的用户，我们需要提供简单的逻辑检查。如果是的用户被转发到正确的屏幕（UserDetailsActivity 在我们的情况下）的 UserComponent 自动创建。

**程序进程被系统杀死**

但也有一个我们经常忘记的情况。 应用程序进程可以被系统杀死（例如由于内存不足）。 这意味着所有的应用程序数据被破坏（应用程序，活动，静态字段）。 不幸的是，这个问题不能很好地处理 - 我们在应用程序生命周期中没有任何回调。 更复杂的是，Android 保存活动堆栈。 这意味着当用户决定启动在流程中间某个地方死亡的应用程序时，系统会尝试将用户带回这个屏幕。

对于我们来说，这意味着我们需要做好准备，从它所使用的任何屏幕上恢复 UserComponent。 
我们来考虑一下这个例子：
1. 用户在 UserDetailsActivity 屏幕上最小化的应用程序
2. 整个应用程序被系统杀死（后来我将展示如何模拟它）
3. 用户打开任务切换器，点击我们的应用程序屏幕。
4. 系统创建应用程序的新实例。意思是创建新的 AppComponent。然后，不打开 LoginActivity（这是我们的启动器活动），系统立即打开 UserDetailsActivity。

对于我们来说，这意味着必须返回 UserComponent（必须创建新实例）。这是我们的责任。示例解决方案可以如下所示：
```java
public abstract class BaseUserActivity extends BaseActivity {

    @Inject
    UserManager userManager;

    @Override
    protected void setupActivityComponent(AppComponent appComponent) {
        appComponent.inject(this);
        setupUserComponent();
    }

    private void setupUserComponent() {
    isUserSessionStarted = userManager.isUserSessionStartedOrStartSessionIfPossible();
    onUserComponentSetup(userManager.getUserComponent());

    //This screen cannot work when user session is not started.
    if (!isUserSessionStarted) {
        finish();
    }
}

    protected abstract void onUserComponentSetup(UserComponent userComponent);

}
```
每个使用 UserComponent 的 Activity 必须扩展 BaseUserActivity 类（setupActivityComponent（）方法在 BaseActivity 中的 onCreate（）方法中调用）。

UserManager 是从 Application 创建的 AppComponent 中注入的。会话以这种方式开始：
```java
public boolean isUserSessionStartedOrStartSessionIfPossible() {
    return userComponent != null || startUserSession();
}

private boolean startUserSession() {
    //This method was described earlier
}
```

**如果用户不存在，该怎么办？**

所以还有另一种情况需要处理 - 如果用户注销（例如，通过 SyncAdapter）并且无法创建 UserComponent？这就是为什么我们在 BaseUserActivity 中有这些行：

```java
private void setupUserComponent() {
    //...

    //This screen cannot work when user session is not started.
    if (!isUserSessionStarted) {
        finish();
    }
}
```
但是，如果有可能无法创建 UserComponent，我们必须记住依赖注入不会发生。这就是为什么我们需要在 onCreate（）方法中每次检查它以防止注入的依赖项上的 NullPointerExceptions。

```java
//UserDetailsActivity
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_user_details);
    tvUser = (TextView) findViewById(R.id.tvUser);
    tvUserUrl = (TextView) findViewById(R.id.tvUserUrl);

    //This check has to be called. Otherwise, when user is not logged in, presenter won't be injected and this line will cause NPE
    if (isUserSessionStarted()) {
        presenter.onCreate();
    }
}
```

**如何模拟应用程序进程被系统 kill**
- 在你想要测试的屏幕上打开应用程序
- 使用系统主页按钮最小化应用程序。
- 在 Android Studio 中，Android 监视器选择你的应用程序，然后单击终止
- 现在在你的设备上打开任务切换器，找到你的应用程序（你仍然应该看到最后一个可见的屏幕预览）。
- 你的应用程序已启动，新的应用程序实例已创建，且活动已恢复。

## 示例源码

显示如何创建和使用 UserComponent 的工作示例的源代码可在 Github 中获取：[Dagger2 Recipes - UserScope](https://github.com/frogermcs/Dagger2Recipes-UserScope)。

