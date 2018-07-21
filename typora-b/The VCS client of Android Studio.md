# Android Studio 版本控制

> 原文 (saulmm.github.io)：[The VCS client of Android Studio](http://saulmm.github.io/vcs-android-studio)
>
> 作者：[saulmm](http://saulmm.github.io)

[TOC]

现在，我们无法想象在没有[版本控制系统](https://en.wikipedia.org/wiki/Version_control)的情况下开始一个新的软件项目。

然而，这种工具所需要的学习曲线往往非常高，因为版本控制系统所包含的各种各样的特性和选项使它同时变得复杂和强大 

对于大多数实验开发者和初学者来说，使用好的工具或足够好的定制环境，以便以有效和快速的方式处理 VCS 非常重要。

## 比较

在使用版本控制系统的好处之间，一个强大的好处是随着时间的推移，将文件与各自的状态进行比较。 为此，[JetBrains](https://www.jetbrains.com/) 包含了最强大和直观的 diff / merge 工具之一。 

使用这些工具可以加快日常工作，因为这些比较在修改代码(添加、删除或更改行)时，除了审查之外，这些比较是经常发生的。 此外，这些修改通常很难表现出来，所以很难找到一个很好的工具来清楚地显示这些变化。 

## IntelliJ 作为一个差异，合并工具

从2016年版本的 IntelliJ IDEA 开始，JetBrains 包含了一个叫做[创建命令行启动器](https://www.jetbrains.com/help/idea/2016.2/running-intellij-idea-as-a-diff-or-merge-command-line-tool.html)的有趣功能，Android Studio 在2.2版本中继承了这个功能。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowug568gj317c0qsn6s.jpg)

这个新功能允许使用 IntelliJ 或 Android Studio 外部使用的 diff / merge 工具。 这样我们就可以从命令行或者我们最喜欢的第三方 VCS  客户端使用它。 

![](http://saulmm.github.io/resources/cvs-studio/external_diff.gif)

因此，在其他有趣的情况中，我们可以用这些外部工具来解决冲突或者比较不同的文件，这真的很方便。 作为替代品，像 [Kaleidoscope](http://www.kaleidoscopeapp.com/) 或 [Filemerge](https://developer.apple.com/xcode/features/) 这样的工具也是为了这个目的。 

我通常使用的 VCS 外部客户端是 [Atlassian](https://www.atlassian.com/) 的 [SourceTree](https://www.sourcetreeapp.com/)。要将 Android Studio 差异/合并工具设置为SourceTree 的默认工具，我们只需要在 SourceTree Settings / Diff 的 settings 部分设置 IDE 创建的二进制文件（默认为/ user / local / bin / studio），将外部工具设置为其他参数。

![](http://saulmm.github.io/resources/cvs-studio/sourcetree_conf.png)

## 将当前文件与分支进行比较

IntelliJ IDEA 和 Android Studio 允许将编辑器中的当前打开的文件与其他分支中的各自版本进行比较。这在审查代码，验证冲突是否得到妥善解决等各种情况下可能会很有意思。

这个特性，被称为“与分支比较”，可在 VCS 菜单中找到，或者使用查找操作（⌘+ A）键入部分功能名称。

在选择 VCS 中存在的任何分支（包括本地版本和远程版本）之后，立即会显示一个差异工具以比较分支之间的变化。

![](http://saulmm.github.io/resources/cvs-studio/compare_with_branch.gif)

## 与之前的提交进行比较

这个特性将允许我们使用一个对话框，选择之前做出的提交，以便比较从该点到文件当前状态所做的更改。 

![](http://saulmm.github.io/resources/cvs-studio/compare_with-history.gif)

相反，如果我们想要显示当前文件已经改变的提交历史，我们可以使用特性 Show history ... 它将使用一个在 IDE 底部的固定面板来实现这个目的。 

## 处理更改

通常，在修改了一些文件之后，可能有助于检查从最后一个提交的提交中做了哪些修改(这些修改还没有发生)。 特性局部更改将把焦点放在底部面板中，显示所有未实现的本地修改。 

在本地更改面板中，我们可以不用离开键盘就可以执行一些功能: 

### 显示 diff 工具与从最新提交的提交的变更（⌘+ D）

![](http://saulmm.github.io/resources/cvs-studio/local_changes_diff.gif)

## 恢复更改 - ⌥+⌘+ Z

此外，这个特性可以在日志之外的更多上下文中使用，在任何文件上使用 ⌥+⌘+ Z，将显示恢复对话框，允许从最新的提交中恢复最新的修改。 

![](http://saulmm.github.io/resources/cvs-studio/local_changes_revert.gif)

注意: 通常情况下，intellij / android Studio 中显示的对话框可以通过快捷键 tab + enter 来关闭。 

## VCS 的日志

尽管如此，就我个人而言，我更喜欢第三方 VCS 客户端，比如 SourceTree (它有一个很好的显示日志的方式) ，Android Studio 和 IntelliJ 也实现了这一功能，可以通过在 VCS 菜单下找到操作 Show VCS log 。 

![](http://saulmm.github.io/resources/cvs-studio/vcs_log.gif)

作为优势，集成在 IDE 中，它利用了 IntelliJ 的所有导航技能。 因此，我们可以用一种非常简单和快速的方式与日志交互，使键盘发射不同的动作。 

如果我们选择某个提交，我们可以通过找到动作来访问一些有趣的特性（⇧+⌘+ A）来访问与所选提交相关的有趣功能，如 resets ，checkout ，diffs  使用（⌘+ D）等。

![](http://saulmm.github.io/resources/cvs-studio/vcs_log_diff.gif)

## VCS 操作弹出（⌥+ V）

我们可以显示一个对话框，其中使用了 VCS 中最常用的操作：commit，branches，stash 等。使用快捷键（⌥+ V）。

![](http://saulmm.github.io/resources/cvs-studio/vcs_dialog.gif)

## 提交

正确使用 VCS 的关键要求之一是非常频繁地执行提交。当所有的修改与语义上的一个改变相关时，提交被认为是原子的。另外，一个很好的提交应该用清晰，简洁和描述性的标题来表达正在做的事情。

但是，在使用第三方 VCS 或自定义命令行不佳的环境中，它可能很麻烦，执行提交的方式很快。 

## 快速提交 - （⌘+ K）+（tab + enter）

使用集成在 Android Studio 或 IntelliJ 中的 VCS 客户端，利用快捷方式（⌘+ K）+（⇥+ enter），我们可以在不到5秒的时间内完成提交。此外，还有一些有趣的功能，我们可以在提交之前启用，如代码检查，组织导入等。

![](http://saulmm.github.io/resources/cvs-studio/quick_commit.gif)

## GitHub 集成

IntelliJ，Android Studio 以及其他 JetBrains 环境中，我们可以利用与 GitHub 的集成，在设置凭据后，我们将能够使用一些有用的功能与社交编码平台进行交互。

获取直接的网络链接到存储库

无论是从文件资源管理器还是编辑文件，功能打开 GitHub，将打开一个浏览器与该文件的远程版本。而且，如果我们通过选择文本来运行这个功能，相同的文本将在使用 GitHub 版本创建的链接中突出显示。

![](http://saulmm.github.io/resources/cvs-studio/github_open.gif)

创建合并请求

有一种方法可以直接从编辑器创建 pull 请求，使用 Create Pull Request 功能（通常可以通过 find 操作访问），IDE 将提示一个对话框询问基本分支和合并候选者。 填写表格后，GitHub 会创建一个拉取请求。

![](http://saulmm.github.io/resources/cvs-studio/github_pr.gif)

创造要点

同样的，我们可以在 GitHub 上使用功能 Create [Gist](https://gist.github.com/) 在 Android Studio 上创建私有的，匿名的和公共的 Gist。

![](http://saulmm.github.io/resources/cvs-studio/github_gist.gif)

## 阅读

[How to write a commit message](http://chris.beams.io/posts/git-commit/) - Chris Beams.

[Version Control Help](https://www.jetbrains.com/help/idea/2016.2/version-control.html) - IntelliJ IDEA

