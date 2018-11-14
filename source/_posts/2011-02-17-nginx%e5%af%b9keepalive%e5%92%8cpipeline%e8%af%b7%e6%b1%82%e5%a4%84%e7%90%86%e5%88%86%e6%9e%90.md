---
id: 225
title: nginx对keepalive和pipeline请求处理分析
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=225'
permalink: >-
  /2011/02/17/nginx%e5%af%b9keepalive%e5%92%8cpipeline%e8%af%b7%e6%b1%82%e5%a4%84%e7%90%86%e5%88%86%e6%9e%90/
categories:
  - nginx
  - server
tags:
  - http
  - keepalive
  - nginx
  - pipeline
translate_title: analysis-of-keepalive-and-pipeline-request-processing-by-nginx
date: 2011-02-17 13:26:34
---
这次主要来看nginx中对keepalive和pipeline的处理，这里概念就不用介绍了。直接来看nginx是如何来做的。

首先来看keepalive的处理。我们知道http 1.1中keepalive是默认的，除非客户端显式的指定connect头为close。下面就是nginx判断是否需要keepalive的代码。
  
```
  
void
  
ngx_http_handler(ngx_http_request_t *r)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
          
switch (r->headers_in.connection_type) {
          
case 0:
  
//如果版本大于1.0则默认是keepalive
              
r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
              
break;

case NGX_HTTP_CONNECTION_CLOSE:
  
//如果指定connection头为close则不需要keepalive
              
r->keepalive = 0;
              
break;

case NGX_HTTP_CONNECTION_KEEP_ALIVE:
              
r->keepalive = 1;
              
break;
          
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
}
  
```
  
<!--more-->


  
然后我们知道keepalive也就是当前的http request执行完毕后并不会直接关闭当前的连接，因此nginx的keepalive的相关处理也就是清理request的函数中。

nginx清理requst的函数是ngx_http_finalize_request，这个函数中会调用ngx_http_finalize_connection来释放连接，而keepalive的相关判断就在这个函数中。

```

static void
  
ngx_http_finalize_connection(ngx_http_request_t *r)
  
{
      
ngx_http_core_loc_conf_t *clcf;

clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//可以看到如果设置了keepalive，并且timeout大于0，就进入keepalive的处理。
      
if (!ngx_terminate
           
&& !ngx_exiting
           
&& r->keepalive
           
&& clcf->keepalive_timeout > 0)
      
{
          
ngx_http_set_keepalive(r);
          
return;

} else if (r->lingering_close && clcf->lingering_timeout > 0) {
          
ngx_http_set_lingering_close(r);
          
return;
      
}

ngx_http_close_request(r, 0);
  
}
  
```

通过上面我们能看到keepalive是通过ngx_http_set_keepalive来进行设置的，接下来我们就来详细的看这个函数。

在这个函数里面会顺带处理pipeline的请求，因此我们一并来看，首先nginx是如何区分pipeline请求的呢，它会假设如果从客户端读取的数据多包含了一些数据，也就是解析完当前的request之后，还有一部分数据，这时，就认为是pipeline请求。

还有一个很重要的地方就是http_connection,我们在前面的blog知道，如果需要alloc large header时候，会先从hc->free里面取，如果没有的话，会新建，然后交给hc->busy去管理。而这个buf，就会在这里被重用,因为large buf的话，需要重新alloc第二次，如果这里buf有重用的话，减少一次分配。

```
      
hc = r->http_connection;
      
b = r->header_in;
  
//一般情况下，当解析完header_in之后，pos会设置为last。也就是读取到的数据刚好是一个完整的http请求.当pos小于last，则说明可能是一个pipeline请求。
      
if (b->pos < b->last) {

/\* the pipelined request \*/

if (b != c->buffer) {

/*
               
* If the large header buffers were allocated while the previous
               
* request processing then we do not use c->buffer for
               
* the pipelined request (see ngx_http_init_request()).
               
*
               
* Now we would move the large header buffers to the free list.
               
*/

cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
  
//如果free为空，则新建
              
if (hc->free == NULL) {
  
//可以看到是large_client_headers的个数
                  
hc->free = ngx_palloc(c->pool,
                    
cscf->large_client_header_buffers.num \* sizeof(ngx_buf_t \*));

if (hc->free == NULL) {
                      
ngx_http_close_request(r, 0);
                      
return;
                  
}
              
}
  
//然后清理当前的request的busy
              
for (i = 0; i < hc->nbusy &#8211; 1; i++) {
                  
f = hc->busy[i];
                  
hc->free[hc->nfree++] = f;
                  
f->pos = f->start;
                  
f->last = f->start;
              
}
  
//保存当前的header_in buf,以便与下次给free使用。
              
hc->busy[0] = b;
              
hc->nbusy = 1;
          
}
      
}
  
```

然后接下来这部分就是free request，并设置keepalive 定时器.
  
