# 现代 Android Kotlin 开发 - 第2部分

> 原文 (Medium)：[Modern Android development with Kotlin (Part 2)](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-2-17444fcdbe86)
>
> 作者：[Mladen Rakonjac](https://proandroiddev.com/@mladenrakonjac?source=post_header_lockup)

[TOC]

0. Android Studio 3, beta 1 [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)


1. Kotlin language [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)


2. Build Variants [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)


3. ConstraintLayout [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)


4. Data binding library [Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)

5. MVVM architecture + repository pattern + Android Manager Wrappers 

6. RxJava2 and how it helps us in architecture 


7. Dagger 2.11, what is Dependency Injection, why you should use it. 


8. Retrofit (with Rx Java2)
9. Room (with Rx Java2)

## 5. MVVM architecture + repository pattern + Android Manager Wrappers

### 关于 Android 世界中的架构

很长一段时间，Android 开发者在他们的项目中没有任何架构。 在过去的三年里，架构在 Android 开发者群体中得到了大量的宣传。 上帝活动的时代已经过去，谷歌发布了安卓建筑蓝图库，里面有许多关于不同建筑方法的示例和说明。 最后，在 Google IO'17上，他们引入了 [Android 架构组件](https://developer.android.com/topic/libraries/architecture/index.html)，这是一个帮助我们拥有更清洁的代码和更好的应用程序的库。 组件描述你可以使用所有这些或只是其中的一些。 尽管如此，我发现它们都非常有用。 在下面的文本和下面的部分中，我们将使用这些组件。 首先，我将解决这个问题，然后使用这些组件和库重构代码，看看这些库将解决哪些问题。 

有两种主要的[架构模式](https://en.wikipedia.org/wiki/Architectural_pattern)可以分离 GUI 代码: 

- MVP
- MVVM

很难说什么是更好的。 你应该两者都尝试一下，然后再做决定。 我更喜欢使用生命周期感知组件的 MVVM，并将其写入。 如果你从来没有尝试过 MVP，那么在 Medium 上有很多很好的文章。 

### 什么是 MVVM 模式？

MVVM 模式是一种架构模式。 它代表 Model-View-ViewModel。 我觉得这个名字让开发者感到困惑。 如果我是命名它的人，我会给它一个 View-ViewModel-Model 名称，因为 ViewModel 是连接 View 和 Model 的中间的一个。

视图 是活动、片段或任何其他 Android 自定义视图的抽象名称。 请注意，不要将这个视图放在 Android 视图中是非常重要的。 视图应该是愚蠢的，我们不应该在我们的视图中写任何逻辑。 视图不应保存任何数据。 它应该有 ViewModel 实例引用，所有需要的数据都应该从它来看。 此外，当 ViewModel 的数据被更改时，应该对这些数据和布局进行观察。 总之，View 有一个责任: 布局如何寻找不同的数据和状态。 

视图模型 是一个抽象的类的名称，它包含数据，并且对于数据何时被提取以及数据何时提交有逻辑。 视图模型保持当前状态。 此外，ViewModel 还引用了一个或多个 Model 实例，所有数据都应该从这些实例中获取。 例如，它不应该知道来自数据库或远程服务器的数据。 此外，ViewModel 根本不应该知道这个视图。 此外，ViewModel 根本不应该知道任何关于 Android 框架的内容。 

模型 是我们为 ViewModel 准备数据的层的抽象名称。 在这个类中，我们将从远程服务器获取数据，并将其缓存在内存中或保存在本地数据库中。 请注意，它与那些不同: 用户、汽车、方形等模型类只是保存数据。 通常，它是一个存储库模式的实现，我们将在下面的文本中涵盖这个模式。 模型不应该知道 ViewModel。 

如果实现正确，MVVM 是一个很好的方法来分离你的代码，使它更加可测试。 它帮助我们遵循 [SOLID](https://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29) 原则，这样我们的代码更容易维护。 

代码示例

我将写一个最简单的例子来解释它是如何工作的。首先，让我们做一个简单的模型，这样可以返回一些字符串: 

```kotlin
class RepoModel {

    fun refreshData() : String {
        return "Some new data"
    }
}
```

通常，获取数据是异步调用，所以我们必须等待它。 为了模拟它，我将类改为如下: 

```kotlin
class RepoModel {

    fun refreshData(onDataReadyCallback: OnDataReadyCallback) {
        Handler().postDelayed({ onDataReadyCallback.onDataReady("new data") },2000)
    }
}

interface OnDataReadyCallback {
    fun onDataReady(data : String)
}
```

首先，我已经创建了 onDataReady 函数的 OnDataReadyCallback 接口。 现在，我们的 refreshData 函数将实现 OnDataReadyCallback 。 为了模拟等待，我使用了 Handler。 一旦2秒钟过去 ，onDataReady 函数将调用 OnDataReadyCallback 的实现。 

现在让我们来做一个 ViewModel 

```java
class MainViewModel {
    var repoModel: RepoModel = RepoModel()
    var text: String = ""
    var isLoading: Boolean = false
}
```

正如你所看到的，有一个 RepoModel 的实例，我们将要显示的文本是 isLoading 布尔值来保存当前状态。现在，我们来做一个刷新函数，负责获取数据: 

```java
class MainViewModel {
    ...

    val onDataReadyCallback = object : OnDataReadyCallback {
        override fun onDataReady(data: String) {
            isLoading.set(false)
            text.set(data)
        }
    }

    fun refresh(){
        isLoading.set(true)
        repoModel.refreshData(onDataReadyCallback)
    }
}
```

refresh 函数使得一个 refreshData  调用例子的 RepoModel 需要实现 OnDataReadyCallback。 好吧，但是对象是什么？ 无论何时，当你希望实现某些接口或扩展某个类而不创建子类时，将使用对象声明。 如果你想使用它作为一个匿名类呢？ 在这种情况下，你必须使用对象表达式: 

```kotlin
class MainViewModel {
    var repoModel: RepoModel = RepoModel()
    var text: String = ""
    var isLoading: Boolean = false

    fun refresh() {
        repoModel.refreshData( object : OnDataReadyCallback {
        override fun onDataReady(data: String) {
            text = data
        })
    }
}
```

当我们调用刷新时，我们应该将视图更改为加载状态，一旦数据出现，我们应该设置 isLoading 为 false。 

此外，我们应该将文本更改为 ObservableField\<String>  和 isLoading 更改为 ObservableField\<Boolean> 。 ObservableField 是一个来自数据绑定库的类，我们可以使用，而不是创建一个可观察的对象。 它包裹了我们希望被观察到的物体。 

```kotlin
class MainViewModel {
    var repoModel: RepoModel = RepoModel()

    val text = ObservableField<String>()

    val isLoading = ObservableField<Boolean>()

    fun refresh(){
        isLoading.set(true)
        repoModel.refreshData(object : OnDataReadyCallback {
            override fun onDataReady(data: String) {
                isLoading.set(false)
                text.set(data)
            }
        })
    }
}
```

请注意，我使用 val 而不是 var，因为我们只会在字段中改变值，而不是字段本身。 如果你想初始化它，你应该做: 

```kotlin
 val text = ObservableField("old data")
 val isLoading = ObservableField(false)
```

让我们改变我们的布局，以便它可以观察文本和 isLoading。 首先，我们将绑定 MainViewModel 而不是 Repository: 

```xml
<data>
    <variable
        name="viewModel"
        type="me.fleka.modernandroidapp.MainViewModel" />
</data>
```

那么，让我们：

- 修改 TextView 以观察来自 MainViewModel 实例的文本 
- 添加 ProgressBar，只有当 isLoading 是 true 的时候才能被看到 
- 添加按钮，点击后将调用来自 MainViewModel 实例的刷新函数，并且只有当 isLoading 是 false 的时才可以点击 

```xml
...

        <TextView
            android:id="@+id/repository_name"
            android:text="@{viewModel.text}"
            ...
            />

        ...
        <ProgressBar
            android:id="@+id/loading"
            android:visibility="@{viewModel.isLoading ? View.VISIBLE : View.GONE}"
            ...
            />

        <Button
            android:id="@+id/refresh_button"
            android:onClick="@{() -> viewModel.refresh()}"
            android:clickable="@{viewModel.isLoading ? false : true}"
            />
...
```

如果你运行此操作，你将得到错误因为 View.VISIBLE 和 View.GONE 不能使用，因为 View 还没有导入。 因此，我们必须导入它: 

```xml
<data>
        <import type="android.view.View"/>

        <variable
            name="viewModel"
            type="me.fleka.modernandroidapp.MainViewModel" />
</data>
```

好了，这就是布局。 现在我们必须完成绑定。 正如我们所说，View 应该有 ViewModel 的实例: 

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var binding: ActivityMainBinding

    var mainViewModel = MainViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        binding.viewModel = mainViewModel
        binding.executePendingBindings()

    }
}
```

最后，我们可以运行它：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frov8uorb4g30dc08kmys.gif)

你可以看到旧数据更改为新数据。

这是最简单的 MVVM 示例。 有一个关于这个问题，现在让我们旋转手机：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frov8zvhveg30dc09agot.gif)

新数据返回为旧数据。这怎么可能？看看 Activity 的生命周期：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frov94kctjj30e90if0u5.jpg)

当你旋转屏幕时，创建了 Activity 的新实例，并调用 onCreate（）方法。现在，看看我们的活动：

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var binding: ActivityMainBinding

    var mainViewModel = MainViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        binding.viewModel = mainViewModel
        binding.executePendingBindings()

    }
}
```

正如你所看到的，当一个新的 Activity 实例被创建时，MainViewModel 的一个实例也被创建。如果我们可以为每个重新创建的 MainActivity 拥有相同的 MainViewModel 的实例，那该多好啊？ 

### 引入生命周期感知组件

因为很多开发者面临这个问题，Android 框架团队的开发人员决定建立一个帮助我们解决这个问题的库。 类是其中之一。 它是我们所有 ViewModels 应该扩展的类。 

让我们使我们的 MainViewModel 从生命周期感知组件扩展 ViewModel。 首先，我们应该在我们的 build.gradle 文件中添加生命周期感知组件库：

```groovy
dependencies {
    ... 

    implementation "android.arch.lifecycle:runtime:1.0.0-alpha9"
    implementation "android.arch.lifecycle:extensions:1.0.0-alpha9"
    kapt "android.arch.lifecycle:compiler:1.0.0-alpha9"
}
```

现在，让我们的 MainViewModel 扩展 ViewModel：

```kotlin
package me.fleka.modernandroidapp

import android.arch.lifecycle.ViewModel

class MainViewModel : ViewModel() {
    ...
}
```

而在 MainActivity 的 onCreate ( ) 函数中，你应该有：

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var binding: ActivityMainBinding
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        binding.viewModel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        binding.executePendingBindings()

    }
}
```

请注意，我们不是在做 MainViewModel 的新实例。 现在，我们从 ViewModelProviders 获取。 ViewModelProviders 是一个实用工具类，它有获取 ViewModelProvider 的方法。 这完全是范围的问题。 因此，如果你在活动中调用 ViewModelProviders.of (this) ，那么 ViewModel 会一直存在，只要活动是活的(被破坏而不是重建)。 因此，如果你在片段中调用它，那么 ViewModel 就会一直存在，只要那个片段还活着，等等。 看看这个图表: 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frov9a7r3gj30ei0f3wev.jpg)

如果你的 activity / fragment 被重新创建，那么 ViewModelProvider 将负责创建新的实例或者返回旧实例。 

不要与以下内容混淆：

```kotlin
MainViewModel::class.java
```

在 Kotlin，如果你只需这样做：

```kotlin
MainViewModel::class
```

它会返回给你一个 [KClass](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/index.html) ，它和 Java 中的 Class 不一样。所以，如果我们这样做：.java 通过它的文档：

> 返回与给定的 KClass 实例相对应的 Java [类](http://docs.oracle.com/javase/8/docs/api/java/lang/Class.html)实例。

现在让我们看看如果我们旋转屏幕会发生什么事情：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frov9glfabg30dc084ah0.gif)

我们有和旋转之前一样的数据。

在上一篇文章中，我说我们的应用程序将获取 Github Repository 列表并显示它。为了做到这一点，我们必须添加 getRepositories 函数，该函数将返回存储库的伪造列表：

```kotlin
class RepoModel {

