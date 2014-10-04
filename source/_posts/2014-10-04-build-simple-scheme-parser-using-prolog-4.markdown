---
layout: post
title: "基于Prolog构造Scheme解析器之四：运行时环境"
date: 2014-10-04 09:49
comments: true
categories: 
 - scheme
 - parser
 - porlog
 - language
 - compiler
---
当解析器生成了抽象语法树(AST)后，运行时环境可以对它进行处理，计算语法树表达的数值。如果代码只是如下简单的四则运算，

{% codeblock lang:scheme %}
(- 3 1) 
(+ (* 3 5) 2)
{% endcodeblock %}

那么对AST遍历一次就足够了，甚至在解析的过程中就可以获得结果了，根本不需要生成AST。但是，我们希望这个运行时能够支持通过define定义变量，甚至变量的作用域是全局的，而不仅仅是从它声明的地方开始，

{% codeblock lang:scheme %}
(define a 1) 
(+ a b) 
(define b 2)
{% endcodeblock %}

上面这段程序的输出应该是3，而不是“b没有声明”的错误信息，因为define定义的变量的作用域贯穿所有的代码。对于这一要求，我们需要遍历两次AST，第一次构建符号表，第二次完成运算。在实际的开发中，我们会先完成第二步的功能，即先不考虑代码中存在变量的情况，然后在逐步地加入对变量的支持。

让我们呢先看看最简单的情形，求这个表达式的值

{% codeblock lang:scheme %}
(+ 1 2 3)
{% endcodeblock %}
按照前面的分析，解析器会生成下面的AST，

{% codeblock lang:scheme %}
[plus, 1, 2, 3]
{% endcodeblock %}

连续累加plus后面的元素，这需要一个简单的递归就够了：

{% codeblock lang:scheme %}
getValue([add, FirstOperand|OtherOperands], Result) :- 
    getValue(FirstOperand, OtherOperands, Result).

getValue(ResultSoFar, [], ResultSoFar).

getValue(ResultSoFar, [H|T], Result) :-
    NewResult is ResultSoFar + H,
    getValue(NewResult, T, Result).
{% endcodeblock %}

第一行的getValue谓词把操作数分成了FirstOperand和OtherOperands，实际上表明了[add]这样的输入是不合法的，它后面必须至少有一个操作数。下面让问题再复杂一些，Operands中不仅包含数字，还递归地包含一个加法表达式，

{% codeblock lang:scheme %}
[add, 1, [add, 2, 3], 4]
{% endcodeblock %}
观察前面的代码，我们发现这时会引发问题的地方是下面这行代码，

{% codeblock lang:scheme %}
NewResult is ResultSoFar + H,
{% endcodeblock %}

它从操作数列表中取出第一个元素（H），然后把它追加到ResultSoFar。但是这里并没有区分H的类型——数字还是列表，如果H是一个列表，那么我们还要先对它进行求值，

{% codeblock lang:scheme %}
getValue(H, Value), 
NewResult is ResultSoFar + Value,
{% endcodeblock %}

当然，如果H是一个数字，那么Value就是它自身，下面的谓词表述了这个含义，

{% codeblock lang:scheme %}
getValue(Number, Number) :- number(Number).
{% endcodeblock %}
这样，我们就扩展了getValue谓词的抽象级别，而且也反映了scheme语言的“函数性”。在解释执行scheme的过程中，要做的事情就是不断地求值，表达式有值，一个数字也有值（就是它自身），后面会看到遇到变量时也要对它求值。

下面要让运行时可以执行除了加法以外其他的四则运算了。在前面的程序中，“add”其实只是个花瓶，对后面的计算过程没有丝毫影响，事实上“NewResult is ResultSoFar + H”会无条件地执行加法。因此我们要把操作符add（以及minus，multi，div）传递给后面的谓词，当然，传递前我们判断它们是否是合法的操作符：

{% codeblock lang:scheme %}
operator(add).
operator(minus).
operator(multi).
operator(div).
{% endcodeblock %}

上面四个谓词描述了目前我们支持的四种操作符。下面所有元数为3（参数个数为3）的getValue谓词都要增加一个参数，即操作符。最后，还需要一组谓词来根据不同的操作符完成最终的计算，

{% codeblock lang:scheme %}
getValue([Operator, FirstOperand|OtherOperands], Result) :- 
    operator(Operator),
    getValue(Operator, FirstOperand, OtherOperands, Result).

getValue(Operator, ResultSoFar, [H|T], Result) :-
    getValue(H, Value),
    calculate(Operator, ResultSoFar, Value, NewResult),
    getValue(Operator, NewResult, T, Result).

