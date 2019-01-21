---
id: 346
title: linux下tcp选项TCP_DEFER_ACCEPT详解
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=346'
permalink: >-
  /2011/08/28/linux%e4%b8%8btcp%e9%80%89%e9%a1%b9tcp_defer_accept%e8%af%a6%e8%a7%a3/
posturl_add_url:
  - 'yes'
categories:
  - kernel
  - 协议
tags:
  - kernel
  - tcp
  - tcp/ip
translate_title: detailed-explanation-of-tcp-option-tcp_defer_accept-under-linux
date: 2011-08-28 16:04:46
---
TCP_DEFER_ACCEPT这个选项可能大家都知道，不过我这里会从源码和数据包来详细的分析这个选项。要注意，这里我所使用的内核版本是3.0.

首先看man手册中的介绍(man 7 tcp):

> TCP_DEFER_ACCEPT (since Linux 2.4)
    
> Allow a listener to be awakened only when data arrives on the socket. Takes an integer value (seconds), this can bound the maximum number of attempts TCP will make to complete the connection. This option should not be used in code intended to be portable. 

我先来简单介绍下，这个选项主要是针对server端的服务器，一般来说我们三次握手，当客户端发送syn，然后server端接收到，然后发送syn + ack,然后client接收到syn+ack之后，再次发送ack(client进入establish状态),最终server端收到最后一个ack，进入establish状态。

而当正确的设置了TCP_DEFER_ACCEPT选项之后，server端会在接收到最后一个ack之后，并不进入establish状态，而只是将这个socket标记为acked，然后丢掉这个ack。此时server端这个socket还是处于syn_recved，然后接下来就是等待client发送数据， 而由于这个socket还是处于syn_recved,因此此时就会被syn_ack定时器所控制，对syn ack进行重传,而重传次数是由我们设置TCP_DEFER_ACCEPT传进去的值以及TCP_SYNCNT选项，proc文件系统的tcp_synack_retries一起来决定的(后面分析源码会看到如何来计算这个值).而我们知道我们传递给TCP_DEFER_ACCEPT的是秒，而在内核里面会将这个东西转换为重传次数.

这里要注意，当重传次数超过限制之后，并且当最后一个ack到达时，下一次导致超时的synack定时器还没启动，那么这个defer的连接将会被加入到establish队列，然后通知上层用户。这个也是符合man里面所说的(Takes an integer value (seconds), this can bound the maximum number of attempts TCP will make to complete the connection.) 也就是最终会完成这个连接.

<!--more-->


  
我们来看抓包，这里server端设置deffer accept，然后客户端connect并不发送数据，我们来看会发生什么:

```
  
//客户端发送syn
  
19:38:20.631611 IP T-diaoliang.60277 > T-diaoliang.sunproxyadmin: Flags [S], seq 2500439144, win 32792, options [mss 16396,sackOK,TS val 9008384 ecr 0,nop,wscale 4], length 0
  
//server回了syn+ack
  
19:38:20.631622 IP T-diaoliang.sunproxyadmin > T-diaoliang.60277: Flags [S.], seq 1342179593, ack 2500439145, win 32768, options [mss 16396,sackOK,TS val 9008384 ecr 9008384,nop,wscale 4], length 0

//client发送最后一个ack
  
19:38:20.631629 IP T-diaoliang.60277 > T-diaoliang.sunproxyadmin: Flags [.], ack 1, win 2050, options [nop,nop,TS val 9008384 ecr 9008384], length 0

//这里注意时间，可以看到过了大概1分半之后，server重新发送了syn+ack
  
19:39:55.035893 IP T-diaoliang.sunproxyadmin > T-diaoliang.60277: Flags [S.], seq 1342179593, ack 2500439145, win 32768, options [mss 16396,sackOK,TS val 9036706 ecr 9008384,nop,wscale 4], length 0
  
19:39:55.035899 IP T-diaoliang.60277 > T-diaoliang.sunproxyadmin: Flags [.], ack 1, win 2050, options [nop,nop,TS val 9036706 ecr 9036706,nop,nop,sack 1 {0:1}], length 0

//再过了1分钟，server close掉这条连接。
  
19:40:55.063435 IP T-diaoliang.sunproxyadmin > T-diaoliang.60277: Flags [F.], seq 1, ack 1, win 2048, options [nop,nop,TS val 9054714 ecr 9036706], length 0

19:40:55.063692 IP T-diaoliang.60277 > T-diaoliang.sunproxyadmin: Flags [F.], seq 1, ack 2, win 2050, options [nop,nop,TS val 9054714 ecr 9054714], length 0

19:40:55.063701 IP T-diaoliang.sunproxyadmin > T-diaoliang.60277: Flags [.], ack 2, win 2048, options [nop,nop,TS val 9054714 ecr 9054714], length 0
  
```

这里要注意，close的原因是当synack超时之后，nginx接收到了这条连接，然后读事件超时，最终导致close这条连接。

接下来就来看内核的代码。

先从设置TCP_DEFER_ACCEPT开始，设置TCP_DEFER_ACCEPT是通过setsockopt来做的，而传递给内核的值是秒，下面就是内核中对应的do_tcp_setsockopt函数，它用来设置tcp相关的option，下面我们能看到主要就是将传递进去的val转换为将要重传的次数。
  
