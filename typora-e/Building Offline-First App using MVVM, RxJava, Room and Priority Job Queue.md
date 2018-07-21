# 使用 MVVM、 RxJava、 Room 和 Priority Job Queue 构建首先离线应用程序

> 原文 (Medium) ：[Building Offline-First App using MVVM, RxJava, Room and Priority Job Queue](https://proandroiddev.com/offline-apps-its-easier-than-you-think-9ff97701a73f)
>
> 作者 ：[James Shvarts](https://proandroiddev.com/@jshvarts?source=post_header_lockup)

今年早些时候，我参加了纽约市的 Android 开发人员会议。 发言人，Instagram 开发者，报道了他们让 Android 应用离线工作的努力，使它成为一个伟大的体验，甚至对于那些无／低网络连接性的人来说也是一个很好的体验。

下面我将简要介绍 Instagram 的方法，使用 MVVM，Google 的新的生命周期架构组件，Android 优先工作队列库，Room，RxJava2，Dagger Android 2.11，Retrofit，ButterKnife，然后提供一个离线应用程序的工作实例。

### 为什么离线模式？

80% 的 Instagram 用户不在美国。 其中许多人生活在网络连接有限或数据计划有限的发展中国家。 这些用户主要使用安卓设备，这就是为什么 Instagram 的团队首先将 Android 平台作为目标，努力让离线功能可用。

### 离线读支持

Instagram 的离线用户现在可以通过提供之前加载的缓存数据来看到他们的新闻源。 这个想法很简单: 每个读取请求都由请求任务(缓存键)识别。 每个 feed 响应都会被响应存储器缓存在设备上。 随后，当用户离线并请求为同一个请求任务请求数据时，请求被重播，并由本地响应存储器对该缓存键作出回应。 用户界面不在乎所提供的数据是缓存版本。 当用户试图显式刷新他们的 feed 时，一条消息告诉他们他们现在是离线状态。

### 离线写支持

Instagram的离线用户可以留下评论，比如内容，关注他人，保存媒体等等。 每个用例都实现了 PendingActionStore，所有用例都由管理器管理，用于通知用户连接更改和生命周期事件。基本上，每个 PendingActionStore 都知道如何在本地管理数据，如何远程同步，并在管理员指示它执行操作时开始工作。

### 滚动我自己的

让我们来看一个我创建的首先离线应用程序的例子。 它可能不包括离线应用程序的所有方面，但是它涵盖了它的要点: 首先保存本地化，远程同步，并在过程中反映用户界面的变化。 这个应用可以让用户在离线时无缝地发布评论，评论存储在本地，然后进行远程同步。 直接跳转到源代码。

[OfflineSampleApp | github.com](https://github.com/jshvarts/OfflineSampleApp) 

 ![](http://ww2.sinaimg.cn/large/006tNc79gy1frl93dnibsj30700b4wg9.jpg)

### 工作流程

总的工作流可以用这个图表来总结:

![](http://ww1.sinaimg.cn/large/006tNc79gy1frl94x5ch7j30ml0jg3zk.jpg)

- 提交新评论时，首先将其存储在本地数据库中。 记录标记为 syncPending = true。 用户界面中的评论文字颜色显示为灰色。
- 如果 Internet 连接可用，后台作业会将此记录与远程仓库同步。
- 如果没有连接可用，后台同步作业将排队，直到连接可用。 在此期间，用户可以关闭应用程序或甚至重新启动设备 - 挂起的同步请求保证能够存活并等待连接可用。
- 连接恢复后，后台作业会将记录与远程数据存储同步。
- 如果记录与远程仓库成功同步，则使用 syncPending = false 更新相应的本地记录。 用户界面中的评论文字颜色将变黑。
- 如果与远程仓库同步失败，本地记录将被删除以提供数据完整性。 这种情况是一个例外，而不是规则。 在真实世界的应用程序中，您可能想要通过显示通知来告知用户这一点。

> 您的数据应该首先在本地存储，然后远程同步

### 真理的唯一来源

在我们的应用程序中，本地数据库是真理的唯一来源。 虽然数据与远程(云)数据库同步，但数据查找总是在本地数据库中发生。 当与远程数据库同步失败时，我们会删除本地记录，以保证本地数据库和远程数据存储库之间的数据完整性。

### 离线设计策略

不幸的是，当涉及到离线应用程序设计时，没有一个适合所有解决方案的解决方案。 我的示例应用程序显示了2种可能的方式来完成离线功能（详情见下文）。 您的实现可能会有所不同 - 这完全取决于您的特定应用程序要求。

1. 你会告诉用户这个应用什么时候离线吗？ 如果是这样，那该怎么做？
2. 当远程同步失败 / 成功时，你会通知他们吗？ 如果是这样，那该怎么做？
3. 当用户试图显式刷新内容时，如何告诉用户他们是离线的？
4. 当你在线时，你是否会记录下最后一个滚动位置，只显示他们现在离线时没有看到的内容？

这些只是你在为你的应用程序设计离线功能时可能面临的一些问题。

### 应用架构

- MVVM (Model-View-ViewModel)模式使用 Google 的新的生命周期架构组件，如 ViewModel，ViewModelFactory，LiveData,，LifecycleObserver。
- 具有良好的层隔离和通过接口抽象的数据库的简洁式体系结构。 这样可以减少维护和可测性。 作业队列整合确实还有改进的空间
- 使用远程服务同步评论的后台作业使用[优先工作队列](https://github.com/yigit/android-priority-jobqueue)
- [RxJava 2](https://github.com/ReactiveX/RxJava), [RxRelay](https://github.com/JakeWharton/RxRelay) 用来异步地在图层间进行通信
- 本地数据库使用 Google 的新版本 Room 持久化库
- 远程 API 调用是使用 Retrofit 实现的，假远程数据源是在 [JSONPlaceholder](https://jsonplaceholder.typicode.com/) REST API 的帮助下完成的。
- 新的 Dagger Android 2.11注入 API 用于注入依赖关系
- 作为奖励，下面的质量检查被整合到构建过程中: lint，checkstyle，pmd，findbugs

### 优先工作队列

[tight/android-priority-jobqueue | github.com](tight/android-priority-jobqueue | github.com)

这个开源库是我们离线应用程序功能的支柱。 它由 Googler Yigit Boyar 维护，当你在 Android 上研究首先离线应用程序时，他的名字肯定会出现。 这个成熟的库允许你在棒棒糖上使用 Google 的 [JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler.html) API，并且可以通过 [GcmNetworkManager](https://developers.google.com/android/reference/com/google/android/gms/gcm/GcmNetworkManager) 来支持 API 级别9以上。

就像设计离线应用有多种方式一样，Android 优先工作队列也有不同的使用方式。 这个应用程序包含两个分支，展示了使用该库的2种方法:

1. 这个应用程序必须打开，以便数据与远程仓库同步
2. 这个应用程序不需要为后台同步请求打开。 优先工作队列可以在同一个过程中唤醒应用来执行后台同步工作

后一种方法是我的最爱。 当符合某些背景工作条件时(例如可以访问网络，可以使用 WiFi 等)时，可以在设备启动上唤醒应用程序。

优先工作队列中的作业是高度[可配置](https://github.com/yigit/android-priority-jobqueue/wiki/Job-Configuration)的。 一些常见的配置参数是:

- 定制工作优先级
- 网络可用性
- 重新审视和取消工作的逻辑
- 工作持久策略

当安卓框架的作业调度器被使用(在棒棒糖和以上)时，工作调度由操作系统管理，该操作系统优化远程调用并保证电池寿命。 此外，调度程序符合 [Doze 和 Stand By](https://developer.android.com/training/monitoring-device-state/doze-standby.html) 的限制。

### RxJava 的功能

虽然不适用于离线应用，但请注意 RxJava 如何帮助我们保持代码的简洁。

这里的评论是在成功的本地添加评论调用时远程同步的。

```Java
public class AddCommentUseCase {
    private final LocalCommentRepository localCommentRepository;
    private final SyncCommentUseCase syncCommentUseCase;

    public AddCommentUseCase(LocalCommentRepository localCommentRepository, SyncCommentUseCase syncCommentUseCase) {
        this.localCommentRepository = localCommentRepository;
        this.syncCommentUseCase = syncCommentUseCase;
    }

    public Completable addComment(String commentText) {
        return localCommentRepository.add(ModelConstants.DUMMY_PHOTO_ID, commentText)
                .flatMapCompletable(syncCommentUseCase::syncComment);
    }
}
```

一旦初始加载，我们的视图数据通过使用 Flowable 始终保持最新。 没有显式调用来刷新数据。 这使得我们的数据和用户界面之间的互动真的是被动的。

```Java
Flowable<List<Comment>> getComments(long photoId);
```

### 交流同步事件

优先工作队列库建议使用 EventBus 来将事件从后台工作传回到用户界面。 然而，考虑到1)行业趋势似乎是用 RxJava 代替 EventBus，2)我已经在应用程序中广泛使用了 RxJava，我决定创建一个自定义的 RxBus 解决方案，而不是 EventBus:

```Java
import com.example.offline.model.Comment;
import com.jakewharton.rxrelay2.PublishRelay;

import io.reactivex.Observable;

public class SyncCommentRxBus {

    private static SyncCommentRxBus instance;
    private final PublishRelay<SyncCommentResponse> relay;

    public static synchronized SyncCommentRxBus getInstance() {
        if (instance == null) {
            instance = new SyncCommentRxBus();
        }
        return instance;
    }

    private SyncCommentRxBus() {
        relay = PublishRelay.create();
    }

    public void post(SyncResponseEventType eventType, Comment comment) {
        relay.accept(new SyncCommentResponse(eventType, comment));
    }

    public Observable<SyncCommentResponse> toObservable() {
        return relay;
    }
}
```

下面是我们的后台作业如何在这个总线上发布同步响应事件:

```Java

// remote call was successful--the Comment will be updated locally to reflect that sync is no longer pending
Comment updatedComment = CommentUtils.clone(comment, false);
SyncCommentRxBus.getInstance().post(SyncResponseEventType.SUCCESS, updatedComment);

// sync to remote failed
SyncCommentRxBus.getInstance().post(SyncResponseEventType.FAILED, comment);
```

接收来自 RxBus 的事件排放取决于您选择处理同步请求的时间:

1. 应用程序打开的时候
2. 当应用程序不打开的时候

让我们来看看以下两种情况:

### 开启应用程序时同步数据

当应用程序打开时，以及当 CommentsActivity 恢复时接收事件排放，有一个 SyncCommentLifecycleObserver。 它注册来观察我们的视图，CommentsActivity 的生命周期事件

```Java
public class CommentsActivity extends AppCompatActivity implements LifecycleRegistryOwner {

    @Inject
    CommentsViewModelFactory viewModelFactory;

    @Inject
    SyncCommentLifecycleObserver syncCommentLifecycleObserver;

    private CommentsViewModel viewModel;

    private LifecycleRegistry registry = new LifecycleRegistry(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        AndroidInjection.inject(this);
        super.onCreate(savedInstanceState);
        setContentView(R.layout.comments_activity);

        getLifecycle().addObserver(syncCommentLifecycleObserver);
        
        // the above is just a snippet of our Activity
    }
}
```

生命周期事件 Lifecycle.Event.ON_RESUME 订阅我们 RxBus 的数据排放。 这大致相当于 EventBus.getDefault( ).unregister(this)

生命周期事件 Lifecycle.Event.ON_PAUSE 会从我们的 RxBus 排放中触发取消订阅(清除 Disposables)。 这大致相当于 EventBus.getDefault( ).unregister(this)

```Java
import android.arch.lifecycle.Lifecycle;
import android.arch.lifecycle.LifecycleObserver;
import android.arch.lifecycle.OnLifecycleEvent;

import com.example.offline.domain.DeleteCommentUseCase;
import com.example.offline.domain.UpdateCommentUseCase;
import com.example.offline.model.Comment;

import io.reactivex.android.schedulers.AndroidSchedulers;
import io.reactivex.disposables.CompositeDisposable;
import io.reactivex.schedulers.Schedulers;
import timber.log.Timber;

/**
 * Updates local database after remote comment sync requests
 */
public class SyncCommentLifecycleObserver implements LifecycleObserver {
    private final UpdateCommentUseCase updateCommentUseCase;
    private final DeleteCommentUseCase deleteCommentUseCase;
    private final CompositeDisposable disposables = new CompositeDisposable();

    public SyncCommentLifecycleObserver(UpdateCommentUseCase updateCommentUseCase,
                                        DeleteCommentUseCase deleteCommentUseCase) {
        this.updateCommentUseCase = updateCommentUseCase;
        this.deleteCommentUseCase = deleteCommentUseCase;
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume() {
        Timber.d("onResume lifecycle event.");
        disposables.add(SyncCommentRxBus.getInstance().toObservable()
                .subscribe(this::handleSyncResponse));
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause() {
        Timber.d("onPause lifecycle event.");
        disposables.clear();
    }

    private void handleSyncResponse(SyncCommentResponse response) {
        if (response.eventType == SyncResponseEventType.SUCCESS) {
            onSyncCommentSuccess(response.comment);
        } else {
            onSyncCommentFailed(response.comment);
        }
    }

    private void onSyncCommentSuccess(Comment comment) {
        Timber.d("received sync comment success event for comment %s", comment);
        disposables.add(updateCommentUseCase.updateComment(comment)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(() -> Timber.d("update comment success"),
                        t -> Timber.e(t, "update comment error")));
    }

    private void onSyncCommentFailed(Comment comment) {
        Timber.d("received sync comment failed event for comment %s", comment);
        disposables.add(deleteCommentUseCase.deleteComment(comment)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(() -> Timber.d("delete comment success"),
                        t -> Timber.e(t, "delete comment error")));
    }
}
```

上面的设置利用了 Google 最新的架构组件，通过坚持单一责任原则，帮助我们保持代码干净且易于管理。

为了测试这个设置，试着在离线时添加评论。 它们将只存储在本地ー评论文本颜色将是灰色的。 然后，重新启用网络连接，并观察您的后台作业更新远程数据存储一次一个评论他们被添加他们的用户。 当评论成功同步时，文本颜色将变为黑色。

> **专业建议** : 开发脱机功能的同时，使用"电梯测试"来模拟关键网络连接。 在电梯中使用这个应用程序并观察它的行为，如果连接丢失后重新建立

### 当应用程序不打开时同步数据

当应用程序没有打开或后台时，为了接收事件排放，请按以下步骤:

将 RECEIVE_BOOT_COMPLETED 的权限添加到 AndroidManifest.xml 中，这样你的应用程序在重新启动后可以被调度器唤醒。

创建作业调度程序时，设置 batch = false:

```Java
// we are setting batch param below to false so that sync results are observed and acted upon in the background.
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    builder.scheduler(FrameworkJobSchedulerService.createSchedulerFor(context,
            SchedulerJobService.class), false);
} else {
    int enableGcm = GoogleApiAvailability.getInstance().isGooglePlayServicesAvailable(context);
    if (enableGcm == ConnectionResult.SUCCESS) {
        builder.scheduler(GcmJobSchedulerService.createSchedulerFor(context,
                GcmJobSchedulerService.class), false);
    }
}
```

通过添加.persist（）作业配置参数使您的后台作业持久。

```Java
public SyncCommentJob(Comment comment) {
        super(new Params(JobPriority.MID)
                .requireNetwork()
                .groupBy(TAG)
                .persist());
        this.comment = comment;
    }
```

注意，SyncCommentJob 有一个单独的评论实例变量，这是一个简单的 POJO。 这是因为默认情况下，通过 Serialization 来实现 Android 优先任务队列的任务持久化，所以最容易使用轻量级的可序列化 pojo 构造 Jobs，并将其他依赖关系暴露出来(如 RemoteCommentService.getInstance ())。 或者，你可以 a)使用暂时的依赖和一些[dagger 的技巧](https://github.com/yigit/android-priority-jobqueue/issues/343)或者 b)使用 [GSON](https://github.com/google/gson) 或 [Protobuf](https://github.com/google/protobuf) 等库实现自定义序列化。

同步评论响应由 SyncCommentResponseObserver 观察，当应用被唤醒时，我们的应用程序类初始化了同步评论响应。

```kotlin
import android.support.annotation.WorkerThread;

import com.example.offline.domain.DeleteCommentUseCase;
import com.example.offline.domain.UpdateCommentUseCase;
import com.example.offline.domain.services.SyncCommentResponse;
import com.example.offline.domain.services.SyncCommentRxBus;
import com.example.offline.domain.services.SyncResponseEventType;
import com.example.offline.model.Comment;

import timber.log.Timber;

public class SyncCommentResponseObserver {

    private final UpdateCommentUseCase updateCommentUseCase;
    private final DeleteCommentUseCase deleteCommentUseCase;

    public SyncCommentResponseObserver(UpdateCommentUseCase updateCommentUseCase,
                                       DeleteCommentUseCase deleteCommentUseCase) {
        this.updateCommentUseCase = updateCommentUseCase;
        this.deleteCommentUseCase = deleteCommentUseCase;
    }

    void observeSyncResponse() {
        SyncCommentRxBus.getInstance().toObservable()
                .subscribe(this::handleSyncResponse);
    }

    private void handleSyncResponse(SyncCommentResponse response) {
        if (response.eventType == SyncResponseEventType.SUCCESS) {
            onSyncCommentSuccess(response.comment);
        } else {
            onSyncCommentFailed(response.comment);
        }
    }

    @WorkerThread
    private void onSyncCommentSuccess(Comment comment) {
        Timber.d("received sync comment success event for comment %s", comment);
        updateCommentUseCase.updateComment(comment)
                .subscribe(() -> Timber.d("update comment success"),
                        t -> Timber.e(t, "update comment error"));
    }

    @WorkerThread
    private void onSyncCommentFailed(Comment comment) {
        Timber.d("received sync comment failed event for comment %s", comment);
        deleteCommentUseCase.deleteComment(comment)
                .subscribe(() -> Timber.d("delete comment success"),
                        t -> Timber.e(t, "delete comment error"));
    }
}
```

为了测试这个设置，试着在离线时添加评论。 评论将在本地保存，用界面中的评论文本颜色(灰色)来显示。 试着关闭应用程序或重启设备。 暂时不要打开应用程序，但要重新启用网络连接。 您的工作现在被发送到远程数据存储器进行同步。 打开应用程序，确认这些工作已经同步到评论文本颜色(黑色)中。

### 结论

让我们再次总结一下离线应用的好处:

1. 即使离线，用户也可以继续享受这个应用
2. 由于网络连接问题，用户不再收到错误消息
3. 用户可以从响应更快的应用程序中受益
4. 用户从节约电池寿命中受益
5. 对进度条等的需求减少了，因为用户只能与快速本地存储进行交互。 作为奖励，这简化了构建用户界面的过程

鉴于上述情况，重视用户体验的应用程序应该认真对待离线支持。 为什么每次用户界面交互都要进行网络调用？ 我怀疑你的用户是否喜欢等待网络响应，看到进度指标，并且在网络连接丢失时被告知"再试一次"。 让我们把用户体验放在首位，让我们的应用程序对所有用户开放，而不仅仅是那些在快速网络上的用户。

### 建议资源

- 文章相关的示例项目: [OfflineSampleApp | github.com](https://github.com/jshvarts/OfflineSampleApp) 
- [Offline Mode at Instagram | youtube.com](https://www.youtube.com/watch?v=wctcEdogSIA) 2017年1月在纽约 Android Developers Meetup 上发布（快进至26:02）
- [Facebook Gets An Offline Mode | code.facebook.com](https://code.facebook.com/posts/1535185823471329/continuing-to-build-news-feed-for-all-types-of-connections/)
- [Android application architecture: Get ready for the next billion! | youtube.com](https://www.youtube.com/watch?v=70WqJxymPr8&t=574s) 演讲，由 [Yigit Boyar](https://twitter.com/yigitboyar)
- 另一个优先工作队列 [solidgear/priority-job-queue | github.com](https://github.com/solidgear/priority-job-queue) 

