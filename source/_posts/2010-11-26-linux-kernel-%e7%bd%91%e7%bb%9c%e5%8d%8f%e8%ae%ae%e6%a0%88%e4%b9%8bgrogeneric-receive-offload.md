---
id: 159
title: linux kernel 网络协议栈之GRO(Generic receive offload)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=159'
permalink: >-
  /2010/11/26/linux-kernel-%e7%bd%91%e7%bb%9c%e5%8d%8f%e8%ae%ae%e6%a0%88%e4%b9%8bgrogeneric-receive-offload/
categories:
  - kernel
tags:
  - gro
  - kernel
  - tcp/ip
translate_title: gro-(generic-receive-of-fload)-of-linux-kernel-network-protocol-stack
date: 2010-11-26 13:00:59
---
GRO(Generic receive offload)在内核2.6.29之后合并进去的，作者是一个华裔Herbert Xu ,GRO的简介可以看这里：
  
http://lwn.net/Articles/358910/

先来描述一下GRO的作用，GRO是针对网络接受包的处理的，并且只是针对NAPI类型的驱动，因此如果要支持GRO，不仅要内核支持，而且驱动也必须调用相应的借口，用ethtool -K gro on来设置，如果报错就说明网卡驱动本身就不支持GRO。

GRO类似tso，可是tso只支持发送数据包，这样你tcp层大的段会在网卡被切包，然后再传递给对端，而如果没有gro，则小的段会被一个个送到协议栈，有了gro之后，就会在接收端做一个反向的操作(想对于tso).也就是将tso切好的数据包组合成大包再传递给协议栈。

如果实现了GRO支持的驱动是这样子处理数据的，在NAPI的回调poll方法中读取数据包，然后调用GRO的接口napi_gro_receive或者napi_gro_frags来将数据包feed进协议栈。而具体GRO的工作就是在这两个函数中进行的，他们最终都会调用__napi_gro_receive。下面就是napi_gro_receive，它最终会调用napi_skb_finish以及__napi_gro_receive。
  
<!--more-->


  
```
  
gro_result_t napi_gro_receive(struct napi_struct \*napi, struct sk_buff \*skb)
  
{
	  
skb_gro_reset_offset(skb);

return napi_skb_finish(__napi_gro_receive(napi, skb), skb);
  
}
  
```

然后GRO什么时候会将数据feed进协议栈呢，这里会有两个退出点，一个是在napi_skb_finish里，他会通过判断__napi_gro_receive的返回值，来决定是需要将数据包立即feed进协议栈还是保存起来，还有一个点是当napi的循环执行完毕时，也就是执行napi_complete的时候，先来看napi_skb_finish,napi_complete我们后面会详细介绍。

在NAPI驱动中，直接调用netif_receive_skb会将数据feed 进协议栈，因此这里如果返回值是NORMAL，则直接调用netif_receive_skb来将数据送进协议栈。

```
  
gro_result_t napi_skb_finish(gro_result_t ret, struct sk_buff *skb)
  
{
	  
switch (ret) {
	  
case GRO_NORMAL:
  
//将数据包送进协议栈
		  
if (netif_receive_skb(skb))
			  
ret = GRO_DROP;
		  
break;
  
//表示skb可以被free，因为gro已经将skb合并并保存起来。
	  
case GRO_DROP:
	  
case GRO_MERGED_FREE:
  
//free skb
		  
kfree_skb(skb);
		  
break;
  
//这个表示当前数据已经被gro保存起来，但是并没有进行合并，因此skb还需要保存。
	  
case GRO_HELD:
	  
case GRO_MERGED:
		  
break;
	  
}

return ret;
  
}
  
```

GRO的主要思想就是，组合一些类似的数据包(基于一些数据域，后面会介绍到)为一个大的数据包(一个skb)，然后feed给协议栈，这里主要是利用Scatter-gather IO，也就是skb的struct skb_shared_info域(我前面的blog讲述ip分片的时候有详细介绍这个域)来合并数据包。

在每个NAPI的实例都会包括一个域叫gro_list,保存了我们积攒的数据包(将要被merge的).然后每次进来的skb都会在这个链表里面进行查找，看是否需要merge。而gro_count表示当前的gro_list中的skb的个数。
  
