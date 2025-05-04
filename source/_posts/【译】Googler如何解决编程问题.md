---
title: 【译】Googler如何解决编程问题
date: 2019-04-02 23:10:08
tags: 技术杂谈
comments: true
---

本文是Google工程师[Steve Merritt](https://blog.usejournal.com/@steve_45636?source=user_popover)的一篇博客，向大家介绍他自己和身边的同事解决编程问题的方法。

原文地址：https://blog.usejournal.com/how-a-googler-solves-coding-problems-ec5d59e73ec5 <!-- more -->

在本文中，我将完整的向你介绍一种解决编程问题的策略，这个策略是我在日常工作中一直使用的，并且用它来帮助各个等级的程序员（包括新手、大学生和实习生）学习和成长。应用这个结构化流程可以最大幅度的减少那令人沮丧的调试时间，并且能够在尽可能短的时间内编写出更加整洁、错误率更低的代码。

#### 一步步

接下来我将用一个栗子来说明。

问题：有两个字符串` sourceString`和`searchString`，返回`searchString`在`sourceString`中第一次出现的位置，如果`searchString`不是` sourceString`的字串，就返回-1。

#### 第一步：画下来

当你拿到一个需求，马上就开始着手写代码是一个非常愚蠢的主意。在写一篇文章之前，首先要弄清楚论证和论据，并且你要保证论证是有意义的。如果你没有这么做的话，当你意识到你所写的东西前言不搭后语时，你可能会因为浪费了大把的时间而想请自己吃一顿大嘴巴子。编程也是一样的道理，甚至比这还严重，严重到像洗澡的时候把洗发水弄进眼睛里。

问题的解决方法通常很重要，即使它看上去很简单。在写代码之前，首先要做的就是把这个方法在纸面上呈现出来，并且保证在不同的情况下适用。

所以不找急着写代码，甚至都不要思考如何写。后面你会有充足的时间去敲代码，在这之前，你要把自己当成一台计算机，弄清楚你这台计算机会怎么解决这个问题。

你可以使用流程图，或者使用其他能帮你具象化的方法，总之我们的目标是解决问题。你可以用纸和笔随意发挥，不需要收到键盘的限制。

我们从一些简单的情况开始，如果一个方法是“输入一个字符串”，那么“abc”可以作为第一个例子。首先要搞清楚正确的结果是什么，然后去想怎么样得到正确的结果，并且一步一步的进行。

我们假设输入的字符串是这样：

``` bash
sourceString: "abcdyesefgh"
searchString: "yes"
```

我的想法是这样的：

我看到了`searchString`在`sourceString`里，但是我要怎么做呢？我从` sourceString`的第一个字符开始，逐字去读，一直到最后，判断每一个三个字符是不是`yes`。比如`abc`，`bcd`，`cdy`等等。当index值为4时，我找到了字符串```yes```，所以我知道结果是index为4。

当我们写下算法时，必须要保证我们考虑了所有的情况，并且处理了所有可能的场景。当我们找到匹配的字符串时，返回结果。如果找不到匹配的字符串，同样要返回结果。

我们来尝试另一组字符串：

``` bash
sourceString: "abcdyefg"
searchString: "yes"
```

我们重复刚才的操作，当我们读到下标为4的字符时，找到的字符串是```yef```，这是最接近的结果了，但是第三个字符却不同，所以我们继续往后读，一直到最后，没有找到匹配的字符串，所以我们决定返回-1。

我们确定这一系列步骤（程序设计中，我们称之为算法）解决了我们的问题，并且处理了两种不同的场景，每次都得到了正确的结果。这时，我们就对我们的算法比较有信心了，并且可以将它形成条目。我们一起来进行下一步：

#### 第二步：用英语写下来

这里我们考虑将第一步形成的算法用英语写下来。这可以使每一步变得更加具体，以便我们后面写代码的时候有所参考。

1. 从字符串的第一个字符开始
2. 查看每一组3个的字符（其实是`searchString`的长度）
3. 如果匹配上`searchString`，就返回当前的index
4. 如果一直到末尾都没有找到匹配的字符串，就返回-1

看起来很棒，不是吗

#### 第三步：写伪代码

伪代码并不是真正的代码，只是一种模拟形式，这里我写下上面的算法的伪代码：

``` bash
for each index in sourceString,
    there are N characters in searchString
    let N chars from index onward be called POSSIBLE_MATCH
    if POSSIBLE_MATCH is equal to searchString, return index
at the end, if we haven't found a match yet, return -1.
```

我也可以用一种更接近代码的形式来写：

``` bash
for each index in sourceString,
    N = searchString.length
    POSSIBLE_MATCH = sourceString[index to index+N]
    if POSSIBLE_MATCH === searchString:
        return index
return -1
```

你的伪代码写得有多像真正的代码，你就会发现它有多么好用。

#### 第四步：翻译可以编码的内容

*注意：对于简单的问题，这一步可以和上一步合并*

到这时我们才第一次需要考虑语法、方法参数和语言规则的问题。可能你不是全部代码都会写，但是没关系，先把会写的写出来。

``` java
function findFirstMatch(searchString, sourceString) {
    let length = searchString.length;
    for (let index = 0; index < sourceString.length; index++) {
        let possibleMatch = <the LENGTH chars starting at index i>
        if (possibleMatch === searchString) {
            return index;
        }
    }
    return -1;
}
```

注意我留了一部分没有写，这是故意的！我不确定JavaScript切分字符串的语法要怎么写，所以我会在下一步查找它。

#### 第五步：不要猜测

我发现所有新手程序员都会犯一个共同的错误，就是从网上找到一个方法，觉得“可能有用”，然后不经过测试就写进代码里。你不理解的代码越多，就越不可能找到正确的方法。

你的程序的错误可能是你不了解代码的两倍还多。有一处不理解，如果程序出错，那么罪魁祸首只有一处。如果你有两处不理解，那就有三种可能出错（A出错，B出错，或者A和B都出错）。如果有三处不理解，就会有七种情况……很快它就失控了。

*边注：程序出错的情况种类遵循Mersenne公式 a(n) = (2^n) — 1*

首先要测试你的新代码。从互联网上找答案是好的，但是在写进你的代码之前，你要先对它进行单独的测试，确保它能按照你想要的方式执行。

在上一步中，我不确定JavaScript怎么切分字符串，所以我选择**面向Google编程**

https://www.google.com/search?q=how+to+select+part+of+a+string+in+javascript

第一条结果来自w3schools，有点小过时，但比较可靠

https://www.w3schools.com/jsref/jsref_substr.asp

基于此，我觉得我应该使用`substr(index, searchString.length)`来提取`sourceString` ，但这只是个假设，所以我要先来测试一下：

``` bash
>> let testStr = "abcdefghi"
>> let subStr = testStr.substr(3, 4);  // simple, easy usage
>> console.log(subStr);
"defg"
>> subStr = testStr.substr(8, 5);   // ask for more chars than exist
"i"
```

现在我确定这个函数是可以用的，如果程序出错，就不是这个函数不可用导致的。最后我补充上最后的代码。

``` javascript
function findFirstMatch(searchString, sourceString) {
    let length = searchString.length;
    for (let index = 0; index < sourceString.length; index++) {
        let possibleMatch = (
            sourceString.substr(index, length));
        if (possibleMatch === searchString) {
            return index;
        }
    }
    return -1;
}
```

#### 结论

如果你读到这里，我要说的只有：”干就完了！“

再尝试处理一下上周遇到的困难，用上我教你的方法，我保证你很快就会有提高。

祝你好运，编码愉快！

*译者注：个人认为作者还是强调要先想清楚，再动手写代码。而且要学会面向Google编程*