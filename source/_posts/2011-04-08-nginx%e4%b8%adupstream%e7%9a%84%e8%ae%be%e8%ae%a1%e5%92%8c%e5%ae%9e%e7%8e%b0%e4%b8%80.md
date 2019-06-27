---
id: 251
title: nginx中upstream的设计和实现(一)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=251'
permalink: >-
  /2011/04/08/nginx%e4%b8%adupstream%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0%e4%b8%80/
posturl_add_url:
  - 'yes'
categories:
  - nginx
  - server
  - 源码阅读
tags:
  - nginx
translate_title: design-and-implementation-of-upstream-in-nginx-1
date: 2011-04-08 19:27:37
---
在nginx的模块中，分为3种类型，分别是handler，filter和upstream，其中upstream可以看做一种特殊的handler，它主要用来实现和后端另外的服务器(php/jboss等)进行通信，由于在nginx中全部都是使用非阻塞，并且是一个流式的处理，所以upstream的实现很复杂，接下来我会通过几篇blog来详细的分析下nginx中upstream的设计和实现。

upstream顾名思义，真正产生内容的地方在&#8221;上游&#8221;而不是nginx，也就是说nginx是位于client和后端的upstream之间的桥梁，在这种情况下，一个upstream需要做的事情主要有2个，第一个是当client发送http请求过来之后，需要创建一个到后端upstream的请求。第二个是当后端发送数据过来之后，需要将后端upstream的数据再次发送给client.接下来会看到，我们编写一个upstream模块，最主要也是这两个hook方法。

首先来看如果我们要写一个upstream模块的话，大体的步骤是什么，我们以memcached模块为例子，我们会看到如果我们自己编写upstream模块的话，只需要编写upstream需要的一些hook函数，然后挂载到upstream上就可以了。
  
<!--more-->

首先来看它的初始化部分。这个函数是命令memcached_pass的handle，它主要做了两件事情，第一件是保存目的upstream(memcached_pass 的参数).第二个是设置core module的handler(这个handler会在update_location中设置给content_handle)，也就是说一个upstream其实也就是一个处于content phase的handler。

```
  
static char *
  
ngx_http_memcached_pass(ngx_conf_t \*cf, ngx_command_t \*cmd, void *conf)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
      
value = cf->args->elts;

ngx_memzero(&u, sizeof(ngx_url_t));
      
u.url = value[1];
      
u.no_resolve = 1;
  
//根据url，取得对应的upstream
      
mlcf->upstream.upstream = ngx_http_upstream_add(cf, &u, 0);
      
if (mlcf->upstream.upstream == NULL) {
          
return NGX_CONF_ERROR;
      
}

clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
  
//设置handler
      
clcf->handler = ngx_http_memcached_handler;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
```

然后来看ngx_http_memcached_handler，它主要是初始化upstream的相关回调，然后调用ngx_http_upstream_init设置对应的读写回调等等其他的操作。这里我们要主要看它设置的几个回调函数。
  
```
  
static ngx_int_t
  
ngx_http_memcached_handler(ngx_http_request_t *r)
  
{
      
ngx_int_t rc;
      
ngx_http_upstream_t *u;
      
ngx_http_memcached_ctx_t *ctx;
      
ngx_http_memcached_loc_conf_t *mlcf;

if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
          
return NGX_HTTP_NOT_ALLOWED;
      
}
  
//首先discard request body
      
rc = ngx_http_discard_request_body(r);

if (rc != NGX_OK) {
          
return rc;
      
}
  
//设置content type。
      
if (ngx_http_set_content_type(r) != NGX_OK) {
          
return NGX_HTTP_INTERNAL_SERVER_ERROR;
      
}
  
//创建一个upstream
      
if (ngx_http_upstream_create(r) != NGX_OK) {
          
return NGX_HTTP_INTERNAL_SERVER_ERROR;
      
}

u = r->upstream;
  
//设置schema
      
ngx_str_set(&u->schema, "memcached://");
      
u->output.tag = (ngx_buf_tag_t) &ngx_http_memcached_module;

mlcf = ngx_http_get_module_loc_conf(r, ngx_http_memcached_module);
  
//设置config，可以看到它就是在memcached_pass中add的upstream
      
u->conf = &mlcf->upstream;
  
//开始设置回调，接下来会解释这几个回调的含义
      
u->create_request = ngx_http_memcached_create_request;
      
u->reinit_request = ngx_http_memcached_reinit_request;
      
u->process_header = ngx_http_memcached_process_header;
      
u->abort_request = ngx_http_memcached_abort_request;
      
u->finalize_request = ngx_http_memcached_finalize_request;
  
//创建上下文
      
ctx = ngx_palloc(r->pool, sizeof(ngx_http_memcached_ctx_t));
      
if (ctx == NULL) {
          
return NGX_HTTP_INTERNAL_SERVER_ERROR;
      
}

ctx->rest = NGX_HTTP_MEMCACHED_END;
      
ctx->request = r;

ngx_http_set_ctx(r, ctx, ngx_http_memcached_module);
  
//设置另外的回调，这几个回调主要是针对非buffering的情况
      
u->input_filter_init = ngx_http_memcached_filter_init;
      
u->input_filter = ngx_http_memcached_filter;
      
u->input_filter_ctx = ctx;

r->main->count++;
  
//进入upstream的处理，接下来就会详细分析这个函数
      
ngx_http_upstream_init(r);

return NGX_DONE;
  
}
  
```

