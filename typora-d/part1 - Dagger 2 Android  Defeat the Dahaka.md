# Dagger 2 Android : 第1部分

> 原文 (Medium)：[Dagger 2 Android : Defeat the Dahaka](https://proandroiddev.com/dagger-2-android-defeat-the-dahaka-b1c542233efc?source=user_profile---------4----------------)
>
> 作者：[Garima Jain](https://proandroiddev.com/@ragdroid?source=post_header_lockup)

[TOC]

到目前为止，我们已经在 Dagger 和依赖注入方面进行过多次演讲和博客文章。 我必须指出，人们在解释各种依赖注入概念方面做得很好， 包括技术细节和案例研究。 但是，在我看来，Dagger 已经成为我们现在编码生活的一部分。 然而，我们仍然不时发现自己迷失在依赖注入的世界，并且很难处理 “Dahaka”。生成代码的野兽 

**开始学习 Dagger？**首先创建一个 @Module 和一个 @Component 。添加 @Singleton 范围。

**依赖关系失控？**创建另一个依赖组件或子组件。

**创建另一个组件？**你应该创造另一个范围！如果创建限定符不够的话， 请使用 LazyInjection，ProviderInjection，staticInjection，AsyncInjection，MultiBinding 以及其他相关的！

**还活着吗？**现在我们向你介绍 Dagger Android，我确信它一定会杀死你。

但是等等！ 模块和组件之间究竟有什么关系？ 

我们不知道。 谢天谢地！Dagger  2在编译时生成所有的代码，我们可以亲眼看到背后发生了什么。 

我们可以看到，对于我们的每个组件，Dagger 都会生成一个 DaggerComponent，它和我们的 Module 有一个 “has-a” 的关系。

到现在为止还挺好？ “Dahaka”（Providers，Factories，Builders，MemberInjectors，DoubleChecks，Lazy ......等在海洋中的无限风暴），你会很容易迷失在这些类中。

在阅读下面这一段引用前，可以先播放一段[音轨](https://www.youtube.com/watch?v=mrDEwksdDoI)。

> “大多数人认为依赖注入就像一条向一个方向迅速而确定地流动的河流，但我看到了依赖的面孔，我可以告诉你他们错了。 依赖注入是暴风雨中的海洋。 你可能想知道我是谁，我为什么这么说; 坐下来，我会告诉你一个你从来没有听过的故事! “ - 引用来自[波斯王子](http://www.imdb.com/title/tt0384444/quotes)的启发

![@Dahaka](https://ws1.sinaimg.cn/large/006tKfTcgy1frot63mayvj30m80c8ac6.jpg)

在这一系列的博客文章中，我们的目标是打败 “Dahaka”（生成类的海洋中的风暴），或者至少试着驯服这头野兽。 我们将首先回顾 “Dagger 2” 的基础知识，并试图通过进入生成的类来释放这个野兽，并找出各种模式。 然后，我们将讨论这些模式的细节，并学习如何应用这些模式。 然后，我们将找到我们的方式，针对 Dagger Android 的技术细节。还有如何在不盲目应用这些注解的情况下转向 Dagger Android，同时也不要死在"Dahaka"手中， 而是要将链索重新放在野兽身上。

阅读这些系列文章后，你应该更好地了解幕后的情况，以及如何巧妙地走向 Dagger Android，并保持冷静:)

以下是我们如何驯服野兽的方法: 

- **“时间规则”：简介：** 我们已经有了关于 Dagger2 基础知识的非常棒的文章。这将是我的最爱的集合。
- **“Dahaka 的面孔”：定义：**窥探生成的代码和各种 Dagger 术语和模式。让我们在 [Factories](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)， MemberInjectors， [ProviderInjection](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)， LazyInjections ,  AsyncInjection 的海洋中游泳。
- **“发挥野兽”：关系：**
  - [作用域（**Scope**），组件（Components），子组件（ Subcomponents），依赖组件（Dependent Components）和建造者（Builders）的故事](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37)。
  - [Dagger 2 : Component Relationships & Custom Scopes](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37)
- **“打败 Dahaka”：战斗：**让我们使用手中的工具，如 Builders ，[@Binds ](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f)，@BindsInstance ，[Component.Builder](https://medium.com/@ragdroid/dagger-2-component-builder-1f2b91237856) ，Subcomponent.Builder， [DoubleCheck ](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)，Multibinding， [@ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f) ...来驯服野兽。
  - [Dagger 2 : Component.Builder](https://medium.com/@ragdroid/dagger-2-component-builder-1f2b91237856)
  - [Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)
  - [Dagger 2 Annotations : @Binds & @ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f)
- **“交友”：实现：** Dagger 2 Android😎！

## TL; DR

系列的文章列表：这里是文章的链接：

- [Dagger 2 : Component.Builder](https://medium.com/@ragdroid/dagger-2-component-builder-1f2b91237856)
- [Dagger 2 : Check — SingleCheck — DoubleCheck … Scopes](https://medium.com/@ragdroid/dagger-2-check-singlecheck-doublecheck-scopes-4ee48fc31736)
- [Dagger 2 : Component Relationships & Custom Scopes](https://medium.com/@ragdroid/dagger-2-component-relationships-custom-scopes-8d7e05e70a37)
- [Dagger 2 Annotations : @Binds & @ContributesAndroidInjector](https://proandroiddev.com/dagger-2-annotations-binds-contributesandroidinjector-a09e6a57758f)

可以观看视频：[[Video]](https://www.youtube.com/watch?v=iczp_toHxmA)