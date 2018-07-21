# 使用 ViewModel (Android 体系结构组件)和 Dagger 2 

>原文 (Medium) ：[ViewModel with Dagger2 (Android Architecture Components)](https://proandroiddev.com/viewmodel-with-dagger2-architecture-components-2e06f06c9455)
>
>作者：[Mario Sanoguera de Lorenzo](https://proandroiddev.com/@sanogueralorenzo?source=post_header_lockup)

大家好！ 在这个故事中，我想分享一些关于如何使用 ViewModel (Android 体系结构组件)和 Dagger 2 注入。

如果你不熟悉 ViewModel 和 LiveData，我强烈推荐我[以前的故事]( https: / / proandroiddev.com / the-death-of-presenters-and-rise-of-viewmodels-aac-f14d54b419a):

在开始之前，如果你正在考虑用 ViewModels 和 LiveData 取代 presenter，你可以[查看]( https: / / github.com / sanogueralorenzo / android-kotlin-clean-architecture / pull / 9)。

当重构时，评估任务的大小和实际的效益是非常重要的。 另一个解决办法完全可以建立一些一般性的规则，比如:

> 从现在开始使用 LiveData 和 ViewModel。
>
> 如果演示者中发生重大变化，请将 Presenter重构为ViewModel。

无论如何，让我们回到 Dagger 2的问题上，然后跳转到代码！

我将以我的样本项目为例:

[sanogueralorenzo/Android-Kotlin-Clean-Architecture | github.com](https://github.com/sanogueralorenzo/Android-Kotlin-Clean-Architecture)

#### 创建 ViewModel 然后用 @Inject 注入一个用例和映射器

```kotlin
class PostListViewModel @Inject constructor(private val useCase: UsersPostsUseCase,
                                            private val mapper: PostItemMapper) : ViewModel() {

    val posts = MutableLiveData<Data<List<PostItem>>>()
    private val compositeDisposable = CompositeDisposable()

    init {
        get()
    }

    fun get(refresh: Boolean = false) =
            compositeDisposable.add(useCase.get(refresh)
                    .doOnSubscribe { posts.postValue(Data(dataState = DataState.LOADING, data = posts.value?.data, message = null)) }
                    .subscribeOn(Schedulers.io())
                    .observeOn(Schedulers.io())
                    .map { mapper.mapToPresentation(it) }
                    .subscribe({
                        posts.postValue(Data(dataState = DataState.SUCCESS, data = it, message = null))
                    }, { posts.postValue(Data(dataState = DataState.ERROR, data = posts.value?.data, message = it.message)) }))

    override fun onCleared() {
        compositeDisposable.dispose()
        super.onCleared()
    }
}
```

将 @Inject 添加到你的 ViewModel 中将无法正常工作。 那么我们怎样才能让我们的 ViewModel 找出依赖关系呢？

#### 通过创建我们自己的 ViewModelFactory

```kotlin
@Singleton
class ViewModelFactory @Inject constructor(private val viewModels: MutableMap<Class<out ViewModel>, Provider<ViewModel>>) : ViewModelProvider.Factory {

    override fun <T : ViewModel> create(modelClass: Class<T>): T = viewModels[modelClass]?.get() as T
}

@Target(AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.PROPERTY_SETTER)
@kotlin.annotation.Retention(AnnotationRetention.RUNTIME)
@MapKey
internal annotation class ViewModelKey(val value: KClass<out ViewModel>)

@Module
abstract class ViewModelModule {

    @Binds
    internal abstract fun bindViewModelFactory(factory: ViewModelFactory): ViewModelProvider.Factory

    @Binds
    @IntoMap
    @ViewModelKey(PostListViewModel::class)
    internal abstract fun postListViewModel(viewModel: PostListViewModel): ViewModel

    //Add more ViewModels here
}
```

上面的代码将这个 ViewModelModule 添加到我们的 Injector 类是我们需要做的一切！ (记得在 ViewModelModule 中添加任何新的 ViewModels)。

最后，我们的活动将注入 ViewModelProvider.Factory ，并将作为工厂（第二个参数）传递给ViewModelProviders.of（ ）方法。

```kotlin
class PostListActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelFactory: ViewModelProvider.Factory
    @Inject
    lateinit var postListNavigator: PostListNavigator
    @Inject
    lateinit var userDetailsNavigator: UserDetailsNavigator

    private val avatarClick: (String) -> Unit = { userDetailsNavigator.navigate(this, it) }
    private val itemClick: (PostItem) -> Unit = { postListNavigator.navigate(this, it) }
    private val adapter = PostListAdapter(avatarClick, itemClick)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_post_list)
        getAppInjector().inject(this)

        postsRecyclerView.adapter = adapter
        swipeRefreshLayout.setColorSchemeColors(ContextCompat.getColor(this, R.color.colorAccent))

        val vm = ViewModelProviders.of(this, viewModelFactory)[PostListViewModel::class.java]
        vm.posts.observe(this, Observer(::updatePosts))
        swipeRefreshLayout.setOnRefreshListener { vm.get(refresh = true) }
    }

    private fun updatePosts(data: Data<List<PostItem>>?) {
        data?.let {
            when (it.dataState) {
                DataState.LOADING -> swipeRefreshLayout.startRefreshing()
                DataState.SUCCESS -> swipeRefreshLayout.stopRefreshing()
                DataState.ERROR -> swipeRefreshLayout.stopRefreshing()
            }
            it.data?.let { adapter.addItems(it) }
            it.message?.let { toast(it) }
        }
    }
}
```

完成！我们的 ViewModels 和 @Inject 现在可以工作。

作为额外的一步，我想指出一些我从 [Antonio Leiva](https://medium.com/@antoniolg) 那里学到的很酷的东西。

```kotlin
//Standard way 
val vm = ViewModelProviders.of(this, viewModelFactory)[PostListViewModel::class.java]
vm.posts.observe(this, Observer(::updatePosts))
swipeRefreshLayout.setOnRefreshListener { vm.get(refresh = true) }

//Antonio's Leiva way
withViewModel<PostListViewModel>(viewModelFactory) {
    observe(posts, ::updatePosts)
    swipeRefreshLayout.setOnRefreshListener { get(refresh = true) }
}
```

正如你所看到的，withviewModel 更酷，你需要的只是3种方法：

```kotlin
inline fun <reified T : ViewModel> FragmentActivity.getViewModel(viewModelFactory: ViewModelProvider.Factory): T {
    return ViewModelProviders.of(this, viewModelFactory)[T::class.java]
}

inline fun <reified T : ViewModel> FragmentActivity.withViewModel(viewModelFactory: ViewModelProvider.Factory, body: T.() -> Unit): T {
    val vm = getViewModel<T>(viewModelFactory)
    vm.body()
    return vm
}

fun <T : Any, L : LiveData<T>> LifecycleOwner.observe(liveData: L, body: (T?) -> Unit) {
    liveData.observe(this, Observer(body))
}
```

前两个有一个 viewModelFactory 添加，但所有这些和更多(如没有 Dagger 注入的工厂)在这里解释: 

<https://antonioleiva.com/architecture-components-kotlin>

如果你还在这里，你可能想知道 @Inject 如何向 ViewModel 注入动态参数？ (例如，通过意图传递给新活动的用户 id)。

即使可以，在 MVVM 中，ViewModel 也不应该从视图中得到任何东西，即使它是通过 Dagger。

相反，让我们做一个简单的解决方案，这样活动就可以将 userId 设置为 ViewModel。

```kotlin
class UserDetailsViewModel @Inject constructor(private val useCase: UserUseCase,
                                               private val mapper: UserItemMapper) : ViewModel() {

    val user = MutableLiveData<UserItem>()

    var userId: String? = null
        set(value) {
            if (field == null) {
                field = value
                get()
            }
        }
    
    //method to fetch user
    fun get(refresh: Boolean = false) {}
}
```

这里重要的是值 userId 只会被设置一次，当它发生时它也会触发方法 get ( )（提取用户）。 这将避免重新获取数据，而不管屏幕经过的旋转量。

这个问题很有趣，可以理解为什么 ViewModel 不能通过注入来获得它: 

<https://github.com/googlesamples/android-architecture-components/issues/207>

