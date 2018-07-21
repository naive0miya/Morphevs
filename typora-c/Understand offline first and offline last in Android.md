# Android 的首先离线和最后离线策略

> 原文 (Medium)：[Understand offline first and offline last in Android](https://medium.com/@coreflodev/understand-offline-first-and-offline-last-in-android-71191e92b426)
>
> 作者：[Florent Guillemot](https://medium.com/@coreflodev?source=post_header_lockup)

[TOC]

本文是[如何实现一个简单的离线工作的 Android 应用程序](https://medium.com/p/make-an-easy-offline-working-android-application-fdbf1d679710)的后续。 在本文中，我将向您解释主要的离线策略：首先离线，最后离线。 如前一篇文章中所述，将使用 HTTP 缓存完成实现。

## 首先离线策略

即使服务器可用，您也可以从缓存中获取信息。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovbszgdbj30kw05a74j.jpg)

正如您在架构中所看到的，应用程序几乎每次都会从缓存中请求信息。这项规则的两个值得注意的例外是: 

- 缓存是空的。 这意味着您的应用程序永远不会连接到 Internet。
- 缓存已过时。确定缓存是否过期的条件由您决定。例如，可能是您的应用收到推送通知的时间。

下面是这个策略的可能实现 。

HttpService-offline-first.java

```java
package io.coreflodev.openchat.common.network;

import android.content.Context;

import java.io.File;
import java.util.concurrent.TimeUnit;

import okhttp3.Cache;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

public class HttpService {

    public static final String HEADER_CACHE = "android-cache";
    private static final String CACHE_DIR = "httpCache";
    private OkHttpClient httpClient;

    public HttpService(Context context) {
        File httpCacheDirectory = new File(context.getCacheDir(), CACHE_DIR);
        Cache cache = new Cache(httpCacheDirectory, 10  1024  1024);

        httpClient = new OkHttpClient.Builder()
                .cache(cache)
                .connectTimeout(5, TimeUnit.SECONDS)
                .addInterceptor(chain -> {
                    Request request = chain.request();
                    if (request.header(HEADER_CACHE) != null) {
                        Request offlineRequest = request.newBuilder()
                                .header("Cache-Control", "only-if-cached, "
                                        + "max-stale=" + request.header(HEADER_CACHE))
                                .build();
                        Response response = chain.proceed(offlineRequest);
                        if (response.isSuccessful()) {
                            return response;
                        }
                    }
                    return chain.proceed(request);
                })
                .build();
    }

    public OkHttpClient getHttpClient() {
        return httpClient;
    }
}
```

Service.java

```java
package io.coreflodev.openchat.api;

import java.util.List;

import io.reactivex.Observable;
import retrofit2.http.GET;
import retrofit2.http.Headers;

import static io.coreflodev.openchat.common.network.HttpService.HEADER_CACHE;

public interface ChatService {

    @GET("/messages")
    @Headers(HEADER_CACHE + ": 60")
    Observable<List<ChatMessage>> getMessages();
}
```

这个实现由两个步骤组成：

- 在 HttpService 第15行中，在 ChatService 行14中使用了一个声明的常量 HEADER_CACHE。该常量表示您希望将请求保留在缓存中的秒数。 我已经这样做了，因为我认为每个请求都会有不同的价值。 事实上，对于包含可变值的响应，5分钟应该足够了，而包含电话号码的响应可以存储一天。
- 然后在 HttpService 第26行中有一个 HTTP 拦截器。 这个拦截器将读取先前声明的头文件，如果它有一个值，它将检查缓存的请求时间。 这意味着在我的情况下，我只会每60秒要求一次服务器。

这个实现的主要优点是，如果你旋转你的设备，并对服务做其他请求。这很好，因为你不会真的查询服务器，只是查询你的缓存。

## 最后离线的战略

只有当服务器不可用时，你才能从缓存中获取信息。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frovbx7030j30ko056mxd.jpg)

正如您在架构中所看到的，只有服务器不可用时，应用程序才会从缓存中请求信息。

下面是这个策略的可能实现 。

HttpService-offline-last.java

```java
package io.coreflodev.openchat.common.network;

import android.content.Context;

import java.io.File;
import java.util.concurrent.TimeUnit;

import okhttp3.Cache;
import okhttp3.OkHttpClient;
import okhttp3.Request;

public class HttpService {

    private static final String CACHE_DIR = "httpCache";

    private OkHttpClient httpClient;

    public HttpService(Context context) {
        File httpCacheDirectory = new File(context.getCacheDir(), CACHE_DIR);
        Cache cache = new Cache(httpCacheDirectory, 10  1024  1024);

        httpClient = new OkHttpClient.Builder()
                .cache(cache)
                .connectTimeout(5, TimeUnit.SECONDS)
                .addInterceptor(chain -> {
                    try {
                        return chain.proceed(chain.request());
                    } catch (Exception e) {
                        Request offlineRequest = chain.request().newBuilder()
                                .header("Cache-Control", "public, only-if-cached, "
                                        + "max-stale=" + 60  60  24)
                                .build();
                        return chain.proceed(offlineRequest);
                    }
                })
                .build();
    }

    public OkHttpClient getHttpClient() {
        return httpClient;
    }
}
```

为了实现这一点，重要的部分是拦截器，第25行。 默认情况下，请求将进入服务器，除非引发异常(通常为 ConnectException)。 在这种情况下，请求被重新路由到缓存中。 对于这个请求还有一个最大化的属性，因为我认为24小时内没有连接到服务器的缓存是无效的。 

这个实现的主要优势是，如果你的用户没有互联网或者你的服务器出了问题，你仍然可以让你的应用程序可用，如果它不需要为用户提供新的数据。 这比在没有互联网的情况下，你的应用崩溃或者根本不能使用要好。 

## 如何同时使用这两种策略？

这是可能的，实现遵循这个模式：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frovc00nuzj30m806zaa9.jpg)

