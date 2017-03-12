---
layout: post
title: "ESB不好用"
date: 2017-03-12 19:08
comments: true
categories: 
 - arch
 - enterprise
 - integration
 - esb
---

加入一家百年的制造业公司的IT部门已经一个多月了。这段时间一直在梳理这家公司下的数十个大大小小的系统。在这些纷繁复杂的系统之中，有几个系统是由一套企业消息总线（ESB）集成起来的。	为了熟悉这个对我来说很陌生的庞然大物，我询问了ESB系统项目的成员，听到一些有趣的信息。当然，也坚定了我的一个信念——ESB难以支撑今天的企业级应用了。

虽然ESB曾经是企业级应用程序的定海神针，它能够把大量的系统集成起来。不过，今天的企业级应用却在发生变化，让ESB开始有些力不从心了。这些变化包括：

1. 从企业内部走向企业外部 <!-- more -->

这应该是最本质的变化了。以前企业级应用主要是用来服务企业内部业务运行的，其用户总量是非常有限的。但是今天，企业的很多业务要通过互联网在线完成，很有以往的内部服务要开放给外部用户，用户总量会有几个数量级的增加。即使对于内部员工来说，随着BYOD的流行，员工可以随时接入公司的服务，这让企业内部的系统承受比以前大很多的压力。正如Jeff Dean在其著名的演讲“[Challenges in Building Large-Scale Information Retrieval Systems](http://videolectures.net/wsdm09_dean_cblirs/)”中所说“Right design at X may be very wrong at 10X or 100X, and design for 10X growth, but plan to rewrite before 100X”。

2. 数据、机器、性能...

随着用户量的增加，相应的数据量、托管服务所需的计算资源量也大幅度增加了。数据量从以前的MB，很快进入到GB甚至TB级别；托管主机则从十几台服务器到现在的数以千计的虚拟机、容器等。同时对性能的要求却越来越严格，从几秒钟的相应时间到几毫秒都令人感到漫长。

3. 人也在变




