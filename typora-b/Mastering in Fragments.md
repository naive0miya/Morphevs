# 掌握 Fragments

> 原文 (Medium)：[Mastering in Fragments](https://medium.com/@elifbon/mastering-in-fragments-2430c4bf6441)
>
> 作者：[Elif Boncuk](https://medium.com/@elifbon?source=post_header_lockup)

[TOC]

我是 Fragments 的早期用户之一。既然已经有一段时间了，但我仍然在学习新的东西。

片段的重要性在于，如果你没有任何实时错误，你应该了解它的每个细节。它与 Activity 非常相似，但也完全不同。我猜这是家里最疯狂的男孩。

我试着分享一些我最好的。

- 不要只相信 setUserVisibilityHint 方法，保证你的安全!  setUserVisilibityHint 方法看起来非常有用，给出了片段的可见性。所以如果你正在使用一个 viewpager 并且需要一些东西的时候，它似乎是很方便的使用。没那么简单！ 首先要知道的是，这个方法可以被称为与片段生命周期无关的。我的意思是，是的，你的片段是可见的，但不是布局 inflated  或者 attached 活动。所以我们应该像下面检查一样，检查是否并行恢复。

  ![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowa3wli9j30fp096jru.jpg)

  另一个令人惊讶的是可见性的默认值是 true 的。要发现这种情况非常罕见，但是如果你将片段编号/类型从 remoteconfig 类似的东西更改，并在运行时更改，而用户使用当前页面，那么因为第一个值是 true 的，而它实际上是不可见的，它将返回一个标志，就好像它是可见的。另一方面, 上述解决方案也会有所帮助 。我还是不明白为什么默认值是 true 的。 


- 不要忘记，当你从活动中分离时，片段仍然存在！如果你在活动中使用片段并且可以保留在堆栈中，那么这些片段将移动到活动中，但不会附加到你的活动中。例如，你有一个带有 viewpager 的片段，并在其上打开另一个活动。当第二个活动被打开后，片段将从你的活动中分离出来。那么会发生什么？

  如果你实现了一些侦听器方法，启动了异步任务，已经使用了像 picasso 这样的第三方库，那么你在 onStop ( ) 并没有取消注册/停止它们，最后如果它们使用上下文（getActivity ( )），那就是绝对崩溃。

  我们如何防止我们的应用程序崩溃？ 如果有异步任务，则应在 onStop ( ) 取消。如果你注册了任何侦听器，你应该在 onResume ( ) 上注册并在 onStop ( ) 上取消注册。如果你使用的是 picasso（我已添加，因为很常见），请取消你的请求。

  记住，Fragment 和 Activity 一样有生命周期方法，但是当它们调用时并不与附加活动的同时。

  ```java
  Picasso.with(context).cancelRequest(YourImageView);
  ```

- 不要使用片段方法为你的接口方法提供相同的方法名称！那么，这不仅在 evert native 组件的片段上是错误的。可悲的是，如果你使用它，你将无法看到它为什么不能在混淆 apk 的时候在你的本地工作。

- 找到一种检查和缓存片段的方法 以下代码显示了我们如何在使用 viewpager 时重新使用我们的片段。

```java
FragmentX fragmentX = null;
FragmentY fragmentY = null; 
@Override
public Fragment getItem(int position) {
    if(position == 0){
        if(fragmentX == null) {
            fragmentX = FragmentX.newInstance();
        }
        return fragmentX;
    }else {
        if(fragmentY == null) {
            fragmentY = FragmentY.newInstance();
        }
        return fragmentY;
    }
}

@Override
public Object instantiateItem(ViewGroup container, int position) {
    Object ret = super.instantiateItem(container, position);
    if(position == 0){
        fragmentX = (FragmentX) ret;
        return fragmentX;
    }else {
        fragmentY = (FragmentY) ret;
        return fragmentY;
    }
}
```

我最喜欢的片段是这样的，我知道随着时间的推移，这个列表会继续下去。

如果你有类似的故事，请随时联系我或发表评论。

谢谢你的阅读！ 

