# RxJava2 示例- 在 Android 中下载项目

> 原文 (Medium)：[RxJava2 Demo2- Downloading songs/images using Android Download Manager.](https://medium.com/mindorks/rxjava2-demo2-downloading-songs-in-android-2ebf91ac3a9a)
>
> 作者：[Anshul Jain](https://medium.com/@anshuljain?source=post_header_lockup)

[TOC]

这是系列文章的第二部分，旨在使用 RxJava2 编写关于 Android 演示项目。

## 关于该项目

该项目用于使用 RxJava2 在 Android 中下载项目（歌曲，图像等）。但是，我已经设置了2个条件才能下载。

- 每次只能下载2个项目。所以即使用户点击了多个项目进行下载，其中只有其中的两个实际上会一次下载，其余的下载将被排队。
- 下载百分比显示给用户。但是，只有当前百分比和以前显示的百分比之间的差异大于5％时才显示出来。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frotpqn710g30800e8h0y.gif)

现在让我们直接进入代码。 每个可下载的项目都有以下属性。

```java
public class DownloadableItem {
  
  // Id of the download
  private String id;
  //Download id which is given by the DownloadManager
  private long downloadId;
  //Title of the item
  private String itemTitle;
  //cover id of the item
  private int itemCoverId;
  //Downloading status i.e NOT_DOWNLOADED, IN_PROGRESS, WAITING or DOWNLOADED
  private DownloadingStatus downloadingStatus;
  //Url to download
  private String itemDownloadUrl;
  //Current download percent of the item
  private int itemDownloadPercent;
  //Previous download percent shown on the UI
  private long lastEmittedDownloadPercent = -1;
}
```

## 创建下载请求流

```java
FlowableOnSubscribe flowableOnSubscribe = new FlowableOnSubscribe() {
      @Override
      public void subscribe(FlowableEmitter e) throws Exception {
        downloadsFlowableEmitter = e;
      }
    };
// Create the flowable with Buffer as the backpressure strategey
final Flowable flowable = Flowable.create(flowableOnSubscribe, BackpressureStrategy.BUFFER);
```

在上面的代码片段中，使用 Flowable 创建了一个流，它将发出可下载的项目。在创建 FlowableOnSubscribe 的实例时，我们在 onSubscribe ( ) 方法中获得 FlowableEmitter 的一个对象，将用于发射物品。如果流出物品的数量大于用户消耗的物品数量，Flowable 会将物品保存在缓冲区中。

现在，当用户点击该项目的下载按钮时，该项目就从 Flowable 发出。

```java
// Handle the click on the download button of the song.
@Override
  public void onClick(View v) {
   downloadsFlowableEmitter.onNext(downloadableItem);
}
```

## 创建用于使用下载请求的订阅者

```java
final Subscriber subscriber = new Subscriber() {
      @Override
      public void onSubscribe(Subscription downloadRequestsSubscription) {
        //At the time of subscription, request 2 downloads.
        downloadRequestsSubscription.request(2);
      }

      @Override
      public void onNext(Object o) {
        currentDownloadsCount++;
        //Start the download here.
      }

      @Override
      public void onError(Throwable t) {}
      
      @Override
      public void onComplete() {}
    };
}

//Here we subscribe to the stream
flowable.subscribeWith(subscriber);
```

在上面的代码中，我们创建一个订阅者，然后订阅 Flowable。在创建 Subscriber 实例的同时，我们获得了 Subscription 类的一个对象，该类用于请求 Flowable 中的项目。

我们保留一个局部变量 currentDownloadsCount 来跟踪当前下载的数量。当我们在订阅者的 onNext ( ) 方法中获取一个项目时，这意味着订户已收到一个项目以供下载。所以我们将变量的值增加1.当下载完成时，我们将该值减1。

```java
@Override
  public void onDownloadComplete() {
    //Decrement the current number of downloads by 1
    currentDownloadsCount--;
    //In this project the value of MAX_COUNT_OF_SIMULTANEOUS_DOWNLOADS is set to 2.
    downloadRequestsSubscription.request(Constants.MAX_COUNT_OF_SIMULTANEOUS_DOWNLOADS - currentDownloadsCount);
}
```

下载完成后，使用 Subscription 对象请求 Flowable 流中的新项目。 当活动被破坏时，不要忘记处理用户。

```java
if (downloadRequestsSubscription != null) {
  downloadRequestsSubscription.cancel();
}
```

## 创建下载百分比流

```java
 ObservableOnSubscribe observableOnSubscribe = new ObservableOnSubscribe() {
      @Override
      public void subscribe(ObservableEmitter e) throws Exception {
        percentageObservableEmitter = e;
      }
    };

final Observable observable = Observable.create(observableOnSubscribe);
```

在上面的代码中，使用 Observable 而不是 Flowable 创建了一个流（因为我们不需要 Backpressure 策略），它将发出可下载的项目。在创建 ObservableOnSubscribe 的实例时，我们在 onSubscribe ( ) 方法中获得 ObservableEmitter 对象，该对象将用于发射项目。

当下载到 DownloadManager 时，它会更新状态（成功，失败，正在进行中等）以及在其数据库中为给定可下载项目下载的总字节数。要知道下载状态和百分比，我们每200毫秒轮询一次数据库以获取有关下载项目的信息。

```java
public void updateItem(DownloadableItem downloadableItem, String query) {
      Cursor cursor =  downloadManager.query(query);
      if (cursor == null || !cursor.moveToFirst()) {
        return downloadableResult;
      }
      //Get the download percent
      float bytesDownloaded =
          cursor.getInt(cursor.getColumnIndex(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR));
      float bytesTotal =
          cursor.getInt(cursor.getColumnIndex(DownloadManager.COLUMN_TOTAL_SIZE_BYTES));
      int downloadPercent = (int) ((bytesDownloaded / bytesTotal) * 100);
      if (downloadPercent <= Constants.DOWNLOAD_COMPLETE_PERCENT) {
        downloadableItem.setDownloadPercent(downloadPercent);
      }
 }
```

获取有关该项目的信息后，我们检查当前的下载百分比和最后发布的下载百分比。如果差异大于5％或者当前下载是100％，那么我们发出这个项目，否则我们跳过它。这样可以避免我们不必要地更新 UI，以免两次调查之间发生实质性的百分比变化。

```java
 public static void queryDownloadPercents(final DownloadableItem downloadableItem,
                                           final ObservableEmitter percentFlowableEmiitter) {

 
    long lastEmittedDownloadPercent = downloadableItem.getLastEmittedDownloadPercent();
    long currentDownloadPercent = downloadableItem.getItemDownloadPercent();

    if ((currentDownloadPercent - previousDownloadPercent >= 5) ||
        currentDownloadPercent == 100) {
      //Emit the item if the above condition is met.
      percentFlowableEmiitter.onNext(downloadableItem);
      //Set the percent as the last emitted download percent.
      downloadableItem.setLastEmittedDownloadPercent(currentDownloadPercent);
    }
    }
}
```

## 创建一个观察者并订阅它

最后，我们创建观察者并订阅它。在创建 Subscriber 的实例时，我们得到一个 Disposable 类的对象，一旦我们完成接收事件，该对象将用于处理与 Observable 的连接。 

```java
final Observer observer = new Observer() {
      @Override
      public void onSubscribe(Disposable d) {
        downloadPercentDisposable = d;
      }

      @Override
      public void onNext(Object value) {
        if (!(value instanceof DownloadableItem)) {
          return;
        }
       //Update the UI
       updateDownloadableItem(downloadableItem)
      }

      @Override
      public void onError(Throwable e) {}

      @Override
      public void onComplete() {}
    };
  }
  
observable.subscribeWith(observer);
```

现在，只要我们在观察者的 onNext ( ) 函数中接收到该项目，就会更新 UI。

```java
 public void updateDownloadableItem(DownloadableItem downloadableItem, int position) {
    if (downloadableItem == null || contextWeakReference.get() == null) {
      return;
    }
   
    ItemDetailsViewHolder itemDetailsViewHolder = (ItemDetailsViewHolder)
        recyclerView.findViewHolderForLayoutPosition(position);

    //It means that the viewholder is not currently displayed.
    if (itemDetailsViewHolder == null) {
      return;
    }
    itemDetailsViewHolder.updateImageDetails(downloadableItem);
}
```

updateImageDetails ( ) 函数负责根据其下载状态和下载百分比来更新每个项目（在 Viewholder 中）。

最后，一旦活动被破坏，我们就处理连接。

```java
if (downloadPercentDisposable != null) {
    downloadPercentDisposable.dispose();
}
```

该项目在 [Github](https://github.com/Ansh1234/RxDownloader) 上。如果你觉得它有用，请在 Github 上 star。这将有助于我开展更多项目。

感谢你阅读文章。 RxJava 上有很多文档和文章，但很少有实际的例子，这使得它很难理解它的概念。我希望通过这篇文章，我能够让读者对这个概念有点了解。

