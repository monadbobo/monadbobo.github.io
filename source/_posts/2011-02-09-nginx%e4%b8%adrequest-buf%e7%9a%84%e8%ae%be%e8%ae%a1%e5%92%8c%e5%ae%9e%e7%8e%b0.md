---
id: 220
title: nginx中request buf的设计和实现
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=220'
permalink: >-
  /2011/02/09/nginx%e4%b8%adrequest-buf%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0/
categories:
  - nginx
  - server
tags:
  - nginx
  - server
translate_title: design-and-implementation-of-request-buf-in-nginx
date: 2011-02-09 11:56:22
---
在nginx中request的buffer size我们能够通过两个命令进行设置，分别是large_client_header_buffers和client_header_buffer_size。这两个是有所不同的。
  
在解析request中，如果已经读取的request line 或者 request header大于lient_header_buffer_size的话，此时就会重新分配一块大的内存，然后将前面未完成解析的部分拷贝到当前的大的buf中，然后再进入解析处理，这部分的buf也就是large_client_header_buffers,也叫做large hader的处理。接下来我会通过代码来详细的分析这个过程。
  
<!--more-->

首先来看client_header_buffer_size，他的默认值是1024,用这个值来初始化header_in(在ngx_http_init_request中)。

```
      
if (c->buffer == NULL) {
  
//创建buffer
          
c->buffer = ngx_create_temp_buf(c->pool,
                                          
cscf->client_header_buffer_size);
          
if (c->buffer == NULL) {
              
ngx_http_close_connection(c);
              
return;
          
}
      
}

if (r->header_in == NULL) {
  
//可以看到header_in就是上面创建的buffer，大小为client_header_buffer_size.
          
r->header_in = c->buffer;
      
}
  
```

然后我们来看当已经接收的request line或者request header大于设置的client_header_buffer_size的时候，nginx如果处理，这里nginx判断接收的数据大小在两个地方，一个是在处理request line，一个是处理request header时候。首先来看的是处理request line的时候，下面这段代码是在ngx_http_process_request_line中的，到达下面这里说明request line只解析了一部分，因此这里需要判断是否分配的buf已经全部使用了，如果全部使用则需要进入large header的处理部分。

```
  
//如果pos等于 end，说明buf已经完全使用。
          
if (r->header_in->pos == r->header_in->end) {
  
//进入large header处理部分.
              
rv = ngx_http_alloc_large_header_buffer(r, 1);

if (rv == NGX_ERROR) {
                  
ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                  
return;
              
}

if (rv == NGX_DECLINED) {
                  
r->request_line.len = r->header_in->end &#8211; r->request_start;
                  
r->request_line.data = r->request_start;
  
//返回414给客户端.
                  
ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                
"client sent too long URI");
                  
ngx_http_finalize_request(r, NGX_HTTP_REQUEST_URI_TOO_LARGE);
                  
return;
              
}
          
}
  
```

第二个地方就是处理request head，这部分是在ngx_http_process_request_headers中进行的,流程和上面的类似。
  
```
              
if (r->header_in->pos == r->header_in->end) {

rv = ngx_http_alloc_large_header_buffer(r, 0);

if (rv == NGX_ERROR) {
                      
ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                      
return;
                  
}
  
```

从上面两段代码我们可以看到处理large header的部分都是在ngx_http_alloc_large_header_buffer中。因此接下来我们就来详细的分析这个函数。

首先来看这个函数的声明
  
```
  
static ngx_int_t
  
ngx_http_alloc_large_header_buffer(ngx_http_request_t *r,
      
ngx_uint_t request_line)
  
```

可以看到它的第二个函数表示了是在处理request line还是说是request header。我们来一段段的分析这个函数。
  
下面这段首先判断如果是在处理完request_line并且状态为0，则说明用户的request line的第一行是空(注释里面写的比较详细)，此时我们需要忽略这个回车换行。
  
```
      
if (request_line && r->state == 0) {
          
/\* the client fills up the buffer with "\r\n" \*/

r->request_length += r->header_in->end &#8211; r->header_in->start;

r->header_in->pos = r->header_in->start;
          
r->header_in->last = r->header_in->start;

return NGX_OK;
      
}
  
```

然后接下来这部分就是判断已经解析完毕的request line或者header的大小是否大于large_client_header_buffers的大小，如果大于则说明当前的request 太长，所以直接返回。