    fun refreshData(onDataReadyCallback: OnDataReadyCallback) {
        Handler().postDelayed({ onDataReadyCallback.onDataReady("new data") },2000)
    }

    fun getRepositories(onRepositoryReadyCallback: OnRepositoryReadyCallback) {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First", "Owner 1", 100 , false))
        arrayList.add(Repository("Second", "Owner 2", 30 , true))
        arrayList.add(Repository("Third", "Owner 3", 430 , false))
        
        Handler().postDelayed({ onRepositoryReadyCallback.onDataReady(arrayList) },2000)
    }
}

interface OnDataReadyCallback {
    fun onDataReady(data : String)
}

interface OnRepositoryReadyCallback {
    fun onDataReady(data : ArrayList<Repository>)
}
```

另外，我们应该在我们的 MainViewModel 中有一个从 RepoModel 调用 getRepositories 的函数：

```kotlin
class MainViewModel : ViewModel() {
    ...
    var repositories = ArrayList<Repository>()

    fun refresh(){
        ...
    }

    fun loadRepositories(){
        isLoading.set(true)
        repoModel.getRepositories(object : OnRepositoryReadyCallback{
            override fun onDataReady(data: ArrayList<Repository>) {
                isLoading.set(false)
                repositories = data
            }
        })
    }
}
```

最后，我们应该在 RecyclerView 中显示这些存储库。要做到这一点，我们必须：

- 创建 rv_item_repository.xml 布局
- 在我们的 activity_main.xml 布局中添加 RecyclerView
- 创建 RepositoryRecyclerViewAdapter
- 将适配器设置给 recyclerview

为了创建 rv_item_repository.xml 我使用了 CardView 库，所以我们需要将它添加到 build.gradle（app）中：

```groovy
implementation 'com.android.support:cardview-v7:26.0.1'
```

看起来是这样的 ：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <import type="android.view.View" />

        <variable
            name="repository"
            type="me.fleka.modernandroidapp.uimodels.Repository" />
    </data>

    <android.support.v7.widget.CardView
        android:layout_width="match_parent"
        android:layout_height="96dp"
        android:layout_margin="8dp">

        <android.support.constraint.ConstraintLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <TextView
                android:id="@+id/repository_name"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="16dp"
                android:layout_marginStart="16dp"
                android:text="@{repository.repositoryName}"
                android:textSize="20sp"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintHorizontal_bias="0.0"
                app:layout_constraintLeft_toLeftOf="parent"
                app:layout_constraintRight_toRightOf="parent"
                app:layout_constraintTop_toTopOf="parent"
                app:layout_constraintVertical_bias="0.083"
                tools:text="Modern Android App" />

            <TextView
                android:id="@+id/repository_has_issues"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="16dp"
                android:layout_marginStart="16dp"
                android:layout_marginTop="8dp"
                android:text="@string/has_issues"
                android:textStyle="bold"
                android:visibility="@{repository.hasIssues ? View.VISIBLE : View.GONE}"
                app:layout_constraintBottom_toBottomOf="@+id/repository_name"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintHorizontal_bias="1.0"
                app:layout_constraintStart_toEndOf="@+id/repository_name"
                app:layout_constraintTop_toTopOf="@+id/repository_name"
                app:layout_constraintVertical_bias="1.0" />

            <TextView
                android:id="@+id/repository_owner"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_marginBottom="8dp"
                android:layout_marginEnd="16dp"
                android:layout_marginStart="16dp"
                android:text="@{repository.repositoryOwner}"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@+id/repository_name"
                app:layout_constraintVertical_bias="0.0"
                tools:text="Mladen Rakonjac" />

            <TextView
                android:id="@+id/number_of_starts"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginBottom="8dp"
                android:layout_marginEnd="16dp"
                android:layout_marginStart="16dp"
                android:layout_marginTop="8dp"
                android:text="@{String.valueOf(repository.numberOfStars)}"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintHorizontal_bias="1"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@+id/repository_owner"
                app:layout_constraintVertical_bias="0.0"
                tools:text="0 stars" />

        </android.support.constraint.ConstraintLayout>

    </android.support.v7.widget.CardView>

</layout>
```