它是这样工作的：

- 在 t0，向服务器发出一个请求正确的响应。
- 在 t0 和 t1 之间，HTTP 客户端只会查找缓存的响应
- 在 t1 时刻，如果服务器返回 t0 状态，则向服务器发出请求。
- 在 t1 和 t2 之间，每当 HTTP 客户端无法访问服务器时，它将从上次缓存的响应中读取。如果它访问服务器，那么我们又回到了案例。
- 在 t2 之后，你应该让你的用户连接到互联网，因为你在缓存中的数据是完全过时的。

这里有一个可能的实现：

```java
package io.coreflodev.openchat.common.network;

import android.content.Context;

import java.io.File;
import java.util.concurrent.TimeUnit;

import okhttp3.Cache;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

public class HttpService {

    public static final String HEADER_CACHE = "android-cache";
    private static final String CACHE_DIR = "httpCache";
    private OkHttpClient httpClient;

    public HttpService(Context context) {
        File httpCacheDirectory = new File(context.getCacheDir(), CACHE_DIR);
        Cache cache = new Cache(httpCacheDirectory, 10  1024  1024);

        httpClient = new OkHttpClient.Builder()
                .cache(cache)
                .connectTimeout(5, TimeUnit.SECONDS)
                .addInterceptor(chain -> {
                    Request request = chain.request();
                    if (request.header(HEADER_CACHE) != null) {
                        Request offlineRequest = request.newBuilder()
                                .header("Cache-Control", "only-if-cached, " +
                                        "max-stale=" + request.header(HEADER_CACHE))
                                .build();
                        Response response = chain.proceed(offlineRequest);
                        if (response.isSuccessful()) {
                            return response;
                        }
                    }
                    try {
                        return chain.proceed(chain.request());
                    } catch (Exception e) {
                        Request offlineRequest = request.newBuilder()
                                .header("Cache-Control", "public, only-if-cached, " +
                                        "max-stale=" + 60  60  24)
                                .build();
                        return chain.proceed(offlineRequest);
                    }
                })
                .build();
    }

    public OkHttpClient getHttpClient() {
        return httpClient;
    }
}
```

正如您所看到的，它采取了与以前相同的实现: 

- 在第27行和第37行之间，这是首先离线的实现。
- 在第37行和第47行之间，这是最后离线的实现。