```
      
old = request_line ? r->request_start : r->header_name_start;

cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
  
//state不为0，则说明request未解析完毕，此时如果已经解析完毕的部分大小太大，则直接返回.
      
if (r->state != 0
          
&& (size_t) (r->header_in->pos &#8211; old)
                                       
>= cscf->large_client_header_buffers.size)
      
{
          
return NGX_DECLINED;
      
}
  
```

接下来这部分就是准备分配新的内存供request使用。这里有一个http_connection的概念，在nginx中，这个主要用于pipeline请求和keepalive请求，等以后详细分析pipeline和keepalive的时候会涉及到这个东西，现在只需要知道这个东西主要是为了buf的重用来设计的。

```
  
//取得http_connection.
      
hc = r->http_connection;
  
//如果有已经被free，也就是空闲的buf，则我们就不用重新分配
      
if (hc->nfree) {
  
//直接取得buf。
          
b = hc->free[&#8211;hc->nfree];

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                         
"http large header free: %p %uz",
                         
b->pos, b->end &#8211; b->last);

} else if (hc->nbusy < cscf->large_client_header_buffers.num) {
  
//否则如果large_client_header_buffers的没有被使用完毕，则我们重新alloc新的buf.
          
if (hc->busy == NULL) {
              
hc->busy = ngx_palloc(r->connection->pool,
                    
cscf->large_client_header_buffers.num \* sizeof(ngx_buf_t \*));
              
if (hc->busy == NULL) {
                  
return NGX_ERROR;
              
}
          
}
  
//创建新的buf
          
b = ngx_create_temp_buf(r->connection->pool,
                                  
cscf->large_client_header_buffers.size);
          
if (b == NULL) {
              
return NGX_ERROR;
          
}

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                         
"http large header alloc: %p %uz",
                         
b->pos, b->end &#8211; b->last);

} else {
  
//否则直接返回。
          
return NGX_DECLINED;
      
}
  
```

下面这段就开始复制request buf.
  
```
  
//这里nbusy作为一个计数，用来限制large_client_header_buffers中限制的buf个数
      
hc->busy[hc->nbusy++] = b;

if (r->state == 0) {
          
/*
           
* r->state == 0 means that a header line was parsed successfully
           
* and we do not need to copy incomplete header line and
           
* to relocate the parser header pointers
           
*/

r->request_length += r->header_in->end &#8211; r->header_in->start;

r->header_in = b;

return NGX_OK;
      
}

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                     
"http large header copy: %d", r->header_in->pos &#8211; old);
  
//设置大小
      
r->request_length += old &#8211; r->header_in->start;
  
//指向新的buf
      
new = b->start;
  
//开始复制
      
ngx_memcpy(new, old, r->header_in->pos &#8211; old);
  
//更新buf
      
b->pos = new + (r->header_in->pos &#8211; old);
      
b->last = new + (r->header_in->pos &#8211; old);
  
```

最后一部分就是更新对应的指针
  
```
      
if (request_line) {
          
r->request_start = new;

if (r->request_end) {
              
r->request_end = new + (r->request_end &#8211; old);
          
}

r->method_end = new + (r->method_end &#8211; old);

r->uri_start = new + (r->uri_start &#8211; old);
          
r->uri_end = new + (r->uri_end &#8211; old);

if (r->schema_start) {
              
r->schema_start = new + (r->schema_start &#8211; old);
              
r->schema_end = new + (r->schema_end &#8211; old);
          
}

if (r->host_start) {
              
r->host_start = new + (r->host_start &#8211; old);
              
if (r->host_end) {
                  
r->host_end = new + (r->host_end &#8211; old);
              
}
          
}

if (r->port_start) {
              
r->port_start = new + (r->port_start &#8211; old);
              
r->port_end = new + (r->port_end &#8211; old);
          
}

if (r->uri_ext) {
              
r->uri_ext = new + (r->uri_ext &#8211; old);
          
}

if (r->args_start) {
              
r->args_start = new + (r->args_start &#8211; old);
          
}

if (r->http_protocol.data) {
              
r->http_protocol.data = new + (r->http_protocol.data &#8211; old);
          
}

} else {
          
r->header_name_start = new;
          
r->header_name_end = new + (r->header_name_end &#8211; old);
          
r->header_start = new + (r->header_start &#8211; old);
          
r->header_end = new + (r->header_end &#8211; old);
      
}

r->header_in = b;
  
```