ok，接下来来看上面设置到的几个回调函数。
  
```
  
//这个回调是创建到后端upstream的request时候被调用.
      
ngx_int_t (\*create_request)(ngx_http_request_t \*r);
  
//这个是重新初始化到后端upstream的请求，主要是重新初始化context
      
ngx_int_t (\*reinit_request)(ngx_http_request_t \*r);
  
//这个是当后端upstream发送数据到nginx，然后nginx解析这个数据的时候被调用
      
ngx_int_t (\*process_header)(ngx_http_request_t \*r);
  
//这个回调暂时还没有nginx使用
      
void (\*abort_request)(ngx_http_request_t \*r);
  
//request(和后端upstream)结束，需要释放资源
      
void (\*finalize_request)(ngx_http_request_t \*r,
                                           
ngx_int_t rc);
  
//用于upstream中的重定向
      
ngx_int_t (\*rewrite_redirect)(ngx_http_request_t \*r,
                                           
ngx_table_elt_t *h, size_t prefix);
  
```

其实除了上面的几个回调，还有几个很重要的回调，那几个我们在后面会详细分析。

接下来我们就跳出memcached模块，进入upstream 模块的处理分析了，首先来看下面upstream初始化基本的流程图，下面这张图只到挂载完upstream的读写回调函数。

