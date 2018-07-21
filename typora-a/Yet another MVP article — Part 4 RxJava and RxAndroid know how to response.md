# 另一篇 MVP 文章 - 第4部分：RxJava 和 RxAndroid 知道如何响应

> 原文 (Medium)：[Yet another MVP article — Part 4: RxJava and RxAndroid know how to response](https://hackernoon.com/yet-another-mvp-article-part-4-rxjava-and-rxandroid-knows-how-to-response-cde42ccc4958)
>
> 作者： [Mohsen Mirhoseini](https://hackernoon.com/@mirhoseini)

[TOC]

## RxJava 是一个革命，真理或虚构

在我作为一个软件开发者的整个职业生涯中，我可以说 RxJava 是在 Java 8中的 Lambda 表达式之后最好的变化之一。 通过将这两者混合在一起的配方，你可以像魔术师一样开发软件。 

根据 RxJava 官方网站 [reactivex.io](http://reactivex.io/) 的说法，它是"一个异步编程的 API "，这在 Android 开发中是必不可少的。 

幸运的是，这个库的文件在它的网站上是完美的，你可以找到任何你想知道的图表。 

## 所以让我们回到我们的示例项目：

在这个项目中，RxJava 主要用于从 Model 层检索数据，Presentation 层调用 View 层的方法来处理响应。

作为调用 API 方法的结果，正在通过 MVP 的 Presentation 层观察到正在提供的 Retrofit 库。

关于 RxJava 的神奇之处在于它有可能被观察和用不同的线程订阅。 因为我们总是把结果交给 Android UI 线程(即 MainThread) ，所以我们需要从应用程序模块的核心模块内部链接到这个线程。 

## 如何指定从 Android 到核心的调度程序！

RxJava 中的[调度程序](http://reactivex.io/documentation/scheduler.html)是 java 中的线程。要将 API 的响应传递给视图层，必须将其推送到 Android UI MainThread 中。

要指定它，我们使用 RxJava 的 observeOn 方法。

RxJava 有一个名为 RxAndroid 的 Android 扩展，它提供了与 RxJava 一起使用的 android 线程。 为了使核心和应用程序模块之间的连接成为可能，应用程序模块在 [AppSchedulerProvider](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/util/AppSchedulerProvider.java) 类中实现了 [SchedulerProvider](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/util/SchedulerProvider.java) 接口，该接口同时实现了两个方法（mainThread / backgroundThread）：

```java
package com.mirhoseini.marvel.util;

import javax.inject.Inject;

import rx.Scheduler;
import rx.android.schedulers.AndroidSchedulers;
import rx.schedulers.Schedulers;

public class AppSchedulerProvider implements SchedulerProvider {
    @Inject
    public AppSchedulerProvider() {
    }

    @Override
    public Scheduler mainThread() {
        return AndroidSchedulers.mainThread();
    }

    @Override
    public Scheduler backgroundThread() {
        return Schedulers.io();
    }
}
```

```java
package com.mirhoseini.marvel.util;

import rx.Scheduler;

public interface SchedulerProvider {
    
    Scheduler mainThread();
    
    Scheduler backgroundThread();
    
}
```

最后通过一些 Dagger 注入，我们可以在我们的演示者类中使用 AppSchedulerProvider 作为它的父类（SchedulerProvider），这是 MVP 中 SOLID 原则中 [Liskov 替换](https://en.wikipedia.org/wiki/Liskov_substitution_principle) 的另一个例子。

## RxJava MAP 操作来帮助...

**RxJava** 有许多有用的操作符，但我认为 MAP 是最常用的操作符。 

看看 [SearchPresenterImpl](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/character/search/SearchPresenterImpl.java) 里面的这部分代码来做字符搜索：

```java
@Override
public void doSearch(boolean isConnected, String query, long timestamp) {
    if (null != view) {
        view.showProgress();
    }

    subscription = interactor.loadCharacter(query, Constants.PRIVATE_KEY, Constants.PUBLIC_KEY, timestamp)
            // check if result code is OK
            .map(charactersResponse -> {
                if (Constants.CODE_OK == charactersResponse.getCode())
                    return charactersResponse;
                else
                    throw Exceptions.propagate(new ApiResponseCodeException(charactersResponse.getCode(), charactersResponse.getStatus()));
            })
            // check if is there any result
            .map(charactersResponse -> {
                if (charactersResponse.getData().getCount() > 0)
                    return charactersResponse;
                else
                    throw Exceptions.propagate(new NoSuchCharacterException());
            })
            // map CharacterResponse to CharacterModel
            .map(Mapper::mapCharacterResponseToCharacter)
            // cache data on database
            .map(character -> {
                try {
                    databaseHelper.addCharacter(character);
                } catch (SQLException e) {
                    throw Exceptions.propagate(e);
                }

                return character;
            })
            .observeOn(scheduler.mainThread())
            .subscribe(character -> {
                        if (null != view) {
                            view.hideProgress();
                            view.showCharacter(character);

                            if (!isConnected)
                                view.showOfflineMessage();
                        }
                    },
                    // handle exceptions
                    throwable -> {
                        if (null != view) {
                            view.hideProgress();
                        }

                        if (isConnected) {
                            if (null != view) {
                                if (throwable instanceof ApiResponseCodeException)
                                    view.showServiceError((ApiResponseCodeException) throwable);
                                else if (throwable instanceof NoSuchCharacterException)
                                    view.showQueryNoResult();
                                else
                                    view.showRetryMessage(throwable);
                            }
                        } else {
                            if (null != view) {
                                view.showOfflineMessage();
                            }
                        }
                    });
}
```

API 的响应并不总是我们想要的视图层，甚至可能需要一些数据库缓存，因此，使用 [MAP](http://reactivex.io/documentation/operators/map.html) 运算符我们可以在 UI 知道之前做一些技巧。

[SearchPresenterImpl](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/character/search/SearchPresenterImpl.java) 中有四个 MAP 运算符应用于 API 响应：

- 第一个 map，检查来自API的响应代码是否正确，如果不是，将会抛出异常。确保你使用  Exceptions.propagate 方法，否则你必须使用 try / catch 语句在那里处理异常！
- 第二个 map，检查是否有任何搜索字符的结果，如果不是一个异常将抛出以后 
- 第三个 map，使用 Mapper 类将响应模型真正映射到数据库模型中 
- 最后，将响应插入到数据库中，以便进一步缓存 

使用 RxJava，所有对 API 响应的操作都可以在一行代码中实现！

并记住所有映射的事情发生在后台线程内部，因为我们在 [SearchInteractorImpl](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/character/search/SearchInteractorImpl.java) 类中使用 subscribeOn 来指定它，这就是为什么它永远不会中断 MainThread 导致任何延迟甚至 [ANRs](https://developer.android.com/training/articles/perf-anr.html)。 

**还有什么地方？**
 RxJava 也在[适配器](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/character/cache/adapter/CharactersRecyclerViewAdapter.java)中用于通知列表中的项目点击，我认为这很容易理解，你可以看看。

