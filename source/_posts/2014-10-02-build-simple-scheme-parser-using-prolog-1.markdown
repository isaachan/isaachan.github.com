---
layout: post
title: "Scheme on Prolog (1)：读取源文件"
date: 2014-10-02 22:53
comments: true
categories: 
 - scheme
 - parser
 - prolog
 - language
 - compiler
---
在接下来的一系列文章里，我将详细讲述如何利用Prolog，从零开始构造一个Scheme子集的[运行时环境](https://github.com/isaachan/scheme-runtime)。这个运行时环境可以解释执行如下四则运算的Scheme代码：

{% codeblock lang:scheme %}
(+ 1 2) 
(+ (- 5 3) (* 2 3))
{% endcodeblock %}

另外，它还可以通过“define”定义变量，
{% codeblock lang:scheme %}
(define a 1) 
(define b 2) 
(+ a b) 
...
{% endcodeblock %}

选择Scheme的原因在于它足够简单而且有趣。另外，通过实现这个简单的运行时环境，我们也可以领略Prolog的独特之处。Prolog作为应用最广泛的逻辑式程序设计语言，在描述文法生成式时具有很多语言无可比拟的优势；另外作为声明式语言，Prolog不必描述运算的细节（甚至语句执行的顺序），因此程序的可读性比命令式语言要好很多。
<!-- more -->

正如前面的代码所示，我们实现的解析器只能执行简单的四则运算，但是它却涉及到读取源文件、词法分析、语法分析和构造符号表等技术，这些是所有高级编译器都具有的功能。因此本文分为四个部分，分别讲述下面四个问题：

* 读取源文件
* 词法分析
* 语法分析
* 运行时环境
 

## 读取源文件 ##

读取源代码文件是后续分析、解释执行的第一步。这个示例程序并没有提供交互式运行的机制，因此从文本文件中读取代码并运行是唯一的启动程序的方式。

谓词推演是Prolog解决问题的手段，无论待解决的问题看上去和谓词推演多么的不相关。不过，谓词推演是足够强大的，足以解决“读取文件内容”的问题。我们设计一个谓词，用来从最高的层次上读取文件的内容：

{% codeblock lang:scheme %}
file(FilePath, FileContent).
{% endcodeblock %}

这个谓词的含义是：当FilePath是一个文件路径，并且FileContent是该文件的内容时，这个谓词为真，否则为假。当然，如果我们只告诉这个谓词FilePath的信息，Prolog运行时会找到FileContent的值，使这个谓词为真。正是由于这一点，file谓词可以为我们读取文件的内容了。

为了实现这个谓词，我们还需要完成一些工作，比如如何获取Prolog运行时内部的文件句柄？如何从文件句柄中依次读取数据？如何知道是否读到了文件的尾部？

调用其他Prolog运行时内置的与文件相关的谓词，它们要能够获取文件句柄，并从句柄中依次读取文件字节。简单地查找Prolog的文档，不难发现下面两个内置的谓词：

{% codeblock lang:scheme %}
open(+FilePath, +OpenMode, -Stream, +Options).
get0(+Stream, -Char).
{% endcodeblock %}

open谓词的第一个参数是文件路径，第二个是打开文件的模式，可以是read/write/append/update之中的一个，第三个是文件句柄，第四个是额外的选项，比如字符集、缓存大小等。get0的第一个参数是文件句柄，它通常是open的输出参数Stream，第二个参数是读出的字符。get0会维护访问文件的指针，当每次调用get0时，指针会自动向后移动，当到达文件尾部的时候，Char等于-1。有了这两个谓词，我们可以进一步实现file谓词，

{% codeblock lang:scheme %}
file(FilePath, FileContent) :- 
    open(FilePath, read, Stream, [eof_action(eof_code)]), 
    readFile(Steam, FileContent).
{% endcodeblock %}

这里增加了一个readFile谓词，它会把Stream中的字节流保存到FileContent中。谓词readFile的实现可以描述如下：如果从Stream中读取的字节是-1（即到文件尾），那么读取完毕；如果从Stream中读取的字节不是-1，那么把该字节追加到已经读取的序列中，然后递归地调用readFile。下面的代码反映了这一描述：

{% codeblock lang:scheme %}
readFile(Stream, ContentSoFar, FileContent) :- 
    get0(Stream, -1),
    reverse(ContentSoFar, FileContent).
readFile(ContentSoFar, FileContent) :-
    get0(Stream, Char),
    readFile(Stream, [Char|ContentSoFar], FileContent).
{% endcodeblock %}

在这段代码中，我们首先为readFile增加了一个参数：ContentSoFar，用来保存在计算过程中收集的不完整的文件内容，当将文件全部读取结束后，再把ContentSoFar的内容反序排列后与FileContent进行合一。我们注意到在第二个readFile中，[Char|ContentSoFar]会导致先读取到字节后放到ContentSoFar序列的后端。这就是说，如果读取的文件内容是123abc，那么ContentSoFar最终的值将是['c', 'b', 'a', '3', '2', '1']。因此在第一个readFile的最后，需要把ContentSoFar反序排列后再与FileContent进行合一。

另外，get0的文档提到了，它读取的字节是一个int数值，代表了这个字节的ASCII码。出于测试的目的，我们不想看到读出的内容是一串数字，那么可以用atom_chars谓词将数字串转化为字符串，并使用write谓词将它输出到控制台上：

{% codeblock lang:scheme %}
display(file) :- 
    readFile(file, FileContent), 
    atom_char(FileContent, FileContentChar),
    write(FileContentChar).
{% endcodeblock %}

现在回过头来看看我们刚刚完成的readFile谓词，它的运行起来效果如何呢？我在尝试读取的文件如下文件：

{% codeblock lang:scheme %}
(define a 11) 
(+ 1 a)
{% endcodeblock %}

发现程序总是意外出错中止。检查了错误信息后，我发现它读取的内容是：

{% codeblock lang:scheme %}
dfn 1 
+1a
{% endcodeblock %}

程序每读取一个字节会跳过一个字节。经过分析，我找到了错误的原因。当调用readFile谓词的时候，Prolog首先尝试第一个readFile的定义，即调用get0，并期望读到文件尾（-1），这当然是不成立的，于是程序回朔，继续尝试第二个readFile的定义。但是上一次失败已经改变了get0内部维护的文件指针，而且文件指针也不会随着回朔而改变，这就产生了我们前面看到的情况。每经由递归读取一个字节时，都要尝试第一个readFile的定义，而直到真的读取到了文件尾之前，它总是会失败而引起回朔，从而丢失一个字节。为了修正这个问题，需要如下修改我们的代码：

{% codeblock lang:scheme %}
readFile(Stream, Content) :- 
    get0(Stream, A), 
    readFile(Stream, A, Content). 
readFile(_, -1, []). 
readFile(Stream, A, [A|Content]) :- readFile(Stream, Content).
{% endcodeblock %}

这样，判断读取的字符是否为-1的任务下推给了readFile谓词，确保get0不会参与到回朔过程。另外，由于使用了不同的递归方式，新的实现可以直接读取正确顺序的文件字节流，不需要再次反序排列。

完成了读取源文件的工作，接下来我们就可以进行[词法分析](/blog/2014/10/04/build-simple-scheme-parser-using-prolog-2/)了。