# 谷歌示例 "架构组件" 的指南

> 原文：[The Missing Google Sample of Android “Architecture Components” Guide.](https://proandroiddev.com/the-missing-google-sample-of-android-architecture-components-guide-c7d6e7306b8f)
>
> 作者：[Philippe BOISNEY](https://proandroiddev.com/@Phil_Boisney?source=post_header_lockup)

[TOC]

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowu1jbc9j30p00di400.jpg)

谷歌最近发布了 [Android 架构组件](https://developer.android.com/topic/libraries/architecture/index.html)，这是一组库，可帮助您设计健壮的，可测试的和可维护的应用程序

这将真正改变 android 开发人员设计应用程序的方式。

然而，在仔细阅读了关于 Android Architecture 组件的 [Google 指南](https://developer.android.com/topic/libraries/architecture/guide.html)之后，我真的很沮丧，他们没有向我们提供完整解释的用例样本......我甚至在 Github 上检查了专门用于 Android 架构组件的 GoogleSamples 存储库，但也没有...

所以我决定实施它，并创建这个着名指南的相关应用。您可以在此存储库中找到最终的应用程序：[[PR](https://github.com/PhilippeBoisney/GithubArchitectureComponents)]

我选择尽可能多地遵循指南。我只是添加了那些著名的（但推荐）库：

- Dagger 2
- Butterknife & Glide
- Gson

您可以在 [build.gradle](https://github.com/PhilippeBoisney/GithubArchitectureComponents/blob/master/app/build.gradle) 中找到此项目中使用的库的完整列表。

## 这款应用有什么作用？

这个简单的应用程序是由一个屏幕组成的。当此屏幕出现时，我们将获取（Retrofit）[Jake Wharton](https://medium.com/@jakewharton) 的 Github 信息，并将这些信息立即保存在应用程序存储（Room）中。

接下来，当屏幕重新启动（或由于旋转而重新创建）时，我们将首先在 Room 数据库中获得相同的信息，并且只在必要时刷新 Github Api 中的信息。 

## 1. 配置 Room

在安装这些库之后，让我们创建持久的模型。创建表示 Github 用户的 User 实体，该实体将被持久化（通过 Room）并从 Github Api（通过 Retrofit）获取。

User.java

```java
@Entity
public class User {

    @PrimaryKey
    @NonNull
    @SerializedName("id")
    @Expose
    private String id;

    @SerializedName("login")
    @Expose
    private String login;

    @SerializedName("avatar_url")
    @Expose
    private String avatar_url;

    @SerializedName("name")
    @Expose
    private String name;

    @SerializedName("company")
    @Expose
    private String company;

    @SerializedName("blog")
    @Expose
    private String blog;

    private Date lastRefresh;

    // --- CONSTRUCTORS ---

    public User() { }

    public User(@NonNull String id, String login, String avatar_url, String name, String company, String blog, Date lastRefresh) {
        this.id = id;
        this.login = login;
        this.avatar_url = avatar_url;
        this.name = name;
        this.company = company;
        this.blog = blog;
        this.lastRefresh = lastRefresh;
    }

    // --- GETTER ---

    public String getId() { return id; }
    public String getAvatar_url() { return avatar_url; }
    public Date getLastRefresh() { return lastRefresh; }
    public String getLogin() { return login; }
    public String getName() { return name; }
    public String getCompany() { return company; }
    public String getBlog() { return blog; }

    // --- SETTER ---

    public void setId(String id) { this.id = id; }
    public void setAvatar_url(String avatar_url) { this.avatar_url = avatar_url; }
    public void setLastRefresh(Date lastRefresh) { this.lastRefresh = lastRefresh; }
    public void setLogin(String login) { this.login = login; }
    public void setName(String name) { this.name = name; }
    public void setCompany(String company) { this.company = company; }
    public void setBlog(String blog) { this.blog = blog; }
}
```

接下来，我们将创建 DAO 来将用户持久化到 Room 中。 

UserDao.java

```java
@Dao
public interface UserDao {

    @Insert(onConflict = REPLACE)
    void save(User user);

    @Query("SELECT  FROM user WHERE login = :userLogin")
    LiveData<User> load(String userLogin);

    @Query("SELECT  FROM user WHERE login = :userLogin AND lastRefresh > :lastRefreshMax LIMIT 1")
    User hasUser(String userLogin, Date lastRefreshMax);
}
```

> [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html) 是一个可观察的数据持有者。它可以让你应用中的组件观察到 LiveData 对象的更改，而不会在它们之间创建明确的和严格的依赖关系路径。也尊循你的应用组件的生命周期状态(活动、片段、服务) ，并且做了正确的事情来防止对象泄露，这样你的应用程序就不会消耗更多的内存 

因为 Room 不能持久化 Date 对象，所以我们必须创建一个 TypeConverter：

DateConverter.java

```java
public class DateConverter {
    @TypeConverter
    public static Date toDate(Long timestamp) {
        return timestamp == null ? null : new Date(timestamp);
    }

    @TypeConverter
    public static Long toTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```

最后，我们创建数据库对象：

```java
@Database(entities = {User.class}, version = 1)
@TypeConverters(DateConverter.class)
public abstract class MyDatabase extends RoomDatabase {

    // --- SINGLETON ---
    private static volatile MyDatabase INSTANCE;

    // --- DAO ---
    public abstract UserDao userDao();
}
```

## 2. 配置 Retrofit

正如我之前所说，user 对象将被 Retrofit 和 Room 使用。所以现在，我们只需要创建一个 Retrofit 接口：

```java
public interface UserWebservice {
    @GET("/users/{user}")
    Call<User> getUser(@Path("user") String userId);
}
```

## 3. 配置存储库

在我看来，这个组件是最重要的。在里面，我们将根据预定义的场景选择使用哪一个数据源来获取用户的数据：

- 第一次用户启动应用程序时使用 Web 服务（通过 Retrofit ）
- 用户数据的 API 最后一次读取时间大于3分钟前，使用 Webservice 而不是数据库。
- 否则，使用数据库（通过 Room ）。

> 存储库模块负责处理数据操作。他们为应用程序的其他部分提供了一个干净的 API。他们知道从何处获取数据以及在更新数据时调用哪些 API。您可以将它们视为不同数据源（持久模型，Web 服务，缓存等）之间的中介。

UserRepository.java

```java
@Singleton
public class UserRepository {

    private static int FRESH_TIMEOUT_IN_MINUTES = 3;

    private final UserWebservice webservice;
    private final UserDao userDao;
    private final Executor executor;

    @Inject
    public UserRepository(UserWebservice webservice, UserDao userDao, Executor executor) {
        this.webservice = webservice;
        this.userDao = userDao;
        this.executor = executor;
    }

    // ---

    public LiveData<User> getUser(String userLogin) {
        refreshUser(userLogin); // try to refresh data if possible from Github Api
        return userDao.load(userLogin); // return a LiveData directly from the database.
    }

    // ---

    private void refreshUser(final String userLogin) {
        executor.execute(() -> {
            // Check if user was fetched recently
            boolean userExists = (userDao.hasUser(userLogin, getMaxRefreshTime(new Date())) != null);
            // If user have to be updated
            if (!userExists) {
                webservice.getUser(userLogin).enqueue(new Callback<User>() {
                    @Override
                    public void onResponse(Call<User> call, Response<User> response) {
                        Toast.makeText(App.context, "Data refreshed from network !", Toast.LENGTH_LONG).show();
                        executor.execute(() -> {
                            User user = response.body();
                            user.setLastRefresh(new Date());
                            userDao.save(user);
                        });
                    }

                    @Override
                    public void onFailure(Call<User> call, Throwable t) { }
                });
            }
        });
    }

    // ---

    private Date getMaxRefreshTime(Date currentDate){
        Calendar cal = Calendar.getInstance();
        cal.setTime(currentDate);
        cal.add(Calendar.MINUTE, -FRESH_TIMEOUT_IN_MINUTES);
        return cal.getTime();
    }
}
```

## 4. 配置 ViewModel

在创建 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 之前，我们必须为 ViewModel 创建一个工厂，以处理更 “简洁” 的依赖注入。

FactoryViewModel.java

```java
@Singleton
public class FactoryViewModel implements ViewModelProvider.Factory {

    private final Map<Class<? extends ViewModel>, Provider<ViewModel>> creators;

    @Inject
    public FactoryViewModel(Map<Class<? extends ViewModel>, Provider<ViewModel>> creators) {
        this.creators = creators;
    }

    @SuppressWarnings("unchecked")
    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        Provider<? extends ViewModel> creator = creators.get(modelClass);
        if (creator == null) {
            for (Map.Entry<Class<? extends ViewModel>, Provider<ViewModel>> entry : creators.entrySet()) {
                if (modelClass.isAssignableFrom(entry.getKey())) {
                    creator = entry.getValue();
                    break;
                }
            }
        }
        if (creator == null) {
            throw new IllegalArgumentException("unknown model class " + modelClass);
        }
        try {
            return (T) creator.get();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

所以现在，我们来创建将在片段中使用的 UserViewModel。

> ViewModel 提供特定 UI 组件的数据，例如片段或活动，并处理与业务部分的数据处理部分，例如调用其他组件加载数据或转发用户修改。 Viewmodel 不知道视图，也不会受到配置变化的影响，比如重新创建旋转活动 

UserProfileViewModel.java

```java
public class UserProfileViewModel extends ViewModel {

    private LiveData<User> user;
    private UserRepository userRepo;

    @Inject
    public UserProfileViewModel(UserRepository userRepo) {
        this.userRepo = userRepo;
    }

    // ----

    public void init(String userId) {
        if (this.user != null) {
            return;
        }
        user = userRepo.getUser(userId);
    }

    public LiveData<User> getUser() {
        return this.user;
    }
}
```

## 5. 配置依赖注入

由于 Dagger 2有时会让人难以理解，所以我将许多单独的模块组合成一个 AppModule。在此之前，让我们创建 Dagger 注入所有这些依赖关系所需的所有类：

ViewModelKey.java

```java
@Documented
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@MapKey
public @interface ViewModelKey {
    Class<? extends ViewModel> value();
}
```

ViewModelModule.java

```java
@Module
public abstract class ViewModelModule {

    @Binds
    @IntoMap
    @ViewModelKey(UserProfileViewModel.class)
    abstract ViewModel bindUserProfileViewModel(UserProfileViewModel repoViewModel);

    @Binds
    abstract ViewModelProvider.Factory bindViewModelFactory(FactoryViewModel factory);
}
```

FragmentModule.java

```java
@Module
public abstract class FragmentModule {
    @ContributesAndroidInjector
    abstract UserProfileFragment contributeUserProfileFragment();
}
```

ActivityModule.java

```java
@Module
public abstract class ActivityModule {
    @ContributesAndroidInjector(modules = FragmentModule.class)
    abstract MainActivity contributeMainActivity();
}
```

AppModule.java

```java
@Module(includes = ViewModelModule.class)
public class AppModule {

    // --- DATABASE INJECTION ---

    @Provides
    @Singleton
    MyDatabase provideDatabase(Application application) {
        return Room.databaseBuilder(application,
                MyDatabase.class, "MyDatabase.db")
                .build();
    }

    @Provides
    @Singleton
    UserDao provideUserDao(MyDatabase database) { return database.userDao(); }

    // --- REPOSITORY INJECTION ---

    @Provides
    Executor provideExecutor() {
        return Executors.newSingleThreadExecutor();
    }

    @Provides
    @Singleton
    UserRepository provideUserRepository(UserWebservice webservice, UserDao userDao, Executor executor) {
        return new UserRepository(webservice, userDao, executor);
    }

    // --- NETWORK INJECTION ---

    private static String BASE_URL = "https://api.github.com/";

    @Provides
    Gson provideGson() { return new GsonBuilder().create(); }

    @Provides
    Retrofit provideRetrofit(Gson gson) {
        Retrofit retrofit = new Retrofit.Builder()
                .addConverterFactory(GsonConverterFactory.create(gson))
                .baseUrl(BASE_URL)
                .build();
        return retrofit;
    }

    @Provides
    @Singleton
    UserWebservice provideApiWebservice(Retrofit restAdapter) {
        return restAdapter.create(UserWebservice.class);
    }
}
```

AppComponent.java

```java
@Singleton
@Component(modules={ActivityModule.class, FragmentModule.class, AppModule.class})
public interface AppComponent {

    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder application(Application application);
        AppComponent build();
    }

    void inject(App app);
}
```

现在我们必须让 Dagger 进入我们项目的 Application 类:  

App.java

```java
public class App extends Application implements HasActivityInjector {

    @Inject
    DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;

    public static Context context;

    @Override
    public void onCreate() {
        super.onCreate();
        this.initDagger();
        context = getApplicationContext();
    }

    @Override
    public DispatchingAndroidInjector<Activity> activityInjector() {
        return dispatchingAndroidInjector;
    }

    // ---

    private void initDagger(){
        DaggerAppComponent.builder().application(this).build().inject(this);
    }
}
```

## 6. 配置片段

快结束了！ 我们必须将 ViewModel 添加到我们的片段 UserProfileFragment (您可以在这里找到它的[布局](https://github.com/PhilippeBoisney/GithubArchitectureComponents/blob/master/app/src/main/res/layout/fragment_user_profile.xml)) ，订阅 LiveData 流，并在收到数据时更新用户界面。 

UserProfileFragment.java

```java
public class UserProfileFragment extends Fragment {

    // FOR DATA
    public static final String UID_KEY = "uid";
    @Inject
    ViewModelProvider.Factory viewModelFactory;
    private UserProfileViewModel viewModel;

    // FOR DESIGN
    @BindView(R.id.fragment_user_profile_image) ImageView imageView;
    @BindView(R.id.fragment_user_profile_username) TextView username;
    @BindView(R.id.fragment_user_profile_company) TextView company;
    @BindView(R.id.fragment_user_profile_website) TextView website;

    public UserProfileFragment() { }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_user_profile, container, false);
        ButterKnife.bind(this, view);
        return view;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        this.configureDagger();
        this.configureViewModel();
    }

    // -----------------
    // CONFIGURATION
    // -----------------

    private void configureDagger(){
        AndroidSupportInjection.inject(this);
    }

    private void configureViewModel(){
        String userLogin = getArguments().getString(UID_KEY);
        viewModel = ViewModelProviders.of(this, viewModelFactory).get(UserProfileViewModel.class);
        viewModel.init(userLogin);
        viewModel.getUser().observe(this, user -> updateUI(user));
    }

    // -----------------
    // UPDATE UI
    // -----------------

    private void updateUI(@Nullable User user){
        if (user != null){
            Glide.with(this).load(user.getAvatar_url()).apply(RequestOptions.circleCropTransform()).into(imageView);
            this.username.setText(user.getName());
            this.company.setText(user.getCompany());
            this.website.setText(user.getBlog());
        }
    }
}
```

## 7. 配置 MainActivity

最后，我们创建包含 UserProfileFragment 的活动（您可以在这里找到它的[布局](https://github.com/PhilippeBoisney/GithubArchitectureComponents/blob/master/app/src/main/res/layout/activity_main.xml)）。

MainActivity.java

```java
public class MainActivity extends AppCompatActivity implements HasSupportFragmentInjector {

    @Inject
    DispatchingAndroidInjector<Fragment> dispatchingAndroidInjector;

    private static String USER_LOGIN = "JakeWharton";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        this.configureDagger();
        this.showFragment(savedInstanceState);
    }

    @Override
    public DispatchingAndroidInjector<Fragment> supportFragmentInjector() {
        return dispatchingAndroidInjector;
    }

    // ---

    private void showFragment(Bundle savedInstanceState){
        if (savedInstanceState == null) {

            UserProfileFragment fragment = new UserProfileFragment();

            Bundle bundle = new Bundle();
            bundle.putString(UserProfileFragment.UID_KEY, USER_LOGIN);
            fragment.setArguments(bundle);

            getSupportFragmentManager().beginTransaction()
                    .add(R.id.fragment_container, fragment, null)
                    .commit();
        }
    }

    private void configureDagger(){
        AndroidInjection.inject(this);
    }
}
```

完成！ 😅

我知道，有很多类（特别是因为 Dagger），但我们完全遵循了[关注点的分离](https://en.wikipedia.org/wiki/Separation_of_concerns)，这是 Android 架构组件的顶层！

欢迎评论和分享！

