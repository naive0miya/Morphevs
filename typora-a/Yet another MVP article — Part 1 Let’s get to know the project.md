# 另一篇 MVP 文章 - 第1部分：让我们了解这个项目

> 原文 (Medium)：[Yet another MVP article — Part 1: Let’s get to know the project](https://hackernoon.com/yet-another-mvp-article-part-1-lets-get-to-know-the-project-d3fd553b3e21)
>
> 作者： [Mohsen Mirhoseini](https://hackernoon.com/@mirhoseini)

[TOC]

## 变化如何开始
这一切都始于 [SOLID](https://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29)（面向对象的设计原则），感谢亲爱的 [Robert C. Martin](https://en.wikipedia.org/wiki/Robert_C._Martin)。
根据[维基百科](https://en.wikipedia.org/wiki/SOLID_%28object-oriented_design%29)的 SOLID 文章，它代表：
>- S （SRP） ：单一责任原则
>- O（OCP）：打开/关闭原则
>- L （LSP） ：Liskov 替代原则
>- I  （ISP）  ：接口隔离原理
>- D （DIP） ：依赖倒置原则

MVP 以某种方式试图遵循所有五项原则。我会尽我最大的努力在我的样品项目中一个一个的精确定位，使它更加透明。  根据这篇完美的 MVP [文章](http://hannesdorfmann.com/mosby/mvp/)，它表示：

>- 模型是将在视图（用户界面）中显示的数据。
>- 视图是一个接口，用于显示数据（模型），并将用户命令（事件）路由到演示者，以处理该数据。该视图通常对其演示者有一个地址引用。
>- 演示者是“中间人”（由 MVC 中的控制器扮演），并且具有对视图和模型的引用。



## 模型？！！！
>请注意，“模型”一词是误导性的！
>
>它应该是检索或操纵模型的业务逻辑。
>
>例如：例如: 如果你有一个数据库存储用户，你的视图希望显示一个用户列表，然后演示者将参考你的数据库业务逻辑(如 DAO) ，从这里演示者将查询用户列表 

## 你能再多解释一下 MVP 吗？

Noooooo！[谷歌一下](https://www.google.com/#q=android+mvp)，你会找到所有关于这种新方法的理论（或者至少看看这篇[文章](http://hannesdorfmann.com/mosby/mvp/)。）

## 这个示例项目是关于什么的？
这个应用程序是一个漫威的字符搜索应用，是一个简单的 Android 客户端为漫威网站。 这个应用程序是由我创建的，作为技术评估的一部分由 [smava GmbH](https://www.smava.de/) 技术团队进行。 

![@Marvel Android Application screenshot](https://ws1.sinaimg.cn/large/006tNc79gy1froo4k7yekj30go0tmdt2.jpg)

应用程序必须搜索字符、查看信息和缓存最后的搜索。 

这个项目使用 MVP 实现，包含了现代的 Android 开发概念和库，它们可以真正改变你的职业生活！ 

我会尽力解释这篇文章的不同部分 ：dagger，Retrofit，RxJava 和测试。

该项目还使用 [Circleci.com](https://circleci.com/) 和 [Travis-ci.org](https://travis-ci.org/) 进行了持续集成（CI），并使用 [Codecov.io](https://codecov.io/) 编写了代码覆盖率报告，同时还使用了你可以自行研究谷歌新的 Firebase 服务，这不是本文的主题。 

在开始之前，你可以通过阅读 [README](https://github.com/mirhoseini/marvel/blob/master/README.md) 和 [Task](https://github.com/mirhoseini/marvel/blob/master/Task.pdf) 来熟悉项目。

好吧，给我看看你有什么：

[mirhoseini/marvel | GitHub](https://github.com/mirhoseini/marvel)


我们来看看这个项目的结构。

我个人喜欢简洁的代码，所以我非常喜欢把一个项目分解成一些有意义的部分，帮助我和整个团队更清晰地处理任务。 

![](https://ws2.sinaimg.cn/large/006tNc79gy1froo4pr5w9j30m80usq7v.jpg)

## 模块：
项目由两个主要的和一个 java 控制台示例模块组成：

- app（marvel-app）：Android 应用程序模块
- core-lib（marvel-core）：核心库模块
- console（marvel-console）：一个 Java 控制台示例模块

应用程序模块包含 MVP 的 Android 视图层，而其他层(Model & Presentation)都被放置在纯 java 的核心中，编译之后就是 jar 库。 

## 将代码分解成模块有什么用？
- 首先，分离 Android 应用程序模块会提醒你不要将 Context 或任何与 Android 相关的对象传递给 Presenter 或者 Model！ 所以请不要再这么做了 
- 其次，可以确保核心部分是完整的，甚至可以在另一个 UI 中使用它（即：Java 控制台示例，Web Applet 或作为一个预测即将在不久的将来使用 Java 的 iOS 应用程序）
- 最后，我和我们的团队非常喜欢开发这样的应用程序！整个团队成员从核心分离中获益，甚至把核心部署在一个 git 子模块中，并将其用于不同的核心和不同用户界面的项目。

![@Java控制台示例模块的结果，它与Android应用程序的核心一样工作](https://ws3.sinaimg.cn/large/006tNc79gy1froo4tyui2j30m803rwf0.jpg)

## 小模块的名称清理：
为了使模块看起来更加方便和漂亮， 你可以像这样编辑 settings.gradle 文件：

```groovy
include ':marvel-app', ':marvel-core', ':marvel-console'
project(':marvel-app').projectDir = new File('app')
project(':marvel-core').projectDir = new File('core-lib')
project(':marvel-console').projectDir = new File('console')
```
这让你看起来很干净：
![](https://ws3.sinaimg.cn/large/006tNc79gy1froo4wnk2vj30bm03cwek.jpg)

输出 APK 文件的相关名称。 

## 如何避免不同模块中的版本冲突和冗余？

使用 Gradle 功能可以方便地获得一个干净的 build.gradle 文件，避免版本冲突和项目模块中的冗余。

首先，将项目的所有依赖项放在 Gradle 文件中，比如 libraries.Gradle: 

```groovy
ext {
    minSdkVersion = 9
    compileSdkVersion = 25
    buildToolsVersion = "25.0.0"

    //Android
    androidSupportVersion = "25.0.0"
    butterknifeVersion = "8.0.1"
    
    /*...*/

    libraries = [
            androidSupport   : "com.android.support:support-v4:${androidSupportVersion}",
            appCompat        : "com.android.support:appcompat-v7:${androidSupportVersion}",
            designSupport    : "com.android.support:design:${androidSupportVersion}",
            
            /*...*/
            
    ]
    
    /*...*/

}
```
然后将其添加到你的项目主 [build.gradle](https://github.com/mmirhoseini/marvel/blob/master/build.gradle) 文件（注意最后一行）
```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        
        /*...*/

    }
}

apply from: "./libraries.gradle"
```
最后，在你的应用程序模块 build.gradle 文件（注意依赖部分）
```groovy
apply plugin: 'com.android.application'
/*...*/

android {
    compileSdkVersion rootProject.ext.compileSdkVersion
    buildToolsVersion rootProject.ext.buildToolsVersion

    /*...*/
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(':marvel-core')

    testCompile rootProject.ext.testLibraries.junit
    testCompile rootProject.ext.testLibraries.robolectric

    androidTestCompile rootProject.ext.testLibraries.mockito
    compile rootProject.ext.libraries.appCompat
    compile rootProject.ext.libraries.androidSupport
    compile rootProject.ext.libraries.designSupport
    
    /*...*/
}
```
你的核心模块 [build.gradle](https://github.com/mmirhoseini/marvel/blob/master/core-lib/build.gradle) 文件：
```groovy
apply plugin: 'java'
/*...*/

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile rootProject.ext.libraries.rxjava

    testCompile rootProject.ext.testLibraries.junit
    testCompile rootProject.ext.testLibraries.mockito

    compile rootProject.ext.libraries.retrofit
    
    /*...*/
}
```
## 核心模块内部发生了什么？
![@核心模块的文件结构](https://ws1.sinaimg.cn/large/006tNc79gy1froo51d72sj30go0sqabs.jpg)
- 基本包：保存所有基本接口，包含所有交互器，演示者和视图的通用方法。
- 字符包：具有搜索和缓存漫威字符信息的应用程序的主要功能 
- 数据库包：缓存中的字符数据是使用 [OrmLite](http://ormlite.com/) 在这个示例中完成的，我不会解释太多，因为它是脱离主题的，但是你可以花时间查看所有代码！
- 域包：包含使用 retrofit2 和 RxJava 库连接到网络 API 的代码。
- util 包：这个项目需要有很多有用的类，比如：包含所有核心常量变量的常量，用于 Marvel API 调用哈希参数的 HashGenerator 和调度器接口，用于帮助 RxJava 和 RxAndroid 的多线程调用。 线程（我将在本文的相关部分进行更多解释）。

**参照 SOLID 的依存倒置原则，“应该依赖于抽象，不依赖于具体” 或参考 Novoda 的[这篇精彩的文章](https://www.novoda.com/blog/designing-something-solid/) “You wouldn’t wire a lamp directly to your house”，两个模块之间的所有链接应用程序和核心）是使用接口和 dagger 连接的。**

## 应用程序模块内部发生了什么？
![](https://ws2.sinaimg.cn/large/006tNc79gy1froo577vanj30j20vmtb7.jpg)

- 活动包：拥有3个活动，这是 Android 应用程序 UI 的后台。
- 基本包：具有2个基本的活动和片段的抽象类，其中包含一般的注入方法，可用于其他常规方法 
- 字符包：包含使用两个 Search＆Cache 片段实现的应用程序的主要功能。
- 数据库包：包含使用 OrmLite 这是脱离主题的 Android 端数据库代码！
- util 包：完整的项目需要的有用的 Android 端类，即：扩展核心的常量类，包含应用程序常量变量的 AppConstants，实现核心的 SchedulerProvider 的 AppSchedulerProvider，并提供 RxAndroid 调度程序，CustomBindingAdapter 帮助新的 Android DataBinding 插件 使用 Picasso 库和 GridSpacingItemDecoration 加载图像，这有助于 RecyclerView 网格项目间距。 （这是一个简短的信息，我将在本文的相关部分进一步解释）。

**这就是现在...**
请从 GitHub 的项目克隆，并更熟悉它，因为在下一部分我会解释更多关于 dagger，以及它如何帮助连接模块和图层的不同对象。


