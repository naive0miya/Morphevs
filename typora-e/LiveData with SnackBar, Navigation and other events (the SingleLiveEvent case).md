# SnackBar, Navigation 和其它事件使用 LiveData ( SingleLiveEvent 案例)

> 原文 (Medium) ：[LiveData with SnackBar, Navigation and other events (the SingleLiveEvent case)](https://medium.com/google-developers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)
>
> 作者 ：[Jose Alcérreca](https://medium.com/@JoseAlcerreca?source=post_header_lockup)

视图（活动或片段）与 ViewModel 进行通信的一种便捷方式是使用 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata) 观察者。 该视图订阅 LiveData 中的更改并对其作出反应。 这适用于连续显示在屏幕上的数据。

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvmpg056j30ca03st8l.jpg)

然而，有些数据只应该被消耗一次，比如 Snackbar 消息、Navigation 事件或 Dialog 触发器。

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvmwuxa6j30ca03s747.jpg)

与其试图通过库或者架构组件的扩展来解决这个问题，它应该作为一个设计问题来面对。 我们建议你把你的事件作为你状态的一部分来对待。 在这篇文章中，我们展示了一些常见的错误和建议的方法。

### ❌ Bad: 1. 使用 LiveData 来处理事件

该方法直接在 LiveData 对象内部持有 Snackbar 消息或导航信号。 虽然在原则上它看起来像一个常规的 LiveData 对象可以用来做这个，但它提出了一些问题。

在一个 master/detail 应用程序中，这里是主人的 ViewModel:

```kotlin
// Don't use this for events
class ListViewModel : ViewModel {
    private val _navigateToDetails = MutableLiveData<Boolean>()

    val navigateToDetails : LiveData<Boolean>
        get() = _navigateToDetails


    fun userClicksOnButton() {
        _navigateToDetails.value = true
    }
}
```

在视图(活动或片段) :

```kotlin
myViewModel.navigateToDetails.observe(this, Observer {
    if (it) startActivity(DetailsActivity...)
})
```

这种方法的问题在于_navigateToDetails 的值长时间保持为 true，而且无法返回到第一个屏幕。 一步一步来:

1. 用户点击按钮，细节活动就开始了
2. 用户回复，回到主活动
3. 观察者在活动处于后台堆栈时不活动后，再次变为活动状态
4. 其值仍然存在 true，所以细节活动又错误地开始了

一个解决方案是从 ViewModel 上点击导航，然后立即设置 false:

```kotlin

fun userClicksOnButton() {
    _navigateToDetails.value = true
    _navigateToDetails.value = false // Don't do this
}
```

然而，需要记住的一件重要的事情是 LiveData 保持了它的值，但并不能保证它会释放出它所接收到的每一个值。 例如: 当没有观察者活动时，可以设置一个值，所以一个新的将只是替换它。 此外，从不同的线程设置值可能导致只会产生一个观察者调用的竞争条件。

但是这种方法的主要问题在于它很难理解，而且很丑陋。 在导航事件发生后，我们如何确保重置值？

### **❌ Better: 2.** 使用 LiveData 来处理事件，在 observer 中重置事件值

使用此方法，你可以从视图中添加一个方法来指示你已经处理了事件，并且应该重置该事件。

#### 使用

如果我们的观察者有一点小小的改变，我们或许可以找到解决办法:

```kotlin
listViewModel.navigateToDetails.observe(this, Observer {
    if (it) {
        myViewModel.navigateToDetailsHandled()
        startActivity(DetailsActivity...)
    }
})
```

在 ViewModel 中添加新方法如下:

```kotlin
class ListViewModel : ViewModel {
    private val _navigateToDetails = MutableLiveData<Boolean>()

    val navigateToDetails : LiveData<Boolean>
        get() = _navigateToDetails


    fun userClicksOnButton() {
        _navigateToDetails.value = true
    }

    fun navigateToDetailsHandled() {
        _navigateToDetails.value = false
    }
}
```

#### 问题

这种方法的问题在于有一些模板(每个事件中的 ViewModel 中的一个新方法) ，而且它容易出错; 很容易忘记观察者对 ViewModel 的调用。

### **✔️ OK:** 使用 SingleLiveEvent

SingleLiveEvent 类是为样本创建的，作为适用于特定场景的解决方案。这是一个只会发送一次更新的 LiveData。

#### 使用

```kotlin
class ListViewModel : ViewModel {
    private val _navigateToDetails = SingleLiveEvent<Any>()

    val navigateToDetails : LiveData<Any>
        get() = _navigateToDetails


    fun userClicksOnButton() {
        _navigateToDetails.call()
    }
}
```



```kotlin
myViewModel.navigateToDetails.observe(this, Observer {
    startActivity(DetailsActivity...)
})
```

#### 问题

SingleLiveEvent 的问题在于它只限于一个观察者。 如果你不小心添加了一个以上，只有一个会被调用，而且不能保证是哪一个。

![](http://ww1.sinaimg.cn/large/006tNc79gy1frl1fxskcuj30ca03s3yi.jpg)



### **✔️** 推荐: 使用事件包装器

在这种方法中，你可以明确管理事件是否已经处理，从而减少了错误。

#### 使用

```kotlin
/**
 * Used as a wrapper for data that is exposed via a LiveData that represents an event.
 */
open class Event<out T>(private val content: T) {

    var hasBeenHandled = false
        private set // Allow external read but not write

    /**
     * Returns the content and prevents its use again.
     */
    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }

    /**
     * Returns the content, even if it's already been handled.
     */
    fun peekContent(): T = content
}
```



```kotlin
class ListViewModel : ViewModel {
    private val _navigateToDetails = MutableLiveData<Event<String>>()

    val navigateToDetails : LiveData<Event<String>>
        get() = _navigateToDetails


    fun userClicksOnButton(itemId: String) {
        _navigateToDetails.value = Event(itemId)  // Trigger the event by setting a new Event as a new value
    }
}
```



```kotlin
myViewModel.navigateToDetails.observe(this, Observer {
    it.getContentIfNotHandled()?.let { // Only proceed if the event has never been handled
        startActivity(DetailsActivity...)
    }
})
```

这种方法的优点是用户需要使用 getContentIfNotHandled ( ) 或 peekContent ( ) 来指定意图。 这种方法将这些事件作为状态的一部分来模拟: 它们现在只是消费或不消费的信息。

![](http://ww2.sinaimg.cn/large/006tNc79gy1frl1k7z5pvj30ca03s3yh.jpg)

总之，设计事件作为你状态的一部分。 使用你自己的事件包装器在 LiveData 观察者，并定制它适合你的需要。

额外奖励！ 使用这个 [EventObserver](https://gist.github.com/JoseAlcerreca/e0bba240d9b3cffa258777f12e5c0ae9) 来删除一些重复的代码，如果你最终有很多事件。