下一步是在 activity_main.xml 中添加 RecyclerView。在做之前，不要忘记添加 RecyclerView 库：

```groovy
implementation 'com.android.support:recyclerview-v7:26.0.1'
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <import type="android.view.View"/>

        <variable
            name="viewModel"
            type="me.fleka.modernandroidapp.MainViewModel" />
    </data>

    <android.support.constraint.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context="me.fleka.modernandroidapp.MainActivity">

        <ProgressBar
            android:id="@+id/loading"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:indeterminate="true"
            android:visibility="@{viewModel.isLoading ? View.VISIBLE : View.GONE}"
            app:layout_constraintBottom_toTopOf="@+id/refresh_button"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <android.support.v7.widget.RecyclerView
            android:id="@+id/repository_rv"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:indeterminate="true"
            android:visibility="@{viewModel.isLoading ? View.GONE : View.VISIBLE}"
            app:layout_constraintBottom_toTopOf="@+id/refresh_button"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            tools:listitem="@layout/rv_item_repository" />



        <Button
            android:id="@+id/refresh_button"
            android:layout_width="160dp"
            android:layout_height="40dp"
            android:layout_marginBottom="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginStart="8dp"
            android:layout_marginTop="8dp"
            android:onClick="@{() -> viewModel.loadRepositories()}"
            android:clickable="@{viewModel.isLoading ? false : true}"
            android:text="Refresh"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintVertical_bias="1.0" />

    </android.support.constraint.ConstraintLayout>

</layout>
```

