# 架构组件陷阱ー第1部分

> 原文 (Medium)：[Architecture Components pitfalls — Part 1](https://medium.com/@BladeCoder/architecture-components-pitfalls-part-1-9300dd969808)
>
> 作者：[Christophe Beyls](https://medium.com/@BladeCoder?source=post_header_lockup)

[TOC]

在几个月的公开测试之后，新的 [Android 架构组件](https://developer.android.com/topic/libraries/architecture/)很快就会被宣布为稳定的。 

已经有很多关于基础的文章（从非常好的文档开始），所以我不会在这里涵盖它们。 相反，我想关注一些重要的缺陷，这些缺陷大多没有记录，很少讨论，如果你错过了它们，可能会导致你的应用程序出现问题。在第一篇文章中，我会谈论我们钟爱的片段。

> 编辑(2018年5月14日) : 最终在支持库28.0.0版本和 AndroidX 1.0.0中解决了这个问题。 看吧，下面的4号方案。

架构组件为活动和片段提供默认的 ViewModelProvider 实现。 它们允许你将 [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 实例存储在 [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html) 中，以便在配置更改中重用。 活动的使用非常简单，因为活动生命周期很好地映射到了体系结构组件的 [Lifecycle](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.html) 接口，但是如果你不小心的话，它的[片段生命周期](https://developer.android.com/guide/components/fragments.html#Lifecycle)更加复杂，可能会产生微妙的副作用。 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1froubcq9kaj30m80p0wjp.jpg)

片段可以分离和重新连接。当它们被分离时，它们的视图层次被破坏，它们变得不可见和不活动，但是它们的实例并没有被破坏。当它们稍后重新连接时，将创建一个新的视图层次结构，并在 onCreateView ( ) 和 onActivityCreated ( ) 上创建了一个新的视图层次结构。 

因此 通常推荐在 onActivityCreated ( ) 中初始化 Loader 和其他异步加载操作的地方，最终将与视图层次结构交互。我们可以假设这也是通过订阅一个新 Observer 来初始化 LiveData 实例的最佳地点。 大多数官方的架构组件样本也是在那里做的。 你会期待典型的代码看起来像这样: 

```kotlin
class Page1Fragment : Fragment() {

    private var textView: TextView? = null

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val view = inflater.inflate(R.layout.fragment_page1, container, false)
        textView = view.findViewById(R.id.result)
        return view
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        val vm = ViewModelProviders.of(this)
                .get(Page1ViewModel::class.java)

        vm.myData.observe(this, Observer { textView?.text = it })
    }

    override fun onDestroyView() {
        super.onDestroyView()
        textView = null
    }
}
```

这个代码存在一个隐藏的问题。 我们在片段的生命周期内订阅了一个匿名的观察者(使用片段作为生命周期所有者) ，这意味着当片段被破坏时，它将自动被取消订阅。然而，当片段被分离和重新连接时，它不会被破坏，并且将在 onActivityCreated ( ) 中添加一个新的相同观察者实例，而前一个没有被删除！结果就是同时有越来越多的同样的观察者处于活跃状态，同样的代码被多次执行，这既是一个内存泄漏，也是一个性能问题(直到片段被销毁)。 

这个问题涉及任何在 onCreateView ( ) 中为其自己的生命周期订阅一个观察者的任何片段，或者后来，没有采取任何额外的步骤来取消它。 

更糟糕的是，它还会影响[保留的片段](https://developer.android.com/reference/android/support/v4/app/Fragment.html#setRetainInstance%28boolean%29)，这些片段在配置改变时没有被破坏，而是重新附加到一个新的活动中。 

我们如何解决这个问题？ 有一些解决方案需要探索，有些比其他的更好。 以下就是我目前为止所发现的。 

## 1.在 onCreate（）中观察

你的第一个尝试可能是简单地将观察者订阅移到 onCreate ( ) 而不是 onActivityCreated ( )。 但是如果不添加更多的模板代码，它就不能正常工作: LiveData 跟踪哪些结果已经传递给了哪个观察者，它不会再次传递最新的结果到一个新的视图层次结构，因为观察者没有改变。这意味着你必须手动检查最新结果，并将它绑定到 onActivityCreated ( ) 中的新视图层次结构，这并不是很优雅: 

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    vm = ViewModelProviders.of(this)
            .get(Page1ViewModel::class.java)

    vm.myData.observe(this, Observer(this::bindResult))
}

override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)

    vm.myData.value?.apply(this::bindResult)
}

private fun bindResult(result: String?) {
    textView?.text = result
}
```

请注意，绑定代码需要在两个不同的地方调用，所以你不能只是写一个匿名的观察者。最后，当前的LiveData API 没有办法将空结果与当前值的结果区分开来，因此最好不要在使用此解决方案时返回任何空结果（例如在发生错误的情况下），这个解决方案总体上可能是最糟糕的一个。

## 2.在 onDestroyView（）中手动取消观察者

这比第一个解决方案要好一些，但是你仍然不能使用内联的匿名观察者。 你必须将其声明为最终字段， 在 onActivityCreated（）中订阅它，并且不要忘记在 onDestroyView（）中取消订阅它，所以仍然存在样板代码和错误的空间。

```kotlin
private val observer = Observer<String?> { textView?.text = it }

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    vm = ViewModelProviders.of(this)
            .get(Page1ViewModel::class.java)
}

override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)

    vm.myData.observe(this, observer)
}