calculate(add, Oper1, Oper2, Result) :- Result is Oper1 + Oper2.
calculate(minus, Oper1, Oper2, Result) :- Result is Oper1 - Oper2.
calculate(multi, Oper1, Oper2, Result) :- Result is Oper1 * Oper2.
calculate(div, Oper1, Oper2, Result) :- Result is Oper1 / Oper2.
{% endcodeblock %}

如上代码所示，真正的运算是在calculate谓词中完成的，它会根据不同的操作符进行不同的计算。现在，我们还有一种语句需要处理，(define a 1)，目前的谓词不能识别第一个元素是define的列表。所以我们要一个谓词来“忽略”这样的语句：

{% codeblock lang:scheme %}
getValue([_|_], _).
{% endcodeblock %}

单独一个运算语句已经能够处理了，我们可以简单地把它扩展到多个语句的情况。一个run谓词可以完成这件事：

{% codeblock lang:scheme %}
run([]).
run([FirstStmt|OtherStmts]) :-
    getValue(FirstStmt, Result),
    writeln(Result),
    run(OtherStmts).
{% endcodeblock %}

现在，这个运行时环境可以解释执行下面“复杂”的代码了：

{% codeblock lang:scheme %}
(define a 1) 
(* 3 5) 
(+ 1 (- 5 2) (* 3 (/ 20 5)))
{% endcodeblock %}

计算的结果是15和16，而第一行define则被忽略掉了。

到目前为止，我们完成了源代码读取、词法分析、文法分析、运行时解析的步骤，这个简单的解析器能够计算不含变量的四则运算。但是作为一个体面的运行时，我们要给它增加定义变量的能力。关于变量的作用域，在本文的开始已经描述过。我们知道，“符号表”是支持变量的核心要素，本文的最后一部分我将讲述如何构建符号表。

符号表本质上就是个字典，里面保存了变量名和变量值的映射。Prolog并没有直接提供字典，但是我们可以用一个二维数组来模拟：

{% codeblock lang:scheme %}
[[ident(a), 1], [ident(b), 2]]
{% endcodeblock %}

ident(a)和ident(b)表示变量a和b，这种表示是为了和a、b的原子形式区分。接下来，创建符号表的谓词并不复杂：

{% codeblock lang:scheme %}
createSymbolTables(Stmts, ST) :-
    createSymbolTable(Stmts, [], ST).

createSymbolTable([], ST, ST).

createSymbolTable([[define, ident(Id), Number]|OtherStmts], STSoFar, ST) :-
    number(Number),
    createSymbolTable(OtherStmts, [[Id,Number]|STSoFar], ST).

createSymbolTable([_|T], ST) :- createSymbolTables(T,ST).
{% endcodeblock %}

其中最后一个谓词是为了在创建符号表的过程中，只关心define语句，忽略掉其他运算语句。

关于符号表，创建它只是故事的一半，通过符号表查询某个变量的值是另一件重要的事。

{% codeblock lang:scheme %}
search(_, [], _) :- false.
search(Id, [[Id,Value]|_], Value).
search(Id, [_|T], Value) :- 
    search(Id, T, Value).
{% endcodeblock %}

当遍历到最后一个符号表的时候，如果仍然找不到某个变量的值，这将被认为是一个程序错误，所以，上面第一个search谓词是以false结束的。另外，如果search在第一个谓词时失败的话，它已经没有回朔的余地了，这样将继而导致整个Prolog运行时失败。这个行为是合理的，任何时候，当遇到一个没有定义的变量时程序都应该停机。

创建完成了符号表，我们现在可以扩展运行时来支持定义变量了。运行时现在应该遍历两次AST，第一次创建符号表ST，第二次解释求值：

{% codeblock lang:scheme %}
scheme(Stmts) :-
    createSymbolTables(Stmts, ST),
    run(Stmts, ST).
{% endcodeblock %}

我们看到run谓词已经多了一个ST参数，ST也会进而传递给下面的getValue谓词。在前面的代码中，getValue可以对四则运算和数字求值了，现在我们要扩展它，让它可以对变量求值，当然，这是指从符号表中找出这个变量对应的值：

{% codeblock lang:scheme %}
getValue(ST, ident(Id), Value) :- search(Id, ST, Value).
{% endcodeblock %}

正如刚刚提到的，getValue多了一个参数ST，其他的getValue也是一样，尽管它们并不需要它。

好了，深呼吸一下吧，再整理整理手头的代码。如果你跟上了这篇文章的步伐，会看到一个Scheme子集的运行时环境。如果没跟上也没关系，下面的[链接](https://github.com/isaachan/scheme-runtime)提供了源代码。
