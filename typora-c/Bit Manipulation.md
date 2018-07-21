# 位操作

> 原文 (Medium)：[Bit Manipulation](https://medium.com/software-engineering-101/bit-manipulation-b13b94e70f3b)
>
> 作者：[Joe Birch](https://medium.com/@hitherejoe?source=post_header_lockup)

[TOC]

欢迎阅读 “软件工程101”（Software Engineering 101）上的最新文章，该文章旨在每周（ISH）深入探讨算法，设计模式，数据结构等内容。

Software Engineering 101 Deep-dives into Algorithms, Design patterns and more... [medium.com](https://medium.com/software-engineering-101)

本周我们将讨论位操作的话题。 这是在位序列上执行逻辑操作的过程，以便达到所希望的结果。 位操作允许我们使用这些操作符以一种干净和有效的方式达到某些序列，这只是为什么我们必须了解哪些操作符是可用的，以及我们如何使用它们的一个原因。 

## Operators

我们可以通过使用一些基本的逻辑运算符来执行操作，让我们看看有什么可用的。 

### XOR ( ^ )

如果只在一个操作数中设置，则可以使用 XOR 操作符复制位值。 

![](https://ws2.sinaimg.cn/large/006tKfTcgy1froui3kuc4j308g06v0sq.jpg)

如上所示，前两行计算为 x 的值，因为只有一个操作数设置为x。但是，第三行中的两个操作数的值都是 x，意味着该行等于0。

### AND ( & )

如果在两个操作数中都存在，则可以使用 AND 运算符复制位值。 

![](https://ws3.sinaimg.cn/large/006tNc79gy1froup1n481j308j076dfv.jpg)

正如你在上面看到的，第一个语句等于0，因为两个操作数都是不同的值。第二个语句等同于 x，因为...最后，第三个语句也等于x，因为两个操作数都等于 x。

### OR ( | )

如果 OR 操作符存在于任一提供的操作数中，则可以使用 OR 操作符来复制位值。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froup5jgawj307q07iq2u.jpg)

如上所见，第一个语句等同于 x，因为 x 存在于两个操作数之一中。

### Ones Complement ( ~ )

补码是一个一元操作（也称为否定），具有简单的“翻转”到相反的值的效果，如下所示：

![](https://ws1.sinaimg.cn/large/006tNc79gy1froup8cis0j307904h745.jpg)

正如您在上面看到的那样，补码操作符只是将给定位的值翻转为相反的值。

## Manipulation Examples

为了更好地理解位操作，让我们来看看一些真实的例子！首先我们来看一些另外的例子：

![](https://ws4.sinaimg.cn/large/006tNc79gy1froupbr3ztj30gw0293yb.jpg)

这里我们简单地将两个值加在一起，0100 = 4和0001 = 1。将它们加在一起给出5，它具有0101的二进制表示。

![](https://ws4.sinaimg.cn/large/006tNc79gy1froupdw1haj30gu01y0sj.jpg)

再次，我们只是将两个值相加，即0101 = 5和0001 = 1。将它们加在一起给出6，它具有0110的二进制表示。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froupgrn8tj30gw021wea.jpg)

这里我们简单地减去两个值，0010 = 2和0001 = 1。减去这些给我们1，其具有0001的二进制表示。

![](https://ws3.sinaimg.cn/large/006tNc79gy1froupj6qmsj30gz026q2q.jpg)

再次，我们只是减去两个值，1100 = 12和0010 = 2.减去这些给我们1，其具有1010的二进制表示。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froupldgb0j30gf02et8j.jpg)

在这里，我们只是乘以这两个值，即0010 = 2和0110 = 6。乘以这些就得到12，它具有1100的二进制表示。

![](https://ws2.sinaimg.cn/large/006tNc79gy1frouposawyj30ge01xglf.jpg)

再次，我们只是乘以两个值，0011 = 3和0011 = 3。乘以这些给我们9，其具有1001的二进制表示。

![](https://ws1.sinaimg.cn/large/006tNc79gy1froupsozh9j30go023t8k.jpg)

这里我们对位序列进行 AND 运算。正如我们以前所看到的，只有两个都被设置的位（值为1）保留。所以在这里我们清除除了第三个以外的所有位 - 因为这是在两个操作数中设置的唯一位。

![](https://ws2.sinaimg.cn/large/006tNc79gy1froupvag15j30fp024wea.jpg)

这里我们对位序列执行 OR 操作。 正如我们以前所见，如果左操作数不等于右操作数，那么我们将该值设置为右操作数的值。 如果操作数值相等，则该位保持不变。 在这个操作中唯一的变化是第一位 - 左操作数第一位的值为0，而右操作数的第一位值为1，所以结果的第一位等于1。

![](https://ws2.sinaimg.cn/large/006tNc79gy1froupx54n5j30h501yweb.jpg)

这里我们对位序列执行 XOR 操作。 正如我们之前所看到的，XOR 操作意味着如果只有当前位置的右/左操作数位中的一个保持该值，那么该位只会被设置为1。 在上面的例子中，我们清除了第三位，因为左操作数和右操作数的值均为1.但是，第一位被设置为1，因为只有右操作数序列在该位置保持该值。

## Two’s Complement

二进制补码允许我们以二进制序列的形式表示负值，这是通过使用1（用于负数）或0（用于正数）作为符号位值来表示值是正还是不正确。 一个 N 比特的数字（不包括符号比特）被表示为相对于 2 ^ N 的二进制补码值。

例如，-6的4位整数值将被表示为使用符号位的第一位和其余3的值本身的两个部分。 对于等于8的 2 ^ N（其中 N = 3），与6（-6的绝对值）的补码是2 - 二进制中的是010.作为二进制表示，使用第一位作为符号位 -6等于1010。

这样做的另一种方法是简单地翻转当前的表示形式，并将1的值添加到结果中，结果如下所示：

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouq01qd9j307i023a9v.jpg)

我们从最初的二进制表示6开始，我们希望把它转换成-6的表示。接下来，我们翻转一下他们相反的表示：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouq2e6qfj303w01vmwx.jpg)

这个结果给我们的代表一个符号位，使其成为负值。但是，我们仍然需要执行将值1加到我们的二进制和的最后一步：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frouq4lh3ej3097026dfn.jpg)

正面和负面表示的其他例子如下：

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouq71za6j30ei0hfdg3.jpg)

## Logical Shift

逻辑移位是将所有位按所需方向（左或右）移位并将最高有效位置为0的过程 - 此过程由>>>运算符表示。 例如，下面的二进制序列的逻辑右移将如下所示：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frouqa1d5lj30m8097mxi.jpg)

你可以在这里看到我们已经把所有的位移到了右边，只是用0来填充主导位置。这已经将我们的二进制表示从负面转换为正面，并且显着增加了它的价值。

## Arithmetic Shift

算术移位是将所有位按所需方向（左或右）再次移位的过程，但是这次我们将符号位的值放在最高有效位 - 该过程由>>操作符表示。 例如，在同一个二进制序列上的算术逻辑右移看起来是这样的：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouqe7pqzj30m80910t2.jpg)

因为我们的符号位等于1，所以在执行移位之后，我们用1的值填充显性位的位置。在这种情况下，二进制序列仍然是一个负面的表示，但我们的数量大大增加了价值。

## Retrieving a Bit

如果我们想从我们的二进制序列的特定位置检索位，那么我们可以使用一个简单的方法：

```java
public boolean getBit(int n, int i) {
    return ((n & (1 << i)) != 0);
}
```

首先，我们取值1，并使用i参数作为右操作数执行左移操作。 我们和这个结果与我们的n参数去除位置i以外的所有其他位。 然后将这个剩余的位与0进行比较，如果该位等于1则返回true。

### Example

我们从一个二进制序列开始：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouqhd2qjj30800203yb.jpg)

假设我们想要得到序列中的第二位，使得i = 2。我们首先通过将值移到i，给我们：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frouqj8gl1j30cm02a3yc.jpg)

然后我们 AND 这个结果与我们的序列n：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frouqlj47vj30gj024q2s.jpg)

最后一个阶段就是比较这个结果的第一位到0，如果它不等于0，那么第i位必须等于1（反之亦然）。

![](https://ws4.sinaimg.cn/large/006tNc79gy1frouqocplnj30gi04ddfq.jpg)

## Setting a Bit

如果我们想在二进制序列的特定位置设置一些位，那么我们可以使用一个简单的方法，其中n是我们的二进制序列，而我是我们希望设置的位：

```java
public int setBit(int n, int i) {
    return n | (1 << i);
}
```

首先，我们使用i参数作为右操作数对值1再次执行左移操作。 接下来，我们使用这个值和我们的 n 参数执行一个或运算 - 这将导致我（只有我）的位改变。 我们掩码的其他值都是0，所以它们对我们的序列 n 没有任何影响。

### Example

让我们再次从最后一个例子开始使用相同的二进制序列：

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouqsizh7j30800203yb.jpg)

假设我们要设置序列中的第四位，使得i = 4。我们首先通过将值移到i，给我们：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frouquucfcj30cs0290sk.jpg)

