---
id: 358
title: SPDY协议介绍
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=358'
permalink: /2011/10/22/spdy%e5%8d%8f%e8%ae%ae%e4%bb%8b%e7%bb%8d/
categories:
  - SPDY
  - 协议
tags:
  - http
  - spdy
  - 协议
translate_title: introduction-to-spdy-protocol
date: 2011-10-22 18:48:11
---
SPDY的主页： http://www.chromium.org/spdy

我主要看的是SPDY Protocol Drafts 3，这个草稿现在还没完成，google的人将它放在github上面: http://mbelshe.github.com/SPDY-Specification/

按照我的理解，SPDY只是在性能上对http做了很大的优化(比如它的核心思想就是尽量减少连接个数)，而对于http的语义并没有做太大的修改(删除了一些头)，基本上还是兼容http.

SPDY相对于HTTP，主要是有下面4个增强：

1 Multiplexed requests

对于一条SPDY连接，并发的发起多少request没有任何限制,其实也就是可以拥有多条stream(接下来会介绍stream).

2 Prioritized requests

提供具有优先级的请求(同一个SPDY 连接)。这个主要是解决了HTTP中的pipeline请求是严格的FIFO的。比如有多个request，如果先到的一个request处理时间比较长，则后面的request会被阻塞住，而在SPDY中，就会优先处理高优先级的stream中的frame(后面会介绍这个)

3 Compressed headers

在SPDY中，头是可以被压缩的。

4 Server pushed streams

Server可以主动的push数据给client，而不需要客户端的request.

&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;

然后来看SPDY中的一些基本的概念。

首先在SPDY中的第一个概念是session，这里的Session也就是代表一条tcp连接。

在SPDY中第二个概念就是frame，frame也就是SPDY中server和client之间交互的数据。 SPDY Framing Layer是在tcp之上的一层，而当client端和server端建立连接之后，他们之间的交互数据就是frame。 SPDY中分为2种类型的frame，分别是control frame 和data frame。(具体frame的结构，我这里就不介绍了，可以去看协议中的介绍).而每种frame都有对应的flag.

第三个是Stream的概念，一条tcp连接，可以有多条的Stream，每个stream都有一个stream id.在SPDY中有3种control frame来控制Stream的生命周期,分别是:

> SYN_STREAM &#8211; Open a new stream
  
> SYN_REPLY &#8211; Remote acknowledgement of a new, open stream
  
> RST_STREAM &#8211; Close a stream

可以看到，和tcp建立连接很像。这里要注意Stream也是有优先级的。如果一端发送一个设置了FLAG_FIN标记的frame，则这个stream将会成为半关闭的.(也就是不会再发送数据，而只能够接收收据，这个和tcp的半关闭很类似).如果一个发送端发送了带有FLAG_FIN标记的flag，如果它再次发送数据，则将会收到一个RST_STREAM的frame.

要注意，只有synstreamframe(建立一个stream的控制frame)才拥有priority，也就是说在SPDY中优先级只到stream这个级别，只有某个stream中的request会被优先处理，而同一个stream中的frame则类似于http的行为。

从stream我们可以看到相比较于http，SPDY可以很打程度上减少建立的连接的数目，因为每个stream其实就类似于一个虚拟的连接。

假设现在需要做一个http->spdy的代理，当一个http request过来的时候,如果这个request没有body，则将会发送一个设置了FLAG_FIN标记的frame,而对应的http头都将会放入到这个frame中。可以看下go 中对于SYN frame的结构定义：
  
```
  
type SynStreamFrame struct {
	  
CFHeader ControlFrameHeader
	  
StreamId uint32
	  
AssociatedToStreamId uint32
	  
// Note, only 2 highest bits currently used
	  
// Rest of Priority is unused.
	  
Priority uint16
	  
Headers http.Header
  
}
  
```
  
这里可以看到http的头就是紧挨着SPDY的frame，不过这里注意由于http是文本协议，所以最终SPDY还是会将Headers转成二进制的。

reponse类似，下面就是go中的synReplay frame,可以看到和syn类似:
  
```
  
// SynReplyFrame is the unpacked, in-memory representation of a SYN_REPLY frame.
  
type SynReplyFrame struct {
	  
CFHeader ControlFrameHeader
	  
StreamId uint32
	  
Headers http.Header
  
}
  
```

最后来看下pushed stream，在SPDY中，server能够发送多个replay给一个request，就是说有时候server能够知道需要发送多个资源给client，此时就需要server push资源给client，而如果没有这个特性，则需要client不断的发多个request来请求多个资源。这样子就极大的RT延迟。

在SPDY中，如果server要push数据给client，则它必须选择一个已经存在的stream id，并且server端必须得设置一个比较高的优先级，以便于客户端能够迅速的发现pushed stream，然后响应它。server端能push的内容必须是客户端能够通过一个get请求所得到的资源(也就是push过来的syn frame必须包含一些头信息)。