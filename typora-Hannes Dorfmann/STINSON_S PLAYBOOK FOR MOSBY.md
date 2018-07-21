# Mosby 指南

> 原文：[STINSON'S PLAYBOOK FOR MOSBY](http://hannesdorfmann.com/android/mosby-playbook)
>
> 作者：[Hannes Dorfmann](http://hannesdorfmann.com/about)



[TOC]

在我之前的博客文章中，我介绍了一个用于 android 的模型 - 视图 - 演示者库 Mosby。 这次我将会讨论一些关于 MVP 和 Mosby 的更多细节。 我已经实现了一个可以在 [Github](https://github.com/sockeqwe/mosby/tree/master/sample-mail)上找到的邮件客户端示例，这个示例在这个博客中用来描述如何使用 Mosby，并回答一些在发布 Mosby 之后被问到的常见问题。

更新： Mosby 有它自己的 [github 页面](http://hannesdorfmann.com/mosby/)，有关MVP的详细信息以及如何使用 Mosby 构建一个 Android 应用程序。 以下博客文章已经为 Mosby 1.0 编写，并且自 Mosby 2.0 发布以来已经过时。 你一定要继续阅读 github 页面！

虽然我已经写了示例应用程序，我也改善了 Mosby。 我很高兴地宣布我已经发布了 [Mosby 1.1.0](https://github.com/sockeqwe/mosby/releases/tag/1.1.0)。 就像你已经知道（或建议），Mosby 这个名字来自我最喜爱的电视节目之一：How I met your Mother。 在这个博客文章中，我会给你一些有关 MVP 和 Mosby 在 Barney Stinson 的 legen 的提示...等待它... dary playbook 风格。

如前所述，我写了一个模拟邮件客户端的示例应用程序。 这不是一个真正的邮件客户端，它没有连接到一个真正的 POP3 或 IMAP 服务器。 所有数据都是在应用程序启动时随机生成的。 没有像本地 sqlite 数据库一样的持久层。 APK 文件可以从[这里下载](https://github.com/sockeqwe/mosby/releases/download/1.1.0/sample-mail.apk)。 如果您很懒惰不想在您的手机上安装 APK 文件，可以看这个视频：[Mosby: mail client](https://www.youtube.com/watch?v=_dEYtXgoyBM)

当然，整个应用程序是基于 Mosby，正如你在视频中所看到的，在方向改变时一切都保持状态。关于数据结构：邮件链接到一个人作为发件人和另一个人作为接收者。每个邮件都与一个标签关联。标签就像是一个 “文件夹”。邮件可以分配给一个标签。一个标签有任意多个邮件分配。标签基本上只是一个字符串。在这个示例应用程序中有四个标签：“收件箱”，“发送”，“垃圾邮件” 和 “垃圾”。因此，删除邮件表单收件箱只是将邮件的标签从“收件箱”重新分配给“垃圾箱”。 MailProvider 是中央业务逻辑元素，我们可以查询邮件列表等。此外，负责验证用户的 AccountManager 存在。正如你在视频中看到的第一个应用程序开始，你必须登录才能看到邮件。在内部用于业务逻辑（MailProvider 和 AccountManager），我使用 RxJava。如果你不熟悉 RxJava，别担心。所有你必须知道的是，RxJava 提供了一个 Observable 来做一些查询，而 onData（）， onCompleted（） 或 onError（）方法被调用。如果您熟悉 RxJava，请不要看代码！这是非常丑陋的，因为我在开发过程中多次改变主意，并没有重构所有的东西，就像我为一个真正的应用程序做的那样。请注意，这个示例应用程序是关于 Mosby，而不是如何用 RxJava 编写一个干净的业务逻辑。另外请注意，即使我已经使用 Dagger2 进行依赖注入，这不是 Dagger2 的参考应用程序，也不是用于材料设计。肯定有改进的空间，我很乐意接受这个示例应用程序的拉请求。

正如你在视频中看到的，我正在模拟网络流量，为每个请求添加延迟两秒（如加载邮件列表）。 我也模拟网络问题：每五个请求都会失败（这就是为什么你经常在视频中看到错误视图）。 此外，我模拟身份验证问题：15个请求后，用户必须再次登录（登录按钮将被延迟）。

这篇博文中的第一个提示通常与软件架构和MVP相关，而最后一个则是关于 Mosby 的。

## Tip 1: Don’t “over-architect”

花点时间做出正确的决定。一个干净的软件架构是必要的。但是，在某些时候，你必须开始编码。规划是重要的，但是在真实世界的项目中，即使在分析应用程序的每个方面的许多小时之后，您也必须重构代码。实际上，重构是发展的重要组成部分，你根本无法避免它（但是你可以尽量减少重构的次数）。不要陷入[分析麻痹](https://sourcemaking.com/antipatterns/analysis-paralysis)反模式。我们的 Java 开发人员也倾向于遇到类似的“问题”：不要过度概括所有的东西。从我个人的经验来看，我认为与 iOS 开发人员相比，我们的 android 开发人员倾向于编写更清洁的代码，而不是通用代码，这就是为什么开发 android 应用程序通常需要更长时间才能开发 iOS 应用程序的原因之一。不要误解我的意思我真的很感激干净的代码和设计模式。我可以从我的经验中给你一个具体的例子：在我们的一个应用程序中，我们使用了一种 ImageProxy。这个想法很简单：给出一个图像的网址，我们知道我们的服务器提供了不同版本的这个图像。根据设备，ImageView 大小，互联网连接和速度，应该加载该图像的正确版本。所以基本上我们只是把 http://www.example.com/foo.jpg 的网址“重写”到 http://www.example.com/foo.jpg?quality=high&width=400。听起来很简单，不是吗？所以我开始实现这个 ImageProxy 库。我在内部使用毕加索加载和缓存图像。但是，我决定保留它非常抽象，使库的组件模块化和可变，即你可以用任何其他http图像加载库，如 Glide 取代毕加索。我还实施了[策略模式](http://en.wikipedia.org/wiki/Strategy_pattern)，用于动态选择要加载的正确版本的图像，并使其更具可扩展性。这个想法是，您可以在运行时添加更多的预配置或动态配置的策略实施。猜猜看，我们从来没有改变图像加载（毕加索）的内部核心，也没有增加更多的策略，因为我们在3年前添加了 ImageProxy 到我们的应用程序（应用程序仍然定期每月大量更新）。我花了2天时间来实现这个 ImageProxy 库，我写了10个类/接口。我的同事在 iOS 上执行同样的事情花了大约6个小时，写了3个主要包含 if-else 语句的类。你可能会问自己，这个提示与 Mosby 有关。把它作为一般的建议。从简单的事情开始，然后重构，随着应用程序和需求的增长，使其更具可扩展性。在使用 Mosby 的时候也是一样的。尽量保持演示者简单。不要想太久，“如果我们在一年内添加特征 XY，主持人怎么能延长？”。

## Tip 2: Use inheritance wisely

从一开始就说：我不是继承者最大的粉丝。为什么？继承是很好的添加属性和功能。不幸的是我看到很多开发者使用它错误比如你正在某个应用程序的团队中工作。您已经编写了一个 MVP 视图，该视图在 ListView 上实现了下拉刷新。这是您在应用程序的几乎每个屏幕中使用或扩展的基本 MVP 视图。这是非常方便的，所有你需要做的实现一个新的屏幕是从该基类的子类，踢在自定义适配器，你已经准备好了。有一天，你的一个队友必须实现一个 ListView，而不需要进行刷新。因此，许多开发人员做出错误的决定，不是写一个全新的类，而是从您的从下拉刷新的 ListView，尝试找到一个 hacky 的解决方法，以“删除”下拉刷新功能。这可能会导致不可维护的代码，您可能会看到空的实现覆盖的方法，最后但并非最不重要的是，从下拉刷新的 ListView 到新的类引入某种紧耦合，因为您无法更改基本的“刷新 ListView 实现，而不必担心打破非拉取式 ListView 类。一个更好的解决方案是重构最初的pull-to-refresh ListView 类，并拉出一个ListView类而不进行 pull-to-refresh，并使得拉到刷新的 ListView 类从非pull-to-refresh ListView 类扩展。然而，如果你决定这样做，你可能会得到一个很大的继承层次结构，因为每个子类都增加了一个小的特性，比如扩展 A（增加特性 X），扩展 B（增加特性 Y），扩展 C（增加功能 Z）等。您在示例邮件应用程序中看到的情况非常相似：

- AuthFragment : 带附加状态的 LCE（加载内容错误）片段显示“未经身份验证 - 登录”按钮。
- AuthRefreshFragment : 使用 SwipeRefreshLayout 作为主要内容视图的 AuthFragment 扩展。所以人们可以说：“这是一个”支持刷新支持的 AuthFragment“。
- AuthRefreshRecyclerFragment : 将一个 RecyclerView 放到 SwipeRefreshLayout 中。
- BaseMailsFragment: 通过使用 RecyclerView 来显示邮件列表（ List ）的抽象类
- 有一些具体的类从 BaseMailsFragment 扩展，如 MailsFragment， SearchFragment 和 ProfileMailsFragment
- 继承层次结构可能是正确的，但是你真的认为最近加入你的团队的人会选择正确的基类来实现一个新的特征 XY？ 在Java中它是关于接口，所以它应该在 Android 上。 我个人比较喜欢接口和委托对比继承。

## Tip 3: Don’t see MVP as an MVC variant

有些人发现，如果你试图解释 MVP 是 MVC（模型 - 视图 - 控制器）的变种或增强，那么很难理解什么是演示者。尤其是 iOS 开发人员，他们很难理解 Controller 和 Presenter 的区别，因为他们是在 iOS 的 UIViewController 的固定想法和定义下“长大”的。从我的角度来看，MVP 并不是 MVC 的变种或增强，因为这意味着控制器被Presenter所取代。在我看来 MVP 包装 MVC。看看你的 MVC 供电的应用程序。通常你有 View 和一个控制器（例如 iOS 上的 Android 或 UIViewController 的片段），它处理点击事件，绑定数据和观察者 ListView（或者在 iOS 上为UITableView 实现一个 UITableViewDelegate）等等。如果你有这个图片，现在退一步，想象一下，控制器是视图的一部分，而不是直接连接到你的模型（业务逻辑）。演示者坐在控制器和模型之间的中间，如下所示：

![](http://hannesdorfmann.com/images/mosby/mvp-controller.png)

我想用我以前的博客文章中已经使用的示例来解释这一点。 在这个例子中，你想显示从数据库查询的用户列表。 当用户点击“加载按钮”时，动作开始。 在查询数据库（异步）时，会显示一个 ProgressBar，然后显示带查询项的 ListView。 工作流程如下所示：

![](http://hannesdorfmann.com/images/mosby/mvp-workflow.png)

在我看来，演示者不会取代控制器。相反，演示者“协调”控制器所属的视图。 Controller 是处理单击事件并调用相应的 Presenter 方法的组件。控制器是负责控制动画的组件，如隐藏 ProgressBar 和显示 ListView。控制器正在侦听 ListView 上的滚动事件，即在滚动ListView时进行一些视差项目动画或滚动和滚动工具栏。因此，所有与 UI 相关的东西仍然由控制器控制，而不是由演示者控制（即演示者不应该是 OnClickListener）。主讲者负责协调视图层的整体状态（由 UI 部件和控制器组成）。因此，演示者的工作是告诉视图层现在应该显示加载动画，或者现在应该显示 ListView，因为数据已准备好显示。在上面的工作流程图中可以看到一个很好的例子：用户点击“加载用户按钮”（步骤1）后，视图不直接显示加载动画（ProgressBar）。演示者（步骤2）明确告诉视图显示加载动画。

顺便说一下：是的，我认为 MVP 在 iOS 上也很适合！

## Tip 4: Take separation of Model, View and Presenter serious

这可能是显而易见的，但要实现一个干净的，模块化的，可重用的和可维护的代码库，您应该仔细查看代码，并问问自己该做什么代码行。 是与 View（UI 组件或 UI 控制器，如 OnClickListener）或执行演示者工作（协调视图状态）或业务逻辑（即加载数据）相关的代码行。 以下（伪）代码将显示一个 [BLOB](http://www.antipatterns.com/briefing/sld024.htm)Fragment（所有内容都在一个巨大的类中），用于显示登录表单（类似于邮件示例登录）：

```java
 public class LoginFragment extends Fragment {

   @InjectView(R.id.username) EditText username;
   @InjectView(R.id.password) EditText password;
   @InjectView(R.id.loginButton) ActionProcessButton loginButton;
   @InjectView(R.id.errorView) TextView errorView;
   @InjectView(R.id.loginForm) ViewGroup loginForm;

   AsyncTask loginTask;
   AccountManager accountManager;

   public void onCreateView(LayoutInflater inflater, ViewGroup root, Bundle b){
     return inflater.inflate(R.layout.fragment_login, root, false);
   }

   @OnClick(R.id.loginButton) public void onLoginClicked() {


     // Controll UI components --> Controllers job
     errorView.setVisibility(View.GONE);
     loginForm.clearAnimation();

     doLogin();
   }

   private void doLogin(){

     // Coordinates the state of the view --> Presenters job
     showLoading();


     loginTask = new AsyncTask(){

       // returns true if login successful, otherwise false
       protected boolean doInBackground(){

         // Interacts with Business logic --> Presenters job
         User user = accountManager.login(username, password);
         return user != null;
       }

       protected void onPostExecute(Boolean successful) {

         // Coordinates the state of the view --> Presenters job
          if (successful){
            loginSuccessful();
          } else {
            showError();
          }
        }
      }.execute();

    }

    private void showError(){

     // Manipulate the UI --> Controllers job
     loginForm.clearAnimation();
     Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
     loginForm.startAnimation(shake);
     loginButton.showLoading(false);

     errorView.setVisibility(View.VISIBLE);
    }

    @Override public void showLoading() {

    errorView.setVisibility(View.GONE);
    loginButton.showLoading(true);
  }


  @Override public void loginSuccessful() {
    getActivity().finish();
  }

 }
```

我已经在 MVP 中指出了责任。让我们重构一个巨大的类并应用 Model-View-Presenter 模式。模型是 AccountManager。这个观点应该没有模型的知识，所以把这个抽出来。所有协调 Views 模式的东西都应该移到 Presenter 中。所以 AsyncTask 被移到 Presenter 中。演示者必须能够协调视图。因此，我们引入了一个界面，用于在 ViewView 中定义方法 showError（）， loginSuccessful（）和 showLoading（）。通过定义接口，我们保持 Presenter 尽可能独立于平台（Presenter 没有与 Android SDK 相关的代码）。此外，它使编写单元测试更容易（阅读更多关于我以前的博客文章）。正如我们在前面的讨论中所讨论的，Fragment 是一个控制器，因此也是 View 的一部分。所以最后的片段应该只包含控制 UI 组件的代码。重构代码（具有 Mosby View State 支持）如下所示：

```java
 public class LoginFragment extends MvpViewStateFragment<LoginView, LoginPresenter>
    implements LoginView {

  @InjectView(R.id.username) EditText username;
  @InjectView(R.id.password) EditText password;
  @InjectView(R.id.loginButton) ActionProcessButton loginButton;
  @InjectView(R.id.errorView) TextView errorView;
  @InjectView(R.id.loginForm) ViewGroup loginForm;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setRetainInstance(true);
  }

  @Override protected int getLayoutRes() {
    return R.layout.fragment_login;
  }

  @Override public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    super.onViewCreated(view, savedInstanceState);

    int startDelay = getResources().getInteger(android.R.integer.config_mediumAnimTime) + 100;
    LayoutTransition transition = new LayoutTransition();
    transition.enableTransitionType(LayoutTransition.CHANGING);
    transition.setStartDelay(LayoutTransition.APPEARING, startDelay);
    transition.setStartDelay(LayoutTransition.CHANGE_APPEARING, startDelay);
    loginForm.setLayoutTransition(transition);
  }

  @Override public ViewState createViewState() {
    return new LoginViewState();
  }

  @Override public LoginPresenter createPresenter() {
    return new LoginPresenter();
  }

  @OnClick(R.id.loginButton) public void onLoginClicked() {

    // Check for empty fields
    String uname = username.getText().toString();
    String pass = password.getText().toString();

    loginForm.clearAnimation();

    // Start login
    presenter.doLogin(new AuthCredentials(uname, pass));
  }

  @Override public void onNewViewStateInstance() {
    showLoginForm();
  }

  @Override public void showLoginForm() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoginForm();

    errorView.setVisibility(View.GONE);
    setFormEnabled(true);
    loginButton.setLoading(false);
  }

  @Override public void showError() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowError();

    loginButton.setLoading(false);

    if (!isRestoringViewState()) {
      // Enable animations only if not restoring view state
      loginForm.clearAnimation();
      Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
      loginForm.startAnimation(shake);
    }

    errorView.setVisibility(View.VISIBLE);
  }

  @Override public void showLoading() {

    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoading();
    errorView.setVisibility(View.GONE);
    setFormEnabled(false);

    loginButton.setLoading(true);
  }

  private void setFormEnabled(boolean enabled) {
    username.setEnabled(enabled);
    password.setEnabled(enabled);
    loginButton.setEnabled(enabled);
  }

  @Override public void loginSuccessful() {
    getActivity().finish();
  }

}
```

正如你所看到的，LoginFragment 现在只包含与 UI 组件相关的代码。 演示者看起来像这样（使用 RxJava 而不是 AsyncTask）：

```java
public class LoginPresenter extends MvpBasePresenter<LoginView> {

  private AccountManager accountManager;
  private Subscriber<Account> subscriber;

  @Inject public LoginPresenter(AccountManager accountManager) {
    this.accountManager = accountManager;
  }

  public void doLogin(AuthCredentials credentials) {

    if (isViewAttached()) {
      getView().showLoading();
    }

    // Kind of "callback"
    subscriber = new Subscriber<Account>() {
      @Override public void onCompleted() {
        if(isViewAttached()){
          getView().loginSuccessful();
        }
      }

      @Override public void onError(Throwable e) {
        if (isViewAttached()){
          getView().showError();
        }
      }

      @Override public void onNext(Account account) {
      }
    };

    // do the login
    accountManager.doLogin(credentials).subscribe(subscriber);
  }

}
```

所以，现在我们有两个类，每行大约有100行代码，而不是有一个超过300行代码的巨大片段。而且，现在我们拥有可重用，可维护的代码和明确的职责分离。

## Tip 5: Presentation Model

在一个完美的世界中，我们将获得数据（模型）完全适合在我们的 GUI（视图）中显示它的方式。 通常情况下，您通过公共 API 从后端检索数据，这些公共 API 无法更改以适合您的UI需求。 实际上，后端根据你的用户界面提供一个 API 并不是一个好主意，因为如果你改变你的用户界面，你也可能需要改变后端。 因此，您必须将模型转换为 GUI 可以轻松显示的内容。 一个典型的例子是从一个 REST json API 中加载一个 Items 列表，比如一个 List of Users，并将它们显示在一个 ListView 中。 在 MVP 这个完美的世界里，这样工作：

![](http://hannesdorfmann.com/images/mosby/p-model1.png)

这里没有新东西。 List<User> 被加载，并且 GUI 通过使用 UserAdapter 在 ListView 中显示用户。 我很确定你以前使用过百万次的 ListView 和 Adapter，但是你有没有想过 Adapter 背后的想法？ 适配器使您的模型可以通过 android UI 小部件显示。 这是[适配器设计模式](https://sourcemaking.com/design_patterns/adapter)，因此命名。 如果我们想要支持手机和平板电脑，但是都以不同的方式显示项目呢？ 我们实现了两个适配器：PhoneUserAdapter 和 TabletUserAdapter，我们在运行时选择正确的一个。

那就是“完美的世界”情景。 如果我们必须对用户列表进行排序或者显示一些必须以更复杂（和 CPU 密集型）方式进行计算的事情呢？ 我们不能在 UserAdapter 中这样做，因为您将注意到滚动性能问题，因为在主 UI 线程上完成了艰苦的工作。 因此，我们在一个单独的线程中做到这一点。 有两个问题：第一个是我们如何转换数据？ 我们拿我们的用户类，并添加一些额外的领域，用户类？ 我们是否覆盖用户类的值？

```java
public class User {
  String firstname;
  String lastname;
}
```

假设我们的 UserView 想要显示完整的名字并且计算一个排序，

```java
public class User {
  String firstname;
  String lastname;
  int ranking;

  public String getFullname(){
    return firstname +" "+lastname;
  }
}
```

虽然引入方法 getFullname（）没问题，但添加一个字段排名可能会导致问题，因为我们假设用户从后端检索，并没有在它的 json 表示中的排名。 所以，如果你看看你的 json api 提要，并将其与我们的用户类进行比较，最后但不是最小的排名将作为默认值为零，因为我们还没有计算排名。 如果我们使用了一个对象而不是一个整数，那么默认值就是 null，并且很可能会遇到 NullPointerException。

解决方案是引入一个[演示模型](http://martinfowler.com/eaaDev/PresentationModel.html)。这个模型只是为我们的 GUI 优化的一个类：

```java
public class UserPresentationModel {
  String fullname;
  int ranking;

  public UserPresentationModel(String fullname, int ranking) { ... }
}
```

通过这样做，我们确信排名总是被设置为一个具体值，并且在滚动 ListView（PresentationModel 在独立线程中被实例化）时，fullname 不会被计算。 UserView 现在显示 List <UserPresentationModel> 而不是 List <User>。

第二个问题是：在哪里做异步转换？视图，模型或演示者？ View 很明显会进行这种转换，因为 View 知道如何在屏幕上显示事物。

![](http://hannesdorfmann.com/images/mosby/p-model2.png)

PresentationModelTransformer 是接收 List <User> 并将其转换为 List <UserPresentationModel>（适配器模式，因此我们有两个适配器：一个转换为表示模型和 UserAdapter 以在 ListView 中显示它们）的组件。 将PresentationModelTransformer 集成到视图中的优势在于，视图知道如何显示内容，并且可以在手机和平板电脑优化的演示模型（可能平板电脑的用户界面具有其他要求，然后是调用 UI）之间轻松进行内部更改。 但是，最大的缺点是现在视图必须控制异步线程和视图状态（在转换正在进行中显示 ProgressBar？！？），这显然是 Presenter 的工作。 因此，让转换成为视图的一部分并不是一个好主意。 在演示者中包括转换是要走的路：

![](http://hannesdorfmann.com/images/mosby/p-model3.png)

正如我们前面已经讨论过的，Presenter 负责协调 View，因此 Presenter 告诉视图在 UserPresentationModel 转换完成后显示 ListView。 演示者还可以控制所有异步线程（转换运行异步），并在需要时取消它们。 顺便说一下：使用 RxJava 可以使用像 map（）或 flatMap（）这样的操作来无缝地进行这种转换（关于线程）。 如果我们想要支持手机和平板电脑，我们可以使用不同的 PresentationModelTransformer 实现来定义两个 Presenter  PhoneUserPresenter 和 TabletUserPresenter 。在 Mosby，View 创建了 Presenter。 由于 View 在运行时知道如果View是手机或平板电脑，它可以在运行时 Presenter 中选择实例化（PhoneUserPresenter 或 TabletUserPresenter）。 或者，您可以将一个 UserPresenter 用于手机和平板电脑，并只需替换 PresentationModelTransformer 实现，即使用依赖注入。

## Tip 6: Navigation is a UI thing

在其他 MVP 库或平台，如单个页面的 HTML 网站（JavaScript），它的演示者被实例化为第一个组件，并且演示者创建视图。 在这种情况下，从 Presenter 导航到 Presenter 是有意义的。 但是，Android 上的情况并非如此（或者至少在使用 Mosby 的情况下），因为 Android Framework 将 Activity 定义为入口点，Activity 被视为 View 层的一部分。 因此，让演示者启动一个新的活动的意图是没有意义的。

## Tip 7: onActivityResult() and MVP

有些情况下，如果您想与其他应用程序（如相机应用程序）进行通信，则必须使用 onActivityResult（）。 你应该将结果转发给演示者吗？ 这取决于：如果你只是想显示照片在 ImageView 中，那么没有必要将其转发给演示者。 请记住，演示者负责协调状态。 因此，在大多数情况下在 ImageView 中显示该图像不会改变状态，可以直接完成。

```java
 @Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        imageView.setImageBitmap(imageBitmap);
    }
}
```

如果你必须做一些额外的图像处理，那么你会做的异步。在这种情况下，您应该使用演示者并让演示者控制视图（如在处理图像时显示加载视图），如下所示：

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
   if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
       Bundle extras = data.getExtras();
       Bitmap imageBitmap = (Bitmap) extras.get("data");
       presenter.makeThumbnail(imageBitmap);
   }
}
```

## Tip 8: MVP and EventBus - A match made in heaven

EventBus 允许您通过发布和接收事件在分离的组件之间进行通信。 我假设你已经熟悉了一个 EventBus 的概念。  EventBus 不是 MVP 相关的，但它适合 MVP，就像手套一样。 我在2009年底与 GWT 开发了第一次与 EventBus 和 MVP 联系，这是令人兴奋的。 从现在开始，我总是使用 MVP 与 EventBus，所以我在 Android 上做。 使用 RxJava 可以用 RxJava 的观察者 “push updates mechanism” 来替换 EventBus 所做的一些事情（查看具体用例的[SqlBrite](https://github.com/square/sqlbrite) ）。 RxJava 的推送机制与 EventBus 之间的区别在于，在 RxJava 中，您显式地必须订阅具体的可观察事件，而 EventBus 稍微松散一些，因为您只需要知道正在监听的事件。 所以当我在这里谈论一个 EventBus 的时候，你可以想到每一个基于推送的更新机制（比如 RxJava）。

我看到 EventBus 有两个用例：

- 用于业务逻辑事件的 EventBus：与业务逻辑相关的事件被发布到 EventBus。 正如我们前面所讨论的那样， View 不应该有业务逻辑的任何知识。 因此，演示者是通过 EventBus 监听业务逻辑事件的组件。 在 Mosby 中，您应该分别在 attachView（）和 detachView（）中从 EventBus 注册和注销演示者。 为什么不在 Presenter 的构造函数中注册？ 因为如果您保留了视图（如保留碎片），则只能创建一次演示者，但在方向更改期间，可以多次将演示者附加和分离视图。 在邮件示例中，我使用了很多 EventBus。 您可能在视频中注意到的第一件事是，登录成功后，所有视图自动开始加载数据。 如果您在平板电脑上启动邮件应用程序，则可以更好地看到这一点。[Mosby: login successfull propagation over EventBus](https://www.youtube.com/watch?v=d898QONavBo)

  这不是通过在 Activity.onResume（）中自动执行刷新来实现的。 LoginPresenter 触发 LoginSuccessfulEvent 通知所有其他演示者，该用户已成功登录。 演示者然后将触发重新加载并更新他们的视图。

  ![](http://hannesdorfmann.com/images/mosby/login-eventbus.jpg)

- 用于 UI 事件的 EventBus：另一个用例是在 UI 中使用 EventBus，或者在 Fragments 和 Activities 之间进行通信。 实现碎片和主机活动之间通信的官方文档建议是定义一个侦听器接口，并将 Activity 的片段onAttach（）方法转换为侦听器：

  ```java
    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);

        // This makes sure that the container activity has implemented
        // the callback interface. If not, it throws an exception
        try {
            mCallback = (OnHeadlineSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString()
                    + " must implement OnHeadlineSelectedListener");
        }
    }
  ```

  我不喜欢（将对象投射到界面）。如果在他的代码中看到类似的东西，所有其他的 Java 开发人员，即在 J2EE 世界中，都会惊恐地举起双手。那么，为什么它应该是推荐的方式（顺便说一句，你在 iOS 上做的是相同的）？我通常使用 EventBus 来做到这一点。该片段触发 EventBus 上的事件，并在 EventBus 上订阅该父事件。如果您有一个拆分窗格布局，在左侧显示片段 A，在右侧显示片段 B，则 EventBus 的优势甚至更大。如果片段 A 想与片段 B 进行通信，则文档建议片段 A 应与父活动对话，然后活动将信息转发给片段 B.使用事件总线片段 A 可以发射事件和片段 B，事件总线上的事件，立即得到通知，而没有作为中间人的活动。这样，所有3个组件（活动，片段 A 和片段 B）都是分离的（没有人相互参照）。

  EventBus 有用的常见用例是实现导航抽屉时的情况。如果你点击一个菜单项，Activity 应该用与被点击的菜单项关联的一个片段替换当前的主片段。事实证明，Android 本身已经提供了一个你已经使用的 EventBus：Intents。您可以像使用邮件示例一样使用 Intent，而不使用外部 EventBus。我将 MainActivity 设置为 android：launchMode =“singleTop”，并在 onNewIntent（）中检查导航事件。 Intents 的优势在于，这种导航可以在系统中使用（而不仅仅是像 EventBus 这样的应用程序）。例如，当收到新的邮件时，状态栏将显示通知。点击通知打开邮件。通知中可能会有一个按钮，通过单击该按钮打开收件箱。而且，如果您在应用程序中使用 EventBus 进行导航，则可能会面临必须处理 Intents 和 EventBus 中的事件以在应用程序中导航的问题。您应该考虑一下这种情况，如果您的应用程序无法运行这种问题，则只能使用 EventBus 进行导航。

## Tip 9: Optimistic propagation

让我们来看看邮件应用程序。 当您星星或取消星标邮件时，星号图标会立即更改。 但是，保存邮件已经更改需要两秒钟，可能会失败。 为了获得更好的用户体验，我们假设更改邮件将会成功。 它给你的应用程序的用户一个立即反馈，并感觉到你的应用程序闪电般的工作。

[mosby optimistic propagation staring maiil](https://www.youtube.com/watch?v=7HPtTg35ZS0)

乍一看，您可能会认为这对于您目前正在使用的简单应用来说太复杂了。 看来你必须尊重这么多的东西来实现它。 不要担心，像 MVP 和 EventBus 这样的干净架构很容易：

```java
 public class BaseRxMailPresenter<V extends BaseMailView<M>, M>
     extends BaseRxAuthPresenter<V, M> {

   @Inject public BaseRxMailPresenter(MailProvider mailProvider, EventBus eventBus) {
     super(mailProvider, eventBus);
   }

   public void starMail(final Mail mail, final boolean star) {

     // optimistic propagation
     if (star) {
       eventBus.post(new MailStaredEvent(mail.getId()));
     } else {
       eventBus.post(new MailUnstaredEvent(mail.getId()));
     }

     mailProvider.starMail(mail.getId(), star)
         .subscribe(new Subscriber<Mail>() {
           @Override public void onCompleted() {
           }

           @Override public void onError(Throwable e) {
             // Oops, something went wrong, "undo"
             if (star) {
               eventBus.post(new MailUnstaredEvent(mail.getId()));
             } else {
               eventBus.post(new MailStaredEvent(mail.getId()));
             }

             if (isViewAttached()) {
               if (star) {
                 getView().showStaringFailed(mail);
               } else {
                 getView().showUnstaringFailed(mail);
               }
             }
           }

           @Override public void onNext(Mail mail) {
           }
         });

     // Note: that we don't cancel this operation in detachView().
     // We want to ensure that this operation finishes
   }

  ...

 }
```

如果我们为 Mail 发送邮件，我们会发送 MailStaredEvent 消息，如果我们取消发送邮件，我们会发送 MailUnstaredEvent 消息。正如您在视频中看到的那样，显示邮件列表的左侧窗格也会更新，因为演示者已在这些事件中注册。诀窍是我们在进行相应的业务逻辑调用之前触发相应的事件，并在业务逻辑调用失败的情况下使用相应的事件来“撤销”它。在 Presenter 中管理这个逻辑非常简单，就像你在代码中看到的那样。但是我们也必须更新观点。在 MVP 中，View 和业务逻辑之间有明确的区分，因此 Presenter 负责通过调用视图界面中定义的相应方法来更新视图。正如你看到一个干净的 MVP 架构和 EventBus 做这样的“复杂”的事情是非常容易的。 Presenter 正在做业务逻辑人员，然后你的 View 实现中的代码只是一行代码。但是，如果你在 Fragment（没有 MVP）中这么做，那么是的，保持正确的状态可能会更复杂一些。

请注意，我们并没有在 presenter.detachView（）中取消这个业务逻辑调用，因为我们要确保这个调用完成。 您现在可能会问自己，如果这在方向更改期间不会导致内存泄漏。 答案是“这取决于内存泄漏的定义”。 在屏幕方向更改期间，视图会从演示者分离。 所以像 Activity 或 Fragment 这样的大内存消耗视图对象不会泄漏内存。 但是，Presenter 创建一个新的匿名订阅服务器对象。 每个匿名对象都对周围的对象有一个很强的引用。 所以是的，只要订阅者没有被接受（业务逻辑调用已经完成），演示者就不能被垃圾收集。 但我不会把这称为内存泄漏。 这是一种暂时和想要的内存泄漏，从我的角度来看，完全没问题，因为演示者是一个轻量级的对象，你知道演示者可以在业务调用完成后被垃圾收集。

当一个邮件从“收件箱”移动到“垃圾邮件”时，乐观传播改善了您应用的用户体验的另一个示例是：[mosby optimisitc propagation change label](https://www.youtube.com/watch?v=rA22tsGdeY4)

## Tip 10: MVP scales

大多数情况下，您有一个 View（和 Presenter）填充整个屏幕。 但事实并非如此。 通常片段是一个很好的候选人，可以用自己的 Presenter 将屏幕分割成模块化的分离视图。 示例邮件客户端也可以通过单击发件人图像（头像）打开配置文件来实现此功能：

[mosby - profiles - subviews](https://www.youtube.com/watch?v=4Mce1ljHJz0)

在上面的视频中显示的邮件应用程序的示例中，活动本身是一个 ViewPager 作为内容视图。演示者加载 ProfileScreen 的列表。 ProfileScreen 只是一个 POJO。

```java
public class ProfileScreen implements Parcelable {

  public static final int TYPE_MAILS = 0;
  public static final int TYPE_ABOUT = 1;

  int type;
  String name;

  private ProfileScreen() {
  }

  public ProfileScreen(int type, String name) {
    this.type = type;
    this.name = name;
  }

  public int getType() {
    return type;
  }

  public String getName() {
    return name;
  }
}
```

```java
 public class ProfileScreensAdapter extends FragmentPagerAdapter {


  @Override public Fragment getItem(int position) {

    ProfileScreen screen = screens.get(position);
    if (screen.getType() == ProfileScreen.TYPE_MAILS) {
      return new ProfileMailsFragmentBuilder(person).build();
    }
    if (screen.getType() == ProfileScreen.TYPE_ABOUT) {
      return new AboutFragmentBuilder(person).build();
    }

    throw new RuntimeException("Unknown Profile Screen (no fragment associated with this type");
  }

  @Override public CharSequence getPageTitle(int position) {
    return screens.get(position).getName();
  }
}
```

ProfileScreen 的加载列表需要2秒（模拟从后端动态加载 ViewPager 的“屏幕显示为标签”）。 所以基本上我们有一个 LCE（Loading-Content-Error），所以我们可以使用 Mosby 的 MvpLceViewStateActivity。 接下来， ViewPager 中的每个片段都是一个 MVP 视图，并拥有自己的演示者。 我想你会得到整体情况：一个 MVP 视图可以包含独立的 MVP 视图。

## Tip 11: Not every View needs MVP

这可能是显而易见的，但是一旦您处于“我是超级软件架构师”模式，您可能会忘记有些视图仅显示静态内容。静态内容不需要 MVP。

## Tip 12: Only display one Model per MVP view

Mosby 假设每个 MVP 视图都只显示一个模型（注意，显示项目列表仍然显示一个模型，列表）。 为什么？ 因为在同一视图中处理不同的模型会大大增加复杂性，而且我们显然希望避免意大利面代码。 在示例应用程序中，我遇到了菜单中的问题。 我还没有重构，还没有给你一个具体的代码示例了解问题的可能性。 看看菜单的截图：

![](http://hannesdorfmann.com/images/mosby/mail-menu.png)

标题显示当前已认证的用户，表示为 Account，而可点击的菜单项是 List <Label>（仅忽略统计信息）。 所以我们有两个模型显示在相同的 MVP 视图，即 MenuFragment。 问题是，如果在同一视图中使用两个模型，它会使代码更加复杂和难以维护。 如果你决定使用 Mosby 的 ViewState，事情会变得更加复杂。 因为现在你不仅需要在加载标签时存储显示标签的状态，加载标签和错误，还必须存储用户是否被认证。

解决这个问题的方法是将一个 MVP 视图分成两个视图，分别拥有自己的 Presenter 和自己的 ViewState：

![](http://hannesdorfmann.com/images/mosby/menu-refactored.jpg)

## Tip 13: Android Services

Android 服务是 Android 应用程序开发的基础部分。 服务显然是“商业逻辑”。 因此，演示者显然负责与服务进行沟通。 在邮件应用程序中，我使用了一个 IntentService 来发送邮件。 SendMailService 通过 EventBus 连接到WritePresenter：

```java
public class WritePresenter extends MvpBasePresenter<WriteView> {

  private EventBus eventBus;
  private IntentStarter intentStarter;

  public void writeMail(Context context, Mail mail) {
    getView().showLoading();
    intentStarter.sendMailViaService(context, mail);
  }

  public void onEventMainThread(MailSentErrorEvent errorEvent){
    if (isViewAttached()){
      getView().showError(errorEvent.getException());
    }
  }

  public void onEventMainThread(MailSentEvent event){
    if (isViewAttached()){
      getView().finishBecauseSuccessful();
    }
  }
}
```

## Tip 14: Use LCE only if you have LCE Views

LCE 代表加载内容错误。 Mosby 随 MvpLceViewStateFragment 一起实现了 MvpLceView，它显示内容（如邮件列表）或加载（如进度条）或错误视图（如显示错误消息的 TextView）。 这可能是方便的，因为你不必实现显示内容的切换，显示错误，显示你自己的加载，但是你不应该把它用作所有东西的基类。 只有在视图可以达到所有三种视图状态（显示加载，内容和错误）时才使用它。 看看登录视图：

```java
public interface LoginView extends MvpView {

  // Shows the login form
  public void showLoginForm();

  // Called if username / password is incorrect
  public void showError();

  // Shows a loading animation while checking auth credentials
  public void showLoading();

  // Called if sign in was successful. Finishes the activity. User is authenticated afterwards.
  public void loginSuccessful();
}
```

乍一看，你可能会认为 LoginView 是一个 MvpLceView，只是将 showContent（）重命名为 showLoginForm（）。 仔细看看 MvpLceView 的定义：

```java
public interface MvpLceView<M> extends MvpView {

  public void showLoading(boolean pullToRefresh);

  public void showContent();

  public void showError(Throwable e, boolean pullToRefresh);

  public void setData(M data);

  public void loadData(boolean pullToRefresh);
}
```

类似于 loadData（）和 setData（）的方法在 LoginView 中不需要，也不需要支持 pull-to-refresh。你可以简单地用一个空的实现来实现这个方法。但是这是一个坏主意，因为定义一个接口就像是与其他软件组件签订协议：你的接口承诺该方法是“可调用的”（在某种意义上是有用的），但是在调用这个方法时无所作为违反了合同。另外，如果你有很多从其他组件调用的方法，但是你什么都不做，你的代码就会变得更难理解和维护。而且，如果你使用 MvpLceView，你会使用已经存在的 MvpLcePresenter 实现作为 LoginPresenter 的基类。问题是 MvpLcePresenter 对于 MvpLceView 是“优化的”。因此，您可能必须在您的 Presenter 中实施一些解决方案，该解决方案从 MvpLcePresenter 扩展到实现您想要执行的操作。如果您的视图没有完整的 LCE 支持，那么就避免使用 LCE 相关的类。

## Tip 15: Writing custom ViewStates

Mosby 有一个机制来保存和恢复对于处理方向变化很有用的视图状态。 一些 MvpLceView 的 ViewState 实现已经由 Mosby 提供。 然而，LoginView 不是 MvpLceView（在前面的技巧中讨论过），因此需要自己的视图状态实现。 编写自定义视图状态非常简单：

```java
public class LoginViewState implements ViewState<LoginView> {

  final int STATE_SHOW_LOGIN_FORM = 0;
  final int STATE_SHOW_LOADING = 1;
  final int STATE_SHOW_ERROR = 2;

  int state = STATE_SHOW_LOGIN_FORM;

  public void setShowLoginForm() {
    state = STATE_SHOW_LOGIN_FORM;
  }

  public void setShowError() {
    state = STATE_SHOW_ERROR;
  }

  public void setShowLoading() {
    state = STATE_SHOW_LOADING;
  }

  /
    Is called from Mosby to apply the view state to the view.
    We do that by calling the methods from the View interface (like the presenter does)
   /
  @Override public void apply(LoginView view, boolean retained) {

    switch (state) {
      case STATE_SHOW_LOADING:
        view.showLoading();
        break;

      case STATE_SHOW_ERROR:
        view.showError();
        break;

      case STATE_SHOW_LOGIN_FORM:
        view.showLoginForm();
        break;
    }
  }

}
```

我们简单地将当前的视图状态作为整数状态存储在内部。 方法 apply（）从内部调用 Mosby，这是我们将视图状态恢复到相关视图的点。 您可能想知道 ViewState 如何连接到您的片段或活动。 MvpViewStateFragment 在内部有一个由 Mosby 调用的方法 createViewState（），你必须实现这个方法。 你只需要返回一个 LoginViewState 实例。 但是，您必须手动设置 LoginViewState 的“内部状态”。 通常，您可以在 LoginView 界面定义的方法中执行此操作，如下所示：

```java
public class LoginFragment extends MvpViewStateFragment<LoginView, LoginPresenter>
    implements LoginView {

  @Override public void showLoginForm() {

    // Set View state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoginForm();

    errorView.setVisibility(View.GONE);

    ...
  }

  @Override public void showError() {

    // Set the view state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowError();

    if (!isRestoringViewState()) {
      // Enable animations only if not restoring view state
      loginForm.clearAnimation();
      Animation shake = AnimationUtils.loadAnimation(getActivity(), R.anim.shake);
      loginForm.startAnimation(shake);
    }

    errorView.setVisibility(View.VISIBLE);

    ...
  }

  @Override public void showLoading() {

    // Set the view state
    LoginViewState vs = (LoginViewState) viewState;
    vs.setShowLoading();

    errorView.setVisibility(View.GONE);

    ...
  }
}
```

有时你必须知道是否从演示者调用方法或者恢复视图状态，通常是使用动画时。 你可以像 showError（）那样用 isRestoringViewState（）来检查（见上）。

## Tip 16: Testing custom ViewState

由于 LoginView 和 LoginViewState 是简单的旧的 Java 类，与 android 框架没有依赖关系，所以你可以简单地为 LoginViewState 编写简单的旧 Java 单元测试：

```java
@Test
public void testShowLoginForm(){

  final AtomicBoolean loginCalled = new AtomicBoolean(false);
  LoginView view = new LoginView() {

    @Override public void showLoginForm() {
      loginCalled.set(true);
    }

    @Override public void showError() {
      Assert.fail("showError() instead of showLoginForm()");
    }

    @Override public void showLoading() {
      Assert.fail("showLoading() instead of showLoginForm()");
    }

    @Override public void loginSuccessful() {
      Assert.fail("loginSuccessful() instead of showLoginForm()");
    }
  };

  LoginViewState viewState = new LoginViewState();
  viewState.setShowLoginForm();
  viewState.apply(view, false);
  Assert.assertTrue(loginCalled.get());

}
```

这个想法很简单：我们创建一个 LoginViewState 并设置内部状态 viewState.setShowLoginForm（）。 接下来我们创建一个模拟视图并在视图状态上调用 apply（）方法。 然后我们检查预期的方法是否被调用。 你也可以用Mockito 这样的模仿库来模拟 LoginView，但是我想保持样本简单而且独立于其他第三方库。

为了测试 LoginFragment 是否实现了 LoginView，在屏幕方向改变时恢复它的状态是正确的，所以测试 ViewState 就足够了，因为你可以假设 Mosby 的内部工作正常。

到目前为止，我们只测试了恢复工作是否正确。 我们还必须测试是否正确设置视图状态。 不过，这是在 LoginFragment 中完成的，所以我们必须测试 Fragment 而不是 LoginViewState。 感谢 Robolectric，这也很简单：

```java
public void testSettingState(){
  LoginFragment fragment = new LoginFragment();
  FragmentTestUtil.startFragment(fragment);
  fragment.showLoginForm();

  LoginViewState vs = (LoginViewState) fragment.getViewState();

  Assert.assertEquals(vs.state, vs.STATE_SHOW_LOGIN_FORM);
}
```

我们只需调用 showLoginForm（），并在此调用之后检查 ViewState 是否为 STATE_SHOW_LOGIN_FORM。

您还可以测试 LoginFragment 是否通过书写测试（即使用 espresso）来正确处理屏幕方向更改，但重点是在 Mosby 中，您在视图状态和视图状态之间有清晰的分离。 因此，您可以独立测试视图状态，这通常需要编写更少的测试代码（将上面的代码与仪器测试进行比较）。

## Tip 17: ViewState variants

也许现在还不清楚 ViewState 究竟是什么。 用一个例子来解释 ViewState 的概念很容易：例如显示一个加载视图（ProgressBar）而不是内容视图（ListView 显示项目）。 这显然是两种不同的视图状态：视图应显示加载视图或内容视图。 但是，如果我们为内容视图（ListView 周围的 SwipeRefreshLayout）添加拉到刷新支持呢？ 如果用户触发 pull-to-refresh 动作，我们同时显示 ListView（内容视图）和加载指示器。 问题是：那是哪个状态？ 视图总是处于一种状态（这是视图状态的定义）。 所以在进行刷新时，视图的状态不是同时显示内容和显示加载。 内部视图仍处于一个状态。 有两种可能性：

- 引入一个新的状态来进行刷新（同时显示 ListView 和 ProgressBar）。
- 扩展“显示加载视图状态”：另外，我们还存储触发拉取到刷新的信息（即布尔标志），并显示 ListView。 使用现有 ViewState 并使用其他信息（即，用于拉动到刷新的布尔标志）来扩展该视图被称为 ViewState 的变体。 我们通过添加带有 “show loading view state” 两个变体的布尔 pull-to-refresh 标志来扩展 “show loading view state”。 第一个变体是 pull-to-refresh 标志为 true（view 应该显示 ListView 和 pull-to-refresh 标志），第二个变体是 pull-to-refresh 标志为 false 的地方（view 只能显示 ProgressBar）。

它使用这两个选项中的哪一个没有什么区别，它只是一个内部实现细节。 与 ViewState 的关键在于，视图所处的视图状态完全相同。但是，一个视图状态可以有几个变体。 具有变体可能需要存储多个信息以及要调用多个视图方法来恢复该视图状态变体。 Mosby 默认的 MvpLceViewState 实现使用方法2，看起来像这样：

```java
 private boolean pullToRefresh;

 @Override
 public void apply(V view, boolean retained) {
   if (currentViewState == STATE_SHOW_LOADING) {

    if (pullToRefresh) {
      view.setData(loadedData);
      view.showContent();
    }

    // view displays pull-to-refresh indicator if pullToRefresh == true,
    // otherwise hides content view or error view and displays big ProgressBar instead
    view.showLoading(pullToRefresh);
  }

  ...
}
```

不过，我建议不要有两个以上的 ViewState 变体，如果一个变体很简单并且不需要复杂的恢复（在 apply（）中），那么只需要定义一个新的视图状态。

好的，现在您应该知道视图状态（即显示加载）和视图状态变体（即显示加载刷新）之间的区别是什么。

我们来看看邮件示例应用程序。 在这里，我们可以找到一些自定义的视图状态，这些状态是从现有的视图状态延伸出 例如 AuthParcelableDataViewState 是 LceViewState 的扩展，有四种状态：

![](http://hannesdorfmann.com/images/mosby/inbox_states.jpg)

添加一个额外的状态是相当容易的。 内部 LceViewstate（AuthParcelableDataViewState 从其继承）具有用于保存当前状态的字段 int 状态。 我们只是重复使用该字段为用户未通过身份验证的情况添加新状态，如下所示：

```java
public class AuthParcelableDataViewState<D extends Parcelable, V extends AuthView<D>>
    extends ParcelableDataLceViewState<D, V> implements AuthViewState<D, V> {

  public static final int SHOWING_AUTHENTICATION_REQUIRED = 2;

  @Override public void apply(V view, boolean retained) {

    if (currentViewState == SHOWING_AUTHENTICATION_REQUIRED) {
      view.showAuthenticationRequired();
    } else {
      // call super to handle the states showing loading or error or content
      super.apply(view, retained);
    }
  }

  @Override public void setShowingAuthenticationRequired() {
    currentViewState = SHOWING_AUTHENTICATION_REQUIRED;

    // Delete any previous stored data of other view states
    loadedData = null;
    exception = null;
  }
}
```

代码应该是自我解释。在 SearchFragment 实现 SerachView 中，我已经实现了分页（通过以块填充整个列表来显示邮件列表，即显示20个邮件，并且如果用户已经滚动到该列表然后加载下20封邮件）。基本上，SearchView 显示邮件列表，加载并显示错误视图。因此我们可以使用基于 LCE 的视图状态实现。问题是使用哪个 ViewState 添加一个变体：“显示加载” 或 “显示内容”？在这种情况下，这取决于你的实现。我所做的显示“加载更多”是我添加了一个额外的 ViewType SearchResultAdapter 显示在 RecyclerView（内容视图）中匹配搜索条件的邮件列表。因此，RecyclerView 显示的最后一个项目是显示 ProgressBar 的行。由于触发“加载更多操作”需要滚动内容视图（RecyclerView），因此将视图状态变体添加到 “显示内容” 而不是 “显示加载” 更为自然。也是 RecyclerViews（内容视图）适配器的 “加载更多指示器” 部分。要实现“显示内容”的变体，我们只需在 SearchViewState 中添加一个布尔标志 loadingMore：

```java
// AuthCastedArrayListViewState is a LCE ViewState with an additional "not authenticated" state
public class SearchViewState extends AuthCastedArrayListViewState<List<Mail>, SearchView> {

  boolean loadingMore = false;

  public void setLoadingMore(boolean loadingMore) {
    this.loadingMore = loadingMore;
  }


  @Override public void apply(SearchView view, boolean retained) {

    super.apply(view, retained);

    if (currentViewState == STATE_SHOW_CONTENT) {
      view.showLoadMore(loadingMore);
    }

  }
}
```

如果你对 ViewState Variants 不满意，那就不要使用 Variant。 而是定义新的 ViewStates。 定义新的视图状态更清晰，但是可能需要编写更多的代码。 因为我是一个懒惰的人，我想指出，变体也可以使用。 然而，变体不是一切和每个人的解决方案。

## Tip 18: Not every UI representation is a ViewState

如果您打开空的收件箱，该怎么办？然后，示例邮件应用程序描述：

![](http://hannesdorfmann.com/images/mosby/no_mails_state.png)

那是另一种观点状态吗？不，这只是视图状态“显示内容”列表作为数据的空列表。在 MailsFragment 中，一个空列表就像这样显示。 UI 允许以不同的方式表示相同的视图状态。

## Tip 19: Not every View needs a ViewState

我相信你会发现 Mosby 的 ViewState 概念是 Mosby 提供的最舒适的功能之一。处理方位变化就像是一个开箱即用的魅力。然而，这是与你必须支付的价格。 ViewState 存储在 onSaveInstanceState（）中使用的 Bundle 中。该捆绑包的大小限制在 1 MB 左右。如果您的视图显示了一个巨大的项目列表，那么您可能会运行超过 1 MB 的限制，因为项目列表也存储在 Bundle 中（作为 ViewState 的一部分）。如果使用保留片段，则不会遇到这种问题，因为视图状态存储在内存中而不是 Bundle 中。另一件事你必须记住的是，一个活动可以被 Android 操作系统销毁，并重新创建，如果你回到活动。返回时 ViewState 得到恢复，Activity 可能会显示陈旧的数据，因为在此期间，您的应用程序的另一个 Activity 中已经编辑了相同的数据。通常这不会经常发生。但是，如果显示敏感数据，则应牢记这一点，在这种情况下最好不要使用 ViewState。

## Tip 20: Avoid Fragments on the backstack and child fragments

我相信你们所有人都听说过 Fragment 的生命周期有多糟糕，而且在这样的生命周期中很难工作。 不幸的是，许多反对 Fragments 的论点是正确的，如果你把 Fragments 放在堆栈上或者使用子片段（片段中的片段）， Fragments 的生命周期会变得更加奇怪。 Mosby 可以对付他们。 但总的来说，如果您知道可能有麻烦，从一开始就避免问题是一个好主意。 你知道把片段放在背后可能会造成问题，那么就避免把片段放在背后。 这同样适用于子片段。

## Tip 21: Mosby supports MVP for ViewGroups

如果你想避免碎片一般你可以做到这一点。 Mosby 提供与 ViewGroups 的 Activities 和 Fragments 相同的 MVP 脚手架，ViewState 支持。 API 与 Activity 和 Fragment 相同。 一些默认的实现已经可用，如 MvpViewStateFrameLayout，MvpViewStateLinearLayout 和 MvpViewStateRelativeLayout。 这个 ViewGroup 实现也是有用的，如果你想在一个片段中有子视图（因为你应该避免子片段）。 当你想重新分配一个邮件的标签时，邮件示例使用它：

[Mosby - Assign label](https://www.youtube.com/watch?v=AWt-JhWi3lo)

这个用于分配标签的“按钮”是一个 MvpViewStateLinearLayout。 它在加载时将 ProgressBar（loadingView）动画化，并使用 ListPopupWindow（contentView）显示标签列表并使用 Toast 通知错误。 顺便说一下：ViewGroups 没有 LCE 默认实现（你将如何实现 LinearLayout？）。

## Tip 22: Mosby provides Delegates

您可能想知道，如果没有代码复制（复制和粘贴相同的代码），Mosby如何为所有类型的视图（Activity，Fragment 和 ViewGroup）提供相同的 API。 答案是委托。 Mosby 从一开始就使用委托（1.0.0版）。 但是，这个委托类有点隐藏，很难理解，它们实际上是可以使用的委托，是公共 API 的一部分，即 ViewStateSupport 和ViewStateManager。 为了方便开发者使用它们，这个委托类已经在 Mosby 1.1.0 中重新命名了。 代理的方法也被重命名为与 Activity 或 Fragments 生命周期方法名称（受到 Android 支持库中最新的 AppCompatDelegate 的启发），以便更好地理解应从哪个 Activity 或 Fragment 生命周期方法调用哪个委托方法：

- MvpDelegateCallback : Mosby 中的每个 MvpView 都需要实现一个接口。基本上它只是提供了一些 MVP 相关的方法，如 createPresenter（）等。这个方法在内部被 ActivityMvpDelegate 或 FragmentMvpDelegate 调用。

- ActivityMvpDelegate : 这是一个界面。 通常你使用 ActivityMvpDelegateImpl 这是默认的实现。 所有你需要做的 Mosby MVP 支持你的自定义活动是调用像 onCreate（）， onPause（）， onDestroy（）等活动生命周期方法相应的委托方法，并实现 MvpDelegateCallback：

  ```java
   public abstract class MyActivity extends Activity implements MvpDelegateCallback<> {

      protected ActivityMvpDelegate mvpDelegate = new ActivityMvpDelegateImpl(this);

      @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mvpDelegate.onCreate(savedInstanceState);
      }

      @Override protected void onDestroy() {
        super.onDestroy();
        mvpDelegate.onDestroy();
      }

      @Override protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        mvpDelegate.onSaveInstanceState(outState);
      }

      ... // other lifecycle methods
    }
  ```

  ​

- FragmentMvpDelegate : 与 ActivityMvpDelegate 相同，但是 Fragments。 要将 Mosby MVP 支持带到您的自定义片段中，您只需要执行以下操作：创建 FragmentMvpDelegate 并从相应的 Fragment 生命周期方法中调用方法。 你的片段也必须实现 MvpDelegateCallback。 通常你使用默认的委托实现 FragmentMvpDelegateImpl

- ViewGroupMvpDelegate : 这个委托用于 ViewGroup，比如 FrameLayout 等，将 Mosby MVP 支持带给你的自定义 ViewGroup。 与 Fragments 相比，生命周期方法更简单：onAttachedToWindow（）和onDetachedFromWindow（）。 默认的实现是 ViewGroupMvpDelegateImpl。

到目前为止，我们已经介绍了如何将 Mosby MVP 支持带到您的自定义 Activity，Fragment 或 ViewGroup。 为了支持 ViewState，你必须使用下面的委托来代替上面讨论的委托（ViewState 委托已经包含了 MVP 委托的功能）：

- MvpViewStateDelegateCallback : 这个接口从 MvpDelegateCallback 扩展，并定义你必须实现的方法，如createViewState（）。
- ActivityMvpViewStateDelegateImpl : 这个委托是 ActivityMvpDelegateImpl 的扩展，其工作原理与前面的代码片段中所示的方式完全相同：必须从相应的活动生命周期方法中调用委托方法。 如上所示，您的自定义活动必须实现 MvpViewStateDelegateCallback 并使用 ActivityMvpViewStateDelegateImpl，而不是非 ViewState 相关的：

```java
  public abstract class MyViewStateActivity extends Activity implements MvpViewStateDelegateCallback<> {

    protected ActivityMvpDelegate mvpDelegate = new ActivityMvpViewStateDelegateImpl(this);

    // The rest is still the same as shown above without ViewState support

    @Override protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      mvpDelegate.onCreate(savedInstanceState);
    }

    @Override protected void onDestroy() {
      super.onDestroy();
      mvpDelegate.onDestroy();
    }

    @Override protected void onSaveInstanceState(Bundle outState) {
      super.onSaveInstanceState(outState);
      mvpDelegate.onSaveInstanceState(outState);
    }

    ... // other lifecycle methods
  }
```

- FragmentMvpViewStateDelegateImpl : 从 FragmentMvpDelegateImpl 扩展到添加 ViewState 支持。
- ViewGroupMvpViewStateDelegateImpl : ViewGroups 的委托像 FrameLayout 来添加 ViewState 支持。

委托的好处是你可以添加 Mosby MVP 和 ViewState 支持任何其他活动，片段或 ViewGroup 不包括在 Mosby 库像DialogFragment。 在邮件示例的主菜单中，您可以看到 “统计” 菜单项。 如果你点击它，就会显示一个DialogFragment get。 这是这样实现的：

```java
public class StatisticsDialog extends DialogFragment
    implements StatisticsView, MvpViewStateDelegateCallback<StatisticsView, StatisticsPresenter> {

  @InjectView(R.id.contentView) RecyclerView contentView;
  @InjectView(R.id.loadingView) View loadingView;
  @InjectView(R.id.errorView) TextView errorView;
  @InjectView(R.id.authView) View authView;

  StatisticsPresenter presenter;
  ViewState<StatisticsView> viewState;
  MailStatistics data;
  StatisticsAdapter adapter;

  // Delegate
  private FragmentMvpDelegate<StatisticsView, StatisticsPresenter> delegate =
      new FragmentMvpViewStateDelegateImpl<>(this);


  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    delegate.onCreate(savedInstanceState);
  }

  @Override public void onDestroy() {
    super.onDestroy();
    delegate.onDestroy();
  }

  @Override public void onPause() {
    super.onPause();
    delegate.onPause();
  }

  @Override public void onResume() {
    super.onResume();
    delegate.onResume();
  }

  @Nullable @Override public View onCreateView(LayoutInflater inflater, ViewGroup container,
      Bundle savedInstanceState) {
    return inflater.inflate(R.layout.fragment_statistics, container, false);
  }

  @Override public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    super.onViewCreated(view, savedInstanceState);
    delegate.onViewCreated(view, savedInstanceState);

    ButterKnife.inject(this, view);
    adapter = new StatisticsAdapter(getActivity());
    contentView.setAdapter(adapter);
    contentView.setLayoutManager(new LinearLayoutManager(getActivity()));
  }

  @Override public void onStart() {
    super.onStart();
    delegate.onStart();
  }

  @Override public void onStop() {
    super.onStop();
    delegate.onStop();
  }

  @Override public void onAttach(Activity activity) {
    super.onAttach(activity);
    delegate.onAttach(activity);
  }

  @Override public void onDetach() {
    super.onDetach();
    delegate.onDetach();
  }

  @Override public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    delegate.onActivityCreated(savedInstanceState);
  }

  @Override public void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    delegate.onSaveInstanceState(outState);
  }

  ...

}
```

委托允许您将功能添加到 [RoboGuice](https://github.com/roboguice/roboguice) 等任何包含第三方的框架。 Mosby 的船只支持库片段（Fragments from support library）。 如上所述，您可以使用 “native” android.app.Fragment。

委托的另一个优点是可以通过实现一个自定义委托来更改 Mosby 的默认行为。 例如：Mosby 在方向更改期间如何处理 Presenter 的默认实现是重新创建演示者并重新启动请求（除了保留演示者存活的碎片之外）。 您可以编写另一个 ActivityMvpDelegate 或 FragmentMvpDelegate，在内部使用 HashMap <Integer，MvpPresenter> 来存储已存在的 Presenter，并在方向更改（而不是创建新的和重新启动请求）后重新创建 View 时重新使用它：

```java
public interface IdBasedMvpView extends MvpView {
  public void setId(int id);
  public int getId();
}
```

```java
public class IdBasedActivityMvpDelegate<V extends IdBasedMvpView, P extends MvpPresenter<V>>
    implements ActivityMvpDelegate {

  // Every view gets an unique id
  private static int lastViewId = 0;

  // Map to store presenter
  private static Map<Integer, MvpPresenter> presenterMap = new HashMap<>();

  // The reference to the Activity
  private MvpDelegateCallback<V, P>  delegateCallback;


  public ActivityMvpDelegateImpl(MvpDelegateCallback<V, P> delegateCallback) {
    this.delegateCallback = delegateCallback;
  }


  // Called from Activity.onCreate()
  @Override
  public void onCreate(Bundle bundle) {

    // Returns 0 if no view id is assigned
    int viewId = bundle.getInt("MvpViewId");

    MvpPresenter presenter = presenterMap.get(viewId);

    if (presenter == null){
      // No presenter in Map --> View starting first time
      // so create a new presenter and put it into presenterMap
      presenter = delegateCallback.createPresenter();

      viewId = ++lastViewId;
      presenterMap.put(viewId, presenter);
    }

    V view = delegateCallback.getMvpView();

    view.setViewId(viewId);
    view.setPresenter(presenter);
    presenter.attachView(view);
  }

  @Override
  public void onSaveInstanceState(Bundle outState) {
    int viewId = delegateCallback.getMvpView().getViewId();
    outState.putInt("MvpViewId", viewId);
  }

  @Override
  public void onDestroy() {
    delegateCallback.getPresenter().detachView(true); // true == presenter retains instance state
  }
}
```

```java
public MyActivity extends MvpActivity implements IdBasedMvpView {

  ActivityMvpDelegate mvpDelegate;

  @Override
  protected ActivityMvpDelegate<V, P> getMvpDelegate() {
    if (mvpDelegate == null) {
      mvpDelegate = new IdBasedActivityMvpDelegate(this);
    }

    return mvpDelegate;
  }
}
```

我们为每个视图分配一个唯一的 ID。有了这个 ID，我们可以在演示者映射中找到演示者。如果没有演示者可用，那么我们创建一个新的演示者并将其放到演示者地图中。视图的唯一标识存储在视图包中。这样你就可以保留活动和非保留碎片的主持人。那么为什么这不是 Mosby 的默认实现呢？这种方法的问题是内存管理。演示者什么时候从这张地图中删除？如果 Presenter 永远不会被删除会造成内存泄漏？通常演示者是轻量级的对象，但大多数时候他们是一个异步运行线程的回调。为了避免内存泄漏，这不是 Mosby 的默认策略。你也可以实现一个 ActivityMvpDelegate，它将 Presenters 状态存储在一个 Bundle 中（比如 Mortar 和匕首范围）。所以你看：莫斯比提供了一个灵活的脚手架和一个默认的实现，匹配大多数场景。但是，Mosby 可以根据边缘情况进行定制。

## Tip 23: Tips are good - experience is better

每当 Barney Stinson 在午夜离开 MacLaren 的酒吧时，脸上露出一个大大的笑容，旁边还有一个美丽的女人，他知道自己没有成功，因为剧本。 其实他成功了，因为他有很多与女人交谈的经验（穿西装！）。 当然，一些提示和建议是有帮助的，但最后每个人都必须做出自己的经验（包括失败）。 编写应用程序时也是如此：在开发过程中学习，随着每个开发了经验级别的应用程序的增加，您的下一个应用程序将从您的体验中受益。 从这个意义上说：快乐黑客！

我非常感谢 [Kirill Boyarshinov](https://twitter.com/naghtarr) 帮助我。 示例邮件应用程序的源代码可以在 [Github](https://github.com/sockeqwe/mosby/tree/master/sample-mail) 上找到。如果您有任何问题，请不要犹豫，留下评论。





