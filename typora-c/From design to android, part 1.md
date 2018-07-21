# 从设计到 android - 第一部分

> 原文 (saulmm.github.io)：[From design to android, part 1](http://saulmm.github.io/from-design-to-android-part1)
>
> 作者：[Saúl Molinero](http://saulmm.github.io/)

[TOC]

由于像 [Dribbble](https://dribbble.com/) 或 [MaterialUp](https://material.uplabs.com/) 这样的惊人的设计平台，我们作为开发人员可以访问一个巨大的设计概念，界面和点击提案目录。尽管如此，有时候一些细节几乎不可能实现，有时候 UX 的某些方面也没有考虑到。

出于这个原因，我认为创建一个关于编写一些 Dribbble 或 MaterialUp 镜头并在 android 上实现它们的帖子的项目会很好，解释一些实现细节和我认为重要的 UI / UX android 技巧。

## The concept

这是我为第一部分选择的概念，简单而复杂，足以阻止一些有趣的主题，如 [ConstraintLayout](https://developer.android.com/training/constraint-layout/index.html) ＆ [chains](https://developer.android.com/training/constraint-layout/index.html#constrain-chain)，[DataBinding](https://developer.android.com/topic/libraries/data-binding/index.html)，优化 UI 层次和[场景](https://developer.android.com/training/transitions/scenes.html)的重要性。

- 这个概念的作者 [Johnyvino](https://material.uplabs.com/johnyvino)，并已发表在 [MaterialUp](https://material.uplabs.com/posts/preferred-date-and-time)上
- 这个实现可以在 GitHub 的这个[仓库](https://github.com/saulmm/from_design_to_android_part1)找到。

![](http://saulmm.github.io/resources/codeUI1/concept.gif)

让我们开始吧！

### Bottom sheets from the support library

在拍摄中，[bottoms sheets](https://material.io/guidelines/components/bottom-sheets.html) 用于引导用户通过购买过程。在 Android 中，您可以找到一些第三方库来实现这种视图，如 [Umano 的 AndroidSlidingUpPanel 库](https://github.com/umano/AndroidSlidingUpPanel)。

最近，底部纸张被纳入[设计支持库](https://developer.android.com/training/material/design-library.html)，所以我决定用它们来学习如何工作。

![](http://saulmm.github.io/resources/codeUI1/bottom_sheets.gif)

底部纸张可以两种方式使用，持续补充主视图（使用附加到 [CoordinatorLayout](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.html) 内的视图组的 [BottomSheetBehavior](https://developer.android.com/reference/android/support/design/widget/BottomSheetBehavior.html)），或者如果以模态方式显示信息，则可以使用 [BottomSheetDialogFragment](https://developer.android.com/reference/android/support/design/widget/BottomSheetDialogFragment.html)。

对于这个例子，我选择了 BottomSheetDialogFragment，因为信息不那么重要  - 主要视图和需要的焦点 最终以模态的方式显示。 就代码而言，这个新元素的使用类似于 [DialogFragment](https://developer.android.com/reference/android/app/DialogFragment.html)。

HomeActivity.java

```java
OrderDialogFragment.newInstance(
  fakeProducts.get(position))
    .show(getSupportFragmentManager(),null);
```

OrderDialogFragment.java

```java
  @Override
public View onCreateView(LayoutInflater inflater,
    ViewGroup container,Bundle savedInstanceState) {

    super.onCreateView(inflater,
        container, savedInstanceState);

    binding = FragmentOrderFormBinding.inflate(
        inflater, container, false);

    return binding.getRoot();
}
```

## ConstraintLayout

Google I / O16 上呈现的这个强大的布局最近达到了稳定版本（写作时为1.0.2）。

除了其他好处之外，它还允许在一个平面视图层次结构中构建复杂的响应式布局。

![](http://saulmm.github.io/resources/codeUI1/constraint_sample.png)

在 android 中的一个常见建议是避免我们的布局内部的深层次的视图，因为损害了我们的 UI 在屏幕上绘制的性能和时间。这是我们使用 [ConstraintLayout](https://developer.android.com/training/constraint-layout/index.html) 时免费获得的。

![](http://saulmm.github.io/resources/codeUI1/constraint_hierarchy.png)

基本上，[ConstraintLayout](https://developer.android.com/training/constraint-layout/index.html) 与 [RelativeLayout](https://developer.android.com/reference/android/widget/RelativeLayout.html) 类似，这是定义视图和屏幕之间的关系。不过，ConstraintLayout 不仅更加流畅，而且在 Android Studio 中表现得非常好。 

其中好处有更多有趣的机制，比如视图之间的 chains （当不同的视图被相互约束时）或者像 guidelines （可以用于从百分比或固定距离限制视图的指南）那样的元素。

### 使用 chains

假设我们有两个视图 A 和 B，分别限制在右边/左边的屏幕边缘。 如果我们限制 A 为正确的 B，而 B 为左边的这些视图可以以特殊的方式处理，在约束布局领域这被称为链。

![](http://saulmm.github.io/resources/codeUI1/chain1.png)

使用 Android Studio 布局编辑器可以很容易地在布局编辑器的上下文菜单中创建和处理链。默认情况下，Android Studio 创建传播链，视图分散。

![](http://saulmm.github.io/resources/codeUI1/chain_creation.gif)

### Chain 类型

![](http://saulmm.github.io/resources/codeUI1/chains-styles.png)

在这个例子中，只使用了传播链和包装链，然而，根据您的需要，有很多类型可供选择，最近 Google 已经改进了很多关于 [ConstraintLayout](https://developer.android.com/reference/android/support/constraint/ConstraintLayout.html) 和[链](https://developer.android.com/reference/android/support/constraint/ConstraintLayout.html#Chains)的文档，所以开始使用它们！

## 平移所选视图

这看起来很简单，每次用户点击一个产品参数时，选定的视图会左移到左下角的标签。

![](http://saulmm.github.io/resources/codeUI1/translating_views.gif)

为此，将新视图添加到用户刚刚点击的父布局中。

OrderDialogFragment.java

```java
private void transitionSelectedView(View v) {
    final View selectionView = createSelectionView(v);

    binding.mainContainer.addView(selectionView);

    startCloneAnimation(selectionView, getTargetView(v));
}
```

然后，平移视图，使用 [TransitionManager](https://goo.gl/QwCNwA) 中的 [beginDelayedTransition](https://goo.gl/QwCNwA) 方法完成此操作。该方法检测布局中是否发生了变化，如果是，则使用所选的变换执行动画。

OrderDialogFragment.java

```java
private void startCloneAnimation(View clonedView, View targetView) {
    clonedView.post(() -> {
        TransitionManager.beginDelayedTransition(
            (ViewGroup) binding.getRoot(), selectedViewTransition);

        // Fires the transition
        clonedView.setLayoutParams(SelectedParamsFactory
            .endParams(clonedView, targetView));
    });
}
```

## ViewSwitcher

 Android SDK 的 [ViewSwitcher](https://developer.android.com/reference/android/widget/ViewSwitcher.html)，一个小部件可能不是很受欢迎，它允许在两个视图之间移动和移出动画。对于这种情况，完全适合购买过程中的布局步骤变化。对于这种情况，完全适合购买过程中的布局步骤变化。

fragment_order_form.xml

```xml
<ViewSwitcher
    android:id="@+id/switcher"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:inAnimation="@anim/slide_in_right"
    android:outAnimation="@anim/slide_out_left"
    >

    <include
        android:id="@+id/layout_step1"
        layout="@layout/layout_form_order_step1"
        />

    <include
        android:id="@+id/layout_step2"
        layout="@layout/layout_form_order_step2"
        />
</ViewSwitcher>
```

![](http://saulmm.github.io/resources/codeUI1/switcher.gif)

OrderDialogFragment.java

```java
private void showDeliveryForm() {
    binding.switcher.setDisplayedChild(1);
    initOrderStepTwoView(binding.layoutStep2);
}
```

## Databinding

对于这个概念的实现，我已经使用了 [Databinding](https://developer.android.com/topic/libraries/data-binding/index.html)，对于我来说，可以在单个对象中容纳整个布局的层次结构，但是还有其他一些值得一提的奇特的 Databading 机制。

### Saving click listeners

由于在这些布局中有很多侦听器可以处理，所以我决定发送一个侦听器对象绑定，在 xml 中，每个视图将指定感兴趣的方法，并将事件正确地重定向到我们的对象。

layout_form_order_step1.xml

```xml
<data>
    <variable
        name="listener"
        type="com.saulmm.cui.OrderDialogFragment.Step1Listener"
        />
</data>

<CircleImageView
    android:id="@+id/img_color_blue"
    style="@style/Widget.Color"
    android:onClick="@{listener::onColorSelected}"
    />

<CircleImageView
    android:id="@+id/img_color_blue"
    style="@style/Widget.Color"
    android:onClick="@{listener::onColorSelected}"
    />

```

OrderDialogFragment.java

```java
layoutStep1Binding.setListener(new Step1Listener() {
    @Override
    public void onSizeSelected(View v) {
        // ...
    }

    @Override
    public void onColorSelected(View v) {
        // ...
    }
})
```

## Saving views with Spannables + databinding + ConstraintLayout

让我们在一分钟之内思考如何实现这个概念的这一部分

![](http://saulmm.github.io/resources/codeUI1/date_layout.png)

一个解决方案可以设置为垂直 LinearLayout 每个项目，包装到另一个水平 LinearLayout 将包括三个项目包裹在一个更多的 LinearLayout 将包含两个标签和两个项目容器。

不要在 Home 这样做

![](http://saulmm.github.io/resources/codeUI1/bad_hierarchy.png)

这看起来不太好吧？之前我们谈到了拥有一个扁平的层次结构的重要性，所以 ConstrainLayout 作为容器是必须的。

每个项目的 LinearLayout 是昂贵的，我们可以意识到，我们正在显示不同大小的文本加上一个边界，所以也许在某种程度上，我们可以使每个项目只能使用一个 TextView。

![](http://saulmm.github.io/resources/codeUI1/good_hierarchy.png)

## Spannables

[Spannables](https://developer.android.com/reference/android/text/Spannable.html) 可以完成不同大小文本的显示工作，并且可以绘制边框。通过这种方式，我们将保存每个项目3个视图，远远好于第一个方法。

使用 Spannables 的问题是，如果我们在某个时间或某个日期项目上，大小写字母的数量将会改变。

有了 [Databinding](https://developer.android.com/topic/libraries/data-binding/index.html) 和他们惊人的 [BindingAdapters](https://developer.android.com/reference/android/text/Spannable.html) 机制，我们可以创建一个属性来设置增加的字母数量。

layout_form_order_step2.xml

```xml
<TextView
    android:id="@+id/txt_time1"
    style="@style/Widget.DateTime"
    app:spanOffset="@{3}"
    />

<TextView
    android:id="@+id/txt_date1"
    style="@style/Widget.DateTime"
    app:spanOffset="@{2}"/>
```

OrderDialogFragment.java

```java
@BindingAdapter("app:spanOffset")
public static void setItemSpan(View v, int spanOffset) {
    final String itemText = ((TextView) v).getText().toString();
    final SpannableString sString = new SpannableString(itemText);

    sString.setSpan(new RelativeSizeSpan(1.75f), itemText.
        length() - spanOffset, itemText.length(),
        Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);

    ((TextView) v).setText(sString);
}
```

## Scenes

当实现这个概念的时候，实质的一点是如何处理 Book 按钮和确认视图之间的转换。

![](http://saulmm.github.io/resources/codeUI1/scene_transition.gif)

如果我们利用烟雾和镜子，并不那么难。如果我们不再把按钮看作是一个按钮，并将它看作一个容器，我们可以意识到我们可以用场景来实现动画。

作为整个形式的第一个场景，第二个是确认视图，按钮作为共享元素。这个共享元素是一个视图，在场景中共享相同的视图，该框架将关注在给定转换的情况下对视图进行适当的动画处理。

一旦我们有了这个想法，改变两个场景的逻辑是直截了当的。

OrderDialogFragment.java

```java
private void changeToConfirmScene() {
    final LayoutOrderConfirmationBinding confBinding =
        prepareConfirmationBinding();

    final Scene scene = new Scene(binding.content,
        ((ViewGroup) confBinding.getRoot()));

    scene.setEnterAction(onEnterConfirmScene(confBinding));

    final Transition transition = TransitionInflater
        .from(getContext()).inflateTransition(
              R.transition.transition_confirmation_view);

    TransitionManager.go(scene, transition);
}
```

## Result

![](http://saulmm.github.io/resources/codeUI1/final_result.gif)

## References

- [ConstraintLayout Documentation](https://developer.android.com/training/constraint-layout/index.html) - Android developers
- [Mastering android drawables](https://speakerdeck.com/cyrilmottier/mastering-android-drawables) - Cyril Mottier
- [Efficient android layouts](https://speakerdeck.com/dlew/efficient-android-layouts-goto-conference) - Dan Lew
- [Plaid](https://github.com/nickbutcher/plaid) - Nick Butcher