请注意，我们从前面删除了一些 TextView 元素，现在按钮触发 loadRepositories 而不是刷新：

```xml
 <Button
    android:id="@+id/refresh_button"
    android:onClick="@{() -> viewModel.loadRepositories()}" 
    ...
    />
```

让我们删除 MainViewModel 中的刷新函数和 RepoModel 中的 refreshData 函数，因为我们不再需要它们。

现在，我们应该为我们的 RecyclerView 添加适配器：

```kotlin
class RepositoryRecyclerViewAdapter(private var items: ArrayList<Repository>,
                                    private var listener: OnItemClickListener)
    : RecyclerView.Adapter<RepositoryRecyclerViewAdapter.ViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup?, viewType: Int): ViewHolder {
        val layoutInflater = LayoutInflater.from(parent?.context)
        val binding = RvItemRepositoryBinding.inflate(layoutInflater, parent, false)
        return ViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int)
            = holder.bind(items[position], listener)

    override fun getItemCount(): Int = items.size

    interface OnItemClickListener {
        fun onItemClick(position: Int)
    }

    class ViewHolder(private var binding: RvItemRepositoryBinding) :
            RecyclerView.ViewHolder(binding.root) {

        fun bind(repo: Repository, listener: OnItemClickListener?) {
            binding.repository = repo
            if (listener != null) {
                binding.root.setOnClickListener({ _ -> listener.onItemClick(layoutPosition) })
            }

            binding.executePendingBindings()
        }
    }

}
```

