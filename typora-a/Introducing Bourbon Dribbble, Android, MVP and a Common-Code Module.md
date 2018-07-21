# 介绍 Dribbble Android 客户端 Bourbon： MVP  和 Common-Code 模块

> 原文 (Medium)：[Introducing Bourbon: Dribbble, Android, MVP and a Common-Code Module](https://medium.com/exploring-android/introducing-bourbon-dribbble-android-mvp-and-a-common-code-module-1d332a4028b5)
>
> 作者：[Joe Birch](https://medium.com/@hitherejoe)

[TOC]

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo6o56mcj31jk0jgtku.jpg)

在我的 Android 项目中使用 MVP 已经有一段时间了，我一直在想在创建不仅仅是移动设备的应用程序时重用 Presenter 类。 基于这个原因，我建立了 Bourbon 作为一个实验，看看我可以在不同的应用模块之间共享的代码类型。 我很高兴地说，结果是非常积极的，因为我能够分享大量的应用程序逻辑在手机，手表和电视模块。 没有太深入，我决定把这篇文章放在一起，以便与其他人分享我的实验。 

这篇文章分为两个部分。 在第一部分，我们将快速查看 Bourbon 应用程序及其功能——这可以让你先看到应用程序的视觉效果。 然后在第二部分，我们将看看不同的应用程序实例之间如何共享代码，以及如何构建通用代码模块。 

## 什么是 Bourbon？

[Bourbon](https://github.com/hitherejoe/Bourbon)（可能是另一个）支持安卓、电视和移动设备的 Dribbble 客户端(它也是为平板电脑优化的!) . 如果你不知道 dribbble 已经，这是一个社区领导的设计平台，允许设计师以 shots 的形式分享他们最新的作品。 的目的是向用户显示最新的20张照片，让他们可以从他们选择的安卓设备中浏览一些鼓舞人心的东西。 

## 但为什么？

我创建这个项目的想法是在不同的应用程序模块上演示代码的重用——与 MVP 一样，这被证明是一个很有价值的方法。 虽然我想减少这个实验的代码库，但我也想确保该项目仍然易于维护，代码仍然可以读懂——这是在将代码分解成模块时很容易被破解的东西。 

当我最初打算为 Android TV 创建一个 dribbble 客户端时，如同 MVP 一样，一个单一的活动 / 片段结构看起来会是这样的: 

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo6r6m6aj30m8095q34.jpg)

我不想在 MVP 中加入太多内容，但是简单地说，我们有一个实现 MvpView 的活动 / 片段——这是定义我们的活动 / 片段需要实现的方法的接口。 这样就可以让我们的演示器和活动 / 片段彼此交流。 

但是，我决定做一些事情，而不仅仅是为电视创建另一个应用程序，所以我想为什么不同时为手机和 Wear 创建这个应用呢！ 然而，如果没有在应用程序模块中分享任何代码，我们可能会得到这样的结果: 

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo6u3d4xj30rs0c4wf0.jpg)

这只是对这个问题的一个最小的看法。 还会有其他多个类——如 DataManager、数据模型、 API 服务和 Dagger 注入组件 / 模块等。 

这是一个很大的重复，特别是当3个不同的应用程序模块具有完全相同的功能时。 虽然这种情况下，演示者和 MvpView 类是完全相同的，但是这样做不是更好吗？ 

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo6xakbaj30m80eft96.jpg)

因此，我们仍然有我们的单独活动 / 片段实例，但我们将共享相同的 MvpView 和 Presenter 类，这意味着它们只需要定义一次，以便与三个应用程序模块一起使用。 这样好多了，不是吗？  😺

我们将进一步研究 Bourbon 的技术细节及其结构，但首先让我们先来看看这些漂亮的东西。 

## 浏览 Dribbble Shots

应用程序的第一个屏幕只是浏览屏幕。 这个屏幕向用户显示一连串的镜头，以及标题和计数。 这种方法在不同模块之间也是相似的，由共享模块完全处理了镜头的检索——这意味着用于这样做的逻辑仅仅为三个不同的应用模块定义了一次。 在这种情况下，我们的 [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) 定义了下面的接口方法: 

```java
void showShots(List<Shot> shots);
```

