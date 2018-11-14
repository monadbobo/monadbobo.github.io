---
id: 457
title: Early Retransmit for TCP原理以及实现
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=457'
permalink: >-
  /2013/05/11/early-retransmit-for-tcp%e5%8e%9f%e7%90%86%e4%bb%a5%e5%8f%8a%e5%ae%9e%e7%8e%b0/
posturl_add_url:
  - 'yes'
categories:
  - kernel
  - 协议
tags:
  - kernel
  - tcp
  - tcp/ip
translate_title: the-principle-and-implementation-of-early-retransmit-for-tcp
date: 2013-05-11 10:10:15
---
Early Retransmit for TCP(ER)是google为了解决快重传的一些局限，从而对快重传(fast retransmit)做出的一些改变，其中ER在linux kernel 3.5进入了内核,他的paper在这里：

http://tools.ietf.org/html/rfc5827

首先我们要知道快重传算法的弱点很多，比如如果发送端接收不到足够数量(一般来说是3个)的ack，那么快重传算法就无法起作用，这个时候就只能等待RTO超时。ER主要就是为了解决这个问题的。在下面的条件下，就会导致收不到足够的ack。

  * 拥塞窗口比较小
  * 窗口中一个很大的段丢失或者在传输的结尾处发生了丢包

<!--more-->


  
如果满足了上面的两个条件，那么就会发生发送端由于接收不到足够数量的ack，导致快重传算法无法生效。举个例子，比如拥塞窗口是3，然后第一个段丢失了，那么理论上最多发送段只可能收到2个重复的(duplicate)ack,此时由于快重传要求3个重复的ack，那么发送端将会等待RTO超时，然后重传第一个段.

在上面的第二个条件中，有两种可能性，其中ER是为了解决第一种可能性(也就是当一个比较大的段丢失),而第二种情况则需要TLP来解决(TLP应该会合并进下个版本的内核，我后续也会分析TLP).

下面是我对ER做的一些摘要笔记：

  * 在linux下early retransit 和thin dupack是互斥的。
  * early retransmit也就是会在小窗口下(flight count 小于4),减小快重传的数量，比如减小到1或者2。而在linux的实现里面，会人为的加上一个延迟，为了防止假超时。
  * ER打开的条件有两个，第一个是outstand的数据很小，第二个是前一个未发送的数据不能被发送
  * 由于early retransmit必须很小心，因此必须通过sack来猜测中间的一个段已经丢失才会启动early retransmit，比如有3个 in flight的段，那么sack必须有2个段，也就是这样才会启动early retrainsmit.
  * 快重传也只是每次新的数据发送过来之后才会激活，因此最少窗口要有4个段。
  * 当接收端收到一个outstand的数据的话，必须立即发送一个重复的ack，这个就是为了快重传(rfc2581/5681)
  
    一般来说是通过sack来判断快重传(sack_outs),也就是说每次快重传的ack(重复ack)都会带sack.

接下来来描述一下ER的算法，ER可以基于两种模式，一种是基于字节的，一种是基于段(segment-based)的，两种算法基本差不多，我这里主要描述基于段的，因为linux kernel中实现的就是基于段的。

当一个ack到来后，发送端启动early retransmit(ER)只有满足下面两个条件。

  * outstanding段的个数(oseg),小于4.
  * 要么没有未发送的数据，要么advertised 接受窗口不允许新的段被发送.

当上面的两个条件满足并且当前tcp链接不提供sack，那么duplicate ack threshold(快重传的ack个数阈值),必须被减少为 ER_thresh=oseg &#8211; 1.

当上面的两个条件满足，并且当前tcp连接支持sack，那么只有当(oseg-1)个数的段已经被sacked时，ER才会被使用。

ER的优点我们都知道了，接下来来看看ER的缺点，或者说可能会引起的问题。当应用层(application)不是连续的发送数据的时候，ER会导致很多不必要的重传。举个例子，假设一个application发送两个段的数据，然后紧跟着一段idle期。那么此时如果网络reorder这两个段，那么发送者将会发送没必要的一个段(ER的重传).如果application持续的这样发送数据，那么将会有1/3的段是没必要发送的。

接下来来看对应的linux kernel实现，我这里内核版本是3.7.

首先内核增加了一个sysctl的选项tcp_early_retrans，这个选项用来开关ER，这个选项默认值是2，也就是打开ER，并且打开delay定时器。

