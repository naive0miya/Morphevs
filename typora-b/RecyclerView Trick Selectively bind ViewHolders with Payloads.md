# RecyclerView 技巧: 绑定 ViewHolders 时选择性地使用 Payloads

> 原文 (Medium)：[RecyclerView Trick: Selectively bind ViewHolders with Payloads](https://medium.com/livefront/recyclerview-trick-selectively-bind-viewholders-with-payloads-4b28e3d2cce8)
>
> 作者：[Andrew Haisting](https://medium.com/@ahaisting?source=post_header_lockup)

[TOC]

> 如果你想更新一个 RecyclerView 但不想重新绑定整个视图吗？ 实际上有一个 notifyItemChanged 的变体，让我们可以做到这一点：

## 以前:

考虑下面这个例子， RecyclerView 项目的点击事件：

```java
override fun onImageClick(item: ImageItem) {
  item.isFavorited = !item.isFavorited
  recycler.adapter.notifyItemChanged(items.indexOf(item))
}
```

在调用 notifyItemChanged 之后，适配器会被通知，然后调用 onBindViewHolder。不幸的是，重新绑定整个视图可能会导致一些视觉闪烁：

![](https://cdn-images-1.medium.com/max/800/1*-XkJjrDFYKYvf7Emq-M9IQ.gif)

如果我们只想更新实际上改变了的部分视图会怎么样？ 当我们通知适配器时，我们可以传递一个"payload"对象: 

```java
override fun onImageClick(item: ImageItem) {
    item.isFavorited = !item.isFavorited
    recycler.adapter.notifyItemChanged(items.indexOf(item), 
                                       ImageAdapter.Payload.FAVORITE_CHANGE)
    // We'll check for Payload.FAVORITE_CHANGE in our adapter
}
```

然后我们在 adapter 中添加逻辑来处理这些特殊的"payload": 

```java
override fun onBindViewHolder(holder: ViewHolder,
                              position: Int,
                              payloads: MutableList<Any>) {
    if (!payloads.isEmpty()) {
        // Because these updates can be batched,
        // there can be multiple payloads for a single bind
        when (payloads[0]) {
            Payload.FAVORITE_CHANGE -> {
                // Change only the "favorite" icon,
                // leave background image alone:
                bindFavoriteIcon(holder,
                        items[position].isFavorited)
            }
        }
    }
    // When payload list is empty,
    // or we don't have logic to handle a given type,
    // default to full bind:
    super.onBindViewHolder(holder, position, payloads)
}
```

## 之后:

![](https://cdn-images-1.medium.com/max/800/1*kHqxP5jYb30s3ZGUM7VpDQ.gif)

## 这个策略有几个优点：

1. 我们将所有的视图都保留在我们的 adapter 类。
2. 我们有关于如何绑定的灵活选项：只想绑定视图的某些部分？当然可以。想要动画来显示更改，但仅限于某些更新？当然可以。想要将多个调用链接到 notifyItemChanged 以反映多个更新？当然可以。 onBindViewHolder 会给你一个完整的 List \<Object>，它将你的所有 payloads 合并在一起。
3. 我们总是可以回到默认更新; 任何时候适配器不知道如何处理某个 payloads，视图就会完全反弹 

背景闪烁实际上是预期的行为，并且由于 DefaultItemAnimator 的“更改”动画的实现而发生。调用 [setSupportChangeAnimations（false）](https://developer.android.com/reference/android/support/v7/widget/SimpleItemAnimator.html#setSupportsChangeAnimations%28boolean%29)也会“修复”背景闪烁。然而，这个解决方案有点笨重。在 ItemAnimator 上禁用变化动画将停止所有动画时，我们往往只是想禁用一些动画。 

Happy recycling!

