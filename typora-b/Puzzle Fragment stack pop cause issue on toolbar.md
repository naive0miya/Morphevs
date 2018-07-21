# 难题:导致工具栏上片段堆栈弹出问题的原因

> 原文 (Medium)：[Puzzle: Fragment stack pop cause issue on toolbar](https://medium.com/@elye.project/puzzle-fragment-stack-pop-cause-issue-on-toolbar-8b947c5c07c6)
>
> 作者：[Elye](https://medium.com/@elye.project?source=post_header_lockup)

[TOC]

使用片段，使用工具栏是非常普遍的，而且直到这个星期之前我都不期望找到任何问题。我面对这个看似奇怪的问题，发现它的原因花了我大约2天。如果你曾经看到过这样的行为，分享这些信息可以让你更早地了解它。 

另外我还有一个相关的谜题供你解决😉。

## 背景

我有一个应用程序，在活动中有一个布局容器。容器占据了整个活动的宽度，并且它被片段或一组片段充满。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frowjxzih6j30dc09qaaf.jpg)

## 在每个片段中设置工具栏

在我的应用程序中，每个片段都有它们各自的工具栏和它的菜单。由于没有通用的工具栏，因此它被设置在片段中而不是活动中。代码如下。

```kotlin
(activity as AppCompatActivity).setSupportActionBar(toolbar)
```

在我的所有工具栏中，我都有一个工具菜单，可以让你点击。

```kotlin
override fun onOptionsItemSelected(item: MenuItem): Boolean {
    if (item.itemId == android.R.id.home) {
        activity?.onBackPressed()
        return true
    } else {
        // Do some other things to other menu
        return super.onOptionsItemSelected(item)
    }
}
```

在我的活动中， onBackPressed ( ) 弹出片段。

```kotlin
override fun onBackPressed() {
    if (supportFragmentManager.backStackEntryCount > 0) {
        supportFragmentManager.popBackStackImmediate()
    } else {
        super.onBackPressed()
    }
}
```

## 添加和替换片段混合使用

此外，在某些情况下，我使用 replace 将我的片段添加到堆栈，有时我使用 add 将我的片段添加到堆栈。

```kotlin
private fun addFragment(fragment: Fragment, name: String) {
    supportFragmentManager.beginTransaction()
            .add(R.id.container, fragment, name)
            .addToBackStack(name).commit()
}

private fun replaceFragment(fragment: Fragment, name: String) {
    supportFragmentManager.beginTransaction()
            .replace(R.id.container, fragment, name)
            .addToBackStack(name).commit()
}
```

> 注意：使用替换片段优于使用添加，因为它不会在内存中存储不必要的片段，并且还减少了 UI 渲染需求。但是，如果当前片段包含 webview，要打开下一个片段，则使用 add 更好，当你弹出那个"下一个"片段时，你现有的带有 webview 的片段是保留的，不需要重新加载。

## 应用程序示例

这就是我的应用程序行为。为了更好地说明这一点，我创建了一个有8个按钮的应用程序，4个按钮是添加1-4的片段，4个按钮用片段1-1来替换。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowk1t2h0j308w0foaaj.jpg)

代码可以在下面访问。[[PR](https://github.com/elye/issue_android_fragment_replace_add_replace)]

## 问题说明

上面的应用程序，当你点击任何按钮时，一个片段将被添加或替换到活动容器。对于其中任何一个(添加或替换) ，它们都是在 back-stack 中的跟踪，这样用户就可以以相反的顺序将它们返回。 

为了更好地说明这一点，您可以单击。

### 添加4个片段

添加片段1→添加片段2→添加片段3→添加片段4，然后在容器中堆叠4个片段。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1frowk4gwqoj30c209qjru.jpg)

当您按下工具菜单上的返回按钮（即左上角的←键）时，每个片段都会弹出，直到达到原始状态（没有片段）。

这一切都很好。 

另一种情况是。

### 替换4个片段

用片段1替换→用片段2替换→用片段3替换→用片段4替换，那么你就会得到一个片段，即剩余的片段4。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowk6r43pj30f009ojrw.jpg)

