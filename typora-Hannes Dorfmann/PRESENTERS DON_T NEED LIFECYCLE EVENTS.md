# 演示者不需要生命周期回调方法

> 原文：[PRESENTERS DON'T NEED LIFECYCLE EVENTS](http://hannesdorfmann.com/android/presenters-dont-need-lifecycle)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about/)

[TOC]

已经有好几次被问到为什么 [Mosby](https://github.com/sockeqwe/mosby)  的演示者（MVP库）没有像 onCreate（Bundle），onResume（）等生命周期回调方法。另外，SoundCloud 上的杰出人物已经发布了一个名为 [LightCycle](https://github.com/soundcloud/lightcycle) 的库， 将活动或片段分割成更小的容器，绑定到父项活动或片段的生命周期。 虽然这个库很棒，也很有帮助，但是他们在[例子](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java)中也提到这个库可以用在 MVP 中来为演示者带来生命周期。 我个人认为，演示者不需要生命周期回调方法，在这篇博客文章中，我将讨论为什么。

好吧，在我们开始之前，先想想 MVP 中演示者的定义和规则。这是我的定义：

> Presenter 负责协调视图，将 Model 转换为 “PresentationModel”，以便 View 可以更方便地显示数据，并成为 “业务逻辑” 的桥梁，以检索应该显示在 View 中的数据。 MVP 是关于分离的关注。

鉴于 Presenter 的 “定义”，我没有看到 Presenter 需要像 onCreate（），onPause（）和 onResume（）这样的生命周期回调的原因。 所有演示者需要知道的是视图是否附加到演示者（由于 androids 屏幕方向的困境）。 应该那么简单。 实际上，这是为什么一些开发人员提倡针对片段（复杂生命周期）而偏好 “自定义视图”（即从 FrameLayout 扩展）的一个论点，因为自定义视图只有两个生命周期事件： ViewGroup.onAttachedToWindow（）和 ViewGroup.onDetachedFromWindow ）（顺便说一句，这就是为什么如果你想把它们作为 “生命周期事件”，[Mosby Presenters](https://github.com/sockeqwe/mosby/blob/master/mvp-common/src/main/java/com/hannesdorfmann/mosby/mvp/MvpPresenter.java) 只有 attachView（）和 detachView（）方法的原因）。

而且，生命周期管理已经是一个复杂的话题。如果您开始在 Presenter 中移动生命周期逻辑，那么您基本上只是将 Activity 中的意大利面代码移至 Presenter。显然，[LightCycle](https://github.com/soundcloud/lightcycle) 的意图是解决这个问题：LightCycle 帮助您将通常在您的 Activity 或 Fragment 的生命周期方法中编写的意大利面代码逻辑拆分或委托给多个遵循[单一职责原则](https://en.wikipedia.org/wiki/Single_responsibility_principle)的较小组件。然而，根据 LightCycle 的[例子](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java)，他们促进（间接）用生命周期方法来构建演示者。不要误解我的意思： LightCycle 的想法非常好，通过注释处理实现它非常聪明，但生命周期管理不是演示者的责任，因此演示者不需要生命周期回调方法。这篇博客并不提倡反对 LightCycle（这太棒了！）。这个博客文章是主张反对 Presenters 一般的生命周期！

有了这个说法，你可以理解我的观点，但是你也面临着从你的角度来看 Presenter 确实需要生命周期回调方法的场景。好吧，我们来谈谈吧！

例如，让我们假设我们要构建一个应用程序，在应用程序的地图上显示用户当前的 GPS 位置。 当屏幕关闭，我们应该停止检索 GPS 位置更新，以安全的电池寿命。 那么我们如何用 MVP 来实现呢？ 我们将有一个 TrackingActivity 作为 View（TrackingView）显示用户当前 GPS 位置的地图。 我们还需要一个 GpsTracker 类作为业务逻辑“（MVP 中的模型），负责检测当前 GPS 位置并通知观察者（听众）关于位置变化。TrackingPresenter 就是这样一个观察者。 他被注册为 GpsTracker 的监听者，并在 GPS 位置改变时更新视图。

到现在为止还挺好。 如前所述，为了不耗尽电池，我们希望在显示屏关闭时停止 GPS 跟踪，换句话说，停止 Activity.onPause（）中的 GPS 跟踪，并在 Activity.onResume（）时继续。 再仔细阅读这个句子。 你明白了吗？ 演示者不需要 onPause（）和 onResume（）生命周期事件。 相反，“业务逻辑” 即 GpsTracker 需要这些生命周期事件。

但是我们如何实现呢？ 我们应该简单地将 Activity.onPause（）转发给 Presenter.onPause（），然后调用GpsTracker.stop（）？ 所以我们确实需要使用生命周期方法的 Presenter，否则我们无法将暂停事件转发到GpsTracker.stop（），对吗？ 我觉得还有更好的办法。 如前所述，演示者不负责处理生命周期事件。 实际上，已经有一个组件负责生命周期事件：Activity（或者 Fragment）。 因此，不要将 Activity.onPause（）和 Activity.onResume（）转发给演示者，只需执行如下操作：

```java
class TrackingActivity extends MvpActivity implements TrackingView {
     private GpsTracker tracker;

     public onCreate(){
            tracker = new GpsTracker(this); // might need a context
            ...
     }

     public void onPause(){
         tracker.stop();
     }

    public void onResume(){
         tracker.start();
     }

    @Override
    public void createPresenter(){ // Called by Mosby
          return new TrackingPresenter(tracker);
     }
}
```

正如你所看到的，诀窍是 GpsTracker 直接绑定到负责管理生命周期的组件中的 Activity 的生命周期中： Activity！ 然后我们将 GpsTracker 作为构造函数参数传递给 Presenter。 此外，现在 TrackingPresenter 履行我以前的定义的单一责任：它只负责更新视图。

```java
class TrackingPresenter extends MvpBasePresenter<TrackingView>
                        implements GpsUpdateListener{

  GpsTracker tracker;

  public TrackingPresenter(GpsTracker tracker){
    this.tracker = tracker;
    tracker.setGpsUpdateListener(this);
  }


  @Override
  public void onGpsLocationUpdated(GpsPosition position){
    view.showCurrentPosition(position.getLat(), position.getLng());
  }
}
```

而已。所有组件的单一责任！

当然，我们可以使用依赖注入和 LightCycle 来调试上面显示的代码：

```java
class TrackingActivity extends MvpActivity implements TrackingView {

    @Inject @LightCycle GpsTracker tracker;

    @Override
    public TrackingPresenter createPresenter(){
          return getObjectGraph.get(TrackingPresenter.class); // Dagger 1
          // or
          return getComponent().trackingPresenter(); // Dagger 2
     }
}
```

你在寻找更多的例子吗？ Mosby 的另一个用户（MVP库）在 github 上向我提出了一个类似的问题，他正在开发一个音乐播放器应用程序。我给了一个类似的[答案](https://github.com/sockeqwe/mosby/issues/124)。

当然，这只是我个人的看法，也有例外，您可能确实需要演示者中的生命周期事件，但我认为在 99％ 的业务逻辑中需要生命周期事件，而不是演示者。 通常（我会说 95％ 的应用程序在那里）没有生命周期感知的业务逻辑。 如果一个应用程序真的有这样的生命周期感知业务逻辑，那么 Presenter 的 99％ 的时间不需要知道生命周期，而是业务逻辑。

所以故事的寓意是：你应该尽可能避免生命周期感知组件。 业务逻辑（模型）或演示者无关紧要。 这是我的一般建议。 生命周期只引入不希望的复杂性。 但是如果你确实真的需要知道生命周期的组件，那么很可能业务逻辑（Model）应该是生命周期感知的，而不是 Presenter。

最后但并非最不重要的，我想说的是，这篇博客的目的不是为了抹黑 SoundCloud 的 LigthCycle！ 我只是想说，从我的角度来看，LightCycle 的[例子](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java)（有一般的生命周期回调方法的 Presenter）不是我的一杯茶。 我认为这主要是由于我对 Presenter 的定义（即，我也不想在演示者中包含 Bundle 之类的 android SDK 依赖项）完全不同于 Presenter 的 SoundClouds 定义。

## Update (03/25/2016)

正如有些人正确指出的（见 reddit）上面显示的代码导致 TrackingView（TrackingActivity）知道 GpsTracker，这绝对是危险的。 我想演示的是在应用程序中已经有一个负责生命周期管理的组件，而不是 Presenter。 我没有说应该直接使用 TrackingView 来处理 GpsTracker。 这仍然是 TrackingPresenter 的工作。 但是，是的，TrackingActivity 有一个 GpsTracker 的参考，可能会被滥用。 真正的问题是 Activity 是 View 和生命周期管理器。 我看到了这个问题的两个解决方案：

- 单独的视图和生命周期责任：我们如何做到这一点？ 那么我们必须将 View 责任或生命周期管理移出活动。 很明显，我们不能轻易地从 Activity 中移除生命周期管理，但是我们可以引入一个额外的图层，这个图层实际上只是一个 TrackingView：

  ```java
  class TrackingActivity extends MvpActivity<TrackignView, TrackingPresenter> {

     @Inject @LightCycle GpsTracker tracker;
     TrackingView view;

     public void onCreate(n){
       super.onCreate(b);
       setContentView(R.layout.activity_tracking);
       view = (TrackingView) findViewById(R.id.tracking_view);
     }

     @Override // called by Mosby, same as before
     public TrackingPresenter createPresenter(){ ... }

     @Override // called by Mosby to connect view with presenter
     public TrackingView getMvpView(){
       return view;
     }
  }
  ```

  这里的东西是我们引入一个额外的视图层。这可以像 TrackingLayout 一样扩展 FrameLayout 实现 TrackingView，所以你可以直接在 xml 布局中使用它，或者你可以构建某种 “包装类”，从 Activity 获取根布局，并在内部管理 UI 小部件或任何最适合你的东西。这种方法可能是最干净的解决方案，因为现在我们在活动（即只有生命周期）中才有真正的单一责任。但是这个解决方案有一个价格：一个额外的层。我不了解你，但是对于我来说，增加另一层是过度工程的开始。如果你需要一个视图中的 Bundle（你真的想使用 BaseSavedState），那么这个视图层上还有一些依赖关系呢？如果视图需要引用窗口来设置隐藏导航栏，沉浸式模式或共享元素转换等标志，该怎么办？也许我们可以使用像 Dagger 这样的依赖注入来解决这样的问题？是的，但不久后你会被依赖注入库抓住，这样我们就再也无法做任何事情了。所以即使在理论上介绍一个专门的“视图”层似乎是一个好主意，但这不是我推荐的解决方案。

- 从视图隐藏 GpsTracker：这似乎是一个工作，但从我的经验来看，这是一个更简单，更实际的解决方案。 原来的问题是什么？ 问题在于 TrackingActivity 有一个对 GpsTracker 的引用，因此可以访问和误用 GpsTracker。 如何解决这个问题呢？ 在我看来，引入一个额外的层让消防队熄灭蜡烛。 为什么不简单地从 TrackingActivity “隐藏” GpsTracker 就像这样：

  ```java
  class LifecycleController  extends DefaultActivityLightCycle {
    private GpsTracker tracker;

    @Inject
    public LifecycleController(GpsTracker tracker){ ... }

    @Override
    public void onPause(){
      tracker.stop();
    }

    @Override
    public void onResume(){
      tracker.start();
    }
  }
  ```

  然后我们将 Activity 视为 TrackingView，就像我们以前所做的那样：

  ```java
  class TrackingActivity extends MvpActivity implements TrackingView {

      @Inject @LightCycle LifecycleController lifecycleController;

      @Override
      public void createPresenter(){ // Called by Mosby
            return new TrackingPresenter(tracker);
       }
  }
  ```

  通过这样做，TrackingActivity 不再有对 GpsTracker 的引用，可以在没有额外图层的情况下被滥用。

  请注意，在这个博客文章中，我们正在讨论像生命周期感知的 GpsTracker 这样的业务逻辑组件。 显然，我不希望你为所有的业务逻辑组件做这件事，即使它们根本不是生命周期知识。 这是无稽之谈。 “传统” MVP 方法相当不错。 当业务逻辑是生命周期感知的组件时，我只是主张不要让 Presenter 生命周期知道。

