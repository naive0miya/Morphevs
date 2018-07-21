# Android 系统使用 ThreadPoolExecutor

> 原文 (Medium)：[Using ThreadPoolExecutor in Android](https://medium.com/mindorks/threadpoolexecutor-in-android-8e9d22330ee3)
>
> 作者：[Amit Shekhar](https://medium.com/@amitshekhar?source=post_header_lockup)

[TOC]

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovc6zkqwj30m80dwwlh.jpg)

本文将介绍 pool，ThreadPoolExecutor 及其在 Android 中的使用。我们将通过大量示例代码深入讨论这些主题。

## 线程池

线程池管理工作线程池(确切数目视执行方式而异)。 

任务队列保存任务，等待池中任何一个空闲线程执行的任务。 生产者将任务添加到队列中，而当有一个准备执行新的后台执行的空闲线程时，工作线程就以消费者的身份执行任务。 

## 线程池执行器

ThreadPoolExecutor 使用线程池中的一个线程执行给定的任务。

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
int corePoolSize,
int maximumPoolSize,
long keepAliveTime,
TimeUnit unit,
BlockingQueue<Runnable> workQueue
);
```

这些参数是什么？

1. corePoolSize : 保留在池中的最小线程数。 最初在池中没有线程。 但是随着任务被添加到队列中，新的线程被创建。 如果有空闲的线程 - 但线程数低于 corePoolSize - 则新线程将继续被创建。
2. maximumPoolSize : 池中允许的最大线程数。 如果这个大小超过了 corePoolSize，而且当前线程的数目是 corePoolSize  ，那么只有当队列已满时，才会创建新的工作线程。
3. keepAliveTime : 当线程数大于核心时，非核心线程（多余的空闲线程）将等待新的任务，如果他们没有在这个参数定义的时间内得到一个，它们就会终止 。
4. unit :  keepAliveTime 的时间单位。
5. workQueue : 任务队列，它只会保存可运行的任务。 它必须是一个 [BlockingQueue](http://developer.android.com/reference/java/util/concurrent/BlockingQueue.html)。

## 为什么在 Android 或 JAVA 应用程序中使用线程池执行器？

1. 这是一个强大的任务执行框架，它支持在队列中添加任务、取消任务和任务优先级 。
2. 它减少了与线程创建相关联的开销，因为它在其线程池中管理一定数量的线程 。

#### 使用 Android 中的线程池执行器

首先，创建一个 PriorityThreadFactory：

```java
public class PriorityThreadFactory implements ThreadFactory {

    private final int mThreadPriority;

    public PriorityThreadFactory(int threadPriority) {
        mThreadPriority = threadPriority;
    }

    @Override
    public Thread newThread(final Runnable runnable) {
        Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                try {
                    Process.setThreadPriority(mThreadPriority);
                } catch (Throwable t) {

                }
                runnable.run();
            }
        };
        return new Thread(wrapperRunnable);
    }

}
```

创建一个 MainThreadExecutor：

```java
public class MainThreadExecutor implements Executor {

    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    public void execute(Runnable runnable) {
        handler.post(runnable);
    }
}
```

创建一个 DefaultExecutorSupplier：

```java
/**
 * Singleton class for default executor supplier
 */
public class DefaultExecutorSupplier{
    /**
     * Number of cores to decide the number of threads
     */
    public static final int NUMBER_OF_CORES = Runtime.getRuntime().availableProcessors();
    
    /**
     * thread pool executor for background tasks
     */
    private final ThreadPoolExecutor mForBackgroundTasks;
    /**
     * thread pool executor for light weight background tasks
     */
    private final ThreadPoolExecutor mForLightWeightBackgroundTasks;
    /**
     * thread pool executor for main thread tasks
     */
    private final Executor mMainThreadExecutor;
    /**
     * an instance of DefaultExecutorSupplier
     */
    private static DefaultExecutorSupplier sInstance;