请注意，ViewHolder 需要 RvItemRepositoryBinding 类型的实例，而不是 View 类型，所以我们可以在 ViewHolder 中为每个项目实现数据绑定。另外，不要被下面的 oneline 函数弄糊涂了：

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int)            = holder.bind(items[position], listener)
```

这只是写的更短的简单方法：

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int){
    return holder.bind(items[position], listener)
}
```

 items[position] 是索引操作符的实现。它和 items.get(position) 一样。

还有一个可能会让你困惑的点是：

```kotlin
binding.root.setOnClickListener({ _ -> listener.onItemClick(layoutPosition) })
```

如果你不使用它，你可以用 `_` 替换参数。不错吧？  :)

我们添加了适配器，但我们仍然没有将其设置给我们的 MainActivity 中的 recyclerView：

```kotlin
class MainActivity : AppCompatActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {

    lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        val viewModel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        binding.viewModel = viewModel
        binding.executePendingBindings()

        binding.repositoryRv.layoutManager = LinearLayoutManager(this)
        binding.repositoryRv.adapter = RepositoryRecyclerViewAdapter(viewModel.repositories, this)

    }

    override fun onItemClick(position: Int) {
        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }
}
```

让我们试试看：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frov9portog30dc09cdl1.gif)

这很奇怪。这里发生了什么？

- 活动已创建，因此将使用实际上为空的存储库创建新的适配器
- 我们点击按钮
- 调用 loadRepositories 函数，显示进度
- 2秒后，我们得到了仓库，进度是隐藏的，但仓库不是。 这是因为 notifyDataSetChanged 不在适配器上调用
- 一旦我们旋转屏幕，创建新的活动，所以创建新的适配器与实际上有一些项目的存储库参数

那么，MainViewModel 应该如何通知 MainActivity 有关新项目的信息，所以我们可以调用 notifyDataSetChanged？

这是不应该的。 

这真的很重要，MainViewModel 根本就不应该知道 MainActivity。 

