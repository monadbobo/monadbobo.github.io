---
id: 168
title: linux kernel 网络协议栈之xps特性详解
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=168'
permalink: >-
  /2010/12/12/linux-kernel-%e5%8d%8f%e8%ae%ae%e6%a0%88xps-patch%e8%af%a6%e8%a7%a3/
categories:
  - kernel
tags:
  - google
  - kernel
  - tcp/ip
translate_title: xps-characteristics-of-linux-kernel-network-protocol-stack
date: 2010-12-12 18:04:32
---
[xps](http://lwn.net/Articles/416646/)全称是Transmit Packet Steering，是rfs/rps的作者Tom Herbert提交的又一个patch，预计会在2.6.37进入内核。

这个patch主要是针对多队列的网卡发送时的优化，当发送一个数据包的时候，它会根据cpu来选择对应的队列，而这个cpu map可以通过sysctl来设置：
  
```/sys/class/net/eth<n>/queues/tx-<n>/xps_cpus```
  
这里xps_cpus是一个cpu掩码，表示当前队列对应的cpu。

而xps主要就是提高多对列下的数据包发送吞吐量，具体来说就是提高了发送的局部性。按照作者的benchmark，能够提高20%.

<!--more-->


  
原理很简单，就是根据当前skb对应的hash值(如果当前socket有hash，那么就使用当前socket的)来散列到xps_cpus这个掩码所设置的cpu上，也就是cpu和队列是一个1对1，或者1对多的关系，这样一个队列只可能对应一个cpu，从而提高了传输结构的局部性。

没有xps之前的做法是这样的，当前的cpu根据一个skb的4元组hash来选择队列发送数据，也就是说cpu和队列是一个多对多的关系，而这样自然就会导致传输结构的cache line bouncing。

这里还有一个引起cache line bouncing的原因，不过这段看不太懂： 

> Also when sending from one CPU to a queue whose
  
> transmit interrupt is on a CPU in another cache domain cause more
  
> cache line bouncing with transmit completion.

接下来来看代码，我这里看得代码是net-next分支，这个分支已经将xps合并进去了。

先来看相关的数据结构,首先是xps_map,这个数据结构保存了对应的cpu掩码对应的发送队列，其中queues队列就保存了发送对列.这里一个xps_map有可能会映射到多个队列。

```
  
struct xps_map {
  
//队列长度
	  
unsigned int len;
	  
unsigned int alloc_len;
	  
struct rcu_head rcu;
  
//对应的队列序列号数组
	  
u16 queues[0];
  
};
  
```

而下面这个结构保存了设备的所有的cpu map，比如一个设备 16个队列，然后这里这个设备的xps_dev_maps就会保存这16个队列的xps map(sysctl中设置的xps_map),而每个就是一个xps_map结构。
  
```
  
struct xps_dev_maps {
  
//rcu锁
	  
struct rcu_head rcu;
  
//所有对列的cpu map数组
	  
struct xps_map __rcu *cpu_map[0];
  
};
  
```

然后就是net_device结构增加了一个xps_dev_maps的域来保存这个设备所有的cpu map。

```
  
struct net_device {
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
#ifdef CONFIG_XPS
  
//保存当前设备的所有xps map.
	  
struct xps_dev_maps __rcu *xps_maps;
  
#endif
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
}
  
```

我们知道内核发送数据包从ip层到驱动层是通过调用dev_queue_xmit，而在dev_queue_xmit中会调用dev_pick_tx来选择一个队列，这里这个patch就是修改这个函数，我们接下来就来看这个函数。

先来分析下这个函数的主要流程，首先，如果设备只有一个队列，那么就选择这唯一的队列。
  
```
  
if (dev->real_num_tx_queues == 1)
		  
queue_index = 0;
  
```
  
然后如果设备设置了回调函数ndo_select_queue，则调用ndo_select_queue来选择队列号，这里要注意，当编写驱动时，如果设置了回调函数ndo_select_queue，此时如果需要xps特性，则最好通过get_xps_queue来取得队列号。
  
```
  
else if (ops->ndo_select_queue) {
		  
queue_index = ops->ndo_select_queue(dev, skb);
		  
queue_index = dev_cap_txqueue(dev, queue_index);
  
```

然后进入主要的处理流程，首先从skb从属的sk中取得缓存的队列索引，如果有缓存，则直接返回这个索引，否则开始计算索引，这里就通过调用xps patch最重要的一个函数get_xps_queue来计算queue_index.

```
  
static struct netdev_queue \*dev_pick_tx(struct net_device \*dev,
					  
struct sk_buff *skb)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
else {
		  
struct sock *sk = skb->sk;
		  
queue_index = sk_tx_queue_get(sk);

if (queue_index < 0 || skb->ooo_okay ||
		      
queue_index >= dev->real_num_tx_queues) {
			  
int old_index = queue_index;
  
//开始计算队列索引
			  
queue_index = get_xps_queue(dev, skb);
			  
if (queue_index < 0)
  
//调用老的计算方法来计算queue index.
				  
queue_index = skb_tx_hash(dev, skb);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
		  
}
	  
}
  
//存储队列索引
	  
skb_set_queue_mapping(skb, queue_index);
  
//返回对应的queue
	  
return netdev_get_tx_queue(dev, queue_index);
  
}
  
```

接下来我们来看get_xps_queue，这个函数是这个patch的核心，它的流程也很简单，就是通过当前的cpu id获得对应的xps_maps,然后如果当前的cpu和队列是1:1对应则返回对应的队列id，否则计算skb的hash值，根据这个hash来得到在xps_maps 中的queue的位置，从而返回queue id.
  
```
  
static inline int get_xps_queue(struct net_device \*dev, struct sk_buff \*skb)
  
{
  
#ifdef CONFIG_XPS
	  
struct xps_dev_maps *dev_maps;
	  
struct xps_map *map;
	  
int queue_index = -1;

rcu_read_lock();
	  
dev_maps = rcu_dereference(dev->xps_maps);
	  
if (dev_maps) {
  
//根据cpu id得到当前cpu对应的队列集合
		  
map = rcu_dereference(
		      
dev_maps->cpu_map[raw_smp_processor_id()]);
		  
if (map) {
  
//如果队列集合长度为1，则说明是1:1对应
			  
if (map->len == 1)
				  
queue_index = map->queues[0];
			  
else {
  
//否则开始计算hash值，接下来和老的计算hash方法一致。
				  
u32 hash;
  
//如果sk_hash存在，则取得sk_hash(这个hash，在我们rps和rfs的时候计算过的,也就是四元组的hash值)
				  
if (skb->sk && skb->sk->sk_hash)
					  
hash = skb->sk->sk_hash;
				  
else
  
//否则开始重新计算
					  
hash = (__force u16) skb->protocol ^
					      
skb->rxhash;
				  
hash = jhash_1word(hash, hashrnd);
  
//根据hash值来选择对应的队列
				  
queue_index = map->queues[
				      
((u64)hash * map->len) >> 32];
			  
}
			  
if (unlikely(queue_index >= dev->real_num_tx_queues))
				  
queue_index = -1;
		  
}
	  
}
	  
rcu_read_unlock();

return queue_index;
  
#else
	  
return -1;
  
#endif
  
}
  
```