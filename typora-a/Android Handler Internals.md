# Android Handler 机制

> 原文 (Medium)：[Android Handler Internals](https://medium.com/@jagsaund/android-handler-internals-b5d49eba6977)
>
> 作者：[Jag Saund](https://medium.com/@jagsaund)

[TOC]

为了让 Android 应用程序响应，你需要防止 UI 线程阻塞。 当阻塞或计算密集型任务被卸载到工作线程时，响应性也会增加。 这些操作的结果通常需要更新用户界面组件，这些组件必须在 UI 线程上执行。 阻塞队列、共享内存和管道等机制给用户界面线程带来了阻塞问题。 为了避免这个问题，Android 提供了自己的消息传递机制——处理程序。 处理程序是 Android 框架的一个基本组件。 它提供了一个非阻塞消息传递机制。 在消息移交过程中，无论是生产者还是消费者都不会阻止。 

尽管处理程序经常被使用，但是很容易忽略它们是如何工作的。 本文将更深入地研究处理程序的各个组成部分。 它解释了为什么 Handler 是一个跨越工作线程和 UI 线程通信的强大组件。 

## 图像查看器示例

 让我们从一个例子开始，说明如何在应用程序中使用处理程序。 考虑一个需要显示来自网络的图像的活动。 有几种方法可以做到这一点。 在这个例子中，我们将启动一个新的工作线程来执行网络调用并检索图像有效载荷¹。  

```java
public class MainActivity extends AppCompatActivity {
    private static final String IMAGE_URL = "https://www.android.com/static/2016/img/logo-android-green_2x.png";

    private ProgressBar progressIndicator;
    private ImageView imageView;
    private Handler handler;

    class ImageFetcher implements Runnable {
        final String url;

        ImageFetcher(String url) {
            this.url = url;
        }

        @Override
        public void run() {
            // download image -- code removed
            final Bitmap bitmap = BitmapFactory.decodeStream(inputstream);
            handler.post(new Runnable() {
                @Override
                public void run() {
                    imageView.setImageBitmap(bitmap);
                }
            });
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        final Thread workerThread = new Thread(new ImageFetcher(IMAGE_URL));
        workerThread.start();
    }
}
```

另一种方法是使用 Handler  消息而不是 Runnables。

```java
public class MainActivity extends AppCompatActivity {
    private static final String IMAGE_URL = "https://www.android.com/static/2016/img/logo-android-green_2x.png";
    private static final int MSG_SHOW_PROGRESS = 1;
    private static final int MSG_SHOW_IMAGE = 2;

    private ProgressBar progressIndicator;
    private ImageView imageView;
    private Handler handler;

    class ImageFetcher implements Runnable {
        final String url;

        ImageFetcher(String url) {
            this.url = url;
        }

        @Override
        public void run() {
            handler.obtainMessage(MSG_SHOW_PROGRESS).sendToTarget();
            // download image -- code removed
            final Bitmap bitmap = BitmapFactory.decodeStream(inputstream);
            handler.obtainMessage(MSG_SHOW_IMAGE, bitmap).sendToTarget();
        }
    }

    class UIHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SHOW_PROGRESS: {
                    progressIndicator.setVisibility(View.VISIBLE);
                    break;
                }
                case MSG_SHOW_IMAGE: {
                    progressIndicator.setVisibility(View.GONE);
                    imageView.setImageBitmap((Bitmap) msg.obj);
                    break;
                }
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        handler = new UIHandler();
        final Thread workerThread = new Thread(new ImageFetcher(IMAGE_URL));
        workerThread.start();
    }
}
```

在第二个例子中，一个工作线程从网络中获取图像。 一旦图片下载，我们需要更新 ImageView 与位图。 我们知道我们不能从非 UI 线程中触摸 UI 组件，所以我们使用处理程序。 处理程序充当工作线程和 UI 线程之间的中介。 此消息由处理程序对工作线程进行了查询，并由处理程序在 UI 线程上处理。 

## 处理程序内部深入了解

Handler 的组件是：

- Handler 处理程序 
- Message 信息 
- Message Queue 消息队列 
- Looper 轮询器

我们将看看每个组件，看看它们是如何相互影响的。

### Handler

[处理程序](https://developer.android.com/reference/android/os/Handler.html)是在线程之间传递消息的直接接口。 消费者线程和生产线程通过调用下列操作与处理程序进行交互: 

- 从消息队列创建、插入或删除消息 
- 处理消费者线程上的消息 

![android.os.Handler组件](https://ws2.sinaimg.cn/large/006tNc79gy1fron1ewo1ej30m8088mxl.jpg)

每个处理程序都与一个 Looper 和一个消息队列相关联。有两种方法来创建一个处理程序：

- 通过使用与当前线程关联的 Looper 的默认构造函数
- 通过明确指定使用哪个 Looper

如果没有 Looper，Handler 将无法运行，因为它不能将消息放入 Message Queue 中。因此，它不会收到任何消息来处理。

```java
public Handler(Callback callback, boolean async) {
    // code removed for simplicity
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException( “Can’t create handler inside thread that has not called Looper.prepare()”);
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

[Android Source](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/Handler.java#L188)（上面）的片段演示了构建新 Handler 的逻辑。 处理程序检查当前线程是否有有效的 Looper。 如果不是，则会引发运行时异常。 然后处理程序接收到一个引用到 Looper 的消息队列。 

**注意：**与同一个线程关联的多个处理程序共享相同的消息队列，因为它们共享同一个 Looper。

回调是一个可选的参数。如果提供，则处理 Looper 发送的消息。

### Message

该[消息](https://developer.android.com/reference/android/os/Message.html)充当任意数据的容器。 生产者线程将消息发送到处理程序，该处理程序排入消息队列。 消息提供了Handler 和 Message Queue 处理消息所需的三条附加信息：

- what - 处理程序可用于区分消息和处理不同信息的标识符 
- time - 通知消息队列何时处理消息
- target - 指示哪个处理程序应该处理消息

![android.os.Message组件](https://ws3.sinaimg.cn/large/006tNc79gy1fron1j6u2kj30m808i0tj.jpg)

消息创建通常使用以下处理程序方法之一：

```java
public final Message obtainMessage()
public final Message obtainMessage(int what)
public final Message obtainMessage(int what, Object obj)
public final Message obtainMessage(int what, int arg1, int arg2)
public final Message obtainMessage(int what, int arg1, int arg2, Object obj)
```

消息是从消息池中获取的。提供的参数填充消息的字段。处理程序还将消息的目标设置为自己。这使我们能够像这样连接调用：

```java
mHandler.obtainMessage(MSG_SHOW_IMAGE, mBitmap).sendToTarget();
```

消息池是 Message 对象的 LinkedList，其最大池大小为50.处理程序处理消息后，消息队列将该对象返回到池并重置所有字段。

当通过 post（Runnable r）向处理程序发布 Runnable 时，Handler 隐式构造一个新的 Message。它还设置回调字段来保存 Runnable。

```java
Message m = Message.obtain();
m.callback = r;
```

![生产者线程发送消息与处理程序的交互](https://ws1.sinaimg.cn/large/006tNc79gy1fron1nqukwj30m80dagn8.jpg)

在这一点上，我们可以看到生产者线程和处理程序之间的交互。 生产者创建一个消息并将其发送给处理程序。 处理程序然后将消息排入消息队列。 处理程序将来会在消费者线程上处理消息。

### Message Queue

[消息队列](https://developer.android.com/reference/android/os/MessageQueue.html)是消息对象的无限制 LinkedList 。它按时间顺序插入消息，最低时间戳首先发送。

![android.os.MessageQueue组件](https://ws3.sinaimg.cn/large/006tNc79gy1fron1r3jabj30m80bswf4.jpg)

MessageQueue 还根据 SystemClock.uptimeMillis 维护一个表示当前时间的调度屏障。当消息时间戳小于此值时，消息由处理程序调度和处理。

处理程序提供了三种发送消息的变体：

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
public final boolean sendMessageAtFrontOfQueue(Message msg)
public boolean sendMessageAtTime(Message msg, long uptimeMillis)
```

通过延迟发送消息将消息的时间字段设置为 SystemClock.uptimeMillis ( ) + delayMillis。

发送延迟的消息将时间字段设置为 SystemClock.uptimeMillis ( ) + delayMillis。 而发送到队列前面的消息将时间字段设置为 0，并在下一个消息循环迭代中进行处理。 小心使用此方法，因为它可能会使消息队列挨饿，导致排序问题，或者有其他意想不到的副作用。

处理程序通常与引用活动的 UI 组件相关联。处理程序返回到这些组件的引用可能会泄漏活动。考虑以下情况：

```java
public class MainActivity extends AppCompatActivity {
    private static final String IMAGE_URL = "https://www.android.com/static/img/android.png";

    private static final int MSG_SHOW_PROGRESS = 1;
    private static final int MSG_SHOW_IMAGE = 2;

    private ProgressBar progressIndicator;
    private ImageView imageView;
    private Handler handler;

    class ImageFetcher implements Runnable {
        final String imageUrl;

        ImageFetcher(String imageUrl) {
            this.imageUrl = imageUrl;
        }

        @Override
        public void run() {
            handler.obtainMessage(MSG_SHOW_PROGRESS).sendToTarget();
            InputStream is = null;
            try {
                // Download image over the network
                URL url = new URL(imageUrl);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();

                conn.setRequestMethod("GET");
                conn.setDoInput(true);
                conn.connect();
                is = conn.getInputStream();

                // Decode the byte payload into a bitmap
                final Bitmap bitmap = BitmapFactory.decodeStream(is);
                handler.obtainMessage(MSG_SHOW_IMAGE, bitmap).sendToTarget();
            } catch (IOException ignore) {
            } finally {
                if (is != null) {
                    try {
                        is.close();
                    } catch (IOException ignore) {
                    }
                }
            }
        }
    }

    class UIHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SHOW_PROGRESS: {
                    imageView.setVisibility(View.GONE);
                    progressIndicator.setVisibility(View.VISIBLE);
                    break;
                }
                case MSG_SHOW_IMAGE: {
                    progressIndicator.setVisibility(View.GONE);
                    imageView.setVisibility(View.VISIBLE);
                    imageView.setImageBitmap((Bitmap) msg.obj);
                    break;
                }
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        progressIndicator = (ProgressBar) findViewById(R.id.progress);
        imageView = (ImageView) findViewById(R.id.image);

        handler = new UIHandler();

        final Thread workerThread = new Thread(new ImageFetcher(IMAGE_URL));
        workerThread.start();
    }
}
```

在这个例子中，Activity 启动一个新的工作者线程来下载并在 ImageView 中显示一个图像。 工作者线程通过 UIHandler 传递 UI 更新，UIHandler 保留对视图的引用并更新其状态（切换可见性，设置位图）。

假设由于网络速度较慢，工作线程正在花费很长时间来下载镜像。 在工作线程完成之前销毁活动会导致活动泄漏。 在这个例子中有两个强大的引用。 一个在工作线程和 UIHandler 之间，另一个在 UIHandler 和视图之间。 这可以防止垃圾收集器收回活动引用。

现在我们来看另一个例子：

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "Ping";

    private Handler handler;

    class PingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            Log.d(TAG, "Ping message received");
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        handler = new PingHandler();
        
        final Message msg = handler.obtainMessage();
        handler.sendEmptyMessageDelayed(0, TimeUnit.MINUTES.toMillis(1));
    }
}
```

在这个例子中，发生以下一系列事件：

- PingHandler 被创建
- 该活动发送一个延迟消息给处理程序，该消息程序排入 MessageQueue
- 该消息在发送之前被销毁
- 消息由 UIHandler 调度和处理，并输出日志语句

虽然起初可能并不明显，但这个例子中的 Activity 也是有漏洞的。

在销毁活动之后，处理程序引用应该可用于垃圾收集。但是，当我们创建一个 Message 对象时，它保留了对 Handler 的引用：

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

Android 源代码片段（上面）显示发送给 Handler 的所有消息最终调用 enqueueMessage。 请注意，处理程序引用已明确分配给 msg.target。 这告诉 Looper 从 MessageQueue 中检索哪个 Handler 应该处理消息。

Message 被添加到 MessageQueue 中，MessageQueue 现在持有对 Message 的引用。 MessageQueue 也有一个与之相关的 Looper。 一个明确的 Looper 会一直存在，直到它被终止，而 Main Looper 只要应用程序一直存在。 只要消息不被 MessageQueue 回收，处理程序引用就处于活动状态。 一旦它被回收，它的字段，包括目标引用，被清除。

尽管对处理程序有很长的参考，但活动是否泄漏尚不清楚。 为了检查泄漏，我们必须确定处理程序是否也引用类中的活动。 在这个例子中，它的确如此。 有一个隐式引用由非静态类成员保留给它的封闭类。 具体来说，PingHandler 没有被声明为一个静态类，所以它有一个对 Activity 的隐式引用。

使用 WeakReference 和静态类修饰符的组合可防止处理程序泄漏活动。 当活动被销毁时，WeakReference 允许垃圾收集器收回你想保留的对象（通常是一个活动）。 内部 Handler 类的静态修饰符可以防止对父类的隐式引用。

让我们来修改我们的 UIHandler 来解决这个问题：

```java
static class UIHandler extends Handler {
    private final WeakReference<ImageFetcherActivity> mActivityRef;
    
    UIHandler(ImageFetcherActivity activity) {
        mActivityRef = new WeakReference(activity);
    }
    
    @Override
    public void handleMessage(Message msg) {
        final ImageFetcherActivity activity = mActivityRef.get();
        if (activity == null) {
            return
        }
        
        switch (msg.what) {
            case MSG_SHOW_LOADER: {
                activity.progressIndicator.setVisibility(View.VISIBLE);
                break;
            }
            case MSG_HIDE_LOADER: {
                activity.progressIndicator.setVisibility(View.GONE);
                break;
            }
            case MSG_SHOW_IMAGE: {
                activity.progressIndicator.setVisibility(View.GONE);
                activity.imageView.setImageBitmap((Bitmap) msg.obj);
                break;
            }
        }
    }
}
```

现在，UIHandler 构造函数接受包装在 WeakReference 中的 Activity。 这允许垃圾收集器在活动被销毁时收回活动引用。 要与 Activity 的 UI 组件进行交互，我们需要对来自 mActivityRef 的 Activity 的强有力的引用。 由于我们使用的是 WeakReference，所以在访问 Activity 时一定要谨慎。 如果活动引用的唯一路径是通过 WeakReference，那么垃圾收集器可能已经收回它。 我们需要检查是否发生。 如果有的话，处理程序实际上是不相关的，应该忽略这些信息。

尽管这个逻辑解决了内存泄漏问题，但仍然存在问题。 活动已经被销毁，但垃圾收集器还没有收回参考。 根据正在执行的操作，这可能会导致应用程序崩溃。 要解决这个问题，我们需要检测活动的状态。

让我们更新 UIHandler 逻辑来考虑这些情况：

```java
static class UIHandler extends Handler {
    private final WeakReference<ImageFetcherActivity> mActivityRef;
    
    UIHandler(ImageFetcherActivity activity) {
        mActivityRef = new WeakReference(activity);
    }
    
    @Override
    public void handleMessage(Message msg) {
        final ImageFetcherActivity activity = mActivityRef.get();
        if (activity == null || activity.isFinishing() || activity.isDestroyed()) {
            removeCallbacksAndMessages(null);
            return
        }
        
        switch (msg.what) {
            case MSG_SHOW_LOADER: {
                activity.progressIndicator.setVisibility(View.VISIBLE);
                break;
            }
            case MSG_HIDE_LOADER: {
                activity.progressIndicator.setVisibility(View.GONE);
                break;
            }
            case MSG_SHOW_IMAGE: {
                activity.progressIndicator.setVisibility(View.GONE);
                activity.imageView.setImageBitmap((Bitmap) msg.obj);
                break;
            }
        }
    }
}
```

现在，我们可以概括一下 MessageQueue， Handler 和 Producer Threads 之间的交互：

![MessageQueue，处理程序和生产者线程之间的交互](https://ws1.sinaimg.cn/large/006tNc79gy1fron1ynziyj30m80c6js9.jpg)

在上图中，多个生产者线程将消息提交给不同的处理程序。 但是，每个 Handler 都与同一个 Looper 关联，所以所有的 Message 都发布到同一个 MessageQueue。 这很重要，因为 Android 创建了几个与主循环相关的处理程序：

- Choreographer：处理 vsync 和帧更新
- ViewRoot：处理输入和窗口事件，配置
- InputMethodManager： 处理键盘触摸事件
- 还有其他几个

**提示：**确保生产者线程不会产生几个消息，因为它们可能会导致处理系统生成的消息。

![主循环者派出的一小部分事件](https://ws2.sinaimg.cn/large/006tNc79gy1fron22uvvaj30m803iwh2.jpg)

**调试提示：**

你可以通过附加 LogPrinter 来调试/转储由 Looper 调度的所有消息：

```java
final Looper looper = getMainLooper();
looper.setMessageLogging(new LogPrinter(Log.DEBUG, "Looper"));
```

同样，你可以通过将 LogPrinter 附加到你的处理程序来调试/转储与 Handler 关联的 MessageQueue 中的所有待处理消息：

```java
handler.dump(new LogPrinter(Log.DEBUG, "Handler"), "");
```

### Looper

 [Looper](https://developer.android.com/reference/android/os/Looper.html) 从消息队列中读取消息，并将执行分派给目标处理程序。 一旦消息通过调度屏障，就有资格让循环者在下一个消息循环中读取它。 当没有消息符合发送条件时，Looper 会阻塞。 当消息可用时恢复。

只有一个 Looper 可以与一个线程关联。将另一个 Looper 附加到线程会导致 RuntimeException。在 Looper 类中使用静态 ThreadLocal 对象确保只有一个 Looper 被附加到 Thread。

调用 Looper.quit 立即终止 Looper。 它还放弃通过调度屏障的消息队列中的所有消息。 调用 Looper.quitSafely 可确保所有准备好分派的消息在未决消息被丢弃之前得到处理。

![Handler与MessageQueue和Looper交互的整体流程](https://ws2.sinaimg.cn/large/006tNc79gy1fron26hsk8j30m80arjs1.jpg)

Looper 是在 Thread 的 run ( ) 方法中设置的。 对静态方法 Looper.prepare ( ) 的调用将检查先前存在的 Looper 是否与此线程相关联。 它通过使用  Looper 的 ThreadLocal 来检查一个 Looper 对象是否已经存在。 如果没有，则会创建一个新的 Looper 对象和一个新的 MessageQueue。 [Android 源代码](https://github.com/android/platform_frameworks_base/blob/e71ecb2c4df15f727f51a0e1b65459f071853e35/core/java/android/os/Looper.java#L83)片段（下面）演示了这一点。

**注意：**公共 prepare Looper 方法在内部调用 prepare (true)。

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException(“Only one Looper may be created per thread”);
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Handler 现在可以接收消息并将其添加到消息队列中。 执行静态 Looper.loop ( ) 方法将开始读取队列中的消息。 每个循环迭代检索下一条消息，将其分发到目标处理程序，并将其回收到消息池。 Looper.loop 将继续这个过程，直到 Looper 被终止。 [Android 源代码](https://github.com/android/platform_frameworks_base/blob/e71ecb2c4df15f727f51a0e1b65459f071853e35/core/java/android/os/Looper.java#L123)片段（下面）演示了这一点：

```java
public static void loop() {
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
    }
}
```

没有必要创建自己的附加 Looper 的线程。 Android 为此提供了一个便利的类 - HandlerThread。 它扩展了 Thread 类并管理 Looper 的创建。 下面的代码段描述了一个典型的使用模式

```java
private final Handler handler;
private final HandlerThread handlerThread;
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate();
    handlerThread = new HandlerThread("HandlerDemo");
    handlerThread.start();
    handler = new CustomHandler(handlerThread.getLooper());
}
@Override
protected void onDestroy() {
    super.onDestroy();
    handlerThread.quit();
}
```

onCreate ( ) 方法构造一个新的 HandlerThread。当 HandlerThread 启动时，它准备了 Looper 并将其附加到线程中。 Looper 现在开始处理 HandlerThread 上 MessageQueue 的消息。

**注意：**当活动被销毁时，终止 HandlerThread 是很重要的。这也终止了 Looper。

## 结束

Android Handler 在应用程序的生命周期中起着不可或缺的作用。 它设置了半同步/半异步架构模式的基础。 各种内部和外部资源依靠 Handler 进行异步事件分派，因为它可以最小化开销并保持线程安全。

更深入地了解组件如何工作有助于解决难题。 它还使我们能够最佳地使用组件的 API。 我们经常使用 Handler 作为工作者到 UI 线程通信的机制，但不止于此。 处理程序出现在 [IntentService](https://developer.android.com/reference/android/app/IntentService.html)，[Camera2](https://developer.android.com/reference/android/hardware/camera2/CameraCaptureSession.html) API 和其他许多应用程序中。 在这些 API 中，更常用的是专注于任意线程之间的通信。

我们可以对 Handler 的这个更深入的理解来构建更高效，简单和强大的应用程序。

¹ 这个例子演示了在后台线程中执行阻塞工作，并利用 Handler 来更新 UI 组件。 有几个图像加载和缓存库可用，应该使用，而不是实现自己的图像管理逻辑。

## 阅读

1. [Android Source Code](https://github.com/android/platform_frameworks_base/tree/master/core/java/android/os)
2. [Android Developer Documentation](https://developer.android.com/reference/android/os/Handler.html)
3. [Efficient Android Threading: Asynchronous Processing Techniques for Android Applications](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/)