```
  
struct napi_struct {
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//个数
	  
unsigned int gro_count;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//积攒的数据包
	  
struct sk_buff *gro_list;
	  
struct sk_buff *skb;
  
};
  
```

紧接着是gro最核心的一个数据结构napi_gro_cb,它是保存在skb的cb域中，它保存了gro要使用到的一些上下文，这里每个域kernel的注释都比较清楚。到后面我们会看到这些域的具体用途。
  
```
  
struct napi_gro_cb {
	  
/\* Virtual address of skb_shinfo(skb)->frags[0].page + offset. \*/
	  
void *frag0;

/\* Length of frag0. \*/
	  
unsigned int frag0_len;

/\* This indicates where we are processing relative to skb->data. \*/
	  
int data_offset;

/\* This is non-zero if the packet may be of the same flow. \*/
	  
int same_flow;

/\* This is non-zero if the packet cannot be merged with the new skb. \*/
	  
int flush;

/\* Number of segments aggregated. \*/
	  
int count;

/\* Free the skb? \*/
	  
int free;
  
};
  
```

每一层协议都实现了自己的gro回调函数，gro_receive和gro_complete，gro系统会根据协议来调用对应回调函数，其中gro_receive是将输入skb尽量合并到我们gro_list中。而gro_complete则是当我们需要提交gro合并的数据包到协议栈时被调用的。

下面就是ip层和tcp层对应的回调方法：
  
```
  
static const struct net_protocol tcp_protocol = {
	  
.handler = tcp_v4_rcv,
	  
.err_handler = tcp_v4_err,
	  
.gso_send_check = tcp_v4_gso_send_check,
	  
.gso_segment = tcp_tso_segment,
  
//gso回调
	  
.gro_receive = tcp4_gro_receive,
	  
.gro_complete = tcp4_gro_complete,
	  
.no_policy = 1,
	  
.netns_ok = 1,
  
};

static struct packet_type ip_packet_type __read_mostly = {
	  
.type = cpu_to_be16(ETH_P_IP),
	  
.func = ip_rcv,
	  
.gso_send_check = inet_gso_send_check,
	  
.gso_segment = inet_gso_segment,
  
//gso回调
	  
.gro_receive = inet_gro_receive,
	  
.gro_complete = inet_gro_complete,
  
};
  
```

gro的入口函数是napi_gro_receive，它的实现很简单，就是将skb包含的gro上下文reset，然后调用__napi_gro_receive,最终通过napi_skb_finis来判断是否需要讲数据包feed进协议栈。

```
  
gro_result_t napi_gro_receive(struct napi_struct \*napi, struct sk_buff \*skb)
  
{
  
//reset gro对应的域
	  
skb_gro_reset_offset(skb);

return napi_skb_finish(__napi_gro_receive(napi, skb), skb);
  
}
  
```

napi_skb_finish一开始已经介绍过了，这个函数主要是通过判断传递进来的ret(__napi_gro_receive的返回值),来决定是否需要feed数据进协议栈。它的第二个参数是前面处理过的skb。

这里再来看下skb_gro_reset_offset，首先要知道一种情况，那就是skb本身不包含数据(包括头也没有),而所有的数据都保存在skb_shared_info中(支持S/G的网卡有可能会这么做).此时我们如果想要合并的话，就需要将包头这些信息取出来，也就是从skb_shared_info的frags[0]中去的，在 skb_gro_reset_offset中就有做这个事情,而这里就会把头的信息保存到napi_gro_cb 的frags0中。并且此时frags必然不会在high mem,要么是线性区，要么是dma(S/G io)。 来看skb_gro_reset_offset。

```
  
void skb_gro_reset_offset(struct sk_buff *skb)
  
{
	  
NAPI_GRO_CB(skb)->data_offset = 0;
	  
NAPI_GRO_CB(skb)->frag0 = NULL;
	  
NAPI_GRO_CB(skb)->frag0_len = 0;
  
//如果mac_header和skb->tail相等并且地址不在高端内存，则说明包头保存在skb_shinfo中，所以我们需要从frags中取得对应的数据包
	  
if (skb->mac_header == skb->tail &&
	      
!PageHighMem(skb_shinfo(skb)->frags[0].page)) {
  
//可以看到frag0保存的就是对应的skb的frags的第一个元素的地址
		  
NAPI_GRO_CB(skb)->frag0 =
			  
page_address(skb_shinfo(skb)->frags[0].page) +
			  
skb_shinfo(skb)->frags[0].page_offset;
  
//然后保存对应的大小。
		  
NAPI_GRO_CB(skb)->frag0_len = skb_shinfo(skb)->frags[0].size;
	  
}
  
}
  
```

