---
id: 72
title: nginx中处理http header详解(2)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=72'
permalink: /2010/09/26/nginx%e4%b8%ad%e5%a4%84%e7%90%86http-header%e8%af%a6%e8%a7%a32/
views:
  - '19'
bot_views:
  - '20'
categories:
  - nginx
  - server
tags:
  - nginx
  - server
translate_title: detailed-explanation-of-http-header-in-nginx-(2)
date: 2010-09-26 14:04:08
---
然后是charset filter，这个主要是处理nginx内部的charset命令，转换为设置的编码。这个filter就不介绍了，主要是一个解码的过程。

再接下来是chunk filter,它主要是生成chunk数据，这里要注意nginx只支持服务端生成chunk，而不支持客户端发送的chunk数据。chunk的格式很简单，简单的来说就是大小+数据内容。
  
先来看chunk的header filter，在filter中，主要是用来判断是否需要chunk数据，然后设置相关标记位，以便于后面的body filter处理.
  
<!--more-->

```
  
static ngx_int_t
  
ngx_http_chunked_header_filter(ngx_http_request_t *r)
  
{
      
ngx_http_core_loc_conf_t *clcf;
  
//如果是304，204或者是head方法，则直接跳过chunk filter。
      
if (r->headers_out.status == NGX_HTTP_NOT_MODIFIED
          
|| r->headers_out.status == NGX_HTTP_NO_CONTENT
          
|| r != r->main
          
|| (r->method & NGX_HTTP_HEAD))
      
{
          
return ngx_http_next_header_filter(r);
      
}

//如果content_length_n为－1 则进入chunk处理，下面我们会看到这个值在那里设置，也就是nginx中如何打开chunk编码.
      
if (r->headers_out.content_length_n == -1) {
          
if (r->http_version < NGX_HTTP_VERSION_11) {
              
r->keepalive = 0;

} else {
              
clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
  
//chunked_transfer_encoding命令对应的参数，这个值默认是1，因此默认chunk是打开的。
              
if (clcf->chunked_transfer_encoding) {

r->chunked = 1;

} else {
                  
r->keepalive = 0;
              
}
          
}
      
}

return ngx_http_next_header_filter(r);
  
}
  
```
  
然后来看content_length_n何时被改变为－1，也就是准备chunk编码，这个值是在ngx_http_clear_content_length中被改变了。也就是如果希望chunk编码的话，必须调用这个函数。

```
  
#define ngx_http_clear_content_length(r) \
                                                                                
\
  
//设置长度为-1.
      
r->headers_out.content_length_n = -1; \
      
if (r->headers_out.content_length) { \
          
r->headers_out.content_length->hash = 0; \
          
r->headers_out.content_length = NULL; \
      
}
  
```

然后来看body filter是如何处理的。这里的处理其实很简单，只不过特殊处理下last buf.大体流程是这样子的，首先计算chunk的大小，然后讲将要发送的buf串联起来，然后将大小插入到数据buf之前，最后设置tail buf，如果是last buf，则结尾是
  
```
  
CRLF "0" CRLF CRLF
  
```
  
如果不是last buf，则结尾就是一个CRLF，这些都是严格遵守rfc2616。
  
来看详细的代码：
  
```
      
if (size) {
          
b = ngx_calloc_buf(r->pool);
          
if (b == NULL) {
              
return NGX_ERROR;
          
}

/\* the "0000000000000000" is 64-bit hexadimal string \*/
  
//分配大小
          
chunk = ngx_palloc(r->pool, sizeof("0000000000000000" CRLF) &#8211; 1);
          
if (chunk == NULL) {
              
return NGX_ERROR;
          
}

b->temporary = 1;
          
b->pos = chunk;
  
//设置chunk第一行的数据，也就是大小＋回车换行
          
b->last = ngx_sprintf(chunk, "%xO" CRLF, size);

out.buf = b;
      
}
  
//如果是最后一个buf。
      
if (cl->buf->last_buf) {
          
b = ngx_calloc_buf(r->pool);
          
if (b == NULL) {
              
return NGX_ERROR;
          
}

b->memory = 1;
          
b->last_buf = 1;
  
//设置结尾
          
b->pos = (u_char *) CRLF "0" CRLF CRLF;
   
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.

} else {
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
          
b->memory = 1;
  
//否则则是中间的chunk块，因此结尾为回车换行。
          
b->pos = (u_char *) CRLF;
          
b->last = b->pos + 2;
      
}

tail.buf = b;
      
tail.next = NULL;
      
*ll = &tail;
  
```