    /**
     * returns the instance of DefaultExecutorSupplier
     */
    public static DefaultExecutorSupplier getInstance() {
       if (sInstance == null) {
         synchronized(DefaultExecutorSupplier.class){                                                                  
             sInstance = new DefaultExecutorSupplier();      
        }
        return sInstance;
    }

    /**
     * constructor for  DefaultExecutorSupplier
     */ 
    private DefaultExecutorSupplier() {
        
        // setting the thread factory
        ThreadFactory backgroundPriorityThreadFactory = new 
                PriorityThreadFactory(Process.THREAD_PRIORITY_BACKGROUND);
        
        // setting the thread pool executor for mForBackgroundTasks;
        mForBackgroundTasks = new ThreadPoolExecutor(
                NUMBER_OF_CORES  2,
                NUMBER_OF_CORES  2,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(),
                backgroundPriorityThreadFactory
        );
        
        // setting the thread pool executor for mForLightWeightBackgroundTasks;
        mForLightWeightBackgroundTasks = new ThreadPoolExecutor(
                NUMBER_OF_CORES  2,
                NUMBER_OF_CORES  2,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>(),
                backgroundPriorityThreadFactory
        );
        
        // setting the thread pool executor for mMainThreadExecutor;
        mMainThreadExecutor = new MainThreadExecutor();
    }

    /**
     * returns the thread pool executor for background task
     */
    public ThreadPoolExecutor forBackgroundTasks() {
        return mForBackgroundTasks;
    }

    /**
     * returns the thread pool executor for light weight background task
     */
    public ThreadPoolExecutor forLightWeightBackgroundTasks() {
        return mForLightWeightBackgroundTasks;
    }

    /**
     * returns the thread pool executor for main thread task
     */
    public Executor forMainThreadTasks() {
        return mMainThreadExecutor;
    }
}
```

注意：可用于不同线程池的线程数将取决于你的要求。

#### 现在在代码中使用下面的代码

```java
/**
 * Using it for Background Tasks
 */
public void doSomeBackgroundWork(){
  DefaultExecutorSupplier.getInstance().forBackgroundTasks()
    .execute(new Runnable() {
    @Override
    public void run() {
       // do some background work here.
    }
  });
}

/**
 * Using it for Light-Weight Background Tasks
 */
public void doSomeLightWeightBackgroundWork(){
  DefaultExecutorSupplier.getInstance().forLightWeightBackgroundTasks()
    .execute(new Runnable() {
    @Override
    public void run() {
       // do some light-weight background work here.
    }
  });
}

/**
 * Using it for MainThread Tasks
 */
public void doSomeMainThreadWork(){
  DefaultExecutorSupplier.getInstance().forMainThreadTasks()
    .execute(new Runnable() {
    @Override
    public void run() {
       // do some Main Thread work here.
    }
  });
}
```

这样，我们可以为网络任务，I / O 任务，繁重的后台任务和其他任务创建一个不同的线程池。

#### 如何取消一项任务？

要取消任务，你必须得到这个任务的 future 。 所以，而不是使用 execute，你将需要使用 submit，这将返回一个 future 。 现在这个 future 可以用来取消任务。

```java
/**
 * Get the future of the task by submitting it to the pool
 */
Future future = DefaultExecutorSupplier.getInstance().forBackgroundTasks()
    .submit(new Runnable() {
    @Override
    public void run() {
      // do some background work here.
    }
});

/**
 * cancelling the task
 */
future.cancel(true); 
```

#### 如何设定任务的优先级？

假设一个队列中有20个任务，线程池只有4个线程。 我们根据它们的优先级执行新的任务，因为线程池一次只能执行4个任务。 

但是假设我们需要在队列中首先执行的最后一个任务。 我们需要为该任务设置即时优先级，以便当线程从队列中获取新任务时，它首先执行此任务(因为它具有最高优先级) 

要设置任务的优先级，我们需要创建一个线程池执行器。 

创建优先级的 ENUM：

```java
/**
 * Priority levels
 */
public enum Priority {
    /**
     * NOTE: DO NOT CHANGE ORDERING OF THOSE CONSTANTS UNDER ANY CIRCUMSTANCES.
     * Doing so will make ordering incorrect.
     */