接下来就是__napi_gro_receive，它主要是遍历gro_list,然后给same_flow赋值，这里要注意，same_flow是一个标记，表示某个skb是否有可能会和当前要处理的skb是相同的流,而这里的相同会在每层都进行判断，也就是在设备层，ip层，tcp层都会判断，这里就是设备层的判断了。这里的判断很简单，有2个条件：
  
1 设备是否相同
  
2 mac的头必须相等

如果上面两个条件都满足，则说明两个skb有可能是相同的flow，所以设置same_flow,以便与我们后面合并。
  
```
  
static gro_result_t
  
__napi_gro_receive(struct napi_struct \*napi, struct sk_buff \*skb)
  
{
	  
struct sk_buff *p;

if (netpoll_rx_on(skb))
		  
return GRO_NORMAL;
  
//遍历gro_list,然后判断是否有可能两个skb 相似。
	  
for (p = napi->gro_list; p; p = p->next) {
  
//给same_flow赋值
		  
NAPI_GRO_CB(p)->same_flow =
			  
(p->dev == skb->dev) &&
			  
!compare_ether_header(skb_mac_header(p),
					        
skb_gro_mac_header(skb));
		  
NAPI_GRO_CB(p)->flush = 0;
	  
}
  
//调用dev_gro_receiv
	  
return dev_gro_receive(napi, skb);
  
}
  
```

接下来来看dev_gro_receive，这个函数我们分做两部分来看，第一部分是正常处理部分，第二部份是处理frag0的部分。

来看如何判断是否支持GRO，这里每个设备的features会在驱动初始化的时候被初始化，然后如果支持GRO，则会包括NETIF_F_GRO。 还有要注意的就是，gro不支持切片的ip包，因为ip切片的组包在内核的ip会做一遍，因此这里gro如果合并的话，没有多大意义，而且还增加复杂度。

在dev_gro_receive中会遍历对应的ptype(也就是协议的类链表，以前的blog有详细介绍),然后调用对应的回调函数，一般来说这里会调用文章开始说的ip_packet_type，也就是 inet_gro_receive。

而 inet_gro_receive的返回值表示我们需要立刻feed 进协议栈的数据包，如果为空，则说明不需要feed数据包进协议栈。后面会分析到这里他的详细算法。

而如果当inet_gro_receive正确返回后，如果same_flow没有被设置，则说明gro list中不存在能和当前的skb合并的项，因此此时需要将skb插入到gro list中。这个时候的返回值就是HELD。

