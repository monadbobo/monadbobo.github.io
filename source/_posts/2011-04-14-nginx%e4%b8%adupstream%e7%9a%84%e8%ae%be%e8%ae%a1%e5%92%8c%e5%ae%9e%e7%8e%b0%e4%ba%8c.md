---
id: 259
title: nginx中upstream的设计和实现(二)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=259'
permalink: >-
  /2011/04/14/nginx%e4%b8%adupstream%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0%e4%ba%8c/
categories:
  - nginx
  - server
tags:
  - nginx
  - opensource
  - server
  - web server
translate_title: design-and-implementation-of-upstream-in-nginx-2
date: 2011-04-14 08:50:36
---
这次主要来看upstream的几个相关的hook函数。

首先要知道，对于upstream，同时有两个连接，一个时client和nginx，一个是nginx和upstream，这个时候就会有两个回调，然后上篇blog中，我们能看到在upstream中，会改变read_event_handler和write_event_handler,不过这里有三个条件，分别是
  
1 没有使用cache，
  
2 不忽略client的提前终止
  
3 不是post_action
  
```
  
//条件赋值
      
if (!u->store && !r->post_action && !u->conf->ignore_client_abort) {
  
//然后给读写handler赋值
          
r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
          
r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
      
}
  
```
  
<!--more-->


  
然后我们来看这个两个函数，这两个都会调用ngx_http_upstream_check_broken_connection，因此我们就先来详细分析这个函数。

这个函数主要是用来检测client的连接是否完好。因此它使用了MSG_PEEK这个参数，也就是预读，然后通过recv的返回值来判断是否连接已经断开。