然后我们使用这个结果来执行一个与我们的序列的 OR 操作，给出：

![](https://ws1.sinaimg.cn/large/006tNc79gy1frouqx7fgzj30g402qwea.jpg)

由此我们可以看出，从 n 序列中的第4位已经被设置。

## Clearing a Bit

这个函数的行为与设置一个位相反，我们可以通过使用下面的函数来清除一下：

```java
public int clearBit(int n, int i) {
    int mask = ~(1 << i);
    return num & mask;
}
```

首先，我们使用i参数作为右操作数对值1再次执行左移操作。 但是，这次我们通过使用补码操作符翻转所有的位来反转数值表示。 接下来，我们使用这个值和我们的 n 参数执行 AND 操作 - 这将导致我（只有我）的位被清除。 我们掩码的其他值都是0，所以它们对我们的序列 n 没有任何影响。

### Example

让我们再次从最后一个例子开始使用相同的二进制序列：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frouqzl89nj30800203yb.jpg)

假设我们想要清除序列中的第三位，使得i = 3。我们首先通过将值移到i，给我们：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frour1t1z2j30cn0293yc.jpg)

接下来，使用补码操作符将此结果转换为：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frour47k84j30cz029a9v.jpg)

我们接下来把这个结果与我们的n序列如下：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frour9ba8tj30gl025t8k.jpg)