```
  
enum gro_result dev_gro_receive(struct napi_struct \*napi, struct sk_buff \*skb)
  
{
	  
struct sk_buff **pp = NULL;
	  
struct packet_type *ptype;
	  
__be16 type = skb->protocol;
	  
struct list_head *head = &ptype_base[ntohs(type) & PTYPE_HASH_MASK];
	  
int same_flow;
	  
int mac_len;
	  
enum gro_result ret;
  
//判断是否支持gro
	  
if (!(skb->dev->features & NETIF_F_GRO))
		  
goto normal;
  
//判断是否为切片的ip包
	  
if (skb_is_gso(skb) || skb_has_frags(skb))
		  
goto normal;

rcu_read_lock();
  
//开始遍历对应的协议表
	  
list_for_each_entry_rcu(ptype, head, list) {
		  
if (ptype->type != type || ptype->dev || !ptype->gro_receive)
			  
continue;

skb_set_network_header(skb, skb_gro_offset(skb));
		  
mac_len = skb->network_header &#8211; skb->mac_header;
		  
skb->mac_len = mac_len;
		  
NAPI_GRO_CB(skb)->same_flow = 0;
		  
NAPI_GRO_CB(skb)->flush = 0;
		  
NAPI_GRO_CB(skb)->free = 0;
  
//调用对应的gro接收函数
		  
pp = ptype->gro_receive(&napi->gro_list, skb);
		  
break;
	  
}
	  
rcu_read_unlock();
  
//如果是没有实现gro的协议则也直接调到normal处理
	  
if (&ptype->list == head)
		  
goto normal;

//到达这里，则说明gro_receive已经调用过了，因此进行后续的处理

//得到same_flow
	  
same_flow = NAPI_GRO_CB(skb)->same_flow;
  
//看是否有需要free对应的skb
	  
ret = NAPI_GRO_CB(skb)->free ? GRO_MERGED_FREE : GRO_MERGED;
  
//如果返回值pp部位空，则说明pp需要马上被feed进协议栈
	  
if (pp) {
		  
struct sk_buff \*nskb = \*pp;

*pp = nskb->next;
		  
nskb->next = NULL;
  
//调用napi_gro_complete 将pp刷进协议栈
		  
napi_gro_complete(nskb);
		  
napi->gro_count&#8211;;
	  
}
  
//如果same_flow有设置，则说明skb已经被正确的合并，因此直接返回。
	  
if (same_flow)
		  
goto ok;
  
//查看是否有设置flush和gro list的个数是否已经超过限制
	  
if (NAPI_GRO_CB(skb)->flush || napi->gro_count >= MAX_GRO_SKBS)
		  
goto normal;

//到达这里说明skb对应gro list来说是一个新的skb，也就是说当前的gro list并不存在可以和skb合并的数据包，因此此时将这个skb插入到gro_list的头。
	  
napi->gro_count++;
	  
NAPI_GRO_CB(skb)->count = 1;
	  
skb_shinfo(skb)->gso_size = skb_gro_len(skb);
  
//将skb插入到gro list的头
	  
skb->next = napi->gro_list;
	  
napi->gro_list = skb;
  
//设置返回值
	  
ret = GRO_HELD;
  
```

然后就是处理frag0的部分，以及不支持gro的处理。

这里要需要对skb_shinfo的结构比较了解，我在以前的blog对这个有很详细的介绍，可以去查阅。

```
  
pull:
  
//是否需要拷贝头
	  
if (skb_headlen(skb) < skb_gro_offset(skb)) {
  
//得到对应的头的大小
		  
int grow = skb_gro_offset(skb) &#8211; skb_headlen(skb);

BUG_ON(skb->end &#8211; skb->tail < grow);
  
//开始拷贝
		  
memcpy(skb_tail_pointer(skb), NAPI_GRO_CB(skb)->frag0, grow);

skb->tail += grow;
		  
skb->data_len -= grow;
  
//更新对应的frags[0]
		  
skb_shinfo(skb)->frags[0].page_offset += grow;
		  
skb_shinfo(skb)->frags[0].size -= grow;
  
//如果size为0了，则说明第一个页全部包含头，因此需要将后面的页全部移动到前面。
		  
if (unlikely(!skb_shinfo(skb)->frags[0].size)) {
			  
put_page(skb_shinfo(skb)->frags[0].page);
  
//开始移动。
			  
memmove(skb_shinfo(skb)->frags,
				  
skb_shinfo(skb)->frags + 1,
				  
&#8211;skb_shinfo(skb)->nr_frags * sizeof(skb_frag_t));
		  
}
	  
}

ok:
	  
return ret;

normal:
	  
ret = GRO_NORMAL;
	  
goto pull;
  
}
  
```

接下来就是inet_gro_receive，这个函数是ip层的gro receive回调函数，函数很简单，首先取得ip头，然后判断是否需要从frag复制数据，如果需要则复制数据

```
  
//得到偏移
		  
off = skb_gro_offset(skb);
  
//得到头的整个长度(mac+ip)
	  
hlen = off + sizeof(*iph);
  
//得到ip头
	  
iph = skb_gro_header_fast(skb, off);
  
//是否需要复制
	  
if (skb_gro_header_hard(skb, hlen)) {
		  
iph = skb_gro_header_slow(skb, hlen, off);
		  
if (unlikely(!iph))
			  
goto out;
	  
}
  
```
  
然后就是一些校验工作，比如协议是否支持gro_reveive,ip头是否合法等等
  
```
	  
proto = iph->protocol & (MAX_INET_PROTOS &#8211; 1);

rcu_read_lock();
	  
ops = rcu_dereference(inet_protos[proto]);
  
//是否支持gro
	  
if (!ops || !ops->gro_receive)
		  
goto out_unlock;
  
//ip头是否合法
	  
if (\*(u8 \*)iph != 0x45)
		  
goto out_unlock;
  
//ip头教研
	  
if (unlikely(ip_fast_csum((u8 *)iph, iph->ihl)))
		  
goto out_unlock;
  
```

