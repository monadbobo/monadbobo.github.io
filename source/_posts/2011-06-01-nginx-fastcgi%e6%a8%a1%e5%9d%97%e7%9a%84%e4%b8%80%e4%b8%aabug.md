---
id: 293
title: nginx fastcgi模块的一个bug
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=293'
permalink: /2011/06/01/nginx-fastcgi%e6%a8%a1%e5%9d%97%e7%9a%84%e4%b8%80%e4%b8%aabug/
categories:
  - nginx
  - server
translate_title: bug-of-nginx-fastcgi-module
date: 2011-06-01 11:39:17
---
上周服务器更新到nginx的0.8.X之后，nginx出现了core dump的情况，而在0.7.X并不会出现，通过察看core dump文件以及nginx 0.8.x和0.7.x的比较，发现core dump是nginx 0.8.40引入下面这个feature才导致的:

> *) Feature: a &#8220;fastcgi_param&#8221; directive with value starting with
         
> &#8220;HTTP_&#8221; overrides a client request header line.

在nginx 0.8.40之后，如果你的fastcgi_param定义的变量以HTTP_开头，则传递给后端的头会忽略request header中的这个头，比如定义了一个 fastcgi_param $HTTP_HOST test, 那么传递给后端时，host这个头的值就是test.

这里的逻辑是这样子的，当nginx创建一个fastcgi request的时候，会先计算所需要的长度，首先是计算header的长度，在计算之前会先分配一个ignored数组(用来保存将要被忽略的头),它的大小是配置文件中fastcgi_param定义的以HTTP_开头的变量的个数. 然后遍历所有的request header，如果发现header的名字和fastcgi_param中定义的变量的(HTTP_开头)名字相同(使用hash)，则将这个header指针放到ignored数组中，最后在拷贝request header的时候直接在这个数组里面查找，如果有则跳过，否则拷贝头以及它的值。

看起来没什么问题，可是这里忽略了request header有可能会有重复的这个情况，此时ignored数组可能就会越界，从而导致core dump.

<!--more-->


  
来看对应的代码,引起问题的代码是下面这段(ngx_http_fastcgi_create_request).

```
  
//这里header_params就是fastcgi_param中定义的变量的(HTTP_开头)个数
          
if (flcf->header_params) {
  
//分配内存
              
ignored = ngx_palloc(r->pool, flcf->header_params \* sizeof(void \*));
              
if (ignored == NULL) {
                  
return NGX_ERROR;
              
}
          
}

part = &r->headers_in.headers.part;
          
header = part->elts;
  
//开始遍历
          
for (i = 0; /\* void \*/; i++) {

if (i >= part->nelts) {
                  
if (part->next == NULL) {
                      
break;
                  
}

part = part->next;
                  
header = part->elts;
                  
i = 0;
              
}

if (flcf->header_params) {
           
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//headers_hash就是fastcgi_param中定义的变量(HTTP_开头)名字的hash表
                  
if (ngx_hash_find(&flcf->headers_hash, hash, lowcase_key, n)) {
  
//可以看到只要找到相同的hash，则header_params就会加一.而如果重复的头大于fastcgi_param中定义的变量的(HTTP_开头)的个数，则ignored肯定会越界.
                      
ignored[header_params++] = &header[i];
                      
continue;
                  
}

n += sizeof("HTTP_") &#8211; 1;

} else {
                  
n = sizeof("HTTP_") &#8211; 1 + header[i].key.len;
              
}
  
```

举个例子，配置文件里面包含下面的命令：
  
fastcgi_param HTTP_HOST $http_host;

然后客户端传过来的头中如果包含多个host头，则nginx就会core dump掉.