---
layout: post
title: "基于Prolog构造Scheme解析器之三：文法分析"
date: 2014-10-04 09:49
comments: true
categories: 
 - scheme
 - parser
 - porlog
 - language
 - compiler
---

我们知道，形式文法是由一组文法生成式组成的。一个生成式可能是这样子的，

{% codeblock lang:antlr %}
IfStat -> 'if' '(' Expr ')' StateBlock
{% endcodeblock %}
这是常见的描述if语句的生成式，它表达了这样的含义：如果匹配了‘if’，并且匹配了'('，并且匹配了Expr，并且匹配了')'，并且遇到了StateBlok，那么就匹配了一个完整的IfStat。这里我故意使用了“如果”、“并且”、“那么”，目的是想说明文法生成式和Prolog语句是有着惊人的相似的。每一个生成式实际上就是一个命题。因此，相对于命令式语言，用Prolog进行文法解析是一件比较简单的工作。下面，我们开始用Prolog实现简单Scheme的文法分析部分。

## 文法分析  ##

我们先来看一下本文中作为范例的Scheme的文法，它自身非常简单，这也是选择它做范例的原因：

{% codeblock lang:antlr %}
prog -> statement* 
statement -> '(' item* ')'
item -> identifier |
         digit |
         '+' |
         '-' |
         '*' |
         '/' |
         statement
{% endcodeblock %}

很简单，是不是？好，下面我们从最高层的谓词开始，由抽象到具体，逐步构造scheme的解析器。第一个谓词是parser，它的输入参数是一串词法单元，输出参数是statement的列表。如果输入的词法单元为空，那么输出的statement也为空；否则，parser会调用其他谓词来收集所有的statement：

{% codeblock lang:scheme %}
parser([], []). 
parser(Tokens, Statements) :- matchStatements ...
{% endcodeblock %}

matchStatements谓词真正完成了收集statment的工作。下面是matchStatements的代码：

{% codeblock lang:scheme %}
matchStatements([], _, StatementsSoFar, Statements) :- 
    reverse(StatementsSoFar, Statements).

matchStatements(In, Out, StatementsSoFar, Statements) :-
    statement(In, TempOut, OneStatement),
    matchStatements(TempOut, Out, [OneStatement|StatementsSoFar], Statements).
{% endcodeblock %}

它和前面getTokens谓词几乎是一样的。事实上，无论是识别Token，还是识别statement，它们的过程并无本质的区别。

接下来看看statement谓词如何实现。从前面的文法可知，要匹配statement，需要先匹配一个左括号，然后是若干个item，最后是一个右括号。把这段话“翻译”为Prolog代码是很容易的：

{% codeblock lang:scheme %}
statement(In, Out, Items) :- 
    match(In, lp, AfterMatchedLp),
    matchItems(AfterMatchedLp, AfterMatchedItems, Items),
    match(AfterMatchedItems, rp, Out)
{% endcodeblock %}

这里我们遇到两个新的谓词，match和matchItems。首先是match，它用于推断一个词法单元序列的第一个元素是否是某个特定的词法单元，而它的输出参数是去掉一个元素后剩余的词法单元。由于match非常简单，此处暂时不给出其代码了。

最后是matchItems谓词。它的任务就是收集statement中的每一项（item），比如对于(+ 1 2)，matchItems就会发现三个项，'+'、'1'、'2'；另外，对于(+ (- 3 2) 1)，matchItems也会发现三个项，'+'、'(- 3 2)'、'1'，不过第二项比较特殊，它又是一个statement，由另外三项组成。最后，何时结束matchItems呢？那就是当matchItems遇到'('的时候。基于上述讨论的三种情况，我们需要分别写出三个matchItems谓词，按照由特殊到一般的原则，我们首先处理matchItems遇到'('和')'的情况，最后是一般的情况。

{% codeblock lang:scheme %}
matchItems(In, Out, ItemsSoFar, Items) :- 
    statement(In, TempOut, ItemValue), 
    matchItems(TempOut, Out, [ItemValue|ItemsSoFar], Items). 

matchItems([rp|In], [rp|In], ItemsSoFar, Items) :- reverse(ItemsSoFar, Items). 

matchItems(In, Out, ItemsSoFar, Items) :- 
    match(In, Item, TempOut), 
    matchItems(TempOut, Out, [Item|ItemsSoFar], Items)
{% endcodeblock %}

相信现在你应该能够很容易地看懂这段代码了吧。

最后我们再来看看解析器的输出。前面file和getTokens的谓词都十分明确，前者是文件的字节流，后者是词法单元的序列。而parser的输出通常有两种，一种是基于语法制导，在解析的过程里完成对语言的解释和执行，很多脚本语言和简单的语言都是这样做的；另一种则是生成抽象语法树(AST)或者其他中间形式，以供后续的阶段消费。多数复杂的语言都会在解析阶段输出AST。AST相对于源代码要精简很多，便于语言运行时进行多趟处理，比如优化、类型推演等，是难以在一趟中完成的。在本文的例子中，由于我们要支持define定义的变量，而且变量的定义是可以后于其使用的，因此运行时需要在编译后和运行前增加一个步骤——创建符号表。综上，本文选择了AST作为parser谓词的最终输出。

然而，Prolog并不直接支持tree数据结构，只有支持列表，如何用列表实现tree是二维结构呢？这也不难。计算机的存储结构自身是一维线性的，本质上就是一个列表，因此任何tree结构最终都要转化成列表，比如下面的tree

![计算机结构图](/images/scheme-parser-using-prolog/computer.gif)
计算机结构图

那么我们可以用下面的列表表示这棵树[computer, [cpu, Intel, 2.83G], [memery, 8G, DDR], [monitor, 15', LCD], [keyborad, “USA Standard”]]，这样二维的数据结构就可以由一维的列表表示了。对于Prolog来说，这件事格外容易。假如有下面的代码：

{% codeblock lang:scheme %}
(define a 1) 
(+ (- 3 2) 1)
{% endcodeblock %}

那么parser谓词的输出将是：[[define, a, 1], [add, [minus, 3, 2], 1]]。最外层的列表包含两个元素，[define, a, 1]和[add, [minus, 3, 2], 1]，它们分别都是列表，并且第二个元素中又包含了其他的列表。

现在，可以把我们已经有的谓词组合到一起了，

{% codeblock lang:scheme %}
:- file(“example.ss”, FileContent), getTokens(FileContent, Tokens), parser(Tokens, Statements).
{% endcodeblock %}

下来是本文的最后一部份，scheme的运行时，启程吧！