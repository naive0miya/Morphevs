# 使用 Kotlin & RxJava 2 检测 Android 键盘是否打开

> 原文 (Medium)：[Detecting if the Android keyboard is open using Kotlin & RxJava 2](https://medium.com/@munnsthoughts/detecting-if-the-android-keyboard-is-open-using-kotlin-rxjava-2-8aee9fae262c)
>
> 作者：[Andrew Munn](https://medium.com/@munnsthoughts?source=post_header_lockup)

[TOC]

如果软键盘打开或关闭，[Android SDK](https://groups.google.com/forum/#!topic/android-platform/FyjybyM0wGA) 并没有提供一个直接的方式来获得通知。 值得庆幸的是，Stack overflow 上有一些[解决方案](https://stackoverflow.com/questions/4745988/how-do-i-detect-if-software-keyboard-is-visible-on-android-device)。

我采取了解决方法逻辑并将其封装在一个小型的 Kotlin 实用工具类中，它提供了可观察的键盘状态。这是一个有用的和惯用的例子，如何将 Android SDK 中的 Listener 式的回调转换为 Observable 。

```kotlin
/
  Manages access to the Android soft keyboard.
 /
class KeyboardManager @Inject constructor(private val activity: Activity) {

  /
    Observable of the status of the keyboard. Subscribing to this creates a
    Global Layout Listener which is automatically removed when this 
    observable is disposed.
   /
  fun status() = Observable.create<KeyboardStatus> { emitter ->
    val rootView = activity.findViewById<View>(android.R.id.content)

    // why are we using a global layout listener? Surely Android 
    // has callback for when the keyboard is open or closed? Surely
    // Android at least lets you query the status of the keyboard? 
    // Nope! https://stackoverflow.com/questions/4745988/                                            
    val globalLayoutListener = ViewTreeObserver.OnGlobalLayoutListener {

      val rect = Rect().apply { rootView.getWindowVisibleDisplayFrame(this) }

      val screenHeight = rootView.height

      // rect.bottom is the position above soft keypad or device button.
      // if keypad is shown, the rect.bottom is smaller than that before.
      val keypadHeight = screenHeight - rect.bottom

      // 0.15 ratio is perhaps enough to determine keypad height.
      if (keypadHeight > screenHeight  0.15) {
        emitter.onNext(KeyboardStatus.OPEN)
      } else {
        emitter.onNext(KeyboardStatus.CLOSED)
      }
    }

    rootView.viewTreeObserver.addOnGlobalLayoutListener(globalLayoutListener)

    emitter.setCancellable {
      rootView.viewTreeObserver.removeOnGlobalLayoutListener(globalLayoutListener)
    }
                                                                                
  }.distinctUntilChanged()
}

enum class KeyboardStatus {
  OPEN, CLOSED
}
```

这就是当键盘打开的时候，你如何使用它来隐藏一个横幅。 

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
  
    keyboardManager.status()
        .to(ObservableScoper(this)) // AutoDispose https://github.com/uber/AutoDispose
        .subscribe { showAnnoyingBanner(visible = KeyboardStatus.CLOSED == it) }
}
```

 stack overflow 更多[讨论](https://stackoverflow.com/questions/5105354) 。

