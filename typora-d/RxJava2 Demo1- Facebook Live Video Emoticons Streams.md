# RxJava2 示例- Facebook Live 视频中表情符号流

> 原文 (Medium)：[RxJava2 Demo 1- Facebook Live Video Emoticons Streams.](https://medium.com/mindorks/rxjava2-demo-1-facebook-live-video-emoticons-streams-10f5211bc62)
>
> 作者：[Anshul Jain](https://medium.com/@anshuljain?source=post_header_lockup)

[TOC]

RxJava 现在是流行词。许多开发人员正在尝试使用 RxJava。我和 RxJava 的第一次约会是在一个月前开始的，当时我的一位同事向我转发了[一本书](http://shop.oreilly.com/product/0636920042228.do)。我开始阅读这本书，但最初发现 RxJava 的概念很难掌握。所以我决定在一个项目中使用 RxJava。由于 RxJava 是关于流的，所以我决定使用 RxJava 制作一个类似于 Facebook Live 视频中表情符号流的项目。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1froto0blizg30a00hse82.gif)

## 关于该项目

这个项目是为了模仿 facebook 的视频功能，在屏幕上流动的表情符号。 但是，我保留了一个条件 - 每个表情符号应该在上一个表情符号后至少300毫秒流动。我在代码中使用 RxJava 2。与 Facebook 类似，我在该项目中使用了6个表情符号，但为了解释，我将展示 like 和 love 按钮如何工作并生成一系列事件。

让直接进入代码！

## 通过点击事件创建流：

在 RxJava 中，一切都可以转换为流。因此，对各种表情符号的点击可以被认为是一个单一的流，它将在不同的时间间隔发出不同类型的项目。 

```java
  //Create an instance of FlowableOnSubscribe which will convert click events to streams
FlowableOnSubscribe flowableOnSubscribe = new FlowableOnSubscribe() {
      @Override
      public void subscribe(final FlowableEmitter emitter) throws Exception {
         //On click of like button, emit an Enum LIKE
         likeEmoticonButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
               emitter.onNext(Emoticons.LIKE);
            }
        });
        
          //On click of love button, emit an Enum LOVE
          loveEmoticonButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
              emitter.onNext(Emoticons.LOVE);
            }
      });
    }
```

## 创建一个 Flowable

由于我们需要检查两个流项目之间的时间差，我们需要从每个项目获取时间戳信息。

首先，我们从 [FlowableOnSubscribe](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/FlowableOnSubscribe.html) 创建一个 [Flowable](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Flowable.html)。我使用 Flowable 来代替 Observable，因为我需要一个背压策略，当用户无法接收事件的速度与源发出的速度一样快时使用。

```java
 //Give the backpressure strategey as BUFFER, so that the click items do not drop.
 Flowable emoticonsFlowable = Flowable.create(flowableOnSubscribe, BackpressureStrategy.BUFFER);
```

在上面的代码中，我将背压策略保持为 BUFFER。因此，当消息来源产生事件的速度比订阅者消耗的速度快时，流项目将被缓冲。接下来，我们将这个 Flowable 转换为 Flowable \<Timed>。因此每个项目都会有时间戳信息。

```java
//Convert the stream to a timed stream, as we require the timestamp of each event
Flowable<Timed> emoticonsTimedFlowable = emoticonsFlowable.timestamp();
```

## 创建 Subscriber

现在，由于我们有一个 Flowable，我们需要订阅者订阅 Flowable 生成的事件。

一个订阅者有4个方法，分别是 onSubscribe ( )，onNext ( )，onError ( ) 和 onCompleted ( )。

```java
 subscriber = new Subscriber() {
      @Override
      public void onSubscribe(Subscription s) {
        emoticonSubscription = s;
        //As soon as we are subscribing, we are requesting first elemetn
        emoticonSubscription.request(1);
      }

      //This is executed every time we get a new element
      @Override
      public void onNext(Object o) {
      }

      @Override
      public void onError(Throwable t) {
      }

      @Override
      public void onComplete() {
      }
    };
    //Subscribe
 emoticonsTimedFlowable.subscribeWith(subscriber);
```

在 onNext ( ) 方法中，我们需要获得最后发射的项目的时间戳，并且如果当前时间戳和最后发射的时间戳之间的差值大于300 ms，那么只有我们从流中请求更多项目。否则，在请求下一个项目之前，我们会异步等待300秒完成。

```java
//This is the current timestamp
long currentTimeStamp = System.currentTimeMillis();
//Difference between current timestsamp and timestamp of the last emitted item
long diffInMillis = currentTimeStamp - ((Timed) o).time();
if (diffInMillis > MINIMUM_DURATION_BETWEEN_EMOTICONS) {
    emoticonSubscription.request(1);
 } else {
 Handler handler = new Handler();
 handler.postDelayed(new Runnable() {
 @Override
  public void run() {
       emoticonSubscription.request(1);
        }
  }, MINIMUM_DURATION_BETWEEN_EMOTICONS - diffInMillis);
}
}
```

## 从 Flowable 取消订阅

一旦用户不再对流感兴趣，取消订阅就非常重要。

```java
 @Override
  public void onStop() {
    super.onStop();
    if (emoticonSubscription != null) {
      emoticonSubscription.cancel();
    }
  }
```

由于这是我在 RxJava 上的第一个项目，因此可能会有几件事情可以更有效地完成。如果你觉得代码可以改进，请随时放弃评论。

该项目在 [Github](https://github.com/Ansh1234/RxFbLiveVideoEmoticons) 上。如果你发现它有用，请为项目 star 。这将使我有更多项目工作的动力。感谢你阅读文章。

