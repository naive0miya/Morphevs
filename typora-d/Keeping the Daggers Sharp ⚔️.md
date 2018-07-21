# 来自 Square 的 Dagger 最佳实践

> 原文 (Medium)：[Keeping the Daggers Sharp ⚔️](https://medium.com/square-corner-blog/keeping-the-daggers-sharp-%EF%B8%8F-230b3191c3f)
>
> 作者：[Py ⚔](https://medium.com/@pyricau?source=post_header_lockup)

[TOC]

Dagger 2 是一个很棒的依赖注入库，但它锋利的边缘可能很难处理。 让我们来回顾一下为了防止手机工程师伤害自己，我们来看看这些来自 Square  最佳实践吧！ 

## 构建器注入优于字段的注入

- 字段注入要求字段是非最终的和非私有的。

```java
// BAD
class CardConverter {
  @Inject PublicKeyManager publicKeyManager;
  @Inject public CardConverter() {}
}
```

- 忘记字段上的 @Inject 导致 NullPointerException。

```java
// BAD
class CardConverter {
  @Inject PublicKeyManager publicKeyManager;
  Analytics analytics; // Oops, forgot to @Inject
  @Inject public CardConverter() {}
}
```

- 构造函数注入之所以更好，是因为它允许不可变的因此线程安全的对象不具有部分构造状态 。

```java
// GOOD
class CardConverter {
  private final PublicKeyManager publicKeyManager;
  @Inject public CardConverter(PublicKeyManager publicKeyManager) {
    this.publicKeyManager = publicKeyManager;
  }
}
```

- Kotlin 消除了构造器注入样板：

```kotlin
class CardConverter
@Inject constructor(
  private val publicKeyManager: PublicKeyManager
)
```

- 对于系统构建的对象，我们仍然使用字段注入，如 Android 活动: 

```java
public class MainActivity extends Activity {
  public interface Component {
    void inject(MainActivity activity);
  }
  @Inject ToastFactory toastFactory;
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    Component component = SquareApplication.component(this);
    component.inject(this);
  }
}
```

## 单例应该是极其罕见的

- 当我们需要集中访问一个可变状态时， 单例是有用的。

```java
// GOOD
@Singleton
public class BadgeCounter {
  public final Observable<Integer> badgeCount;
  @Inject public BadgeCounter(...) {
     badgeCount = ...
  }  
}
```

- 如果一个对象没有可变状态，它不需要是一个单例。

```java
// BAD, should not be a singleton!  
@Singleton
class RealToastFactory implements ToastFactory {
  private final Application context;
  @Inject public RealToastFactory(Application context) {
    this.context = context;
  }
  @Override public Toast makeText(int resId, int duration) {
    return Toast.makeText(context, resId, duration);
  }
}
```

- 在极少数情况下，我们使用范围界定来缓存创建或重复创建和丢弃的实例 。

## @Inject 优于 @Provides

- @Provides 方法不应该重复构造函数模板 
-  当耦合关注在一个地方时，代码更容易理解 

```java
@Module
class ToastModule {
  // BAD, remove this binding and add @Inject to RealToastFactory
  @Provides RealToastFactory realToastFactory(Application context) {
    return new RealToastFactory(context);
  }
}
```

- 这对单例来说尤其重要。这是你阅读该类时需要了解的关键实现细节。

```java
// GOOD, I have all the details I need in one place.
@Singleton
public class BadgeCounter {
  @Inject public BadgeCounter(...) {}  
}
```

## 赞成 static @Provides 方法

- Dagger @Provides 方法可以是静态的。

```java
@Module
class ToastModule {
  @Provides
  static ToastFactory toastFactory(RealToastFactory factory) {
    return factory;
  }
}
```

- 生成的代码可以直接调用该方法，而不必创建模块实例。该方法调用可以由编译器内联。

```java
@Generated
public final class DaggerAppComponent extends AppComponent {
  // ...
  @Override public ToastFactory toastFactory() {
    return ToastModule.toastFactory(realToastFactoryProvider.get())
  }
}
```

- 一个静态方法不会有太大的改变，但是所有的静态绑定都会导致性能的大幅提升。
- 让你的模块抽象，如果其中一个 @Provides 方法不是静态的，那么会导致 Dagger 在[编译时会失败](https://github.com/google/dagger/issues/621#issuecomment-321868005)。

```java
@Module
abstract class ToastModule {
  @Provides
  static ToastFactory toastFactory(RealToastFactory factory) {
    return factory;
  }
}
```

## @Binds 优于 @Provides

- 当你将一种类型映射到另一种类型时，@Binds 将替换 @Provides。

```java
@Module
abstract class ToastModule {
  @Binds
  abstract ToastFactory toastFactory(RealToastFactory factory);
}
```

- 该方法必须是抽象的。它永远不会被调用；生成的代码将知道如何直接使用实现 

```java
@Generated
public final class DaggerAppComponent extends AppComponent {
  // ...
  private DaggerAppComponent() {
    // ...
    this.toastFactoryProvider = (Provider) realToastFactoryProvider;
  }
  @Override public ToastFactory toastFactory() {
    return toastFactoryProvider.get();
  }
}
```

## 在接口绑定上避免 @Singleton

> 有状态是一个实现细节

- 只有实现知道他们是否需要确保集中访问可变状态 
- 当将实现绑定到接口时，不应该有任何范围注解

```java
@Module
abstract class ToastModule {
  // BAD, remove @Singleton
  @Binds @Singleton
  abstract ToastFactory toastFactory(RealToastFactory factory);
}
```

## 启用 error-prone

几个 Square 团队正在使用它来检测常见的 Dagger 错误，[检查它](https://github.com/google/error-prone)。

## 结束

这些指导原则适用于我们的上下文：在大型共享 Android 代码库上工作的小型异构团队。由于你的环境可能不同，你应该应用对你的团队最有意义的内容。

