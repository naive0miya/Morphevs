# 探索 Java 隐藏的开销

> 原文：[探索 Java 隐藏的开销](https://academy.realm.io/cn/posts/360andev-jake-wharton-java-hidden-costs-android/)
>
> 作者：[Jake Wharton](https://twitter.com/jakewharton)

[TOC]

随着 Android 引入 Java 8 的一些功能，请记住每一个标准库的 API 和语言特性都会带来一些相关的开销，这很重要。虽然设备越来越快而且内存越来越多，代码大小和性能优化之间仍然是有着紧密关联的。这篇 [360AnDev](https://360andev.com/) 的演讲会探索一些 Java 功能的隐藏开销。我们会关注**对库开发者和应用开发者都有关系的优化**和**能够衡量它们影响的工具**。

------

## 介绍 [(0:00)](javascript:presentz.changeChapter(0,0,true);)

在这篇演讲里面，我将讨论我近六个月以来一直在探索的事情，而且我想披露一些信息。随着你的深入了解，你可能得不到一些明确的能够应用在你的应用程序上的东西。但是，到结束的时候，我会有一些具体的技巧来展示如何避免我今天讲的这些开销。我也会展示许多我使用的命令行工具，这些资源的链接都在文章结束的地方。

## Dex 文件 [(1:14)](javascript:presentz.changeChapter(0,1,true);)

我们将从一个多项选择的问题开始。下面这段代码有多少个方法？没有，一个或者两个？

```java
class Example {
}
```

你可能马上就有直觉的反应了。也许没有，也许一个，也许两个。让我们看看我们是不是能回答这个问题。首先，类里面没有方法。我在源文件里面没有任何方法，所以看起来可以这么说。当然，这样的答案真的没有什么意思。让我们开始把我们的类在 Android 里编译一下，看看会发生什么：

```shell
$ echo "class Example {
}" > Example.java

$ javac Example.java

$ javap Example.class
class Example {
Example();
}
```

我们把内容写到一个文件里面，然后用 Java 编译器编译源代码然后把它变成 class 文件。我们可以使用其他的非 Java 开发套件的工具，它叫做 `javap`。这使得我们能够深入了解编译出来的 class 文件。如果在我们编译的 class 文件上运行它，我们能看到我们的例子的 class 里面有一个构造函数。我们没有在源文件里面编写它，但是 Java C 决定自动增加一个那样的构造函数。这意味着源文件里面没有方法，但是 class 文件里面有一个。但这不是 Android 编译停止的地方：

```shell
$ dx --dex --output=example.dex Example.class

$ dexdump -f example.dex
```

在 Android SDK 中，有一个工具叫做 `dx`，它完成 *dexing*，这使得 Java class 文件变成 Android Dalvik 二进制码。我们通过 `dex` 运行我们的例子，Android SDK 里面还有另外一个工具叫做 `dexdump`，这个工具会给我们一些关于 dex 文件内部的信息。你运行它，它会打印一串东西。它们是文件的偏移量和计数器还有各种表。如果我们详细点看看，一个明显的事情是，dex 文件里面有一个函数列表：

```shell
method_ids_size : 2
```

它说我们的 class 里面有两个方法。这说不通。不幸的是，`dexdump` 并没有给我一个简单的方法来了解这两个方法是什么。因为如此，我写了一个小工具来输出 dex 文件里面的方法：

```shell
$ dex-method-list example.dex
Example <init>()
java.lang.Object <init>()
```

如果我们这样做，我就能看到它返回了两个方法。它返回了我们的构造函数，我们知道它是 Java 编译器创建的，虽然我们没有去写它。但是它还说有一个对象构造函数。当然，我们的代码没有四处调用 new 对象，所以这个方法是哪里产生的呢，然后又在 dex 文件里面引用的呢？如果我们回到能打印 class 文件信息的 `javap` 工具，你能通过一些额外的标志来找到 class 里面的深度信息。我将使用 `-c`，这会把二进制代码反编译成可读的信息。

```shell
$ javap -c Example.class
class Example {
    Example();
        Code:
            0: aload_0
            1: invokespecial #1 //java/lang/Object."<init>":()V
            4: return
}
```

在索引 1 处，是我们的对象构造函数，它被父类的构造函数调用。这是因为，即使我们不声明它，`Example` 也是继承于 `Object` 的。每一个构造函数都会调用它的父类的构造函数。它是自动插入的。这意味着我们的 class 流中有两个方法。

所有这些关于我的初始问题的答案都是对的。区别就是术语不同。这是真实的情况。我们没有定义任何方法。但是只有人类关心它。作为人类，我们读写这些源文件。我们是唯一关心它们内部构造的人。另外两个方法更重要，方法的个数实际上是编译进 class 文件里面了。无论是否声明，这些方法都在 class 的内部。

这两个方法是引用方法的数目。它和我们自己编写的方法的计数是类似的，和所有其他在函数里引用以及 Android logger 函数的调用也差不多。我这里引用的 `Log.d` 函数和这个引用方法计数不一样，因为这是我们在 dex 文件里面的计数。这也是人们经常在 Android 里面讨论方法计数时的常用方法，因为 dex 有着声名狼藉的对于引用方法的个数的限制。

我们看到一个没有声明的构造函数被创建了，所以让我们看看其他自动生成的，我们可能不知道的隐藏开销。嵌套类是一个有用的例子：

```java
// Outer.java
public class Outer {
    private class Example {
    }
}
```

Java 1.0 不支持这样做。它们是在晚些的版本里才出现的。当你在一个视图或者展示层里面定义适配器的时候，你能看到这样的东西。

```java
// ItemsView.java
public class ItemsView {
    private class ItemsAdapter {
    }
}

$ javac ItemsView.java

$ ls
ItemsView.class
ItemsView.java
ItemsView$ItemsAdapter.class
```

如果我们编译这个 class，这是一个有两个 class 的文件。一个嵌套在另一个里面。如果我们编译它，我们能在文件系统中看到两个独立的 class 文件。如果 Java 真的有内嵌类，我们就应该只能看到一个 class 文件。我们会得到 `ItemsView.class`。但是这里 Java 没有真正的嵌套，那么这些类文件里面是什么呢？在这个 `ItemsView` 里面，外层类，我们有的只是构造函数。这里没有引用，没有内嵌类的任何迹象：

```shell
$ javap 'ItemsView$ItemsAdapter'
class ItemsView$ItemsAdapter {
    ItemsView$ItemsAdapter();
}
```

如果我们看看嵌套类的内容，你可以看到它有隐式的构造函数，而且你知道它在外部类的里面，因为它的名字被扰乱了。另外一个重要的事情是如果我返回去，我能看到这个 `ItemsView` 类是公共的，这和我们在源文件里面定义的一样。但是内部类，内嵌类，虽然它定义为私有的，在类文件里面它不是私有的。它是包作用范围的。这是对的，因为我们在同一个包中有两个生成的类文件。重申一次，这进一步证明了在 Java 里面没有真正的内嵌类。

```java
// ItemsView.java

public class ItemsView {
}

// ItemsAdapter.java

class ItemsAdapter {
}
```

虽然你内嵌了两个类的定义，你可以有效地创建两个类文件，它们在同一个包里紧邻着对方。如果你想这样做的话，你可以实现。你可以作为两个独立的文件使用命名规则：

```java
// ItemsView.java

public class ItemsView {
}

// ItemsView$ItemsAdapter.java

class ItemsView$ItemsAdapter {
}
```

美元符在 Java 里面是名字的有效字符。对方法或者附加名字也有效：

```java
// ItemsView.java
public class ItemsView {
    private static String displayText(String item) {
        return ""; // TODO
    }
private class ItemsAdapter {
    }
}
```

然而，这是真正有意思的地方，因为我知道我能够做一些事情在外部类里找到一个 `private static` 方法，而且我能在内部类里面引用那个私有的方法：

```java
// ItemsView.java

public class ItemsView {
    private static String displayText(String item) {
        return ""; // TODO
    }
}

// ItemsView$ItemsAdapter.java

class ItemsView$ItemsAdapter {
    void bindItem(TextView tv, String item) {
        tv.setText(ItemsView.displayText(item));
    }
}
```

现在我们知道没有真正的内嵌，但是，这在我们假设的独立系统里面是如何工作的呢，这里我们的 `ItemsAdapter` 类需要引用 `ItemsView` 的私有方法？这没有编译，而且它们会被编译：

```java
// ItemsView.java

public class ItemsView {
    private static String displayText(String item) {
        return ""; // TODO
    }

    private class ItemsAdapter {
        void bindItem(TextView tv, String item) {
            tv.setText(ItemsView.displayText(item));
        }
    }
}
```

发生了什么？当你回到我们的工具的时候，我们能再次使用 `javac` 。

```shell
$ javac -bootclasspath android-sdk/platforms/android-24/android.jar \
    ItemsView.java

$ javap -c 'ItemsView$ItemsAdapter'
class ItemsView$ItemAdapter {
    void bindItem(android.widget.TextView, java.lang.String);
    Code:
        0: aload_1
        1: aload_2
        2: invokestatic #3 // Method ItemsView.access$000:…
        5: invokevirtual #4 // Method TextView.setText:…
        8: return
}
```

我在引用 `TextView`，这样我才能在 Java 里面增加 Android APIs。现在我将打印出内嵌类的内容，来看看哪个函数被调用了。如果你看看索引 2，它没有调用 `displayText` 方法。它调用的是 `access$000`，我们没有定义它。它在 `ItemsView` 类里面吗？

```shell
$ javap -p ItemsView123

class ItemsView {
    ItemsView();
    private static java.lang.String displayText(…);
    static java.lang.String access$000(…);
}
```

如果我们仔细看看，是的，它在。我们看到我们的 `private static` 方法仍然在那，但是我们现在需要这个我们没有编写的额外方法自动加入。

```shell
$ javap -p -c ItemsView123

class ItemsView {
    ItemsView();
        Code: <removed>

private static java.lang.String displayText(…);
    Code: <removed>
    
static java.lang.String access$000(…);
    Code:
        0: aload_0
        1: invokestatic #1 // Method displayText:…
        4: areturn
}
```

如果我们看看这个函数的内容，它做的事情就是调用我们原来的 `displayText` 方法。这有意义，因为我们需要一个从包的作用域到类里调用它的私有方法的途径。Java 会合成一个包作用域的方法来帮助实现这个函数调用。

```shell
// ItemsView.java
public class ItemsView {
    private static String displayText(String item) {
        return ""; // TODO
    }

    static String access$000(String item) {
        return displayText(item);
    }
}

// ItemsView$ItemsAdapter.java
class ItemsView$ItemsAdapter {
    void bindItem(TextView tv, String item) {
        tv.setText(ItemsView.access$000(item));
    }
}
```

如果我们回到我们两个类文件的例子，我们手工的例子，我们能让编译器按照同样的方法工作。我们能够增加方法，我们能更新另一个类，然后引用它。dex 文件有方法的限制，所以当你有这些因为你编写源文件的方式的不同，而导致的必须要添加新的的方法加的话，这些函数的个数都是计算在内的。理解这点是很重要的，因为我们尝试在某处访问一个私有成员是不可能的。

## Dex 进阶 [(10:52)](javascript:presentz.changeChapter(0,51,true);)

所以你可能会说，”好吧，你只做了 Java C。也许 dex 工具能看到这些，而且自动地为我们去除这些函数。”

```shell
$ dx --dex --output=example.dex *.class

$ dex-method-list example.dex

ItemsView <init>()
ItemsView access$000(String) → String
ItemsView displayText(String) → String
ItemsView$ItemsAdapter <init>(ItemsView)
ItemsView$ItemsAdapter bindItem(TextView, String)
android.widget.TextView setText(CharSequence)
java.lang.Object <init>()
```

如果我们编译这两个生成的类，然后显示它们，你可以看到实际情况不是这样。dex 工具编译它就好像它是个任意的其他方法一样。在你的 dex 文件里面就这样结束了。

你会说，”好吧，我听说过这个新的 [Jack](https://source.android.com/source/jack.html) 编译器。而且 Jack 编译器直接编译源文件，然后直接产生 dex 文件，所以也许它做了些什么事情使得它不需要产生额外的方法。” 这样肯定没有 access 方法。但是，有一个 `-wrap0` 方法，它实际上做的是同样的事情：

```shell
$ java -jar android-sdk/build-tools/24.0.1/jack.jar \
        -cp android-sdk/platforms/android-24/android.jar \
        --output-dex . \
        ItemsView.java

$ dex-method-list classes.dex

ItemsView -wrap0(String) → String
ItemsView <init>()
ItemsView displayText(String) → String
ItemsView$ItemsAdapter <init>(ItemsView)
ItemsView$ItemsAdapter bindItem(TextView, String)
android.widget.TextView setText(CharSequence)
java.lang.Object <init>()
```

还有一个工具叫做 [ProGuard](https://developer.android.com/studio/build/shrink-code.html)，许多人都用它。你可能会说，”好吧，ProGuard 应该会处理这些事情，对吧？” 我可以写一个快速的 ProGuard key。我能在我的类文件上运行 ProGuard，然后打印这些方法。这里是我得到的东西：

```shell
$ echo "-dontobfuscate
-keep class ItemsView$ItemsAdapter { void bindItem(...); }
" > rules.txt

$ java -jar proguard-base-5.2.1.jar \
    -include rules.txt \
    -injars . \
    -outjars example-proguard.jar \
    -libraryjars android-sdk/platforms/android-24/android.jar

$ dex-method-list example-proguard.jar

ItemsView access$000(String) → String
ItemsView$ItemsAdapter bindItem(TextView, String)
android.widget.TextView setText(CharSequence)
```

构造函数被移出了，因为它们没有被使用。我将把它们加回来因为正常情况下它们是在的：

```shell
$ dex-method-list example-proguard.jar

ItemsView <init>()
ItemsView access$000(String) → String
ItemsView$ItemsAdapter <init>(ItemsView)
ItemsView$ItemsAdapter bindItem(TextView, String)
android.widget.TextView setText(CharSequence)
java.lang.Object <init>()
```

你能看到 access 函数还在那里。但是如果你仔细看看，你保持了 access 方法，但是 `displayText` 消失了。这里发生了什么呢？你可以解压缩 ProGuard 产生的 `jar` 包，然后回到我们的 `javap` 工具，看看 ProGuarded 类文件里面到底有些什么：

```shell
$ unzip example-proguard.jar

$ javap -c ItemsView

public final class ItemsView {
    static java.lang.String access$000(java.lang.String);
        Code:
            0: ldc #1 // String ""
            2: areturn
}
```

如果我们看看 access 函数，它不再调用 `displayText`。ProGuard 把 `displayText` 的内容抽取出来，然后移到 access 函数里面并且删除了 `displayText` 函数。这个 access 函数是我们唯一引用的私有函数，所以它会变成 inline 函数，因为没有其他人使用它了。是的，ProGuard 在某种程度上能够帮得上忙。但是它不保证能够有用。我们很幸运，因为这是一个小例子，但是优化也是不能保证的。你可能会想，”好吧，我真的没有使用那么多的内嵌类，也许有一组。如果我只是会得到一组额外的函数，这关系不大，对吗？”

## 匿名类 [(13:06)](javascript:presentz.changeChapter(0,65,true);)

让我向你介绍一些我们的朋友，匿名类：

```java
class MyActivity extends Activity {
    @Override protected void onCreate(Bundle state) {
        super.onCreate(state);

        setContentView(R.layout.whatever);
        findViewById(R.id.button).setOnClickListener(
            new OnClickListener() {
                @Override public void onClick(View view) {
                    // Hello!
                }
            });
    }
}
```

匿名类的行为几乎和内嵌类完全一样。它们本质上是一样的东西。它是一个内嵌类，但是没有名字。如果在这些监听者里面，这是我们常用的方法，你引用一个类里面的私有方法，这样就会产生一个合成的 access 函数。

```java
class MyActivity extends Activity {
    @Override protected void onCreate(Bundle state) {
        super.onCreate(state);

        setContentView(R.layout.whatever);
        findViewById(R.id.button).setOnClickListener(
            new OnClickListener() {
                @Override public void onClick(View view) {
                    doSomething();
                }
            });
    }
    
    private void doSomething() {
        // ...
    }
}
```

对于成员来说，事实也是这样：

```java
class MyActivity extends Activity {
    private int count;

    @Override protected void onCreate(Bundle state) {
        super.onCreate(state);

        setContentView(R.layout.whatever);
        findViewById(R.id.button).setOnClickListener(
            new OnClickListener() {
                @Override public void onClick(View view) {
                    count = 0;
                    ++count;
                    --count;
                    count++;
                    count--;
                    Log.d("Count", "= " + count);
                }
            });
    }
}
```

我认为这是一个有许多共性的例子。我们在外部类里面有这些成员，我们修改状态的这些 activity 的成员在这些监听者里面。我们做了一个完整的美妙实现，但是我们做的是设置一个值。我使用了前置加加，前置减减，后置加加和后置减减，然而日志消息需要从成员中读取数值。我们在这里的函数有多少个呢？也许只有两个。也许一个读，一个写，然后加加和减减变成读加和写。如果这是事实的话：

```shell
$ javac -bootclasspath android-sdk/platforms/android-24/android.jar \
MyActivity.java

$ javap MyActivity
class MyActivity extends android.app.Activity {
    MyActivity();
    protected void onCreate(android.os.Bundle);
    static int access$002(MyActivity, int); // count = 0    write
    static int access$004(MyActivity);        // ++count         preinc
    static int access$006(MyActivity);        // --count         predec
    static int access$008(MyActivity);        // count++         postinc
    static int access$010(MyActivity);        // count--         postdec
    static int access$000(MyActivity);        // count         read
}
```

我们编译它，然后就为每一个类型都产生了一个函数。所以如果你觉得在一个 activity 或者 fragment 或者其它什么东西，你有四到五个监听者，和大概 10 个在外部类里的私有成员。你就会有一个很棒的 access 方法爆炸。你也许还没有被说服这是个问题。你会说，”好吧，也许是 50，也许是 100.这真的有关系吗？” 我们下面来看看。事实证明一切。

## 现实情况 [(15:03)](javascript:presentz.changeChapter(0,73,true);)

你可以看到在现实中这是多么的普遍。这些命令可以帮你拿出手机上所有的 APK。每一个你安装的应用，都是一个第三方的应用：

```shell
$ adb shell mkdir /mnt/sdcard/apks

$ adb shell cmd package list packages -3 -f \
| cut -c 9- \
| sed 's|=| /mnt/sdcard/apks/|' \
| xargs -t -L1 adb shell cp

$ adb pull /mnt/sdkcard/apks
```

我们可以写一个脚本来使用这些 dex 函数列表，然后 greps 所有的不同数值：

```shell
#!/bin/bash                                                 accessors.sh
set -e

METHODS=$(dex-method-list $1 | \grep 'access\$')
ACCESSORS=$(echo "$METHODS" | wc -l | xargs)
METHOD_AND_READ=$(echo "$METHODS" | egrep 'access\$\d+00\(' | wc -l | xargs)
WRITE=$(echo "$METHODS" | egrep 'access\$\d+02\(' | wc -l | xargs)
PREINC=$(echo "$METHODS" | egrep 'access\$\d+04\(' | wc -l | xargs)
PREDEC=$(echo "$METHODS" | egrep 'access\$\d+06\(' | wc -l | xargs)
POSTINC=$(echo "$METHODS" | egrep 'access\$\d+08\(' | wc -l | xargs)
POSTDEC=$(echo "$METHODS" | egrep 'access\$\d+10\(' | wc -l | xargs)
OTHER=$(($ACCESSORS - $METHOD_AND_READ - $WRITE - $PREINC - $PREDEC - $POSTINC - $POSTDEC))

NAME=$(basename $1)

echo -e "$NAME\t$ACCESSORS\t$READ\t$WRITE\t$PREINC\t$PREDEC\t$POSTINC\t$POSTDEC\t$OTHER"
```

然后我们运行这个疯狂的命令，它会遍历每一个从手机里面获取的 APK，运行这个脚本，然后你得到一个漂亮的报表:

```shell
$ column -t -s $'\t' \
<(echo -e "NAME\tTOTAL\tMETHOD/READ\tWRITE\tPREINC\tPREDEC\tPOSTINC\tPOSTDEC\tOTHER" \
&& find apks -type f | \
xargs -L1 ./accessors.sh | \
sort -k2,2nr)
```

你能在 77 页上看到这个表，它把使用 accessor 函数的包排了个序。在我的手机里面，我有几千个。Amazon 占据了前六名中的五个。前几名有 5000 个合成的 accessor 函数。5000 个函数，那是一整个库了。这好像一个 apk 的 pad。你有一整个 apk 的 pad， 里面都是无用的函数，它们的存在只是为了跳转到其它的函数去。

同样的，因为我们使用了 ProGuard，混淆会使得这些数值变得更难以确认。初始化会搞砸这些数据。不要认为它们是准确的数据。它只是给了你一个大约的数值，让你明白你创造了多少个函数。你的应用里面会有多少函数是比这些无用的 access 函数有用的？顺便说一句，Twitter 在列表的末尾？它们有 1000 个。他们 ProGuarding 了他们的应用，所以可能实际情况更糟。但是我想这很有趣，因为它们有最多的函数，但是报告了最少的 access 函数数量。他们有 171,000 个函数，但是只用了 1,000 个合成的 accessors。这很令人吃惊。

我们能改变这个情况。这是很容易的。我们不需要把一些东西作为私有成员。当我们跨边界引用它们的时候，我们需要让它们成为包作用域级别。 IntelliGate 提供了这样的检查。它默认不起作用，但是你可以进去然后搜索一个私有方法。搜索是一件有意思的事情，如果采用我们的例子，它会将结果标记为黄色高亮。你可以 `选择性进入` ，它会让你跳转到你访问的私有成员那里，然后把它设为包作用域的。

当你考虑这些内嵌类的时候，试着把它们想象成为兄弟姐妹，而不是父子关系。你不能从外部类里面访问一个私有成员。你需要让它成为包级别的。这才不会出问题，因为即使你在编写一个库，这些人也不会在同样的包里面放入类文件，然后访问这些你设为更容易访问的内容。我会在功能需求里面放上一个这方面的 link 检查。希望在未来，你在构建的时候，如果你做了类似的事情，构建就会失败。

我遍历过许多开源库，而且修改过这些可见性的问题，这样这些库本身就不需要在 impose 成百上千的外部函数了。这对于生成代码的库来说尤为重要。我们能在我们的应用里面减少 2700 个函数，只需要改变一个我们代码生成的步骤。只需要把一些东西从私有作用域改到包作用域，就能很轻松地减少 2700 个函数了。

## 合成方法 [(18:45)](javascript:presentz.changeChapter(0,82,true);)

这些合成方法之所以叫做合成方法是因为你没有编写它们。

```java
// Callbacks.java

interface Callback<T> {
    void call(T value);
}

class StringCallback implements Callback<String> {
    @Override public void call(String value) {
        System.out.println(value);
    }
}
```

它们是为你自动生成的。这些 accessor 函数是唯一自动生成的函数。Generics 是另一个在 Java 1.0 之后才出现的东西，而且它必须重新翻译成 Java 的工作方式。我们在库里面，甚至在我们的应用里面能看到许多这样的东西。我们使用这些泛型接口，因为它们非常方便，而且它们让我们能保持类型正确。

```shell
$ javac Callbacks.java

$ javap StringCallback
class StringCallback implements Callback<java.lang.String> {
    StringCallback();
    public void call(java.lang.String);
    public void call(java.lang.Object);
}
```

如果我们有一个函数接收一个泛型值，当你编译它的时候，你会发现所有接收泛型参数的函数最终会变成两个。一个接收串，不论你的泛型是什么，另一个接收对象。这和 erasure 很像。你听到许多人都谈论 erasure，你也许不理解到底发生了些什么。这更像一个 erasure 的说明。我们必须产生使用对象的代码，因为当你访问这个泛型函数的时候，这就是你最终调用的函数。

```shell
$ javap -c StringCallback
class StringCallback implements Callback<java.lang.String> {
    StringCallback();
        Code: <removed>

    public void call(java.lang.String);
        Code: <removed>

    public void call(java.lang.Object);
        Code:
        0: aload_0
        1: aload_1
        2: checkcast #4 // class java/lang/String
        5: invokevirtual #5 // Method call:(Ljava/lang/String;)V
        8: return
}
```

如果我们看看那个额外的函数里面发生了什么，它只有一个转换。它强转成了 year 类型然后它调用了接收泛型的真实实现。任何调用这个函数的人都会使用这个对象函数。调用代码会传入它们有的任何对象，然后这部分代码就完成强制转换，调用真正的实现。每一个使用泛型的函数，最后都会成为两个函数。

```shell
// Providers.java

interface Provider<T> {
    T get(Context context);
}

class ViewProvider implements Provider<View> {
    @Override public View get(Context context) {
        return new View(context);
    }
}
```

返回值也是一样。如果你有一个返回泛型的函数，你将会看到一样的事情。

```shell
$ javac -bootclasspath android-sdk/platforms/android-24/android.jar \
    Example.java

$ javap -c ViewProvider
class ViewProvider implements Provider<android.view.View> {
    ViewProvider();
        Code: <removed>

    public android.view.View get(android.content.Context);
        Code: <removed>

    public java.lang.Object get(android.content.Context);
        Code:
            0: aload_0
            1: aload_1
            2: invokevirtual #4 // Method get:(…)Landroid/view/View;
            5: areturn
}
```

产生了两个函数。一个作为返回值。在这种情况下，是我们的视图。然后在底部，返回了对象。这个函数非常简单因为它不用完成任何事情，它仅仅是接收 view 然后把它变成对象。

```java
// Providers.java

interface Provider<T> {
    T get(Context context);
}

class ViewProvider implements Provider<View> {
    @Override public View get(Context context) {
        return new View(context);
    }
}
```

另一个需要指出的并且许多人都没有意识到的问题是，这是 Java 语言的功能。

```java
class ViewProvider implements Provider<View> {
    @Override public View get(Context context) {
        return new View(context);
    }
}

class TextViewProvider extends ViewProvider {
    @Override public TextView get(Context context) {
        return new TextView(context);
    }
}
```

如果你有需要重载的函数，你可以改变它的返回值成为更具体的类型。这叫做协变返回类型。不一定和我们这个例子里面的一样，我们在实现接口。基类不需要是一个接口或者其它的东西。你可以从基类重载一个方法。你可以把它的返回类型变成其他更具体的类型。

你这样做的原因是如果你在我们第二个类里面有其他方法，它又要调用这个 `get` 函数的话，你就会这样做了，因为它们需要实现的类型。不需要更宽泛的类型了，在这个例子里面，是 `View` 类型。它们完全可以只做对于 `TextView` 的定制化，因为他们已经在那个类里面了。

## 协变返回类型 [(21:58)](javascript:presentz.changeChapter(0,94,true);)

协变返回类型。我肯定你们能猜到这里面发生了什么。

```shell
$ javap TextViewProvider

class TextViewProvider extends ViewProvider {
    TextViewProvider();
    public android.widget.TextView get(android.content.Context);
    public android.view.View get(android.content.Context);
    public java.lang.Object get(android.content.Context);
}
```

我们有了另一个函数。在这个例子里面，它既是泛型而且还有协变返回类型。我们把一个函数变成了三个基本上不做任何事情的函数。这是一个深入内部的 python 脚本：

```shell
#!/usr/bin/python

import os
import subprocess
import sys

list = subprocess.check_output(["dex-method-list", sys.argv[1]])

class_info_by_name = {}

for item in list.split('\n'):
    first_space = item.find(' ')
    open_paren = item.find('(')
    close_paren = item.find(')')
    last_space = item.rfind(' ')

    class_name = item[0:first_space]
    method_name = item[first_space + 1:open_paren]
    params = [param for param in item[open_paren + 1:close_paren].split(', ') if len(param) > 0]
    return_type = item[last_space + 1:]
    if last_space < close_paren:
        return_type = 'void'

    # print class_name, method_name, params, return_type

    if class_name not in class_info_by_name:
        class_info_by_name[class_name] = {}
    class_info = class_info_by_name[class_name]

    if method_name not in class_info:
        class_info[method_name] = []
    method_info_by_name = class_info[method_name]

    method_info_by_name.append({
        'params': params,
        'return': return_type
    })

count = 0
for class_name, class_info in class_info_by_name.items():
    for method_name, method_info_by_name in class_info.items():
        for method_info in method_info_by_name:
            for other_method_info in method_info_by_name:
                if method_info == other_method_info:
                    continue # Do not compare against self.
                params = method_info['params']
                other_params = other_method_info['params']
                if len(params) != len(other_params):
                    continue # Do not compare different numbered parameter lists.

                match = True
                erased = False
                for idx, param in enumerate(params):
                    other_param = other_params[idx]
                    if param != 'Object' and not param[0].islower() and other_param == 'Object':
                        erased = True
                    elif param != other_param:
                        match = False

                return_type = method_info['return']
                other_return_type = other_method_info['return']
                if return_type != 'Object' and other_return_type == 'Object':
                    erased = True
                elif return_type != other_return_type:
                    match = False

                if match and erased:
                    count += 1
                    # print "FOUND! %s %s %s %s" % (class_name, method_name, params, return_type)
                    # print " %s %s %s %s" % (class_name, method_name, other_params, other_return_type)

print os.path.basename(sys.argv[1]) + '\t' + str(count)
```

这得花了很长时间才能弄明白，但是我想知道这在应用里面有多流行。我可以采用同样的流程，然后对我设备上安装的所有应用都运行一遍。

```shell
$ column -t -s $'\t' \
    <(echo -e "NAME\tERASED" \
        && find apks -type f | \
            xargs -L1 ./erased.py | \
            sort -k2,2nr)
```

有几千个。你做的不全在这。我说过 ProGuard 在某种程度上能帮上忙。好处是如果 ProGuard 能够发现没有任何人引用这个接收一个对象然后返回对象的泛型函数的话。它就能消灭它，所以你能看到 ProGuard 消灭了成千上万个函数。但是有的函数不能被移除因为你在接口那里用抽象的方式调用了这些方法。

最后一个我想讨论的例子，对于 Android 来说是新的，并且即将出现。这就是 Java 8 语言特性。

```java
class Greeter {
    void sayHi() {
        System.out.println("Hi!");
    }
}

class Example {
    public static void main(String... args) {
        Executor executor = Executors.newSingleThreadExecutor();
        final Greeter greeter = new Greeter();
        executor.execute(new Runnable() {
            @Override public void run() {
                greeter.sayHi();
            }
        });
    }
}
```

我们有 retro-lamina 一段时间了。但是现在 Jack compiler 在同样的精神指导下实现了这些功能，这样会让它们向后兼容。但是新语言也会有相关的开销吗？

我有一些简单的类，它们在调用它的函数的时候会打印 `Hi`。我想做的事情是让我的 `Greeter` 在 `Executor` 的时候打印 hello。`Executor` 有一个函数叫做 `run`，它接收 `Runnable`。在当前的情况下，我们设置创建者类型为 `final`。然后我们就能创建一个新的 runnable 来直接调用函数了。

```java
class Greeter {
    void sayHi() {
        System.out.println("Hi!");
    }
}

class Example {
    public static void main(String... args) {
        Executor executor = Executors.newSingleThreadExecutor();
        Greeter greeter = new Greeter();
        executor.execute(() -> greeter.sayHi());
    }
}
```

在 Lambda 的世界里，这变得特别简洁了。它也会创建一个 `Runnable`，但是它是隐式创建的。你不需要指定类型。你不需要真正的函数名和参数类型。

最后一个是函数引用。这有些意思，因为这是一个不返回任何东西而且不接受任何参数的函数，我可以把它自动地变成 `Runnable`，因为我知道我需要做的事情就是调用这个函数。

```java
class Greeter {
    void sayHi() {
        System.out.println("Hi!");
    }
}

class Example {
    public static void main(String... args) {
        Executor executor = Executors.newSingleThreadExecutor();
        Greeter greeter = new Greeter();
        executor.execute(greeter::sayHi);
    }
}
```

## 这些开销有多大？[(24:45)](javascript:presentz.changeChapter(0,102,true);)

这很有趣，但是这些开销到底有多大？采用这些语言特性的开销到底多大？我准备了一个 [Retrolambda](https://github.com/orfjackal/retrolambda) 工具链和一个使用 Jack 的工具链。

```shell
Retrolambda toolchain

$ javac *.java

$ java -Dretrolambda.inputDir=. -Dretrolambda.classpath=. \
    -jar retrolambda.jar

$ dx --dex --output=example.dex *.class

$ dex-method-list example.dex


Jack toolchain 

$ java -jar android-sdk/build-tools/24.0.1/jack.jar \
        -cp android-sdk/platforms/android-24/android.jar \
        --output-dex . *.java

$ dex-method-list classes.dex
```

这两个都没有使用 ProGuard，而且最简化了 Jack，因为它不影响结果。在匿名类的例子中，对于我们到目前为止常常使用的例子而言，它总共有两个方法。

```shell
Example$1 <init>(Greeter)
Example$1 run()

$ javap -c 'Example$1'
final class Example$1 implements java.lang.Runnable {
    Example$1(Greeter);
        Code: <removed>

    public void run();
        Code:
        0: aload_0
        1: getfield #1 // Field val$greeter:LGreeter;
        4: invokevirtual #3 // Method Greeter.sayHi:()V
        7: return
}
```

如果我们编译我们的 example，我们可以看到这就是我们的匿名类，它给附上了简单升序的数字。我们看看构造函数。构造函数为我们接收了 `Greeter` 类，然后它有一个 run 函数，如果我们反编译它，它所做的事情就是调用函数。这就是我们期望的事情，非常直接。

当我们使用 lambda 的时候，如果你使用的是一个老版本的 retrolambda，开销就很大了。简单的一小行代码能变成六个或者七个函数来完成功能。谢天谢地，现在最小的版本是 4。 而且 Jack 和版本 3 工作的很好，所以只会比匿名类多出一个函数。但是区别在哪里呢？为什么会有多余的函数呢？

我们知道如何弄清楚。这是 retrolambda，它有两个额外的函数：

```shell
Example lambda$main$0(Greeter)
Example$$Lambda$1 <init>(Greeter)
Example$$Lambda$1 lambdaFactory$(Greeter) → Runnable
Example$$Lambda$1 run()

$ javap -c 'Example$$Lambda$1'

final class Example$$Lambda$1 implements java.lang.Runnable {
    public void run();
        Code:
            0: aload_0
            1: getfield #15 // Field arg$1:LGreeter;
            4: invokestatic #21 // Method Example.lambda$main$0:
            7: return
}
```

开始的那个函数是新的。这里发生的事情是：你在 lambda 里面定义了一段代码，这段代码需要被封装到某个地方。它不在定义这个 lamda 的函数里面，因为如果这样就会很怪。它不属于那个函数。它需要在其他的某个地方定义，然后在你需要的时候传入。

这就是开始的那个函数。它只是拷贝粘贴了你在同一个类里面的 lambda 的内容。这里你可以看实现方法。它做的所有事情就是代理 `sayHi` 函数。和我们的 runnable 实现非常类似。我们仍然有构造函数。除了 run 函数的修改有点不同外。它会回到原来的类然后调用 lambda 函数，而不是直接调用 `Greeter`。这就是那个额外的函数。接下来看看 retrolambda 是如何工作的。

```shell
Example lambda$main$0(Greeter)
Example$$Lambda$1 <init>(Greeter)
Example$$Lambda$1 lambdaFactory$(Greeter) → Runnable
Example$$Lambda$1 run()
```

它生成了另外一个函数，这是一个静态工厂方法，它调用了构造函数，而不是直接调用构造函数来创建生成类。Jack 做的事情也很类似，除了额外的静态方法。

```shell
Example -Example_lambda$1(Greeter)
Example <init>()
Example main(String[])
Example run(Runnable)
Example$-void_main_java_lang_String__args_LambdaImpl0 <init>(Greeter)
Example$-void_main_java_lang_String__args_LambdaImpl0 run()
Greeter <init>()
Greeter sayHi()
java.io.PrintStream println(String)
java.lang.Object <init>()
java.lang.Runnable run()
```

我们仍然有 lambda one，虽然它的名字很疯狂，而且生成类的命名也很有创造性，它在名字里使用了整个类型签名。如上所示。三个方法。Lambda 产生了开始的额外方法。这就是另外一个额外方法开销的原因。

函数引用也很有趣。Retrolambda 和 Jack 本质上是绑定的。Retrolambda 有时候需要产生一个额外的方法，原因是你可能需要引用一个私有方法，所以这不是 Retrolambda 产生的。Java 产生了这些合成 accessor 函数中的一个，因为你给另外一个类传递了一个私有方法的引用，它做不到。这是第四个函数产生的原因。

Jack，非常有趣，为每一个单独的函数引用产生了三个函数。除了私有类，它应该为每一个函数产生两个。这种情况下，它应该产生三个函数。现在，它为每一个函数引用都产生了一个 accessor 函数。这看起来像一个 bug，所以希望我们能看到 Jack 减少到两个函数。这很重要，因为在函数引用上，这和匿名类一样了。那么从匿名类转到函数引用的抽象就没有开销了，这会很棒。

Lambda 不幸的是，不太可能减少到同样的数量。原因是你还是可能在 lambda 里面引用私有函数或者私有成员。它不可能被拷贝到生成的 runnable 里面。因为，它没有任何访问这些东西的方法。我们也只能这么指望了。

## 现实中的 Lambdas [(30:05)](javascript:presentz.changeChapter(0,127,true);)

让我们看看现实中有多少 lambda 正被使用着。同样的方法。顺便说一句，这花费的时间很长。

```shell
#!/bin/bash             lambdas.sh
set -e

ALL=$(dex-method-list $1)

RL=$(echo "$ALL" | \grep ' lambda\$' | wc -l | xargs)
JACK=$(echo "$ALL" | \grep '_lambda\$' | wc -l | xargs)

NAME=$(basename $1)

echo -e "$NAME\t$RL\t$JACK"
```

```shell
$ column -t -s $'\t' \
    <(echo -e "NAME\tRETROLAMBDA\tJACK" \
        && find apks -type f | \
            xargs -L1 ./lambdas.sh | \
            sort -k2,2nr)

NAME                         RETROLAMBDA     JACK
com.squareup.cash             826             0
com.robinhood.android         680             0
com.imdb.mobile             306             0
com.stackexchange.marvin     174             0
com.eventbrite.attendee     53                 0
com.untappdllc.app             53                 0
```

差不多 10 分钟。这也取决于你最后获取的应用有多少。如果你需要这样做，请耐心一点。它需要一些时间，但是最后会有结果的。

不幸的是，显然没有多少人使用 lambda，我很兴奋，因为我们使用的最多。我们有 826 个 lambda。这是 lambda 的个数而不是函数的个数。我们的函数个数是 lambda 的个数 826 乘以三或者四。

没有人使用 Jack，或者至少我安装的应用没有用，没有人同时使用 Jack 和 Lambda。他们可能用了 Jack 而没有使用 lambda，这很奇怪。或者，他们使用了 ProGuarding。

所以，再强调一遍，ProGuard 完全隐藏了 lambda 类和函数名。如果你是一个使用 lambda 的流行应用，然后你使用了 ProGuard，这可能就是你不在这个列表上的原因。或者是因为我不喜欢你的应用。这是关于函数的一切。

我研究这个的原因是为了突破 65K 的限制。但是这些函数还有运行时候的开销。加载额外的字节流是有开销的。运行的时候如果你需要遍历它们，那么还有额外的开销。私有变量是我最喜欢的部分，因为许多时候你都能看到这些匿名的监听者里面发生了什么。主线程上的 UI 交互往往会导致这些结果。

你不想要的正是这个在主线程上运行的计算开销昂贵的代码，不论它是动画，计算大小或者其它的任何事情。每一次你应用那些私有变量的时候，你都不想要这些开销。你都不想跳转到那个额外的函数去。查找一个变量是很快的。调用一个函数，然后查找一个变量也很快，但是肯定比查找一个变量要慢些。因为这些 accessor 函数的存在，你没有直接立即作用，而是引入了这些间接的途径。但是它们是些无用的方法，它们能做的事情就是增大你的 APK ，然后拖慢你的应用。

## 集合 [(33:21)](javascript:presentz.changeChapter(0,129,true);)

我想对齿轮做些修改，然后讨论些更加关注运行时情况的事情。这和集合有关。

```java
HashMap<K, V>                ArrayMap<K, V>
HashSet<K, V>                ArraySet<V>
HashMap<Integer, V>            SparseArray<V>
HashMap<Integer, Boolean>    SparseBooleanArray
HashMap<Integer, Integer>    SparseIntArray
HashMap<Integer, Long>        SparseLongArray
HashMap<Long, V>            LongSparseArray<V>
```

如果你的应用里面有类似的事情，你可能浪费了比你需要的要多的资源。

Android 有这些专门的集合，我想大多数人现在都很了解。实现各式各样，但是它们是专门为非常普遍的情况定制的。例如，当你需要一个 map 中整型的索引来引用某些值的时候。就有一个专门的集合可以使用。

许多时候人们都谈论着 autoboxing 的内容。Autoboxing 是什么？如果你不知道的话。当我们有一个 HashMap 的时候，这个 HasMap 接受整型的 key，这样你有一个整型的值，然后你想把这个值放进 Map 里面。 或者相反的，你在遍历整个条目，你想根据 key 得到一个值，这个转换不是直接能实现的。它需要一个额外的步骤叫做 aotoboxing，那里它先获得主要的值，然后转换成它的类版本，这个例子里面是整型。

最后的动作开销不是很大。它是封装一个类型。开始的例子开销就大了。小数字还好，因为有缓存，但是如果你有一个随机的变化剧烈的整形，每一次你调用一个方法的时候都会分配空间。这很普遍，而且接受大的整型。这是大部分人认为是优点的地方。这是个很明显的优点，但是它还有两个我们从未谈及的其他优点。

第一个是数据间接性。如果你看看 HashMap 是如何实现的，它有一个这些节点的数组，这个数组有自己的大小。当你插入一个数值或者查找一个数值的时候，它就会跳到这个数组去。这就是 hash 的步骤。找到这个 hash 值是有时空开销的，然后它返回数组的偏移。然而这是一个节点数组。节点的类型有 key 和 value。它也有 hash，它有一个指向额外节点的指针。我们回到那个数组，找到节点的引用，然后我们需要跳转到那个节点了。如果我们需要值，我们就需要访问这个节点。我们获得节点的引用，然后跳转过去。我们需要遍历这些间接引用。它们在内存里面是不同的大小。你需要跳过这些内存来得到 key 的值。或者在一个 key 上赋一个值。这会更糟。

这就是 hash 碰撞问题，当两个 hash 碰到同一个 bucket 的时候，HashMap 在那个 bucket 里面会有一个链表。如果我们碰巧碰到这种情况，我们就需要遍历链表来获取合适的 hash 值。我将讨论一下 sparse 数组。 sparse 数组就是替代解决方案。我们马上就会讨论它，但是在这之前，另外一个优点就是开销。这些集合的内存开销。这里我们有两个类。

```shell
$ java -jar jol-cli-0.5-full.jar internals java.util.HashMap
# Running 64-bit HotSpot VM.
# Objects are 8 bytes aligned.
java.util.HashMap object internals:
OFFSET     SIZE     TYPE DESCRIPTION                 VALUE
0         4         (object header)                 01 00 00 00
4         4         (object header)                 00 00 00 00
8         4         (object header)                 9f 37 00 f8
12         4         Set AbstractMap.keySet             null
16         4         Collection AbstractMap.values     null
20         4         int HashMap.size                 0
24         4         int HashMap.modCount             0
28         4         int HashMap.threshold             0
32         4         float HashMap.loadFactor         0.75
36         4         Node[] HashMap.table             null
40         4         Set HashMap.entrySet             null
44         4         (loss due to the next object alignment)

Instance size: 48 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

我们可以使用一个叫做 Java 对象布局的工具，它会告诉你创建一个这样的对象在内存里面的开销。对 HashMap 运行它，它就会打印一堆东西。它会显示你的每一个字段的开销。这里重要的数字在最底下，每一个 HashMap, 只是 HashMap, 不是节点，不是 key 和 value 或者其它什么东西。就是 HashMap 对象本身是 48 个字节。这不差，这很小。

```shell
$ java -jar jol-cli-0.5-full.jar internals 'java.util.HashMap$Node'
# Running 64-bit HotSpot VM.
# Objects are 8 bytes aligned.

java.util.HashMap$Node object internals:
OFFSET     SIZE     TYPE DESCRIPTION     VALUE
0         12         (object header)     N/A
12         4         int Node.hash         N/A
16         4         Object Node.key     N/A
20         4         Object Node.value     N/A
24         4         Node Node.next         N/A
28         4         (loss due to the next object alignment)

Instance size: 32 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

我们对节点对象运行它。一样的东西。我们看到我们的四个字段，这个是 32 个字节。每一个 map 里面的节点都是 32 个字节。Map 里面的每一个成员，每一个键值对都会在这个节点里。它是 32 乘以条目的个数。

当我们开始插入值的时候，我们可以使用这样类似的公式来计算运行时一个对象的大小是多少。这不全是正确的。你还需要算上数组的开销。还有一个数组持有这些数值，所以我们需要加上这些数值占用的空间。这很复杂。它是 4，这是一个单独整形的大小，它也是对每个数值的引用，乘以数组的大小。

问题是，HashMap 有一个叫做 load 因子的东西。末尾的 8 字节是每个数组都有的开销。但是 load 因子从来都不是饱和的。它会维系某种程度的饱和，所以当它达到那个饱和因子的时候，它就会像一个数组列表一样增长。HashMap 也会持续增长这样它就可以维系一些空的空间。

这样做的原因是，如果不这样做的话，你会有许多的碰撞和性能损失。这就差不多是一个 HashMap 不论有多少个条目，也不论 load 因子是多大的开销了。我们能算出来它需要占用多少内存。顺便说一句，默认的 load 因子是 75%。HashMap 只会达到 3/4 的容量。

Sparse 数组是你替代这样的 HashMap 的东西。让我们看看 HashMap 的两个例子，间接引用，间接引用的级别和内存的大小。Sparse 存储了两个差不多的数组。一个是 key， 另一个是 value。如果我们寻找 map 里面的某个值，或者插入某个值，我们要做的第一件事就是访问这个整型数组，不像 HashMap，不是固定时间开销。它需要对数组做二叉树查找。然后我们能访问到这个数值，在这个例子里，二叉树查找给了我们查找值的特定单元。因为这个数组是值，我们能直接返回，直接跳到引用，然会返回。

关于内存，这里的间接引用就少了很多。整型数组是连续的，它不是链表。我们能直接地在数组内部跳转。它没有我们需要解析的节点对象，然后才能获得值的过程。间接引用少了许多。然而，非固定时间有时会慢很多。这就是为什么你想尽可能的保证数组小的原因。说到小，意味着成百上千的条目。如果你有上万个条目，性能就会很糟，相比较而言，这个时候 HashMap 的内存开销反而更有吸引力一些。

```shell
$ javac SparseArray.java

$ java -cp .:jol-cli-0.5-full.jar org.openjdk.jol.Main \
internals android.util.SparseArray

# Running 64-bit HotSpot VM.
# Objects are 8 bytes aligned.

android.util.SparseArray object internals:
OFFSET     SIZE     TYPE DESCRIPTION                 VALUE
0         4         (object header)                 01 00 00 00
4         4         (object header)                 00 00 00 00
8         4         (object header)                 1a 69 01 f8
12         4         int SparseArray.mSize             0
16         1         boolean SparseArray.mGarbage     false
17         3         (alignment/padding gap)         N/A
20         4         int[] SparseArray.mKeys         [0, 0, 0, 0, 0, 0, …]
24         4         Object[] SparseArray.mValues     [null, null, null, …]
28         4         (loss due to the next object alignment)

Instance size: 32 bytes
Space losses: 3 bytes internal + 4 bytes external = 7 bytes total
```

在这个例子里面，因为类都是差不多的，而且是 JVM 64 bit， Android 现在也是 64 bit 了，数字是一样的。如果你不习惯，把它们看作是个近似值，然后增加 20% 的变量。这肯定比弄清楚在 Android 里对象的大小要容易得多。Android 里面几乎不可能弄清楚，这个方法会容易许多。

Sparse 数组的对象本身稍微小一点，32 字节。这个大小没有太大关系。真正有影响的是你知道它们不可能只有 32 字节。我们仍然有条目增长问题。我们需要计算整个数组的大小，那是所有整形 key，那是 4 乘以条目的个数然后加上 8. 同时还有 value 数组的大小，是同样的计算方式，4 乘以条目的个数然后加上 8.

Sparse 数组这里有一点不同，就是它没有 load 因子。但是它也隐含着因子的概念，因为这些数组是树，它们是二叉树。它们不是全部填满的。也不是连续的。它们中间有些空间是没有被占用的。我们需要用另外一个捏造的因子来说明这个问题，这个因子和 load 因子很像。在这个例子里面，这完全取决于你往 map 里面插入什么元素。我使用同样的值。我假设这些数组会 75% 的满载，这是一个非常安全的假设。

```
SparseArray<V>
32 + (4 * entries + 4 * entries) / 0.75

HashMap<Integer, V>
48 + 32 * entries + 4 * (entries / loadFactor) + 8


```

现在我们能直接比较这些内容了。我们知道我们能计算间接跳转。那很容易。现在我们能用这些公式来计算这些实例真正占用多少内存了。

```
SparseArray<V>
32 + (4 * 50 + 4 * 50) / 0.75 = 656

HashMap<Integer, V>
48 + 32 * 50 + 4 * (50 / 0.75) + 8 = 1922


```

对于 HashMap 我将使用默认的 .75 因子。我们可以假设一个条目的个数。在这个例子里，我选择了 50。也许这是一个美国各州的 Map。然后我们能进行这些计算。你能看到 Sparese 数组大概是 HashMap 的 1/3。你需要记住，这里会有一些性能的损失，因为每一个操作都不再是瞬时的时间了。这里是 50 个元素，所以二叉树的搜索是非常快的。

## 结论 [(44:18)](javascript:presentz.changeChapter(0,166,true);)

讲了很多东西。都是些非常琐碎的需要去做的事情，这样做才能避免函数编译时的开销，运行时不同对象的开销，还有间接引用的开销。

第一个我已经展示了。打开私有成员检查然后注意它。不要忽视它们。你不需要为每一个类型都这样做。你不需要都看一遍，然后全部一次解决。这是你在开发应用时能同时完成的事情。相反的，对于库的开发者来说，这会更重要一点。

作为一个库，你希望最小化你对 APK 大小的影响和运行时性能的影响。或者 deck 大小和运行时的性能。如果你在开发一个库，你可能需要遍历一遍然后找出所有的问题。合成的生成函数在库里面没有任何意义。它只是浪费了 dex 文件的空间。也浪费了运行的时间，把它们当作 Bug 一样看待。

如果你在使用 retrolambda，请千万确认你已经升级到最新的版本了，因为你可能浪费了成千上万个函数。如果你在编写一个开源库，请注意处理好你的匿名类。也许这没有太大的影响。但是，在强调一遍，你是追求对使用你库的应用开发者造成最小的影响的，这样做你的库就不会有问题了。

试试 Jack。它很好用。它还是快速的开发中。它现在还缺少许多东西来使得每一个人都能使用。但是肯定的是，总有些不太重要的应用是不希望在编译的时候做些不一样的事情的。

不要忽略 bug。不要回避，找到它，并且解决它。”哦，不管怎么样，我都用了 2 年了。” 你可以这么说，在你转回 Java C 索引的时候上报这些 bug。这是未来的趋势，你不可能闭关锁国，因为它在发生着。现在接受这些事情比未来要容易些。然后是打开 ProGuard。ProGuard 也有作用。无论你是不是在用 ProGuard 或者是其他增量缩减工具，而且 instant run 也在一起配合的很好，想必这也是 Jack 未来会使用的东西。

不要死板地套用规则。如果你在 ProGuard 的规则文件里面看见 `* *`，那肯定是 100% 错误的。这一定不是你希望使用的方式，因为这样做你不会从 ProGuard 里面得到任何好处。你可能说 “是的，我在使用 OkHttp，但是我不想使用 HTTP/2 的所有方法，我也从来没有用过。我不想它们被移除，我希望保持它们以备未来使用” 这并不聪明。如果你在开源库里看到这些，这是一个 bug。上报这个 bug。如果你的应用里面有这些东西，尝试着理解为什么它们会被加入。把它们拿出来，然后想想，看看 ProGuard 报的错误，然后看看你能对你的规则做些什么特别的修改。如果这些东西引起你的注意了，而且你感兴趣，这里还有其他的一些演讲你可以作为参考。

## 参考资源

- [Jack 工具链](https://source.android.com/source/jack.html)
- [Proguard](https://developer.android.com/studio/build/shrink-code.html)
- [Retrolambda](https://github.com/orfjackal/retrolambda)
- [减少代码开销](https://jakes.link/eliminate-overhead)
- [Dex Ed](http://jakes.link/dex-ed)
- [安卓内存](http://jakes.link/android-perf)
- [Jack 为函数引用产生额外的方法](http://b.android.com/218301)
- [Retrolambda 对 lambdas 或者 方法引用产生额外的方法](https://github.com/orfjackal/retrolambda/issues/81)
- [dex-method-list 工具，显示 class/jar/aar/dex/apk 里的方法](https://github.com/JakeWharton/dex-method-list)
- [Java 对象布局 (“jol”) 工具，显示类型的内存消耗](http://openjdk.java.net/projects/code-tools/jol/)
- [accessors.sh, erased.py, lambdas.sh 工具脚本](http://jakes.link/exploring-scripts)