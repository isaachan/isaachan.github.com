---
layout: post
title: "Scheme on Prolog (2)：词法分析"
date: 2014-10-04 09:49
comments: true
categories: 
 - scheme
 - parser
 - porlog
 - language
 - compiler
---
在第一节中，我们实现了readFile谓词，利用它可以读取源文件的内容，并把字节流保存在一个整数的列表中。接下来我们将消费这些数据，通过它们创建词法单元。因此，本节将讲述如何构造一个词法分析器来完成这件事。

## 词法分析 ##

就像在第一节已经指出的，本文处理的只是Scheme的子集，它的词法规则如下：

- +, -, *, /, (, ),Whitespace
- 'define'
- identifier number
<!-- more -->

第一行列出的词法单元只包含一个字符，第二行的‘define’是一个关键字，虽然它不是一个字符组成的，但是仍然是固定长度的。第三行的identifier和number是标识符和数字，它们几乎出现在所有语言中，并且它们的长度不是固定的。

很明显，第一行中的词法单元是最容易识别的，下面这些谓词描述如何识别它们：

{% codeblock lang:scheme %}
getToken([40|Rest], Rest, lp). %% ( 
getToken([41|Rest], Rest, rp). %% ) 
getToken([42|Rest], Rest, mult). %% * 
getToken([43|Rest], Rest, plus). %% + 
getToken([45|Rest], Rest, minus). %% - 
getToken([47|Rest], Rest, div). %% / 
getToken([32|Rest], Rest, ws). %% WhiteSpace 
getToken([9|Rest], Rest, ws). %% WhiteSpace 
getToken([10|Rest], Rest, ws). %% WhiteSpace
{% endcodeblock %}

这段代码实现了getToken谓词，这个谓词的第一个参数是进行词法单元匹配以前的字节流，第二个参数是匹配过词法单元后剩余的字符流，第三个参数表示此次getToken匹配到的词法单元。代码中出现的整数，40，41，42...是字符的ASCII码，后面的注释解释了它代表的字符。上述代码中的[40|Rest]的词法单元是一个列表，它的第一个元素是40，其他元素是Rest。这很类似于scheme中的car和cdr操作。[]则表示一个空的列表。Prolog的这种文法极大的方便了列表的处理。如果你熟悉Erlang，会发现Prolog处理列表的文法和语义是一样的（事实上这两种语言也确实深有渊源）。

接下来，我们看看如何识别define关键字。由于这个词法单元的长度是固定的，因此识别它也并不复杂：

{% codeblock lang:scheme %}
getToken([100,101,102,105,110,101|Rest], Rest, define). %% define
{% endcodeblock %}

这里，我们用到了列表的另一种文法，[100,101,102,105,110,101|Rest]表示列表的前六个元素一次是 100,101,102,105,110,101。因此这个谓词可以匹配define。

现在我们已经可以匹配除标识符和数字以外的其他词法单元了，那么如何把所有匹配到的词法单元收集到一起呢？嗯，我们需要一个getTokens谓词来收集词法单元，如果输入的字节列表为空，那么收集工作宣告结束；否则，它应该能重复地调用getToken，前一个getToken的输出是后一个getToken的输入。哦，对了，Prolog没有for循环，用递归才能实现重复执行一段代码的工作。OK，getTokens谓词的代码应该是这样子了：

{% codeblock lang:scheme %}
getTokens([], TokensSoFar, Tokens) :- reverse(TokensSoFar, Tokens). 
getTokens(Input, TokensSoFar, Tokens) :- 
    getToken(Input, Rest, OneToken), 
    getTokens(Rest, [OneToken|TokensSoFar], Tokens).
{% endcodeblock %}

TokensSoFar用来收集计算过程中匹配的词法单元，但是它收集的顺序和词法单元真正的顺序是相反的，因为geTokens总是把后遇到的词法单元放到TokensSoFar的头部，这是受到Prolog列表处理的限制。当输入字节流为空时，把TokensSoFar中的词法单元反向排序，就得到了我们想要的答案。

