# 项目和代码-指南

> 原文 (github.com/ribot)：[project_and_code_guidelines](https://github.com/ribot/android-guidelines/blob/master/project_and_code_guidelines.md)
>
> 作者：[ribot](https://github.com/ribot)

[TOC]

# 1. 工程项目指引 

## 1.1 工程项目结构 

新的项目应该遵循安卓 Gradle 项目结构，这是在[安卓 Gradle 插件用户指南](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Project-Structure)中定义的。 [Ribot Boilerplate](https://github.com/ribot/android-boilerplate) 项目是一个很好的开始参考。 

## 1.2 文件命名 

### 1.2.1 类文件 

类名写在 [UpperCamelCase](http://en.wikipedia.org/wiki/CamelCase) 中 .

对于扩展安卓组件的类，类的名称应该以组件的名称结束; 例如: SignInActivity，SignInFragment，ImageUploaderService，ChangePasswordDialog。 

### 1.2.2 资源文件

资源文件名是用小写下划线写的。 

#### 1.2.2.1 Drawable 文件

drawables 的命名约定 :

| 资产类别     | 前缀            | 例子                       |
| ------------ | --------------- | -------------------------- |
| Action bar   | `ab_`           | `ab_stacked.9.png`         |
| Button       | `btn_`          | `btn_send_pressed.9.png`   |
| Dialog       | `dialog_`       | `dialog_top.9.png`         |
| Divider      | `divider_`      | `divider_horizontal.9.png` |
| Icon         | `ic_`           | `ic_star.png`              |
| Menu         | `menu_`         | `menu_submenu_bg.9.png`    |
| Notification | `notification_` | `notification_bg.9.png`    |
| Tabs         | `tab_`          | `tab_pressed.9.png`        |

图标命名约定(摘自[安卓图标指南](http://developer.android.com/design/style/iconography.html)) : 

| 资产类别                        | 前缀             | 例子                       |
| ------------------------------- | ---------------- | -------------------------- |
| Icons                           | `ic_`            | `ic_star.png`              |
| Launcher icons                  | `ic_launcher`    | `ic_launcher_calendar.png` |
| Menu icons and Action Bar icons | `ic_menu`        | `ic_menu_archive.png`      |
| Status bar icons                | `ic_stat_notify` | `ic_stat_notify_msg.png`   |
| Tab icons                       | `ic_tab`         | `ic_tab_recent.png`        |
| Dialog icons                    | `ic_dialog`      | `ic_dialog_info.png`       |

selector 命名约定如下 : 

| State    | Suffix      | Example                    |
| -------- | ----------- | -------------------------- |
| Normal   | `_normal`   | `btn_order_normal.9.png`   |
| Pressed  | `_pressed`  | `btn_order_pressed.9.png`  |
| Focused  | `_focused`  | `btn_order_focused.9.png`  |
| Disabled | `_disabled` | `btn_order_disabled.9.png` |
| Selected | `_selected` | `btn_order_selected.9.png` |

#### 1.2.2.2 布局文件 

布局文件应该与他们要使用的安卓组件的名称相匹配，但要将顶级组件名称移动到开始。 例如，如果我们正在为 SignInActivity 创建一个布局，那么布局文件的名称应该是 activity_sign_in.xml  。 

| 组件             | 类名称                 | 布局名称                     |
| ---------------- | ---------------------- | ---------------------------- |
| Activity         | `UserProfileActivity`  | `activity_user_profile.xml`  |
| Fragment         | `SignUpFragment`       | `fragment_sign_up.xml`       |
| Dialog           | `ChangePasswordDialog` | `dialog_change_password.xml` |
| AdapterView item | ---                    | `item_person.xml`            |
| Partial layout   | ---                    | `partial_stats_bar.xml`      |

一个稍微不同的情况是，当我们创建一个布局，这个布局将被一个 Adapter inflated  ，例如填充一个 ListView。 在这种情况下，布局的名称应该以 item_ 开始。 

请注意，在有些情况下，这些规则不可能适用。 例如，当创建布局文件时，打算成为其他布局的一部分。 在这种情况下，你应该使用前缀  partial_。 

#### 1.2.2.3 Menu 文件

类似于布局文件，菜单文件应该匹配组件的名称。 例如，如果我们定义的菜单文件将用于 UserActivity，那么该文件的名称应该是 activity_user.xml 。

一个好的做法是不要将 menu 一词作为名称的一部分，因为这些文件已经位于 Menu 目录中。 

#### 1.2.2.4 Values 文件

值文件夹中的资源文件应该是复数形式，例如 strings.xml、 styles.xml、 colors.xml、 dimens.xml 。

# 2 代码指南 

## 2.1  Java 语言规则 

### 2.1.1 不要忽视异常 

你绝对不能做以下事情: 

```java
void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) { }
}
```

虽然你可能认为你的代码永远不会遇到这种错误条件，或者无需处理它，但忽略上面这些异常会在代码中创建一些地雷，让其他人有一天会被绊倒。 你必须以某种原则性的方式处理代码中的每个异常。 具体的处理方式取决于具体情况。 - ([Android 代码样式指引](https://source.android.com/source/code-style.html)) 

在这里可以看到其他选择。 [[here]](https://source.android.com/source/code-style.html#dont-ignore-exceptions)

### 2.1.2 不要捕捉通用异常 

你不应该这样做 : 

```java
try {
    someComplicatedIOFunction();        // may throw IOException
    someComplicatedParsingFunction();   // may throw ParsingException
    someComplicatedSecurityFunction();  // may throw SecurityException
    // phew, made it all the way
} catch (Exception e) {                 // I'll just catch all exceptions
    handleError();                      // with one generic handler!
}
```

看看为什么，还有其他的选择。 [[here]](https://source.android.com/source/code-style.html#dont-catch-generic-exception)

### 2.1.3 不要使用终结器 

我们不使用终结器(finalizers)。因为不能保证什么时候会调用终结器，甚至根本不会被调用。 在大多数情况下，你可以从具有良好异常处理的终结器中完成你所需要的事情。 如果你绝对需要它，请定义一个close( ) 方法(或类似的) ，并准确地记录什么时候需要调用该方法。 看看 InputStream 就是一个例子。 在这种情况下，只要预期不会淹没日志，那么从终结器打印一个简短的日志消息是适当的，但不是必须的。 - ([Android 代码样式指引](https://source.android.com/source/code-style.html#dont-use-finalizers)) 

### 2.1.4 完全符合资格的 imports

糟糕的 : `import foo.;`

好的 : import foo.Bar;

请点击这里查看更多信息。 [[here]](https://source.android.com/source/code-style.html#fully-qualify-imports)

## 2.2 样式规则 

### 2.2.1 字段的定义和命名 

字段应该在文件的顶部定义，它们应该遵循下面列出的命名规则。 

- 私有的、非静态的字段名称以 m 开头。 
- 私有的，静态字段名称以 s 开头。
- 其他字段以小写字母开头。
- 静态最终字段(常数)是带有下划线的所有大写。

例子: 

```java
public class MyClass {
    public static final int SOME_CONSTANT = 42;
    public int publicField;
    private static MyClass sSingleton;
    int mPackagePrivate;
    private int mPrivate;
    protected int mProtected;
}
```

### 2.2.3 把首字母缩略词当作单词来对待 

| 很好             | 糟糕             |
| ---------------- | ---------------- |
| `XmlHttpRequest` | `XMLHTTPRequest` |
| `getCustomerId`  | `getCustomerID`  |
| `String url`     | `String URL`     |
| `long id`        | `long ID`        |

### 2.2.4 使用空格进行缩进 

使用4个空间缩进用于块: 

```java
if (x == 1) {
    x++;
}
```

使用8个空间缩进用于行包装: 

```java
Instrument i =
        someLongExpression(that, wouldNotFit, on, one, line);
```

### 2.2.5 使用标准的支架样式 

括号与前面的代码在同一行上。 

```java
class MyClass {
    int func() {
        if (something) {
            // ...
        } else if (somethingElse) {
            // ...
        } else {
            // ...
        }
    }
}
```

除非条件和身体匹配一行，否则需要在语句周围加上括号。 

如果条件和身体适合于一行，而且该行比最大行长短，则不需要括号，例如。 

```java
if (condition) body();
```

这很糟糕: 

```java
if (condition)
    body();  // bad!
```

### 2.2.6 注释 

#### 2.2.6.1 注释练习

根据 Android 代码样式指南，Java 中一些预定义注释的标准实践是: 

- `@Override`: 当方法从超级类中重写声明或实现时，必须使用 @override 注释。 例如，如果你使用 @inheritdocs Javadoc 标记，并从类(而不是接口)派生，则必须注释方法 @overrides 父类的方法。 
- `@SuppressWarnings`: 注释只能在无法消除警告的情况下使用。 如果一个警告通过了这个"不可能消除"的测试，则必须使用 @suppresswarnings 注释，以确保所有警告都反映代码中的实际问题。 

有关注释指南的更多信息可以在这里找到。 [[here]](http://source.android.com/source/code-style.html#use-standard-java-annotations)

#### 2.2.6.2 注释格式

类、方法和构造函数 

当对类、方法或构造函数应用注释时，它们会在文档块之后列出，并且应该以每行的一个注释的形式出现。 

```java
/ This is the documentation block about the class /
@AnnotationA
@AnnotationB
public class MyAnnotatedClass { }
```

字段

应在同一行上列出应用于字段的注释，除非该行符合最大行长。 

```java
@Nullable @Mock DataManager mDataManager;
```

### 2.2.7 限制变量范围 

局部变量的范围应保持在最小值(Effective  Java 项目29)。 通过这样做，你可以提高代码的可读性和可维护性，并减少出错的可能性。 每个变量应该在最内部的块中声明，该块包含变量的所有用法。 

应该在第一次使用的时候声明局部变量。 几乎每个局部变量声明都应该包含一个初始化器。 如果你还没有足够的信息来明智地初始化变量，那么你应该推迟声明，直到这样做。 - ([Android 代码样式指引](https://source.android.com/source/code-style.html#limit-variable-scope)) 

### 2.2.8 import 顺序 

如果你使用的是 Android Studio 这样的 IDE，你不必担心这个，因为你的 IDE 已经在遵守这些规则了。 如果没有，看看下面。 

import 顺序如下 

1. Android imports
2. 第三方 import (com, junit, net, org)
3. java 和 javax
4. 同一项目 imports

为了与 IDE 设置完全匹配，导入应该是: 

- 在每个分组内按字母顺序排列，在小写字母前加上大写字母(例如: z 前 a) 。
- 每个主要分组 (android, com, junit, net, org, java, javax) 之间应该有一个空行。 

更多信息请点击这里。 [[here]](https://source.android.com/source/code-style.html#limit-variable-scope) 

### 2.2.9 日志指引 

使用日志类提供的日志方法打印出可能有助于开发人员识别问题的错误消息或其他信息: 

- `Log.v(String tag, String msg)` (verbose)
- `Log.d(String tag, String msg)` (debug)
- `Log.i(String tag, String msg)` (information)
- `Log.w(String tag, String msg)` (warning)
- `Log.e(String tag, String msg)` (error)

作为一般规则，我们使用类名作为标记，并将其定义为文件顶部的静态最终字段。 例如: 

```java
public class MyClass {
    private static final String TAG = MyClass.class.getSimpleName();

    public myMethod() {
        Log.e(TAG, "My error message");
    }
}
```

在 release  版本生成时必须禁用 VERBOSE 和 DEBUG 日志。 还建议禁用 INFORMATION, WARNING 和 ERROR  ，但如果你认为这些日志可能有助于识别发布版本中的问题，那么你可能希望保持这些日志的启用。 如果你决定启用它们，你必须确保它们不会泄露诸如电子邮件地址、用户 id 等等的私人信息。 

只显示调试构建中的日志: 

```java
if (BuildConfig.DEBUG) Log.d(TAG, "The value of x is " + x);
```

### 2.2.10 类成员顺序

没有一个单一的正确解决方案，但使用一个逻辑和一致的顺序将提高代码的可学习性和可读性。 建议使用下列顺序: 

1. 常数 
2. 字段
3. 构造函数 
4. 覆盖方法和回调(公开或私有) 
5. 公共方法 
6. 私人方法 
7. 内部类或接口 

例子: 

```java
public class MainActivity extends Activity {

    private static final String TAG = MainActivity.class.getSimpleName();

    private String mTitle;
    private TextView mTextViewTitle;

    @Override
    public void onCreate() {
        ...
    }

    public void setTitle(String title) {
    	mTitle = title;
    }

    private void setUpView() {
        ...
    }

    static class AnInnerClass {

    }

}
```

如果你的类正在扩展一个 Android 组件，例如活动或片段，那么按顺序重写方法，使它们与组件的生命周期匹配是一个很好的实践。 例如，如果你有一个实现 create ( )、 onDestroy ( )、 onPause ( ) 和 onResume ( ) 的活动，那么正确的顺序是: 

```java
public class MainActivity extends Activity {

	//Order matches Activity lifecycle
    @Override
    public void onCreate() {}

    @Override
    public void onResume() {}

    @Override
    public void onPause() {}

    @Override
    public void onDestroy() {}

}
```

### 2.2.11 方法中的参数排序 

当为 Android 编程时，定义使用上下文的方法是很常见的。 如果你正在编写这样的方法，那么上下文必须是第一个参数。 

相反的情况是回调接口，它应该是最后一个参数。 

例子: 

```java
// Context always goes first
public User loadUser(Context context, int userId);

// Callbacks always go last
public void loadUserAsync(Context context, int userId, UserCallback callback);
```

### 2.2.13 字符串常数、命名和值 

安卓 SDK 的许多元素，如 SharedPreferences，Bundle，或 Intent 使用一个 key-value pair 方法，所以很可能即使对于一个小型应用程序，你最终也很可能不得不编写大量的 String 常量。 

在使用其中一个组件时，必须将键定义为静态的最终字段，并且它们应该如下面所示的那样预固定。 

| 元素               | 字段名称前缀 |
| ------------------ | ------------ |
| SharedPreferences  | `PREF_`      |
| Bundle             | `BUNDLE_`    |
| Fragment Arguments | `ARGUMENT_`  |
| Intent Extra       | `EXTRA_`     |
| Intent Action      | `ACTION_`    |

请注意，Fragment.getarguments ( ) 的参数也是一个 Bundle。 然而，由于这是一个相当常见的使用 Bundle ，我们为它们定义了一个不同的前缀。 

例子: 

```java
// Note the value of the field is the same as the name to avoid duplication issues
static final String PREF_EMAIL = "PREF_EMAIL";
static final String BUNDLE_AGE = "BUNDLE_AGE";
static final String ARGUMENT_USER_ID = "ARGUMENT_USER_ID";

// Intent-related items use full package name as value
static final String EXTRA_SURNAME = "com.myapp.extras.EXTRA_SURNAME";
static final String ACTION_OPEN_USER = "com.myapp.action.ACTION_OPEN_USER";
```

### 2.2.14 片段和活动中的参数

当数据通过 Intent 或 Bundle  传递到活动或片段时，不同值的键必须遵循上一节中描述的规则。 

当一个活动或片段期望参数时，它应该提供一个公共静态方法，以便于创建相关的意图或片段。 

在活动的情况下，这个方法通常被称为 getStartIntent ( ) : 

```java
public static Intent getStartIntent(Context context, User user) {
	Intent intent = new Intent(context, ThisActivity.class);
	intent.putParcelableExtra(EXTRA_USER, user);
	return intent;
}
```

对于片段，它被命名为 newInstance ( ) ，并且使用正确的参数来处理片段的创建: 

```java
public static UserFragment newInstance(User user) {
	UserFragment fragment = new UserFragment();
	Bundle args = new Bundle();
	args.putParcelable(ARGUMENT_USER, user);
	fragment.setArguments(args)
	return fragment;
}
```

注1: 这些方法应该在 onCreate ( ) 之前在类的顶部进行。 

注2: 如果我们提供上面描述的方法，额外和参数的密钥应该是私有的，因为它们不需要暴露在类之外。 

### 2.2.15 行长限制 

代码行不应超过100个字符。 如果行长超过这个限度，通常有两个选项可以缩短它的长度: 

-  提取本地变量或方法(更好)。 
- 应用行包装法将一行划分为多个行。

有两种异常情况，即可能有超过100条的线路: 

- 不可分割的行，例如注释中的长 url 。
- package 和 import 声明.

#### 2.2.15.1 行包装策略 

没有一个精确的公式来解释如何进行包装和相当多的不同的解决方案是有效的。 然而，有一些规则可以适用于常见的情况。 

操作符断点

当操作符在行中断时，断点在操作符之前。 例如: 

```java
int longName = anotherVeryLongVariable + anEvenLongerOne - thisRidiculousLongOne
        + theFinalOne;
```

分配操作符异常 

运算符规则中断的一个例外是赋值运算符，在操作符之后应该发生行中断。 

```java
int longName =
        anotherVeryLongVariable + anEvenLongerOne - thisRidiculousLongOne + theFinalOne;
```

方法链案例 

当多个方法被链接在同一行中——例如在使用 Builders 时——每次调用方法都应该在它自己的行中，在 ` . ` 之前。

```java
Picasso.with(context).load("http://ribot.co.uk/images/sexyjoe.jpg").into(imageView);
```

```java
Picasso.with(context)
        .load("http://ribot.co.uk/images/sexyjoe.jpg")
        .into(imageView);
```

长参数案例 

当一个方法有很多参数或参数很长时，我们应该在每个逗号之后打破这条线。

```java
loadPicture(context, "http://ribot.co.uk/images/sexyjoe.jpg", mImageViewProfilePicture, clickListener, "Title of the picture");
```

```java
loadPicture(context,
        "http://ribot.co.uk/images/sexyjoe.jpg",
        mImageViewProfilePicture,
        clickListener,
        "Title of the picture");
```

### 2.2.16 RxJava 链式样式 

操作符的 Rx 链需要行包装。 每个操作符都必须在一条新的行上运行，并且在。 

```java
public Observable<Location> syncLocations() {
    return mDatabaseHelper.getAllLocations()
            .concatMap(new Func1<Location, Observable<? extends Location>>() {
                @Override
                 public Observable<? extends Location> call(Location location) {
                     return mRetrofitService.getLocation(location.id);
                 }
            })
            .retry(new Func2<Integer, Throwable, Boolean>() {
                 @Override
                 public Boolean call(Integer numRetries, Throwable throwable) {
                     return throwable instanceof RetrofitError;
                 }
            });
}
```

## 2.3  XML 样式规则 

### 2.3.1 使用自闭标签 

当 XML 元素没有任何内容时，必须使用自闭标记。 

好的： 

```java
<TextView
	android:id="@+id/text_view_profile"
	android:layout_width="wrap_content"
	android:layout_height="wrap_content" />
```

糟糕的 :

```java
<!-- Don\'t do this! -->
<TextView
    android:id="@+id/text_view_profile"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" >
</TextView>
```

### 2.3.2 资源命名 

资源 id 和名称是用小写下划线写的。 

#### 2.3.2.1  ID 命名 

Id 应该以小写下划线中元素的名称为前缀。 例如: 

| 元素        | 前缀      |
| ----------- | --------- |
| `TextView`  | `text_`   |
| `ImageView` | `image_`  |
| `Button`    | `button_` |
| `Menu`      | `menu_`   |

图片视图示例: 

```xml
<ImageView
    android:id="@+id/image_profile"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />
```

菜单示例: 

```xml
<menu>
	<item
        android:id="@+id/menu_done"
        android:title="Done" />
</menu>
```

#### 2.3.2.2 Strings

字符串名称以识别它们所属部分的前缀开始。 例如 registration_email_hint 或 registration_name_hint 。 如果一个字符串不属于任何部分，那么你应该遵循以下规则: 

| 前缀      | 描述                     |
| --------- | ------------------------ |
| `error_`  | 错误信息                 |
| `msg_`    | 一个常规的信息信         |
| `title_`  | 标题，即对话框标题       |
| `action_` | "保存"或"创建"之类的行动 |

#### 2.3.2.3 风格和主题 

与其他资源不同，样式名是用 UpperCamelCase 编写的。 

### 2.3.3 属性顺序

作为一般规则，你应该尝试将类似的属性组合在一起。 排序最常见属性的一个好方法是: 

1. 视图 id 。
2. 风格 。
3. 布局宽度和布局高度 。
4. 其他布局属性，按字母顺序排序 。
5. 剩余属性，按字母顺序排序 。

## 2.4 测试样式规则 

### 2.4.1 单元测试 

测试类应该与测试所针对的类的名称匹配，然后是 Test。 例如，如果我们创建一个包含 DatabaseHelper 测试的测试类，我们应该把它命名为 DatabaseHelperTest。 

测试方法用 @Test 进行注释，通常应该从正在测试的方法的名称开始，然后是一个先决条件和 / 或预期的行为。 

- Template: `@Test void methodNamePreconditionExpectedBehaviour()`
- Example: `@Test void signInWithEmptyEmailFails()`

如果没有这些先决条件和 / 或预期行为足够清楚，则不一定需要先决条件和 / 或预期行为。 

有时一个类可能包含大量的方法，同时需要对每个方法进行多个测试。 在这种情况下，可以推荐将测试类分成多个类。 例如，如果 DataManager 包含许多方法，我们可以将它分成 DataManagerSignInTest，DataManagerLoadUsersTest 等。 一般来说，你可以看到哪些测试是一起的，因为它们有共同的测试设备。 

### 2.4.2 Espresso 测试 

每一个 Espresso 测试类通常针对一个活动，因此名称应该与目标活动的名称相匹配，然后再进行测试，例如 SignInActivityTest 。

当使用 Espresso API 时，常用的做法是将链式方法放在新的行上。 

```java
onView(withId(R.id.view))
        .perform(scrollTo())
        .check(matches(isDisplayed()))
```