MainActivity 是具有 MainViewModel 的实例，因此应该监听更改，并通知 Adapter 有关更改的内容。

但是，怎么做呢？ 

我们可以在存储库上观察，所以一旦数据改变，我们可以改变我们的适配器。

这种解决方案有什么问题？

让我们来看看下面这个例子: 

- 在 MainActivity 中，我们观察存储库：一旦发生改变，我们调用 notifyDataSetChanged
- 我们点击按钮
- 在等待数据更改时，由于配置更改，而重新创建 MainActivity。
- 我们的 MainViewModel 还活着。
- 2秒后，存储库字段将获取新项目并通知观察者数据已更改
- 观察者尝试在不存在的适配器上执行 notifyDataSetChanged，因为 MainActivity 被重新创建。

所以，我们的解决方案不够好。

### 介绍 LiveData

LiveData 是另一个生命周期感知组件。基本上可以观察到，它知道视图的生命周期。 因此，一旦由于配置变化而损坏了活动，LiveData 就会知道它，因此它也会破坏的活动观察者。 

让我们在我们的 MainViewModel 中实现它：

```kotlin
class MainViewModel : ViewModel() {
    var repoModel: RepoModel = RepoModel()

    val text = ObservableField("old data")

    val isLoading = ObservableField(false)

    var repositories = MutableLiveData<ArrayList<Repository>>()

    fun loadRepositories() {
        isLoading.set(true)
        repoModel.getRepositories(object : OnRepositoryReadyCallback {
            override fun onDataReady(data: ArrayList<Repository>) {
                isLoading.set(false)
                repositories.value = data
            }
        })
    }
}
```

并留意  MainActivity 中的变化：

```kotlin
class MainActivity : LifecycleActivity(), RepositoryRecyclerViewAdapter.OnItemClickListener {

    private lateinit var binding: ActivityMainBinding
    private val repositoryRecyclerViewAdapter = RepositoryRecyclerViewAdapter(arrayListOf(), this)


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        val viewModel = ViewModelProviders.of(this).get(MainViewModel::class.java)
        binding.viewModel = viewModel
        binding.executePendingBindings()

        binding.repositoryRv.layoutManager = LinearLayoutManager(this)
        binding.repositoryRv.adapter = repositoryRecyclerViewAdapter
        viewModel.repositories.observe(this,
                Observer<ArrayList<Repository>> { it?.let{ repositoryRecyclerViewAdapter.replaceData(it)} })

    }

    override fun onItemClick(position: Int) {
        TODO("not implemented") //To change body of created functions use File | Settings | File Templates.
    }
}
```

it 关键字是什么意思？如果某个函数只有一个参数，那么该参数可以通过关键字来访问。所以，假设 lambda 表达式的乘法乘以2: 

```kotlin
((a) -> 2  a) 
```

我们可以用 it 来代替。

如果你现在运行它，你可以看到一切正常：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frov9x62vlg30dc09e487.gif)

### 为什么我更喜欢 MVVM 而不是 MVP？

- 没有无聊的接口，ViewModel 没有对 View 的引用。
- Presenter 没有无聊的接口，没有必要。
- 处理配置更改要容易得多。
- 使用 MVVM，我们在活动，片段等中有较少的代码。 

### 存储库模式

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frova17ocdj30k109h0t3.jpg)

正如我前面所说，Model 只是我们准备数据的层的抽象名称。 通常，它包含存储库和数据类。 每个实体（数据）类都应该有对应的 Repository 类。 例如，如果我们有 User 和 Post 数据类，我们也应该有 UserRepository 和 PostRepository。 所有的数据应该直接来自它。 我们绝不应该从 View 或 ViewModel 调用 Shared Preferences 实例或数据库实例。

因此，我们可以将我们的 RepoModel 重命名为 GitRepoRepository，其中 GitRepo 来自 Github Repository， Repository 来自 Repository 模式。

```kotlin
class GitRepoRepository {

    fun getGitRepositories(onRepositoryReadyCallback: OnRepositoryReadyCallback) {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First", "Owner 1", 100, false))
        arrayList.add(Repository("Second", "Owner 2", 30, true))
        arrayList.add(Repository("Third", "Owner 3", 430, false))

        Handler().postDelayed({ onRepositoryReadyCallback.onDataReady(arrayList) }, 2000)
    }
}

interface OnRepositoryReadyCallback {
    fun onDataReady(data: ArrayList<Repository>)
}
```

