# Android N：引入升级的通知

> 原文 (Medium)：[Android N: Introducing upgraded Notifications](https://medium.com/exploring-android/android-n-introducing-upgraded-notifications-d4dd98a7ca92)
>
> 作者：[Joe Birch](https://medium.com/@hitherejoe)

[TOC]

![](https://ws2.sinaimg.cn/large/006tNc79gy1fron2dv85xj30m806k0sk.jpg)

在撰写了关于 Android N 的全新 [Picture-in-Picture](https://medium.com/@hitherejoe/android-n-introducing-picture-in-picture-for-android-tv-35f2392fb609#.5xzui44op) 功能之后，我决定深入探讨另一个我们在开发者预览版发布的新功能 - 通知。 Android N 看到了通知 API 的一些很棒的新增功能，所以我们来看看这些新功能以及如何将它们实现到我们的 Android 应用程序中。

为了更好地掌握这些新的通知功能，我创建了 [Notifi](https://github.com/hitherejoe/Notifi)，一个简单的 Android 项目来演示本文中的所有示例。

在安卓系统中，通知已经得到了相当大的升级。 我们现在对我们的应用程序可以使用的通知的外观和感觉有了更多的控制，并且还被赋予了一些新的功能来增强用户在处理这些通知时的体验。 那么，有什么新鲜事？ 好吧，这个更新向我们介绍了: 

- 新的通知系统布局模板意味着，默认情况下，它们更简单、更清洁——使它们感觉更加简洁 
- 捆绑现在意味着通知不再会淹没用户状态，如果需要，可以在一个可扩展的通知组下显示与条相关的通知 
- 使用新的内联回复操作，我们的用户可以对来自通知本身的文本操作做出响应，而不需要打开应用程序，从而分心于他们目前正在从事的工作 
- 自定义视图允许我们明智地设计我们的通知，使其看起来完全符合我们的要求。

这一切听起来都很棒，不是吗？ 所以，让我们继续下去，看看所有这些的实现细节。 

## 通知模板

![](https://ws1.sinaimg.cn/large/006tNc79gy1fron2h2rdhj30m806k0sk.jpg)

Android N 显示了在显示通知时使用了一些新的模板。 这些新的模板重新排列了先前 SDK 版本中存在的组件，使它们看起来更简单更干净。 这些新的模板是自动使用的系统，所以没有必要改变你的代码创建通知。 

就像以前一样，我们可以像这样实现一个简单的通知：

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
        .setSmallIcon(R.drawable.ic_phonelink_ring_primary_24dp)
        .setContentTitle(title)
        .setContentText(message)
        .setLargeIcon(largeIcon)
        .setAutoCancel(true);
```

以前，使用上面的代码（在 N 之前的设备上）显示通知将如下所示：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron2k4ruoj30m803jq3j.jpg)

现在在 Android N 上，新的模板（没有改变上面使用的代码！）意味着我们的通知现在看起来像这样：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron2mfx1pj30m804vt9b.jpg)

这种通知模板的新风格不仅看起来更干净，而且我觉得它有助于将更多的注意力放在视觉方面，并且在通知内容可用的小空间中创建更少的杂波。 同样，在你的代码中没有什么需要改变的——系统将自动使用这个新的外观和感觉默认。 

**注意：**请记住，你可以通过一些样式来设置通知，并通过在 Builder 实例中使用 setColor ( ) 方法来设置通知使用的颜色：

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
    // Other properties
    .setColor(ContextCompat.getColor(context, R.color.primary));
```

使用这将设置通知图标和标题的颜色，以及你添加到通知中的任何操作。

![记得给你的通知一些颜色！](https://ws3.sinaimg.cn/large/006tNc79gy1fron2q4ev9j30m807v3z6.jpg)

## 捆绑式通知

![](https://ws2.sinaimg.cn/large/006tNc79gy1fron2tac5jj30m80hm0sq.jpg)

有了 Android n，我们现在可以捆绑我们的通知——这些组的相关通知项，这样我们就可以在一个通知头下显示多个通知。 当你在应用程序中发送多个相关的通知时，你应该确保将它们分组，以避免用通知淹没用户状态栏。 这对用户来说很烦人，所以对相关的通知进行分组可以帮助提高用户的体验。 

因此，为了实现捆绑式通知，我们必须首先将其中一个通知设置为"组摘要"通知，这个组摘要通知将不会出现在通知堆栈中，而是在扩展之前显示在设备上的唯一通知，作为通知堆栈的标题。 在组中的通知后，将构成此通知堆栈的主体部分。 

**注意：**必须将通知设置为组摘要，否则你的通知将不会作为组的一部分出现。 

接下来，我们可以通过在构建通知实例时使用 setGroup ( ) 方法简单地分配组 ID 字符串来对我们的通知进行分组，如下所示：

```java
NotificationCompat.Builder builderOne = new NotificationCompat.Builder(context)
    // Other properties
    .setGroupSummary(true)
    .setGroup(KEY_NOTIFICATION_GROUP);
    
NotificationCompat.Builder builderTwo = new NotificationCompat.Builder(context)
    // Other properties
    .setGroupSummary(true)
    .setGroup(KEY_NOTIFICATION_GROUP);
    
NotificationCompat.Builder builderThree = new NotificationCompat.Builder(context)
    // Other properties
    .setGroupSummary(true)
    .setGroup(KEY_NOTIFICATION_GROUP);
```

这告诉通知管理器，这些通知都属于同一组，然后系统会将这些通知组合成一个包，这真的很简单！ 这是减少用户状态栏中显示的通知的一个很好的方法，我们现在可以将它们分组在同一个通知下同时查看。 产生如下结果: 

![在Android N中捆绑通知允许我们释放我们的通知抽屉的杂乱](https://ws2.sinaimg.cn/large/006tNc79gy1fron362dk2g30k20h119o.gif)

## 捆绑式通知的行动

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron3d57csj30m80izdfu.jpg)

如果这还不够的话，就像标准通知一样，我们还可以使用包中的通知操作。 首先，这些操作将作为通知的一部分而崩溃，而用户不能看到，但是，用户可以通过点击所需通知上的显示箭头来显示这些操作，如下图所示: 

![Android N中的捆绑通知也可以有单独的操作](https://ws4.sinaimg.cn/large/006tNc79gy1fron3v58hig30k00khnb9.gif)

这些操作与通知的标准方法完全相同，因此不需要额外的工作。 我们需要做的就是创建新的 Action 实例并将它们分配给我们的通知，就像这样: 

```java
PendingIntent archiveIntent = PendingIntent.getActivity(...);
NotificationCompat.Action replyAction = ...;
NotificationCompat.Action archiveAction = ...;

NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
    // Other properties
    .setGroup(KEY_NOTIFICATION_GROUP)
    .addAction(replyAction)
    .addAction(archiveAction);
```

## 直接回复

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron48f1edj30m808cq2t.jpg)

使用 [RemoteInput](http://developer.android.com/reference/android/support/v4/app/RemoteInput.html) API (自从 API 级别4开始我们就已经有了这个 API 了!) 我们现在可以在 Android n 中显示通知，允许用户直接响应通知，而无需打开我们的应用程序。 这是通过显示一个内联的回复操作来完成的，它在我们的通知中显示了一个额外的按钮，允许用户显示文本输入字段。 这是伟大的，因为它允许我们的用户与我们的应用程序进行几乎毫无摩擦的交互，而不必打开它，允许他们从他们当前的任务中保持不分心，并且随心所欲地做出回应。 

当用户决定与内联的回复操作进行交互时，用户提交的文本附加到我们指定的动作意图上，然后发送到我们的应用程序(活动、服务等) ，在那里我们可以检索和处理这些内容。 

## 添加内联操作

所以我们现在对这个新的内嵌回复操作有了一些了解，但是我们如何实现它呢！ 好吧，为了给我们的通知添加一个内联的回复操作，我们首先需要创建一个新的 RemoteInput 实例: 

```java
RemoteInput remoteInput = new RemoteInput.Builder(KEY_TEXT_REPLY)
    .setLabel(LABEL_REPLY)
    .build();
```

RemoteInput 构建器需要使用一个标识键（在我的情况下是 KEY_TEXT_REPLY），我们可以在我们的应用程序中使用它来检索用户在我们的内联回复操作中输入的文本。 在此之后，我们可以简单地设置标签，用于在输入字段处于焦点位置时显示为提示(而且是空的!) . 内联响应操作的提交图标会自动显示在系统提供给我们的通知中。 

接下来，我们需要创建 PendingIntent 实例，当我们的操作被用户交互时，我们希望启动这些实例。 我们这样做，就像我们通常在创建 PendingIntents 时所做的那样: 

```java
PendingIntent replyIntent = PendingIntent.getActivity(context,
        REPLY_INTENT_ID,
        getDirectReplyIntent(context, LABEL_REPLY),
        PendingIntent.FLAG_UPDATE_CURRENT);

PendingIntent archiveIntent = PendingIntent.getActivity(context,
        ARCHIVE_INTENT_ID,
        getDirectReplyIntent(context, LABEL_ARCHIVE),
        PendingIntent.FLAG_UPDATE_CURRENT);
        
private static Intent getDirectReplyIntent(Context context, String label) {
        return MessageActivity.getStartIntent(context)
                .addFlags(Intent.FLAG_INCLUDE_STOPPED_PACKAGES)
                .setAction(REPLY_ACTION)
                .putExtra(CONVERSATION_LABEL, label);
}
```

所以现在我们有了这些 PendingIntents，实际上我们需要利用它们！ 这就是我们的 remoteInput 实例的发挥作用。 我们需要从创建我们的行动开始，所以让我们来看看我们的重新设计行动实例如下: 

```java
NotificationCompat.Action replyAction =
        new NotificationCompat.Action.Builder(R.drawable.ic_reply,
                LABEL_REPLY, replyIntent)
                .addRemoteInput(remoteInput)
                .build();
        
NotificationCompat.Action archiveAction =
        new NotificationCompat.Action.Builder(R.drawable.ic_archive,
                LABEL_ARCHIVE, archiveIntent)
                .build();
```

在这里我们将我们的操作创建为正常，然后使用 addRemoteInput ( ) 方法将 remoteInput 实例附加到它上面。 这就是说，当这个操作被交互时，我们希望在我们的通知中向用户显示内联的回复操作。 一旦用户提交了内联答复，这个内联回复的输入数据将包含在我们的重新 plyintent 意图中。 

在这之后，我们可以简单地将我们的操作添加到 notificationbuilder 下面(同样，这和我们之前的方法一样) ，不需要额外的代码来实现内联的响应操作。 

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
        .setSmallIcon(R.drawable.ic_phonelink_ring_primary_24dp)
        .setContentTitle(title)
        .setContentText(message)
        .setLargeIcon(largeIcon)
        .addAction(replyAction);
        .addAction(archiveAction);
        .setAutoCancel(true);
```

一旦我们实现了以上所述，实现的结果将会给我们下面的结果 : 

![在Android N上显示通知内联回复操作](https://ws1.sinaimg.cn/large/006tNc79gy1fron4h6g38g30ex08ck0k.gif)

## 从内联行动接收数据

因此，我们已经实现了内联的回复操作，它看起来很棒，但是为了有用，我们需要实际检索用户提交的数据。 这很容易做到: 

```java
Bundle bundle = RemoteInput.getResultsFromIntent(intent);
if (remoteInput != null) {
    return bundle.getCharSequence(NotificationUtil.KEY_TEXT_REPLY);
}
```

在这里，我们使用 RemoteInput API 来从意图中检索输入结果。 此时，如果结果存在，我们就可以从 bundle 中检索数据。 在这里我们使用了我们之前用来存储数据的密钥，这是我们在上一节中提到的 KEY_TEXT_REPLY  。 

## 设计内联的回复

我们还可以在构建通知时使用 setColor ( ) 方法来设置我们的内联回复操作的颜色。它看起来不错吗？

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron4llm9yj30m807faal.jpg)

你也可以看到这个设置了我们通知中所有组件的颜色。这很容易实现，就像这样：

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
    // Other properties
    .setColor(ContextCompat.getColor(context, R.color.primary));
```

## Heads up 通知

![](https://ws4.sinaimg.cn/large/006tNc79gy1fron4q57ypj30m808cglg.jpg)

Heads up 通知的行为和 Marshmallow 一样，但是现在他们使用 Android N 系统使用的更新模板。这给了我们一个新的通知外观和感觉，如下所示：

![Heads-Up通知现在使用Android N中使用的新通知模板](https://ws4.sinaimg.cn/large/006tNc79gy1fron500f3lg30i408in8g.gif)

Android N 中 heads-up 通知的好处是我们现在可以使用自定义布局来声明我们的通知应该如何显示。我们将在下一节中介绍

## 自定义查看通知

与以前版本的 Android 一样，我们仍然可以使用自定义布局来使用我们的通知。 这意味着我们可以使用我们自己的布局资源来声明通知的外观（只是不要被带走！）。 这对以下方面很有帮助: 

- 显示 Notifications API 以前不支持的组件
- 在通知中显示更有意义，更清晰的信息，为用户提供更有意义的知识，而无需打开你的应用程序
- 将品牌添加到你的通知，以匹配你的应用程序的外观和感觉

但是说到这里，你应该确保将通知的设计保持在最低限度。 我们不想用信息压倒用户，尤其是在这么小的空间里。 允许用户受益于你的通知，同时仍然保持细节和设计至少。 

因此，考虑到这一点，我认为我们已经准备好为我们的通知创建一个自定义布局！ 这样做相当简单，我们可以使用 RemoteViews 类来构建自定义通知。 

```java
RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.notif_custom_view);
remoteViews.setImageViewResource(R.id.image_icon, iconResource);
remoteViews.setTextViewText(R.id.text_title, title);
remoteViews.setTextViewText(R.id.text_message, message);
remoteViews.setImageViewResource(R.id.image_end, imageResource);
```

我们首先创建一个 RemoteViews 类的新实例，传递应用程序的包名(系统在处理通知时需要这个名称)和我们希望用于通知的布局本身。 然后我们可以使用 RemoteView 类提供的方法来设置我们布局中的视图所使用的数据 / 资源。 从 RemoteView 类中，我使用了以下方法: 

- **setImageViewResource(viewId, resource)**

我们的图像资源和我们希望使用我们的图像资源的视图 ID。

- **setTextViewText(viewId, text)**

我们希望在给定的文本视图中显示的文本，由它的 ID 引用。

这个类允许我们设置数据值、资源、侦听器以及更多的视图。 一定要查看完整列表的文档！ 

所以现在我们创建了一个 RemoteViews 实例，我们需要在通知中使用它来显示我们想要的外观和感觉。

## 折叠视图

在前面的部分中，我们声明 RemoteViews 实例中的布局以及用于这些视图的数据，所以在构建我们的通知时，我们需要做的就是将 RemoteViews 实例传递到我们生成器的 setCustomContentView 方法。 在这个例子中，我们为通知的折叠状态提供了自定义视图: 

```java
Notification.Builder builder = new Notification.Builder(context)
        .setSmallIcon(R.drawable.ic_phonelink_ring_primary_24dp)
        .setCustomContentView(remoteViews)
        .setStyle(new Notification.DecoratedCustomViewStyle());
        .setAutoCancel(true);
```

为我们通知的崩溃状态设置一个自定义内容视图只会显示我们对这个状态的通知。 我们还没有设置一个扩展状态，所以通知将不可扩展，并将显示如下: 

![使用CustomContentView进行通知](https://ws1.sinaimg.cn/large/006tNc79gy1fron56mwq6j30m804tgls.jpg)

## 展开视图

与上面类似，我们还可以通过将 RemoteViews 实例传递给 setCustomBigContentView ( ) 方法来为通知的扩展（更大）状态提供自定义视图

```java
Notification.Builder builder = new Notification.Builder(context)
        .setSmallIcon(R.drawable.ic_phonelink_ring_primary_24dp)
        .setCustomBigContentView(bigRemoteView)
        .setStyle(new Notification.DecoratedCustomViewStyle());
        .setAutoCancel(true);
```

如果我们使用 setCustomBigContentView ( ) 方法而不使用 setCustomContentView ( ) 方法设置折叠的布局，那么我们的通知将简单地折叠为仅显示应用程序名称的状态，如下所示：

![使用CustomBigContentView进行通知](https://ws2.sinaimg.cn/large/006tNc79gy1fron5iih3vg30m809xter.gif)

最佳做法是，如果你提供展开尺寸的通知布局，则应提供折叠的布局。 这是因为上面的空的折叠状态没有给用户提供上下文，这违背了通知的要点！ 如果你只打算使用单个尺寸（折叠或展开），则应该仅使用折叠状态。

## 使用折叠和展开视图

如果我们希望我们的通知既可扩展又可折叠，那么我们可以通过调用 setCustomContentView ( ) 和 setCustomBigContentView ( ) 方法来为这两种状态提供 RemoteView 实例。

```java
Notification.Builder builder = new Notification.Builder(context)
        .setSmallIcon(R.drawable.ic_phonelink_ring_primary_24dp)
        .setCustomContentView(remoteView)
        .setCustomBigContentView(bigRemoteView)
        .setStyle(new Notification.DecoratedCustomViewStyle());
        .setAutoCancel(true);
```

现在我们已经为通知设置了折叠和展开视图，用户可以扩展通知以查看有关通知的更多信息。

![使用CustomContentView和CustomBigContentView进行通知](https://ws4.sinaimg.cn/large/006tNc79gy1fron5rrfnjg30i40a77c5.gif)

## 自定义 Heads-up 视图

当涉及到 Heads-up 通知时，我们也可以使用自定义布局。 在前面的章节中使用相同的方法来创建一个提醒通知，我们可以通过简单地将一个 RemoteViews 实例传递给 setCustomContentView ( ) 方法来使用自定义视图。

![使用CustomContentView进行提醒通知](https://ws3.sinaimg.cn/large/006tNc79gy1fron5z90zig30i408i7b4.gif)

## 结束

因此，我们已经看过了 Android n 中的这个新通知，以及如何实现它们ーー所以现在是时候将它构建到现有 / 新的 Android n 应用程序中了！ 我认为这些是 Android 平台的一些重要补充，不仅通知功能更强大(也更漂亮) ，而且我期待着在应用程序中看到更好的通知体验。 如果你有任何问题，请随时发 tweet 或在下面留下回复！ 

如果你喜欢这篇文章，不要忘了点击推荐按钮:)

[Joe Birch (@hitherejoe) | Twitter](https://twitter.com/hitherejoe)

