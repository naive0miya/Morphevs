# MVP 中的演示模型和被动视图ーーAndroid 的方式

> 原文 (Medium)：[Presentation Model and Passive View in MVP — The Android way](https://android.jlelse.eu/presentation-model-and-passive-view-in-mvp-the-android-way-fdba56a35b1e)
>
> 作者：[Andrzej Chmielewski](https://android.jlelse.eu/@andrzejchm?source=post_header_lockup)

[TOC]

每次我参与一个项目，在某种程度上融合了 MVP 模式时，我总是会把问题放在什么地方。众所周知，一个优秀的程序员能够提出好的[关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns)，所以代码是可读的，任何人都可以理解。 

> 我应该把它放在演示者还是活动中？ 我应该在哪里处理来自我们的 REST API 的这些数据？

（顺便说一下 ，请务必查看[关于模拟 REST API 响应的文章](https://medium.com/@andrzejchm/ittd-instrumentation-ttd-for-android-4894cbb82d37)）

这些都是我经常问自己的最常见的问题。  最后，我并没有摆脱[上帝的活动对象](https://en.wikipedia.org/wiki/God_object)，大多数时候我只是把所有的上帝都转移到演示者上。 这里唯一的好处是现在用单元测试而不是用 UI 测试，更容易测试，但仍然难以阅读和维护。 我试图找出解决这个问题的方法，最后我想到了几种模式，比如[被动视图](http://martinfowler.com/eaaDev/PassiveScreen.html)、[演示模型](http://martinfowler.com/eaaDev/PresentationModel.html)和 [MVP](http://martinfowler.com/eaaDev/uiArchs.html#Model-view-presentermvp)。 

这就是 [DroidMVP](https://github.com/andrzejchm/DroidMVP) 库诞生的原因，应该能够更容易地将应用程序的某些责任分离出来，并使得维护整个代码变得更加容易。 那么它是如何运作的呢？ 

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frovb714cgj30ii07m74k.jpg)

## View — HOW

活动，片段，一个自定义视图ーー任何符合你需要的东西。 该视图的主要责任是向用户显示某种状态。 一个进度条，一个文本，一个显示成功操作的对话框。所有对用户有意义的事情。 视图应该知道如何显示某些东西，而不是什么时候显示，所以基本上你的活动或片段(或者任何你想出来的东西)应该被当作一个由安卓系统(如 TextView 或 ImageView)组成的元部件(如 TextView 或 ImageView) ，该接口会公开某些操作，比如: 

```java
showProductsList(List<Product> products);
showProgress();
showUpdateCompleted();
```

> 你的视图只是一组帮助你操作它的状态的操作 

记住，决定它是否应该转移到某种状态并不是视图的责任。 举个例子，如果用户点击一个按钮，这个按钮会触发 REST API 调用，不要这样做: 

```java
public class BadActivity extends DroidMVPActivity<GoodPresentationModel, GoodView, GoodPresenter> {
    ...
    
    @OnClick(R.id.button) public void onButtonClicked() {
        showProgress();
        presenter.onButtonClicked();
    }
    
    ...
}
```

> 永远不要让你的视图决定转移状态。相反，让演示者做这项工作。

相反，让你的演示者决定进度条是否应该显示，所以它看起来更像这样: 

```java
public class GoodActivity extends DroidMVPActivity<GoodPresentationModel, GoodView, GoodPresenter> {
    ...
    
    @OnClick(R.id.button) public void onButtonClicked() {
        presenter.onButtonClicked();
    }
    
    ...
}

--
public class GoodPresenter extends SimpleDroidMVPPresenter {
    ...
    
    public void onButtonClicked() {
      getMvpView().showProgress();
      // request data from API
    }
    
    ...
}
```

> 一个很好的关注点分离：View 知道 HOW，Presenter 知道 WHEN

好处：通过使视图尽可能愚蠢（这就是被动视图模式通常所说的），将所有逻辑转移到与 Android 无关的类上，因此你可以省略 UI 测试以及测试你的视图是最小的（喜欢单元测试优于 UI 测试）。

## Presenter — WHEN

主持人本身就是我们视图的控制者。 它改变了视图的状态，并处理了所有的用户交互，把硬件工作委托给域层(比如 REST API 或 usecase 类)。 演示者知道何时显示某些状态并触发演示模型更新，同时将所有工作委派给域层。 例如: 当用户点击一个按钮时，演示者告诉视图显示进度条，请求域层进行数据 / 更新，将结果存储在演示模型中，并将视图状态与在演示模型中转换的数据更新，如下所示: 

```java
public class GoodPresenter extends SimpleDroidMVPPresenter<GoodView,GoodPresentationModel> {

    public void onShowUsersButtonClicked() {
        getMvpView().showProgress();
        repository.getUsers(new Callback() {
            public void onSuccess(List<Users> users) {
                usersListFetched(users);
            }

            public void onFailure(Exception e) {
                usersListFetchError(e);
            }
        })
    }

    private void usersListFetched(List<Users> users) {
        PresentationModel model = getPresentationModel();
        model.setUsers(users);
        if (getMvpView() == null) {
            return;
        }
        if (model.shouldDisplayEmptyState()) {
            getMvpView().showEmptyState();
        } else {
            getMvpView().showUsersList(model.getAllUsers());
        }
        if(model.hasAnyUserBirthdayToday()) {
            getMvpView().showBirthdayAlert(model.getUsersWithBirthday());
        }
    }

    private void usersListFetchError(Exception e) {
        getPresentationModel().setUsersListFetchError(e);
        if (getMvpView() != null) {
            getMvpView().showUsersListFetchError();
        }
    }
}
```

如你所见，演示者负责控制视图状态，同时将所有业务逻辑委托给域层和视图逻辑(WHAT 部分)到演示模型。 

好处: 通过将你所有的业务逻辑和演示逻辑从演示者身上移开，你的演示者会变得更清晰，更容易阅读，当你阅读代码的时候，你可以专注于什么事情应该发生，而不需要关心应该如何显示和应该显示什么。 

## Presentation Model — WHAT

你的应用程序中最重要的部分之一就是演示模型。 它负责保持你视图中用户可见的当前状态，并将你的域模型转换为演示模型。 演示模型知道应该向用户显示什么，这是由演示者触发的模型更新决定的。 例如，当从 API 中获取用户列表时，演示者将用户列表设置在演示模型中，并根据该数据，模型知道是否应该显示用户生日的警报，以及哪些用户受到了"影响"。 这个逻辑被封装在 Model 的公共方法中: 

```java
public class GoodPresentationModel implements Serializable {

    public static final Birthday TODAY = new Birthday();
    private List<User> users = Collections.emptyList();

    public void setUsers(List<User> users) {
        this.users = users;
    }

    public boolean shouldDisplayEmptyState() {
        return users.isEmpty();
    }

    public List<User> getAllUsers() {
        return Collections.unmodifiableList(users);
    }

    public List<User> hasAnyUserBirthdayToday() {
        for (User user : users) {
            if (isBirthdayToday(user.getBirthday())) {
                return true;
            }
        }
        return false;
    }

    public List<User> getUsersWithBirthday() {
        List<Users> result = new LinkedList<>();
        for (User user : users) {
            if (isBirthdayToday(user.getBirthday())) {
                result.add(user);
            }
        }
        return Collections.unmodifiableList(result);
    }

    private boolean isBirthdayToday(Birthday birthday) {
        return birthday.getMonth() == TODAY.getMonth() && birthday.getDay() == TODAY.getDay();
    }
}
```

主要特点：

- 试着使模型中存储的数据是不可变的。 唯一负责改变数据的实体是模型本身，所以通过使它不可变，你不会激发任何使用代码的人在模型之外进行改变。
- 使你的演示模型可序列化，因为它将被保存和恢复与你的活动和片段(编辑: 你的演示模型现在实现Parcelable ) 。 
- 尽可能让你的模特的测试覆盖率保持在100% 左右。 因为它不包含任何针对 android 的东西，所以单元测试应该很容易。 记住，先测试比调试要容易得多 。

好处: 你的视图逻辑与 Android 框架分离，并且自己进行演示，所以很容易发现潜在的错误，并使你的测试易于编写。 

## DroidMVP

上面解释的所有概念很容易用 [DroidMVP](https://github.com/andrzejchm/DroidMVP) 的小库来完成，你可以在 Github 找到它。 它包括活动 / 片段的基类(或者任何由 [DroidMVPViewDelegate](https://github.com/andrzejchm/DroidMVP/blob/develop/library/src/main/java/io/appflate/droidmvp/DroidMVPViewDelegate.java) 和 Presenter 组成的 MVP 视图的任何角色。 

依赖注入友好 

如果你愿意的话，你可以通过提供你自己的扩展 DroidMVPFragment 或者 DroidMVPActivity 的基类来很容易地将 Dagger 依赖注入与库合并在一起：

```java
public abstract class BaseActivity<M extends Serializable, V extends DroidMVPView, P extends DroidMVPPresenter<V, M>>
    extends DroidMVPActivity<M, V, P> {
    @Inject protected P presenter;

    @NonNull @Override protected P createPresenter() {
        //this field will be populated by field injeciton from dagger
        // your presenter should not accept the presentationModel as its constructor's paramteter.
        // Instead, it will be provided to your presenter in #attachView method.
        return presenter;
    }
}
```

> BaseActivity 使演示者注入 Dagger 更容易

## 以现代的方式测试 REST API

ITDD — Instrumentation TDD for Android using RESTMock Some time ago I had an opportunity to work on a project that utilized a REST API in Android app (No big deal, right?)… [medium.com](https://medium.com/@andrzejchm/ittd-instrumentation-ttd-for-android-4894cbb82d37)

