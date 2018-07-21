# 另一篇 MVP 文章 - 第3部分：使用 Retrofit 调用 API

> 原文 (Medium)：[Yet another MVP article — Part 3: Calling APIs using Retrofit](https://hackernoon.com/yet-another-mvp-article-part-3-calling-apis-using-retrofit-23757f4eee05)
>
> 作者： [Mohsen Mirhoseini](https://hackernoon.com/@mirhoseini)

[TOC]

## 让我们进入网络和 API：

几乎所有的移动应用程序都使用互联网与其服务进行通信。 Retrofit 是由 [Jake Wharton](https://medium.com/@jakewharton) 贡献的完美库，它可以随时随地帮助你开发 Android 应用程序的网络层。
长话短说，在这个 Retrofit 项目中，使用了一个名为 [MarvelApi](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/domain/client/MarvelApi.java) 的接口，它显示了 API 的实现方式：

```java
package com.mirhoseini.marvel.domain.client;

import com.mirhoseini.marvel.domain.model.CharactersResponse;

import retrofit2.http.GET;
import retrofit2.http.Query;
import rx.Observable;

public interface MarvelApi {

    String NAME = "name";
    String API_KEY = "apikey";
    String HASH = "hash";
    String TIMESTAMP = "ts";

    // http://gateway.marvel.com:80/v1/public/characters?name=Iron%20Man&apikey=PUBLIC_API_KEY&hash=HASH&ts=TIMESTAMP
    @GET("v1/public/characters")
    Observable<CharactersResponse> getCharacters(
            @Query(NAME) String query,
            @Query(API_KEY) String publicKey,
            @Query(HASH) String hash,
            @Query(TIMESTAMP) long timestamp);
}
```

使用  Retrofit 非常简单，你可以看到有一个 REST 风格的 [API 方法](https://developer.marvel.com/docs)位于 http://gateway.marvel.com:80/v1/public/characters 需要一些参数来检索你想要的，这是字符数据使用其名称。

## MarvelApi 位于 MVP 图层哪个？

所有的网络和数据库的东西发生在模型层。

这个层只适用于数据和业务逻辑，演示者知道如何调用或如何为视图层准备数据。 

请停止在模型层中操作数据，并将其安全、健全地传递给演示层，它知道该怎么做。 

## 更多的 SOLID 原则...

根据 SOLID 原则的"[单一责任](https://en.wikipedia.org/wiki/Single_responsibility_principle)"原则,"[一个厨师不能管理整个餐厅](https://www.novoda.com/blog/designing-something-solid/)"，然后模型和演示完全分离了要做的任务。 

## 我们可以有一个很好的日志，我们在 API 响应中所调用的和接收到的信息？ 为什么不呢

Retrofit 库使用 OkHttp 库，其中是一个功能强大的网络库。 拦截器发生了所有的魔法。 我会一一解释，最后告诉你如何将它们应用到你的 OkHttpClient。

通过一个小配置，你可以要求这个库，展示一个美观的 logcat 日志，它总是很方便的：

```java
@Singleton
@Provides
public HttpLoggingInterceptor provideHttpLoggingInterceptor() {
    HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
    logging.setLevel(HttpLoggingInterceptor.Level.BODY);
    return logging;

}
```

但不要忘记在发布版本中禁用这个日志记录的东西！你可以通过检查在应用程序构建期间生成的 BuildConfig.DEBUG 变量来执行此操作。

## 缓存 API 结果导致数据库消失! !

首次在新闻阅读器应用程序中执行此操作时，所有数据库记录更新和查询几乎都杀死了所有团队成员。

使用 OkHttp 缓存功能，你可以处理 header  缓存，并且还具有脱机文件缓存，有时可能是数据库痛苦的完美替代。

```java
@Provides
@Singleton
public Cache provideCache(@Named("cacheDir") File cacheDir, @Named("cacheSize") long cacheSize) {
    Cache cache = null;

    try {
        cache = new Cache(new File(cacheDir.getPath(), HTTP_CACHE_PATH), cacheSize);
    } catch (Exception e) {
        e.printStackTrace();
    }

    return cache;
}

@Singleton
@Provides
@Named("cacheInterceptor")
public Interceptor provideCacheInterceptor(@Named("cacheMaxAge") int maxAgeMin) {
    return chain -> {
        Response response = chain.proceed(chain.request());

        CacheControl cacheControl = new CacheControl.Builder()
                .maxAge(maxAgeMin, TimeUnit.MINUTES)
                .build();

        return response.newBuilder()
                .header(CACHE_CONTROL, cacheControl.toString())
                .build();
    };
}

@Singleton
@Provides
@Named("offlineInterceptor")
public Interceptor provideOfflineCacheInterceptor(StateManager stateManager, @Named("cacheMaxStale") int maxStaleDay) {
    return chain -> {
        Request request = chain.request();

        if (!stateManager.isConnect()) {
            CacheControl cacheControl = new CacheControl.Builder()
                    .maxStale(maxStaleDay, TimeUnit.DAYS)
                    .build();

            request = request.newBuilder()
                    .cacheControl(cacheControl)
                    .build();
        }

        return chain.proceed(request);
    };
}
```

这个示例中有两个缓存拦截器:

- 一个缓存拦截器改变 API 响应的头部和可用的缓存2分钟(maxAgeMin) ，这意味着从 API 中删除用户调用2分钟 

- 另一个是离线拦截器，它使用缓存目录(提供应用程序模块中的 Dagger)将 API 响应保存到文件中7天(maxStaleDay) 


  不要惊慌，如果一切都是新的，你将学习如何使用它，通过一些示例项目...）

## 你是否相信 API 调用失败后的网络状态？ 不，那么请再试一次

网络连接总是处于亏损状态，甚至服务可能会宕机几秒钟，而且可能会给应用程序带来麻烦，所以请在通知用户有关情况之前再试一次！ 

我寻找使用 RxJava 的重试策略，但是没有一个是用 Retrofit 工作的，但是这个拦截器工作: 

```java
@Singleton
@Provides
@Named("retryInterceptor")
public Interceptor provideRetryInterceptor(@Named("retryCount") int retryCount) {
    return chain -> {
        Request request = chain.request();
        Response response = null;
        IOException exception = null;

        int tryCount = 0;
        while (tryCount < retryCount && (null == response || !response.isSuccessful())) {
            // retry the request
            try {
                response = chain.proceed(request);
            } catch (IOException e) {
                exception = e;
            } finally {
                tryCount++;
            }
        }

        // throw last exception
        if (null == response && null != exception)
            throw exception;

        // otherwise just pass the original response on
        return response;
    };
}
```

如果你需要更多的重试时间或者更多的条件来决定重试或者不重试，你可以自定义它，但是我认为这个是好的。 :) 

## 以及如何将所有拦截器连接到我的 OkHttp 客户端？

所有的拦截器都在 [ClientModule](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/domain/ClientModule.java) 内部提供，并将它们全部添加到 OkHttpClient 中：

```java
package com.mirhoseini.stylight.domain;

/*...*/

@Module
public class ClientModule {
    /*...*/

    @Singleton
    @Provides
    public OkHttpClient provideOkHttpClient(HttpLoggingInterceptor loggingInterceptor,
                                            @Named("networkTimeoutInSeconds") int networkTimeoutInSeconds,
                                            @Named("isDebug") boolean isDebug,
                                            Cache cache,
                                            @Named("cacheInterceptor") Interceptor cacheInterceptor,
                                            @Named("offlineInterceptor") Interceptor offlineCacheInterceptor,
                                            @Named("retryInterceptor") Interceptor retryInterceptor) {

        OkHttpClient.Builder okHttpClient = new OkHttpClient.Builder()
                .addNetworkInterceptor(cacheInterceptor)
                .addInterceptor(offlineCacheInterceptor)
                .addInterceptor(retryInterceptor)
                .cache(cache)
                .connectTimeout(networkTimeoutInSeconds, TimeUnit.SECONDS);

        //show logs if app is in Debug mode
        if (isDebug)
            okHttpClient.addInterceptor(loggingInterceptor);

        return okHttpClient.build();
    }
    
    /*...*/
    
}
```

注意由于 BuildConfig.DEBUG 变量而产生的 if（isDebug）语句。 还有 Network Timeout 告诉 OkHttp 等待响应的时间。

## 然后 retrofit 在哪里！

好吧，所有这些编码是为了最后一步...在 [ApiModule](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/domain/ApiModule.java) 里面，我们使用 OkHttpClient 来创建 Retrofit 实例并使用它来创建 MarvelApi。

```java
package com.mirhoseini.marvel.domain;

/*...*/

@Module
public class ApiModule {

    @Provides
    @Singleton
    public MarvelApi provideMarvelApi(Retrofit retrofit) {
        return retrofit.create(MarvelApi.class);
    }

    @Provides
    @Singleton
    public Retrofit provideRetrofit(HttpUrl baseUrl, Converter.Factory converterFactory, CallAdapter.Factory callAdapterFactory, OkHttpClient okHttpClient) {
        return new Retrofit.Builder()
                .baseUrl(baseUrl)
                .addConverterFactory(converterFactory)
                .addCallAdapterFactory(callAdapterFactory)
                .client(okHttpClient)
                .build();
    }

    @Provides
    @Singleton
    public Converter.Factory provideGsonConverterFactory(Gson gson) {
        return GsonConverterFactory.create(gson);
    }

    @Singleton
    @Provides
    public Gson provideGson() {
        return new Gson();
    }

    @Provides
    @Singleton
    public CallAdapter.Factory provideRxJavaCallAdapterFactory() {
        return RxJavaCallAdapterFactory.create();
    }

}
```

你必须提供 BaseUrl，它通常是你所有服务 API URL 的第一个静态部分，在这个示例中是 “http://gateway.marvel.com:80/”。 （不要忘记最后一个 ' / '，它是 Retrofit  所必需的，可能会导致异常）

以这种方式提供的 API 将由与服务通信的 SearchInteractorImpl 使用。 Retrofit 知道如何使用 RxJava 进行响应，你可以轻松捕获使用 Gson 解析的结果，并将其与其他 RxJava [运算符](http://reactivex.io/documentation/operators.html)（即 map， flat map，filter 等）一起使用来操作和准备响应，并将其交给视图呈现给用户。

