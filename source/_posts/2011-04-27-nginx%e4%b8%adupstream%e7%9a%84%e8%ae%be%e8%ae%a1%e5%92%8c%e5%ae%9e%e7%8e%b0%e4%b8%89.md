---
id: 273
title: nginx中upstream的设计和实现(三)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=273'
permalink: >-
  /2011/04/27/nginx%e4%b8%adupstream%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0%e4%b8%89/
categories:
  - nginx
  - server
tags:
  - nginx
translate_title: design-and-implementation-of-upstream-in-nginx-(three)
date: 2011-04-27 10:47:48
---
这次主要来分析当upstream发送过来数据之后，nginx是如何来处理。不过这里我忽略了cache部分，以后我会专门来分析nginx的cache部分。

在前面blog我们能得知upstream端的读回调函数是ngx_http_upstream_process_header，因此这次我们就从ngx_http_upstream_process_header的分析开始。

下面是ngx_http_upstream_process_header执行的流程图.

[<img src="http://farm6.static.flickr.com/5186/5660159919_146459ed32.jpg" width="444" height="456" alt="upstream_process_header" />](http://www.flickr.com/photos/67458145@N00/5660159919/ "upstream_process_header by Minibobo, on Flickr")

<!--more-->

首先，需要分配内存用来接收后端的数据，这里这个buffer就是u->buffer,在fastcgi或者proxy中，我们可以通过fastcgi_buffer_size或者proxy_buffer_size对这个值进行设置，它的初始大小是页的大小。

还有一个要注意的就是u->headers_in，这个头的含义是这样子的，由于upstream是一个通用的组件，因此它不知道后端的协议，而对于client来说，由于http是需要header的，而后端的协议不一定有头，此时就需要我们通过解析后端的协议，然后来设置好发送给client的头，最终发送给client。 

因此此时就需要我们在自己写的upstream模块根据和后端的通信协议来解析数据(就是process_header回调函数)，然后将解析好的数据(header)转换为http的头，此时这些头就是保存在u->headers_in中(详细可以看下fastcgi和process_header或者proxy的).。这里要注意u->headers_in的类型和request里面的类型可是不一样的。不过它和r->headers_in类似，都是将一些常用的头直接放到域里面，而把所有的头都放到u->headers_in.headers这个list中，因此这里就需要初始化u->headers_in.headers这个list.而具体这些是如何做的，我们后面会详细分析.

```
      
if (u->buffer.start == NULL) {
  
//分配一块buffer，可以看到大小为u->conf->buffer_size(fastcgi_buffer_size或者proxy_buffer_size)
          
u->buffer.start = ngx_palloc(r->pool, u->conf->buffer_size);
          
if (u->buffer.start == NULL) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}
  
//然后初始化对应的域
          
u->buffer.pos = u->buffer.start;
          
u->buffer.last = u->buffer.start;
          
u->buffer.end = u->buffer.start + u->conf->buffer_size;
          
u->buffer.temporary = 1;

u->buffer.tag = u->output.tag;
  
//初始化headers.
          
if (ngx_list_init(&u->headers_in.headers, r->pool, 8,
                            
sizeof(ngx_table_elt_t))
              
!= NGX_OK)
          
{
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return;
          
}

#if (NGX_HTTP_CACHE)

if (r->cache) {
              
u->buffer.pos += r->cache->header_start;
              
u->buffer.last = u->buffer.pos;
          
}
  
#endif
      
}
  
```

接下来就是从upstream读取数据,判断返回值，处理错误，如果一切正常，则调用u->process_header,这个也就是我们写upstream模块时，挂载的process_header回调，一般来说，这个回调主要是解析upstream读到数据，得到后端传递过来的头，然后设置u->headers_in中相关的域.

```
      
for ( ;; ) {
  
//接收数据
          
n = c->recv(c, u->buffer.last, u->buffer.end &#8211; u->buffer.last);

if (n == NGX_AGAIN) {
  
#if 0
              
ngx_add_timer(rev, u->read_timeout);
  
#endif

if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
                  
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return;
              
}

return;
          
}
  
//如果为0，则说明upstream已经关闭了连接
          
if (n == 0) {
              
ngx_log_error(NGX_LOG_ERR, c->log, 0,
                            
"upstream prematurely closed connection");
          
}

if (n == NGX_ERROR || n == 0) {
              
ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_ERROR);
              
return;
          
}
  
//更新buffer
          
u->buffer.last += n;

#if 0
          
u->valid_header_in = 0;

u->peer.cached = 0;
  
#endif
  
//然后调用挂载的回调函数
          
rc = u->process_header(r);
  
//如果返回again，则说明后端的数据发送不完全，此时需要再次读取.
          
if (rc == NGX_AGAIN) {

if (u->buffer.pos == u->buffer.end) {
                  
ngx_log_error(NGX_LOG_ERR, c->log, 0,
                                
"upstream sent too big header");

ngx_http_upstream_next(r, u,
                                         
NGX_HTTP_UPSTREAM_FT_INVALID_HEADER);
                  
return;
              
}

continue;
          
}

break;
      
}
  
```

ok，接下来我会以fastcgi中的process_header的代码片段来分析当调用了u->process_header之后，都发生了什么事情。
  
在看这个之前，我们先来看u->headers_in的结构.,可以看到它和r->header_out(ngx_http_headers_out_t)很类似，只不过更简单一些。后面我们可以看到为什么这么类似。

```
  
typedef struct {
  
//保存了所有的将要传递给client的头
      
ngx_list_t headers;
  
//这里用来设置发送给client的 状态码
      
ngx_uint_t status_n;
      
ngx_str_t status_line;
  
//下面这些头是为了更方便的存取
      
ngx_table_elt_t *status;
      
ngx_table_elt_t *date;
      
ngx_table_elt_t *server;
      
ngx_table_elt_t *connection;

ngx_table_elt_t *expires;
      
ngx_table_elt_t *etag;
      
ngx_table_elt_t *x_accel_expires;
      
ngx_table_elt_t *x_accel_redirect;
      
ngx_table_elt_t *x_accel_limit_rate;

ngx_table_elt_t *content_type;
      
ngx_table_elt_t *content_length;

ngx_table_elt_t *last_modified;
      
ngx_table_elt_t *location;
      
ngx_table_elt_t *accept_ranges;
      
ngx_table_elt_t *www_authenticate;

#if (NGX_HTTP_GZIP)
      
ngx_table_elt_t *content_encoding;
  
#endif

off_t content_length_n;

ngx_array_t cache_control;
  
} ngx_http_upstream_headers_in_t;
  
```

然后来看fastcgi的代码片段，下面这段代码就是fastcgi解析upstream中读取的数据，然后将解析到的头设置到u->headers_in中。

```
          
for ( ;; ) {

part_start = u->buffer.pos;
              
part_end = u->buffer.last;
  
//pase header
              
rc = ngx_http_parse_header_line(r, &u->buffer, 1);

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                             
"http fastcgi parser: %d", rc);

if (rc == NGX_AGAIN) {
                  
break;
              
}
  
//到达这里说明一个header已经被解析出来了.
              
if (rc == NGX_OK) {

/\* a header line has been parsed successfully \*/
  
//此时从headers list里面取出一个table。
                  
h = ngx_list_push(&u->headers_in.headers);
                  
if (h == NULL) {
                      
return NGX_ERROR;
                  
}

if (f->split_parts && f->split_parts->nelts) {
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..

} else {
  
//开始构造头，将解析好的指针赋值给h
                      
h->key.len = r->header_name_end &#8211; r->header_name_start;
                      
h->value.len = r->header_end &#8211; r->header_start;

h->key.data = ngx_pnalloc(r->pool,
                                                
h->key.len + 1 + h->value.len + 1
                                                
+ h->key.len);
                      
if (h->key.data == NULL) {
                          
return NGX_ERROR;
                      
}

h->value.data = h->key.data + h->key.len + 1;
                      
h->lowcase_key = h->key.data + h->key.len + 1
                                       
+ h->value.len + 1;
  
//开始复制值
                      
ngx_cpystrn(h->key.data, r->header_name_start,
                                  
h->key.len + 1);
                      
ngx_cpystrn(h->value.data, r->header_start,
                                  
h->value.len + 1);
                  
}

h->hash = r->header_hash;

if (h->key.len == r->lowcase_index) {
                      
ngx_memcpy(h->lowcase_key, r->lowcase_header, h->key.len);

} else {
                      
ngx_strlow(h->lowcase_key, h->key.data, h->key.len);
                  
}

//先从umcf->headers_in_hash中查找，这个hash我们紧接着就会分析。
                  
hh = ngx_hash_find(&umcf->headers_in_hash, h->hash,
                                     
h->lowcase_key, h->key.len);
  
//如果存在则调用hh->handler,接下来会分析这个.
                  
if (hh && hh->handler(r, h, hh->offset) != NGX_OK) {
                      
return NGX_ERROR;
                  
}

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                 
"http fastcgi header: \"%V: %V\"",
                                 
&h->key, &h->value);

if (u->buffer.pos < u->buffer.last) {
                      
continue;
                  
}

/\* the end of the FastCGI record \*/

break;
              
}
  
```

上面的代码有两个疑问，一个是umcf->headers_in_hash，一个是hh->handler,接下来就来分析这两个东西。
  
主要是http有一些头，我们会经常用到，或者有一些头，nginx需要忽略掉(Connection, 因为nginx不支持后端的http 1.1).因此nginx就将这些头特殊处理，常用到的放到固定的域，以便与存取，忽略掉的，则直接赋值为空。

而在nginx中就构造了一个静态数组ngx_http_upstream_headers_in(而umcf->headers_in_hash里面就是这个数组的元素)，这个数组是ngx_http_upstream_header_t类型的.下面我们先来看这个类型的定义。

这个结构有两个需要注意的域，一个是handler，一个是copy_handler,这两个回调的区别是这样子的。
  
```
  
typedef ngx_int_t (\*ngx_http_header_handler_pt)(ngx_http_request_t \*r,
      
ngx_table_elt_t *h, ngx_uint_t offset);
  
```

handler回调用于将传递进来的头(ngx_table_elt_t)赋值(只是改变指针)给ngx_http_upstream_headers_in_t中对应的域。
  
而copy_handler用于将头赋值给r->header_out,也就是发送给client的头。而可以看到r->header_out中的大部分header 域的偏移和u->header_in的偏移是一样的，这样我们赋值给r->header_out就是非常简单了。

```
  
typedef struct {
  
//头的名字
      
ngx_str_t name;
      
ngx_http_header_handler_pt handler;
  
//在ngx_http_upstream_headers_in_t的偏移
      
ngx_uint_t offset;
      
ngx_http_header_handler_pt copy_handler;
      
ngx_uint_t conf;
      
ngx_uint_t redirect; /\* unsigned redirect:1; \*/
  
} ngx_http_upstream_header_t;
  
```

然后我们来看ngx_http_upstream_headers_i这个数组。
  
```
  
ngx_http_upstream_header_t ngx_http_upstream_headers_in[] = {

{ ngx_string("Status"),
                   
ngx_http_upstream_process_header_line,
                   
offsetof(ngx_http_upstream_headers_in_t, status),
                   
ngx_http_upstream_copy_header_line, 0, 0 },

{ ngx_string("Content-Type"),
                   
ngx_http_upstream_process_header_line,
                   
offsetof(ngx_http_upstream_headers_in_t, content_type),
                   
ngx_http_upstream_copy_content_type, 0, 1 },
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
}
  
```

从上面看，这里用的最多的回调就是ngx_http_upstream_process_header_line和ngx_http_upstream_copy_header_line，其它的和他们的功能类似，因此这里我们就详细分析这两个回调，看看里面都是怎么做的。

ngx_http_upstream_process_header_line的做法很简单，取得r->upstream->headers_in的指针，然后通过传递进来的偏移来确定header的位置指针，最后将h赋值给它。

这里可以看到如果已经存在则忽略后面的。
  
```
  
static ngx_int_t
  
ngx_http_upstream_process_header_line(ngx_http_request_t \*r, ngx_table_elt_t \*h,
      
ngx_uint_t offset)
  
{
      
ngx_table_elt_t **ph;
  
//取得header指针
      
ph = (ngx_table_elt_t *\*) ((char \*) &r->upstream->headers_in + offset);

if (*ph == NULL) {
  
//赋值
          
*ph = h;
      
}

return NGX_OK;
  
}
  
```

然后是copy handler，它的实现也很简单，就是从r->headers_out.headers取出来头(这是因为header要在headers和对应的域各保存一个指针)，然后赋值给对应偏移的指针.
  
```
  
static ngx_int_t
  
ngx_http_upstream_copy_header_line(ngx_http_request_t \*r, ngx_table_elt_t \*h,
      
ngx_uint_t offset)
  
{
      
ngx_table_elt_t \*ho, \**ph;
  
//取出header
      
ho = ngx_list_push(&r->headers_out.headers);
      
if (ho == NULL) {
          
return NGX_ERROR;
      
}

\*ho = \*h;
  
//如果offset存在，则赋值.
      
if (offset) {
          
ph = (ngx_table_elt_t *\*) ((char \*) &r->headers_out + offset);
          
*ph = ho;
      
}

return NGX_OK;
  
}
  
```

copy_handler是什么时候调用的呢，我们接着分析下面的代码。让我们回到ngx_http_upstream_process_header，来看它后面的代码。

下面的代码就是调用完u->process_header之后的处理.

```
      
if (rc == NGX_HTTP_UPSTREAM_INVALID_HEADER) {
          
ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_INVALID_HEADER);
          
return;
      
}

if (rc == NGX_ERROR) {
          
ngx_http_upstream_finalize_request(r, u,
                                             
NGX_HTTP_INTERNAL_SERVER_ERROR);
          
return;
      
}

/\* rc == NGX_OK \*/
  
//错误处理.
      
if (u->headers_in.status_n > NGX_HTTP_SPECIAL_RESPONSE) {

if (r->subrequest_in_memory) {
              
u->buffer.last = u->buffer.pos;
          
}

if (ngx_http_upstream_test_next(r, u) == NGX_OK) {
              
return;
          
}

if (ngx_http_upstream_intercept_errors(r, u) == NGX_OK) {
              
return;
          
}
      
}
  
//执行后续工作，主要是设置将要发送给client的header(r->header_out).接下来会详细分析.
      
if (ngx_http_upstream_process_headers(r, u) != NGX_OK) {
          
return;
      
}

if (!r->subrequest_in_memory) {
  
//然后发送response到client端.
          
ngx_http_upstream_send_response(r, u);
          
return;
      
}
  
//下面是subrequst的代码，暂时跳过
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
```

然后来看ngx_http_upstream_process_headers，这个函数会对一个x_accel_redirect的头进行特殊处理，这个头主要是nginx提供了一种机制，让后端的server能够控制访问权限。比如后端限制某个页面不能被用户访问，那么当用户访问这个页面的时候，后端server只需要设置X-Accel-Redirect这个头到一个路径，然后nginx将会输出这个路径的内容给用户.

来看这部分的代码片段.下面这部分就是处理X-Accel-Redirect这个头，首先拷贝header，然后取出X-Accel-Redirect头后面的地址进行内部重定向.

```
      
if (u->headers_in.x_accel_redirect
          
&& !(u->conf->ignore_headers & NGX_HTTP_UPSTREAM_IGN_XA_REDIRECT))
      
{
          
ngx_http_upstream_finalize_request(r, u, NGX_DECLINED);
  
//便利headers
          
part = &u->headers_in.headers.part;
          
h = part->elts;

for (i = 0; /\* void \*/; i++) {

if (i >= part->nelts) {
                  
if (part->next == NULL) {
                      
break;
                  
}

part = part->next;
                  
h = part->elts;
                  
i = 0;
              
}
  
//如果在ngx_http_upstream_headers_in中存在，并且这个头当redirect之后，还是不变的，此时则调用copy_handler.
              
hh = ngx_hash_find(&umcf->headers_in_hash, h[i].hash,
                                 
h[i].lowcase_key, h[i].key.len);

if (hh && hh->redirect) {
                  
if (hh->copy_handler(r, &h[i], hh->conf) != NGX_OK) {
                      
ngx_http_finalize_request(r,
                                                
NGX_HTTP_INTERNAL_SERVER_ERROR);
                      
return NGX_DONE;
                  
}
              
}
          
}
  
//取出uri
          
uri = &u->headers_in.x_accel_redirect->value;
          
ngx_str_null(&args);
          
flags = NGX_HTTP_LOG_UNSAFE;
  
//parse
          
if (ngx_http_parse_unsafe_uri(r, uri, &args, &flags) != NGX_OK) {
              
ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
              
return NGX_DONE;
          
}

if (r->method != NGX_HTTP_HEAD) {
              
r->method = NGX_HTTP_GET;
          
}

r->valid_unparsed_uri = 0;
  
//内部重定向
          
ngx_http_internal_redirect(r, uri, &args);
          
ngx_http_finalize_request(r, NGX_DONE);
          
return NGX_DONE;
      
}
  
```

接下来就是没有X-Accel-Redirect头的情况.这个时候，前部分和上面处理类似，首先从ngx_http_upstream_headers_in查找，如果存在则调用copy_handler,然后再调用ngx_http_upstream_copy_header_line将剩余的头copy到r->header_out.

```
      
part = &u->headers_in.headers.part;
      
h = part->elts;
  
//开始遍历
      
for (i = 0; /\* void \*/; i++) {

if (i >= part->nelts) {
              
if (part->next == NULL) {
                  
break;
              
}

part = part->next;
              
h = part->elts;
              
i = 0;
          
}
  
//查找hash，如果是需要hide的头，则continue
          
if (ngx_hash_find(&u->conf->hide_headers_hash, h[i].hash,
                            
h[i].lowcase_key, h[i].key.len))
          
{
              
continue;
          
}
  
//否则hash查找
          
hh = ngx_hash_find(&umcf->headers_in_hash, h[i].hash,
                             
h[i].lowcase_key, h[i].key.len);

if (hh) {
  
//调用copy_header
              
if (hh->copy_handler(r, &h[i], hh->conf) != NGX_OK) {
                  
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return NGX_DONE;
              
}

continue;
          
}
  
//最后copy剩下的header.
          
if (ngx_http_upstream_copy_header_line(r, &h[i], 0) != NGX_OK) {
              
ngx_http_upstream_finalize_request(r, u,
                                                 
NGX_HTTP_INTERNAL_SERVER_ERROR);
              
return NGX_DONE;
          
}
      
}
  
```

这次的分析就到这里，下一次我将会详细分析 upstream最复杂的一块，也就是发送数据到client的部分(ngx_http_upstream_send_response).