然后就是核心的处理部分，它会遍历整个gro_list,然后进行same_flow和是否需要flush的判断。

这里ip层设置same_flow是根据下面的规则的:
  
1 4层的协议必须相同
  
2 tos域必须相同
  
3 源，目的地址必须相同

如果3个条件一个不满足，则会设置same_flow为0。
  
这里还有一个就是判断是否需要flush 对应的skb到协议栈，这里的判断条件是这样子的。
  
1 ip包的ttl不一样
  
2 ip包的id顺序不对
  
3 如果是切片包

如果上面两个条件某一个满足，则说明skb需要被flush出gro。

不过这里要注意只有两个数据包是same flow的情况下，才会进行flush判断。原因很简单，都不是有可能进行merge的包，自然没必要进行flush了。

```
  
//取出id
	  
id = ntohl(\*(__be32 \*)&iph->id);
  
//判断是否需要切片
	  
flush = (u16)((ntohl(\*(__be32 \*)iph) ^ skb_gro_len(skb)) | (id ^ IP_DF));
	  
id >>= 16;
  
//开始遍历gro list
	  
for (p = *head; p; p = p->next) {
		  
struct iphdr *iph2;
  
//如果上一层已经不可能same flow则直接继续下一个
		  
if (!NAPI_GRO_CB(p)->same_flow)
			  
continue;
  
//取出ip头
		  
iph2 = ip_hdr(p);
  
//开始same flow的判断
		  
if ((iph->protocol ^ iph2->protocol) |
		      
(iph->tos ^ iph2->tos) |
		      
((__force u32)iph->saddr ^ (__force u32)iph2->saddr) |
		      
((__force u32)iph->daddr ^ (__force u32)iph2->daddr)) {
			  
NAPI_GRO_CB(p)->same_flow = 0;
			  
continue;
		  
}
  
//开始flush的判断。这里注意如果不是same_flow的话，就没必要进行flush的判断。
		  
/\* All fields must match except length and checksum. \*/
		  
NAPI_GRO_CB(p)->flush |=
			  
(iph->ttl ^ iph2->ttl) |
			  
((u16)(ntohs(iph2->id) + NAPI_GRO_CB(p)->count) ^ id);

NAPI_GRO_CB(p)->flush |= flush;
	  
}

NAPI_GRO_CB(skb)->flush |= flush;
  
//pull ip头进gro，这里更新data_offset
	  
skb_gro_pull(skb, sizeof(*iph));
  
//设置传输层的头的位置
	  
skb_set_transport_header(skb, skb_gro_offset(skb));
  
//调用传输层的reveive方法。
	  
pp = ops->gro_receive(head, skb);

out_unlock:
	  
rcu_read_unlock();

out:
	  
NAPI_GRO_CB(skb)->flush |= flush;

}
  
```

然后就是tcp层的gro方法，它的主要实现函数是tcp_gro_receive，他的流程和inet_gro_receiv类似，就是取得tcp的头，然后对gro list进行遍历，最终会调用合并方法。

首先来看gro list遍历的部分,它对same flow的要求就是source必须相同，如果不同则设置same flow为0.如果相同则跳到found部分，进行合并处理。

```
  
//遍历gro list
	  
for (; (p = *head); head = &p->next) {
  
//如果ip层已经不可能same flow则直接进行下一次匹配
		  
if (!NAPI_GRO_CB(p)->same_flow)
			  
continue;

th2 = tcp_hdr(p);
  
//判断源地址
		  
if (\*(u32 \*)&th->source ^ \*(u32 \*)&th2->source) {
			  
NAPI_GRO_CB(p)->same_flow = 0;
			  
continue;
		  
}

goto found;
	  
}
  
```

接下来就是当找到能够合并的skb的时候的处理，这里首先来看flush的设置,这里会有4个条件：
  
1 拥塞状态被设置(TCP_FLAG_CWR).
  
2 tcp的ack的序列号不匹配 (这是肯定的，因为它只是对tso或者说gso进行反向操作)
  
3 skb的flag和从gro list中查找到要合并skb的flag 如果他们中的不同位 不包括TCP_FLAG_CWR | TCP_FLAG_FIN | TCP_FLAG_PSH，这三个任意一个域。
  
