#谷歌发布 WorkManager 之前/之后如何使用服务.

> 原文 ：[Services. The life with/without. And WorkManager.](https://medium.com/google-developer-experts/services-the-life-with-without-and-worker-6933111d62a6)
>
> 作者 ：[Yonatan V. Levin](https://medium.com/@yonatanvlevin?source=post_header_lockup)
>
> 

随着最近的 Android 版本涉及到了后台，它变得比以往任何时候都更加复杂。 就像《星球大战》的情节一样，它已经变得越来越相互交织。因此，谷歌已经发布 [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager) 作为 [JetPack](https://developer.android.com/jetpack/) 的一部分——来帮助我们处理这样一个后台。

在了解 WorkManager 是什么之前，了解我们为什么需要 WorkManager 是至关重要的，并且建立它背后的原因是什么。 那些知道组件 / 库背后发生了什么，并理解他们为什么使用它的开发人员ーー是一个更好的开发者。

这将是一篇很长的文章，所以准备一杯好的咖啡和一些饼干。

它分为三个部分: 

Part #1 — Memory Basics 

Part #2 — Existing Background Solutions

 Part #3 — WorkManager

首先，在开始使用所有后台化的东西之前，我们需要了解一些 Android 处理内存管理的基础知识。 这将是我们的第一部分:

### 第一部分: 安卓内存 101

很久很久以前，在一个遥远的星系里，安卓内核最初是基于 linux 内核开发的。 安卓系统和所有其他基于 linux 内核的系统的主要区别在于，Android 没有一个叫做"Swap space"的东西。

当物理内存(RAM)的数量满足时，使用 Linux 中的交换空间。 如果系统需要更多的内存资源，而 RAM 是完整的，内存中的非活动页移动到交换空间。 尽管交换空间可以帮助拥有少内存的机器，但它不应该被认为是多内存的替代品。 交换空间位于硬盘上，硬盘的访问时间比物理内存慢。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvpnzj32j30wu0o8q78.jpg)

Android 没有一个叫做"Swap space"的东西，当系统内存不足时，它使用 OOM Killer 来保存星系。

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvq5nuz4j318g0p1jyd.jpg)

这个家伙的目标是通过基于"可见性状态"和消耗的内存量来杀死进程来释放内存。

每一个进程都由活动管理器给它的 oom adj 分数。 它是应用程序状态(如前台、后台、拥有服务的后台等)的组合。 下面是所有 oom adj 值的一个简短例子:

```
# Define the oom_adj values for the classes of processes that can be
# killed by the kernel.  These are used in ActivityManagerService.
    setprop ro.FOREGROUND_APP_ADJ 0
    setprop ro.VISIBLE_APP_ADJ 1
    setprop ro.SECONDARY_SERVER_ADJ 2
    setprop ro.BACKUP_APP_ADJ 2
    setprop ro.HOME_APP_ADJ 4
    setprop ro.HIDDEN_APP_MIN_ADJ 7
    setprop ro.CONTENT_PROVIDER_ADJ 14
    setprop ro.EMPTY_APP_ADJ 15
```

更高的 omm adj 值更有可能被内核的 OOM Killer 杀死。 当前的前台应用程序有一个0的 omm adj。

OOM Killer 使用基于自由内存和 omm adj 阈值的可配置规则。 例如，规则规定"如果自由内存 < X1，杀死 omm adj > Y1的进程"。

所以基本上流程是这样的:

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvqkgyp2j318g0hj406.jpg)

> 每个进程都分配了一个值

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvqvctiej318g0mrgnp.jpg)

> 内存消耗10kb，杀死所有 adj_oom > 7 的进程

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvr5sq1jj318g0htq42.jpg)

> 内存被释放了。 星系现在得到了很好的控制

现在，我希望你有这样的想法: 你消耗的内存越少，你就有更好的机会去完成重要的事情。

第二个重要的想法是理解应用程序的状态是必不可少的。 所以当你的应用进入后台时，你仍然想把 Luke 送入太空，你必须使用"服务"。

> 服务是一个应用程序组件，可以在后台执行长时间运行的操作，并且不提供用户界面。

你应该使用服务的原因有几个：

