---
layout: post
title: "文本化的幻灯片 - 实用技术"
date: 2013-08-18 21:56
comments: true
categories: 
  - scheme
  - slideshow
  - textuality
  - racket
---

不久前在博客上发了一篇使用Slideshow创建幻灯片的[文章](http://isaachan.github.io/blog/2013/08/17/slide-textuality/)，不过这篇博文却是在两年多前写的。这两年多里，我在公司内部的演讲幻灯片几乎都是用Slideshow做的，外部演讲也用过几次了，比如[Lisp社区的](http://www.lisp.org.cn/wiki/lisp/event/2013-meetup)。在这个过程中，我对Slideshow的强大功能与高效率越来越有信心了。这篇文章分享一些我总结的经验。

## 1. 关于使用 ##

Slideshow本身要求使用者编写scheme代码来创建幻灯片，不过这部分不设计代码，而是一些方便使用的技巧。

#### 显示下一页预览 ####

在幻灯片演示的过程中，按住Alt-c（或者Cmd-c）可以开启、关闭下一页幻灯片的预览。图1显示了启动预览时的效果。
![下一页预览](/images/slide-textuality-tips/slideshow-preview.png)
图1

#### 显示注释 ####

在幻灯片演示的过程中，按住Alt-d（或者Cmd-d）可以显示、隐藏注释内容。至于在幻灯片里添加注释的内容，需要调用comment函数，

{% codeblock lang:scheme %}
(slide
    (t "Hello World!")
    (comment "this is a comment")
)
{% endcodeblock %}
<!-- more -->

#### 移动窗口位置 ####

在幻灯片演示的过程中，按住Shift和一个方向键，可以向方向键的方向移动窗口，每次移动一个像素。当演讲时投影仪设备或角度受限时，这个方法能够临时救急。

#### 导出为PDF格式 ####

随着越来越多的办公软件与格式的流行，人们共享像幻灯片这类文件时兼容性问题越来越多，好在在很多时候，PDF是个几乎所有人都能接受的格式，于是，几乎所有办公软件都有导出PDF格式的功能。Slideshow也不例外。命令行参数"-p"可以告诉Slideshow不要演示幻灯片，而是打印它。打印前只要选择“Save as Pdf”，就可以到处文件，而不必真的执行打印了。

{% codeblock lang:scheme %}
> slideshow -p helloworld.rkt
{% endcodeblock %}

## 2. 关于编码 ##

前面介绍了一些Slideshow的使用技巧，但是这些确实无法和PowerPoint或者Keynote这类成熟的幻灯片工具比较。而Slideshow真正的优势在于可以通过编写scheme代码来创建幻灯片，因此幻灯片就可以具有代码的那种简洁、抽象、确定、优雅的特性了。这里我列举四个例子，

#### 介绍题目的幻灯片 ####

你的幻灯片第一页怎么做？演讲题目、演讲者姓名、公司、邮箱、微博、主页等等。然后排版，标题居中、名字靠右，还是...其实对于你自己来说，每次的不同只有演讲题目而已，其他的像公司、邮箱、样式等等都可以是一样的，这样能够打造你自己一致的风格。而提取相同的部分，隔离不同的部分——幻灯片已经全部变成代码了——当然可以做到：

{% codeblock lang:scheme %}
(define title
  (lambda content
    (slide (vr-append (* gap-size 5)
             (vc-append
               (* gap-size 3)
               (adapt content)           
             )
             (vr-append (* gap-size 0.5)
                (text "Han Kai" (cons 'bold (current-main-font)) 32)
                (text "http://isaachan.github.com" (current-main-font) 24)
                (hc-append
                  (text "Thought" (cons 'bold (current-main-font)) 24)
                  (text "Works" (current-main-font) 24)
                )
                (text "@isaachan" (cons 'bold (current-main-font)) 32)
             )
        )
    )
  )
)
{% endcodeblock %}

这段代码定义了title函数，它接受一个字符串类型的参数，即演讲题目内容，然后将题目以64号字居中显示，右下角以此是演讲名姓名、Blog主页、公司名和微博。那么，当用不同参数调用title函数时，可以看到风格一致、只有题目不同的幻灯片了：（图2）

{% codeblock lang:scheme %}
(title "Logic programming and Prolog")
{% endcodeblock %}

![title_1](/images/slide-textuality-tips/logic_programming_and_prolog.png)
图2

当然，如果title部分需要做一些定制，也没有问题：（图3）

{% codeblock lang:scheme %}
(title
  (text "How to" (current-main-font) 50)
  (text " write your own language in 10 mins" (current-main-font) 50)
)
{% endcodeblock %}

![title_2](/images/slide-textuality-tips/how_to_write_your_own_language_in_10_mins.png)
图3

#### 一页一句话 ####

我们都知道一个演讲的基本原则，就是人是主题，幻灯片是辅助物，因此，应该尽量保持幻灯片简洁，不要密密麻麻地堆彻大量文字。这个原则的一个极限状态就是幻灯片上只有一句话，表明一个主题。我们希望让所有这些幻灯片里的文字都保持同样的样式。这只需一个非常简单的函数就足够了，

{% codeblock lang:scheme %}
(define (topic content) 
  (slide (text content (current-main-font) 70))
) 
{% endcodeblock %}

这个函数只接受一个字符串类型的参数，为显示在幻灯片上的文字内容。另外它将字号设置为70，使用的是系统默认的字体。

#### 修改Master版面 ####

PowerPoint和Keynote都有模版功能，可以对所有的幻灯片做统一的设置。Slideshow也有类似的机制。Slideshow在渲染每一张幻灯片之前，都会调用一个名为current-slide-assembler的函数，Slideshow运行时调用这个函数时，会传入三个参数，slide的title、title与内容的分隔符v-sep、slide的内容。下面的例子完成如下的设置，

 1. 黑底白字
 2. Title内容靠左，不是默认的居中

{% codeblock lang:scheme %}
(set-margin! 0)
(define black-bg
  (filled-rectangle client-w client-h)
)

(current-slide-assembler
  (lambda (s v-sep c)
    (ct-superimpose
      black-bg
      (if s
        (vl-append v-sep (para (titlet s)) (colorize c "white"))
        (colorize c "white")  
      )
    )
  )
)

(current-title-color "white")
{% endcodeblock %}

current-slide-assembler函数接受一个lambda类型的参数，这个lambda接受三个参数。black-bg函数会画一个和用户可见区域一样大小的黑色矩形——这就是“黑底”了。接下来将判断当前幻灯片是否有title，如果有，则将title和内容垂直左对齐；如果没有，则直接显示内容。注意，无论是否有Title，都会将文字的颜色改为白色。最后调用(current-title-color "white")，目的是把Title文字的颜色也改为白色。这就是“白字”了。下面的图4和图5是同样的幻灯片，在改变Master前后的效果，

![改变Master之前](/images/slide-textuality-tips/change-master-1.png)
图4　

![改变Master之后](/images/slide-textuality-tips/change-master-2.png)
图5

#### 布局 ####

有很多时候，我需要对幻灯片进行布局，比如图6示范了一些可能的形式:

![布局效果](/images/slide-textuality-tips/left-right.png)
图6

在Slideshow中实现第一种左右平分的布局非常简单，

{% codeblock lang:scheme %}
(define left-right-panel
  (lambda (left right)
    (slide
      (ht-append
        (cc-superimpose (blank (/ client-w 2) client-h) left)
        (cc-superimpose (blank (/ client-w 2) client-h) right)
      )
    )
  )
)
{% endcodeblock %}

left-right-panel函数接受两个参数，分别是左边和右边的内容。因此，

{% codeblock lang:scheme %}
(left-right-panel (t "Hello") (t "World"))
{% endcodeblock %}

就能实现图6-a的效果了。其中cc-superimpose表示“Hello”和“World”两段文本在各自的区域里面都出于水平居中、垂直居中的位置（cc）。不过，这个函数限制了内容只能在各自的区域内居中，因此无法实现像图6-b那种效果，只是可以修改left-right-panel，使它能够支持不同的居中模式，

{% codeblock lang:scheme %}
(define left-right-panel
  (lambda (left right [left-superimpose cc-superimpose] [right-superimpose cc-superimpose])
      (slide
      (ht-append
        (left-superimpose (blank (/ client-w 2) client-h) left)
        (right-superimpose (blank (/ client-w 2) client-h) right)
      )
    )
  )
)
{% endcodeblock %}

更新后的left-right-panel增加了两个参数left-superimpose和right-superimpose，并且默认值为cc-superimpose和cc-superimpose。如果要实现图6-b的效果，可以调用

{% codeblock lang:scheme %}
(left-right-panel (t "Hello") (t "World") lt-superimpose rt-superimpose)
{% endcodeblock %}

就可以了。另外，图6的其他几种布局的实现方式和left-right-panel非常类似。

## 3.总结 ##

上面四个例子只是很少的一部分。Slideshow的魅力在于让你可以把关于制作幻灯片的经验、模式，通过代码固化下来，并且降低重用和分享的难度。同时，Slideshow可以帮你把内容和样式进行干净的分离，随着使用经验的增加，你会积累更多的样式函数，那时就可以把精力放在幻灯片的内容上，从而提高生产率。





