当您按下工具菜单上的后退按钮（即左上角的←键）时，容器中的片段将弹出，替换为之前的片段，一次一个。所有这些都是好的，因为我们可以到达原来的状态，那里的容器里没有片段。 

## 问题 - 混合添加和替换到堆栈中

当混合添加和替换时，有各种可能的组合。 在其中的一些组合中，当我们开始弹出片段时，它会在某一点上卡住(也就是说，在左上方按下← key，不会弹出片段💥)。 

> 解决方法是，使用硬件键返回，仍然可以弹出片段。只是工具栏←不再起作用。奇怪的行为。

让我选择一些组合，并告诉你哪个是好的，哪个不是（我将缩写添加片段 x 作为添加 x 并将片段 x 替换为 Rep x，然后按←以 pop 弹出）。

基于这个结果，看看你是否能够发现导致这个问题的组合模式？ 

1. `Rep 1` → `Rep 2` → `Add 3` → `Add 4` : `Pop` 四个片段都没问题 
2. `Add 1` → `Rep 2` → `Add 3` → `Add 4` : `Pop` 四个片段都没问题 
3. `Add 1` → `Rep 2` → `Rep 3` → `Rep 4` : `Pop` 四个片段都没问题 
4. `Add 1` → `Rep 2` → `Rep 3` → `Add 4` : `Pop` 四个片段都没问题 
5. `Add 1` → `Add 2` → `Rep 3` → `Add 4` : `Pop` 卡在片段 1 💥
6. `Rep 1` → `Add 2` → `Rep 3` → `Add 4` : `Pop` 卡在片段 1 💥
7. `Rep 1` → `Add 2` → `Add 3` → `Rep 4` : `Pop` 卡在片段 2 💥

你能从上面发现造成问题的模式吗？ 把它当做一个小小的解谜题。 

如果你认为上面给出的组合是不够的，为了产生更多的问题的模式，你可以编译上面给出的代码，然后测试出来。看看你能否找出问题所在。 

> 提示，当你使用工具栏上的←按钮弹出失败时，你可以使用硬件后退按钮，它应该仍然能够弹出它。

不要担心，如果你不能找出问题所在，答案在下面。

## 演示说明

无论如何，对于那些喜欢阅读的人来说，更清楚地说明这个问题。 下面的视频展示了3个场景 

1. `Add 1` → `Add 2` → `Add 3` → `Add 4` : 所有 `pop` 很好。 （注意数字重叠，因为背景是透明的，显示每个片段相互之间增加）
2. `Rep 1` → `Rep 2` → `Rep 3` → `Rep 4` : 所有 `pop` 很好。（该数字不再重叠，因为片段被替换）
3. `Add 1` → `Rep 2` →`Add 3` → `Rep 4` :   `pop` 卡在片段 2💥。为什么？！！？

![](https://cdn-images-1.medium.com/max/800/1*SuNvR_z3AwZLu4BWcfBoUw.gif)

## 令人不安的模式 - 解答这个难题

经过一番调查后，发现造成这个问题的麻烦模式。

这个模式是针对每一个已经显示在那之前的片段的添加和替换，弹出将停留在该片段。 例如: 

`Add or Rep 1` → `Add 2` → `Rep 3`. 然后，`pop` 卡在 1。

您可以恢复给出的上述8种组合，并且确认这种模式是在引起问题的4种组合中显示出来，而不是在其他4个组合中。 

## 深入研究这个问题

鉴于这种令人不安的模式，现在可以更容易地使用重现问题的最小步骤来重复这个问题。 让我们使用下面的流程。 

`Rep 1` → `Add 2` → `Rep 3`

为了深入研究，我们来追踪片段堆栈是如何工作的。 

步骤1：Rep 1 ：用片段1替换空容器。下图显示了该图。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frownt2ojyj30ci0923ym.jpg)

步骤2：Add 2：在片段1的顶部添加片段2。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frowovg9ssj30bi09cmxe.jpg)

步骤3：Rep 3：用片段3替换容器。然后，将片段1和2从容器中推出。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1frowoxv7guj30f009idg8.jpg)