4 tcp的option域不同

如果上面4个条件有一个满足，则会设置flush为1，也就是找到的这个skb(gro list中)必须被刷出到协议栈。

这里谈一下flags域的设置问题首先如果当前的skb设置了cwr，也就是发生了拥塞，那么自然前面被缓存的数据包需要马上被刷到协议栈，以便与tcp的拥塞控制马上进行。

而FIN和PSH这两个flag自然不需要一致，因为这两个和其他的不是互斥的。

```
  
found:
	  
flush = NAPI_GRO_CB(p)->flush;
  
//如果设置拥塞，则肯定需要刷出skb到协议栈
	  
flush |= (__force int)(flags & TCP_FLAG_CWR);
  
//如果相差的域是除了这3个中的，就需要flush出skb
	  
flush |= (__force int)((flags ^ tcp_flag_word(th2)) &
		    
~(TCP_FLAG_CWR | TCP_FLAG_FIN | TCP_FLAG_PSH));
  
//ack的序列号必须一致
	  
flush |= (__force int)(th->ack_seq ^ th2->ack_seq);
  
//tcp的option头必须一致
	  
for (i = sizeof(*th); i < thlen; i += 4)
		  
flush |= \*(u32 \*)((u8 *)th + i) ^
			   
\*(u32 \*)((u8 *)th2 + i);

mss = skb_shinfo(p)->gso_size;

flush |= (len &#8211; 1) >= mss;
	  
flush |= (ntohl(th2->seq) + skb_gro_len(p)) ^ ntohl(th->seq);
  
//如果flush有设置则不会调用 skb_gro_receive，也就是不需要进行合并，否则调用skb_gro_receive进行数据包合并
	  
if (flush || skb_gro_receive(head, skb)) {
		  
mss = 1;
		  
goto out_check_final;
	  
}

p = *head;
	  
th2 = tcp_hdr(p);
  
//更新p的头。到达这里说明合并完毕，因此需要更新合并完的新包的头。
	  
tcp_flag_word(th2) |= flags & (TCP_FLAG_FIN | TCP_FLAG_PSH);
  
```

从上面我们可以看到如果tcp的包被设置了一些特殊的flag比如PSH，SYN这类的就必须马上把数据包刷出到协议栈。

下面就是最终的一些flags判断,比如第一个数据包进来都会到这里来判断。
  
```
  
out_check_final:
	  
flush = len < mss;
  
//根据flag得到flush
	  
flush |= (__force int)(flags & (TCP_FLAG_URG | TCP_FLAG_PSH |
					  
TCP_FLAG_RST | TCP_FLAG_SYN |
					  
TCP_FLAG_FIN));

if (p && (!NAPI_GRO_CB(skb)->same_flow || flush))
		  
pp = head;

out:
	  
NAPI_GRO_CB(skb)->flush |= flush;
  
```

这里要知道每次我们只会刷出gro list中的一个skb节点，这是因为每次进来的数据包我们也只会匹配一个。因此如果遇到需要刷出的数据包，会在dev_gro_receive中先刷出gro list中的，然后再将当前的skb feed进协议栈。

最后就是gro最核心的一个函数skb_gro_receive，它的主要工作就是合并，它有2个参数，第一个是gro list中和当前处理的skb是same flow的skb，第二个就是我们需要合并的skb。

这里要注意就是farg_list,其实gro对待skb_shared_info和ip层切片，组包很类似，就是frags放Scatter-Gather I/O的数据包，frag_list放线性数据。这里gro 也是这样的，如果过来的skb支持Scatter-Gather I/O并且数据是只放在frags中，则会合并frags，如果过来的skb不支持Scatter-Gather I/O(数据头还是保存在skb中)，则合并很简单，就是新建一个skb然后拷贝当前的skb，并将gro list中的skb直接挂载到farg_list。

先来看支持Scatter-Gather I/O的处理部分。
  