好的，MainViewModel 从 GitRepoRepsitories 获取 Github 仓库列表，但 GitRepoRepositories 从哪里获取？

你可以直接在库中调用你的客户端实例或数据库实例，但它仍然不是一个好的实践。 你的应用程序应该尽可能的模块化。如果你决定使用不同的客户端，用 Retrofit 来取代 Volley？ 如果你内部有一些逻辑，将很难重构它。 你的存储库不需要知道你正在使用哪个客户端来获取远程数据。

- 存储库唯一需要知道的是数据来自远程或本地。 没有必要知道我们是如何获取这些远程或本地数据的 
- ViewModel 唯一需要的是数据
- View 唯一应该做的就是显示数据

当我开始 Android 开发时，我想知道应用程序如何脱机工作，以及数据同步如何工作。 这款应用的优秀架构让我们可以轻松地完成它。例如，当调用 ViewModel 中的 loadRepositories 时，如果有互联网连接，GitRepoRepositories 可以从远程数据源获取数据并将其保存在本地数据源中。 一旦手机处于离线模式，GitRepoRepository 可以从本地数据源获取数据。 所以，存储库应该有 RemoteDataSource 和 LocalDataSource 的实例以及处理数据来源的逻辑。

让我们添加本地数据源：

```kotlin
class GitRepoLocalDataSource {

    fun getRepositories(onRepositoryReadyCallback: OnRepoLocalReadyCallback) {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First From Local", "Owner 1", 100, false))
        arrayList.add(Repository("Second From Local", "Owner 2", 30, true))
        arrayList.add(Repository("Third From Local", "Owner 3", 430, false))

        Handler().postDelayed({ onRepositoryReadyCallback.onLocalDataReady(arrayList) }, 2000)
    }

    fun saveRepositories(arrayList: ArrayList<Repository>){
        //todo save repositories in DB
    }
}

interface OnRepoLocalReadyCallback {
    fun onLocalDataReady(data: ArrayList<Repository>)
}
```

这里我们有两种方法：第一种是返回伪造的本地数据，第二种是伪造的数据保存。

我们添加远程数据源：

```kotlin
class GitRepoRemoteDataSource {

    fun getRepositories(onRepositoryReadyCallback: OnRepoRemoteReadyCallback) {
        var arrayList = ArrayList<Repository>()
        arrayList.add(Repository("First from remote", "Owner 1", 100, false))
        arrayList.add(Repository("Second from remote", "Owner 2", 30, true))
        arrayList.add(Repository("Third from remote", "Owner 3", 430, false))

        Handler().postDelayed({ onRepositoryReadyCallback.onRemoteDataReady(arrayList) }, 2000)
    }
}

interface OnRepoRemoteReadyCallback {
    fun onRemoteDataReady(data: ArrayList<Repository>)
}
```

这只是一个方法，返回伪造的远程数据。

现在我们可以为我们的存储库添加一些逻辑: 

```kotlin
class GitRepoRepository {

    val localDataSource = GitRepoLocalDataSource()
    val remoteDataSource = GitRepoRemoteDataSource()

    fun getRepositories(onRepositoryReadyCallback: OnRepositoryReadyCallback) {
       remoteDataSource.getRepositories( object : OnRepoRemoteReadyCallback {
           override fun onDataReady(data: ArrayList<Repository>) {
               localDataSource.saveRepositories(data)
               onRepositoryReadyCallback.onDataReady(data)
           }

       })
    }
}

interface OnRepositoryReadyCallback {
    fun onDataReady(data: ArrayList<Repository>)
}
```

因此，我们可以轻松地在本地保存数据。 

如果你只需要来自网络的数据，你还需要使用存储库模式吗？ 是的。 它使你的代码更容易测试，其他开发人员可以更好的理解你的代码，你可以更快地维护它！  

### Android Manager Wrappers

如果你想检查 GitRepoRepository 中的互联网连接，这样你就可以知道哪些数据来源可以要求数据？ 我们已经说过，我们不应该在 ViewModels 和 Models 中放置任何与 Android 相关的代码，那么如何处理这个问题呢？ 

让我们做一个互联网连接的包装器: 