1. 告诉系统你有一个长时间运行的操作，并相应得到你的进程 oom adj 值。
2. 这是一个很好的例子。 安卓应用程序的4个入口点之一( BroadcastReceiver，Activity，ContentProvider 是其余3个)。
3. 在单独的进程上运行服务。

但是，使用服务有一个黑暗的一面:

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvrgntt8j307c0b6js5.jpg)

当我写我的第一个应用程序时，我成功地在不到3个小时的时间里把电池从100% 下降到了0% 。 怎么做？ 让服务器每3分钟从服务器中提取数据:)

我是个年轻没经验的学徒。

但不知何故，6年后，还有其他一些未知的应用以某种方式成功地做到了这一点:

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvrty6tnj30uk0l6q7n.jpg)

每个开发人员都在没有任何限制的情况下做他们想做的任何事情。 西斯统治着整个星系，只有少数绝地武士反击。

但是谷歌有一些很好的反抗者可以反击。

从 Marshmallow 开始，接着是 Nougat，然后引入 Doze (打盹)模式：

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvs5iwf6j318g0ho41b.jpg)

如果你不熟悉 Doze 模式，那么你真的应该熟悉。 简而言之，当用户关闭了设备的屏幕后，Doze 模式就会激活和禁用网络、同步、 GPS、alarms 和 wifi 扫描。 它一直持续到用户打开屏幕或者连接到充电器。 这个想法是为了减少从事不重要工作的应用程序的数量，并且这样做ーー节省了用户的电池:)

但是这感觉就像是海里的一滴水，所以谷歌甚至更进一步，从 Android Oreo (API 26)开始。

如果一个针对安卓8.0的应用在不允许创建后台服务的情况下尝试使用该方法，那么 startService 方法就会抛出一个 [IllegalStateException](https://developer.android.com/reference/java/lang/IllegalStateException.html)。

永远不要设置针对 SDK 26可以很容易地修复它。 一些"知名"应用程序决定设置针对 SDK 22 ，因为它们不想处理运行时权限。

但还有更多的事情要发生:

- 2018年8月: 新的应用程序需要针对 API 级别26(Android 8.0)或更高
- 2018年11月: 针对 API 级别26或更高版本所需的现有应用更新。
- 2019年及以后: 每年的针对 API 级别要求都会提前。 在每次安卓甜点发布后的一年内，新的应用程序和应用程序更新将需要针对相应的 API 级别或更高级别

说了这么多ーー(我相信你也会得出同样的结论) :

> 我们今天所知道的服务已经被取消了
>
> 它不再被允许实现它的主要目的，即在后台执行长期运行的任务。 因此，它不再可用了

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvtcseg7j318g0otb29.jpg)

除非你不使用 Service 作为前台服务，否则没有任何理由使用服务。 如果你需要依靠它ーー你需要一个 job。

### 第二部分: 我有一个网络调用。那里有什么:

因此，让我们以一个简单的网络调用为例，它可以下载几千字节。

首先，最直接的方法(和错误的方法)是有一个独立的线程来执行你的存储库 / 活动。

```java
int threads = Runtime.getRuntime().availableProcessors();
ExecutorService executor = Executors.newFixedThreadPool(threads);
executor.submit(myWork);
```

考虑一下登录场景。 你的用户填写电子邮件，密码和点击登录按钮。 用户的网络有着糟糕的3G，用户走进电梯。

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvtouz4dg30a00hstaf.gif)

虽然网络调用正在进行，但用户会接到一个调用。

*OkHttp* 默认超时：

connectTimeout = 10_000; 

readTimeout = 10_000; 

writeTimeout = 10_000;

通常，我们将默认设置为三次网络重试。

因此，在最坏的情况下: 3 * 30 = 90秒。

现在试着回答一个问题 —

### 登录成功了吗？

那么，虽然你的应用程序在后台，但事情是 - 你不知道。 正如我们所了解的，你不能相信你的进程会保持存活，以完成网络呼叫，处理响应并保存用户登录信息。 更不用说用户的手机可以脱机并失去互联网连接。