```
	  
case TCP_DEFER_ACCEPT:
		  
/\* Translate value in seconds to number of retransmits \*/
  
//注意参数
		  
icsk->icsk_accept_queue.rskq_defer_accept =
			  
secs_to_retrans(val, TCP_TIMEOUT_INIT / HZ,
					  
TCP_RTO_MAX / HZ);
		  
break;
  
```

这里可以看到通过调用secs_to_retrans来将秒转换为重传次数。接下来就来看这个函数，它有三个参数，第一个是将要转换的秒，第二个是RTO的初始值，第三个是RTO的最大值。 可以看到这里都是依据RTO来计算的，这是因为这个重传次数是syn_ack的重传次数。

这个函数实现很简单，就是一个定时器退避的计算过程(定时器退避可以看我前面的blog的介绍)，每次乘2，然后来计算重传次数。
  
```
  
static u8 secs_to_retrans(int seconds, int timeout, int rto_max)
  
{
	  
u8 res = 0;

if (seconds > 0) {
		  
int period = timeout;
  
//重传次数
		  
res = 1;
  
//开始遍历
		  
while (seconds > period && res < 255) {
			  
res++;
  
//定时器退避
			  
timeout <<= 1;
			  
if (timeout > rto_max)
				  
timeout = rto_max;
  
//定时器的秒数
			  
period += timeout;
		  
}
	  
}
	  
return res;
  
}
  
```

然后来看当server端接收到最后一个ack的处理，这里只关注defer_accept的部分,这个函数是tcp_check_req，它主要用来检测SYN_RECV状态接收到包的校验。

req->retrans表示已经重传的次数。
  
acked标记主要是为了syn_ack定时器来使用的。
  
```
  
//两个条件，一个是重传次数小于defer_accept，一个是序列号，这两个都必须满足。
	  
if (req->retrans < inet_csk(sk)->icsk_accept_queue.rskq_defer_accept &&
	      
TCP_SKB_CB(skb)->end_seq == tcp_rsk(req)->rcv_isn + 1) {
  
//此时设置acked。
		  
inet_rsk(req)->acked = 1;
		  
NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_TCPDEFERACCEPTDROP);
		  
return NULL;
	  
}
  
```

而当tcp_check_req返回之后，在tcp_v4_do_rcv中会丢掉这个包，让socket继续保存在半连接队列中。

然后来看syn ack定时器，这个定时器我以前有分析过（http://simohayha.iteye.com/admin/blogs/481989)
  
，因此我这里只是简要的再次分析下。如果需要更详细的分析，可以看我上面的链接,这个定时器会调用inet_csk_reqsk_queue_prune函数，在这个函数中做相关的处理。

这里我们就主要关注重试次数。其中icsk_syn_retries是TCP_SYNCNT这个option设置的。这个值会比sysctl_tcp_synack_retries优先.然后是rskq_defer_accept，它又比icsk_syn_retries优先.

```
  
void inet_csk_reqsk_queue_prune(struct sock *parent,
				  
const unsigned long interval,
				  
const unsigned long timeout,
				  
const unsigned long max_rto)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//最大的重试次数
	  
int max_retries = icsk->icsk_syn_retries ? : sysctl_tcp_synack_retries;
	  
int thresh = max_retries;
	  
unsigned long now = jiffies;
	  
struct request_sock *\*reqp, \*req;
	  
int i, budget;

&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//更新设置最大的重试次数。
	  
if (queue->rskq_defer_accept)
		  
max_retries = queue->rskq_defer_accept;

budget = 2 * (lopt->nr_table_entries / (timeout / interval));
	  
i = lopt->clock_hand;

do {
		  
reqp=&lopt->syn_table[i];
		  
while ((req = *reqp) != NULL) {
			  
if (time_after_eq(now, req->expires)) {
				  
int expire = 0, resend = 0;
  
//这个函数主要是判断超时和是否重新发送syn ack，然后保存在expire和resend这个变量中。
				  
syn_ack_recalc(req, thresh, max_retries,
					         
queue->rskq_defer_accept,
					         
&expire, &resend);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
				  
if (!expire &&
				      
(!resend ||
				       
!req->rsk_ops->rtx_syn_ack(parent, req, NULL) ||
				       
inet_rsk(req)->acked)) {
					  
unsigned long timeo;
  
//更新重传次数。
					  
if (req->retrans++ == 0)
						  
lopt->qlen_young&#8211;;
					  
timeo = min((timeout << req->retrans), max_rto);
					  
req->expires = now + timeo;
					  
reqp = &req->dl_next;
					  
continue;
				  
}
  
//如果超时，则丢掉这个请求，并对应的关闭连接.
				  
/\* Drop this request \*/
				  
inet_csk_reqsk_queue_unlink(parent, req, reqp);
				  
reqsk_queue_removed(queue, req);
				  
reqsk_free(req);
				  
continue;
			  
}
			  
reqp = &req->dl_next;
		  
}

i = (i + 1) & (lopt->nr_table_entries &#8211; 1);

} while (&#8211;budget > 0);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
}
  
```