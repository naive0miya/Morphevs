# Dagger 2 : Component.Builder

> 原文 (Medium)：[Dagger 2 : Component.Builder](https://proandroiddev.com/dagger-2-component-builder-1f2b91237856?source=user_profile---------3----------------)
>
> 作者：[Garima Jain](https://proandroiddev.com/@ragdroid?source=post_header_lockup)

[TOC]

这篇文章是 [Dagger and the Dahaka](https://medium.com/@ragdroid/dagger-2-android-defeat-the-dahaka-b1c542233efc) 系列的一部分。如果你想了解更多关于 “Dahaka” 的文章，请查看关于这个系列的[主文章](https://medium.com/@ragdroid/dagger-2-android-defeat-the-dahaka-b1c542233efc)，这篇文章将如何使我们能够击败 Dahaka 以及这个系列背后的真正动机。

现在，让我们开始吧。 

举一个非常简单的例子。我们有一个 AppComponent 定义如下：

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {
  
  void inject(MainActivity mainActivity);
  SharedPreferences getSharedPrefs();
}
```

下面是我们简单的 AppModule：

```java
 @Module
 public class AppModule {
 
    Application application;
 
    public AppModule(Application application) {
       this.application = application;
    }
 
    @Provides
    Application providesApplication() {
       return application;
    }
    @Provides
    @Singleton
    public SharedPreferences providePreferences() {
        return application.getSharedPreferences(DATA_STORE,
                              Context.MODE_PRIVATE);
    }
 
 }
```

## 实例化组件

组件可以通过使用 Dagger 生成的 Builders 来实例化：

```java
DaggerAppComponent appComponent = DaggerAppComponent.builder()
         .appModule(new AppModule(this)) //this : application 
         .build();
```

允许我们使用 Component.Builder 来定制生成的构建器。

## @Component.Builder 定义

首先，让我们参考 @Component.Builder 的文档

> 组件的构建器。组件可能有一个嵌套的静态抽象类或者使用
>  @Component.Builder 注解的接口。如果他们这样做，那么组件的生成构建器将匹配类型中的 API。
>
>  - [Source](https://google.github.io/dagger/api/latest/)

让我们在 AppComponent 中定义组件构建器：

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {
   
   void inject(MainActivity mainActivity);
   SharedPreferences getSharedPrefs();
   @Component.Builder
   interface Builder {
        AppComponent build();
        Builder appModule(AppModule appModule);
    }
}
```

使用上面定义的 Component.Builder， Dagger 生成和前面完全一样的 Builder 类。哇！ 

现在使用 @BindsInstance 尝试自定义此 Component.Builder 。

## @BindsInstance 是什么?

下面是定义: 

> 在组件构建器或子组件构建器上标记方法，该方法允许实例绑定到组件内部的某个类型。  - [Source](https://google.github.io/dagger/api/latest/)

WHAT？我也不明白

这里有一个简单的提示: 何时使用它: 

> 应该优先使用 @BindsInstance 方法来编写带有构造函数参数的 @Module，并立即提供这些值。 - [Source](https://google.github.io/dagger//users-guide.html)

让我们重新审视作为构造函数参数提供的依赖关系的 AppModule : 

```java
public AppModule(Application application) {
         this.application = application;
 }
```

让我们看看我们简化的 AppModule 的构造函数和移除 @Provides Application ：

```java
@Module
 public class AppModule {
 
     @Provides
     @Singleton
     public SharedPreferences providePreferences(
                                    Application application) {
         return application.getSharedPreferences(
                                    "store", Context.MODE_PRIVATE);
     }
 }
```

下面是我们现在定制的 Component.Builder：

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {
   void inject(MainActivity mainActivity);
   SharedPreferences getSharedPrefs();
   @Component.Builder
   interface Builder {
   
      AppComponent build();
      @BindsInstance Builder application(Application application);      
  }
}
```

请注意，我们不需要指定 Builder appModule (AppModule appModule)；在 component.Builder 内部。因为我们现在要让 dagger 使用 appModule 的缺省构造函数。 

以下是我们将如何初始化 DaggerAppComponent：

```java
DaggerAppComponent appComponent = DaggerAppComponent.builder()
           .application(this)
           .build();
```

在这里，我们也不需要指定 .appModule (new AppModule ( ))。

## Subcomponent.Builder

我们同样有 @Subcomponent.Builder，它们是 Subcomponents 的 Builders，在 Dagger-Android 中被广泛使用。有关子组件构建器的更多信息在即将发布的帖子中。

## TL; DR

系列的文章列表：这里是文章的链接：

- [Dagger 2 : Component.Builder](https://medium.com/@ragdroid/dagger-2-component-builder-1f2b91237856)
- [Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)
- [Dagger 2 : Component Relationships & Custom Scopes](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37)
- [Dagger 2 Annotations : @Binds & @ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f)

可以观看视频：[[Video]](https://www.youtube.com/watch?v=iczp_toHxmA)