这里的代码分为两部分，第一部分是本身连接在进入这个回调函数之前连接都已经有错误了，这个时候如果是水平触发，则删除事件，然后finalize这个upstream(没有cache&#8217;的情况下），否则就直接finalize这个upstream。

```
      
c = r->connection;
      
u = r->upstream;
  
//如果连接已经出现错误。
      
if (c->error) {
  
//如果是水平触发
          
if ((ngx_event_flags & NGX_USE_LEVEL_EVENT) && ev->active) {

event = ev->write ? NGX_WRITE_EVENT : NGX_READ_EVENT;
  
//删除事件
              
if (ngx_del_event(ev, event, 0) != NGX_OK) {
                  
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return;
              
}
          
}

if (!u->cacheable) {
  
//清理upstream　request
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_CLIENT_CLOSED_REQUEST);
          
}

return;
      
}
  
```

紧接着就是第二部分，这部分的工作就是预读取１个字节，然后来判断是否连接已经被client断掉。

```
  
//读取１个字节
      
n = recv(c->fd, buf, 1, MSG_PEEK);

err = ngx_socket_errno;

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ev->log, err,
                     
"http upstream recv(): %d", n);

if (ev->write && (n >= 0 || err == NGX_EAGAIN)) {
          
return;
      
}
  
//如果水平触发则删除事件
      
if ((ngx_event_flags & NGX_USE_LEVEL_EVENT) && ev->active) {

event = ev->write ? NGX_WRITE_EVENT : NGX_READ_EVENT;

if (ngx_del_event(ev, event, 0) != NGX_OK) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}
      
}
  
//如果还有数据，则直接返回
      
if (n > 0) {
          
return;
      
}

if (n == -1) {
          
if (err == NGX_EAGAIN) {
              
return;
          
}

ev->error = 1;

} else { /\* n == 0 \*/
          
err = 0;
      
}

//到达这里说明有错误产生了
      
ev->eof = 1;
  
//设置错误，可以看到这个值在函数一开始有检测.
      
c->error = 1;
  
//如果没有cache，则finalize　upstream request
      
if (!u->cacheable && u->peer.connection) {
          
ngx_log_error(NGX_LOG_INFO, ev->log, err,
                        
"client closed prematurely connection, "
                        
"so upstream connection is closed too");
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_CLIENT_CLOSED_REQUEST);
          
return;
      
}

ngx_log_error(NGX_LOG_INFO, ev->log, err,
                    
"client closed prematurely connection");
  
//如果有cache，并且后端的upstream还在处理，则此时继续处理upstream，忽略对端的错误.
      
if (u->peer.connection == NULL) {
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_CLIENT_CLOSED_REQUEST);
      
}
  
```

然后我们来看nginx如何连接后端的upstream，在上篇blog的结束的时候，我们看到最终会调用ngx_http_upstream_connect来进入连接upstream的处理，因此我们来详细分析这个函数以及相关的函数。

函数一开始是初始化请求开始事件一些参数
  
```
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//取得upstream的状态
      
u->state = ngx_array_push(r->upstream_states);
      
if (u->state == NULL) {
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}

ngx_memzero(u->state, sizeof(ngx_http_upstream_state_t));

tp = ngx_timeofday();
  
//初始化时间
      
u->state->response_sec = tp->sec;
      
u->state->response_msec = tp->msec;
  
```

然后是调用ngx_event_connect_peer开始连接后端upstream.并且对返回值进行处理，等下会详细分析ngx_event_connect_peer这个函数.

```
  
//连接后端
      
rc = ngx_event_connect_peer(&u->peer);

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                     
"http upstream connect: %i", rc);

if (rc == NGX_ERROR) {
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}
  
//这个是很关键的一个结构peer，后面的blog会详细分析
      
u->state->peer = u->peer.name;

if (rc == NGX_BUSY) {
          
ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no live upstreams");
          
ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_NOLIVE);
          
return;
      
}

if (rc == NGX_DECLINED) {
          
ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
          
return;
      
}
  
```

当返回值为NGX_OK或者NGX_AGAIN的话，就说明连接成功或者暂时异步的连接还没成功，所以需要挂载upstream端的回调函数.这里要注意就是NGX_AGAIN的情况，因为是异步的connect，所以可能会连接不成功。所以如果返回NGX_AGAIN的话，需要挂载写函数.

```
      
/\* rc == NGX_OK || rc == NGX_AGAIN \*/

c = u->peer.connection;

c->data = r;

c->write->handler = ngx_http_upstream_handler;
      
c->read->handler = ngx_http_upstream_handler;
  
//开始挂载回调函数，一个是读，一个是写。
      
u->write_event_handler = ngx_http_upstream_send_request_handler;
      
u->read_event_handler = ngx_http_upstream_process_header;

c->sendfile &= r->connection->sendfile;
      
u->output.sendfile = c->sendfile;

c->pool = r->pool;
      
c->log = r->connection->log;
      
c->read->log = c->log;
      
c->write->log = c->log;

/\* init or reinit the ngx_output_chain() and ngx_chain_writer() contexts \*/

u->writer.out = NULL;
      
u->writer.last = &u->writer.out;
      
u->writer.connection = c;
      
u->writer.limit = 0;
  
```

然后时对request_body的一些处理以及如果request_sent已经设置，也就是这个upstream已经发送过一部分数据了，此时需要重新初始化upstream.

```
      
if (u->request_sent) {
  
//重新初始化upstream
          
if (ngx_http_upstream_reinit(r, u) != NGX_OK) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}
      
}
  
//如果request_body存在的话，保存request_body
      
if (r->request_body
          
&& r->request_body->buf
          
&& r->request_body->temp_file
          
&& r == r->main)
      
{
          
/*
           
* the r->request_body->buf can be reused for one request only,
           
* the subrequests should allocate their own temporay bufs
           
*/

u->output.free = ngx_alloc_chain_link(r->pool);
          
if (u->output.free == NULL) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}
  
//保存到output
          
u->output.free->buf = r->request_body->buf;
          
u->output.free->next = NULL;
          
u->output.allocated = 1;
  
//重置request_body
          
r->request_body->buf->pos = r->request_body->buf->start;
          
r->request_body->buf->last = r->request_body->buf->start;
          
r->request_body->buf->tag = u->output.tag;
      
}
  
```

最后则是先判断rc的返回值，如果是NGX_AGAIN,则说明连接没有返回，则设置定时器，然后返回，否则说明连接成功，这时就需要发送请求到后端。

```
      
if (rc == NGX_AGAIN) {
  
//添加定时器
          
ngx_add_timer(c->write, u->conf->connect_timeout);
          
return;
      
}

ngx_http_upstream_send_request(r, u);
  
```

紧接着我们来看最后的两个函数，分别是上面的ngx_event_connect_peer和ngx_http_upstream_send_request，我们来一个个看。

先来看ngx_event_connect_peer。它主要是用来连接后端，函数比较长，一部分一部分来看。

下面这部分主要是建立socket，然后设置属性，从连接池取出来connection.这里后面的一部分和我们前面client请求上来之后，我们初始化connect类似.

```
  
//取得我们将要发送的upstream对端
      
rc = pc->get(pc, pc->data);
      
if (rc != NGX_OK) {
          
return rc;
      
}
  
//新建socket
      
s = ngx_socket(pc->sockaddr->sa_family, SOCK_STREAM, 0);

ngx_log_debug1(NGX_LOG_DEBUG_EVENT, pc->log, 0, "socket %d", s);

if (s == -1) {
          
ngx_log_error(NGX_LOG_ALERT, pc->log, ngx_socket_errno,
                        
ngx_socket_n " failed");
          
return NGX_ERROR;
      
}

//取得连接
      
c = ngx_get_connection(s, pc->log);

if (c == NULL) {
          
if (ngx_close_socket(s) == -1) {
              
ngx_log_error(NGX_LOG_ALERT, pc->log, ngx_socket_errno,
                            
ngx_close_socket_n "failed");
          
}

return NGX_ERROR;
      
}
  
//设置rcvbuf的大小
      
if (pc->rcvbuf) {
          
if (setsockopt(s, SOL_SOCKET, SO_RCVBUF,
                         
(const void *) &pc->rcvbuf, sizeof(int)) == -1)
          
{
              
ngx_log_error(NGX_LOG_ALERT, pc->log, ngx_socket_errno,
                            
"setsockopt(SO_RCVBUF) failed");
              
goto failed;
          
}
      
}
  
//设置非阻塞
      
if (ngx_nonblocking(s) == -1) {
          
ngx_log_error(NGX_LOG_ALERT, pc->log, ngx_socket_errno,
                        
ngx_nonblocking_n " failed");

goto failed;
      
}

if (pc->local) {
          
if (bind(s, pc->local->sockaddr, pc->local->socklen) == -1) {
              
ngx_log_error(NGX_LOG_CRIT, pc->log, ngx_socket_errno,
                            
"bind(%V) failed", &pc->local->name);

goto failed;
          
}
      
}
  
//开始挂载对应的读写函数.
      
c->recv = ngx_recv;
      
c->send = ngx_send;
      
c->recv_chain = ngx_recv_chain;
      
c->send_chain = ngx_send_chain;

c->sendfile = 1;

c->log_error = pc->log_error;

if (pc->sockaddr->sa_family != AF_INET) {
          
c->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
          
c->tcp_nodelay = NGX_TCP_NODELAY_DISABLED;

#if (NGX_SOLARIS)
          
/\* Solaris&#8217;s sendfilev() supports AF_NCA, AF_INET, and AF_INET6 \*/
          
c->sendfile = 0;
  
#endif
      
}

rev = c->read;
      
wev = c->write;

rev->log = pc->log;
      
wev->log = pc->log;

pc->connection = c;

c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);

#if (NGX_THREADS)

/\* TODO: lock event when call completion handler \*/

rev->lock = pc->lock;
      
wev->lock = pc->lock;
      
rev->own_lock = &c->lock;
      
wev->own_lock = &c->lock;

#endif

if (ngx_add_conn) {
  
//添加读写事件
          
if (ngx_add_conn(c) == NGX_ERROR) {
              
goto failed;
          
}
      
}
  
```

等socket设置完毕，nginx就开始连接后端的upstream，这段代码可以学习一个好的代码是如何处理错误的，

下面这段主要是处理当返回值为-１，并且err不等于NGX_EINPROGRESS的时候，而NGX_EINPROGRESS表示非阻塞的socket，然后connect，然后连接还没有完成，可是提前返回，就回设置这个errno.这个error不算出错，因此需要过滤掉.

```
      
rc = connect(s, pc->sockaddr, pc->socklen);

if (rc == -1) {
          
err = ngx_socket_errno;

//判断错误号
          
if (err != NGX_EINPROGRESS
  
#if (NGX_WIN32)
              
/\* Winsock returns WSAEWOULDBLOCK (NGX_EAGAIN) \*/
              
&& err != NGX_EAGAIN
  
#endif
              
)
          
{
              
if (err == NGX_ECONNREFUSED
  
#if (NGX_LINUX)
                  
/*
                   
* Linux returns EAGAIN instead of ECONNREFUSED
                   
* for unix sockets if listen queue is full
                   
*/
                  
|| err == NGX_EAGAIN
  
#endif
                  
|| err == NGX_ECONNRESET
                  
|| err == NGX_ENETDOWN
                  
|| err == NGX_ENETUNREACH
                  
|| err == NGX_EHOSTDOWN
                  
|| err == NGX_EHOSTUNREACH)
              
{
                  
level = NGX_LOG_ERR;

} else {
                  
level = NGX_LOG_CRIT;
              
}

ngx_log_error(level, c->log, err, "connect() to %V failed",
                            
pc->name);
  
//返回declined
              
return NGX_DECLINED;
          
}
      
}
  
```

然后就是下面的部门就是处理连接成功和错误号为NGX_EINPROGRESS的情况，
  
```
  
//如果当前的事件模型支持add_conn，则事件在开始已经加好了，因此如果rc==-1则直接返回
      
if (ngx_add_conn) {
          
if (rc == -1) {

/\* NGX_EINPROGRESS \*/

return NGX_AGAIN;
          
}

ngx_log_debug0(NGX_LOG_DEBUG_EVENT, pc->log, 0, "connected");

wev->ready = 1;

return NGX_OK;
      
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//添加可读事件
      
if (ngx_add_event(rev, NGX_READ_EVENT, event) != NGX_OK) {
          
goto failed;
      
}

if (rc == -1) {

/\* NGX_EINPROGRESS \*/
  
//如果错误号是　EINPROGRES　添加可写事件
          
if (ngx_add_event(wev, NGX_WRITE_EVENT, event) != NGX_OK) {
              
goto failed;
          
}

return NGX_AGAIN;
      
}

ngx_log_debug0(NGX_LOG_DEBUG_EVENT, pc->log, 0, "connected");

wev->ready = 1;

return NGX_OK;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
```

最后我们来看下ngx_http_upstream_send_request的实现，这个函数是用来发送数据到后端的upstream，然后这里有一个需要注意的地方，那就是在linux下当非阻塞的connect，然后没有连接完成，如果挂载写事件，此时如果写事件上报上来，并不代表连接成功，此时还需要调用getsockopt来判断SO_ERROR，如果没有错误才能保证连接成功。

> SOL_SOCKET
                
> to determine whether connect() completed successfully (SO_ERROR is zero) or unsuccessfully (SO_ERROR is one of the usual error codes listed
                
> here, explaining the reason for the failure).

这里我看了下内核的代码，就是如果连接失败，比如对端不可达，内核会设置sock->sk_soft_err,而在tcp_poll中只会检测sk_err ,　对应的SO_ERROR会检测这两个错误。在内核里面的注释是这样子的

> * @sk_err: last error
    
> * @sk_err_soft: errors that don&#8217;t cause failure but are the cause of a
    
> * persistent failure not just &#8216;timed out&#8217;

这个按照我的理解，内核里面的sk_err 表示４层的错误，而sk_err_soft下层的错误.

在nginx中是在ngx_http_upstream_test_connect中对连接是否断开进行判断的(调用getsockopt).

然后发送数据则是调用ngx_output_chain，不过这里我们知道在ngx_output_chain中会依次调用filter链，可是upstream明显不需要调用filter链，那么nginx是怎么做的呢，是这样子的，在upstream的初始化的时候，已经讲u->output.output_filter改成ngx_chain_writer了:
  
```
      
u->output.output_filter = ngx_chain_writer;
  
```

最后就是一些对错误的处理，我们来看代码

```
  
static void
  
ngx_http_upstream_send_request(ngx_http_request_t \*r, ngx_http_upstream_t \*u)
  
{
      
ngx_int_t rc;
      
ngx_connection_t *c;

c = u->peer.connection;

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                     
"http upstream send request");
  
//如果test connect失败，则说明连接失败，于是跳到下一个upstream，然后返回
      
if (!u->request_sent && ngx_http_upstream_test_connect(c) != NGX_OK) {
          
ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
          
return;
      
}

c->log->action = "sending request to upstream";
  
//发送数据，这里的u->output.output_filter已经被修改过了
      
rc = ngx_output_chain(&u->output, u->request_sent ? NULL : u->request_bufs);

u->request_sent = 1;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//和request的处理类似，如果again，则说明数据没有发送完毕，此时挂载写事件.
      
if (rc == NGX_AGAIN) {
          
ngx_add_timer(c->write, u->conf->send_timeout);

if (ngx_handle_write_event(c->write, u->conf->send_lowat) != NGX_OK) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

return;
      
}

/\* rc == NGX_OK \*/
  
//设置tcp_cork,远离和前面的keepalive部分的处理类似
      
if (c->tcp_nopush == NGX_TCP_NOPUSH_SET) {
          
if (ngx_tcp_push(c->fd) == NGX_ERROR) {
              
ngx_log_error(NGX_LOG_CRIT, c->log, ngx_socket_errno,
                            
ngx_tcp_push_n " failed");
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

c->tcp_nopush = NGX_TCP_NOPUSH_UNSET;
      
}

ngx_add_timer(c->read, u->conf->read_timeout);

#if 1
  
//如果读也可以了，则开始解析头
      
if (c->read->ready) {

/\* post aio operation \*/

/*
           
* TODO comment
           
* although we can post aio operation just in the end
           
* of ngx_http_upstream_connect() CHECK IT !!!
           
* it&#8217;s better to do here because we postpone header buffer allocation
           
*/

ngx_http_upstream_process_header(r, u);
          
return;
      
}
  
#endif
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
}
  
```

在下一篇blog里面，我会详细的分析nginx对后端来的数据如何解析以及如何发送数据到client.