然后是gzip filter，它主要是处理gzip的压缩.其中在header filter中，判断accept-encoding头，来看客户端是否支持gzip压缩，然后设置Content-Encoding为gzip，以便与client解析。然后核心的处理都在body filter里面。

先来介绍下filter的主要流程，这里有一个要强调的，那就是nginx里面所有的filter处理基本都是流式的，也就是有多少处理多少。由于是gzip压缩，因此这里会有一个输入，一个输出，因此这里就分为3步，第一步取得输入buf，第二步设置输出buf，第三步结合前两步取得的buf，交给zlib库去压缩，然后输出到前面设置的buf。

```
      
for ( ;; ) {

/\* cycle while we can write to a client \*/

for ( ;; ) {

/\* cycle while there is data to feed zlib and &#8230; \*/
  
//设置正确的输入buf
              
rc = ngx_http_gzip_filter_add_data(r, ctx);

if (rc == NGX_DECLINED) {
                  
break;
              
}

if (rc == NGX_AGAIN) {
                  
continue;
              
}
  
//设置输出buf
              
rc = ngx_http_gzip_filter_get_buf(r, ctx);

if (rc == NGX_DECLINED) {
                  
break;
              
}

if (rc == NGX_ERROR) {
                  
goto failed;
              
}
  
//开始进行压缩
              
rc = ngx_http_gzip_filter_deflate(r, ctx);

if (rc == NGX_OK) {
                  
break;
              
}

if (rc == NGX_ERROR) {
                  
goto failed;
              
}
              
/\* rc == NGX_AGAIN \*/
          
}
  
```

这里有一个小细节要注意的，就是在ngx_http_gzip_filter_add_data中，在nginx中会一个chain一个chain进行gzip压缩，压缩完毕后，输入chain也就可以free掉了，可是nginx不是这么做的，他会在当所有的chain都被压缩完毕后再进行free，这是因为gzip压缩对于cpu cache很敏感，而当你free buf的时候，有可能会导致cache trashing，也就是会将一些cache的数据换出去。
  
```
      
if (ctx->copy_buf) {
  
//这里注释很详细
          
/*
           
* to avoid CPU cache trashing we do not free() just quit buf,
           
* but postpone free()ing after zlib compressing and data output
           
*/

ctx->copy_buf->next = ctx->copied;
          
ctx->copied = ctx->copy_buf;
  
//简单的赋为NULL，而不是free掉.
          
ctx->copy_buf = NULL;
      
}
  
```

最终在ngx_http_gzip_filter_free_copy_buf中free所有的gzip压缩的数据。从这里我们能看到nginx对于细节已经抓到什么地步了.

最后一个是header filter，也就是发送前最后一个head filter，这个filter里面设置对应的头以及status_lines，并且根据对应的status code设置对应的变量。所以这个filter是只有head filter的。这里的处理都没什么难的地方，就是简单的设置对应的头，因此就不详细的分析代码。它的流程大体就是先计算size，然后分配空间，最后copy对应的头。

就看一段代码，关于keepalive的，我们知道http1.1 keepalive是默认开启的，而http1.0它是默认关闭的，而nginx的keepalive_timeout命令则只是用来设置keepalive timeout的.对应clcf->keepalive_header。

```
  
//是否打开了keepalive，如果是1.1则默认是开启的
      
if (r->keepalive) {
  
//设置Connection的头.
          
len += sizeof("Connection: keep-alive" CRLF) &#8211; 1;
  
//不同浏览器行为不一样的。
          
/*
           
* MSIE and Opera ignore the "Keep-Alive: timeout=<N>" header.
           
* MSIE keeps the connection alive for about 60-65 seconds.
           
* Opera keeps the connection alive very long.
           
* Mozilla keeps the connection alive for N plus about 1-10 seconds.
           
* Konqueror keeps the connection alive for about N seconds.
           
*/

if (clcf->keepalive_header) {
  
//设置timeout
              
len += sizeof("Keep-Alive: timeout=") &#8211; 1 + NGX_TIME_T_LEN + 2;
          
}

} else {
          
len += sizeof("Connection: closed" CRLF) &#8211; 1;
      
}
  
```