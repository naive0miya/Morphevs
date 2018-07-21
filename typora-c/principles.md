# Android 开发原则

> 原文 (java-design-patterns.com)：[principles](http://java-design-patterns.com/principles/)

[TOC]

每个程序员都从理解编程原理和模式中获益。 这个概述是我自己的一个参考，我只是把它放在这里。 也许在设计，讨论或评论期间对你有帮助。 请注意，这还远远没有完成，你经常需要在相互冲突的原则之间进行权衡。

这个列表是受到 “[良好编程原则](http://www.artima.com/weblogs/viewpost.jsp?thread=331531)” 的启发。 我觉得这个列表非常接近，但并不完全符合我个人对类似事物的看法。 此外，我想要更多的推理，细节和链接到进一步的资源。 如果你有任何改进的反馈或建议，请告诉我。 

## Contents

### [通用](http://java-design-patterns.com/principles/#generic)

- [KISS (Keep It Simple Stupid) 保持简单](http://java-design-patterns.com/principles/#kiss)
- [YAGNI](http://java-design-patterns.com/principles/#yagni)
- [Do The Simplest Thing That Could Possibly Work 做最简单的事可能会工作](http://java-design-patterns.com/principles/#do-the-simplest-thing-that-could-possibly-work)
- [Separation of Concerns 关注点分离](http://java-design-patterns.com/principles/#separation-of-concerns)
- [Keep Things DRY 不要重复工作](http://java-design-patterns.com/principles/#keep-things-dry)
- [Code For The Maintainer 维护者的代码](http://java-design-patterns.com/principles/#code-for-the-maintainer)
- [Avoid Premature Optimization 避免过早优化](http://java-design-patterns.com/principles/#avoid-premature-optimization)
- [Boy-Scout Rule 童子军守则](http://java-design-patterns.com/principles/#boy-scout-rule)

### [模块间/类](http://java-design-patterns.com/principles/#inter-moduleclass)

- [Minimise Coupling 尽量减少耦合](http://java-design-patterns.com/principles/#minimise-coupling)
- [Law of Demeter 德米特法则](http://java-design-patterns.com/principles/#law-of-demeter)
- [Composition Over Inheritance 组合优于继承](http://java-design-patterns.com/principles/#composition-over-inheritance)
- [Orthogonality 正价性](http://java-design-patterns.com/principles/#orthogonality)
- [Robustness Principle 稳健性原则](http://java-design-patterns.com/principles/#robustness-principle)
- [Inversion of Control 控制反转](http://java-design-patterns.com/principles/#inversion-of-control)

### [模块/类](http://java-design-patterns.com/principles/#moduleclass)

- [Maximise Cohesion 最大化凝聚](http://java-design-patterns.com/principles/#maximise-cohesion)
- [Liskov Substitution Principle Liskov代替原则](http://java-design-patterns.com/principles/#liskov-substitution-principle)
- [Open/Closed Principle 开闭原则](http://java-design-patterns.com/principles/#openclosed-principle)
- [Single Responsibility Principle 单一责任原则](http://java-design-patterns.com/principles/#single-responsibility-principle)
- [Hide Implementation Details 隐藏实现细节](http://java-design-patterns.com/principles/#hide-implementation-details)
- [Curly's Law](http://java-design-patterns.com/principles/#curlys-law)
- [Encapsulate What Changes 封装变化](http://java-design-patterns.com/principles/#encapsulate-what-changes)
- [Interface Segregation Principle 接口隔离原则](http://java-design-patterns.com/principles/#interface-segregation-principle)
- [Command Query Separation 命令查询分离](http://java-design-patterns.com/principles/#command-query-separation)

## KISS (保持)

大多数系统如果保持简单而不是复杂的话，效果最好。 

为什么 

- 更少的代码花费更少的时间写，有更少的错误，并且更容易修改 
- 简单是终极的复杂性 
- 似乎完美不是在没有什么可以补充的时候达到的，而是当没有什么可以带走的时候 

资源 

- [KISS principle](http://en.wikipedia.org/wiki/KISS_principle)
- [Keep It Simple Stupid (KISS)](http://principles-wiki.net/principles:keep_it_simple_stupid)

## YAGNI

Yagni 的意思是"你不需要它": 在必要的时候不要实施。 

为什么 

- 任何只用于明天需要的功能的工作，都意味着从当前迭代需要完成的功能中失去努力 
- 它会导致代码膨胀; 软件变得更大更复杂 

怎么做 

- 总是在你真正需要它们的时候实现它们，而不是当你预见到你需要它们的时候 

资源 

- [You Arent Gonna Need It](http://c2.com/xp/YouArentGonnaNeedIt.html)
- [You’re NOT gonna need it!](http://www.xprogramming.com/Practices/PracNotNeed.html)
- [You ain't gonna need it](http://en.wikipedia.org/wiki/You_ain't_gonna_need_it)

## Do The Simplest Thing That Could Possibly Work

为什么 

- 如果我们只是研究问题到底是什么，那么对真正问题的真正进展就会最大化 

怎么做 

- 问自己：“什么是最简单的事情可能工作？”

Resources

- [Do The Simplest Thing That Could Possibly Work](http://c2.com/xp/DoTheSimplestThingThatCouldPossiblyWork.html)

## Separation of Concerns

关注点分离是把电脑程式分隔为不同部分的设计原则，这样每个部分就分别处理一个单独的问题。 例如，应用程序的业务逻辑是一个关注点，用户界面是另一个关注的问题。 改变用户界面不应该要求改变业务逻辑，反之亦然。 

引用 [Edsger W. Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra) （1974）：

> 这就是我有时候称之为"关注点分离"的东西，就我所知，即使不完全可能，它也是唯一可用的有效排列思想的技巧。 这就是我所说的"把注意力集中在某些方面"的意思: 它并不意味着忽视其他方面，而是从这一方面的观点来看，另一方面是无关紧要的。 

为什么 

- 简化软件应用程序的开发和维护 
- 当关注点被很好地分离时，单独的部分可以重复使用，也可以独立开发和更新 

怎么做 

- 将程序功能分解成尽可能少重叠的单独模块 

资源 

- [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)

## Keep things DRY

每一个知识都必须在一个系统中有一个单一的、明确的、权威的表示形式。 

程序中的每一个重要功能都应该在源代码中的一个地方实现。 如果类似的函数是通过不同的代码进行的，那么通过抽象出不同的部分，将它们合并为一个函数通常是有益的。 

为什么 

- 重复(无意或有目的的重复)会导致维护性的噩梦、糟糕的代理和逻辑上的矛盾 
- 对系统中任何单个元素的修改不需要在其他逻辑不相关的元素中进行更改 
- 此外，与逻辑相关的所有变化都是可预见的和一致的，因此保持同步 

怎么做 

- 把业务规则、长表达式、如果语句、数学公式、元数据等放在一个地方 
- 确定系统中使用的每一个知识的唯一、确定的来源，然后使用该源生成知识的适用实例(代码、文档、测试等) 
- 应用[第三条规则](http://en.wikipedia.org/wiki/Rule_of_three_(computer_programming))。

资源 

- [Dont Repeat Yourself](http://c2.com/cgi/wiki?DontRepeatYourself)
- [Don't repeat yourself](http://en.wikipedia.org/wiki/Don't_repeat_yourself)
- [Don't Repeat Yourself](http://programmer.97things.oreilly.com/wiki/index.php/Don't_Repeat_Yourself)

相关资料 

- [Abstraction principle](http://en.wikipedia.org/wiki/Abstraction_principle_(computer_programming))
- [Once And Only Once](http://c2.com/cgi/wiki?OnceAndOnlyOnce) 是 DRY 的子集(也称为重构目标) 
- [Single Source of Truth](http://en.wikipedia.org/wiki/Single_Source_of_Truth)
- 违反 DRY 规定的  [WET](http://thedailywtf.com/articles/The-WET-Cart) (把所有东西都写两遍) 

## Code For The Maintainer

为什么 

- 维护是目前任何项目中最昂贵的阶段 

怎么做 

- 做维护者。
- 总是把维护你的代码的人看作是一个知道你住在哪里的暴力精神病患者 
- 总是以这样一种方式编写和评论，如果一个人一个人几个小年级的人拿起代码，他们就会乐于阅读并从中学习 
- [别让我觉得](http://www.sensible.com/dmmt.html)。
- 使用[最小惊讶原则](http://en.wikipedia.org/wiki/Principle_of_least_astonishment)。

资源 

- [Code For The Maintainer](http://c2.com/cgi/wiki?CodeForTheMaintainer)
- [The Noble Art of Maintenance Programming](http://blog.codinghorror.com/the-noble-art-of-maintenance-programming/)

## Avoid Premature Optimization

引用 [Donald Knuth](http://en.wikiquote.org/wiki/Donald_Knuth):

> 程序员浪费了大量的时间去思考或担心程序中非关键部分的速度，而这些效率的尝试实际上在考虑调试和维护时会产生强烈的负面影响。 我们应该忘记小的效率，比如说大约97% 的时间: 过早的优化是所有邪恶的根源。 然而，我们不应该在这个关键的3% 中放弃我们的机会。 

理解什么是"过早"，什么是"不成熟"，当然是至关重要的。 

为什么 

- 目前还不清楚瓶颈在哪里 
- 在优化之后，可能会更难阅读和维护 

怎么做 

- [让它工作做得正确快速](http://c2.com/cgi/wiki?MakeItWorkMakeItRightMakeItFast)
- 不要优化直到你需要，并且只有在分析之后你发现了一个瓶颈优化 

资源 

- [Program optimization](http://en.wikipedia.org/wiki/Program_optimization)
- [Premature Optimization](http://c2.com/cgi/wiki?PrematureOptimization)

## Minimise Coupling

模块/组件之间的耦合是它们相互依赖的程度;较低的耦合是更好的。换句话说，耦合是代码单元 “B” 在代码单元 “A” 的未知变化之后将 “中断” 的概率。

为什么 

-  一个模块的改变通常会在其他模块中产生连锁反应 
- 由于模块间依赖性增加，模块的组装可能需要更多的努力和 / 或时间 
- 由于必须包含相关模块，因此特定模块可能难以重用和 / 或测试 
- 开发人员可能害怕更改代码，因为他们不确定什么可能会受到影响 

怎么做 

- 消除、减少和减少必要关系的复杂性 
- 通过隐藏实现细节，减少了耦合 
- 应用 [Law of Demeter](http://java-design-patterns.com/principles/#law-of-demeter)。

资源 

- [Coupling](http://en.wikipedia.org/wiki/Coupling_(computer_programming))
- [Coupling And Cohesion](http://c2.com/cgi/wiki?CouplingAndCohesion)

## Law of Demeter

不要和陌生人说话。 

为什么 

- 它通常收紧耦合
- 它可能会暴露太多的实现细节 

怎么做 

对象的方法只能调用以下方法：

- 对象本身 
- 方法的一个参数 
- 方法中创建的任何对象 
- 对象的任何直接属性 / 字段 

资源 

- [Law of Demeter](http://en.wikipedia.org/wiki/Law_of_Demeter)
- [The Law of Demeter Is Not A Dot Counting Exercise](http://haacked.com/archive/2009/07/14/law-of-demeter-dot-counting.aspx/)

## Composition Over Inheritance

为什么 

- 减少类之间的耦合 
- 使用继承，子类容易做出假设，并打破 LSP 

怎么做 

-  Lsp (可替换性)的测试来决定何时继承 
- 当存在"has"(或"uses a")关系时组合，当"is a"时继承 

资源 

- [Favor Composition Over Inheritance](http://blogs.msdn.com/b/thalesc/archive/2012/09/05/favor-composition-over-inheritance.aspx)

## Orthogonality

> 正交性的基本思想是，在概念上不相关的事物在系统中不应该被关联。 

取自: [Be Orthogonal](http://www.artima.com/intv/dry3.html)

> 它与简单相关，设计越是正交，例外越少。 这使得在编程语言中更容易学习、阅读和编写程序。 正交特征的意义独立于上下文; 关键参数是对称性和一致性。 

取自: [Orthogonality](http://en.wikipedia.org/wiki/Orthogonality_(programming))

## Robustness Principle

> 在你所做的事情上要保守，在你接受别人的事情上要自由 

协作服务依赖于彼此的接口。 通常，接口需要进化，导致另一端接收未指定的数据。 如果收到的数据没有严格遵循规范，那么天真的实现将拒绝协作。 一个更复杂的实现仍然会忽略它不能识别的数据。 

为什么 

- 为了能够进化服务，你需要确保提供者可以进行更改，以支持新的需求，同时尽量减少现有客户的破损 

怎么做 

-  将命令或数据发送到其他机器(或同一机器上的其他程序)的代码应该完全符合规范，但接收输入的代码只要含义清楚，就应该接受非对称输入 

资源 

- [Robustness Principle in Wikipedia](https://en.wikipedia.org/wiki/Robustness_principle)
- [Tolerant Reader](http://martinfowler.com/bliki/TolerantReader.html)

## Inversion of Control

控制反转也被称为好莱坞原则,"不要打电话给我们，我们会打电话给你"。 这是一个设计原则，在这个设计原则中，计算机程序的自定义编写部分从泛型框架中接收控制流。 控制反转具有强烈的内涵，即可重用代码和特定于问题的代码是独立开发的，即使它们在应用程序中一起运行。 

为什么 

- 控制反转被用来增加程序的模块化并使其可扩展 
- 从执行中分离任务的执行。
- 将模块专注于其设计的任务。
- 将模块从假设中解放出来，假设其他系统如何做他们所做的事情，而是依靠合同 
- 在更换模块时防止副作用 

怎么做 

- 使用工厂模式 
- 使用服务定位器模式 
- 使用依赖注入 
- 使用上下文查找 
- 使用模板方法模式 
- 使用策略模式

资源 

- [Inversion of Control in Wikipedia](https://en.wikipedia.org/wiki/Inversion_of_control)
- [Inversion of Control Containers and the Dependency Injection pattern](https://www.martinfowler.com/articles/injection.html)

## Maximise Cohesion

单个模块 / 组成部分的凝聚力是其责任形成有意义的单位的程度; 更高的凝聚力更好。 

为什么 

- 增加了对模块的理解 
- 维护系统的难度增加，因为域的逻辑变化会影响多个模块，而且一个模块的更改需要相关模块的更改 
- 由于大多数应用程序不需要模块提供的随机操作集，因此在重用模块时遇到的困难增加了 

怎么做 

- 与群组相关的功能分担单一责任(例如在类中) 

资源 

- [Cohesion](http://en.wikipedia.org/wiki/Cohesion_(computer_science))
- [Coupling And Cohesion](http://c2.com/cgi/wiki?CouplingAndCohesion)

## Liskov Substitution Principle

LSP 是关于对象的预期行为：

> 程序中的对象应该被替换为其子类型的实例，而不改变该程序的正确性。 

资源 

- [Liskov substitution principle](http://en.wikipedia.org/wiki/Liskov_substitution_principle)
- [Liskov Substitution Principle](http://www.blackwasp.co.uk/lsp.aspx)

## Open/Closed Principle

软件实体(例如类)应开放供扩展，但不得作出修改。 也就是说，这样的实体可以在不改变其源代码的情况下对其行为进行修改。 

为什么 

- 通过最小化对现有代码的更改来提高可维护性和稳定性 

怎么做 

- 写可以扩展的类(而不是可以修改的类) 
- 只暴露需要改变的移动部分，隐藏其他的一切 

资源 

- [Open Closed Principle](http://en.wikipedia.org/wiki/Open/closed_principle)
- [The Open Closed Principle](https://8thlight.com/blog/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html)

## Single Responsibility Principle

一个类不应该有一个以上的理由去改变。 

长版： 每个类都应该有一个单一的责任，这个责任应该完全由类来封装。 责任可以被定义为一个改变的理由，所以一个类或模块应该有一个，也只有一个理由去改变。 

为什么 

- 可维护性：仅在一个模块或类中才需要更改。

怎么做 

- 应用[卷曲定律](http://java-design-patterns.com/principles/#Curly-s-Law)。

资源 

- [Single responsibility principle](http://en.wikipedia.org/wiki/Single_responsibility_principle)

## Hide Implementation Details

软件模块通过提供接口来隐藏信息(即实现细节) ，而不会泄漏任何不必要的信息。 

为什么 

- 当实现发生变化时，客户端使用的接口不必改变 

怎么做 

- 尽量减少类和成员的可访问性 
- 不要在公开场合公开会员数据 
- 避免将私有实现细节放到类的接口中 
- 减少耦合以隐藏更多的实现细节 

资源 

- [Information hiding](http://en.wikipedia.org/wiki/Information_hiding)

## Curly's Law

Curly的法则是关于为任何特定的代码选择一个单一的，明确定义的目标: 做一件事。 

- [Curly's Law: Do One Thing](http://blog.codinghorror.com/curlys-law-do-one-thing/)
- [The Rule of One or Curly’s Law](http://fortyplustwo.com/2008/09/06/the-rule-of-one-or-curlys-law/)

## Encapsulate What Changes

一个好的设计可以识别最有可能改变的热点，并将它们封装在一个 API 之后。 当一个预期的变化发生时，修改会保持在本地。 

为什么 

- 当发生更改时，最大限度地减少所需的修改 

怎么做 

- 封装在 API 背后的不同概念 
- 可能将不同的概念分离成自己的模块 

资源 

- [Encapsulate the Concept that Varies](http://principles-wiki.net/principles:encapsulate_the_concept_that_varies)
- [Encapsulate What Varies](http://blogs.msdn.com/b/steverowe/archive/2007/12/26/encapsulate-what-varies.aspx)
- [Information Hiding](https://en.wikipedia.org/wiki/Information_hiding)

## Interface Segregation Principle

将臃肿接口减少为多个更小和更具体的客户端特定的接口。 接口应该更依赖于调用它的代码而不是实现它的代码。 

为什么 

- 如果一个类实现了不需要的方法，那么调用者需要知道该类的方法实现。 例如，如果一个类实现了一个方法，但只是简单地抛出，那么调用者需要知道这个方法实际上不应该被调用 

怎么做 

- 避免胖接口。类不应该实施违反[单一责任原则](http://java-design-patterns.com/principles/#single-responsibility-principle)的方法。

资源 

- [Interface segregation principle](https://en.wikipedia.org/wiki/Interface_segregation_principle)

## Boy-Scout Rule

美国童军有一个简单的规则，我们可以把它应用到我们的职业中:"让营地比你发现的更干净"。 童子军规则规定，我们应该总是留下比我们发现的更清洁的代码。 

为什么 

- 当改变现有的代码库时，代码的质量往往会下降，累积的技术债务。 遵循童子军的规则，我们应该注意每个承诺的质量。 技术债务受到持续重构的抵制，无论其规模有多小 

怎么做 

- 每一个提交都要确保它不会降低代码质量 
-  任何时候，当有人看到一些不够清晰的代码时，他们应该抓住机会在那里修正它 

资源 

- [Opportunistic Refactoring](http://martinfowler.com/bliki/OpportunisticRefactoring.html)

## Command Query Separation

命令查询分离原则指出，每个方法都应该是执行一个操作的命令，或者是一个向调用者返回数据的查询，而不是两种方法。 提出问题不应该改变答案。 

有了这个原则，程序员可以更加自信地编写代码。 查询方法可以在任何地方和任何顺序使用，因为它们不会突变状态。 有了命令，你必须更加小心。 

为什么 

- 通过将方法清晰地分离为查询和命令，程序员可以在不知道每个方法的实现细节的情况下，以额外的信心编写代码 

怎么做 

- 将每个方法作为查询或命令来实现 
- 将变数命名原则应用于方法名称，这意味着该方法是查询还是命令 

资源 

- [Command Query Separation in Wikipedia](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)
- [Command Query Separation by Martin Fowler](http://martinfowler.com/bliki/CommandQuerySeparation.html)

