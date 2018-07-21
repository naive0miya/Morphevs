# 为 Android 的 BottomSheet 编排动画

> 原文 (Medium)：[Choreographic animations with Android’s BottomSheet](https://android.jlelse.eu/choreographic-animations-with-androids-bottomsheet-fef06e6ecb81)
>
> 作者：[Orkhan Gasimli](https://android.jlelse.eu/@orkhan.gasimli?source=post_header_lockup)

[TOC]

自从 Google 将 BottomSheet 纳入其设计支持库后，开发人员很容易将其整合到他们的应用程序中。

在本质上，BottomSheet 是从屏幕底部向上滑动以向用户显示更多内容的视图类型。对于本篇文章，我会假设你已经熟悉 BottomSheet 并知道如何将它整合到你的应用程序中。对于那些需要复习 BottomSheet 整合的读者，我会推荐阅读 [Emrullah Lüleci](https://medium.com/@emrullahluleci) 的[这篇文章](https://medium.com/android-bits/android-bottom-sheet-30284293f066)。您还可以在 Google Material Design 指南中找到有关 BottomSheet 的各种类型和用例的详细信息。

我不想谈论整合细节，我想告诉你如何将视图动画附加到 BottomSheet 的滑动动画。所以，让我们开始吧！

为了将动画绑定到 BottomSheet 的滑动效果，我们需要知道 BottomSheet 的当前状态。通过对文档和源代码的快速分析，我们可以发现，[BottomSheetBehavior](https://developer.android.com/reference/android/support/design/widget/BottomSheetBehavior.html) 类已经设置了  [setBottomSheetCallback](https://developer.android.com/reference/android/support/design/widget/BottomSheetBehavior.html#setBottomSheetCallback%28android.support.design.widget.BottomSheetBehavior.BottomSheetCallback%29) 方法，该方法可以用来设置一个回调，用于监视 BottomSheet 的事件。 

让我们看看它的实际使用情况。

```java
 private LinearLayout mBottomSheet;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_main, container, false);

        // find container view
        mBottomSheet = view.findViewById(R.id.bottom_sheet);

        initializeBottomSheet();

        return view;
    }

    private void initializeBottomSheet() {        
        // init the bottom sheet behavior
        BottomSheetBehavior bottomSheetBehavior = BottomSheetBehavior.from(mBottomSheet);

        // set callback for changes
        bottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {

            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                // Called every time when the bottom sheet changes its state.
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                // Called when the bottom sheet is being dragged
            }
            
        });
    }
```

从代码中可以看出，BottomSheetCallback 接口有两种方法可用于获取两种类型的信息：

1. BottomSheet 的新状态
2. BottomSheet 的偏移值

因为我们打算将我们的动画绑定到 BottomSheet 的滑动效果，我们将需要使用 onSlide 回调在我们的动画中提供的偏移值。 根据源代码文件，[-1,1] 范围内的偏移偏移参数变化。 

> “随着 BottomSheet 向上移动，偏移量增加。从0到1 BottomSheet 处于折叠状态和展开状态之间，从-1到0表示处于隐藏状态和折叠状态之间。“

考虑到这一点，让我们深入实际的动画实现。在本篇文章中，我只会向你展示两个动画。尽管如此，我希望这将足以抓住主要想法，并强制你的想象力创造出很棒的东西。

## 1. 背景颜色过渡

作为第一个动画，我们将尝试随着它的展开和折叠逐渐改变 BottomSheet 的背景颜色。让我们先看看实际的代码：

AnimateBackgroundColor.java 

```java
  private LinearLayout mBottomSheet;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_main, container, false);

        // find container view
        mBottomSheet = view.findViewById(R.id.bottom_sheet);

        initializeBottomSheet();

        return view;
    }

    private void initializeBottomSheet() {        
        // init the bottom sheet behavior
        BottomSheetBehavior bottomSheetBehavior = BottomSheetBehavior.from(mBottomSheet);

        // change the state of the bottom sheet
        bottomSheetBehavior.setState(BottomSheetBehavior.STATE_COLLAPSED);

        // set callback for changes
        bottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {
            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                // Called every time when the bottom sheet changes its state.
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                if (isAdded()) {
                    transitionBottomSheetBackgroundColor(slideOffset);
                }
            }
        });
    }

    private void transitionBottomSheetBackgroundColor(float slideOffset) {
        int colorFrom = getResources().getColor(R.color.colorGray);
        int colorTo = getResources().getColor(R.color.colorGrayAlpha80);
        mBottomSheet.setBackgroundColor(interpolateColor(slideOffset,
                colorFrom, colorTo));
    }

    /
      This function returns the calculated in-between value for a color
      given integers that represent the start and end values in the four
      bytes of the 32-bit int. Each channel is separately linearly interpolated
      and the resulting calculated values are recombined into the return value.
     
      @param fraction The fraction from the starting to the ending values
      @param startValue A 32-bit int value representing colors in the
      separate bytes of the parameter
      @param endValue A 32-bit int value representing colors in the
      separate bytes of the parameter
      @return A value that is calculated to be the linearly interpolated
      result, derived by separating the start and end values into separate
      color channels and interpolating each one separately, recombining the
      resulting values in the same way.
     /
    private int interpolateColor(float fraction, int startValue, int endValue) {
        int startA = (startValue >> 24) & 0xff;
        int startR = (startValue >> 16) & 0xff;
        int startG = (startValue >> 8) & 0xff;
        int startB = startValue & 0xff;
        int endA = (endValue >> 24) & 0xff;
        int endR = (endValue >> 16) & 0xff;
        int endG = (endValue >> 8) & 0xff;
        int endB = endValue & 0xff;
        return ((startA + (int) (fraction  (endA - startA))) << 24) |
                ((startR + (int) (fraction  (endR - startR))) << 16) |
                ((startG + (int) (fraction  (endG - startG))) << 8) |
                ((startB + (int) (fraction  (endB - startB))));
    }
```

不要害怕代码的长度。这真的很简单。

首先，我们从回调中获取 slideOffset 参数，并将其传递给我们的 transitionBottomSheetBackgroundColor 方法，该方法又将此值与 fromColor 和 toColor 参数一起传递到我们最后一种名为 interpolateColor 的方法。此方法使用位操作技术根据偏移值计算中间颜色的 int 值并将其返回到 transitionBottomSheetBackgroundColor 方法，该方法将返回的颜色值设置为 BottomSheet 的背景颜色。

## 2. 图像动画

我们的第二个动画将是图像旋转。我们将箭头（可绘制的矢量）放置到 BottomSheet 的顶部，并尝试顺时针和逆时针旋转180°。再次，我们将尝试使动画与 BottomSheet 的滑动效果同步。

ImageRotation.java

```java
 private LinearLayout mBottomSheet;
    private ImageView mLeftArrow;
    private ImageView mRightArrow;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_main, container, false);

        // find container view
        mBottomSheet = view.findViewById(R.id.bottom_sheet);

        // find arrows
        mLeftArrow = view.findViewById(R.id.bottom_sheet_left_arrow);
        mRightArrow = view.findViewById(R.id.bottom_sheet_right_arrow);

        initializeBottomSheet();

        return view;
    }

    private void initializeBottomSheet() {
        // init the bottom sheet behavior
        BottomSheetBehavior bottomSheetBehavior = BottomSheetBehavior.from(mBottomSheet);

        // change the state of the bottom sheet
        bottomSheetBehavior.setState(BottomSheetBehavior.STATE_COLLAPSED);

        // set callback for changes
        bottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {
            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                // Called every time when the bottom sheet changes its state.
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                if (isAdded()) {
                    animateBottomSheetArrows(slideOffset);
                }
            }
        });
    }

    private void animateBottomSheetArrows(float slideOffset) {
        // Animate counter-clockwise
        mLeftArrow.setRotation(slideOffset  -180);
        // Animate clockwise
        mRightArrow.setRotation(slideOffset  180);
    }
```

这个动画相当简单。我们只是将我们的 slideOffset 值传递给 animateBottomSheetArrows 方法，并使用它来设置我们图像的旋转值。

作为我们努力的结果，我们将获得如下美丽的动画：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frovu0h9r6g308c0eu7k2.gif)

如果您已经注意到，我们的底部页面位于 Fragment 中，因此我们在运行任何动画之前调用 Fragment 实例的 isAdded 方法。如果片段当前被添加到其活动，则此方法返回 true。如果您在附加到片段的视图中执行动画，检查这一点非常重要。如果您将运行动画而不检查它，则可能会得到 java.lang.IllegalStateException，该异常表明片段未附加到其活动。

您可以从此链接获取源代码。[[PR](https://github.com/ogasimli/BottomSheetDemo)]

希望你喜欢我的第一篇文章。请分享您的反馈和评论，因为它们对我来说意义重大。

Happy coding!

