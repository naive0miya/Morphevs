# API 设计 — 异常处理

> 原文 (Medium)：[API Design — Handling exceptions](https://medium.com/@ZakTaccardi/api-design-handling-exceptions-84a143e32232)
>
> 作者：[Zak Taccardi](https://medium.com/@ZakTaccardi?source=post_header_lockup)

[TOC]

> 设计错误，报告缺陷



有两种异常。

- Errors  - 我们可以设计的预期状态

  例如：设备没有连接到互联网

  例如：用户提供了错误的凭据

- Defects  - 意外的状态，应用程序不知道如何处理。这些是开发人员未能解决的错误。 报告它们或者更好的做法是，破坏应用程序，以增加这种缺陷不能投入生产的可能性。 当用户界面默默失败时，日志语句很容易被错过 。

> 通过将错误建立在我们 api 的返回值中来设计错误 

## 天真的 API

![](https://ws4.sinaimg.cn/large/006tNc79gy1frospkxfbtj30m802p74k.jpg)

让我们使用我们天真的 Api！

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frosqp3wpnj30m808nwg2.jpg)

惊喜吧！ .login(credentials) 引发异常，因为我们提供了错误的凭证。我们该怎样改进这个？

## 更好的 API

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frosr2zf4sj30ks02ct8p.jpg)

什么是 LoginResponse？它表示登录请求的所有可能结果。让我们更深入地挖掘 : 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frosr8jykzj30m80b6di0.jpg)

现在的用法是: 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frosrfwxdaj30m8081gn8.jpg)

## 处理不同类型的错误

我们来看看 Error 对象究竟是什么样。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frosrksi45j30m805dt9q.jpg)

应用程序可以：

- 当设备重新获得连接时再试一次
- 提示用户输入正确的凭证。

下面是更好的 api 的使用情况，它解释了不同类型的错误。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frosruu1wcj30m805mwg0.jpg)

开发人员将以下内容传递给 error.handle ( )：

- 3个独立的函数能够处理3种错误类型中的每一种
- 只有映射到实际错误类型的函数才会被调用。如果错误是 invalidCredentials 的一个实例，那么只会调用它的函数（error.handle 的第一个参数）。

静态扩展方法 error.handle ( ) 也强制类型安全。它是什么样子的？

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fross05byxj30m808wmyy.jpg)

额外的好处：增加错误实现的新实现的额外弹性。 

## Kotlin 密封类

Kotlin 的最大特点之一是它的密封类，它可以为我们提供编译时间类型安全性。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fross8p0iwj30m8071aby.jpg)

用法现在变成：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frossgprmqj30m80ea0vd.jpg)

## Rx

通过适当的依赖管理和范围界定，您可以将所有的登录尝试都作为可观测的内容从 UI 中轻松使用。 

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frossnjxmxj30m807t76i.jpg)

## 结束

在 API 调用的返回值中建立错误使得所有的可能性对开发人员更加明显，允许应用程序强有力地处理每个场景。 

源代码在 GitHub 上可用。[[PR](https://gist.github.com/ZakTaccardi/cbf5dfb257463fb5c3ae43261507bdc1)]