[<img src="http://farm6.static.flickr.com/5185/5584325156_3857eafb3f.jpg" width="259" height="500" alt="upstream1" />](http://www.flickr.com/photos/67458145@N00/5584325156/ "upstream1 by Minibobo, on Flickr")

从上面的memcached的最后一部分，我们看到最终调用 ngx_http_upstream_init(r)进入upstream的处理，我们就从这个函数开始来分析upstream的初始化实现。

这个函数首先删除设置的定时器，然后如果是边缘触发的话，则挂载write的事件，这是因为走upstream，如果接下来我们connect不成功(NGX_EINPROGRESS),则不会进入ngx_http_finalize_request以挂载写hook(前面request的blog有介绍这部分),所以这里我们需要先挂载写事件.

```
  
void
  
ngx_http_upstream_init(ngx_http_request_t *r)
  
{
      
ngx_connection_t *c;

c = r->connection;

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0,
                     
"http init upstream, client timer: %d", c->read->timer_set);
  
//删除定时器
      
if (c->read->timer_set) {
          
ngx_del_timer(c->read);
      
}
  
//挂载写事件
      
if (ngx_event_flags & NGX_USE_CLEAR_EVENT) {

if (!c->write->active) {
              
if (ngx_add_event(c->write, NGX_WRITE_EVENT, NGX_CLEAR_EVENT)
                  
== NGX_ERROR)
              
{
                  
ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return;
              
}
          
}
      
}
  
//进入upstream的初始化
      
ngx_http_upstream_init_request(r);
  
}
  
```

接下来就来详细分析ngx_http_upstream_init_reques，这个函数比较长，我们一段段来看。

下面这段主要是创建将要发送给后端upstream的请求，这里可以看到调用的是create_request这个回调函数，这个函数是我们编写upstream函数时，挂载的hook之一。

```
      
u->store = (u->conf->store || u->conf->store_lengths);

if (!u->store && !r->post_action && !u->conf->ignore_client_abort) {
          
r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
          
r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
      
}

if (r->request_body) {
          
u->request_bufs = r->request_body->bufs;
      
}
  
//调用create_request来创建请求
      
if (u->create_request(r) != NGX_OK) {
          
ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}
  
```

接下来这段是初始化upstream的一些属性.
  
```
      
u->peer.local = u->conf->local;

clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

u->output.alignment = clcf->directio_alignment;
      
u->output.pool = r->pool;
      
u->output.bufs.num = 1;
      
u->output.bufs.size = clcf->client_body_buffer_size;
      
u->output.output_filter = ngx_chain_writer;
      
u->output.filter_ctx = &u->writer;
  
```

然后接下来就是初始化upstream_states数组，这个数组主要是保存了当前的upstream的一些状态。

```
  
//初始化upstream states
      
if (r->upstream_states == NULL) {

r->upstream_states = ngx_array_create(r->pool, 1,
                                              
sizeof(ngx_http_upstream_state_t));
          
if (r->upstream_states == NULL) {
              
ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

} else {
          
u->state = ngx_array_push(r->upstream_states);
          
if (u->state == NULL) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

ngx_memzero(u->state, sizeof(ngx_http_upstream_state_t));
      
}
  
//开始挂载清理回调函数
      
cln = ngx_http_cleanup_add(r, 0);
      
if (cln == NULL) {
          
ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}

cln->handler = ngx_http_upstream_cleanup;
      
cln->data = r;
      
u->cleanup = &cln->handler;
  
```

然后就是这个函数最核心的处理部分，那就是根据upstream的类型来进行不同的操作，这里的upstream就是我们通过XXX_pass传递进来的值，这里的upstream有可能下面几种情况。

1 XXX_pass中不包含变量。

2 XXX_pass传递的值包含了一个变量($开始).这种情况也就是说upstream的url是动态变化的，因此需要每次都解析一遍.

而第二种情况又分为2种，一种是在进入upstream之前，也就是 upstream模块的handler之中已经被resolve的地址(请看ngx_http_XXX_eval函数)，一种是没有被resolve，此时就需要upstream模块来进行resolve。

接下来的代码就是处理这部分的东西。

```
      
if (u->resolved == NULL) {
  
//这里的话，在XXX_pass命令被解析的时候，upstream已经被解析完毕了.
          
uscf = u->conf->upstream;

} else {
  
//如果地址已经被resolve过了，此时创建round robin peer
          
if (u->resolved->sockaddr) {

if (ngx_http_upstream_create_round_robin_peer(r, u->resolved)
                  
!= NGX_OK)
              
{
                  
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return;
              
}
  
//然后开始连接后端的upstream
              
ngx_http_upstream_connect(r, u);

return;
          
}
  
//否则需要resolve host到ip地址
          
host = &u->resolved->host;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..

temp.name = *host;

ctx = ngx_resolve_start(clcf->resolver, &temp);
          
if (ctx == NULL) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..

ctx->name = *host;
          
ctx->type = NGX_RESOLVE_A;
  
//设置handler，最终在下面的ngx_resolve_name中会调用这个handler。
          
ctx->handler = ngx_http_upstream_resolve_handler;
          
ctx->data = r;
          
ctx->timeout = clcf->resolver_timeout;

u->resolved->ctx = ctx;
  
//resolve hostname，然后会调用上面的handler
          
if (ngx_resolve_name(ctx) != NGX_OK) {
              
u->resolved->ctx = NULL;
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

return;
      
}
  
```

然后我们来看ngx_http_upstream_resolve_handler，它做得事情和上面的u->resolved->sockaddr存在的分支类似，也是创建peer然后连接对端.
  
```
  
static void
  
ngx_http_upstream_resolve_handler(ngx_resolver_ctx_t *ctx)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
      
if (ngx_http_upstream_create_round_robin_peer(r, ur) != NGX_OK) {
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}

ngx_resolve_name_done(ctx);
      
ur->ctx = NULL;

ngx_http_upstream_connect(r, u);
  
}
  
```

这部分就先分析到这里，接下来的一篇我会详细分析nginx connect后端upstream以及读写handler的初始化。