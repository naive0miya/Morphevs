# Android 架构

> 原文 (github.com/ribot)：[android_architecture](https://github.com/ribot/android-guidelines/blob/master/architecture_guidelines/android_architecture.md)
>
> 作者：[ribot](https://github.com/ribot)

[TOC]

## Architecture Guidelines

我们的 Android 应用程序架构是基于 MVP (Model View Presenter)模式。 

- View (UI layer): 这就是活动，片段和其他标准 Android 组件所在的地方。它负责向用户显示从演示者收到的数据。它还处理用户的交互和输入(点击侦听器等) ，并在需要时在演示器中触发正确的操作。 
- Presenter:演示者订阅由 DataManager 提供的 RxJava Observables。他们负责处理订阅生命周期，分析/修改 DataManager 返回的数据，并在视图中调用适当的方法来显示数据。
- Model (Data Layer): 负责检索，保存，缓存和格式数据。它可以与本地数据库和其他数据存储以及平稳的 API 或第三方 SDK 进行通信。它分为两部分：一组助手和一个 DataManager。项目中的助手的数量是不同的，并且它们中的每一个都具有非常特定的功能，例如，与 API 通信或在 SharedPreferences 中保存数据。DataManager 使用 Rx 操作符合并和转换来自不同助手的输出，因此它可以：1）为 Presenter 提供有意义的数据，2）总是一起发生的组操作。 此层还包含定义数据结构的实际模型类。 

[![img](https://ws1.sinaimg.cn/large/006tKfTcgy1frovtnvnfsj30ne0gj75y.jpg)](https://github.com/ribot/android-guidelines/blob/master/architecture_guidelines/architecture_diagram.png)

从右向左看图：	

- Helpers (Model): 一组类，每个类都有一个非常具体的责任。它们的功能可以从访问到API或数据库到实现某些特定的业务逻辑。每个项目都有不同的助手，但最常见的是：
  - DatabaseHelper: 它处理从本地 SQLite 数据库插入，更新和检索数据。它的方法返回 Rx Observable，它发出纯 java 对象（模型）
  - PreferencesHelper: 它保存并从 SharedPreferences 获取数据，它可以直接返回 Observables 或纯 java 对象（模型）。
  - Retrofit services : [Retrofit](http://square.github.io/retrofit) 与 Restful API 交互的接口，每个不同的 API 都将拥有自己的 Retrofit 服务。他们返回 Rx Observables。
- Data Manager (Model): 这是架构的关键部分。它保持对每个助手类的引用，并使用它们来满足来自演示者的请求。其方法广泛使用 Rx 操作符来组合，转换或过滤来自助手的输出，以便为演示者生成所需的输出。它返回发射数据模型的观察值。
- Presenters: 订阅 DataManager 提供的 observables 并处理数据以便在 View 中调用正确的方法。
- Activities, Fragments, ViewGroups (View): 标准 Android 组件，实现演示者可以调用的一组方法。他们还处理点击等用户交互，并通过调用 Presenter 中的相应方法来相应地采取行动。这些组件还实现了框架相关的任务，例如管理 Android 生命周期，附加视图等。
- Event Bus: 它允许 View 组件通知模型中发生的某些类型的事件。一般情况下，DataManager 发布可由活动和片段订阅的事件。事件总线仅用于不止与一个屏幕相关且具有广播本质的非常具体的动作，例如，用户已经退出。

