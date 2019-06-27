---
id: 430
title: tcp中RTO的计算以及linux下的实现
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=430'
permalink: >-
  /2012/08/01/tcp%e4%b8%adrto%e7%9a%84%e8%ae%a1%e7%ae%97%e4%bb%a5%e5%8f%8alinux%e4%b8%8b%e7%9a%84%e5%ae%9e%e7%8e%b0/
posturl_add_url:
  - 'yes'
categories:
  - kernel
  - 协议
tags:
  - kernel
  - tcp
  - tcp/ip
translate_title: rto-calculation-in-tcp-and-implementation-under-linux
date: 2012-08-01 03:35:18
---
计算RTT以及RTO的代码比较简单，我们先来看原理，首先相关的rfc有两篇分别是rfc793以及rfc6298，而相关的paper有一篇，那就是Van Jacobson和Michael J. Karels的 Congestion Avoidance and Control这篇paper，这篇1988年的paper中描述的RTT计算方法，就是我们当前所使用的计算方法，可能有的操作系统有一点修改，不过基本的东西都一样。

首先RTT是什么，RTT简单来说，就是我发送一个数据包，然后对端回一个ack，那么当我接到ack之后，就能计算出从我发送出包到接到过了多久，这个时间就是RTT。RTT的计算是很简单的，就是一个时间差。

而RTO呢，RTO也就是tcp在发送一个数据包之后，会启动一个重传定时器，而RTO就是这个定时器的重传时间，那么这个时候，就有问题了，由于RTO是指的这次发送当前数据包所预估超时时间,那么RTO就需要一个很好的统计方法，来更好的预测这次的超时时间。

我们所能想到的最简单的方法，那就是取平均数，比如第一次RTT是500毫秒，第二次是800毫秒，那么第三次发送的时候，RTO就应该是650毫秒。其实经典的RTO计算方法和取平均有点类似，只不过因子不太一样，取平均的话，也就是老的RTO和新的RTT都是占50%的权重，而在经典的RTO计算中就有些变化了。
  
<!--more-->


  
来看经典的RTT计算方法,这个计算方法是在RFC793中提出的，计算方法是这样子的：
  
```
    
SRTT = ( ALPHA \* SRTT ) + ((1-ALPHA) \* RTT)
      
RTO = min[UBOUND,max[LBOUND,(BETA*SRTT)]]
  
```

其中ALPHA是一个scala因子，一般来说建议ALPHA是0.8和0.9.UBOUND就是RTO的最大值，LBOUND是RTO的最小值，BETA也是一个因子，建议是1.3-2.0。

这里可以看到在经典的RTO计算方法中会有一个很大的问题，就是当RTT对RTO的影响太小了，也就是说经典的RTO计算方法在RTT变化比较大的网络中，会表现的非常不好的。

于是在1988年，大神Van jacobson和Michael J. Karels的 Congestion Avoidance and Control这篇paper中，描述了一种新的计算方法，这里对于rtt的采样，多添加了一个因素，那就是均差(mean deviation)，而这里并不是传统意义上的均差，使用的平均数其实就是我们在经典方法中的srtt。 计算方法是这样子的：

```
  
Err ≡ m−a
  
a←a+gErr
  
v ← v+g(|Err|−v)
  
```

其中m就是当前最新的RTT，然后 a就是经典方法中的的SRTT，然后v就是均差，而Err顾名思义，就是我们预测的RTT的错误偏差。
  
可是这里有个问题，那就是由于因子g的存在，那么就有可能出现浮点数，因此我们只需要做个变化，让g=1/2^n,那么上面的的等式就变为下面这个了
  
```
  
2^na ← 2^na+Err
  
2^nv ← 2^nv+(|Err|−v)
  
```

假设g=1/8,也就是n为3，并且sa为2^n\*a,sv为2^n\*v,那么对应的c伪码就如下，：

```
  
/∗ update Average estimator ∗/
  
m −= (sa >> 3);
  
sa += m;
  
/∗ update Deviation estimator ∗/
  
if (m < 0)
     
m = −m;
  
m −= (sv >> 2);
  
sv += m;
  
rto = (sa >> 3) + sv;
  
```

然后我们再来看rfc6298中对RTO计算的描述，这个rfc基本上就和Van jacobson和Michael J. Karels所描述的计算方法一致。其中第一次RTO的计算方法是这样子的：
  
```

SRTT <- R
                   
RTTVAR <- R/2
                   
RTO <- SRTT + max (G, K*RTTVAR)

```
  
后续的RTO计算是这样子的:

```

RTTVAR <- (1 &#8211; beta) \* RTTVAR + beta \* |SRTT &#8211; R&#8217;|
              
SRTT <- (1 &#8211; alpha) \* SRTT + alpha \* R&#8217;
              
RTO <- SRTT + max (G, 4*RTTVAR) 

```

可以看到和上面的公式有差别，这里RTTVAR就是上面的v，而SRTT就是上面的a。
  
其实这个公式和jacobson的公式是一样的，只需要对公式做个变化。比如
  
```
  
SRTT <- (1 &#8211; alpha) \* SRTT + alpha \* R&#8217; ```这个公式我们可以这样子来：
  
```
  
SRTT <- SRTT &#8211; SRTT\*alpha + alpha\*R\` => SRTT <- SRTT + alpha*(R\` &#8211; SRTT)
  
```
  
