# 另一篇 MVP 文章 - 第5部分：使用 Dagger 和 Espresso 混合测试

>原文 (Medium)：[Yet another MVP article — Part 5: Writing Test using a mixture of Dagger and Espresso](https://hackernoon.com/yet-another-mvp-article-part-5-writing-test-using-a-mixture-of-dagger-and-espresso-15c638182706)
>
>作者： [Mohsen Mirhoseini](https://hackernoon.com/@mirhoseini)

[TOC]

## 测试，测试和测试...为什么？

当你在做一个应用程序或者任何软件项目的时候，通常产品的所有者会带来新的想法和特性，这使得项目更加闪亮！ 所以你开始发展每个人都开心的方式，也不要错过最后期限，但是天知道你是如何通过添加新的特性代码和改变其他模块来适应你的新代码来打破项目的其他部分。 

你完成你的工作，运行应用程序，检查新功能是否完美，并希望你没有打破任何东西。 你发布一个版本给 QA 来测试，并尽快离开办公室! ! ！ 

**对应的**

你已经花了足够的时间写一些测试，检查应用程序功能的所有部分，并在将更改推入仓库之后使用 CI (持续集成)服务，你将得到一个正在通过或失败的测试的完整报告。 

**拜托，写作测试很难**

我相信，在每次特性开发和代码更改之后，在2到3次发布之后找到 bug 要写一些简单的测试要困难得多。 

**让我们开始测试...**

> Android Studio 旨在使测试变得简单。 只需点击几下鼠标，你就可以设置在本地 JVM上运行的 JUnit 测试或者在设备上运行的仪器测试。 当然，你还可以通过集成测试框架（如 [Mockito](https://github.com/mockito/mockito)）来测试 Android API 调用，并在你的本地单元测试中使用 [ Espresso](https://developer.android.com/topic/libraries/testing-support-library/index.html#Espresso) 或 [UI Automator](https://developer.android.com/topic/libraries/testing-support-library/index.html#UIAutomator) 在你的仪器测试中锻炼用户交互，从而扩展测试功能。 你可以使用 [Espresso 测试记录仪](https://developer.android.com/studio/test/espresso-test-recorder.html)自动生成 Espresso 测试。

Android Studio 支持 2 种不同类型的测试：

- JUnit 测试：这是测试 Java 代码，不需要 Android 设备或 SDK。
- Instrumented Test：检测应用程序功能(通常是用户界面反应) ，大部分时间使用安卓设备 。

回到示例项目，我们有 2 个不同的模块，核心和应用程序。核心模块是纯 Java 所以每个测试只发生在使用 JUnit 测试：
![core module test folder|center](https://ws4.sinaimg.cn/large/006tNc79gy1froo3bgt2yj30b809saab.jpg)

不要忘记，JUnit 测试放在 src 里面的测试文件夹里，在主文件夹旁边。包装与主要源代码相同。 应用程序模块包含 Android 和 Java 代码，所以我们需要 JUnit 和 androidTests：

![@app module test & androidTest folders|center](https://ws3.sinaimg.cn/large/006tNc79gy1froo3fjy9uj30ay0g0wf4.jpg)

## 核心模块 JUnit 测试：

我写了一个测试 “ [SearchInteractor](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/test/java/com/mirhoseini/marvel/character/search/SearchInteractorTest.java) ” 类功能的简单的 JUnit 测试 “ [SearchInteractorTest](https://github.com/mirhoseini/marvel/blob/master/core-lib/src/main/java/com/mirhoseini/marvel/character/search/SearchInteractor.java) ”。让我们来看看：

```java
package com.mirhoseini.marvel.character.search;

/*...*/

import org.junit.Before;
import org.junit.Test;

import java.util.Collections;

import rx.Observable;
import rx.observers.TestSubscriber;
import rx.schedulers.Schedulers;

import static org.mockito.Matchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

public class SearchInteractorTest {

    SearchInteractor interactor;
    MarvelApi api;
    SchedulerProvider scheduler;
    CharactersResponse expectedResult;

    @Before
    public void setup() {
        api = mock(MarvelApi.class);
        scheduler = mock(SchedulerProvider.class);

        // Set up the stub we want to return in the mock
        Results character = new Results();
        character.setName("Test Character");

        Results[] characters = new Results[1];
        characters[0] = character;

        Data charactersData = new Data();
        charactersData.setResults(characters);

        // put the test character in a test api result
        expectedResult = new CharactersResponse();
        expectedResult.setData(charactersData);

        // mock scheduler to run immediately
        when(scheduler.mainThread())
                .thenReturn(Schedulers.immediate());
        when(scheduler.backgroundThread())
                .thenReturn(Schedulers.immediate());

        // mock api result with expected result
        when(api.getCharacters(any(String.class), any(String.class), any(String.class), any(Long.class)))
                .thenReturn(Observable.just(expectedResult));

        // create a real new interactor using mocked api and scheduler
        interactor = new SearchInteractorImpl(api, scheduler);
    }

    @Test
    public void testLoadCharacters() throws Exception {
        TestSubscriber<CharactersResponse> testSubscriber = new TestSubscriber<>();

        // call interactor with some random params
        interactor.loadCharacter("query", "privateKey", "publicKey", 1)
                .subscribe(testSubscriber);

        // it must return the expectedResult with no error
        testSubscriber.assertNoErrors();
        testSubscriber.assertReceivedOnNext(Collections.singletonList(expectedResult));
    }
}
```

首先，请记住，通过将测试类保存在正在测试的类的同一个包中，你可以访问该类和包中的所有内容，而不会使它们公开，以便从其他包中访问。 

其次，在这个示例中，我们使用了 mock，这意味着从一些源代码部分创建一个 Maquette，并以你想要的方式使用它们。 在这个例子中，我们嘲笑了两个主要的类，我们不希望他们以一种正常的方式行动: 

- MarvelApi： 使用这一行代码来处理网络 API 调用: 

```java
api = mock(MarvelApi.class)
```

- SchedulerProvider：它提供了使用这一行代码运行代码的线程: 

```java
scheduler = mock(SchedulerProvider.class)
```

下一步是创建一个我们希望 API 返回的测试结果，这非常简单，这里是 Mockito 库的神奇之处：

```java
when(api.getCharacters(any(String.class), any(String.class), any(String.class), any(Long.class))) .thenReturn(Observable.just(expectedResult));
```

测试通常非常接近英语，所以我们可以这样读：

> jUnit，当代码调用方法 getCharacter 并传递任何参数作为任何参数，THENRETURN 单个 RxJava Observable 包含 expectedResult。
>
> 你在安装方法开始时提到了 @Before 注释吗？ JUnit 测试会在任何使用 @Test 注解的方法之前运行这个方法，然后准备你需要的任何测试。

## 应用程序模块测试：

由于应用程序模块包含 MVP 的视图层，几乎每个测试都是关于 UI 功能的。 本例中我使用了两个不同的 UI 功能测试库
![](https://ws1.sinaimg.cn/large/006tNc79gy1froo3lwj5tj30jc03c74l.jpg)

[**Robolectric:**](http://robolectric.org/)

> Robolectric 是一个单元测试框架，它可以去掉 Android SDK jar，所以你可以测试你的 Android 应用程序的开发。测试在几秒钟内在工作站上的 JVM 内部运行。

它使用 Android 设备或模拟器来运行测试，并得到 Google 的支持。

所有 Android 开发人员都知道 UI 测试总是很麻烦，所以 Google 在上一次 I / O 中引入了 [Espresso 测试记录器](https://developer.android.com/studio/test/espresso-test-recorder.html)，这使得所有的测试和断言都变得简单。

**使用 Robolectric 进行 UI 测试**

```java
package com.mirhoseini.marvel.activity;

/*...*/

@RunWith(RobolectricTestRunner.class)
@Config(constants = BuildConfig.class, sdk = 21, shadows = {ShadowSnackbar.class})
public class MainActivityRobolectricTest {

    private final static String TEST_TEXT = "This is a test text.";
    private MainActivity activity;

    @Before
    public void setUp() throws Exception {
        activity = Robolectric.setupActivity(MainActivity.class);

        assertNotNull(activity);
    }

    @Test
    public void testShowToastMessage() throws Exception {
        activity.showMessage(TEST_TEXT);

        assertThat(TEST_TEXT, equalTo(ShadowToast.getTextOfLatestToast()));
    }

    @Test
    public void testShowOfflineMessage() throws Exception {
        activity.showOfflineMessage();

        assertSnackbarIsShown(R.string.offline_message);
    }
}
```

**这是怎么回事？！**

- @RunWith 注释：告诉 IDE 哪个类将运行这个测试类
- @Config 注解：是 Robolectric 库的一部分，并配置测试环境。
- shadows：正在帮助插入 Robolectric 的类，并提供像 Snackbar 一样的新功能。

在这之后，正如我之前在安装方法中提到的那样，我们准备测试所需要的东西，然后我认为很容易理解的测试方法！ 

不要忘了运行这个测试非常简单，不需要模拟器！它也位于测试文件夹（jUnit 测试位置！）内，这意味着它不是 androidTest。

## 使用 espresso 进行 UI 测试示例：

我相信，使用 MVP 编写代码的所有努力的结果将会在使用 espresso 的 UI 测试中显示出来。

在为应用编写测试时，你不负责测试 API 调用结果，所以没有必要调用网络。 使用一些技巧的 dagger，你将注入任何地方模拟的 API 模块! ! ！ 

让我们稍微解释一下 dagger 的模拟： 首先，看看应用程序 build.gradle 文件，并找到这行代码：

```groovy
/* replaced with custom MockTestRunner */
testInstrumentationRunner "com.mirhoseini.marvel.MarvelTestRunner"
```

这是已经被自定义测试运行器替换的类 “ AndroidJUnitRunner ”。

[MarvelTestRunner](https://github.com/mirhoseini/marvel/blob/master/app/src/androidTest/java/com/mirhoseini/marvel/MarvelTestRunner.java) 里面有什么？

```java
package com.mirhoseini.marvel;

/*...*/

public class MarvelTestRunner extends AndroidJUnitRunner {

    @Override
    public Application newApplication(ClassLoader classLoader, String className, Context context)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        // replace Application class with mock one
        return super.newApplication(classLoader, MarvelTestApplication.class.getName(), context);
    }
  
}
```

这个类的重点是用 [MarvelTestApplication](https://github.com/mirhoseini/marvel/blob/master/app/src/androidTest/java/com/mirhoseini/marvel/MarvelTestApplication.java) 替换 [MarvelApplication](https://github.com/mirhoseini/marvel/blob/master/app/src/main/java/com/mirhoseini/marvel/MarvelApplication.java) 类：

```java
package com.mirhoseini.marvel;

/*...*/

public class MarvelTestApplication extends MarvelApplicationImpl {

    @Override
    public ApplicationTestComponent createComponent() {
        return DaggerApplicationTestComponent
                .builder()
                .androidModule(new AndroidModule(this))
                // replace Api Module with Mocked one
                .apiModule(new ApiTestModule())
                .build();
    }

}
```

这个类扩展了 MarvelApplicationImpl 类，这意味着它包含了里面的任何东西，但是替换了 createComponent 方法并且使用了 ApiTestModule。

这就是 Dagger 在需要时将注入模拟 API 类的方式。

这是 [MainActivityTest ](https://github.com/mirhoseini/marvel/blob/master/app/src/androidTest/java/com/mirhoseini/marvel/activity/MainActivityTest.java)UI 功能测试：

```java
package com.mirhoseini.marvel.activity;

/*...*/

@RunWith(AndroidJUnit4.class)
public class MainActivityTest {

    private static final String TEST_CHARACTER_NAME = "Test Name";
    private static final String TEST_CHARACTER_DESCRIPTION = "Test Description";
    private static final String TEST_CHARACTER_THUMBNAIL_PATH = "Test Thumbnail";
    private static final String TEST_CHARACTER_THUMBNAIL_EXTENSION = "Test Extension";

    @Rule
    public ActivityTestRule<MainActivity> mainActivity = new ActivityTestRule<>(
            MainActivity.class,
            true,
            // false: do not launch the activity immediately
            false);

    @Inject
    MarvelApi api;

    CharactersResponse expectedResult;

    @Before
    public void setUp() {
        Instrumentation instrumentation = InstrumentationRegistry.getInstrumentation();
        MarvelTestApplication app = (MarvelTestApplication) instrumentation.getTargetContext().getApplicationContext();
        ApplicationTestComponent component = (ApplicationTestComponent) MarvelApplication.getComponent();

        // inject dagger
        component.inject(this);

        // Set up the stub we want to return in the mock
        Results character = new Results();
        character.setName(TEST_CHARACTER_NAME);
        character.setDescription(TEST_CHARACTER_DESCRIPTION);
        character.setThumbnail(new Thumbnail(TEST_CHARACTER_THUMBNAIL_PATH, TEST_CHARACTER_THUMBNAIL_EXTENSION));

        Results[] characters = new Results[1];
        characters[0] = character;

        Data charactersData = new Data();
        charactersData.setCount(1);
        charactersData.setResults(characters);

        // put the test character in a test api result
        expectedResult = new CharactersResponse();
        expectedResult.setData(charactersData);
        expectedResult.setCode(Constants.CODE_OK);

        // Set up the mock
        when(api.getCharacters(eq(TEST_CHARACTER_NAME), any(String.class), any(String.class), any(Long.class)))
                .thenReturn(Observable.just(expectedResult));
    }

    @Test
    public void shouldBeAbleToSearchForTestCharacter() {
        // Launch the activity
        mainActivity.launchActivity(new Intent());

        // search for Test Character
        ViewInteraction appCompatAutoCompleteTextView = onView(
                allOf(withId(R.id.character), isDisplayed()));
        appCompatAutoCompleteTextView.perform(replaceText(TEST_CHARACTER_NAME));

        ViewInteraction appCompatButton = onView(
                allOf(withId(R.id.show), isDisplayed()));
        appCompatButton.perform(click());

        // Check view is loaded as we expect it to be
        onView(withText(TEST_CHARACTER_NAME)).check(matches(withText(TEST_CHARACTER_NAME)));
        onView(withId(R.id.description)).check(matches(withText(TEST_CHARACTER_DESCRIPTION)));
    }

    @Test
    public void shouldBeAbleToCacheTestCharacter() {
        // Launch the activity
        mainActivity.launchActivity(new Intent());

        // search for Test Character
        ViewInteraction appCompatAutoCompleteTextView = onView(
                allOf(withId(R.id.character), isDisplayed()));
        appCompatAutoCompleteTextView.perform(replaceText(TEST_CHARACTER_NAME));

        ViewInteraction appCompatButton = onView(
                allOf(withId(R.id.show), isDisplayed()));
        appCompatButton.perform(click());

        pressBack();

        // Check view is loaded as we expect it to be
        ViewInteraction cachedName = onView(
                allOf(withId(R.id.name), withText(TEST_CHARACTER_NAME),
                        childAtPosition(
                                childAtPosition(
                                        IsInstanceOf.instanceOf(android.widget.FrameLayout.class),
                                        0),
                                1),
                        isDisplayed()));
        cachedName.check(matches(withText(TEST_CHARACTER_NAME)));
    }
}
```

**TL;DR:**

我们将源代码解耦为不同层次的 MVP，使其阅读和维护变得简单。 像 Retrofit 和 RxJava 这样的新库在这方面帮助了我们。 我们试图编写一个简洁的代码，为了一个快乐的结局，当你想写一个测试时，如果不考虑网络连接或者 API 调用，那么这些努力就会显示出来，以测试用户界面或者代码功能的另一部分。 

在不久的将来，当你想添加一个新功能时，你将会受益于所有这些努力，每个人都会感到快乐。 :) 

