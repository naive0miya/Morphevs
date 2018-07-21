# 测试 Android MVP

> 原文 (Medium)：[Testing Android MVP](https://medium.com/@Miqubel/testing-android-mvp-aa0de6e165e4)
>
> 作者：[Miquel Beltran](https://medium.com/@Miqubel?source=post_header_lockup)

[TOC]

> 100％ 覆盖 Mockito 和 Robolectric

有成千上万的文章解释了 Android 的不同编程模式的好坏和丑陋，遗憾的是，大多数在线发现的文章都忘了解释最后一个重要的步骤: 测试。 

选择一种或另一种模式既是个人偏好的问题，也是项目要求的问题。 我不认为 MVP 优于 MVVM 或完全自定义解决方案，但它的简单性是我在创建新屏幕时的模式。 

## MVP 四个要点的

- MVP 代表 Model-View-Presenter。
- "模型"是你的数据源，无论是数据库、网络调用还是硬编码的对象列表并不重要 。
- "视图"是你的活动、片段或自定义的 Android 视图。 它应该只是关注显示和处理用户交互 。
- “演示者” 是一个简单的旧 Java 对象（POJO），它处理视图和模型之间的通信，进行所有必需的数据转换，处理模型中的错误，并尝试从视图中尽可能多地删除 UX 逻辑

## 测试 MVP 的原则

- 我们的主要原则是我们将使用 JUnit，而不是 Espresso 或任何其他自动化测试框架 
- 第二个原则是，我们将分别测试每个部分，而不是集成测试。 因此，需要对依赖注入框架有所了解 
- 因为我们将使用 JUnit，我们将需要模拟所有的 UI 组件。 为此，我们将使用 Robolectric 
-  为了测试交互和模拟类，对 Mockito 的熟悉也是必要的 

## 测试模型

如果我们正在做 MVP，那么测试模型是我们应该独立完成的事情。 在某些情况下，我们的模型将是一个我们不能测试的第三方库，所以我们应该能够提供一个易于模拟的接口。 

- 该模型不得包含对演示者或视图的任何引用。
- 该模型应作为一个简单的可模拟的接口提供给演示者，特别是当模型是第三方库的一部分时。
- 模型应该独立于所使用的设计模式进行测试。

**范例**

让我们来看一个完整的文章的例子。模型是类，提供用户配置文件作为一个 Observable（我们不知道它是来自本地数据库还是来自互联网），它是用接口定义的。

```java
public interface ProfileInteractor {
    Observable<UserProfile> getProfile();
}
```

我会通过使用 RxJava 的 TestSubscriber 来测试这一点，确保没有错误地获得预期值 。 

```java
public class ProfileInteractorTest {
    private static final String USER = "USERNAME";
    ProfileInteractor interactor;
    
    @Before
    public void setUp() {
        interactor = new ProfileInteractorImpl(...);
    }

    @Test
    public void testGetUserProfile() throws Exception {
        TestSubscriber<UserProfile> subscriber = TestSubscriber.create();
        interactor.getProfile().subscribe(subscriber);
        subscriber.assertNoErrors();
        subscriber.assertCompleted();
        assertThat(subscriber.getOnNextEvents().get(0).getName()).isEqualTo(USER);
    }
}
```

## 测试视图

测试视图比我们想象的要容易，最复杂的部分是 Robolectric 设置，但是在做了一次之后，对所有视图做同样的操作将会很容易。 

像任何其他 MVP 实现一样，在这里我们有视图接口：

```java
public interface ProfileView {
    void display(UserProfile userProfile);
}
```

我们的示例视图将是一个扩展 FrameLayout 的自定义 Android 视图 。

```java
public class ProfileFrameLayout extends FrameLayout implements ProfileView {

    private ProfilePresenter presenter;

    @BindView(R.id.text_username)
    TextView textUsername;

    @Inject
    public void setPresenter(ProfilePresenter presenter) {
        this.presenter = presenter;
        presenter.attachView(this);
    }

    public CollectionFrameLayout(Context context) {
        super(context);
        init();
    }

    public CollectionFrameLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        View view = inflate(getContext(), R.layout.view_profile, this);
        ButterKnife.bind(view);
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        presenter.attachView(this);
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        presenter.detachView();
    }

    @Override
    public void display(UserProfile userProfile) {
        textUsername.setText(userProfile.getName());
    }
```

我们应该在这里测试什么？一切！

- 让我们测试一下视图是否创建正确 
- 让我们测试一下默认值是否正确 
- 让我们把用户交互的内容发送给演示者 
- 让我们测试一下这个视图是否正在执行它的主要任务(显示 UserProfile) 

这里是测试类：

```java
@RunWith(RobolectricGradleTestRunner.class)
@Config(constants = BuildConfig.class)
public class ProfileFrameLayoutTest {
    private ProfileFrameLayout profileView;
    @Mock
    ProfilePresenter presenter;

    @Before
    public void setUp() throws Exception {
        MockitoAnnotations.initMocks(this);
        profileView = new ProfileFrameLayout(RuntimeEnvironment.application);
        profileView.setPresenter(presenter);
    }
    
    @Test
    public void testEmpty() throws Exception {
        verify(presenter).attachView(profileView);
        asserThat(profileView.textUsername.getText().toString()).isEmpty();
    }
    
    @Test
    public void testLeaveView() throws Exception {
        profileView.onDetachedFromWindow();
        verify(presenter).detachView();
    }
    
    @Test
    public void testReturnToView() throws Exception {
        reset(presenter);
        profileView.onAttachedToWindow();
        verify(presenter).attachView(profileView);
    }
    
    @Test
    public void testDisplay() throws Exception {
        UserProfile user = new UserProfile(USER);
        profileView.display(user);
        asserThat(profileView.textUsername.getText().toString()).isEqualTo(USER);
    }
}
```

逐步:

- 顶部注释是 Robolectric 配置的一部分。 我们告诉  JUnit  的是使用标准 BuildConfig (我们没有改变)的 Robolectric 测试运行器 
- 我们持有一个我们自定义视图的实例，它将测试 
- 我们使用 Mockito 创建一个模拟的演示者。我们只想要验证 View 是否在调用 Presenter。
- 我们使用 Robolectric 的 RuntimeEnviroment 创建一个 ProfileFrameLayout 的新实例，从应用程序获取上下文
- 我们设置了我们的模拟 Presenter，因为我们要确保它被调用。
- TestEmpty 检查两件事情：attachView 方法被调用，并且 textUsername 是空的。
- 当 View 获取 Android 事件时，TestLeaveView 检查 detachView 方法是否被调用。 如果是活动，你可以在这里销毁或者暂停，这取决于你的情况 
- TestReturnToView 对于其他事件也是如此。请注意，我重置 Presenter 模拟计数器，因为在开始时调用了 attachView 方法
- 最后，TestDisplay 测试我们通过 display ( ) 方法传递的值是否在 TextView 上正确设置。

我们已经覆盖了我们视图中的所有代码行，一旦你开始添加越来越多的逻辑和 UI 元素，事情就变得没有那么复杂了。 

综述：

- 提供一个模拟的演示者，这样你就可以通过 Mockito 来验证事件和交互 
- 提供访问内部视图的权限，以便你可以验证它们的值 
- 通过 Robolectric 创建一个不依赖于 Android 框架的视图实例 

## 测试演示者

我们将用与测试模型相同的方式测试 Presenter，并使用简单的 JUnit 测试。不过，这次我们将提供一个模拟视图来验证演示者是否正确地与视图进行通信。 

我们的演示者将收到一个视图，并从模型中获取数据，以在视图中显示该视图。 

```java
public void ProfilePresenter {
  private final ProfileInteractor interactor;
  private ProfileView view;
  
  public ProfilePresenter(ProfileInteractor interactor) {
    this.interactor = interactor.
  }
  
  public void attachView(ProfileView view) {
    this.view = view;
    fetchAndDisplay();
  }
  
  public void dettachView() {
    // Not covered by this example:
    // You should handle the subscription
  }
  
  public void fetchAndDisplay() {
    // Not covered by this example:
    // You should handle the subscription
    // You should also check if view is not null
    // You should also handle the onError
    interactor.getUserProfile().subscribe(userProfile -> view.display(userProfile));
  }
}
```

例如，ProfilePresenter 提供的例子没有处理大量的 RxJava 陷阱，因为这篇文章的简洁性已经被删除了。 

我们要测试的是 Model（ProfileInteractor）和 View 接收来自 Presenter 的正确调用。实际上，我们对验证数据不感兴趣，但如果你的 Presenter 正在进行数据转换，那么你也应该测试一下。

你还应该测试演示者如何处理模型中的错误。使用 RxJava，你可以通过从 getUserProfile ( ) 中返回一个 Observable.error (new Throwable ( ))来实现。

```java
public void ProfilePresenterTest {
  @Mock
  ProfileInteractor interactor;
  @Mock
  ProfileView view;
  
  @Before
  public void setUp() throws Exception {
    MockitoAnnotations.initMocks(this);
    when(interactor.getUserProfile()).thenReturn(Observable.just(new UserProfile()));
    presenter = new ProfilePresenter(interactor);
    presenter.attachView(view);
  }
  
  @Test
  public void testDisplayCalled() {
    verify(interactor).getUserProfile();
    verify(view).display(any());
  }
}
```

 逐步:

- 创建模型和视图的模拟实例。
- 模拟 getUserProfile 方法输出。我使用 when ( ) 来告诉模拟交互者从预期结果的空实例返回一个 Observable，使用 just ( ) 方法来创建它。 （你也可以返回 Observable.error ( ) 来测试错误处理）
- 将这些传递给演示者的实例。
- getUserProfile 方法在 View 被附加时调用，所以在我们的测试中我们只需要验证这个方法是否被调用。
- 和视图一样，我们需要验证的方法被调用。 我们实际上并不关心我们得到的内容，所以我使用 any ( ) 作为匹配器。

综上所述：

- 模拟你的模型和你的视图，以便你可以验证他们从演示者接到调用。
- 模型的假反应来测试不同的情况。 你甚至不需要使用 RxJava。

## 关键概念

- 如果没有模拟的模型，包装它。
- 使用依赖注入为你的视图提供一个模拟演示者。**不要在 View 中创建 Presenter**。
- 不仅要测试输出，还要测试交互。
- 如果演示者依赖于视图还需测试视图生命周期。
- 测试视图中的视觉变化，不仅仅是文本，还包括可见性或背景颜色。
- 测试演示者如何处理来自模型的不同响应。

我希望下次你正在编写一个应用程序，并使用 MVP 编程模式，你会考虑到这些概念，这样你就可以创建快速单元测试。 

