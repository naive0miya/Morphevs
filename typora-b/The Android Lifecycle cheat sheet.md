# Android 生命周期备忘单

> 原文 (Medium)：[The Android Lifecycle cheat sheet — part I: Single Activities](https://medium.com/google-developers/the-android-lifecycle-cheat-sheet-part-i-single-activities-e49fd3d202ab)
>
> 原文 (Medium)：[The Android Lifecycle cheat sheet — part II: Multiple activities](https://medium.com/@JoseAlcerreca/the-android-lifecycle-cheat-sheet-part-ii-multiple-activities-a411fd139f24)
>
> 原文 (Medium)：[The Android Lifecycle cheat sheet — part III : Fragments](https://medium.com/@JoseAlcerreca/the-android-lifecycle-cheat-sheet-part-iii-fragments-afc87d4f37fd)
>
> 作者： [Jose Alcérreca](https://medium.com/@JoseAlcerreca) 

[TOC]

# 第1部分：单项活动 - 单一活动生命周期 

## 单一活动 - 情景1：应用程序完成并重新启动

触发者：
- 用户按下后退按钮，或者
- Activity.finish ( ) 方法被调用。

最简单的场景展示了当用户启动、完成和重新启动单个活动应用程序时会发生什么: 
![@情况1：应用程序已完成并重新启动|center](https://ws3.sinaimg.cn/large/006tKfTcgy1frowrf681sj30an0le0t6.jpg)

管理状态
- [onSaveInstanceState](https://developer.android.com/reference/android/app/Activity.html#onSaveInstanceState%28android.os.Bundle%29) 不被调用（因为活动已经完成，所以不需要保存状态）。
- 当应用程序重新打开时，[onCreate](https://developer.android.com/reference/android/app/Activity.html#onCreate%28android.os.Bundle%29) 没有 Bundle，因为活动已经完成，而且不需要恢复状态。

## 单一活动 - 场景2：用户导航
触发者：
- 用户按下 Home 键 。
- 用户切换到另一个应用程序（通过概览菜单，通知，接受电话等）。
  ![@场景2：用户导航|center](https://ws1.sinaimg.cn/large/006tKfTcgy1frowrl60pcg308w0ftqv5.gif)

在这种情况下，系统会[停止](https://developer.android.com/guide/components/activities/activity-lifecycle.html#onstop)活动，但不会立即完成。

管理状态

当你的活动进入停止状态时，系统会使用 onSaveInstanceState 来保存应用状态，以防系统在稍后杀死该应用程序的进程(见下文)。 

假设进程没有被杀死，活动实例保存在内存中，保留所有状态。 当活动回到前台时，活动会回顾这些信息。 您不需要重新初始化之前创建的组件。 

## 单一活动 - 场景3：配置更改
触发者：
- 配置更改，如旋转屏幕。
- 用户在多窗口模式下调整窗口大小。

![@场景3：旋转和其他配置更改|center](https://ws4.sinaimg.cn/large/006tKfTcgy1frowsd2ac3j30de0nct99.jpg)

管理状态

像旋转或调整窗口大小这样的配置更改应该让用户继续保持原来的状态。 

- 活动被完全销毁，但是为了新实例，状态得到了保存和恢复 。
- onCreate 和 onRestoreInstanceState 中的 Bundle 是一样的。

## 单一活动 - 情景4：应用程序被系统暂停
触发者：
- 启用多窗口模式（API 24+）并失去焦点。
- 另一个应用程序部分覆盖正在运行的应用程序（购买对话框，运行时权限对话框，第三方登录对话框...）。
- 出现意图选择器，如共享对话框。
  ![@场景4：应用程序被系统暂停|center](https://ws3.sinaimg.cn/large/006tKfTcgy1frowt1r8f6j30ej0fy0t1.jpg)

这种情况不适用于：
- 对话在同一个应用程序。显示 AlertDialog 或 DialogFragment 不会暂停后台活动。
- 通知。用户收到新通知或拉下通知栏不会暂停后台活动。

# 第2部分：多项活动 - 导航和后台堆栈 



## 后台堆栈 - 场景1：在活动之间导航
![@情景1：在活动之间进行导航|center](https://ws3.sinaimg.cn/large/006tKfTcgy1frowt6eingj30k80r0q40.jpg)
在这种情况下，当一个新的活动开始时，活动1被[停止](https://developer.android.com/guide/components/activities/activity-lifecycle.html#onstop)（但还没被销毁），类似于用户导航离开 （就好像 “home” 被按下）。

当按下后退按钮时，活动2被销毁并结束。

管理状态

请注意，[onSaveInstanceState](https://developer.android.com/reference/android/app/Activity.html#onSaveInstanceState%28android.os.Bundle%29) 被调用，但 [onRestoreInstanceState](https://developer.android.com/reference/android/app/Activity.html#onRestoreInstanceState%28android.os.Bundle,%20android.os.PersistableBundle%29) 没有被调用。 如果第二个活动处于活动状态时发生配置更改，则只有当第二个活动重新聚焦后，第一个活动才会被销毁和重新创建。 这就是为什么保存一个状态实例是很重要的。

如果系统为了节省资源而杀死了应用程序，那么这是另一个需要恢复状态的场景。 

## 后台堆栈 - 场景2：后台堆栈中的活动配置更改
![@方案2：在配置更改的后台堆栈中的活动|center](https://ws4.sinaimg.cn/large/006tKfTcgy1frowtbqu8aj30km0vbwfs.jpg)

管理状态

保存状态不仅对前台中的活动很重要。 堆栈中的所有活动都需要在配置更改之后恢复状态，以重新创建它们的 UI。 

此外，这个系统几乎可以在任何时候杀死你的应用程序的进程，所以你应该准备好在任何情况下恢复状态。 


## 后台堆栈 - 场景3：应用程序的进程被终止
当 Android 操作系统需要资源时，它会在后台杀死应用程序。
![@场景3：应用程序的进程被终止|center](https://ws4.sinaimg.cn/large/006tKfTcgy1frowti8y90j30m80txwh1.jpg)

管理状态

请注意，完整备份堆栈的状态是保存的，但为了有效地使用资源，活动只有在重新创建时才会被恢复。 

另请阅读
- [谁生，谁死？ 安卓系统的优先级](https://medium.com/google-developers/who-lives-and-who-dies-process-priorities-on-android-cb151f39044f)

# 第3部分：片段 - 活动和片段生命周期



## 场景1：带有片段的活动开始并结束
![@场景1：带有片段的活动开始并结束|center](https://ws2.sinaimg.cn/large/006tKfTcgy1frowtms6rej30ja0rs0tr.jpg)

请注意，它保证活动的 onCreate 在 Fragment 之前执行。 然而，回调是并行显示的，例如 onStart 和 onResume ——是并行执行的，因此可以按任一顺序调用。 例如，系统可能在 片段 onStart 方法之前执行活动的 onStart 方法，也可能在活动的 onResume 方法之前执行 片段 onResume 方法。 

要小心管理各自执行顺序的时间，以避免竞争条件。

## 场景2：使用保留片段的活动被旋转

![@场景2：带有片段的活动被旋转|center](https://ws1.sinaimg.cn/large/006tKfTcgy1frowtr35g6j30m60v4jsm.jpg)

状态管理

片段状态以与活动状态以非常相似的方式保存和恢复。 不同之处在于在片断中没有 onRestoreInstanceState ，但是可以在 onCreate、 onCreateView 和 onActivityCreated 中找到 Bundle。  

可以保留片段，这意味着同样的实例用于配置更改。 正如下一个场景所示，这会稍微改变图表。 

## 场景3：带有保留片段的活动被旋转
![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowtupdbsj30m60sdgmp.jpg)

该片段不会在旋转之后被销毁或创建，因为在重新创建活动之后使用了相同的片段实例。 在 onActivityCreated 中，状态包仍然可用。 

不建议使用保留的片段，除非它们用于跨配置更改（在非 UI 片段中）存储数据。这是来自架构组件库的 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 类在内部使用的，但是具有更简单的 API。

如果你发现错误或者你认为有什么重要的东西丢失了，请在评论中告知。 还有，让我们知道你希望我们写些什么其他的场景。 