然后，在调用 [BrowsePresenter’s](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) [getShots( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L37) 方法之后检索镜头时，从我们的演示者（也是共享的）调用此方法：

```java
@Override
public void onSuccess(List<Shot> shots) {
    ...
    if (!shots.isEmpty()) {
        getMvpView().showShots(shots);
    } else {
        ...
    }
}
```

在每个应用程序模块中，showShots ( ) 方法的实现可以在浏览屏幕类中定义它希望如何处理已成功检索的照片显示: 

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo71w2toj30b40kd0vk.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo7d5wd9j30m80duwin.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo7kouboj30go0iljv1.jpg)

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo7r0fnlj30go0nbagd.jpg)

## 处理浏览错误

你可能已经知道，当你的应用处理远程数据时，事情并不总是有效的。 为了处理这种情况，我们只需让用户知道出了问题，并允许他们尝试重新加载数据，如果他们愿意的话。 因为错误状态通过 [BrowsePresenter](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) 进行分类，这个状态由所有3个应用模块的共享演示器类处理。 在这种情况下，我们的 BrowseMvp 视图定义了下面的[接口](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java#L16)方法: 

```java
void showError();
```

然后，在调用 [BrowsePresenter’s](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) [getShots( ) 方法](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L37) 后发生错误时，从我们的演示者（也是共享的）调用此方法：

```java
@Override
public void onError(Throwable error) {
    ...
    getMvpView().showError();
    ...
}
```

在我们的应用程序模块中，showError ( ) 方法的实现可以定义如何处理已发生的错误: 

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo7ux8s2j30b40jzq4v.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo7ytggtj30m80dujtf.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo83asjwj30go0j0gok.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo884xavj30go0ny0ul.jpg)

## 处理浏览空状态

有时候，我们可能没有任何内容可以显示给用户。 为了处理这种情况，我们只需让用户知道没有可以显示的镜头，允许他们再次检查是否愿意。 因为"空状态"通过 [BrowsePresenter](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) 进行分类，这个状态再次由所有3个应用模块的共享演示器类处理。 在这种情况下，我们的 [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) 定义了下面的[接口](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java#L18)方法: 

```java
void showEmpty();
```

然后，在调用 [BrowsePresenter’s](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) [getShots( ) 方法](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L37)后返回一个空的列表，然后从我们的演示者（也是共享的）调用此方法：

```java
@Override
public void onSuccess(List<Shot> shots) {
    ...
    if (!shots.isEmpty()) {
        ...
    } else {
        getMvpView().showEmpty();
    }
}
```

在我们的应用程序模块中，showEmpty ( ) 方法的实现浏览屏幕类可以定义如何处理已发生的空状态: 

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo8cal7xj30b40k4766.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo8hi58wj30m80dugng.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo8l3wiyj30go0j0jua.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo8ofylej30go0njabs.jpg)

## 查看镜头细节和评论

当用户从浏览屏幕中选择一个镜头时，他们会被带到一个镜头细节屏幕，在那里他们可以查看照片和任何可能存在的评论。 同样，这在3个不同的应用程序模块上完全一样，所以我只需要定义一次这个屏幕的逻辑。 

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo8shd6hj30b40k2414.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1froo8x76l1j30m80du40i.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo93vi5ij30go0netbv.jpg)

![](https://ws1.sinaimg.cn/large/006tNc79gy1froo994tcoj30go0kd45h.jpg)

正如你在上面的手机和平板电脑上所看到的，一个镜头的注释只是在图片下方的可滚动列表中显示。 然而，在电视和 Wear 上，我们采取了略微不同的方法——这仅仅是因为在这两个平台上使用了屏幕。 在这两种情况下，我们简单地使用一个 ViewPager 以一种奇异的方式显示注释。 关于 Wear ，用户可以简单地浏览评论，也可以，在电视上，在平板电脑来浏览评论。 

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo9d5owbj30b40cowfw.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo9gtol1j30m80duwfg.jpg)