> tcp_early_retrans &#8211; INTEGER
	  
> Enable Early Retransmit (ER), per RFC 5827. ER lowers the threshold
	  
> for triggering fast retransmit when the amount of outstanding data is
	  
> small and when no previously unsent data can be transmitted (such
	  
> that limited transmit could be used).
	  
> Possible values:
		  
> 0 disables ER
		  
> 1 enables ER
		  
> 2 enables ER but delays fast recovery and fast retransmit
		    
> by a fourth of RTT. This mitigates connection falsely
		    
> recovers when network has a small degree of reordering
		    
> (less than 3 packets).
	  
> Default: 2 

首先来看ER的初始化函数，当每一个socket建立的时候，都会调用tcp_enable_early_retrans来设置对应的ER选项值，这里主要是两个值，一个是do_early_retrans，表示是否打开ER，一个是early_retrans_delayed，表示delay是否已经打开。

```
  
static inline void tcp_enable_early_retrans(struct tcp_sock *tp)
  
{
	  
tp->do_early_retrans = sysctl_tcp_early_retrans &&
		  
!sysctl_tcp_thin_dupack && sysctl_tcp_reordering == 3;
	  
tp->early_retrans_delayed = 0;
  
}
  
```

通过上面的代码我们可以看到ER打开的三个条件分别是

  * 首先sysctl的选项必须打开
  * 第二thin dupack必须关闭
  * 第三tcp_reordering必须为3

然后我们来看tcp_time_to_recover这个函数中ER的相关部分。tcp_time_to_recover如果返回true则我们可能会进入快重传，否则将会继续保持open状态.

```
	  
if (tp->do_early_retrans && !tp->retrans_out && tp->sacked_out &&
	      
(tp->packets_out == (tp->sacked_out + 1) && tp->packets_out < 4) &&
	      
!tcp_may_send_now(sk))
		  
return !tcp_pause_early_retransmit(sk, flag);
  
```

上面的代码很好理解(上一篇blog分析过)，因为判断条件基本和paper描述一致，其中下面这个条件说明，如果收到oseg -1 个dup ack，就会进入ER处理。因为packets_out就表示oseg的段的个数。

```
  
tp->packets_out == (tp->sacked_out + 1) && tp->packets_out < 4
  
```

然后来看tcp_pause_early_retransmit这个函数，它主要是ER对快重传做一定的延迟.这里通过early_retrans_delayed来标记这个延迟.这个延迟时间一般是设置为RTT/4. 也就是当ER判断需要快重传的时候，并不会立即启动快重传，而是启动一个delay定时器，等定时器超时后再重传。

```
  
static bool tcp_pause_early_retransmit(struct sock *sk, int flag)
  
{
	  
struct tcp_sock *tp = tcp_sk(sk);
	  
unsigned long delay;

/* Delay early retransmit and entering fast recovery for
	   
* max(RTT/4, 2msec) unless ack has ECE mark, no RTT samples
	   
* available, or RTO is scheduled to fire first.
	   
*/
	  
if (sysctl_tcp_early_retrans < 2 || (flag & FLAG_ECE) || !tp->srtt)
		  
return false;
  
//设置延迟
	  
delay = max_t(unsigned long, (tp->srtt >> 5), msecs_to_jiffies(2));
	  
if (!time_after(inet_csk(sk)->icsk_timeout, (jiffies + delay)))
		  
return false;
  
//启动定时器
	  
inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS, delay, TCP_RTO_MAX);
	  
tp->early_retrans_delayed = 1;
	  
return true;
  
}
  
```

然后我们来看当delay定时器到期后会发生什么,由于通过上面的代码我们可以看到ER的delay定时器是重传定时器，因此当delay 超时后，将会在重传定时器回调中调用tcp_resume_early_retransmit。

```
  
void tcp_resume_early_retransmit(struct sock *sk)
  
{
	  
struct tcp_sock *tp = tcp_sk(sk);

tcp_rearm_rto(sk);

/\* Stop if ER is disabled after the delayed ER timer is scheduled \*/
	  
if (!tp->do_early_retrans)
		  
return;

tcp_enter_recovery(sk, false);
	  
tcp_update_scoreboard(sk, 1);
	  
tcp_xmit_retransmit_queue(sk);
  
}
  
```