```kotlin
class NetManager(private var applicationContext: Context) {
    private var status: Boolean? = false

    val isConnectedToInternet: Boolean?
        get() {
            val conManager = applicationContext.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
            val ni = conManager.activeNetworkInfo
            return ni != null && ni.isConnected
        }
}
```

只有在清单中添加权限时，此代码才能正常工作：

```groovy
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
```

但是如何在存储库中进行实例处理，因为我们没有上下文？ 我们可以在构造函数中要求它: 

```kotlin
class GitRepoRepository (context: Context){

    val localDataSource = GitRepoLocalDataSource()
    val remoteDataSource = GitRepoRemoteDataSource()
    val netManager = NetManager(context)

    fun getRepositories(onRepositoryReadyCallback: OnRepositoryReadyCallback) {
        remoteDataSource.getRepositories(object : OnRepoRemoteReadyCallback {
            override fun onDataReady(data: ArrayList<Repository>) {
                localDataSource.saveRepositories(data)
                onRepositoryReadyCallback.onDataReady(data)
            }

        })
    }
}

interface OnRepositoryReadyCallback {
    fun onDataReady(data: ArrayList<Repository>)
}
```

我们在 ViewModel 中创建了一个 GitRepoRepository 的新实例。 我们现在如何在 ViewModel 中使用 NetManager，因为我们需要 NetManager 的上下文？ 你可以使用具有上下文的生命周期感知组件库中的 AndroidViewModel。 该上下文是应用程序的上下文，而不是活动的上下文：

```kotlin
class MainViewModel : AndroidViewModel  {

    constructor(application: Application) : super(application)

    var gitRepoRepository: GitRepoRepository = GitRepoRepository(NetManager(getApplication()))

    val text = ObservableField("old data")

    val isLoading = ObservableField(false)

    var repositories = MutableLiveData<ArrayList<Repository>>()

    fun loadRepositories() {
        isLoading.set(true)
        gitRepoRepository.getRepositories(object : OnRepositoryReadyCallback {
            override fun onDataReady(data: ArrayList<Repository>) {
                isLoading.set(false)
                repositories.value = data
            }
        })
    }
}
```

在这一行中：

```kotlin
 constructor(application: Application) : super(application)
```

我们正在为 MainViewModel 定义构造函数。 这是必要的， 因为 AndroidViewModel 在其构造函数中要求 Application 实例。 所以在我们的构造函数中，我们调用了超类方法，所以我们的类将会被调用，所以 AndroidViewModel 的构造函数将被调用。

注意：如果我们这样做，一行解决：

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {
... 
}
```

现在，当我们在 GitRepoRepository 中有 NetManager 的实例时，我们可以检查 Internet 连接：

```kotlin
class GitRepoRepository(val netManager: NetManager) {

    val localDataSource = GitRepoLocalDataSource()
    val remoteDataSource = GitRepoRemoteDataSource()

    fun getRepositories(onRepositoryReadyCallback: OnRepositoryReadyCallback) {

        netManager.isConnectedToInternet?.let {
            if (it) {
                remoteDataSource.getRepositories(object : OnRepoRemoteReadyCallback {
                    override fun onRemoteDataReady(data: ArrayList<Repository>) {
                        localDataSource.saveRepositories(data)
                        onRepositoryReadyCallback.onDataReady(data)
                    }
                })
            } else {
                localDataSource.getRepositories(object : OnRepoLocalReadyCallback {
                    override fun onLocalDataReady(data: ArrayList<Repository>) {
                        onRepositoryReadyCallback.onDataReady(data)
                    }
                })
            }
        }
        
       
    }
}

interface OnRepositoryReadyCallback {
    fun onDataReady(data: ArrayList<Repository>)
}
```

因此，如果我们有互联网连接，我们将获得远程数据并在本地保存。 另一方面，如果我们没有互联网连接，我们将获得本地数据。 

 Kotlin 注意：操作符 let 检查可空性并在其中返回一个值。

在下面的一篇文章中，我会写一些关于依赖注入的文章，为什么在 ViewModel 中对存储库进行实例是不好的，以及如何避免使用 AndroidViewModel。 此外，我还会在代码中写出更多的问题，直到这一点。 我离开他们是有原因的 。

我正试图面对你的问题，所以你可以理解为什么所有这些库很受欢迎，为什么你应该使用它。

附： 我改变了关于 mappers 的想法。 我决定在下一篇文章中介绍它。

