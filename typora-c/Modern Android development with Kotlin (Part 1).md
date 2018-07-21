#  现代 Android Kotlin 开发 - 第1部分

> 原文 (Medium)：[Modern Android development with Kotlin (September 2017) Part 1](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-1-f976483f7bd6)
>
> 作者：[Mladen Rakonjac](https://proandroiddev.com/@mladenrakonjac?source=post_header_lockup)

[TOC]

我们很难找到一个涵盖 Android 开发中所有新功能的项目，所以我决定写一个。在本文中，我们将使用以下内容：

0. Android Studio 3, beta 1

1. Kotlin language

2. Build Variants 

3. ConstraintLayout

4. Data binding library

5. MVVM architecture + repository pattern ( with mappers) + Android Manager Wrappers ( [Part 2](https://proandroiddev.com/modern-android-development-with-kotlin-september-2017-part-2-17444fcdbe86) )
6. RxJava2 and how it helps us in architecture 
7. Dagger 2.11, what is Dependency Injection, why you should use it. 
8. Retrofit (with Rx Java2)
9. Room (with Rx Java2)

我们的应用程序将什么？

我们的应用程序将是最简单的应用程序，涵盖所有提到的事情: 它只有一个功能可以从用户 googlesamples 中获取所有 GitHub 存储库，将这些数据保存在本地数据库中并显示给用户。 

我会尽力解释尽可能多的代码。你可以跟踪查看我在 github 上发布的代码提交。

让我们把手弄脏吧：

## 0. Android studio

要安装 Android Studio 3 beta 1，你必须转到此[页面](https://developer.android.com/studio/preview/index.html)。

注意：如果你想将其与以前的某个版本一起安装，请在 Mac 上将旧版本重命名为 “Android Studio Old”。你可以在[这里](https://developer.android.com/studio/preview/install-preview.html)找到更多的信息，包括 Windows 和 Linux。

Android Studio 支持 Kotlin。 去到创建 Android 项目。 你会看到新的东西：标签复选框包括 Kotlin 的支持。 默认情况下检查它。接下来按两次，然后选择空活动，然后完成。恭喜！ 你已经在 Kotlin 开发了第一款 Android 应用:) 

## 1. Kotlin

你可以看到 MainActivity.kt：

```kotlin
package me.fleka.modernandroidapp

import android.support.v7.app.AppCompatActivity
import android.os.Bundle

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

.kt 扩展名表示该文件是 Kotlin 文件。

MainActivity：AppCompatActivity ( ) 意味着我们正在继承 AppCompatActivity。

此外，所有的方法都必须有一个 fun 关键字，在 Kotlin 你不必使用 `;` ，但是如果你喜欢，你可以用。 你必须使用 override ，而不是注解。 

那么，在 savedInstanceState：Bundle？ ， `？` 这意味着 savedInstanceState 参数可以是 Bundle 类型或 null。 Kotlin 是空安全语言。如果你有：

```kotlin
var a : String
```

你会得到一个编译错误，因为必须初始化，它不能是空的。 这意味着你必须写: 

```kotlin
var a : String = "Init value"
```

此外，如果你这样做，你将会得到一个编译错误: 

```kotlin
a = null
```

为了使一个可以空出来，你必须写: 

```kotlin
var a : String?
```

这个为什么是 Kotlin 语言的重要特征？ 这有助于我们避免 NPE。 Android 开发者对 NPE 非常厌倦。 Tony Hoare 先生，即使是 null 的创造者，也[为发明它而道歉](https://en.wikipedia.org/wiki/Tony_Hoare)。  假设我们有可以为空的 nameTextView。 如果它是空的，下面的代码会给我们一个 NPE： 

```kotlin
nameTextView.setEnabled(true)
```

但是 Kotlin 实际上是好的，甚至不允许我们做这样的事情。这将迫使我们使用 `？` 或者 ` !! ` 操作符。如果我们使用 `？` 操作符：

```
nameTextView?.setEnabled(true)
```

只有当 nameTextView 不为 null 时，该执行才会继续。在另一种情况下，如果我们使用 ` !! ` 操作符：

```kotlin
nameTextView!!.setEnabled(true)
```

如果 nameTextView 为 null，它会给我们一个 NPE。这只是为了冒险家而设计的:)。 

 这是 Kotlin 的一个小介绍。当我们继续前进时，我会停下来描述其他 Kotlin 特定的代码。 

## 2. Build Variants

通常在开发中，你有不同的环境。最常见的是测试和生产环境。这些环境可以在服务器网址、图标、名称、目标 api 等方面有所不同。在 [fleka](http://www.fleka.me/)，在开始的每个项目中，我们都会遵循以下几点: 

- finalProduction 转到 Google Play 商店
- demoProduction 与生产服务器网址版本相同的版本，其新特性仍然不在 Google Play Store 中。 我们的客户可以把这个版本安装在 Google Play 的版本旁边，这样他们就可以测试它并给我们反馈 
- demoTesting 与 demoProduction 相同 ，测试服务器网址。
- mock 对我这个开发者和设计师来说很有用。 有时我们已经准备好了设计，我们的 API 还没有准备好。 等待 API 准备开始开发不是一个解决方案。 这个构建变体提供了假数据，所以设计团队可以测试它并给我们反馈。 不拖延是非常有帮助的。 一旦 API 准备好了，我们就将开发转移到 demoTesting 环境中 。

在这个应用程序中，我们将拥有所有这些。 它们的用途和名称都不一样。 在 gradle 3.0.0 flavorDimension  中有一个新的 api，它允许你混合不同的产品风格，所以你可以混合 demo  和 minApi 23的风格。 在我们的应用程序中，我们将只使用"默认"的。 进入 App 级 build.gradle，并将此代码插入 android { } 。

```groovy
flavorDimensions "default"
    
productFlavors {

    finalProduction {
        dimension "default"
        applicationId "me.fleka.modernandroidapp"
        resValue "string", "app_name", "Modern App"
    }

    demoProduction {
        dimension "default"
        applicationId "me.fleka.modernandroidapp.demoproduction"
        resValue "string", "app_name", "Modern App Demo P"
    }

    demoTesting {
        dimension "default"
        applicationId "me.fleka.modernandroidapp.demotesting"
        resValue "string", "app_name", "Modern App Demo T"
    }


    mock {
        dimension "default"
        applicationId "me.fleka.modernandroidapp.mock"
        resValue "string", "app_name", "Modern App Mock"
    }
}
```

转到 strings.xml 并删除 app_name 字符串，所以我们不会有冲突。 然后按立即同步。 如果你要在屏幕左侧构建变体，你可以看到4个不同的版本变体，每个都有两个构建类型: 调试和发布。切换到 demoProduction 生成变体并运行它。 然后切换到另一个并运行它。 你应该看到两个不同名称的应用程序。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frov83owlqj30ky0dkaap.jpg)

## 3. ConstraintLayout

如果你打开 activity_main.xml 类，你会看到布局是 ConstrainLayout。如果你编码 iOS 应用程序，你知道AutoLayout。  ConstraintLayout 与它非常相似。他们甚至使用相同的 Cassowary 算法。

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="me.fleka.modernandroidapp.MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>
```

约束有助于我们描述视图的关系。 对于每个视图，你应该有4个约束，一个为每一边。 在这种情况下，我们的视图限制在每一方的父级。 

如果你在设计标签中移动 Hello World 文本视图，将出现在 Text 选项卡新行中: 

```xml
app:layout_constraintVertical_bias="0.28"
```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frov87ou1cj30m00wowez.jpg)

设计标签和文本标签都是同步的。 我们在设计标签中的移动影响了文本标签和副词中的 xml。 纵向偏差描述了视角对他的约束的垂直倾向。 如果你希望视图以垂直为中心，则应使用: 

```xml
app:layout_constraintVertical_bias="0.28"
```

让我们让我们的活动只显示一个存储库。 它将有存储库的名称，恒星的数量，拥有者，它会显示存储库是否有问题。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frov8cnqodj30m80j1q5n.jpg)

为了得到这样的布局，xml 必须是这样的: 

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="me.fleka.modernandroidapp.MainActivity">

    <TextView
        android:id="@+id/repository_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="16dp"
        android:layout_marginStart="16dp"
        android:textSize="20sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintHorizontal_bias="0.0"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.083"
        tools:text="Modern Android app" />

    <TextView
        android:id="@+id/repository_has_issues"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="16dp"
        android:layout_marginStart="16dp"
        android:layout_marginTop="8dp"
        android:text="@string/has_issues"
        android:textStyle="bold"
        app:layout_constraintBottom_toBottomOf="@+id/repository_name"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="1.0"
        app:layout_constraintStart_toEndOf="@+id/repository_name"
        app:layout_constraintTop_toTopOf="@+id/repository_name"
        app:layout_constraintVertical_bias="1.0" />

    <TextView
        android:id="@+id/repository_owner"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp"
        android:layout_marginEnd="16dp"
        android:layout_marginStart="16dp"
        android:layout_marginTop="8dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/repository_name"
        app:layout_constraintVertical_bias="0.0"
        tools:text="Mladen Rakonjac" />

    <TextView
        android:id="@+id/number_of_starts"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp"
        android:layout_marginEnd="16dp"
        android:layout_marginStart="16dp"
        android:layout_marginTop="8dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="1"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/repository_owner"
        app:layout_constraintVertical_bias="0.0"
        tools:text="0 stars" />

</android.support.constraint.ConstraintLayout>
```

不要混淆 tools:text。它只是帮助我们有良好的布局预览。

我们可以注意到我们的布局是平坦的。 没有嵌套的布局。 你应尽可能少地使用嵌套布局，因为它可能会影响性能。 有关它的更多信息，你可以在[这里](https://developer.android.com/topic/performance/rendering/optimizing-view-hierarchies.html)找到。 而且，ConstraintLayout 可以很好地处理不同的屏幕尺寸：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frov8hb118j30m80nz76l.jpg)

我有一种能够真正达到想要的结果的感觉。 这是 ConstraintLayout 的一个小介绍。你可以在[这里](https://codelabs.developers.google.com/codelabs/constraint-layout/index.html?index=..%2F..%2Findex#0)找到 Google 代码实验室，在 [github](https://constraintlayout.com/)上有关于 CL 的文档。

## 4. Data binding library

当我听说数据绑定库时，我首先问自己的是，Butterknife 对我来说真的很好。 此外，我正在使用一个插件，帮助我从 XML 获取视图。 我为什么要改变它？“ 一旦我了解了更多关于数据绑定的知识，我就有和第一次使用 ButterKnife 时一样的感觉。

ButterKnife 帮助我们什么  

ButterKnife 可以帮助我们摆脱枯燥的 findViewById。所以，如果你有5个视图，没有 Butterknife 你有5 + 5行绑定你的视图。使用 ButterKnife 你只需5行。就是这样。

ButterKnife 有什么不好的？ 

ButterKnife 仍然没有解决代码维护的问题。 当我使用 ButterKnife 时，经常发现自己得到了一个运行时异常，导致我在 xml 中删除了视图，而且我没有删除 activity / fragment 类中的绑定代码。 另外，如果你想在 XML 中添加视图，你必须再次进行绑定。这真的很无聊。 你正在浪费你的时间来维护绑定。 

数据绑定库怎么样？

有很多好处！ 使用数据绑定库，你可以只用一行代码绑定你的视图！ 让我来告诉你它是如何工作的。 让我们把数据绑定库添加到我们的项目中: 

```groovy
// at the top of file 
apply plugin: 'kotlin-kapt'


android {
    //other things that we already used
    dataBinding.enabled = true
}
dependencies {
    //other dependencies that we used
    kapt "com.android.databinding:compiler:3.0.0-beta1"
}
```

请注意，在你的项目 build.gradle 文件中，数据绑定编译器的版本与 gradle 版本相同 。

```groovy
classpath 'com.android.tools.build:gradle:3.0.0-beta1'
```

现在按下同步键。 使用活动 main.xml 类和包装约束布局，布局标签 : 

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <android.support.constraint.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context="me.fleka.modernandroidapp.MainActivity">

        <TextView
            android:id="@+id/repository_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginEnd="16dp"
            android:layout_marginStart="16dp"
            android:textSize="20sp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintHorizontal_bias="0.0"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintVertical_bias="0.083"
            tools:text="Modern Android app" />

        <TextView
            android:id="@+id/repository_has_issues"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginEnd="16dp"
            android:layout_marginStart="16dp"
            android:layout_marginTop="8dp"
            android:text="@string/has_issues"
            android:textStyle="bold"
            app:layout_constraintBottom_toBottomOf="@+id/repository_name"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintHorizontal_bias="1.0"
            app:layout_constraintStart_toEndOf="@+id/repository_name"
            app:layout_constraintTop_toTopOf="@+id/repository_name"
            app:layout_constraintVertical_bias="1.0" />

        <TextView
            android:id="@+id/repository_owner"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginBottom="8dp"
            android:layout_marginEnd="16dp"
            android:layout_marginStart="16dp"
            android:layout_marginTop="8dp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/repository_name"
            app:layout_constraintVertical_bias="0.0"
            tools:text="Mladen Rakonjac" />

        <TextView
            android:id="@+id/number_of_starts"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginBottom="8dp"
            android:layout_marginEnd="16dp"
            android:layout_marginStart="16dp"
            android:layout_marginTop="8dp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintHorizontal_bias="1"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/repository_owner"
            app:layout_constraintVertical_bias="0.0"
            tools:text="0 stars" />

    </android.support.constraint.ConstraintLayout>

</layout>
```

请注意，你必须将所有 xmlns 移动到布局标签。 然后按建立图标或使用快捷 Cmd + F9。 我们需要构建项目，这样数据绑定库可以生成 ActivityMainBinding，我们将在 MainActivity 类中使用 。 

![img](https://media.giphy.com/media/xT39CX3yiZb8IaTpjq/200w.webp)

如果你不做 Build 项目，你将不会看到 ActivityMainBinding 类，因为它是在编译时生成的。 我们仍然没有完成绑定，我们只是说我们有非空变量，即 ActivityMainBinding 类型。 还有，你可以注意到我没有放？ 在 ActivityMainBinding 结束时，我没有初始化它。 这怎么可能呢？ Lateinit 修饰符允许我们有等待初始化的非空变量。 类似于 ButterKnife，绑定初始化应该在 onCreate 方法中完成，一旦我们的布局准备好了。 此外，你不应该在 onCreate 方法中声明绑定，因为你可能会在 onCreate 方法范围之外使用它。 我们的绑定不应该是空的，所以我们使用 lateinit。 使用 lateinit 修饰符，我们不必在每次访问时都需要校验绑定变量。 

让我们初始化绑定变量。 你应该更换: 

```java
setContentView(R.layout.activity_main)
```

下面是: 

```java
binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
```

就是这样！ 你成功地绑定了你的视图。 现在你可以访问它并应用一些更改。 例如，让我们把存储库名称改为" Modern Android Medium Article ": 

```java
binding.repositoryName.text = "Modern Android Medium Article"
```

正如你所看到的，我们可以通过绑定变量访问活动 main.xml 中的所有视图(当然是 id)。 这就是为什么数据绑定比 ButterKnife 更好的原因。 

Getters and setters in Kotlin

可能你已经注意到，我们没有像 Java 那样的 .setText ( ) 方法。我想在这里停下来解释一下 getter 和 setter 在 Kotlin 和 Java 之间的工作方式。

首先，你应该知道我们为什么要使用 setter 和 getter。 我们使用它来隐藏类的变量，只允许通过方法访问，这样我们就可以从类的客户端隐藏类的细节，并且禁止同一个客户直接更改我们的类。 假设我们在 Java 中有一个正方形类: 

```java
public class Square {
  private int a;
  
  Square(){
    a = 1;
  }

  public void setA(int a){
    this.a = Math.abs(a);
  }
  
  public int getA(){
    return this.a;
  }
  
}
```

使用 setA ( ) 方法，我们禁止类的客户将负值设置为正方形的一个原因侧必须不是负的。 使用这种方法，我们必须使一个私有的，所以它不能直接设置。 这也意味着我们类的客户端无法直接获得 a，所以我们必须提供一个 getter。 这个 getter 返回了一个。 如果你有10个具有相似需求的变量，你必须提供10个 getter。 写这样的代码是一件无聊的事情，我们通常不会使用我们的思想。 

kotlin 使我们的开发人员的生活更轻松。如果你调用

```kotlin
var side: Int = square.a
```

这并不意味着你可以直接访问。 同样的道理: 

```java
int side = square.getA();
```

在 Java 中，Kotlin 会自动生成默认的 getter 和 setter。在 Kotlin 中，只有当你有特殊的 setter 或 getter 时，才应该指定它。否则，Kotlin 为你自动生成它：

```kotlin
var a = 1
   set(value) { field = Math.abs(value) }
```

field ? 这又是什么？为了说清楚，我们来看看这个代码：

```kotlin
var a = 1
   set(value) { a = Math.abs(value) }
```

这意味着你在集合方法中调用 set 方法，因为在 Kotlin 没有直接访问该属性。 这会产生无限的递归。 当你调用 a = something 时，它会自动调用 set 方法。 我希望现在很清楚为什么你必须使用 field 关键字和如何设置和 getter 工作。 

让我们回到我们的代码。我想介绍一下 Kotlin 语言的一个很棒的功能，apply ：

```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        binding.apply {
            repositoryName.text = "Medium Android Repository Article"
            repositoryOwner.text = "Fleka"
            numberOfStarts.text = "1000 stars"
            
        }
    }
}
```

apply 允许你在一个实例中调用多个方法。

我们还没有完成数据绑定，还有更多伟大的事情。 让我们为 Repository 创建 ui 模型类（这是 Github Repository 的 UI Model 类，保存应显示的数据，不要将其与 Repository 模式错误放置）。 为了增加 Kotlin 类，你应该去 New -> Kotlin File/Class :

```kotlin
class Repository(var repositoryName: String?,var repositoryOwner: String?,var numberOfStars: Int? ,var hasIssues: Boolean = false)
```

在 Kotlin 中，主构造函数是类头的一部分。 如果你不想提供第二个构造函数，就是这样！ 你的类制作工作在这里完成。 对于字段分配没有构造函数参数，没有 getter 和 setter。 整个类只有一行！ 

回到 MainActivity.kt 类，并创建一个 Repository 的实例：

```kotlin
var repository = Repository("Medium Android Repository Article",
        "Fleka", 1000, true)
```

正如你所注意到的，对于对象的构建没有 new 关键字。 

现在我们来看一下 activity_main.xml 并添加 data 标签：

```xml
<data>
      <variable
        name="repository"
        type="me.fleka.modernandroidapp.uimodels.Repository"
        />
</data>
```

我们可以访问这个存储库变量，这是我们布局中的存储库类型。 例如，我们可以在 TextView 中使用存储库名称 id 执行以下操作: 

```xml
android:text="@{repository.repositoryName}"
```

repository_name 文本视图将显示从存储库变量的 repositoryName 属性获取的文本。唯一剩下的就是将来自 xml 的存储库变量绑定到 MainActivity.kt 的存储库。

按 Build 创建数据绑定库以生成所需的类并返回主活动并添加以下两行：

```java
binding.repository = repository
binding.executePendingBindings()
```

如果你运行的应用程序，你会看到 textview 将显示“ Medium Android Repository Article ”。不错的功能，是吧？ :)

但是，如果我们做到以下几点: 

```kotlin
Handler().postDelayed({repository.repositoryName="New Name"}, 2000)
```

新的文本会在2000毫升之后显示吗？ 不，不会的。 你必须重新设置存储库。 像这样的事情会起作用: 

```kotlin
Handler().postDelayed({repository.repositoryName="New Name"
    binding.repository = repository
    binding.executePendingBindings()}, 2000)
```

但是，如果我们每次更改一些属性时都必须这样做，那就太无聊了。 有一个更好的解决方案叫做属性观察者。 让我们首先描述一下什么是观察者模式，因为我们在 rxJava 部分也需要它: 

可能你已经听说过 http://androidweekly.net/。 这是一个关于 Android 开发的每周通讯。 如果你想收到它，你必须订阅它给你的电子邮件地址。 过了一段时间，如果你决定，你可以停止在他们的网站取消订阅选项。

这是 Observer / Observable 模式的一个例子。 在这种情况下，Android Weekly 是可观察的，它每周都会发布通讯。 读者是观察者，因为他们订阅它，等待新的发出，一旦他们收到，他们就会阅读它，如果他们中的一些人认为她不喜欢，她 / 他可以停止听它。 

在我们的例子中属性 Observer 是 xml 布局，它将侦听 Repository 实例中的更改。 所以，Repository 是可观察的。 例如，一旦在 Repository 类的实例中更改了存储库名称属性，xml 应该不需要调用就更新: 

```kotlin
binding.repository = repository
binding.executePendingBindings()
```

如何使用数据绑定库？数据绑定库为我们提供了 Repository 类应该实现的 BaseObservable 类：

```kotlin
class Repository(repositoryName : String, var repositoryOwner: String?, var numberOfStars: Int?
                 , var hasIssues: Boolean = false) : BaseObservable(){

    @get:Bindable
    var repositoryName : String = ""
    set(value) {
        field = value
        notifyPropertyChanged(BR.repositoryName)
    }

}
```

BR 是使用 Bindable  注解时自动生成的类。 正如你所看到的，一旦新值被设定，我们就会通知它。 现在你可以运行该应用程序，你将看到存储库名称将在2秒后再次更改，而不需要再次调用 executePendingBindings ( )。 

这一切都是为了这个部分。在下一部分中，我将介绍有关 MVVM 模式和 Repository 模式的内容，并将介绍 Android Wrapper 管理器。你可以在[这里](https://github.com/mladenrakonjac/ModernAndroidApp)找到所有的代码。本文涵盖了以下[提交](https://github.com/mladenrakonjac/ModernAndroidApp/commit/0add63eb710a71f44b294e2af5d289c47a89a39a)代码。