override fun onDestroyView() {
    super.onDestroyView()
    textView = null

    vm.myData.removeObserver(observer)
}
```

使用 LiveData 的主要好处之一就是它可以为你解除订阅观察者的订阅，但遗憾的是，在这种情况下必须手动完成。 继续读下去，我们仍然可以做得更好。 

## 3.重置现有的观察者

它实际上不需要在 onDestroyView ( ) 中准确地取消当前观察者的订阅，因为它最终将在 onDestroy ( ) 中自动取消。 重要的是，在 onActivityCreated ( ) 上订阅一个完全相同的订阅之前取消订阅 ，以避免重复。 因此，另一个有效的解决方案是在订阅之前取消订阅，例如使用这个 Kotlin 扩展函数: 

```kotlin
fun <T> LiveData<T>.reObserve(owner: LifecycleOwner, observer: Observer<T>) {
    removeObserver(observer)
    observe(owner, observer)
}
```

删除并添加相同的观察者将有效地重置它的状态，这样 LiveData 在 onStart ( ) 期间自动发送最新的结果（如果有的话）。上面的函数可以这样使用：

```kotlin
private val observer = Observer<String?> { textView?.text = it }

override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)
    val vm = ViewModelProviders.of(this)
            .get(Page1ViewModel::class.java)

    vm.myData.reObserve(this, observer)
}
```

片段代码现在不那么容易出错，因为还有一个回调需要添加，但是我们仍然需要声明观察者实例为最终字段并重用它，否则取消订阅将会默默失败。 尽管如此，这可能是我个人最喜欢的快速解决方案。 

## 4.为视图层次结构创建一个自定义的生命周期

理想情况下，我们希望在 onActivityCreated ( ) 中订阅的观察者在 onDestroyView ( ) 中自动取消订阅。这意味着我们实际上要跟随当前视图层次结构生命周期，这与片段生命周期不同。 实现此目的的一种方法是创建一个自定义 Fragment，为当前视图层次结构提供额外的自定义 [LifecycleOwner](https://developer.android.com/reference/android/arch/lifecycle/LifecycleOwner.html) 。下面是 Java 中的一个实现。

```java
package be.digitalia.archcomponentsfix.fragment;

import android.arch.lifecycle.Lifecycle.Event;
import android.arch.lifecycle.LifecycleOwner;
import android.arch.lifecycle.LifecycleRegistry;
import android.os.Bundle;
import android.support.annotation.Nullable;
import android.support.v4.app.Fragment;
import android.view.View;

/**
 * Fragment providing separate lifecycle owners for each created view hierarchy.
 * <p>
 * This is one possible way to solve issue https://github.com/googlesamples/android-architecture-components/issues/47
 *
 * @author Christophe Beyls
 */
public class ViewLifecycleFragment extends Fragment {

	static class ViewLifecycleOwner implements LifecycleOwner {
		private final LifecycleRegistry lifecycleRegistry = new LifecycleRegistry(this);

		@Override
		public LifecycleRegistry getLifecycle() {
			return lifecycleRegistry;
		}
	}

	@Nullable
	private ViewLifecycleOwner viewLifecycleOwner;

	/
	  @return the Lifecycle owner of the current view hierarchy,
	  or null if there is no current view hierarchy.
	 /
	@Nullable
	public LifecycleOwner getViewLifecycleOwner() {
		return viewLifecycleOwner;
	}

