# 本地广播，更少的开销和更好的安全

> 原文 (Medium)：[Local Broadcast, less overhead and secure in Android](https://android.jlelse.eu/local-broadcast-less-overhead-and-secure-in-android-cfa343bb05be)
>
> 作者：[Ankit Sinhal](https://android.jlelse.eu/)

[TOC]

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo6afkw1j30m80eq756.jpg)

广播接收器是一个 Android 组件，它允许你发送或接收 Android 系统或应用程序事件。一旦发生事件，所有已注册的应用程序都会被安卓运行时通知。 

它与[发布 - 订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)设计模式类似，并用于异步进程间通讯。 

例如，应用程序可以注册各种系统事件，例如启动完成或电池电量低，和安卓系统发送广播当特定的事件发生。 任何应用程序也可以创建自己的自定义广播 

## 广播基础

我们来讨论一下广播接收机的一些基本概念。

### 注册广播

有两种方式注册广播接收器

**清单声明（静态）**：通过这个接收器可以通过 AndroidManifest.xml 文件注册。

**上下文注册（动态）**：通过这个注册一个接收器动态地通过 Context.registerReceiver ( ) 方法进行接收。



### 接收广播

为了能够接收广播，应用程序必须扩展 [BroadcastReceiver](https://developer.android.com/reference/android/content/BroadcastReceiver.html) 抽象类并覆盖其 onReceive ( ) 方法。

如果广播接收者注册的事件发生，则接收者的 onReceive ( ) 方法由 Android 系统调用。



## 全局广播的问题

当你想要在不同的应用程序之间发送或接收数据时，最好使用广播接收器。但是，如果通讯仅限于你的应用程序，那么使用全局广播并不好。

在这种情况下，Android 使用 [LocalBroadcastManager](https://developer.android.com/reference/android/support/v4/content/LocalBroadcastManager.html) 类提供本地广播。 使用全局广播，任何其他应用程序也可以发送和接收来自我们的应用程序的广播消息。 这可能是我们应用程序的一个严重的安全线程。 全局广播也是全系统发送的，所以不是性能高效的。



## 什么是 LocalBroadcastManager？

由于显而易见的原因，全局广播绝不能含有敏感信息。但是，你可以使用 [LocalBroadcastManager](https://developer.android.com/reference/android/support/v4/content/LocalBroadcastManager.html) 类（这是 Android 支持库的一部分）在本地广播这些信息。

LocalBroadcastManager 更高效，因为它不需要进程间通信。

下面是它的一些好处：

- 广播数据不会离开你的应用程序，所以不必担心泄漏私人数据。
- 其他应用程序不可能将这些广播发送到你的应用程序，所以你不必担心有可能利用的安全漏洞。
- 这比通过系统发送全局广播更有效率。
- 没有全系统广播的开销。

## 实现

在最新版本的 Android Studio 中不需要额外的支持库依赖项。但是，如果要在旧项目中实现本地广播，则需要在应用程序模块的 build.gradle文件中添加以下依赖项：

```java
compile ‘com.android.support:support-v4:23.4.0’
```

创建一个 LocalBroadcastManager 的新实例

```java
LocalBroadcastManager localBroadcastManager = LocalBroadcastManager.getInstance(context);
```

你现在可以使用 sendBroadcast ( ) 方法发送本地广播

```java
// Create intent with action
Intent localIntent = new Intent(“CUSTOM_ACTION”);
// Send local broadcast
localBroadcastManager.sendBroadcast(localIntent);
```

现在创建一个可以响应本地广播操作的广播接收器: 

```java
private BroadcastReceiver listener = new BroadcastReceiver() {
@Override
    public void onReceive( Context context, Intent intent ) {
        String data = intent.getStringExtra(“DATA”);
        Log.d( “Received data : “, data);
    }
};
```

动态注册的接收者必须在不再需要的情况下注销，例如：

```java
localBroadcastManager.unregisterReceiver(myBroadcastReceiver);
```

你可以找到参考代码从 [GitHub](https://github.com/AnkitSinhal/LocalBroadcastManagerSample) 来实现本地广播接收器。在示例代码中，我创建了一个 IntentService，它将广播当前日期，并由相同应用程序的 Activity 接收。

## 如何保证广播安全

### 限制你的应用接收广播

- 在注册广播接收机时指定一个权限参数，只有请求许可的广播机构才能向接收机发送一个意图。

例如，接收应用程序在接收器中具有声明的 SEND_SMS 权限，如下所示：

```java
<receiver android:name=”.MyBroadcastReceiver”
          android:permission=”android.permission.SEND_SMS”>
    <intent-filter>
    <action android:name=”android.intent.action.AIRPLANE_MODE”/>
    </intent-filter>
</receiver>
```

- 在清单中将 android:exported 属性设置为 “ false ”。这限制接收来自应用程序之外的来源的广播。
- 使用 LocalBroadcastManager 仅限本地广播。

### 控制广播的接收器

- 你可以在发送广播时指定权限，然后只有请求该权限的接收者才能接收广播。例如，下面的代码发送一个广播：

  ```java
  sendBroadcast(new Intent(“com.example.NOTIFY”), Manifest.permission.SEND_SMS);
  ```

- 在 Android 4.0 及更高版本中，你可以在发送广播时使用 setPackage (String ) 指定一个包。系统将广播限制为与包匹配的一组应用程序。

- 用 LocalBroadcastManager 发送本地广播。

## 综述

希望你了解全局和本地广播及其安全考虑。 为了提高系统的性能 [Android O](https://android.jlelse.eu/android-o-impact-on-running-apps-and-developer-viewpoint-b9f23047f306) 改变了广播接收机的注册。 从 Android O 中，你不能在你的应用程序清单中注册隐式广播（很少有例外），但是应用程序可以继续在其清单或运行时注册显式广播。 在 [Android 官方文档](https://developer.android.com/guide/components/broadcasts.html)上阅读更多广播接收器。