---
id: 228
title: nginx对TCP_CORK/TCP_NOPUSH的使用
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=228'
permalink: /2011/02/23/nginx%e5%af%b9tcp_corktcp_nopush%e7%9a%84%e4%bd%bf%e7%94%a8/
categories:
  - nginx
  - server
tags:
  - nginx
  - tcp
  - tcp_cork
  - 服务器设计
translate_title: nginx's-use-of-tcp_cork/tcp_nopush
date: 2011-02-23 11:18:15
---
在nginx中使用了send_file 并且配合TCP_CORK/TCP_NOPUSH进行操作，我们一般的操作是这样子的，首先调用tcp_cork，阻塞下层的数据发送，然后调用send_file发送数据，最后关闭TCP_CORK/TCP_NOPUSH.而在nginx中不是这样处理的，前面两步都是一样的，最后一步，它巧妙的利用的http的特性，那就是基本都是短连接，也就是处理完当前的request之后，就会关闭当前的连接句柄，而在linux中，如果不是下面两种情况之一，那么关闭tcp句柄，就会发送完发送buf中的数据，才进行tcp的断开操作(具体可以看我以前写的那篇 “linux内核中tcp连接的断开处理”的 blog) :

1 接收buf中还有未读数据。
  
2 so_linger设置并且超时时间为0.

而如果调用shutdown来关闭写端的话，就是直接发送完写buf中的数据，然后发送fin。

ok，通过上面我们知道每次处理完请求，都会关闭连接(keepalive 会单独处理),而关闭连接就会帮我们将cork拔掉，所以这里就可以节省一个系统调用，从这里能看到nginx对细节的处理到了一个什么程度。

接下来还有一个单独要处理的就是keepalive的连接，由于keepalive是不会关闭当前的连接的，因此这里就必须显式的关闭tcp_cork。
  
<!--more-->

然后我们来看代码，首先来看TCP_CORK/TCP_NOPUSH相关的配置，我们知道在nginx中 TCP_CORK/TCP_NOPUSH是默认关闭的，除非我们显示的打开(tcp_nopush on).而在nginx中对于TCP_CORK/TCP_NOPUSH选项的打开关闭是有两个位置，一个是ngx_http_core_loc_conf_t的tcp_nopush(这个对应配置文件里面的tcp_nopush选项),一个是r->connection->tcp_nopush，它默认是NGX_TCP_NOPUSH_UNSET，可是如果没有设置tcp_nopush on，则这个值就会被设为NGX_TCP_NOPUSH_DISABLED，也就是关闭tcp_cork.而这里主要使用的就是r->connection->tcp_nopush。这里多出来的NGX_TCP_NOPUSH_DISABLED主要就是针对keepalive的情况，后面我们会详细分析.

下面就是tcp_nopush的可选值.
  
```
  
typedef enum {
       
NGX_TCP_NOPUSH_UNSET = 0,
       
NGX_TCP_NOPUSH_SET,
       
NGX_TCP_NOPUSH_DISABLED
  
} ngx_connection_tcp_nopush_e;
  
```

来看core loc conf的值：
  
```
      
ngx_conf_merge_value(conf->tcp_nopush, prev->tcp_nopush, 0);
  
```
  
可以看到他的默认值是0T，也就是tcp_nopush没有打开的情况。

然后是r->connection->tcp_nopush 的设置，它的值依赖于clcf->tcp_nopush，如果clcf->tcp_nopush为(默认值),则关闭tcp_nopush,否则就是默认值打开。而connection中的tcp_nopush默认是NGX_TCP_NOPUSH_UNSET，也就是0。
  
```
      
if (!clcf->tcp_nopush) {
          
/\* disable TCP_NOPUSH/TCP_CORK use \*/
  
//设置关闭
          
r->connection->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
      
}
  
```

ok，接下来我们就直接来看nginx中如何使用TCP_CORK/TCP_NOPUSH,主要的代码是在ngx_linux_sendfile_chain（我们通过分析tcp_cork来分析，bsd的tcp_nopush和linux的处理类似)中，这个函数我前面的blog有详细分析过，这次主要是分析tcp_cork部分，我们主要来看,第一次调用sendfile的时候，会先设置tcp_cork.而设置之后就会设置c->tcp_nopush为NGX_TCP_NOPUSH_SET，也就是接下来的操作都不需要再次设置了。而最终当buf处理完毕，就直接返回。

这里有一个要注意的就是tcp_nodelay和tcp_nopush是互斥的。不过如果你同时设置了两个值的话，将会在第一个buf发送的时候，强制push数据，而第二个buf时，将会调用tcp_cork来打开nagle算法，也就是后面的都会应用tcp_nopush.

