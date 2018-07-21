# 掌握 JobScheduler

> 原文 (Medium)：[Scheduling jobs like a pro with JobScheduler](https://medium.com/google-developers/scheduling-jobs-like-a-pro-with-jobscheduler-286ef8510129)
>
> 作者：[Joanna Smith](https://medium.com/@dontmesswithjo?source=post_header_lockup)

[TOC]

[JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 是在 Lollipop 中引入的，它非常酷，因为它基于条件而不是按时完成工作。 

Video - [Background work with JobScheduler ](https://www.youtube.com/watch?list=PLWz5rJ2EKKc-lJo_RGGXL2Psr8vVCTWjM&v=XFN3MrnNhZA)

JobScheduler 保证完成你的工作，但是由于它在系统层面上运行，它也可以使用几个因素来明智地安排你的后台工作，以便与其他应用程序的工作一起运行。 这意味着我们可以最小化像无线电使用这样的事情，这是一个明显的电池的胜利。至于 API 24，JobScheduler 甚至考虑了内存压力，这对于设备和他们的用户来说是一个明显的整体胜利。 

我们使用 JobScheduler 的目标是找到一种方法，让系统承担一部分创建形式化应用程序的负担。 作为一个开发者，你要尽自己的一份力去创建一个不会冻结的应用程序，但这并不总是转化为设备的电池寿命是健康的。 因此，通过在系统层面上引入 JobScheduler，我们可以聚焦在一起处理类似的工作请求，这对电池和内存都有明显的改善。 

# 如何使用 JobScheduler

当谈到" 使用 JobScheduler "时，谈话实际上是一次解决三个独立的问题。 

## 首先，考虑你的工作。

你想要安排的工作应该在 [JobService](https://developer.android.com/reference/android/app/job/JobService.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 中定义。 你的 JobService 实际上是一个扩展 JobService 类的[服务](https://developer.android.com/reference/android/app/Service.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog)。 无论你的应用程序是否处于活动状态，这都是系统为你执行工作的原因。这个结构中最方便的部分是你可以编写多个 JobServices，每个 JobServices 定义一个不同的任务，这个任务应该在你的应用的某个时候执行。 这有助于模块化你的代码库，使代码维护变得更加简单。 所以这对于应用架构来说是个胜利。 

由于你正在使用 JobService 扩展另一个类，你将需要实现一些必要的方法: 

- [onStartJob( )](https://developer.android.com/reference/android/app/job/JobService.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#onStartJob%28android.app.job.JobParameters%29) 在执行作业时，系统会调用 onStartJob ( )。 如果你的任务是简短而简单的，可以直接在onStartJob ( ) 中实现逻辑，完成后返回 false，让系统知道所有的工作已经完成。但是如果你需要做一个更复杂的任务，比如连接到网络，你需要启动后台线程并返回 true，让系统知道你有一个线程仍在运行，它应该保持你的唤醒状态一段时间了。

> 注意：你的 JobService 将在主线程上运行。这意味着你需要在 onStartJob ( ) 方法中自己管理任何异步任务（如使用 [Thread](https://developer.android.com/reference/java/lang/Thread.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 或 [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 来打开网络连接，然后返回 true）。

通常，你的 onStartJob ( ) 会很小，因为它只是启动了一些别的东西。例如，在 Muzei 应用程序中，它基本上只是初始化一个 AsyncTask。 （如果你想看看这个任务是什么样子，可以在[GitHub](https://github.com/romannurik/muzei/blob/master/main/src/main/java/com/google/android/apps/muzei/sync/DownloadArtworkTask.java)上查看。

```java
@Override
public boolean onStartJob(final JobParameters params) {
  mDownloadArtworkTask = new DownloadArtworkTask(this) {
    @Override
    protected void onPostExecute(Boolean success) {
      jobFinished(params, !success);
    }
  };
  mDownloadArtworkTask.execute();
  return true;
}
```

- [jobFinished( )](https://developer.android.com/reference/android/app/job/JobService.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#jobFinished%28android.app.job.JobParameters,%20boolean%29)  jobFinished ( ) 不是一个你可以忽略的方法，系统不会调用它。 这是因为一旦你的服务或线程完成了工作，你就需要成为调用这个方法的人。 如果 onStartJob ( ) 方法启动另一个线程，然后返回 true，那么在工作完成时，你需要从该线程调用此方法。这是系统如何知道它可以安全地释放你的唤醒锁。如果你忘了调用 jobFinished ( )，你的应用程序将在电池统计列表看起来很内疚。jobFinished ( ) 需要两个参数：当前作业，以便知道哪个唤醒锁可以被释放，以及一个布尔值，指示是否要重新安排作业。 如果传递 true，这将启动 JobScheduler 的指数避退算法逻辑。

在 Muzei 应用程序中，jobFinished ( ) 是基于传入 onPostExecute ( ) 回调的成功值。这是一个非常简洁和简单的方法来包装你的应用程序可能正在执行的任何线程工作。 

```java
@Override
public boolean onStartJob(final JobParameters params) {
  mDownloadArtworkTask = new DownloadArtworkTask(this) {
    @Override
    protected void onPostExecute(Boolean success) {
      jobFinished(params, !success);
    }
  };
  mDownloadArtworkTask.execute();
  return true;
}
```

- [onStopJob( )](https://developer.android.com/reference/android/app/job/JobService.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#onStopJob%28android.app.job.JobParameters%29)  如果作业在完成之前被取消，则系统调用 onStopJob ( )。这通常发生在你的工作条件不再得到满足的情况下，例如设备已经被拔掉或者无法使用 WiFi。 因此，使用这种方法进行任何安全检查和清理，你可能需要做的回应一个半成品的工作。 然后，如果你希望系统重新安排工作，或者如果它不重要，系统会丢掉这个工作，那么请返回 true。大多数情况下，你不应该经常遇到这种情况，但是如果你遇到了很多麻烦，可以考虑试着缩短你的工作。 例如，如果你的下载从未完成，请你的服务器将下载分成更小的数据包，以便更快地检索到。 

在 Muzei 应用程序中，因为一个任务已经执行，如果它仍然运行，这个任务就需要被取消。 这是你需要非常小心的地方，因为任何挥之不去的线程都会在你的应用程序中产生内存泄漏。 所以，清理你的代码吧！ 

```java
@Override
public boolean onStopJob(final JobParameters params) {
  if (mDownloadArtworkTask != null) {
    mDownloadArtworkTask.cancel(true);
  }
  return true;
}
```

与任何服务一样，你需要将此服务添加到你的 AndroidManifest.xml 中。然而，不同的是，你需要添加一个permission  ，允许 JobScheduler 调用你的工作，并且是唯一可以访问你的 JobService 的 ~ 。 

```xml
<service
  android:name=".sync.DownloadArtworkJobService"
  android:permission="android.permission.BIND_JOB_SERVICE"
  android:exported="true"/>
```

### 第二，考虑一下运行你的工作必须符合的条件

记住: 对于 JobScheduler 来说，最大的优势在于它不是完全基于时间，而是基于条件。 (这意味着你不再需要每隔几个小时就会响一次警报，以便你可以检查现在是否适合与服务器同步。) 你可以通过 [JobInfo](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 对象定义这些条件。 要构建这个 JobInfo 对象，你每次都需要两个参数(然后标准是奖励位) : 一个工作号码ーー帮助你区分哪种工作ーー和 JobService。 

在 Muzei 应用中，唯一需要的条件是网络可用。 尽管这很简单，而且可以在任何时候完成，但由于几个原因，与 JobScheduler 一起安排工作仍然是有价值的。 首先，因为这个工作现在可以与需要网络访问的其他工作一起进行处理。 其次，因为使用 JobScheduler 意味着你不必担心工作的成败，因为 JobScheduler 会根据你的要求为你重新安排工作。 这项工作得到了保证，开发人员的工作量很小。(还有，你的应用程序可以免费使用 Doze-aware!) 

```java
JobScheduler jobScheduler =
    (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
jobScheduler.schedule(new JobInfo.Builder(LOAD_ARTWORK_JOB_ID,
    new ComponentName(this, DownloadArtworkJobService.class))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY)
    .build());
```

让我们花一点时间来谈谈你可以在你的 JobInfo 对象中包含的潜在标准。 

- [Network type](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setRequiredNetworkType%28int%29)（计量/无计量）
  如果你的工作需要网络访问，则必须包含此条件。 你可以指定一个计量表或无计量网络，或任何类型的网络。 但是在构建你的 JobInfo 时不会调用这个功能意味着系统将假设你不需要任何网络访问，你将无法联系你的服务器。 

- [Charging](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setRequiresCharging%28boolean%29) 和 [Idle](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setRequiresDeviceIdle%28boolean%29)

  如果你的应用程序需要进行资源密集的工作，那么建议你等到设备插上电源或者空闲时再动手。（请注意，这里的"空闲"与 Doze 的空闲模式不同，只是意味着屏幕关闭，而且已经有一段时间没用了 ）

- Content Provider update

  从 API 24开始，你现在可以使用内容提供者更改作为触发器来执行某些工作。 你将需要指定将通过 ContentObserver 进行监视的[触发器 URI](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#addTriggerContentUri%28android.app.job.JobInfo.TriggerContentUri%29)。 如果你想要确保所有更改在作业运行之前传播，你也可以在作业被触发之前指定[延迟](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setTriggerContentMaxDelay%28long%29)。

- [Backoff criteria](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setBackoffCriteria%28long,%20int%29)

  你可以指定自己的回退 / 重试策略。 这个默认为指数策略，但是如果你设置你自己的，然后返回 true 的重新安排一个工作(例如，onStopJob ( ) ），系统将使用你指定的策略来处理默认情况 。


- [Minimum latency](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setMinimumLatency%28long%29) 和 [override deadline](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setOverrideDeadline%28long%29)

  如果你的工作不能在至少 X 的时间内启动，或者不能延迟超过特定的时间，你可以在这里指定这些值。 即使所有条件都没有满足，你的工作将在截止日期之前运行（你可以检查

  [isOverrideDeadlineExpired ( )](https://developer.android.com/reference/android/app/job/JobParameters.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#isOverrideDeadlineExpired%28%29) 的结果以确定你是否在这种情况下）。 如果他们得到满足，但你的最低延迟还没有过去，你的工作将会举行。

- [Periodic](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setPeriodic%28long%29)

  如果你有工作需要定期完成，你可以设置一个定期的工作。 对于大多数开发人员来说，这是一个重复警报的好选择。 因为你把它全部整理一遍，安排它，每一个指定的时间内，作业都会运行一次。

- [Persistent](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setPersisted%28boolean%29)

  任何需要在重新启动时保留的工作都可以在这里标记。 一旦设备重新启动，作业将根据条件重新安排。 （请注意，尽管你的应用程序需要 RECEIVE_BOOT_COMPLETED 权限才能工作。）

- [Extras](https://developer.android.com/reference/android/app/job/JobInfo.Builder.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#setExtras%28android.os.PersistableBundle%29)

  如果你的工作需要一些来自应用程序的信息来执行它的工作，你可以把原始数据类型作为附加信息传递到 JobInfo 中 

## 最后，考虑何时安排你的工作。

你可以使用 [JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog) 来安排一项工作，你将从系统中检索该工作。然后，使用 [schedule ( )](https://developer.android.com/reference/android/app/job/JobScheduler.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog#schedule%28android.app.job.JobInfo%29) 调用你的 JobInfo 对象 ，你就可以开始了。 不需要担心。 

```java
JobScheduler jobScheduler =
    (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
jobScheduler.schedule(new JobInfo.Builder(LOAD_ARTWORK_JOB_ID,
    new ComponentName(this, DownloadArtworkJobService.class))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY)
    .build());
```

## 接下来是什么？

JobScheduler 正成为在 Android 中执行后台工作的最佳答案。 Android Nougat 引入了[几个后台优化](https://developer.android.com/topic/performance/background-optimization.html?utm_campaign=adp_series_job_scheduler_092216&utm_source=medium&utm_medium=blog)，JobScheduler 是最佳实践解决方案。 所以，如果你还没有，现在是跳到 JobScheduler 列车上的时候了。

可以肯定的是，有很多部分，你需要仔细考虑什么时候应该触发你的工作，以及如果由于某种原因失败会发生什么。 但总的来说，JobScheduler 的设计很容易。 如果你想看到一个真实的例子，请随时查看 [Muzei 应用程序](https://github.com/romannurik/muzei/tree/master/main/src/main/java/com/google/android/apps/muzei/sync)。 （该应用程序甚至可以很好地区分 Lollipop 上的 JobScheduler 和更老的版本上的 AlarmManager）然后，你可以尝试一下，这样你就可以建立更好的应用程序。 

#BuildBetterApps。

> 小记：本次发布的 Muzei Play 商店版本尚未更新，但代码位于 GitHub 上。所以你必须自己构建才能看到它的实际应用。


