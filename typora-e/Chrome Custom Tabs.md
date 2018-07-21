> 原文 ：[Chrome Custom Tabs](https://developer.chrome.com/multidevice/android/customtabs)
>
> 译文 ：[Chrome Custom Tabs最佳实践](https://blog.csdn.net/just_sanpark/article/details/52249051)
>
> 译者 ：[just_sanpark](https://blog.csdn.net/just_sanpark)

##### 本文是 [Chrome Custom Tabs](https://developer.chrome.com/multidevice/android/customtabs) 官方文档的按照自己的理解的整理译文

# 什么是Chrome Custom Tabs（CCT）?

>当开发者是去打开一个URL时候往往有两种方式：WebView和打开浏览器。目前这两种方式皆有不足，WebView不与浏览器共享状态，维护开销大；打开浏览器是一个重量级的操作并且浏览器不可定制化。 此时谷歌爸爸给了我们新的选择，CCT让我们在网络体验上有了很多的控制，使本地和Web内容更加无缝之间的转换，而无需采取WebView的方式。

CCT允许应用定制浏览器的外观和样式,如下：

- **Toolbar颜色**
- **打开关闭时的切换动画**
- **添加Toolbar的Actions,添加Overflow Menu 和底部Toolbar**

CCT还允许开发人员预启动Chrome和更快的内容预抓取加载。 
![](https://ws2.sinaimg.cn/large/006tKfTcgy1ftbtac69ahg30hs0a0apj.gif)

现在可以在Github上测试这些 [sample](https://github.com/GoogleChrome/custom-tabs-client).

------

# 怎么选择Chrome Custom Tabs vs WebView?

WebView是对于展示自己域名下或本地的网页内容很好的一种解决方案，因为你可能会去网页内容做很多自定义的内容。而如果引导用户去第三方的网址时，建议您使用CCT，理由如下：

- **易于实现。无需建代码来管理请求，权限授予和Cookie** 

- **UI上的自定义上文已提到**

- **导航：浏览器提供了一个可回调的外部导航模块**

- 性能优化:

  - **在后台浏览器的预加热，避免从应用程序窃取资源。**

  - **提前提供一个合适的URL到浏览器，它可以进行投机性的工作，加快页面加载时间**

- **生命周期管理：通过提高“foreground”的重要级来防止CCT防止被系统被消灭**

- **共享Cookie和权限模块，用户不必再次登录到他们已经连接或者重新授权他们已经授权的网站**

- **使用CCT用户依旧可以从Data Saver(Chrome插件：帮助用户节省浏览时的数据使用量)获益**

- **更好的完成设备自动同步**

- **简单的定制模式**

- **更快速的返回到本地应用**

- **You want to use the latest browser implementations on devices pre-Lollipop (auto updating WebView) instead of older WebViews.**

------

# 使用条件？

需要安装Chrome45或以上版本，支持的Android版本（Jellybean(4.1)以上）。

------

# 使用向导

一个完整的例子可在<https://github.com/GoogleChrome/custom-tabs-client> 找到。它包含了使用可重复的类来定制UI，连接到后台服务，并处理应用程序和CCT的生命周期。

首先你需要添加CCT的依赖 [Custom Tabs Support Library ](https://developer.android.com/topic/libraries/support-library/features.html#custom-tabs)在你的build.gradle文件里

```groovy
dependencies { 
… 
compile‘com.android.support:customtabs:24.2.0’ 
}
```

当你完成添加依赖你可能有两个配置需要自定义：

- *自定义选项卡的UI和交互*
- *使页面加载速度更快，并使应用程序保活*

UI的自定义你可能会使用到 [CustomTabsIntent](https://developer.android.com/reference/android/support/customtabs/CustomTabsIntent.html) 和 [CustomTabsIntent.Builder](https://developer.android.com/reference/android/support/customtabs/CustomTabsIntent.Builder.html); 性能的改进通过使用[CustomTabsClient](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html) 来连接Custom Tabs service,预加热Chrome以及让它知道那个URL将被打开.

### 快速使用

```java
String url = ¨https://paul.kinlan.me/¨;
CustomTabsIntent.Builder builder = new CustomTabsIntent.Builder();
CustomTabsIntent customTabsIntent = builder.build();
customTabsIntent.launchUrl(this, Uri.parse(url));
```

### 设置Toolbar颜色

```java
builder.setToolbarColor(colorInt);
```

### 自定义Action Buttons

作为开发者你将拥有呈现给用户的Chrome标签内的操作按钮的所有控制。

在大多数情况下，这将是一个主要的作用，如分享，或者你的用户将执行另一种常见的活动。

用户点击操作按钮调用的PendingIntent通过Bundle由Chrome传递过来。该图标是高度最好是24dp和24-48dp的宽度。

```java
// Adds an Action Button to the Toolbar.
// 'icon' is a Bitmap to be used as the image source for the
// action button.


// 'description' is a String be used as an accessible description for the button.

// 'pendingIntent is a PendingIntent to launch when the action button
// or menu item was tapped. Chrome will be calling PendingIntent#send() on
// taps after adding the url as data. The client app can call
// Intent#getDataString() to get the url.

// 'tint' is a boolean that defines if the Action Button should be tinted.

builder.setActionButton(icon, description, pendingIntent, tint);
```

### 自定义菜单

Chrome浏览器CCT将始终会有前进，页面信息，刷新三个图标,菜单上会有“查找页面”和“在浏览器中打开”两项。作为开发者你还可以添加多达5个菜单项。

菜单项是通过调用 [CustomTabsIntent.Builder#addMenuItem](https://developer.android.com/reference/android/support/customtabs/CustomTabsIntent.Builder.html#addMenuItem%28java.lang.String,%20android.app.PendingIntent%29) 和带有标题的PendingIntent，然后Chrome会将用户的行为作为参数传递过来。

```java
builder.addMenuItem(menuItemTitle, menuItemPendingIntent);
```

### 设置切换动画

很多Android应用程序中使用自定义动画来过渡Android上的Activities切换。 CCT并没有什么不同，你可以改变的打开和关闭（当用户按下后退）动画，让他们与你的应用程序的其他部分相一致。

```java
builder.setStartAnimations(this, R.anim.slide_in_right, R.anim.slide_out_left);
builder.setExitAnimations(this, R.anim.slide_in_left, R.anim.slide_out_right);
```

### 预加热Chrome加速加载页面

默认情况下，当[CustomTabsIntent#launchUrl](https://developer.android.com/reference/android/support/customtabs/CustomTabsIntent.html#launchUrl%28android.app.Activity,%20android.net.Uri%29) 被调用时它会启动Chrome和URL，这会占用宝贵的时间和影响切换平滑的感觉。

我们知道用户需要近乎瞬时的体验，所以我们提供一个可以让应用连接并告诉Chrome预加热浏览器和本地组件的服务。同时我们也可以告诉Chrome浏览器你设置的用户将会访问的网址，然后Chrome将会执行一下步骤：

- DNS预解析主域
- DNS预解析最有可能的子资源
- 预连接到目的地，包括HTTPS / TLS。

预加热Chrome的过程如下：

- 使用 [CustomTabsClient#bindCustomTabsService](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#bindCustomTabsService%28android.content.Context,%20java.lang.String,%20android.support.customtabs.CustomTabsServiceConnection%29) 连接到服务
- 一旦服务连接，调用 [CustomTabsClient＃warmup ](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#warmup%28long%29)在后台启动Chrome
- 调用 [CustomTabsClient＃newsession](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#newSession%28android.support.customtabs.CustomTabsCallback%29) 来创建一个新的会话。本次会话是用于所有请求的API
- （可选）当创建一个会话的时候添加参数 [CustomTabsCallback](https://developer.android.com/reference/android/support/customtabs/CustomTabsCallback.html)，可以知道页面是否加载完成
- 通过 [CustomTabsSession＃mayLaunchUrl](https://developer.android.com/reference/android/support/customtabs/CustomTabsSession.html#mayLaunchUrl%28android.net.Uri,%20android.os.Bundle,%20java.util.List%3Candroid.os.Bundle%3E%29) 加载一个页面可以告诉Chrome那个页面是用户最有可能去加载的
- 调用 [CustomTabsIntent.Builder](https://developer.android.com/reference/android/support/customtabs/CustomTabsIntent.Builder.html) 构造函数传递创建的 [CustomTabsSession](https://developer.android.com/reference/android/support/customtabs/CustomTabsSession.html)

### 连接到Chrome Service

[CustomTabsClient#bindCustomTabsService](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#bindCustomTabsService%28android.content.Context,%20java.lang.String,%20android.support.customtabs.CustomTabsServiceConnection%29) 简化了连接到Custom Tabs service的流程，创建  [CustomTabsServiceConnection](http://developer.android.com/reference/android/support/customtabs/CustomTabsServiceConnection.html)，并使用 [onCustomTabsServiceConnected](https://developer.android.com/reference/android/support/customtabs/CustomTabsServiceConnection.html#onCustomTabsServiceConnected%28android.content.ComponentName,%20android.support.customtabs.CustomTabsClient%29) 得到 [CustomTabsClient](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html)的一个实例。操作如下

```java
// Package name for the Chrome channel the client wants to connect to. This
// depends on the channel name.
// Stable = com.android.chrome
// Beta = com.chrome.beta
// Dev = com.chrome.dev
public static final String CUSTOM_TAB_PACKAGE_NAME = "com.android.chrome";  // Change when in stable

CustomTabsServiceConnection connection = new CustomTabsServiceConnection() {
    @Override
    public void onCustomTabsServiceConnected(ComponentName name, CustomTabsClient client) {
        mCustomTabsClient = client;
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
};
boolean ok = CustomTabsClient.bindCustomTabsService(this, mPackageNameToBind, connection);
```

**Warm up the Browser Process** 
[boolean warmup(long flags)](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#warmup%28long%29) 
返回 true 代表成功.

预加热浏览器进程和加载库文件。此操作是异步的，返回值表示请求是否被接受。多次调用成功亦返回true

**创建Tab session** 
[boolean newSession(CustomTabsCallback callback)](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#newSession%28android.support.customtabs.CustomTabsCallback%29)

会话被用于随后与CustomTabsCallback相连接的回调，并产生相互的标签。这里提供的回调与创建会话相关联。所创建的会话的任何更新（见下面的自定义选项卡回调）通过该回调接收。返回值表示会话是否已成功创建。多个调用相同CustomTabsCallback或者空值将返回false。

**设置可能打开的URL** 
[boolean mayLaunchUrl(Uri url, Bundle extras, List otherLikelyBundles)](https://developer.android.com/reference/android/support/customtabs/CustomTabsSession.html#mayLaunchUrl%28android.net.Uri,%20android.os.Bundle,%20java.util.List%3Candroid.os.Bundle%3E%29) 
该方法告诉浏览器一个未来可能加载的URL。[warmup()](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#warmup%28long%29) 首先会被调用，最有可能的URL必须首先指定。（可选）可以提供其它可能的URL列表。它们被视为不太可能相比第一个，并且以降序进行排序。这些额外的URL可能会被忽略。这种方法以前所有的请求都将被deprioritized。返回值表示操作是否成功完成。

**自定义选项卡Connection Callback** 
[void onNavigationEvent(int navigationEvent, Bundle extras)](https://developer.android.com/reference/android/support/customtabs/CustomTabsCallback.html#onNavigationEvent%28int,%20android.os.Bundle%29) 
自定义选项卡有用户行为发生时将被调用。该 `navigationEvent int`是定义的6种页面状态。请参阅下面的详细信息。

```java
/**
* Sent when the tab has started loading a page.
*/
public static final int NAVIGATION_STARTED = 1;

/**
* Sent when the tab has finished loading a page.
*/
public static final int NAVIGATION_FINISHED = 2;

/**
* Sent when the tab couldn't finish loading due to a failure.
*/
public static final int NAVIGATION_FAILED = 3;

/**
* Sent when loading was aborted by a user action before it finishes like clicking on a link
* or refreshing the page.
*/
public static final int NAVIGATION_ABORTED = 4;

/**
* Sent when the tab becomes visible.
*/
public static final int TAB_SHOWN = 5;

/**
* Sent when the tab becomes hidden.
*/
public static final int TAB_HIDDEN = 6;
```

## 如果用户没有安装了最新版本的Chrome会发生什么？

自定义选项卡使用带有附加功能键的ACTION_VIEW意图定制UI。这意味着，在默认情况下页面将会在系统浏览器，或用户的默认浏览器。

如果用户安装了Chrome浏览器并且设置默认的浏览器，它会自动拿起额外设备和呈现的自定义UI。另外，也可以为其他浏览器使用意图额外提供一个类似的定制接口。

## 如何检查Chrome是否支持CCT?

所有版本的Chrome都支持CCT暴露出来的一个服务。要检查是否支持自定义选项卡，尝试绑定到该服务。如果成功，则可以安全使用自定义选项卡。

# 最佳实践

#### **Connect to the Custom Tabs service and call warmup()**

通过这样的方式你最多可以节省700ms，700ms差不多是界定卡与不卡的时间。在Activity的 [onStart()](http://developer.android.com/reference/android/app/Activity.html#onStart%28%29) 去连接Custom Tab service, 连接后调用 [warmup()](https://developer.android.com/reference/android/support/customtabs/CustomTabsClient.html#warmup%28long%29). 这一操作将作为低优先级的进程，**这意味着它不会对你的应用程序性能的负面影响**，但加载链接时会给出一个大的性能提升。

#### **Pre-render content**

预渲染将使外部内容瞬间打开。所以，如果你的用户至少点击链接的**50％**的可能性，调用[mayLaunchUrl（）](https://developer.android.com/reference/android/support/customtabs/CustomTabsSession.html#mayLaunchUrl%28android.net.Uri,%20android.os.Bundle,%20java.util.List%3Candroid.os.Bundle%3E%29)这个方法会提前下载并渲染网页内容，但不可避免的会有一点流量和电量的消耗。如果用户正在使用收费的数据流量，或者手机电量不足，那么这个方法不会生效。所以我们**完全不用自己考虑性能优化**。

#### **备选方案**

如果用户的手机上没有安装Chrome，那么打开默认浏览器可能并不是最好的用户体验。所以如果在bindService那一步失败了，无论是打开默认浏览器还是WebView，选择一个你认为最好的备选方案。

#### **Add your app as the referrer**

很多网站都会统计自己的流量是从哪儿来的，所以最好告诉他们是你的帅气APP给他们带来了流量：

```java
intent.putExtra(Intent.EXTRA_REFERRER, 
        Uri.parse(Intent.URI_ANDROID_APP_SCHEME + "//" + context.getPackageName()));
```

#### **自定义动画**

自定义动画将让你的应用程序将网站内容更平滑的过渡。确保完成动画和动画开始是相对应的，这会帮助用户了解他们已经回到了应用程序。

```java
 //Setting custom enter/exit animations
    CustomTabsIntent.Builder intentBuilder = new CustomTabsIntent.Builder();
    intentBuilder.setStartAnimations(this, R.anim.slide_in_right, R.anim.slide_out_left);
    intentBuilder.setExitAnimations(this, android.R.anim.slide_in_left,
        android.R.anim.slide_out_right);

//Open the Custom Tab        
    intentBuilder.build().launchUrl(context, Uri.parse("https://developer.chrome.com/"));        
```

#### **为ActionButton添加一个Icon**

添加ActionButton可以让用户更多的了解应用有那些功能。你可以创建描述该操作的文本的位图，如果没有一个很好的图标来表示你的操作按钮执行的操作，记住位图的最大尺寸为24dp高x宽48dp。

```java
    String shareLabel = getString(R.string.label_action_share);
    Bitmap icon = BitmapFactory.decodeResource(getResources(),
            android.R.drawable.ic_menu_share);

    //Create a PendingIntent to your BroadCastReceiver implementation
    Intent actionIntent = new Intent(
            this.getApplicationContext(), ShareBroadcastReceiver.class);
    PendingIntent pendingIntent = 
            PendingIntent.getBroadcast(getApplicationContext(), 0, actionIntent, 0);            

    //Set the pendingIntent as the action to be performed when the button is clicked.            
    intentBuilder.setActionButton(icon, shareLabel, pendingIntent);
```

#### **多个浏览器的选择**

用户可以安装多个支持CCT的浏览器。如果有一个以上的浏览器支持自定义选项卡或者其中没有一个是首选浏览器，那么可以询问用户想如何打开链接.另外添加一个可以选择默认浏览器的选项，这样可以让用户自由的去选择是否使用CCT.

```java
    /**
     * Returns a list of packages that support Custom Tabs.
     */ 
    public static ArrayList getCustomTabsPackages(Context context) {
        PackageManager pm = context.getPackageManager();
        // Get default VIEW intent handler.
        Intent activityIntent = new Intent(Intent.ACTION_VIEW,  Uri.parse("http://www.example.com"));

        // Get all apps that can handle VIEW intents.
        List resolvedActivityList = pm.queryIntentActivities(activityIntent, 0);
        ArrayList packagesSupportingCustomTabs = new ArrayList<>();
        for (ResolveInfo info : resolvedActivityList) {
            Intent serviceIntent = new Intent();
            serviceIntent.setAction(ACTION_CUSTOM_TABS_CONNECTION);
            serviceIntent.setPackage(info.activityInfo.packageName);
            // Check if this package also resolves the Custom Tabs service.
            if (pm.resolveService(serviceIntent, 0) != null) {
                packagesSupportingCustomTabs.add(info);
            }
        }
        return packagesSupportingCustomTabs;
    }
```

#### **使用本地应用处理链接**

一些URL可以通过本地应用程序进行处理。如果用户安装了Twitter的应用程序，那么点击链接后更期望在Twitter中打开。所以在应用程序打开一个URL之前，最好先检查本机是否有方法可以代替。

#### **定义Toolbar颜色**

如果你希望用户觉得内容是应用程序的一部分，那么可以设置Toolbar颜色为Primary color。如果你考虑让用户知道该页面已经离开了你的App那么千万不要自定义Toolbar颜色。

#### **添加分享**

大多数情况下用户都会想要分享这个链接但是CCT默认不添加,所以最好自己加上

```java
    //Sharing content from CustomTabs with on a BroadcastReceiver
    public void onReceive(Context context, Intent intent) {
        String url = intent.getDataString();

        if (url != null) {
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_TEXT, url);

            Intent chooserIntent = Intent.createChooser(shareIntent, "Share url");
            chooserIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

            context.startActivity(chooserIntent);
        }
    }
```

#### **定义关闭按钮**

如果你希望用户感觉像CCT是一个对话框可以使用默认的“X”按钮。如果你希望用户感觉到CCT是App的一部分可以使用后退箭头。

```java
    //Setting a custom back button
    CustomTabsIntent.Builder intentBuilder = new CustomTabsIntent.Builder();
    intentBuilder.setCloseButtonIcon(BitmapFactory.decodeResource(
        getResources(), R.drawable.ic_arrow_back));
```

#### **处理内部链接**

当拦截到由android:autoLink或者WebView产生的内部链接点击时，最好让应用程序自己去处理，CCT只处理外部链接。

```java
WebView webView = (WebView)findViewById(R.id.webview);
webView.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        return true;
    }

    @Override
    public void onLoadResource(WebView view, String url) {
        if (url.startsWith("http://www.example.com")) {
            //Handle Internal Link...
        } else {
            //Open Link in a Custom Tab
            Uri uri = Uri.parse(url);
            CustomTabsIntent.Builder intentBuilder =
                    new CustomTabsIntent.Builder(mCustomTabActivityHelper.getSession());
           //Open the Custom Tab        
            intentBuilder.build().launchUrl(context, url));                            
        }
    }
});
```

#### **连击处理**

确保用户点击链接后到打开CCT之间不要超过100ms，否则用户会觉得反应迟钝，并尝试多次点击。当然卡顿是无法避免的，所以在用户多次点击后确保只打开CCT一次。

 

