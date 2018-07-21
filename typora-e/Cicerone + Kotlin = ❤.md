# Cicerone + Kotlin = ❤

> 原文 (Medium) ：[Cicerone + Kotlin = ❤](https://proandroiddev.com/cicerone-kotlin-da5b2f49d759)
>
> 作者 ：[Konstantin Tskhovrebov ](https://proandroiddev.com/@terrakok?source=post_header_lockup)

### 现代安卓应用

现代安卓应用程序的最佳实践之一就是从标准的 Android 类(片段或活动)中将应用逻辑提取到某个特殊的类。 例如，如果你使用 MVP 模式，它将是一个演示者。 这个类包含所有的演示逻辑，并告诉视图实现他们应该向用户展示的内容。 然而，问题出现了: 如何正确地解决不同屏幕之间的导航任务？ 可能，第一个想法是在视图中声明这样的方法:"去屏幕 x"。 然而，这是一个相当肮脏的方式，因为导航的实现不是视图逻辑的一部分，而且我们违反了关注点分离原则。

### 路由器

解决这种情况的一个可能的解决方案是将导航提取到一个特殊的类ー路由器。 这个类应该被注入到屏幕的演示者中，并且它会以某种方式执行转换调用。 在Android中，你只能使用 Context 对象更改活动 Fragment 或 Activity，而 Router 应该将实际的转换任务委托给所需的 Android 框架类。

### Cicerone

路由器的概念已经在 Cicerone 库成功实现，我想介绍给大家。 其他导航解决方案(Magellan，Flow 等)的主要区别在于它是一个库，而不是一个框架。 你不需要扩展库的任何类别。  Cicerone 只负责将导航命令传输到 Android 框架中，并在应用程序发送到后台时保持其顺序。 因此，即使已经发货并在生产中工作，很容易将 Cicerone 集成到任何应用程序中。

### 关于内部的一些事情

这里有一张图表展示了库是如何工作的:

![](https://ws4.sinaimg.cn/large/006tKfTcgy1ftbtasxqiuj30ll07i74u.jpg)

Presenter 调用路由器的过渡方法。 路由器将导航命令发送到缓冲区，如果应用程序被发送到后台，这将保存它们。 从缓冲区，命令去 Navigator，导航器通常位于一个活动中，这里发生的是实际的屏幕切换。关于库如何工作的细节在 [github](https://github.com/terrakok/Cicerone) 上有描述。 欢迎参加讨论，你可以在那里[问你所有的问题](https://github.com/terrakok/Cicerone/issues)！

### 练习

库的 API 以这样的方式创建，即将 Cicerone 与 Kotlin 一起使用会让任何人都开心！

让我们看看它是怎样的。

### 简单的导航

![](https://cdn-images-1.medium.com/max/1600/0*P4IW1yRGuflDedvj.)

最常见的情况是屏幕之间来回的简单导航。  你可以通过下面的方式实现它(如上所述，命令从"演示者"开始) :

```kotlin
class Presenter(
  private val router: Router
) {
  fun onNextButtonClicked() = router.navigateTo("Next screen")
  fun onBackButtonClicked() = router.exit()
}
```

导航类已经实现，所以你只需要描述如何根据给定的键创建屏幕。

```kotlin
class MainActivity : AppCompatActivity() {

  ... 

  private val navigator = object : SupportAppNavigator(this, R.id.mainContainer) {

    override fun createActivityIntent(screenKey: String?, data: Any?): Intent? = when (screenKey) {
        Screens.AUTH_SCREEN -> Intent(this@MainActivity, AuthActivity::class.java)
        else -> null
    }

    override fun createFragment(screenKey: String?, data: Any?): Fragment? = when (screenKey) {
        Screens.MAIN_SCREEN -> MainFragment()
        Screens.INFO_SCREEN -> InfoFragment.createNewInstance(data as Long)
        Screens.ABOUT_SCREEN -> AboutFragment()
        else -> null
    }
  }

  ... 
}
```

### 动画

![](https://cdn-images-1.medium.com/max/1600/0*zOnT338jXkzJwp8Q.)

第一次发布后的主要问题是如何添加片段更改的动画。 从2.0版本开始，我们可以通过下面的一种方式将动画添加到事务中：

```kotlin
override fun setupFragmentTransactionAnimation(command: Command,
                                               currentFragment: Fragment,
                                               nextFragment: Fragment, 
                                               fragmentTransaction: FragmentTransaction) {
  fragmentTransaction.setCustomAnimations(R.anim.enter_from_right, R.anim.exit_to_left)
}
```

很快，你就可以为 Activity 添加过渡动画了。

### 最终结果

第二个最常被问到的问题是如何将结果从一个屏幕返回到前一个屏幕。 现在我们也覆盖它。

```kotlin
private val RESULT_CODE = 42
init {
  router.setResultListener(RESULT_CODE, { resultData ->
  	Log.i("test", "Take result: $resultData")
  })
}

override fun onDestroy() {
  router.removeResultListener(RESULT_CODE)
}
```



```kotlin
fun onPhotoClick(id: Int) = router.exitWithResult(RESULT_CODE, id)
```

### 嵌套导航

![](https://cdn-images-1.medium.com/max/1600/0*iUEdPBVcpK9qjMcJ.)

最复杂的情况是不同标签页中的独立导航堆栈(如 Instagram)。Cicerone 很容易帮助解决这个任务。 然而，这个过程的描述过于冗长，所以我建议你去库看看[样本](https://github.com/terrakok/Cicerone/tree/develop/sample)，在那里我们有一个这样模式的例子。

### Application

许多案例表明，Cicerone 是基于 mvp 的应用程序。 但是，它在 MVVM 中运行良好，甚至在没有表示逻辑分离的最简单的应用程序中也是如此。 如果你认为它太复杂，只要看一下源代码，你就会发现它是多么的简单。 此外，你可以很容易地扩展该库以更好地处理你的特殊情况。

我希望你会喜欢它，我期待你的拉动请求！