在进行词法分析的过程中，我们通常要丢弃一些无用的词法单元，比如注释、空白符等等。我们的Scheme子集不支持注释，所以这里我们只要丢弃掉空白符即可。当getTokens遇到一个ws词法单元的时候，不要把它加到TokensSoFar的头部，而是直接忽略掉：

{% codeblock lang:scheme %}
getTokens([], TokensSoFar, Tokens) :- reverse(TokensSoFar, Tokens). 
getTokens(Input, TokensSoFar, Tokens) :- 
    getToken(Input, Rest, ws), 
    getTokens(Rest, TokensSoFar, Tokens). 
getTokens(Input, TokensSoFar, Tokens) :- 
    getToken(Input, Rest, OneToken), 
    getTokens(Rest, [OneToken|TokensSoFar], Tokens).
{% endcodeblock %}

这个getTokens谓词的位置很重要，它必须位于原来两个谓词之间。Prolog在模式匹配相同签名的谓词的时候，会按照它们声明的先后顺序进行尝试，因此必须把描述特殊情况的谓词排在前面，一般情况的谓词排在后面。

词法分析完成了吗？不不，我们还没有实现匹配标识符和数字的谓词！标识符和数字稍微复杂一些，它们的长度不是固定的，因此在匹配它们的getToken谓词中，还需要多一个参数，用来临时保存计算过程中已匹配的字节。在实现getToken之前，先看一下描述数字的自动机，

标识符的自动机和它是类似的。我们可以这样描述该自动机的行为：如果输入的第一个字节是数字，则进入number状态，并记录下这个数字；在number状态下，如果输入的第一个字节是数字，则继续追加该数字； 在number状态下，如果输入的第一个字节是字符，则是非法输入；在number状态下，如果输入的第一个字节是+ - * / ( ) Ws中的任何一个，则结束number状态，并完成一个number词法单元的识别。

{% codeblock lang:scheme %}
getToken([D|T], Rest, Nubmer) :- 
    digit(D),
    getNumberToken(T, Rest, [D], Number).

getNumberToken([D|T], Rest, DigitsSoFar, Number) :-
    digit(D),
    getToken(T, Rest, [D|DigitsSoFar], Number).

getNumberToken([H|T], [H|T], DigitsSoFar, Number) :-
    single(H),
    reverse(DigitsSoFar, Number).

digit(D) :- 47 < D, D < 58.
single(T) :- T=39; T=9; T=10; T=40; T=41; T=42; T=43; T=45; T=47.
{% endcodeblock %}

第一个getToken谓词描述了从非number状态到number状态的转换。进入Number状态后，开始调用getNumberToken谓词。getNumberToken谓词有两个，分别处理digit和那些只包含一个字符的词法单元。出于简单的考虑，这段程序没有显式地处理遇到字母字符的情况（比如打印一段错误信息），那么，如果在number状态下遇到字母字符时，由于没有相应的谓词可以去匹配，程序同样无法成功运行。最后，在程序的第一行，Rest的值也不是根据它前面的参数确定的了。因为只有读取完所有连续的数字后，才知道Rest的起点在哪里。在第9行，Rest最终被确定下来。这里Rest之所以是[H|T]而不是T，因为当读到一个符合single谓词的字节时，意味着number读取结束了，而刚刚读到的这个字节还要留给下一个谓词去处理，不能在这里吞噬掉。

关于匹配标识符的谓词，和匹配number的谓词几乎一样，在这里就不多赘述了。本文后面会提供所有的源代码。

目前getTokens谓词需要三个参数，FileContent, TokensSoFar和Tokens。其中TokensSoFar的初始值必然是一个空的列表。因此，我们可以再创建一个谓词来“封装”这里参数，从而留给调用者一个更友好的API。

{% codeblock lang:scheme %}
   getTokens(FileContent, Tokens) 
       :- getTokens(FileContent, [], Tokens).
{% endcodeblock %}

现在，我们有file和getTokens两个谓词了，把它们写在一起已经可以分析指定文件的词法了。

{% codeblock lang:scheme %}
:- file(“example.ss”, FileContent), getTokens(FileContent, Tokens).
{% endcodeblock %}

接下了，将是更有挑战的工作——文法分析。