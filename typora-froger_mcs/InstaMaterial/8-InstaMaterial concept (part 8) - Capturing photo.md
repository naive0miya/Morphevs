# InstaMaterial 概念(第8部分) - 捕捉照片

>原文 (mirekstanek.online) ： [InstaMaterial concept (part 8) - Capturing photo](https://mirekstanek.online/instamaterial-concept-part-8-capturing-photo/)
>作者 ： [Mirek Stanek](https://twitter.com/froger_mcs)  

[TOC]

这篇文章是一系列帖子的一部分，展示了使用 [Material Design 概念的 INSTAGRAM](https://www.youtube.com/watch?v=ojwdmgmdR_Q) 的 Android 实现。 今天，我们将创建照片捕捉流程。 这个功能介于概念视频的第38和第41秒之间。 我们会省略一些细节（比如 FAB 按钮的动画，颜色，图标），因为在实现相机方面还有一些工作要做。 我们会在下一个（可能是最后一个）系列的帖子中回到他们。

根据[之前的帖子](http://frogermcs.github.io/InstaMaterial-concept-part-7-navigation-drawer/)，我不得不提到 InstaMaterial 应用程序已经由于违反知识产权而从 Google Play 商店中删除。 我充分理解了这个原因，不久的将来，我可能会准备与 Instagram 应用程序完全不同的布局版本（但仍然具有概念视频中提供的所有功能）。

从今天发布的代码构建的 APK 文件可以在[这里](https://github.com/frogermcs/frogermcs.github.io/raw/master/files/9/instamaterial-debug.apk)找到。

现在让我们回到这个帖子 - 这是我们今天想要达到的最终效果：

<iframe width="560" height="315" src="https://youtu.be/0w3lGJIISTo" frameborder="0" allowfullscreen></iframe>

## 概论
今天描述的几乎所有的解决方案和动画都出现在之前的帖子中。 这就是为什么我不会专注于详细的描述。 但是也会有一个全新的元素摄像机。 那些以前曾经与它合作过的人知道，这个内容对于实现来说有点问题。 为什么？ 简而言之，安卓作为一个开放的平台可以处理非常广泛的设备与完全不同的硬件(特别是相机)规范。 因此，如果你在这里开始你的拍摄工作，你需要注意的是一些简短的列表:

- 不同的预览和捕获的图像

  每个设备都有一个有限的支持尺寸(对于预览和捕获的图像)。 你应该记住，例如，一些设备不能处理1:1比例(平方图像)的尺寸-实际上 Nexus 5就是其中之一。 所以要做好处理位图操作的准备。

- 后/前置摄像头
  每个设备都可以有前后相机。 当然，也有只有一个背面的设备，但请记住，其中一些只有前置摄像头（即Nexus 7 2012）。 当然，他们每个人都可以有不同的支持的大小列表。

- 不同的支持功能 

  有些设备不支持自动对焦，另一个不支持缩放。 在我们开始使用任何功能之前，你应该检查它是否存在。

- 处理相机操作是非常费力的。

  你应该记住，图像(特别是那些在大型分辨率分辨率下拍摄的图片)是非常需要通过设备来维护的。 图像处理不是最轻的事情，所以你应该关注内存管理和主 UI 线程之外的计算。 另一个负面因素是，在调用之后，相机还没有准备好——有时候你应该等待初始化。

根据上面描述的一些观点，我决定使用外部库来处理相机操作，而不是编写自己的实现。 我选择了 [CWAC-Camera](https://github.com/commonsguy/cwac-camera) 库，它对于实现来说也有点复杂，但它需要处理大量的边缘情况。 [即使作者打算完全重写它](http://commonsware.com/blog/2014/12/01/my-mistakes-cwac-camera.html)，在我看来也没有更好的起点来处理照片 / 视频捕捉。

在我们开始之前最后一件事。 在 InstaMaterial 应用程序中的相机实现是非常有限的。 图片文件处理和捕获图像显示没有逻辑可言，这是非常棘手的。 在这个代码准备投入生产之前，还有很多工作要做(特别是使用位图裁剪，选择合适的图像大小等等)。 但我相信，对于更复杂的项目来说，这可能是一个很好的合作伙伴。

## 准备
让我们一如既往地增加一些资源和不那么重要的模板。 还有一些库已经更新。 从现在开始，项目也有了新的结构:
![|center](http://frogermcs.github.io/images/9/project_structure.png)
不要把它当作最终的参考，但是在真实的项目中，这是我最喜欢的结构，其中 ui / logic / data 将树分开（另一个选项是通过功能来指定树，但是在较小的项目中可以是多余的）。

下面是 commit 列表:

- [libraries update](https://github.com/frogermcs/InstaMaterial/commit/cabb984c918e11fddb333bf4e5a2593020f7d633)
- [project structure update](https://github.com/frogermcs/InstaMaterial/commit/9c62d60682990ba549c3fe72b82c956088a61188)
- [new required resources](https://github.com/frogermcs/InstaMaterial/commit/36381c47e47d4da1ae3ce32702fba598fd96346f)

## CWAC-Camera 配置
相机库的安装非常简单。我们所要做的就是在 `<project> `/build.gradle 文件中添加新的 Maven 仓库：
```groovy
repositories {
    maven {
        url "https://repo.commonsware.com.s3.amazonaws.com"
    }
}
```
以及 `<project>` /app/build.gradle 中的新依赖项：
```groovy
dependencies {
	//...
    compile 'com.commonsware.cwac:camera:0.6.12'
}
```
最后，我们必须在 AndroidManifest.xml 文件中添加新的权限：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="io.github.froger.instamaterial">

    <!--...-->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

    <uses-feature
        android:name="android.hardware.camera"
        android:required="true" />
    <uses-feature
        android:name="android.hardware.camera.front"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.camera.autofocus"
        android:required="false" />

    <application
        android:name=".InstaMaterialApplication"
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme">
        
        <!--...-->
        
    </application>

</manifest>
```

## 照片捕捉屏幕
现在我们可以开始实现捕捉屏幕。 我考虑了两种实现方式——两种不同的拍照和编辑活动，或者一个与这些状态的活动。 在我看来，第一个选项更正确(可能我会在生产代码中使用它) ，但最后我选择了第二种。 为什么？ 因为没有简单的方法(但这并不是不可能的!) 从摄像机预览到照片预览的过渡，不需要太多的布局。 默认情况下，相机保存外部存储的照片，另一个活动应该从那个地方读出来。 不幸的是，它会产生一个小延迟，因此我们必须找出一些解决方案，以便在两个活动之间传递位图(比通过 Intent 的 bundle 发送位图或将位图保存为静态字段更复杂)。

我们从介绍动画开始。 对于活动过渡，我们可以使用 RevealBackgroundView 在[之前的文章](http://frogermcs.github.io/InstaMaterial-concept-part-6-user-profile/#custom-revealbackgroundview)中介绍过。 我们必须再次通过起始位置(这次从 FAB 按钮中获取)。 唯一的区别是我们必须添加新的样式，在 TakePhotoActivity 中隐藏状态栏：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- styles.xml-->
<resources>

    <!--...-->

    <style name="AppTheme.TransparentActivity.FullScreen">
        <item name="android:windowFullscreen">true</item>
    </style>

    <!--...-->

</resources>
```
并将其设置在 AndroidManifest 文件中：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="io.github.froger.instamaterial">

    <!--...-->
    
        <activity
            android:name=".ui.activity.TakePhotoActivity"
            android:screenOrientation="portrait"
            android:theme="@style/AppTheme.TransparentActivity.FullScreen" />
    
    <!--...-->

</manifest>
```
提交所有描述的更改可在[此处](https://github.com/frogermcs/InstaMaterial/commit/e19395ef14a0a29a558a43d36de050958351f99c)获得。

## TakePhotoActivity 布局
现在让我们准备捕捉屏幕的布局。 正如我所说，它应该处理两个国家 - 捕获和配置照片。 这就是为什么我们应该实施元素转换。 为此我们将使用 ViewSwitcher，它可以处理他的孩子之间的动画（在我们的项目中，我们使用 TextSwitcher 来实现类似于扩展 ViewSwitcher 类的动画）。

在我们开始布局实现之前，我们应该为捕获按钮准备背景：
![|center](http://frogermcs.github.io/images/9/btn_capture.png)
和捕获选项：
![|center](http://frogermcs.github.io/images/9/btn_capture_options.png)
首先可以在 .xml 文件中创建。我们可以使用两个圆圈层叠列表和顶部圆形笔划：
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--btn_capture.xml-->
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item>
        <shape android:shape="oval">
            <solid android:color="#458dca" />
        </shape>
    </item>
    <item
        android:bottom="16dp"
        android:left="16dp"
        android:right="16dp"
        android:top="16dp">
        <shape android:shape="oval">
            <solid android:color="#529bd8" />
        </shape>
    </item>
    <item>
        <shape android:shape="oval">
            <stroke
                android:width="2dp"
                android:color="#ffffff"
                android:dashWidth="0dp" />
        </shape>
    </item>
</layer-list>
```
第二个更简单 - 只是简单的圆形画笔（内部图像在项目资源中提供）。
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--btn_capture_options.xml-->
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="oval">
    <stroke
        android:width="1dp"
        android:color="#ffffff"
        android:dashWidth="0dp" />
</shape>
```
请记住，他们都不处理 onClick 和启用/禁用状态。你可以自己做。 😄

现在我们有了 TakePhotoActivity 布局所需的所有元素。实现很简单，但有点复杂。这就是为什么我不显示源代码。这里是它的布局的可视化:

![](http://frogermcs.github.io/images/9/take_photo_layout.png)
顶部和底部面板从屏幕的右侧滑动（感谢 ViewSwitcher 父项）。拍摄照片时，拍摄的图像（右侧的白色方块）出现在 CameraView 的顶部。

这里是 .xml 文件的[完整源代码](https://github.com/frogermcs/InstaMaterial/blob/884b6b3219fc8175a1db364135926efd482bf2c4/app/src/main/res/layout/activity_take_photo.xml)。

## 照片捕捉视图实现
最后，我们可以实现最重要的事情——拍照。 简而言之，这里列出了一系列要求:

- 介绍动画（在后台显示动画之后立即滑入顶部和底部面板）
- 从相机捕捉照片
- 两种状态之间的动画 - 捕捉和编辑照片

第一点非常简单。只需隐藏 Acitivity start 的顶部和底部面板，并在背景显示动画结束时显示它们：
```java
public class TakePhotoActivity extends BaseActivity implements RevealBackgroundView.OnStateChangeListener {

    @InjectView(R.id.vUpperPanel)
    ViewSwitcher vUpperPanel;
    @InjectView(R.id.vLowerPanel)
    ViewSwitcher vLowerPanel;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        //...
        
        vUpperPanel.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
            @Override
            public boolean onPreDraw() {
                vUpperPanel.getViewTreeObserver().removeOnPreDrawListener(this);
                vUpperPanel.setTranslationY(-vUpperPanel.getHeight());
                vLowerPanel.setTranslationY(vLowerPanel.getHeight());
                return true;
            }
        });
    }

    @Override
    public void onStateChange(int state) {
        if (RevealBackgroundView.STATE_FINISHED == state) {
            vTakePhotoRoot.setVisibility(View.VISIBLE);
            startIntroAnimation();
        } else {
            vTakePhotoRoot.setVisibility(View.INVISIBLE);
        }
    }

    private void startIntroAnimation() {
        vUpperPanel.animate().translationY(0).setDuration(400).setInterpolator(DECELERATE_INTERPOLATOR);
        vLowerPanel.animate().translationY(0).setDuration(400).setInterpolator(DECELERATE_INTERPOLATOR).start();
    }

    //...
}
```

## 照片捕捉
现在我们必须配置 CameraView。根据[CWAC-Camera文档](https://github.com/commonsguy/cwac-camera)，我们必须将这个视图的活动生命周期（onResume () 和 onPause () 方法）结合起来：
```java
@Override
protected void onResume() {
    super.onResume();
    cameraView.onResume();
}

@Override
protected void onPause() {
    super.onPause();
    cameraView.onPause();
}
```
如果我们在布局中使用 CameraView（而不是使用 CameraFragment），我们的 Activity 必须实现CameraHostProvider 接口。 它提供了应该返回 CameraHost 实现的 getCameraHost () 方法。 这个接口可以为我们的相机配置重新设置。 为了我们的需要，我们创建了非常简单的 MyCameraHost 内部类：

```java
class MyCameraHost extends SimpleCameraHost {

    private Camera.Size previewSize;

    public MyCameraHost(Context ctxt) {
        super(ctxt);
    }

    @Override
    public boolean useFullBleedPreview() {
        return true;
    }

    @Override
    public Camera.Size getPictureSize(PictureTransaction xact, Camera.Parameters parameters) {
        return previewSize;
    }

    @Override
    public Camera.Parameters adjustPreviewParameters(Camera.Parameters parameters) {
        Camera.Parameters parameters1 = super.adjustPreviewParameters(parameters);
        previewSize = parameters1.getPreviewSize();
        return parameters1;
    }

    @Override
    public void saveImage(PictureTransaction xact, final Bitmap bitmap) {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                showTakenPicture(bitmap);
            }
        });
    }
}
```
简而言之，将相同的大小预览和捕获的图像需要注意（由于所拍摄的图像不会被拉伸，比预览）。 useFullBleedPreview () 与 ImageView 中的 centerCrop 缩放类型几乎相同。 因此，如果 CameraView 预览具有与视图布局参数不同的大小（在我们的示例中它具有！），预览将被裁剪并填充整个可用区域。
 saveImage（）方法在捕获操作结束时被调用（所以即使我们修改了给定的位图，也不会影响保存在手机内存上的文件）。 但请记住，只有当我们用适当的参数调用 takePicture () 时，才会调用 saveImage () 方法。

我不会深入 CameraHost 实施的其余部分。只需检查 CWAC-Camera 文档以获取更多信息。

我们现在要做的就是调用 takePicture () 方法。 它有两个布尔参数：needBitmap 和 needByteArray。 如果第一个设置为 true，则将在我们的 CameraHost 实现中调用 saveImage（PictureTransaction xact，final Bitmap）。 第二次调用 saveImage（PictureTransaction xact，byte [] image）。

当然，如果我们想看到快门效应，我们必须自己实现。这是它在我们的项目中的样子：
```java
public class TakePhotoActivity extends BaseActivity implements RevealBackgroundView.OnStateChangeListener,
        CameraHostProvider {

    //...
    
    @OnClick(R.id.btnTakePhoto)
    public void onTakePhotoClick() {
        btnTakePhoto.setEnabled(false);
        cameraView.takePicture(true, false);
        animateShutter();
    }

    private void animateShutter() {
        vShutter.setVisibility(View.VISIBLE);
        vShutter.setAlpha(0.f);

        ObjectAnimator alphaInAnim = ObjectAnimator.ofFloat(vShutter, "alpha", 0f, 0.8f);
        alphaInAnim.setDuration(100);
        alphaInAnim.setStartDelay(100);
        alphaInAnim.setInterpolator(ACCELERATE_INTERPOLATOR);

        ObjectAnimator alphaOutAnim = ObjectAnimator.ofFloat(vShutter, "alpha", 0.8f, 0f);
        alphaOutAnim.setDuration(200);
        alphaOutAnim.setInterpolator(DECELERATE_INTERPOLATOR);

        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.playSequentially(alphaInAnim, alphaOutAnim);
        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                vShutter.setVisibility(View.GONE);
            }
        });
        animatorSet.start();
    }

    //...
}
```
而最后一件事 - 我们应该处理拍照并显示。 在我们的例子中，它以这种方式工作：在 CameraView 之上，我们隐藏了 ImageView。 如果只有我们从 saveImage () 方法获得新的位图，我们把它放到 ImageView 中，并显示在 CameraView 的顶部。 按下后，ImageView 被隐藏，用户可以再次看到 CameraView。 以下是我们代码中的外观：
```java
public class TakePhotoActivity extends BaseActivity implements RevealBackgroundView.OnStateChangeListener,
        CameraHostProvider {
    
    //...

    private void showTakenPicture(Bitmap bitmap) {
        vUpperPanel.showNext();
        vLowerPanel.showNext();
        ivTakenPhoto.setImageBitmap(bitmap);
        updateState(STATE_SETUP_PHOTO);
    }

    @Override
    public void onBackPressed() {
        if (currentState == STATE_SETUP_PHOTO) {
            btnTakePhoto.setEnabled(true);
            vUpperPanel.showNext();
            vLowerPanel.showNext();
            updateState(STATE_TAKE_PHOTO);
        } else {
            super.onBackPressed();
        }
    }

    private void updateState(int state) {
        currentState = state;
        if (currentState == STATE_TAKE_PHOTO) {
            vUpperPanel.setInAnimation(this, R.anim.slide_in_from_right);
            vLowerPanel.setInAnimation(this, R.anim.slide_in_from_right);
            vUpperPanel.setOutAnimation(this, R.anim.slide_out_to_left);
            vLowerPanel.setOutAnimation(this, R.anim.slide_out_to_left);
            new Handler().postDelayed(new Runnable() {
                @Override
                public void run() {
                    ivTakenPhoto.setVisibility(View.GONE);
                }
            }, 400);
        } else if (currentState == STATE_SETUP_PHOTO) {
            vUpperPanel.setInAnimation(this, R.anim.slide_in_from_left);
            vLowerPanel.setInAnimation(this, R.anim.slide_in_from_left);
            vUpperPanel.setOutAnimation(this, R.anim.slide_out_to_right);
            vLowerPanel.setOutAnimation(this, R.anim.slide_out_to_right);
            ivTakenPhoto.setVisibility(View.VISIBLE);
        }
    }
}
```
这里是 [TakePhotoActivity](https://github.com/frogermcs/InstaMaterial/blob/29dfdc0ae8ff74d3508f9ae84bbec8cb22c80223/app/src/main/java/io/github/froger/instamaterial/ui/activity/TakePhotoActivity.java) 的完整源代码。

而今天就是这样。谢谢大家的阅读！ 😄

## 示例代码
 所描述项目的完整源代码在 Github [存储库](https://github.com/frogermcs/InstaMaterial)上可用。