步骤4：Pop 3：按下左上角的←键，弹出 Fragment 3。当片段3弹出时，片段1和片段2被系统带回到容器中。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1frowp01l0cj30bi09cmxe.jpg)

步骤5：Pop 2：再次按左上方的←键，弹出 Fragment 2。片段1仍然保留在容器中。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1frowp2c6haj30ci0923ym.jpg)

步骤6：Pop 1💥：再次按左上方的←键，希望弹出片段1，但没有任何反应？ 💥。左上方的←键不再有效！为什么？

## 问题的罪魁祸首

显然，这个问题的罪魁祸首不在第六步。 触发点发生在上面的步骤4。 这时，片段1和片段2被恢复到容器中。 

当时，这两个碎片被重新创建出来。 当两个片段被重新创建时，两个片段都调用到 

```kotlin
(activity as AppCompatActivity).setSupportActionBar(toolbar)
```

理想情况下，这应该是可行的，因为调用的序列仍然是正确的，片段1首先被恢复，接着是片段2。 然而，由于某种原因，由于两者的调用时间几乎相当接近，片段1的工具栏不再工作了。 

> 我高度怀疑这是 setSupportActionBar上 SDK 库的问题。我通过 [issues](https://issuetracker.google.com/issues/73793626) 向 Google 提交了一个问题。

## 问题的解决方法

鉴于这在我看来像是一个谷歌错误，我不认为我有一个很好的解决方案。 我也不能等待谷歌，因为到目前为止基于经验，它需要一个月的时间来解决所报道的问题，除非这个问题非常明显，影响到每一个人。 

所以我必须解决。

从应用程序的角度来看，使用硬件返回按钮仍然能够触发 onBackPressed ( ) 在活动中的功能，因此弹出片段仍然是可能的。 但是用户真的希望按下左上角的← key 来弹出堆栈。 

## 使工具栏生效的代码

我还发现，如果片段2发生了变化，如果我能让片段1再次调用下面，工具栏就会恢复生效。 

```kotlin
(activity as AppCompatActivity).setSupportActionBar(toolbar)
```

但是，当片段2被弹出时，片段1中的代码将不会触发，因为片段1在容器中已经存在。

## 可行的解决方法

有了上述的想法，我使用的方法是，每当一个弹出发生时，让顶层的片段调用这个 

```kotlin
(activity as AppCompatActivity).setSupportActionBar(toolbar)
```

这将确保顶级片段始终具有它的工具栏重置和生存。 

为了检测片段弹出，我在活动中使用下面的方法，调用顶部片段的 onResume。 

```kotlin
supportFragmentManager.addOnBackStackChangedListener {
    val currentFragment = supportFragmentManager
                            .findFragmentById(R.id.container)
    currentFragment?.onResume()
}
```

> 注意 onBackStackChangedListener 获取片段的 push 和 pop 两个触发器。我没有在那里区分他们。

对于所有的片段，我明确地做出如下的 onResume。

```kotlin
override fun onResume() {
    super.onResume()
    (activity as AppCompatActivity)
                            .setSupportActionBar(toolbar_actionbar)
}
```

使用这些代码时，无论使用添加还是替换，都可以使用工具栏上的←按钮随时弹出片段中的所有按钮和弹出窗口，而不管使用添加还是替换。[[PR](https://github.com/elye/issue_android_fragment_replace_add_replace_workaround)]

我认为我的解决方法也不理想。但至少它能让事情奏效。如果你知道这个问题有更好的解决方案，请随时分享。你也可以将它贴在下面的 stackoverflow 上。

[Replace, add, replace fragment, then the Navigation Back button not working for first fragment Join Stack Overflow to learn, share knowledge, and build your career. I have a fragment stack, where I use replace and… stackoverflow.com](https://stackoverflow.com/questions/48941001/replace-add-replace-fragment-then-the-navigation-back-button-not-working-for)

如果没有，真诚地希望 Google 能够通过 [issues](https://issuetracker.google.com/issues/73793626) 报告解决此问题的一些解决方案。

希望这篇文章是有帮助的，你很欣赏它。请与他人分享。



