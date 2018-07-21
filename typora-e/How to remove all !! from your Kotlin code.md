# 如何移除所有 !! , 从你的 Kotlin 代码

> 原文 (Medium) ：[How to remove all !! from your Kotlin code](https://android.jlelse.eu/how-to-remove-all-from-your-kotlin-code-87dc2c9767fb)
>
> 作者 ：[David Vávra](https://android.jlelse.eu/@david.vavra?source=post_header_lockup)

空安全是 Kotlin 最好的特征之一。 它使你在语言层面上考虑到了可空性，这样你就可以避免在 Java 中很常见的许多隐藏的 NullPointerExceptions。 然而，当你自动将 Java 代码转换成 Kotlin 时，你可以看到很多 !! 符号。 看起来你不应该有 !! ，除非是一个快速的原型。 我相信这是真的，因为！ 基本上意思是"这里有一个潜在的未处理的 KotlinNullPointerException"。 再加上它看起来很老套。

Kotlin 有一些聪明的机制可以避免这种情况，但是弄清楚它们并不容易。 这里有6种方法可以做到:

### 1)使用 val 而不是 var

Kotlin 让你在语言层面上思考不可变性，这很好。val 是只读的 ，var 是可变的。 建议尽可能多地使用只读属性。 它们是线程安全的，在函数式编程方面工作非常出色。 如果你把它们当成不可变的，你就不需要关心可空性。 只是要小心，[val 实际上是可以变换的](http://blog.danlew.net/2017/05/30/mutable-vals-in-kotlin)。

### 2)用 lateinit

有时候你不能使用不可变的属性。 例如，它发生在 Android 上，当一些属性在 create ( ) 调用中被初始化时。 对于这些情况，Kotlin 有一个叫做 lateinit 的语言功能。

它可以让你取代这个：

```kotlin
private var mAdapter: RecyclerAdapter<Transaction>? = null

override fun onCreate(savedInstanceState: Bundle?) {
   super.onCreate(savedInstanceState)
   mAdapter = RecyclerAdapter(R.layout.item_transaction)
}

fun updateTransactions() {
   mAdapter!!.notifyDataSetChanged()
}
```

有了这个:

```kotlin
private lateinit var mAdapter: RecyclerAdapter<Transaction>

override fun onCreate(savedInstanceState: Bundle?) {
   super.onCreate(savedInstanceState)
   mAdapter = RecyclerAdapter(R.layout.item_transaction)
}

fun updateTransactions() {
   mAdapter.notifyDataSetChanged()
}
```

请注意，访问未初始化的 lateinit 属性将导致 UninitializedPropertyAccessException。

可悲的是，lateinit 不适用于像 Int 这样的原始数据类型。 对于原始类型，你可以使用这样的委托:

```kotlin
private var mNumber: Int by Delegates.notNull<Int>()
```

### 3)使用 let 函数

这是 Kotlin 代码中常见的编译时错误:

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvfv0wywj30ic03mjre.jpg)

这让我很恼火: 我知道在 null 检查之后，这个可变的属性不可能被改变。 许多开发人员用以下方法快速修复它:

```kotlin
private var mPhotoUrl: String? = null

fun uploadClicked() {
    if (mPhotoUrl != null) {
        uploadPhoto(mPhotoUrl!!)
    }
}
```

但是使用 let 函数有一个优雅的解决方案:

```kotlin
private var mPhotoUrl: String? = null

    fun uploadClicked() {
        mPhotoUrl?.let { uploadPhoto(it) }
    }
```

### 4)创建全局函数来处理更复杂的情况

对于一个简单的 null 检查来说，let 是个很好的替代品，但是可能还有更复杂的情况。 例如:

```kotlin
if (mUserName != null && mPhotoUrl != null) {
   uploadPhoto(mUserName!!, mPhotoUrl!!)
}
```

你可以嵌入两个 let 调用，但是这不是非常可读的。 在 Kotlin，你可以拥有全局可访问的函数，这样你就可以轻松地构建一个你需要的函数，这个函数可以这样使用:

```kotlin
ifNotNull(mUserName, mPhotoUrl) {
   userName, photoUrl ->
   uploadPhoto(userName, photoUrl)
}
```

此函数的代码:

```kotlin
fun <T1, T2> ifNotNull(value1: T1?, value2: T2?, bothNotNull: (T1, T2) -> (Unit)) {
   if (value1 != null && value2 != null) {
       bothNotNull(value1, value2)
   }
}
```

### 5)使用 Elvis 操作符

当你对 null 的情况有一个回退值时，Elvis 运算符是很棒的。 所以你可以替换这个:

```kotlin
fun getUserName(): String {
   if (mUserName != null) {
       return mUserName!!
   } else {
       return "Anonymous"
   }
}
```

有了这个:

```kotlin
fun getUserName(): String {
   return mUserName ?: "Anonymous"
}
```

### 6)按照自己的方式崩溃

仍然有一些情况，当你知道某个东西不可能为空时，即使类型必须是可空的。 如果它是 null，它是代码中的一个错误，你应该知道它。 然而，离开! ！ 这里给出了一个通用的 KotlinNullPointerException，这是很难调试的。 使用内置函数 requireNotNull 或 checkNotNull，并附带异常消息以便于调试。

取而代之:

```kotlin
uploadPhoto(intent.getStringExtra("PHOTO_URL")!!)
```

有了这个:

```kotlin
uploadPhoto(requireNotNull(intent.getStringExtra("PHOTO_URL"), { "Activity parameter 'PHOTO_URL' is missing" }))
```

### 结论

如果你遵循这6条建议，你可以删除所有!!， 来自你的 Kotlin 代码。 你的代码将更加安全，更加可调试和更简洁。 如果你知道更多处理空ff安全的方法，请在评论中告诉我。

