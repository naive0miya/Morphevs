# 使用片段处理配置更改

> 原文 (androiddesignpatterns.com) ：[Handling Configuration Changes with Fragments](https://www.androiddesignpatterns.com/2013/04/retaining-objects-across-config-changes.html)
>
> 作者 ：[Alex Lockwood](https://google.com/+AlexLockwood)

这篇文章提出了一个常见的问题，这个[问题](http://stackoverflow.com/q/3821423/844882)在 StackOverflow 上经常被问到:

> 保留活动对象的最佳方法是什么? 如运行 Thread、Socket 和 AsyncTask ーー设备配置的更改？

要回答这个问题，我们将首先讨论开发人员在将长时间运行的后台任务与 Activity 生命周期结合使用时遇到的一些常见困难。 然后，我们将描述解决问题的两种常见方法的缺陷 最后，我们将用示例代码来总结推荐的解决方案，该解决方案使用保留的片段来实现我们的目标。

### 配置更改和后台任务

配置更改以及活动经历的销毁和创建周期带来的一个问题是，这些事件是不可预测的，并且随时可能发生。 并发的后台任务只会增加这个问题。 例如，假设一个 Activity 启动了一个 AsyncTask，在用户旋转屏幕后不久，就会导致活动被破坏和重新创建。 当 AsyncTask 最终完成其工作时，它会错误地将其结果报告给旧的活动实例，完全不知道创建了一个新的活动。 好像这还不只一个问题，新的 Activity 实例可能会浪费宝贵的资源，因为它会再次启动后台工作，而不知道旧的 AsyncTask 仍在运行。 由于这些原因，当发生配置更改时，我们正确有效地在活动实例中保留活动对象至关重要。

### 糟糕的做法: 保留活动

也许最猖獗和最广泛滥用的解决方法是通过在 Android 清单中设置 Android: configChanges 属性来禁用默认的销毁和重建行为。 这种简单的方法使得它对开发者极具吸引力; 但，[谷歌的工程师](http://stackoverflow.com/a/5336057/844882)不鼓励使用它。 主要的担心是，它要求你手动处理代码中的设备配置更改。 处理配置更改需要你采取许多额外步骤，以确保每个字符串、布局、可绘制、维度等都与设备的当前配置保持同步，如果你不小心，你的应用程序可以很容易地产生一系列针对特定资源的 bug。

谷歌不鼓励使用它的另一个原因是，许多开发者错误地认为设置 android: configChanges="orientation"(举例来说)将神奇地保护他们的应用程序，使其免受不可预测的场景的影响，在这种情况下，底层活动将被销毁和重新创建。 事实并非如此。 配置更改可能出于多种原因，而不仅仅是屏幕方向的改变。 将你的设备插入一个显示坞，改变默认语言，并修改设备的默认字体缩放因子只是触发设备配置更改事件的三个的例子，所有这些都会通知系统将在下次恢复时销毁和重新创建所有当前运行的活动。 因此，设置 android:configChanges 属性通常不是很好的实践。

### 删除: Override onRetainNonConfigurationInstance ( )

在 Honeycomb 发布之前，建议在活动实例中传输活动对象的方法是覆盖 onRetainNonConfigurationInstance ( ) 和 getLastNonConfigurationInstance ( ) 方法。 使用这种方法，在活动实例中传输一个活动对象仅仅是将活动对象返回到 onRetainNonConfigurationInstance ( ) 并在 getLastNonConfigurationInstance ( ) 中检索它。 至于 API 13，这些方法已经被删除，以支持更多的 Fragment 的 setRetainInstance (boolean) 能力，它提供了一个更简洁和模块化的方法来在配置更改中保留对象。 我们将在下一节讨论这种基于片段的方法。

### 建议: 管理保留片段中的对象

自从在 Android 3.0中引入片段以来，在活动实例中保留活动对象的推荐方法是在保留的"工作者"片段中包装和管理它们。 默认情况下，当发生配置更改时，片段会与其父 Activitys 一起被销毁和重新创建。 调用 片段 Fragment#setRetainInstance(true) 允许我们绕过这个销毁和重建循环，在活动重新创建时，系统将保留片段的当前实例。 正如我们将看到的，这对于持有如运行线程，异步任务，套接字等等对象的片段非常有用。

下面的示例代码是如何使用保留的片段在配置更改中保留异步任务的基本示例。 该代码保证将进度更新和结果传回当前显示的活动实例，并确保我们在配置更改期间异步任务绝不会意外地泄漏。 这个设计包括两个类，一个 MainActivity。

```Java
/**
 * This Activity displays the screen's UI, creates a TaskFragment
 * to manage the task, and receives progress updates and results 
 * from the TaskFragment when they occur.
 */
public class MainActivity extends Activity implements TaskFragment.TaskCallbacks {

  private static final String TAG_TASK_FRAGMENT = "task_fragment";

  private TaskFragment mTaskFragment;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);

    FragmentManager fm = getFragmentManager();
    mTaskFragment = (TaskFragment) fm.findFragmentByTag(TAG_TASK_FRAGMENT);

    // If the Fragment is non-null, then it is currently being
    // retained across a configuration change.
    if (mTaskFragment == null) {
      mTaskFragment = new TaskFragment();
      fm.beginTransaction().add(mTaskFragment, TAG_TASK_FRAGMENT).commit();
    }

    // TODO: initialize views, restore saved state, etc.
  }

  // The four methods below are called by the TaskFragment when new
  // progress updates or results are available. The MainActivity 
  // should respond by updating its UI to indicate the change.

  @Override
  public void onPreExecute() { ... }

  @Override
  public void onProgressUpdate(int percent) { ... }

  @Override
  public void onCancelled() { ... }

  @Override
  public void onPostExecute() { ... }
}
```

还有一个任务片段。

```java
/**
 * This Fragment manages a single background task and retains 
 * itself across configuration changes.
 */
public class TaskFragment extends Fragment {

  /**
   * Callback interface through which the fragment will report the
   * task's progress and results back to the Activity.
   */
  interface TaskCallbacks {
    void onPreExecute();
    void onProgressUpdate(int percent);
    void onCancelled();
    void onPostExecute();
  }

  private TaskCallbacks mCallbacks;
  private DummyTask mTask;

  /**
   * Hold a reference to the parent Activity so we can report the
   * task's current progress and results. The Android framework 
   * will pass us a reference to the newly created Activity after 
   * each configuration change.
   */
  @Override
  public void onAttach(Activity activity) {
    super.onAttach(activity);
    mCallbacks = (TaskCallbacks) activity;
  }

  /**
   * This method will only be called once when the retained
   * Fragment is first created.
   */
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Retain this fragment across configuration changes.
    setRetainInstance(true);

    // Create and execute the background task.
    mTask = new DummyTask();
    mTask.execute();
  }

  /**
   * Set the callback to null so we don't accidentally leak the 
   * Activity instance.
   */
  @Override
  public void onDetach() {
    super.onDetach();
    mCallbacks = null;
  }

  /**
   * A dummy task that performs some (dumb) background work and
   * proxies progress updates and results back to the Activity.
   *
   * Note that we need to check if the callbacks are null in each
   * method in case they are invoked after the Activity's and
   * Fragment's onDestroy() method have been called.
   */
  private class DummyTask extends AsyncTask<Void, Integer, Void> {

    @Override
    protected void onPreExecute() {
      if (mCallbacks != null) {
        mCallbacks.onPreExecute();
      }
    }

    /**
     * Note that we do NOT call the callback object's methods
     * directly from the background thread, as this could result 
     * in a race condition.
     */
    @Override
    protected Void doInBackground(Void... ignore) {
      for (int i = 0; !isCancelled() && i < 100; i++) {
        SystemClock.sleep(100);
        publishProgress(i);
      }
      return null;
    }

    @Override
    protected void onProgressUpdate(Integer... percent) {
      if (mCallbacks != null) {
        mCallbacks.onProgressUpdate(percent[0]);
      }
    }

    @Override
    protected void onCancelled() {
      if (mCallbacks != null) {
        mCallbacks.onCancelled();
      }
    }

    @Override
    protected void onPostExecute(Void ignore) {
      if (mCallbacks != null) {
        mCallbacks.onPostExecute();
      }
    }
  }
}
```

### 事件的流动

当 MainActivity 第一次启动时，它会实例化并将 TaskFragment 添加到活动的状态。 任务片段创建并执行一个 AsyncTask，并通过 TaskCallbacks 接口将进度更新和结果反馈给 MainActivity。 当一个配置更改发生时，MainActivity 会经历其正常的生命周期事件，一旦创建了新的活动实例，将传递给 onAttach (Activity)方法，从而确保任务片段即使在配置更改之后始终保存对当前显示的活动实例的引用。 由此产生的设计既简单又可靠; 应用程序框架将处理重新分配活动实例，因为它们被拆除和重新创建，任务片段及其异步任务从不需要担心配置更改的不可预测性。 还要注意的是，在 onDetach ( ) 和 onAttach ( ) 的调用之间不可能执行 onPostExecute ( ) ，正如 [StackOverflow 的回答](http://stackoverflow.com/q/19964180/844882)和我在[这个 Google + 文章](https://plus.google.com/u/0/+AlexLockwood/posts/etWuiiRiqLf)中对 Doug Stevenson 的回复(在下面的评论中也有一些关于这个的讨论)。

### 结论

使用活动生命周期同步后台任务可能很棘手，而配置更改只会增加混乱。 幸运的是，保留的片段使得处理这些事件变得非常容易，因为它们一直保持对其父活动的引用，即使在被销毁和重新创建之后。

一个示例应用程序说明如何正确使用保留的片段来达到这一效果，可在 [Play Store](https://play.google.com/store/apps/details?id=com.adp.retaintask) 下载。 源代码可以在 [GitHub](https://github.com/alexjlockwood/worker-fragments) 上找到。 下载它，导入到 Eclipse，然后修改你想要的一切！