从用户的角度 - “我已经输入我的电子邮件，密码，并点击登录按钮。 因此我已经登录“。
如果没有，请相信我 - 用户会认为你的用户体验中有一些坏事。 但这不是用户体验问题，而是技术问题。
下一步你可能会想 - 是的，所以一旦我得到一个应用程序将要进入后台的回调，我将切换到服务。 可是等等。 你不能！:(

所以这里的 JobScheduler 来拯救。

```java
ComponentName service = new ComponentName(this, MyJobService.class);
JobScheduler mJobScheduler = (JobScheduler)getSystemService(Context.JOB_SCHEDULER_SERVICE);
JobInfo.Builder builder = new JobInfo.Builder(jobId, serviceComponent)
 .setRequiredNetworkType(jobInfoNetworkType)
 .setRequiresCharging(false)
 .setRequiresDeviceIdle(false)
 .setExtras(extras).build();
mJobScheduler.schedule(jobInfo);
```

计划一个工作开始。 当合适的时间到来时，系统将启动 MyJobService，并执行 startjob () 方法中的任何内容。这个想法在理论上是好的，但 JobScheduler 只能从最小 API 21获得。但是... 

但是，API 21 & 22中的 JobScheduler 有一个非常错误的组件。

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvuakowxj317m05mq58.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvumi3adj30r20jwaeq.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbvuytaxoj318g0pitgz.jpg)

这意味着你可以使用的真正 minSDK 从23开始。

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvv78fncj31100900tc.jpg)

在你的 minSDK < 23中。然后你有一个 JobDispatcher 选项。

```java
Job myJob = firebaseJobDispatcher.newJobBuilder()
 .setService(SmartService.class)
 .setTag(SmartService.LOCATION_SMART_JOB)
 .setReplaceCurrent(false)
 .setConstraints(ON_ANY_NETWORK)
 .build();
firebaseJobDispatcher.mustSchedule(myJob);
```

等等。 它需要 Google Play Services!！

![](https://ws3.sinaimg.cn/large/006tNc79gy1ftbvvnbapbj318g0jw7cw.jpg)

因此，如果你打算使用它，你将会让数以千万计的用户离开 Amazon Fire、Amazon TV 和数百家中国制造商:

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbvw3tl3rj30us0manfx.jpg)

因此... ... JobDispatcher 可能不是一个好的选择... 所以 AlarmManager？ 安排重复发生的警报触发并检查网络调用是否成功，然后尝试再次执行它？...

如果你仍然希望从 pre-O 设备上的旧服务中获益，并运行具有零时延迟的服务来执行调用。

而 JobIntentService 可能会帮助吉迪斯拯救一个星系:

![](https://cdn-images-1.medium.com/max/1600/1*4acReBCQvng9JxElyT7l3A.png)

它将使你在低于26的 SDK 上使用常规 IntentService 和 SDK ≥ 26上使用 JobScheduler 执行作业，并且它是支持库的一部分。

所以我们回到了开始的地方: ，管理我们正在运行的安卓版本，在后台执行调用，然后根据设备状态调整应用程序的后台调度，然后重新安排它们。

天啊... 要成为一个善良的绝地武士是如此的困难，他们愿意拯救用户的电池寿命，并为我们的用户提供惊人的用户体验。

![](https://ws1.sinaimg.cn/large/006tNc79gy1ftbvx3sn64g30m80godll.gif)

### 第三部分: WorkManager。 仅仅因为工作应该是容易做的

所以对于不同的设备状态有不同的解决方案; Android 版本和 / 不包括 Google Play 服务设备。 你可能会开始认为你需要自己完成所有这些艰苦的工作，并将基于"什么和如何"的不同解决方案结合起来 好消息是，Android 框架下的人们听到了我们所有的抱怨ーー他们决定从西斯的重构中拯救我们和整个星系:)

在最后一个 Google I/O Android 框架上，团队宣布 WorkManager:

Workmanager 旨在通过为系统驱动的后台处理提供一个一流的 API 来简化开发人员的体验。 它是为了应用程序不再在前台运行的后台工作。 在可能的情况下，它使用 [JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler.html) 或 [Firebase JobDispatcher](https://github.com/firebase/firebase-jobdispatcher-android) 来完成工作，如果你的应用程序在前台，它甚至会尝试在你的进程中直接完成这项工作。

哇！ 这正是我们所需要的ー一个简单的包装器，用于所有那些疯狂的后台执行选项。

WorkManager 库包含几个组件:

**WorkManager** - 接收具有参数和约束的工作并将其排队。

**Worker** - 只有一种方法来实现在后台线程上执行的 doWork ()。 这里是你所有的后台任务应该完成的地方。 尽可能保持简单。

**WorkRequest** - WorkRequest 指定 Worker，使用什么参数排队，以及它的约束(例如，internet，charging)。

**WorkResult** - 成功、失败、重试。

**Data** - 从 Worker 传入/传出的持久性键/值对。

首先，创建新的 Worker 扩展类，实现 doWork () 方法:

```java
public class LocationUploadWorker extends Worker {
    ...
     //Upload last passed location to the server
    public WorkerResult doWork() {
        ServerReport serverReport = new ServerReport(getInputData().getDouble(LOCATION_LONG, 0),
                getInputData().getDouble(LOCATION_LAT, 0), getInputData().getLong(LOCATION_TIME,
                0));
        FirebaseDatabase database = FirebaseDatabase.getInstance();
        DatabaseReference myRef =
                database.getReference("WorkerReport v" + android.os.Build.VERSION.SDK_INT);
        myRef.push().setValue(serverReport);
        return WorkerResult.SUCCESS;
    }
}
```

第二个调用 WorkManager 来执行这个工作:

```java
Constraints constraints = new Constraints.Builder().setRequiredNetworkType(NetworkType
            .CONNECTED).build();
Data inputData = new Data.Builder()
            .putDouble(LocationUploadWorker.LOCATION_LAT, location.getLatitude())
            .putDouble(LocationUploadWorker.LOCATION_LONG, location.getLongitude())
            .putLong(LocationUploadWorker.LOCATION_TIME, location.getTime())
            .build();
OneTimeWorkRequest uploadWork = new OneTimeWorkRequest.Builder(LocationUploadWorker.class)
            .setConstraints(constraints).setInputData(inputData).build();
WorkManager.getInstance().enqueue(uploadWork);
```

WorkManager 将负责剩余部分。 它会选择最好的时间表排队你的工作; 它会存储所有参数，工作细节和更新工作状态。 你甚至可以使用 LiveData 进行订阅以观察工作状态：

```java
WorkManager.getInstance().getStatusById(locationWork.getId()).observe(this,
        workStatus -> {
    if(workStatus!=null && workStatus.getState().isFinished()){
         ...
    }
});
```

在下面，WorkManager 的架构将会是这样的:

![](https://cdn-images-1.medium.com/max/1600/1*VkznGM_XrSK9kmOujJCV6w.png)

你创建新的 Work 并指定哪个 Worker 应该使用哪些 Arguments ，处于哪种 Constraints 来完成工作。 WorkManager 使用 Room 将 DB 中的 Worker 保存下来，并立即将 Worker 编入队列。它选择最好的可能调度程序(Jobscheduler，JobDispatcher，Executor aka greedysjander，AlarmManager) ，并调用 doWork () 方法。 结果发布在 LiveData 上，输出可用 Arguments 来获取。

就这么简单。

当然，还有更多。

有一个周期性的工作，你可以安排:

```java
Constraints constraints = new Constraints.Builder().setRequiredNetworkType
        (NetworkType.CONNECTED).build();
PeriodicWorkRequest locationWork = new PeriodicWorkRequest.Builder(LocationWork
        .class, 15, TimeUnit.MINUTES).addTag(LocationWork.TAG)
        .setConstraints(constraints).build();
```

你可以连续链式两个或更多的工作:

```
WorkManager.getInstance(this)
        .beginWith(Work.from(LocationWork.class))
        .then(Work.from(LocationUploadWorker.class))
        .enqueue();
```

或者并行

```java
WorkManager.getInstance(this).enqueue(Work.from(LocationWork.class, 
        LocationUploadWorker.class));
```

或者把它们混合在一起。

> 注意: 你不能将周期性和一次性工作的链条连接起来

你可以用 WorkManager 做很多其他的事情: 取消工作，合并工作，链接工作，合并从一个工作到另一个工作。 我鼓励你去探索这些文档。 它有很多好的例子。

#### 现实生活中的场景

我们需要建立一个应用程序，每15分钟跟踪用户的位置。

首先，我们创造了一个工作来获得一个位置。 没有内置的异步执行方式，因为 doWork () 应该返回 Result.SUCCESS，所以如果你需要一种异步的方式来执行 doWork (例如，查询 GPS 位置) ，你可以使用 Latch 和另一个机制来阻止执行:

```java
public class LocationWork extends Worker {

    ...

    public WorkerResult doWork() {
        Log.d(TAG, "doWork: Started to work");
        handlerThread = new HandlerThread("MyHandlerThread");
        handlerThread.start();
        looper = handlerThread.getLooper();
        locationTracker = new LocationTracker(getApplicationContext(), looper);
        locationTracker.start();
        try {
            locationWait = new CountDownLatch(1);
            locationWait.await();
            Log.d(TAG, "doWork: Countdown released");
        } catch (InterruptedException e) {
            Log.d(TAG, "doWork: CountdownLatch interrupted");
            e.printStackTrace();
        }

        cleanUp();
        return WorkerResult.SUCCESS;
    }
 

}
```

安排时间:

```java
PeriodicWorkRequest locationWork = new PeriodicWorkRequest.Builder(
    LocationWork.class, 15, TimeUnit.MINUTES).addTag(LocationWork.TAG).build();
WorkManager.getInstance().enqueue(locationWork);
```

当我们完成定位获取操作后，我们将释放锁存器，当然还要清理所有的混乱:

```java
public class LocationWork extends Worker {

    ...
  
    static void reportFinished() {
        if (locationWait != null) {
            Log.d(TAG, "doWork: locationWait down by one");
            locationWait.countDown();
        }
    }
  
    private void cleanUp() {
        Log.d(TAG, "Work is done");
        locationTracker.stop();
        handlerThread.quit();
        looper.quit();
        handlerThread.quit();
    }

}
```

一旦获得了 LocationTracker，我们需要将它上传到我们的服务器上ーー因此我们将安排另一个一次性工作将其上传到服务器:

```java
//location obtained, need to send it to server
    private void broadcastLocation(Location location) {
        //release latch
        reportFinished();
        //We need to sure that device have internet live
        Constraints constraints = new Constraints.Builder().setRequiredNetworkType
                (NetworkType.CONNECTED).build();
        
        //Parse our location to Data to use it as input for our worker
        Data inputData = new Data.Builder()
                .putDouble(LocationUploadWorker.LOCATION_LAT, location.getLatitude())
                .putDouble(LocationUploadWorker.LOCATION_LONG, location.getLongitude())
                .putLong(LocationUploadWorker.LOCATION_TIME, location.getTime())
                .build();
        //worker itself
        OneTimeWorkRequest uploadWork = new OneTimeWorkRequest.Builder
                (LocationUploadWorker.class).setConstraints(constraints).setInputData
                (inputData).build();
        //Schedule it 
        WorkManager.getInstance().enqueue(uploadWork);
}
```

以及 UploadWork 本身:

```java
public class LocationUploadWorker extends Worker {
    //...
  
    public WorkerResult doWork() {
      //get Data out from input
        double longitude = getInputData().getDouble(LOCATION_LONG, 0);
        double latitude = getInputData().getDouble(LOCATION_LAT, 0);
        long time = getInputData().getLong(LOCATION_TIME, 0);
        String osVersion = "WorkerReport v" + Build.VERSION.SDK_INT;
        //construct our report for server format
        ServerReport serverReport = new ServerReport(longitude, latitude, time);
     
        //Report it to firebise server
        FirebaseDatabase database = FirebaseDatabase.getInstance();
        DatabaseReference myRef = database.getReference(osVersion);
        myRef.push().setValue(serverReport);
      
        return WorkerResult.SUCCESS;
    }
}
```

看起来很简单？ 事实上ーー确实如此。 没有更复杂的 jobschedulers / jobdispatcher / greed Executors 样板代码。 你创建一个工作，计划它，它就完成了。 就这么简单。

![](https://ws2.sinaimg.cn/large/006tNc79gy1ftbw0o3i3sg30m80go4qp.gif)

### 总结

在过去 / 未来的 Android 版本中，随着节省用户电量的意愿，在后台运行会变得更加复杂。 多亏了安卓团队，我们有了一个 WorkManager，它使处理后台变得更加自然和直接。

最后一件事。

没有电池的话，手机看起来怎么样？

![](https://ws4.sinaimg.cn/large/006tNc79gy1ftbw0aq2jyj318g0olb29.jpg)

感谢阅读。 如果你喜欢它，请给我你的 👏 👏和分享它。 我还想听听你的意见和建议:)谢谢