    /**
     * Lowest priority level. Used for prefetches of data.
     */
    LOW,

    /**
     * Medium priority level. Used for warming of data that might soon get visible.
     */
    MEDIUM,

    /**
     * Highest priority level. Used for data that are currently visible on screen.
     */
    HIGH,

    /**
     * Highest priority level. Used for data that are required instantly(mainly for emergency).
     */
    IMMEDIATE;

}
```

创建一个 PriorityRunnable：

```java
public class PriorityRunnable implements Runnable {

    private final Priority priority;

    public PriorityRunnable(Priority priority) {
        this.priority = priority;
    }

    @Override
    public void run() {
      // nothing to do here.
    }

    public Priority getPriority() {
        return priority;
    }

}
```

创建一个继承 ThreadPoolExecutor 的 PriorityThreadPoolExecutor。我们必须创建 PriorityFutureTask，它将实现 Comparable \<PriorityFutureTask> 

```java
public class PriorityThreadPoolExecutor extends ThreadPoolExecutor {

   public PriorityThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
         TimeUnit unit, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit,new PriorityBlockingQueue<Runnable>(), threadFactory);
    }

    @Override
    public Future<?> submit(Runnable task) {
        PriorityFutureTask futureTask = new PriorityFutureTask((PriorityRunnable) task);
        execute(futureTask);
        return futureTask;
    }

    private static final class PriorityFutureTask extends FutureTask<PriorityRunnable>
            implements Comparable<PriorityFutureTask> {
        private final PriorityRunnable priorityRunnable;

        public PriorityFutureTask(PriorityRunnable priorityRunnable) {
            super(priorityRunnable, null);
            this.priorityRunnable = priorityRunnable;
        }
        
        /**
         * compareTo() method is defined in interface java.lang.Comparable and it is used
         * to implement natural sorting on java classes. natural sorting means the the sort 
         * order which naturally applies on object e.g. lexical order for String, numeric 
         * order for Integer or Sorting employee by there ID etc. most of the java core 
         * classes including String and Integer implements CompareTo() method and provide
         * natural sorting.
         */
        @Override
        public int compareTo(PriorityFutureTask other) {
            Priority p1 = priorityRunnable.getPriority();
            Priority p2 = other.priorityRunnable.getPriority();
            return p2.ordinal() - p1.ordinal();
        }
    }
}
```

首先，在 DefaultExecutorSupplier 中，使用类似这样的 ThreadPoolExecutor 代替 ThreadPoolExecutor : 

```java
public class DefaultExecutorSupplier{

private final PriorityThreadPoolExecutor mForBackgroundTasks;

private DefaultExecutorSupplier() {
  
        mForBackgroundTasks = new PriorityThreadPoolExecutor(
                NUMBER_OF_CORES  2,
                NUMBER_OF_CORES  2,
                60L,
                TimeUnit.SECONDS,
                backgroundPriorityThreadFactory
        );

    }
}
```

以下是我们如何为任务设置高优先级的示例：

```java
/**
 *do some task at high priority
 */
public void doSomeTaskAtHighPriority(){
  DefaultExecutorSupplier.getInstance().forBackgroundTasks()
    .submit(new PriorityRunnable(Priority.HIGH) {
    @Override
    public void run() {
      // do some background work here at high priority.
    }
});
}
```

通过这种方式，任务可以被优先考虑。

以上实现也适用于任何 JAVA 应用程序。

我在 [Android 网络库](https://github.com/amitshekhariitbhu/Android-Networking)中使用了这个线程池实现。 有关更多详细的实现，你可以在这里检出 [Android Networking](https://github.com/amitshekhariitbhu/Android-Networking) 的 DefaultExecutorSupplier.java 类。 我希望这对你有用。

