# Dagger 2 的依赖注入 - 活动 子组件 Multibinding

> 原文 (mirekstanek.online)：[Activities Subcomponents Multibinding in Dagger 2](https://mirekstanek.online/activities-subcomponents-multibinding-in-dagger-2/)
>
> 作者 ：[Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章的第一个版本最初写在我的开发博客上：http://frogermcs.github.io/

几个月前，在 [MCE³](http://2016.mceconf.com/) 会议期间，Gregory Kick 在他的[演讲](https://www.youtube.com/watch?v=iwjXqRlEevg&feature=youtu.be)中展示了提供子组件（例如活动）的新概念。 新的方法应该给我们一个方法来创建 ActivitySubcomponent 而不需要有 AppComponent 对象引用（它曾经是一个活动子组件的工厂）。
为了使它成为现实，我们不得不等待新版本的 Dagger：[版本2.7](https://github.com/google/dagger/releases/tag/dagger-2.7)。

## 问题
在 Dagger 2.7 之前，为了创建子组件（例如，作为 AppComponent 子组件的 MainActivityComponent），我们必须在父组件中声明其工厂：
```java
@Singleton
@Component(
    modules = {
        AppModule.class
    }
)
public interface AppComponent {
    MainActivityComponent plus(MainActivityComponent.ModuleImpl module);

    //...
}
```
感谢这个声明，Dagger 知道 MainActivityComponent 可以访问 AppComponent 的依赖关系。

有了这个， MainActivity 中的注入看起来类似于：
```java
@Override
protected ActivityComponent onCreateComponent() {
    ((MyApplication) getApplication()).getComponent().plus(new MainActivityComponent.ModuleImpl(this));
    component.inject(this);
    return component;
}
```
这个代码的问题是：
- 活动依赖于 AppComponent（由 ((MyApplication) getApplication()).getComponent() 返回 - 无论何时我们想要创建 Subcomponent，我们都需要访问父 Component 对象。
- AppComponent 必须为所有子组件（或其构建者）声明工厂，例如： MainActivityComponent plus(MainActivityComponent.ModuleImpl module);

## Modules.subcomponents
从 Dagger 2.7 开始，我们有了新的方式来声明子组件的父母。 @Module 注释具有可选的[子组件](http://google.github.io/dagger/api/2.7/dagger/Module.html#subcomponents--)字段，该字段获取子组件类的列表，子组件应该是安装此模块的组件的子类。

例：
```java
@Module(
        subcomponents = {
                MainActivityComponent.class,
                SecondActivityComponent.class
        })
public abstract class ActivityBindingModule {
    //...
}
```
ActivityBindingModule 安装在 AppComponent 中。 这意味着：MainActivityComponent 和 SecondActivityComponent 都是 AppComponent 的子组件。
以这种方式声明的子组件不必在 AppComponent 中显式声明（就像在本文中的第一个代码清单中所做的那样）。

## 活动多重绑定
我们来看看如何使用 Modules.subcomponents 来构建活动的多重绑定，并摆脱传递给 Activity 的 AppComponent 对象（在本[演讲](https://youtu.be/iwjXqRlEevg?t=1693)结束时也解释了这一点）。我将只通过代码中最重要的部分。

整个实现在 Github上可用：[Dagger2Recipes-ActivitiesMultibinding](https://github.com/frogermcs/Dagger2Recipes-ActivitiesMultibinding)。

我们的应用程序包含两个简单的屏幕：MainActivity 和 SecondActivity。我们希望能够在不传递 AppComponent 对象的情况下为它们提供 Subcomponents。

我们从构建所有活动组件构建器的基础接口开始：
```java
public interface ActivityComponentBuilder<M extends ActivityModule, C extends ActivityComponent> {
    ActivityComponentBuilder<M, C> activityModule(M activityModule);
    C build();
}
```
示例子组件：MainActivityComponent 可能如下所示：
```java
@ActivityScope
@Subcomponent(
        modules = MainActivityComponent.MainActivityModule.class
)
public interface MainActivityComponent extends ActivityComponent<MainActivity> {

    @Subcomponent.Builder
    interface Builder extends ActivityComponentBuilder<MainActivityModule, MainActivityComponent> {
    }

    @Module
    class MainActivityModule extends ActivityModule<MainActivity> {
        MainActivityModule(MainActivity activity) {
            super(activity);
        }
    }
}
```
现在，我们希望将 Subcomponents 构建器的 Map，以便能够为每个活动类获得预期的构建器。为此，我们使用多重绑定功能：

```java
@Module(
        subcomponents = {
                MainActivityComponent.class,
                SecondActivityComponent.class
        })
public abstract class ActivityBindingModule {

    @Binds
    @IntoMap
    @ActivityKey(MainActivity.class)
    public abstract ActivityComponentBuilder mainActivityComponentBuilder(MainActivityComponent.Builder impl);

    @Binds
    @IntoMap
    @ActivityKey(SecondActivity.class)
    public abstract ActivityComponentBuilder secondActivityComponentBuilder(SecondActivityComponent.Builder impl);
}
```
在 AppComponent 中安装 ActivityBindingModule。 正如解释的那样，由于这个 MainActivityComponent 和 SecondActivityComponent 将是 AppComponent 的子组件。

```java
@Singleton
@Component(
        modules = {
                AppModule.class,
                ActivityBindingModule.class
        }
)
public interface AppComponent {
    MyApplication inject(MyApplication application);
}
```
现在我们可以注入 Subcomponents 构建器的 Map（例如 MyApplication 类）：
```java
public class MyApplication extends Application implements HasActivitySubcomponentBuilders {

    @Inject
    Map<Class<? extends Activity>, ActivityComponentBuilder> activityComponentBuilders;

    private AppComponent appComponent;

    public static HasActivitySubcomponentBuilders get(Context context) {
        return ((HasActivitySubcomponentBuilders) context.getApplicationContext());
    }

    @Override
    public void onCreate() {
        super.onCreate();
        appComponent = DaggerAppComponent.create();
        appComponent.inject(this);
    }

    @Override
    public ActivityComponentBuilder getActivityComponentBuilder(Class<? extends Activity> activityClass) {
        return activityComponentBuilders.get(activityClass);
    }
}
```
为了获得额外的抽象，我们创建了 HasActivitySubcomponentBuilders 接口（因为构建器的 Map 不必被注入 Application 类）：

```java
public interface HasActivitySubcomponentBuilders {
    ActivityComponentBuilder getActivityComponentBuilder(Class<? extends Activity> activityClass);
}
```
最后在 Activity 类中注入实现：
```java
public class MainActivity extends BaseActivity {

    //...

    @Override
    protected void injectMembers(HasActivitySubcomponentBuilders hasActivitySubcomponentBuilders) {
        ((MainActivityComponent.Builder) hasActivitySubcomponentBuilders.getActivityComponentBuilder(MainActivity.class))
                .activityModule(new MainActivityComponent.MainActivityModule(this))
                .build().injectMembers(this);
    }
}
```
这与我们第一个实现非常相似，但是如前所述，最重要的是我们不再将 ActivityComponent 对象传递给我们的活动。

## 使用示例 - 仪器测试模拟
除了松散的耦合和固定的循环依赖（Activity \< - > Application），并不总是一个大问题，尤其是在小型项目/团队中，让我们考虑一下真正的用例，其中我们的实现可能是有用的 - 模拟仪器测试中的依赖关系。

目前在 Android Instrumentation 测试中模拟依赖关系的最知名方法之一是使用 [DaggerMock](https://medium.com/@fabioCollini/android-testing-using-dagger-2-mockito-and-a-custom-junit-rule-c8487ed01b56#.v7bm9bd3o)（Github [项目链接](https://github.com/fabioCollini/DaggerMock)）。 虽然 DaggerMock 是强大的工具，但很难理解它是如何工作的。 其中有一些不容易追踪的反射码。

在 Activity 中直接构建子组件不需要访问 AppComponent 类，这使我们可以测试每一个与应用程序其余部分分离的 Activity。

在我们的仪器测试中使用的应用程序类：
```java
public class ApplicationMock extends MyApplication {

    public void putActivityComponentBuilder(ActivityComponentBuilder builder, Class<? extends Activity> cls) {
        Map<Class<? extends Activity>, ActivityComponentBuilder> activityComponentBuilders = new HashMap<>(this.activityComponentBuilders);
        activityComponentBuilders.put(cls, builder);
        this.activityComponentBuilders = activityComponentBuilders;
    }
}
```
方法 putActivityComponentBuilder（）给我们一个方法来替换给定的 Activity 类的 ActivityComponentBuilder 的实现。

现在看一下我们的示例 Espresso Instrumentation Test：
```java
@RunWith(AndroidJUnit4.class)
public class MainActivityUITest {

    @Rule
    public MockitoRule mockitoRule = MockitoJUnit.rule();

    @Rule
    public ActivityTestRule<MainActivity> activityRule = new ActivityTestRule<>(MainActivity.class, true, false);

    @Mock
    MainActivityComponent.Builder builder;
    @Mock
    Utils utilsMock;

    private MainActivityComponent mainActivityComponent = new MainActivityComponent() {
        @Override
        public void injectMembers(MainActivity instance) {
            instance.mainActivityPresenter = new MainActivityPresenter(instance, utilsMock);
        }
    };

    @Before
    public void setUp() {
        when(builder.build()).thenReturn(mainActivityComponent);
        when(builder.activityModule(any(MainActivityComponent.MainActivityModule.class))).thenReturn(builder);

        ApplicationMock app = (ApplicationMock) InstrumentationRegistry.getTargetContext().getApplicationContext();
        app.putActivityComponentBuilder(builder, MainActivity.class);
    }

    @Test
    public void checkTextView() {
        String expectedText = "lorem ipsum";
        when(utilsMock.getHardcodedText()).thenReturn(expectedText);

        activityRule.launchActivity(new Intent());

        onView(withId(R.id.textView)).check(matches(withText(expectedText)));
    }

}
```
逐步：
- 我们提供 MainActivityComponent.Builder 的模拟和所有必须被模拟的依赖关系（在这种情况下只是 Utils）。 我们的模拟 Builder 返回 MainActivityComponent 的自定义实现，它将注入 MainActivityPresenter（带有模拟的Utils对象）。
- 然后，我们的 MainActivityComponent.Builder 替换 MyApplication（第28行）中注入的原始构建器：app.putActivityComponentBuilder（builder，MainActivity.class）;
- 最后测试 - 我们模拟 Utils.getHardcodedText（）方法。注入过程发生在创建 Activity（第36行）时：activityRule.launchActivity（new Intent（））;在这里，我们只是用 Espresso 来检查结果。

就这样。正如你所看到的，MainActivityUITest 类中几乎所有事情都发生了，代码非常简单易懂。

## 示例代码
如果您想自己测试实现，则可以在 Github 上找到具有工作示例的源代码，其中演示了如何创建活动 Multibinding 和 Mock 依赖关系：[Dagger2Recipes-ActivitiesMultibinding](https://github.com/frogermcs/Dagger2Recipes-ActivitiesMultibinding)。



