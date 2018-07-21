# 我们为什么要用 fitsSystemWindows?

> 原文 (Medium)：[Why would I want to fitsSystemWindows?](https://medium.com/google-developers/why-would-i-want-to-fitssystemwindows-4e26d9ce1eec)
>
> 译文：[我们为什么要用fitsSystemWindows?](https://github.com/hehonghui/android-tech-frontier/blob/master/issue-35/%E4%B8%BA%E4%BB%80%E4%B9%88%E6%88%91%E4%BB%AC%E8%A6%81%E7%94%A8fitsSystemWindows.md)
>
> 作者：[Ian Lake](https://medium.com/@ianhlake?source=post_header_lockup)
>
> 译者：[LionelCursor](https://github.com/LionelCursor)



[TOC]

System windows 指的就是屏幕上 status bar、 navigation bar 等系统控件所占据的部分。

绝大多数情况下，你都不需要理会 status bar 或者 navigation bar 下面的空间，不过你需要注意不能让你的交互控件（比如 Button）藏在 status bar 或者 navigation bar 下面。而 `android:fitsSystemWindows="true"` 的默认行为正好解决了这种情况，这个属性的作用就是通过设置 View 的 padding，使得应用的 content 部分—— Activity 中 setContentView( ) 中传入的就是 content ——不会与 system window 重叠。

还有一些事情需要注意：

- fitsSystemWindows 需要被设置给根 View——这个属性可以被设置给任意 View，但是只有根 View（content 部分的根）外面才是 SystemWindow，所以只有设置给根 View 才有用。
- Insets始终相对于全屏幕——`Insets` 即边框，它决定着整个 Window 的边界。对 `Insets` 设置 padding 的时候，这个 padding 始终是相对于全屏幕的。因为 `Insets` 的生成在 View `layout` 之前就已经完成了，所以系统对于 View 长什么样一无所知。
- 其它padding将通通被覆盖。需要注意，如果你对一个 View 设置了 `android:fitsSystemWindows="true"`，那么你对该 View 设置的其他 padding 将通通无效。

在绝大多数情况下，默认情况就已经够用了。比如一个全屏的视屏播放器。如果你不想被 ActionBar 或者其他 System View 遮住的话，那么在 MatchParent 的 ViewGroup 上设置 `fitsSystemWindows="true"` 即可。

或者，也许你希望你的 RecyclerView 能够在透明的 navigation bar 下面滚动。那么只需将 `android:fitsSystemWindows="true"` `android:clipToPadding="false"` 同时使用即可, 滚动的内容会绘制在 navigation bar下面，同时当滚动到最下面的时候，最后一个 item下面依旧会有 padding，使其可以滚到 navigation bar上方（而不是在 navigation bar下面滚不上来！）。

译者注：`clipToPadding` 是 `ViewGroup` 的属性。这个属性定义了是否允许 ViewGroup 在 padding 中绘制,该值默认为 true, 即不允许。

## 自定义 fitsSystemWindows

但是默认毕竟只是默认。 在KITKAT及以下的版本，你的自定义View能够通过覆盖 `fitsSystemWindows() : boolean` 函数，来增加自定义行为。如果返回 `true`，意味着你已经占据了整个 `Insets`，如果返回 `false`，意味着给其他的View依旧留有机会。

而在Lollipop 以及更高的版本，我们提供了一些新的API，使得自定义这个行为更加的方便。与之前的版本不同，现在你只需要覆盖 `OnApplyWindowInsets()` 方法，该方法允许 View 消耗它想消耗的任意空间 （Insets），同时也能够为子方法，调用 `dispatchApplyWindowInsets()`

更妙的是，利用新的 API，你甚至不需要拓展 View 类，你可以使用 `ViewCompat.setOnApplyWindowInsetsListener()`，这个方法优先于 `View.onApplyWindowInsets()` 调用。`ViewCompat` 同时也提供了 `onApplyWindowInsets()` 和 `dispatchApplyWindowInsets()` 的兼容版本，无需冗长的版本判断。

## 自定义 fitsSystemWindows 例子

绝大多数基本的 layouts（FrameLayout）都是使用默认的行为，然而依然有一部分 layouts 已经使用了自定义 `fitsSystemWindow` 来实现自定义的功能。

`navigation drawer` 就是一个例子，它需要充满整个屏幕，绘制在透明的 status bar 下面。

[![enter image description here](https://ws2.sinaimg.cn/large/006tKfTcgy1frovdebpgtj30k00fkwmf.jpg)](https://camo.githubusercontent.com/c6f6c99ab438502ae8af6acebabb4df45e2f7dc1/687474703a2f2f376f746872752e636f6d312e7a302e676c622e636c6f7564646e2e636f6d2f7768792d776f756c642d692d77616e742d746f2d6669747373797374656d77696e646f77732e706e67)

如上图所示，`DrawerLayout` 使用了 `fitsSystemwindows`，他需要让它的子 View 依旧保持默认行为，即不被 actionbar 或其他 system window 遮住，同时依照 Material Design 的定义，又需要在透明的 statusbar下面进行绘制（默认是你在 theme 中设置的 `colorPrimaryDark` 颜色）

你会注意到，在Lollipop及以上版本，`DrawerLayout` 为每一个子View调用了 `dispatchApplyWindowInsets()`，使每一个子View都收到 `fitsSystemWindows`。这与默认行为完全不同，默认行为会使得根 View 消耗所有的 insets，同时子View们永远不会收到 `fitsSystemWindows`。

 `CoordinatorLayout` 也利用了这一特性，使得其子View有机会截断并对 `fitsSystemWindows` 做出自己的反应。同时，它也利用 `fitsSystemWindows` 这一 flag 看其是否需要在 statusbar 的下方绘制。

同样的，`CollapsingToolbarLayout` 以 `fitsSystemWindows` 什么时候把变小的 View 放在什么地方。

如果你对这些 [`Design Library`](http://android-developers.blogspot.com/2015/05/android-design-support-library.html) 里的东西感兴趣，请查看 [Cheesesquare Sample](https://github.com/chrisbanes/cheesesquare)

## 积极使用系统，而不是老想着 Hack

有一件事需要始终牢记，这个属性毕竟不是 `fitsStatusBar` 或者 `fitsNavigationBar`。不管是尺寸还是位置，在不同版本间，系统控件都有很大的差距。

但是尽管放心，无论在什么平台上，`fitsSystemWindows` 都会影响 `Insets`，使你的 content 和 system ui 不会重叠——除非你自定义这一行为。