在这种情况下，我们的 [ShotMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotMvpView.java) 定义了[下面的接口](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotMvpView.java#L14)方法：

```java
void showComments(List<Comment> comments);
```

然后，在调用 [ShotPresenter’s](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotPresenter.java)[ getComments( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotPresenter.java#L39) 方法后返回一个评论列表，然后从我们的演示者（也是共享的）调用此方法：

```java
@Override
public void onSuccess(List<Comment> comments) {
    ...
    if (comments.isEmpty()) {
        getMvpView().showEmptyComments();
    } else {
        getMvpView().showComments(comments);
    }
    ...
}
```

然后，我们的应用程序模块 Shot 屏幕类中的 showComments ( ) 方法实现可以定义一旦它们被检索后，它如何处理显示评论。

## 与“通用”模块共享代码

从上面的截图中你可以看到，这个应用程序在不同的设备平台上的工作基本相同。 想象一下，如果我们必须为每个代码复制这些代码，就会有很多重复的代码——幸运的是我设法避免了这种情况。 由于我们的应用程序模块都具有相同的逻辑， 所以 Bourbon 使用一个 [CoreCommon](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon) 模块，它允许我们在3个不同的应用程序模块之间共享这些不同的类：

#### [BourbonApplication](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/BourbonApplication.java)

这是一个标准的 Android 应用程序类，我简单地重复使用每个应用程序模块。这本质上使用 Dagger 来设置我们的 ApplicationComponent 和 Timber 用于记录目的。

#### [Data Models](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model)

看到我们的应用程序模块都将显示相同的数据，他们分享数据模型是有道理的。 在应用程序中使用的只有4个（最小）模型( [Shot](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/Shot.java), [User](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/User.java), [Comment](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/Comment.java), [Image](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/Image.java) ) ，但是在模块中共享这些模型可以更容易地维护它们。

#### [DataManager](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/DataManager.java)

DataManager 类充当与 BourbonService 进行通信的中间人。再次，应用程序模块都访问相同的数据，所以共享 DataManager 是合乎逻辑的。

#### [BourbonService](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/remote/BourbonService.java)

BourbonService 声明端点并管理从中检索数据。因此，如上所述的 DataManager，跨应用程序模块的行为是相同的。

#### [Dagger Injection Components and Modules](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/injection)

看到我们现在知道我们的三个应用程序模块使用相同的 DataManager，BourbonService 等 - 只有分享与 Dagger  注入相关的逻辑才有意义。 如果你看一下注入包，你会发现有几个类声明了组件和模块，这意味着可以在应用程序中注入相同的依赖关系。

#### [Base Presenter and MvpView](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/base)

Bourbon  使用基本类的演示者和 MvpViews 应该使用时创建这些新类。 为此，通过 CoreCommon 模块使用它们可确保所有类都扩展或实现相同的基类 - 这同样可以减少代码重复。

#### [BrowseMvpView & BrowsePresenter](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse)

每个应用程序模块的浏览屏幕的行为方式完全相同。 检索镜头列表，如果显示给用户，则显示该列表 - 然而，显示/隐藏进度指示器，提出 API 请求，并向用户正确显示任何空/错误状态。 这意味着 Presenter 类将包含相同的逻辑，并且 MvpView 接口将定义完全相同的接口方法。 由于这个原因， [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) & [BrowsePresenter](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) 都保存在 CoreCommon 模块中，所以这些类只需要定义一次就可以在我们的应用程序模块中共享。

#### [ShotMvpView & ShotPresenter](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot)

用于显示“拍摄细节”的屏幕也是如此。用于处理屏幕上内容显示的类和接口，所以我们通过 CoreCommon 模块共享 [ShotMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotMvpView.java) 和 [ShotPresenter](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotPresenter.java) 。

#### [Colors, String & Dimension files](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/res/values)

Bourbon 具有特定的品牌色彩，所以这不会改变它的应用程序模块。 在整个应用程序中使用的字符串也是一样，也有一些 Dimension 值也适用于此。 因此，我将这些值放在 CoreCommon 模块的资源文件中 - 这意味着它们可以在应用程序模块之间共享。 现在，如果这些颜色或字符串中的任何一个需要更改，我只需要做一次！

#### [TestDataFactory](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/util/TestDataFactory.java)

TestDataFactory 是一个类，用于构造在 Unit 和 Instrumentation 测试中使用的虚拟数据模型。 为此目的，这个类存在于 CoreCommon 模块中，这是 AndroidTestCommon 模块可以从中访问这个类的地方。

#### [Unit Tests](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/test/java/com/hitherejoe/bourboncorecommon)

因为需要单元测试的类可以在 CoreCommon 模块中找到，单元测试也可以在这里找到。单独的软件包包含 CoreCommon 模块中定义的 DataManager 和 Presenter 类的测试。

## 项目结构

Bourbon 项目结构可以很容易地浏览代码库。让我们快速看看它是如何分解成模块：

- **CoreCommon**- 该模块包含在移动，电视和穿戴应用程序中共享的所有核心应用程序逻辑和数据管理类。
- **AndroidTestCommon** - 这个模块包含三个 AndroidTest 模块共享的类。
- **Mobile** - 这是移动应用程序的模块。
- **Wear** - 这是 Wear 应用程序的模块。
- **TV** - 这是电视应用程序的模块。
- **mobile-AndroidTest** - 该模块包含移动应用程序的仪器测试。
- **wear-AndroidTest** - 该模块包含 Wear 应用程序的仪器测试。
- **tv-AndroidTest**- 该模块包含电视应用程序的测试测试。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froo9lopukj30m80nwmxy.jpg)

## 浏览屏幕结构

让我们快速浏览浏览屏幕的结构，使事情变得更加清晰。浏览屏幕流程的步骤如下：

- 在我们的 Activity / Fragment 类中，我们首先调用 [BrowsePresenter’s](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) [getShots( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L37)方法 - 这个 Presenter 类可以在 [CoreCommon](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon) 模块中找到。
- 在此方法中，Presenter 使用实现的 [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java)[ showMessageLayout( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L39) 方法告诉我们的 Activity / Fragment 从布局中移除任何错误/空状态消息。
- 接下来，Presenter 使用实现的 [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java)[ showProgress( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L40) 方法告诉我们的活动/片段向用户显示加载指示符。
- 然后，演示者调用 DataManager 的 [getShots( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L42) 方法从 API 中检索最新的 dribbble 镜头。
- 如果这个请求有错误，那么执行 [hideProgress( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L59) 方法的 [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) 被调用。接下来是 [showError( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L60) 方法。
- 如果结果成功，则实现 [hideProgress( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L48) [BrowseMvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) 方法。然后根据是否返回结果，调用 [showShots( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L50) 或 [showEmpty( )](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java#L52) 方法。

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo9q42m1j30m80lnaas.jpg)

希望从这个例子中你可以看到不同的应用程序模块有多相似以及这个代码被共享的好处。

如果情况并非如此，那么我们必须单独定义：

- The [MvpView](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowseMvpView.java) interface
- The [BrowsePresenter](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenter.java) class
- The [Shot](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/Shot.java), [User](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/User.java) and [Image](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/model/Image.java) data models
- The [DataManager](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/DataManager.java) class
- The [BourbonService](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/data/remote/BourbonService.java) class
- [Various Injection logic](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon/src/main/java/com/hitherejoe/bourboncorecommon/injection)

如果你希望在应用程序中获得类似的结果，那么创建一个模块来共享通用代码是相当容易的 - 最好的开始是 [CoreCommon](https://github.com/hitherejoe/Bourbon/tree/master/CoreCommon) 模块和 [build.gradle](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/build.gradle) 文件。

## 测试 Bourbon

在 GitHub 存储库中，你可以找到“单元”和“测试”测试，在 README 中找到有关如何运行这些测试的详细信息。

#### 单元测试

如前所述，单元测试位于 CoreCommon 模块中。 这是由于所有需要单元测试的类都位于该模块中，因此只有测试位于那里才是正确的。 目前的单元测试类可以在以下位置找到：

- [**BrowsePresenterTest**](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/test/java/com/hitherejoe/bourboncorecommon/ui/browse/BrowsePresenterTest.java) - BrowsePresenter 类单元测试
- [**ShotPresenterTest**](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/test/java/com/hitherejoe/bourboncorecommon/ui/shot/ShotPresenterTest.java) - ShotPresenter 类单元测试
- [**DataManagerTest**](https://github.com/hitherejoe/Bourbon/blob/master/CoreCommon/src/test/java/com/hitherejoe/bourboncorecommon/data/DataManagerTest.java) - DataManager 类单元测试

并遵循代码共享的主题，我们也有一个 [**AndroidTestCommon**](https://github.com/hitherejoe/Bourbon/tree/master/AndroidTestCommon) 模块。 这是用来共享三个 androidTest 模块中使用的通用代码。 这仍然需要进一步利用才能发挥最大的作用，但基础在那里。

## Bourbon 的下一步是什么？

Bourbon 作为一个很好的实验，但我期待延伸功能，我打算加入一些事情：

- **Pagination** - 目前只有最新的20张照片才能从 *API* 中检索出来，理想情况下应该分页，这样用户就可以享受无尽的视觉享受。
- **User profiles** - 有一个用户配置文件屏幕，以便你可以从 Shot 中导航以查看所选用户的更多 Shots。
- **Animation & Screen Transitions** - 我喜欢动画，因此我认为只有这样才能实现一些！
- 还有什么我想到的...

## 结束

我喜欢 Bourbon 作为一个实验，并希望听取人们对这种方法的意见。我很兴奋地将更多的功能部署到应用程序中，并重构更多代码，以便将更多功能转移到 CoreCommon 模块。

如果你有任何问题，请随时给我发一条推文或留下回复！附：如果你喜欢这篇文章，不要忘了点击推荐按钮:)