```
  
//如果tcp_nopush为0(也就是配置文件中tcp_nopush on)，则进入tcp_nopush的处理。
          
if (c->tcp_nopush == NGX_TCP_NOPUSH_UNSET
              
&& header.nelts != 0
              
&& cl
              
&& cl->buf->in_file)
          
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//如果nodelay有设置，则进入相关处理，也就是关闭nagle算法。
                          
if (c->tcp_nodelay == NGX_TCP_NODELAY_SET) {
                                                  
tcp_nodelay = 0;

if (setsockopt(c->fd, IPPROTO_TCP, TCP_NODELAY,
                                 
(const void *) &tcp_nodelay, sizeof(int)) == -1)
                                 
{
                                     
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
                                   
}
                                   
else {
  
//如果设置成功，则设置nodelay为NGX_TCP_NODELAY_UNSET.这样下次就不会进入nodelay的处理。
                                       
c->tcp_nodelay = NGX_TCP_NODELAY_UNSET;

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                                     
"no tcp_nodelay");
                                   
}
                           
}
  
//如果tcp_nodelay没有设置，则进入tcp_nopush的处理。
           
if (c->tcp_nodelay == NGX_TCP_NODELAY_UNSET) {
  
//设置tcp_nopush(tcp_cork)
                  
if (ngx_tcp_nopush(c->fd) == NGX_ERROR) {
                      
err = ngx_errno;

/*
                       
* there is a tiny chance to be interrupted, however,
                       
* we continue a processing without the TCP_CORK
                       
*/

if (err != NGX_EINTR) {
                          
wev->error = 1;
                          
ngx_connection_error(c, err,
                                               
ngx_tcp_nopush_n " failed");
                          
return NGX_CHAIN_ERROR;
                      
}

} else {
  
//如果设置成功，则将c->tcp_nopush设置为set，这样当再次需要调用sendfile的时候，就跳过设置tcp_nopush的部分
                      
c->tcp_nopush = NGX_TCP_NOPUSH_SET;

ngx_log_debug0(NGX_LOG_DEBUG_EVENT, c->log, 0,
                                     
"tcp_nopush");
                  
}
              
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
}
  
```

通过上面我们看到nginx最终会调用sendfile发送数据，然后当前的请求结束，最终会调用tcp_close或者tcp_shutdown(设置linger_timeout)来关闭连接，而当关闭连接的同时，内核会将写buf中缓存的数据发送出去，这样，就不需要我们关闭tcp_cork,因为内核会帮我们做这个，于是减少了一次系统调用。

到了这里，会有一个问题，那就是keepalive的情况，这时就会有问题了，因为keepalive是不会关闭连接的，这样，就需要我们显示的调用tcp_push来将数据push出去。接下来我就来分析上篇介绍keepalive的文章中没有涉及到的部分。

接下来的代码就在ngx_http_set_keepalive中。

```
  
//如果tcp_nopush为NGX_TCP_NOPUSH_SET，则说明我们需要关闭tcp_cork,也就是将数据push出去.
      
if (c->tcp_nopush == NGX_TCP_NOPUSH_SET) {
  
//push数据
          
if (ngx_tcp_push(c->fd) == -1) {
              
ngx_connection_error(c, ngx_socket_errno, ngx_tcp_push_n " failed");
              
ngx_http_close_connection(c);
              
return;
          
}
  
//reset tcp_nopush
          
c->tcp_nopush = NGX_TCP_NOPUSH_UNSET;
          
tcp_nodelay = ngx_tcp_nodelay_and_tcp_nopush ? 1 : 0;

} else {
          
tcp_nodelay = 1;
      
}

//如果tcp_nodelay存在，则说明我们并没有使用tcp_nopush，此时如果clcf->tcp_nodelay被设置，则此时需要重新设置tcp_nodelay,也就是关闭nagle算法。
      
if (tcp_nodelay
          
&& clcf->tcp_nodelay
          
&& c->tcp_nodelay == NGX_TCP_NODELAY_UNSET)
      
{
          
ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "tcp_nodelay");

if (setsockopt(c->fd, IPPROTO_TCP, TCP_NODELAY,
                         
(const void *) &tcp_nodelay, sizeof(int))
              
== -1)
          
{
  
#if (NGX_SOLARIS)
              
/\* Solaris returns EINVAL if a socket has been shut down \*/
              
c->log_error = NGX_ERROR_IGNORE_EINVAL;
  
#endif

ngx_connection_error(c, ngx_socket_errno,
                                   
"setsockopt(TCP_NODELAY) failed");

c->log_error = NGX_ERROR_INFO;
              
ngx_http_close_connection(c);
              
return;
          
}
  
//reset tcp_nodelay
          
c->tcp_nodelay = NGX_TCP_NODELAY_SET;
      
}
  
```