```
  
//一些需要用到的变量
	  
struct sk_buff \*p = \*head;
	  
struct sk_buff *nskb;
  
//当前的skb的 share_ino
	  
struct skb_shared_info *skbinfo = skb_shinfo(skb);
  
//当前的gro list中的要合并的skb的share_info
	  
struct skb_shared_info *pinfo = skb_shinfo(p);
	  
unsigned int headroom;
	  
unsigned int len = skb_gro_len(skb);
	  
unsigned int offset = skb_gro_offset(skb);
	  
unsigned int headlen = skb_headlen(skb);
  
//如果有frag_list的话，则直接去非Scatter-Gather I/O部分处理，也就是合并到frag_list.
	  
if (pinfo->frag_list)
		  
goto merge;
	  
else if (headlen <= offset) {
  
//支持Scatter-Gather I/O的处理
		  
skb_frag_t *frag;
		  
skb_frag_t *frag2;
		  
int i = skbinfo->nr_frags;
  
//这里遍历是从后向前。
		  
int nr_frags = pinfo->nr_frags + i;

offset -= headlen;

if (nr_frags > MAX_SKB_FRAGS)
			  
return -E2BIG;
  
//设置pinfo的frags的大小，可以看到就是加上skb的frags的大小
		  
pinfo->nr_frags = nr_frags;
		  
skbinfo->nr_frags = 0;

frag = pinfo->frags + nr_frags;
		  
frag2 = skbinfo->frags + i;
  
//遍历赋值，其实就是地址赋值，这里就是将skb的frag加到pinfo的frgas后面。
		  
do {
			  
\*&#8211;frag = \*&#8211;frag2;
		  
} while (&#8211;i);
  
//更改page_offet的值
		  
frag->page_offset += offset;
  
//修改size大小
		  
frag->size -= offset;
  
//更新skb的相关值
		  
skb->truesize -= skb->data_len;
		  
skb->len -= skb->data_len;
		  
skb->data_len = 0;

NAPI_GRO_CB(skb)->free = 1;
  
//最终完成
		  
goto done;
	  
} else if (skb_gro_len(p) != pinfo->gso_size)
		  
return -E2BIG;
  
```

这里gro list中的要被合并的skb我们叫做skb_s.

接下来就是不支持支持Scatter-Gather I/O(skb的头放在skb中)的处理。这里处理也比较简单，就是复制一个新的nskb，然后它的头和skb_s一样，然后将skb_s挂载到nskb的frag_list上，并且把新建的nskb挂在到gro list中，代替skb_s的位置，而当前的skb

```
	  
headroom = skb_headroom(p);
	  
nskb = alloc_skb(headroom + skb_gro_offset(p), GFP_ATOMIC);
	  
if (unlikely(!nskb))
		  
return -ENOMEM;
  
//复制头
	  
__copy_skb_header(nskb, p);
	  
nskb->mac_len = p->mac_len;

skb_reserve(nskb, headroom);
	  
__skb_put(nskb, skb_gro_offset(p));
  
//设置各层的头
	  
skb_set_mac_header(nskb, skb_mac_header(p) &#8211; p->data);
	  
skb_set_network_header(nskb, skb_network_offset(p));
	  
skb_set_transport_header(nskb, skb_transport_offset(p));

__skb_pull(p, skb_gro_offset(p));
  
//复制数据
	  
memcpy(skb_mac_header(nskb), skb_mac_header(p),
	         
p->data &#8211; skb_mac_header(p));
  
//对应的gro 域的赋值
	  
\*NAPI_GRO_CB(nskb) = \*NAPI_GRO_CB(p);
  
//可以看到frag_list被赋值
	  
skb_shinfo(nskb)->frag_list = p;
	  
skb_shinfo(nskb)->gso_size = pinfo->gso_size;
	  
pinfo->gso_size = 0;
	  
skb_header_release(p);
	  
nskb->prev = p;
  
//更新新的skb的数据段
	  
nskb->data_len += p->len;
	  
nskb->truesize += p->len;
	  
nskb->len += p->len;
  
//将新的skb插入到gro list中
	  
*head = nskb;
	  
nskb->next = p->next;
	  
p->next = NULL;

p = nskb;

merge:
	  
if (offset > headlen) {
		  
skbinfo->frags[0].page_offset += offset &#8211; headlen;
		  
skbinfo->frags[0].size -= offset &#8211; headlen;
		  
offset = headlen;
	  
}

__skb_pull(skb, offset);
  
//将skb插入新的skb的(或者老的skb，当frag list本身存在)fraglist
	  
p->prev->next = skb;
	  
p->prev = skb;
	  
skb_header_release(skb);
  
```