于是我们可以看到这个公式就和上面的公式a是相同的了。

最后我们来看linux下的实现，linux的实现和jacobson所描述的伪码基本一致，这里beta因子是1/4,而alpha因子是1/8.
  
linux下对应的计算是在tcp_rtt_estimator这个函数中，这个函数的第二个参数，就是当前最新的rtt值。linux的实现多了两个东西，分别是mdev和medev_max, 其中mdev就是上面计算方法中的RTTVAR，而mdev_max则就是每次均差的最大值(linux最小是50毫秒).而在linux中RTTVAR也就等于mdev_max.

这里要注意一个东西，那就是RTT的计算是每次都会做的，而对于RTO来说是和数据段相关的，也就是说每次发送的时候估计rtt，然后当估算RTT时发送的段收到ack，之后就应该进行下一次估算了(也就是一个RTT过去了),也就是需要调节对应的rttvar，并且重新初始化一些值.

```

u32 srtt; /\* smoothed round trip time << 3 \*/
	  
u32 mdev; /\* medium deviation \*/
	  
u32 mdev_max; /\* maximal mdev for the last rtt period \*/
	  
u32 rttvar; /\* smoothed mdev_max \*/
	  
u32 rtt_seq; /\* sequence number to update rttvar \*/

static void tcp_rtt_estimator(struct sock *sk, const __u32 mrtt)
  
{
	  
struct tcp_sock *tp = tcp_sk(sk);
	  
long m = mrtt; /\* RTT \*/

/* The following amusing code comes from Jacobson&#8217;s
	   
* article in SIGCOMM &#8217;88. Note that rtt and mdev
	   
* are scaled versions of rtt and mean deviation.
	   
* This is designed to be as fast as possible
	   
* m stands for "measurement".
	   
*
	   
* On a 1990 paper the rto value is changed to:
	   
\* RTO = rtt + 4 \* mdev
	   
*
	   
* Funny. This algorithm seems to be very broken.
	   
* These formulae increase RTO, when it should be decreased, increase
	   
* too slowly, when it should be increased quickly, decrease too quickly
	   
* etc. I guess in BSD RTO takes ONE value, so that it is absolutely
	   
* does not matter how to _calculate_ it. Seems, it was trap
	   
* that VJ failed to avoid. 8)
	   
*/
	  
if (m == 0)
		  
m = 1;
	  
if (tp->srtt != 0) {
  
//开始更新a也就是srtt.可以看到因子为1/8.
		  
m -= (tp->srtt >> 3); /\* m is now error in rtt est \*/
		  
tp->srtt += m; /\* rtt = 7/8 rtt + 1/8 new \*/
  
//开始更新mean deviation.
		  
if (m < 0) {
			  
m = -m; /\* m is now abs(error) \*/
			  
m -= (tp->mdev >> 2); /\* similar update on mdev \*/
			  
/* This is similar to one of Eifel findings.
			   
* Eifel blocks mdev updates when rtt decreases.
			   
* This solution is a bit different: we use finer gain
			   
\* for mdev in this case (alpha\*beta).
			   
* Like Eifel it also prevents growth of rto,
			   
* but also it limits too fast rto decreases,
			   
* happening in pure Eifel.
			   
*/
			  
if (m > 0)
				  
m >>= 3;
		  
} else {
			  
m -= (tp->mdev >> 2); /\* similar update on mdev \*/
		  
}
		  
tp->mdev += m; /\* mdev = 3/4 mdev + 1/4 new \*/
  
//找到最大值付给rttvar
		  
if (tp->mdev > tp->mdev_max) {
			  
tp->mdev_max = tp->mdev;
			  
if (tp->mdev_max > tp->rttvar)
				  
tp->rttvar = tp->mdev_max;
		  
}

//本次RTT的估算结束
		  
if (after(tp->snd_una, tp->rtt_seq)) {
  
//调节rttvar
			  
if (tp->mdev_max < tp->rttvar)
				  
tp->rttvar -= (tp->rttvar &#8211; tp->mdev_max) >> 2;
			  
tp->rtt_seq = tp->snd_nxt;
  
//设置最小值
			  
tp->mdev_max = tcp_rto_min(sk);
		  
}
	  
} else {
  
//第一次进来的情况
		  
/\* no previous measure. \*/
		  
tp->srtt = m << 3; /\* take the measured time to be rtt \*/
		  
tp->mdev = m << 1; /\* make sure rto = 3\*rtt */
		  
tp->mdev_max = tp->rttvar = max(tp->mdev, tcp_rto_min(sk));
		  
tp->rtt_seq = tp->snd_nxt;
	  
}
  
}
  
```

最后来看一下RTO的计算，RTO的计算在__tcp_set_rto中，这个计算它是严格遵守rfc6298中的描述。
  
```
  
static inline u32 __tcp_set_rto(const struct tcp_sock *tp)
  
{
	  
return (tp->srtt >> 3) + tp->rttvar;
  
}
  
```

最后这里有一个要注意的地方，那就是rto的最小值，一般来说，在linux下，这个值是200ms，也就是说最少要200毫秒才会发生重传，而这个值的修改是可以通过ip route来修改的，如何更改请看这里：

http://docs.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/4/html/Release_Notes/U7/ppc/ar01s04.html

而有关定时器退避这些，我在前面的blog有分析，感兴趣的可以看我前面的blog.