# 使用架构组件具有生命周期感知地加载数据

> 原文 (Medium)：[Lifecycle Aware Data Loading with Architecture Components](https://medium.com/google-developers/lifecycle-aware-data-loading-with-android-architecture-components-f95484159de4)
>
> 作者：[Ian Lake](https://medium.com/@ianhlake?source=post_header_lockup)

[TOC]

在我[之前的博客文章](https://medium.com/google-developers/making-loading-data-on-android-lifecycle-aware-897e12760832)中，我介绍了如何使用 [Loaders](https://developer.android.com/guide/components/loaders.html) 以自动处理配置更改的方式加载数据。

随着[架构组件](https://developer.android.com/topic/libraries/architecture/index.html)的引入，有一个替代方案提供了一个现代的，灵活的，可测试的解决方案。

## 关注点分离

Loaders 的两个最大的好处是：

- 它们封装了数据加载的过程。
- 他们在配置更改后存活，防止不必要的重载数据。

对于架构组件，这两个好处现在由两个不同的类来处理: 

- [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 提供用于封装加载数据的生命周期感知基类 。
- [ViewModels](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 在配置更改时自动保留 。

这种分离的一个重要优点是可以在多个 ViewModel 中重复使用相同的 LiveData，通过 [MediatorLiveData](https://developer.android.com/reference/android/arch/lifecycle/MediatorLiveData.html) 将多个LiveData 源组合在一起，或者在 Service 中使用它们，避免尝试将 Loader 放入没有 LoaderManager 的场景中 。

与此同时，Loaders 支持在用户界面和数据加载之间进行分离(这是通向可测试应用程序的第一步!) ，这个模型扩展了这个优势ーー你的 ViewModel 可以通过模拟你的数据源来完全测试，LiveData 可以在完全隔离的情况下测试。 一个干净、可测试的架构是[应用程序体系结构指南](https://developer.android.com/topic/libraries/architecture/guide.html)中的一个重点。 

## 保持简单

这在理论上听起来都不错。 一个再现 [AsyncTaskLoader](https://medium.com/google-developers/making-loading-data-on-android-lifecycle-aware-897e12760832#5aa5) 的示例可能有助于使这些想法更具体: 

```java
public class JsonViewModel extends AndroidViewModel {
  // You probably have something more complicated
  // than just a String. Roll with me
  private final MutableLiveData<List<String>> data =
      new MutableLiveData<List<String>>();
  public JsonViewModel(Application application) {
    super(application);
    loadData();
  }
  public LiveData<List<String>> getData() {
    return data;
  }
  private void loadData() {
    new AsyncTask<Void,Void,List<String>>() {
      @Override
      protected List<String> doInBackground(Void… voids) {
        File jsonFile = new File(getApplication().getFilesDir(),
            "downloaded.json");
        List<String> data = new ArrayList<>();
        // Parse the JSON using the library of your choice
        return data;
      }
      @Override
      protected void onPostExecute(List<String> data) {
        this.data.setValue(data);
      }
    }.execute();
  }
}
```

等等，一个异步任务？ 这样安全吗？ 这里有两个安全特性: 

- [AndroidViewModel](https://developer.android.com/reference/android/arch/lifecycle/AndroidViewModel.html)（ViewModel 的子类）只有一个对应用程序上下文的引用，所以我们非常重要的是不引用活动的上下文，等等，这可能会导致泄漏 - 甚至有一个 Lint 检查来避免这些问题。
- LiveData 只有在有什么东西在观察的时候，才会发出结果 。

但是我们还没有完全抓住架构组件的本质: 我们的 ViewModel 正在直接构建和管理我们的 LiveData。 

```java
public class JsonViewModel extends AndroidViewModel {
  private final JsonLiveData data;
  public JsonViewModel(Application application) {
    super(application);
    data = new JsonLiveData(application);
  }
  public LiveData<List<String>> getData() {
    return data;
  }
}
public class JsonLiveData extends LiveData<List<String>> {
  private final Context context;
  public JsonLiveData(Context context) {
    this.context = context;
    loadData();
  }
  private void loadData() {
    new AsyncTask<Void,Void,List<String>>() {
      @Override
      protected List<String> doInBackground(Void… voids) {
        File jsonFile = new File(getApplication().getFilesDir(),
            "downloaded.json");
        List<String> data = new ArrayList<>();
        // Parse the JSON using the library of your choice
        return data;
      }
      @Override
      protected void onPostExecute(List<String> data) {
        setValue(data);
      }
    }.execute();
  }
}
```

所以我们的 ViewModel 变得相当简单，正如你所期望的那样。 我们的 LiveData 现在完全封装了加载过程，只加载了一次数据。 

## 在 LiveData 世界中的数据变化

就像 [Loaders 如何对其他地方的变化做出反应](https://medium.com/google-developers/making-loading-data-on-android-lifecycle-aware-897e12760832#1e8c)一样，使用 LiveData 时，这个功能也是关键——顾名思义，数据预计会发生变化！ 我们可以轻松地重新设置我们的类，以便在有观察者的情况下继续加载数据: 

```java
public class JsonLiveData extends LiveData<List<String>> {
  private final Context context;
  private final FileObserver fileObserver;
public JsonLiveData(Context context) {
    this.context = context;
    String path = new File(context.getFilesDir(),
        "downloaded.json").getPath();
    fileObserver = new FileObserver(path) {
      @Override
      public void onEvent(int event, String path) {
        // The file has changed, so let’s reload the data
        loadData();
      }
    };
    loadData();
  }
  @Override
  protected void onActive() {
    fileObserver.startWatching();
  }
  @Override
  protected void onInactive() {
    fileObserver.stopWatching();
  }
  private void loadData() {
    new AsyncTask<Void,Void,List<String>>() {
      @Override
      protected List<String> doInBackground(Void… voids) {
        File jsonFile = new File(getApplication().getFilesDir(),
            "downloaded.json");
        List<String> data = new ArrayList<>();
        // Parse the JSON using the library of your choice
        return data;
      }
      @Override
      protected void onPostExecute(List<String> data) {
        setValue(data);
      }
    }.execute();
  }
}
```

现在我们有兴趣监听更改，我们可以利用 LiveData 的 onActive ( ) 和 onInactive ( ) 回调函数来只监听数据的活跃观察者 - 只要有人在观察，他们可以保证得到最新的数据。

## 观察数据

在 Loader 的世界中，将你的数据输入到用户界面将涉及一个 LoaderManager，在正确的位置调用 initLoader ( ) ，并构建一个 LoaderCallbacks。 在架构组件的世界里，变得更加简单。 

我们需要做两件事: 

- 获取对我们的 ViewModel 的引用
- 开始观察我们的 LiveData

但是解释这个问题的时间几乎和代码本身一样长: 

```java
public class MyActivity extends AppCompatActivity {
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    JsonViewModel model =
        ViewModelProviders.of(this).get(JsonViewModel.class);
    model.getData().observe(this, data -> {
      // update UI
    });
  }
}
```

你会注意到事后并不需要清理：ViewModel 只在需要的时候自动生存，LiveData 只在调用 activity / fragment / lifecycleowner 时自动将数据传递给你。 

## 加载所有的东西

现在，如果你仍然在激烈反对 AsyncTask阵营，我完全同意: LiveData 比仅仅绑定在那个结构上灵活得多。 

例如，[Room](https://developer.android.com/topic/libraries/architecture/room.html) 可以让你获得[可观察的查询](https://developer.android.com/topic/libraries/architecture/room.html#daos-query-observable)ーー数据库查询返回 LiveData，这样数据库的更改就会通过 ViewModel 自动传播到用户界面。 有点像触摸游标或装载器的 CursorLoader。 

我们也可以用 LiveData 类重写 [FusedLocationApi](https://medium.com/google-developers/making-loading-data-on-android-lifecycle-aware-897e12760832#85b1) 示例：

```java
public class LocationLiveData extends LiveData<Location> implements
    GoogleApiClient.ConnectionCallbacks,
    GoogleApiClient.OnConnectionFailedListener,
    LocationListener {
  private GoogleApiClient googleApiClient;
  public LocationLiveData(Context context) {
    googleApiClient =
      new GoogleApiClient.Builder(context, this, this)
      .addApi(LocationServices.API)
      .build();
  }
  @Override
  protected void onActive() {
    // Wait for the GoogleApiClient to be connected
    googleApiClient.connect();
  }
  @Override
  protected void onInactive() {
    if (googleApiClient.isConnected()) {
      LocationServices.FusedLocationApi.removeLocationUpdates(
          googleApiClient, this);
    }
    googleApiClient.disconnect();
  }
  @Override
  public void onConnected(Bundle connectionHint) {
    // Try to immediately find a location
    Location lastLocation = LocationServices.FusedLocationApi
        .getLastLocation(googleApiClient);
    if (lastLocation != null) {
      setValue(lastLocation);
    }
    // Request updates if there’s someone observing
    if (hasActiveObservers()) {
      LocationServices.FusedLocationApi.requestLocationUpdates(
          googleApiClient, new LocationRequest(), this);
    }
  }
  @Override
  public void onLocationChanged(Location location) {
    // Deliver the location changes
    setValue(location);
  }
  @Override
  public void onConnectionSuspended(int cause) {
    // Cry softly, hope it comes back on its own
  }
  @Override
  public void onConnectionFailed(
      @NonNull ConnectionResult connectionResult) {
    // Consider exposing this state as described here:
    // https://d.android.com/topic/libraries/architecture/guide.html#addendum
  }
}
```

## 只是抓住架构组件的表面

安卓架构组件还有很多内容，所以一定要检查[所有的文档](https://developer.android.com/topic/libraries/architecture/index.html)。 

我个人强烈建议阅读整个[应用程序体系结构指南](https://developer.android.com/topic/libraries/architecture/guide.html)，让你了解所有这些组件是如何组合在一起，为你的整个应用构建一个坚实的架构。 

