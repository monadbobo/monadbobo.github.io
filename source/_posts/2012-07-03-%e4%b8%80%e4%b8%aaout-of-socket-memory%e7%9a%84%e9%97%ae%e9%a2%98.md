---
id: 413
title: 一个out of socket memory的问题
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=413'
permalink: /2012/07/03/%e4%b8%80%e4%b8%aaout-of-socket-memory%e7%9a%84%e9%97%ae%e9%a2%98/
posturl_add_url:
  - 'yes'
categories:
  - kernel
  - 源码阅读
tags:
  - kenel
  - linux
  - tcp/ip
translate_title: a-problem-of-out-of-socket-memory
date: 2012-07-03 17:08:08
---
今天同事遇到一个问题，就是server(read hat 5, kernel 2.6.18)的dmesg打出了下面两个信息

> TCP: too many of orphaned sockets
  
> Out of socket memory 

一般我们看到这个信息，第一反应肯定是需要调节tcp_mem(/proc/sys/net/ipv4)了，可是根据当时的内存使用情况，使用的内存并没有超过 tcp_mem。然后我先去看了最新的内核代码，3.4.4，其中涉及到socket 内存报警在这里

```
  
bool tcp_check_oom(struct sock *sk, int shift)
  
{
	  
bool too_many_orphans, out_of_socket_memory;

too_many_orphans = tcp_too_many_orphans(sk, shift);
	  
out_of_socket_memory = tcp_out_of_memory(sk);

if (too_many_orphans && net_ratelimit())
		  
pr_info("too many orphaned sockets\n");
	  
if (out_of_socket_memory && net_ratelimit())
		  
pr_info("out of memory &#8212; consider tuning tcp_mem\n");
	  
return too_many_orphans || out_of_socket_memory;
  
}
  
```

上面的代码很简单，就是如果孤儿socket太多，则打印警告，然后如果socket memory超过限制，也打印出警告。
  
<!--more-->


  
于是此时我就怀疑是老版本内核的问题，然后就找到了2.6.18涉及到这部分的代码：

```

static int tcp_out_of_resources(struct sock *sk, int do_reset)
  
{
          
struct tcp_sock *tp = tcp_sk(sk);
          
int orphans = atomic_read(&tcp_orphan_count);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//注意这里
          
if (orphans >= sysctl_tcp_max_orphans ||
              
(sk->sk_wmem_queued > SOCK_MIN_SNDBUF &&
               
atomic_read(&tcp_memory_allocated) > sysctl_tcp_mem[2])) {
                  
if (net_ratelimit())
                          
printk(KERN_INFO "Out of socket memory\n");

/* Catch exceptional cases, when connection requires reset.
                   
\* 1. Last segment was sent recently. \*/
                  
if ((s32)(tcp_time_stamp &#8211; tp->lsndtime) <= TCP_TIMEWAIT_LEN ||
                      
/\* 2. Window is closed. \*/
                      
(!tp->snd_wnd && !tp->packets_out))
                          
do_reset = 1;
                  
if (do_reset)
                          
tcp_send_active_reset(sk, GFP_ATOMIC);
                  
tcp_done(sk);
                  
NET_INC_STATS_BH(LINUX_MIB_TCPABORTONMEMORY);
                  
return 1;
          
}
          
return 0;
  
}
  
```

此时就找到原因了，原来在18的内核中，如果孤儿进程超过限制或者socket的内存超过限制，都会打印出out of socket memory。所以如果是18的内核，这部分是有误的，发生out of socket memory并不代表一定需要调节tcp_mem.