由此我们可以看出，从 n 序列中的第3位已经被清除。

## Updating a Bit

我们可以通过使用下面的函数来更新一下所需的值：

```java
public int updateBit(int n, int i, boolean shouldSetBit) {
    int value = shouldSetBit ? 1 : 0;
    int mask = ~(1 << i);    
    return (num & mask) | (value << i);
}
```

这个函数增加了使用标志 shouldSetBit 声明是否应该有设定值的能力。 我们首先使用掩码序列清除位置i的值。 一旦完成，我们对n和掩码序列执行 AND 操作。 然后，我们使用 i 作为右操作数对值进行移位操作，然后将其用于先前创建的值的 OR 操作。 然后这更新给定索引处的位。

## Example

让我们再次从最后一个例子开始使用相同的二进制序列：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frourcoo4bj30800203yb.jpg)

假设我们想更新序列中的第一位，使得 i = 1。我们首先通过将值移位i给我们：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frourel7rbj30cd0240sk.jpg)

接下来，使用补码操作符将此结果转换为：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frourhaeyzj30cs02aa9v.jpg)

现在掩码值为1110，我们可以和这个结果与我们的 n 序列：

![](https://ws2.sinaimg.cn/large/006tNc79gy1frourj9gjlj30gu029mx0.jpg)

我们接下来需要对我们的值变量进行移位操作：

![](https://ws3.sinaimg.cn/large/006tNc79gy1frourl2fx6j30ck0260sk.jpg)

最后，我们可以对这两个计算值执行 OR 操作：

![](https://ws4.sinaimg.cn/large/006tNc79gy1frournfxr3j30g002u0sj.jpg)

从这个结果可以看出，我们 n 序列中的第一个位已经更新了我们期望的值1。

### And that’s it!

我们已经知道了什么是位操作，以及如何使用按位操作的操作符。 无论您是从头开始学习还是刷新数据结构的知识，我都希望深入了解位操作的工作原理。