```
      
r->keepalive = 0;

ngx_http_free_request(r, 0);

c->data = hc;
  
//设置定时器
      
ngx_add_timer(rev, clcf->keepalive_timeout);
  
//然后设置可读事件
      
if (ngx_handle_read_event(rev, 0) != NGX_OK) {
          
ngx_http_close_connection(c);
          
return;
      
}

wev = c->write;
      
wev->handler = ngx_http_empty_handler;
  
```

然后接下来这部分就是对pipeline的处理。

```
      
if (b->pos < b->last) {

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0, "pipelined request");

#if (NGX_STAT_STUB)
          
(void) ngx_atomic_fetch_add(ngx_stat_reading, 1);
  
#endif
  
//设置标记。
          
hc->pipeline = 1;
          
c->log->action = "reading client pipelined request line";
  
//然后扔进post queue，继续进行处理.
          
rev->handler = ngx_http_init_request;
          
ngx_post_event(rev, &ngx_posted_events);
          
return;
      
}
  
```

到达下面，则说明不是pipeline的请求，因此就开始对request， http_connection 进行清理工作。
  
```
      
if (ngx_pfree(c->pool, r) == NGX_OK) {
          
hc->request = NULL;
      
}

b = c->buffer;

if (ngx_pfree(c->pool, b->start) == NGX_OK) {

/*
           
* the special note for ngx_http_keepalive_handler() that
           
* c->buffer&#8217;s memory was freed
           
*/

b->pos = NULL;

} else {
          
b->pos = b->start;
          
b->last = b->start;
      
}

&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;

if (hc->busy) {
          
for (i = 0; i < hc->nbusy; i++) {
              
ngx_pfree(c->pool, hc->busy[i]->start);
              
hc->busy[i] = NULL;
          
}

hc->nbusy = 0;
      
}
  
```

设置keepalive的handler。
  
```
  
//后面会详细分析这个函数
      
rev->handler = ngx_http_keepalive_handler;

if (wev->active && (ngx_event_flags & NGX_USE_LEVEL_EVENT)) {
          
if (ngx_del_event(wev, NGX_WRITE_EVENT, 0) != NGX_OK) {
              
ngx_http_close_connection(c);
              
return;
          
}
      
}
  
```

最后是对tcp push的处理，这里暂时我就不介绍了，接下来我会有专门一篇blog来介绍nginx对tcp push的操作。

然后我们来看ngx_http_keepalive_handler函数，这个函数是处理keepalive连接，当在连接上再次有可读的事件的时候，就会调用这个handler。

这个handler比较简单，就是创建新的buf，然后重新开始一个http request的执行(调用ngx_http_init_request)。
  
```
      
b = c->buffer;
      
size = b->end &#8211; b->start;

if (b->pos == NULL) {

/*
           
* The c->buffer&#8217;s memory was freed by ngx_http_set_keepalive().
           
* However, the c->buffer->start and c->buffer->end were not changed
           
* to keep the buffer size.
           
*/
  
//重新分配buf
          
b->pos = ngx_palloc(c->pool, size);
          
if (b->pos == NULL) {
              
ngx_http_close_connection(c);
              
return;
          
}

b->start = b->pos;
          
b->last = b->pos;
          
b->end = b->pos + size;
      
}
  
```

然后尝试读取数据，如果没有可读数据，则会将句柄再次加入可读事件
  
```
      
n = c->recv(c, b->last, size);
      
c->log_error = NGX_ERROR_INFO;
      
if (n == NGX_AGAIN) {
          
if (ngx_handle_read_event(rev, 0) != NGX_OK) {
              
ngx_http_close_connection(c);
          
}

return;
      
}
  
```

最后如果读取了数据，则进入request的处理。
  
```
      
ngx_http_init_request(rev);
  
```

最后我们再来看ngx_http_init_request函数，这次主要来看当时pipeline请求的时候，nginx是如何来重用request的。
  
这里要注意hc->busy[0],前面我们知道，如果是pipeline请求，我们会保存前面没有解析完毕的request header_in，这是因为我们可能已经读取了pipeline请求的第二个请求的一些头。
  
```
  
//取得request，这里我们知道，在pipeline请求中，我们会保存前一个request.
      
r = hc->request;

if (r) {
  
//如果存在，则我们重用前一个request.
          
ngx_memzero(r, sizeof(ngx_http_request_t));

r->pipeline = hc->pipeline;
  
//如果nbusy存在
          
if (hc->nbusy) {
  
//则保存这个header_in，然后下面直接解析。
              
r->header_in = hc->busy[0];
          
}

} else {
          
r = ngx_pcalloc(c->pool, sizeof(ngx_http_request_t));
          
if (r == NULL) {
              
ngx_http_close_connection(c);
              
return;
          
}

hc->request = r;
      
}
  
//保存请求
      
c->data = r;
  
```

从上面的代码，然后再结合我前一篇blog，我们就知道large header主要是针对pipeline的了，因为在pipeline中，前一个request如果多读了下一个request的一些头的话，这样子下次解析的时候就有可能会超过本来分配的client_header_buffer_size，此时，我们就需要重新分配一个header，也就是large header了，所以这里httpconnection主要就是针对pipeline的情况，而keepalive的连接并不是pipeline的请求的话，为了节省内存，就把前一个request释放掉了.