	@Override
	public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
		super.onViewCreated(view, savedInstanceState);
		viewLifecycleOwner = new ViewLifecycleOwner();
		viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_CREATE);
	}

	@Override
	public void onStart() {
		super.onStart();
		if (viewLifecycleOwner != null) {
			viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_START);
		}
	}

	@Override
	public void onResume() {
		super.onResume();
		if (viewLifecycleOwner != null) {
			viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_RESUME);
		}
	}

	@Override
	public void onPause() {
		if (viewLifecycleOwner != null) {
			viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_PAUSE);
		}
		super.onPause();
	}

	@Override
	public void onStop() {
		if (viewLifecycleOwner != null) {
			viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_STOP);
		}
		super.onStop();
	}

	@Override
	public void onDestroyView() {
		if (viewLifecycleOwner != null) {
			viewLifecycleOwner.getLifecycle().handleLifecycleEvent(Event.ON_DESTROY);
			viewLifecycleOwner = null;
		}
		super.onDestroyView();
	}
}
```

感谢 getViewLifecycleOwner ( ) 返回的自定义 LifecycleOwner，当视图层次结构被破坏时，LiveData 观察者将被自动取消订阅，而不需要执行任何操作。

>注意：如果 onCreateView ( ) 返回 null，onViewCreated ( ) 将不会被调用，所以自定义的LifecycleOwner 不会被创建用于无头片段(没有任何 UI 的片段 )，并且 getViewLifecycleOwner ( ) 也会在这种情况下返回 null。

通过这个解决方案，代码现在几乎与最初的代码示例相同，但这次它做了正确的事情。 

```kotlin
override fun onActivityCreated(savedInstanceState: Bundle?) {
    super.onActivityCreated(savedInstanceState)
    val vm = ViewModelProviders.of(this)
            .get(Page1ViewModel::class.java)

    vm.myData.observe(viewLifecycleOwner,
            Observer { textView?.text = it })
}
```

缺点是它需要从自定义片段实现继承。 也可以在没有继承的情况下实现相同的功能，但是它需要在需要自定义 LifecycleOwner  的每个片段内部声明一个委托助手类。 

> 编辑（2018年5月14日）：Google 直接在支持库28.0.0和 AndroidX 1.0.0中实施此解决方案。 所有片段现在都提供了一个额外的 getViewLifecycleOwner ( ) 方法，就像上面的示例一样，所以你不需要自己实现它。

## 5.使用数据绑定

> 在 Android Studio 3.1发布之后，这个部分被重写，所描述的解决方案被认为是生产准备 

最后一个解决方案提供了最干净的体系结构，而不是依赖于其他的库。 它包括使用  [Android Data Binding ](https://developer.android.com/topic/libraries/data-binding/index.html) 库来自动将模型绑定到当前的视图层次结构，为了简化模型，模型将是已经用来包含各种 LiveData 的 ViewModel 实例。 

使用该体系结构，ViewModel 所暴露的 LiveData 实例将被生成的绑定类而不是片段本身自动地观察到。 

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="viewModel"
            type="my.app.viewmodel.Page1ViewModel"
            />
    </data>
    <TextView
        android:id="@+id/result"
        style="@style/TextAppearance.AppCompat.Large"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@{viewModel.myData}"/>
</layout>
```

该布局声明了类型 Page1ViewModel 的变量，并将 TextView 的 android: text 属性直接绑定到 LiveData 字段。 当 LiveData 更新时，TextView 会立即反映它的新值。

> 注意：在 Android Studio 3.1中添加了将 LiveData 字段直接绑定到视图并使感知 Binding 类生命周期的能力。 

ViewModel 将始终反映片段的最新可视状态，即使当前没有视图层次结构附加到该片段。每次通过 Binding 类创建视图层次结构时，它都将自己注册为 LiveData 实例上的新观察者，所以我们不需要再手动管理任何观察器，我们就可以解决这个问题。 

```kotlin
class Page1Fragment : Fragment() {

    lateinit var vm: Page1ViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        vm = ViewModelProviders.of(this)
                .get(Page1ViewModel::class.java)
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val binding = FragmentPage1Binding
                .inflate(inflater, container, false)
        binding.viewModel = vm
        binding.setLifecycleOwner(this)
        return binding.root
    }
}
```

由于 LifecycleOwner  被传递到 Binding 实例，LiveData 的加载逻辑仍然会遵循当前片段的生命周期: 只有当片段启动时，视图才会更新，内部观察器将在 onDestroy ( ) 中正确地取消订阅。 

此外，Binding 使用特殊的 LiveData 观察者在内部使用了弱引用，因此，即使片段本身尚未被销毁，在它们的关联视图层次结构被破坏和垃圾收集后，最终会自动取消注册。 

总的来说，这种方法通过移除棘手的观察者逻辑以及所有的 View 模板代码来简化这个片段。 

## 最后警告

应该认真对待这个问题，因为分离一个片段是一个非常常见的操作。 例如，它发生在: 

- 在应用程序的主要活动中切换部分; 
- 使用 FragmentPagerAdapter 在 ViewPager 中的页面之间导航;
- 一个片段被替换为另一个片段，事务被添加到后台堆栈。

而且，当一个片段被分离时，它所有的子片段也都被分离出来。 如果有疑问，最好假设任何片段最终会在某一时刻被分离出来，从第一天开始正确处理这个案例。 

现在你已经意识到了这种行为，并且至少有一些选项可以绕开它。 

>编辑（2018年5月14日）： Google 决定直接在支持库28.0.0和 AndroidX 1.0.0中实施解决方案4。 我现在建议你使用 onCreateView ( ) 中的 getViewLifecycleOwner ( ) 返回的特殊生命周期注册你的观察者。
>
>我认为这篇文章在谷歌解决这个问题和提出解决方案方面发挥了作用。

不要犹豫在评论部分讨论这个问题，如果你愿意，请分享。 

