---
layout: post
title: "幻灯片文本化"
date: 2013-08-17 22:12
comments: true
categories: 
  - scheme
  - slideshow
  - textuality
---

作为一名体面的程序员，作为一名被Unix文化侵染的程序员，作为一名希望世界是由简单的纯文本构成的程序员，生活中充满太多无奈。二进制的格式无处不在，后缀名将文件分出了三六九等。这其中，有两种文件我最无奈，图像和幻灯片，它们总和我的工作生活密切相关。

不过，现在光来了！Kukai在一篇blog里介绍了一个很酷的工具——Dot，它提供了一种可以基于文本描述图形的简洁的DSL。利用这个DSL，可以绘制出相当复杂的图案。激动之余，我开始寻找文本化幻灯片的方法。于是，我发现了DrRacket（它的前身就是大名鼎鼎的DrScheme）。DrRacket是MIT开发的Scheme的运行时，其中包含一个制作幻灯片的组件，名曰Slideshow。

很明显，Slideshow要求使用者用scheme来编写幻灯片，这就满足了像我这样的语言控的怪癖。我也因此放弃了一些其他类似的工具，比如SliTex。在Slideshow中，最核心的概念是pict和slide。slide自然表示一张幻灯片，它是由一个或多个pict按照不同的顺序排列组成的。slideshow的源代码以rkt为后缀名。下面是slideshow的Hello World（hello.rkt）：

{% codeblock lang:scheme %}
#lang slideshow 

(slide (t “Hello World”))
{% endcodeblock %}
<!-- more -->

第一行代码注册了语言的类型，即slideshow，第二行代码包含一个slide函数的调用，它会产生一张幻灯片，幻灯片的内容是(t “Hello World”)函数的结果——包含字符串“Hello World”的pict。t是Slideshow中众多返回pict的函数之一。在默认情况下，slide函数会把pict置于幻灯片的中间，不过它有一个可选的选项layout来控制pict的位置，让我们在刚才的基础上再增加一张幻灯片：

{% codeblock lang:scheme %}
… 
(slide 
   #:layout 'top 
   (t “Hello World”))
{% endcodeblock %}

layout的值包括'center，'top，'tall和'auto，'auto是默认值。slide还有其他选项，title、name、inset和timeout等，可以为slide提供更多可配置的功能。

如果你对t函数使用的字体和大小不满意的话，可以使用text函数来自定义它们。

{% codeblock lang:scheme %}
… 
(slide 
   #:title “Code Snap” 
   (text “String name = getNameFromDB();” '(bold . morden), (+ (current-font-size 10))))
{% endcodeblock %}

我为这张幻灯片增加了title “Code Snap”，它会出现在幻灯片的顶端。然后使用text函数生成一段Java代码片段，为了让它看上去更像“代码”，使用等宽字体morden并加粗，大小也比默认字号大了10。这时在命令行里键入下面的命令

{% codeblock lang:bash %}
> slideshow hello.rkt
{% endcodeblock %}

就能看到前面制作的三张幻灯片依次播放了。如果你现在对Slideshow产生了一点兴趣，那么别犹豫了，在命令行里输入

{% codeblock lang:bash %}
> slideshow
{% endcodeblock %}

就立刻能够看到一个介绍Slideshow入门教程幻灯片，它非常给力，而且它就是用Slideshow制作的，你可以在drrakect的安装目录下找到它的源代码。

希望你能花点儿时间学习一下Slideshow，然后从此爱上幻灯片，爱上Scheme！


