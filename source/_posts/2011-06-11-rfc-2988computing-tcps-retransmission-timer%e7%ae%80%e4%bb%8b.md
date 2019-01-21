---
id: 298
title: 'rfc 2988(Computing TCP&#8217;s Retransmission Timer)简介'
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=298'
permalink: /2011/06/11/rfc-2988computing-tcps-retransmission-timer%e7%ae%80%e4%bb%8b/
categories:
  - 协议
tags:
  - rfc
  - tcp
  - tcp/ip
  - 协议
translate_title: introduction-to-rfc-2988-(computing-tcp-&-s-retransmission-timer)
date: 2011-06-11 14:22:35
---
rfc 2988是描述tcp如何计算定时器的一个rfc，是2000年发布的，而最近2988 已经被更新:
  
http://tools.ietf.org/html/draft-paxson-tcpm-rfc2988bis-02

并且google的jerry chu(2988bis的作者之一)最近在内核提交一个patch，主要是用来修改3次握手时的初始化rto的值(以及rfc 2988bis的一些改变)，当前的内核默认是3，而patch修改这个值为1，主要的修改依据是rfc2988.
  
<!--more-->


  
在net-next-2.6的分支，我们git log可以看到对应的提交log：

> Author: Jerry Chu <hkchu@google.com>
  
> Date: Wed Jun 8 11:08:38 2011 +0000
> 
> tcp: RFC2988bis + taking RTT sample from 3WHS for the passive open side
> 
> This patch lowers the default initRTO from 3secs to 1sec per
      
> RFC2988bis. It falls back to 3secs if the SYN or SYN-ACK packet
      
> has been retransmitted, AND the TCP timestamp option is not on.
> 
> It also adds support to take RTT sample during 3WHS on the passive
      
> open side, just like its active open counterpart, and uses it, if
      
> valid, to seed the initRTO for the data transmission phase.
> 
> The patch also resets ssthresh to its initial default at the
      
> beginning of the data transmission phase, and reduces cwnd to 1 if
      
> there has been MORE THAN ONE retransmission during 3WHS per RFC5681.

可以

> git show 9ad7c049f0f79c418e293b1b68cf10d68f54fcdb

来看对应的patch。有兴趣的话，可以看看这个patch.

接下来我主要是简要的介绍下这个rfc。

1 首先发送者应当设置rto初始值为 1(以前默认值是3) .

将三次握手时的rto初始值由3降低为1，是由于下面几个原因的：

1.1 三次握手时的rto的初始值为3，是在RFC 1122中定义的，而rfc1122是1989年发表的，现在的网络比当初的网络要快太多了。

1.2 google的统计显示大多数的(97.5%)的连接的RTT都是小于1秒的.

1.3 并且统计也显示在三次握手期间的重传率也就只有2%左右.

可是依然有2.5%的连接的RTT是大于1秒的，如果是这些连接，则将会在连接建立的期间引发一个重传，这时就需要将初始化的RTO再次重置为3，不过此时如果tcp头里面包含timestamps域，则不需要将RTO重置为3，因为协议栈此时可以通过timestamps来计算对应的rto.

2 RTO的值必须大于等于1，如果小于1，则向上取整为1。

3 tcp必须使用karn算法，来进行定时器退避策略.

4 如果存在timestamp option，则每次ack都会重新计算rtt.

然后就是核心的定时器管理部分：

1 发送数据时(包括重传)，需要启动重传定时器，当所有发送未确认数据被ack，则删除定时器.

2 如果定时器超时，则重传最早的还没有被receiver ack的段,并且host必须设置RTO = RTO *2 (karn, 定时器退避),而rto的最大值也必需有一个限制(linux下是120秒).

3 当三次握手期间定时器超时，并且初始rto是1,此时RTO必须重新设置为3