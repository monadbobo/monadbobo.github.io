---
id: 235
title: nginx中if命令的设计和实现
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=235'
permalink: >-
  /2011/03/13/nginx%e4%b8%adif%e5%91%bd%e4%bb%a4%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0/
categories:
  - nginx
  - server
tags:
  - nginx
  - web server
  - 服务器设计
translate_title: design-and-implementation-of-if-command-in-nginx
date: 2011-03-13 09:13:03
---
先看这篇文章：http://wiki.nginx.org/IfIsEvil，这篇文章只是简单的介绍了if使用中一些很恶心的地方，接下来我会通过代码来看if为什么是 evil的。

if是rewrite模块里面的一个命令，因此if部分的执行也是在rewrite的phase执行的，下面就来简要的描述下rewrite模块是如何运行的。

这里有一个很关键的函数就是ngx_http_script_code_p，它的原型如下：
  
```typedef void (\*ngx_http_script_code_pt) (ngx_http_script_engine_t \*e);```
  
在rewrite模块中，所有将要在rewrite phase执行的代码的函数都会是一个ngx_http_script_code_pt类型的函数(比如rewrtie的正则匹配，比如if指令等等，而当进入rewrite handler的时候，将会依次执行这些函数，这些函数都是保存在ngx_http_script_engine_t中，下面我们来看这个结构。

```
  
typedef struct {
  
//这个指针指向了所有的需要执行的函数(ngx_http_script_code_pt)数组的首地址.
      
u_char *ip;
      
u_char *pos;
      
ngx_http_variable_value_t *sp;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//表示执行完对应的函数之后的返回值.
      
ngx_int_t status;
      
ngx_http_request_t *request;
  
} ngx_http_script_engine_t;
  
```
  
<!--more-->


  
接下来就是ngx_http_rewrite_handler函数，这个函数是rewrite phase的handler，可以看到它的实现比较简单，先取得将要执行的回调函数的地址，然后依次执行他们，最终通过返回值(e->status)来决定需要如何返回.
  
```
  
static ngx_int_t
  
ngx_http_rewrite_handler(ngx_http_request_t *r)
  
{
      
ngx_http_script_code_pt code;
      
ngx_http_script_engine_t *e;
      
ngx_http_rewrite_loc_conf_t *rlcf;

rlcf = ngx_http_get_module_loc_conf(r, ngx_http_rewrite_module);

if (rlcf->codes == NULL) {
          
return NGX_DECLINED;
      
}

e = ngx_pcalloc(r->pool, sizeof(ngx_http_script_engine_t));
   
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//取得回调函数的地址
      
e->ip = rlcf->codes->elts;
      
e->request = r;
      
e->quote = 1;
      
e->log = rlcf->log;
  
//默认返回值是declined
      
e->status = NGX_DECLINED;
  
//开始遍历回调函数.
      
while (\*(uintptr_t \*) e->ip) {
          
code = \*(ngx_http_script_code_pt \*) e->ip;
  
//执行回调，在回调函数中会更新ip指针，以便与下次调用.
          
code(e);
      
}

if (e->status == NGX_DECLINED) {
          
return NGX_DECLINED;
      
}

if (r->err_status == 0) {
          
return e->status;
      
}

return r->err_status;
  
}
  
```

了解了大体流程之后，我们来看if指令的实现。首先来看ngx_http_rewrite_if的实现，函数比较长，我们分段来看，首先是新建一个ctx，然后新建location(调用create_loc_conf），然后将新建的location挂载到新的ctx中，这里要注意server conf和main conf是不变的。

```
      
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
      
if (ctx == NULL) {
          
return NGX_CONF_ERROR;
      
}

pctx = cf->ctx;
  
//main conf和serv conf不变
      
ctx->main_conf = pctx->main_conf;
      
ctx->srv_conf = pctx->srv_conf;
  
//新建loc conf
      
ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void \*) \* ngx_http_max_module);
      
if (ctx->loc_conf == NULL) {
          
return NGX_CONF_ERROR;
      
}
  
//开始新建location conf
      
for (i = 0; ngx_modules[i]; i++) {
          
if (ngx_modules[i]->type != NGX_HTTP_MODULE) {
              
continue;
          
}

module = ngx_modules[i]->ctx;

if (module->create_loc_conf) {

mconf = module->create_loc_conf(cf);
              
if (mconf == NULL) {
                   
return NGX_CONF_ERROR;
              
}
              
ctx->loc_conf[ngx_modules[i]->ctx_index] = mconf;
          
}
      
}
  
```

接下来就是加新的location(ngx_http_add_location),紧接着就会解析if后面的指令(比如等号，括号等),通过不同的符号设置不同的回调函数，我们后面会分析这个函数,然后就是从codes属猪中取得对应的if_code,然后设置code值,也就是回调函数。
  
```
      
clcf = ctx->loc_conf[ngx_http_core_module.ctx_index];
      
clcf->loc_conf = ctx->loc_conf;
      
clcf->name = pclcf->name;
      
clcf->noname = 1;
  
//加location
      
if (ngx_http_add_location(cf, &pclcf->locations, clcf) != NGX_OK) {
          
return NGX_CONF_ERROR;
      
}
  
//设置if的条件对应的回调.
      
if (ngx_http_rewrite_if_condition(cf, lcf) != NGX_CONF_OK) {
          
return NGX_CONF_ERROR;
      
}
  
//从数组中取得元素(codes默认是一个每个元素为1个字节的数组).
      
if_code = ngx_array_push_n(lcf->codes, sizeof(ngx_http_script_if_code_t));
      
if (if_code == NULL) {
          
return NGX_CONF_ERROR;
      
}
  
//给code赋值，后面会详细分析这个回调函数.
      
if_code->code = ngx_http_script_if_code;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//如果name长度为0，则说明这是一个server if。
      
if (pclcf->name.len == 0) {
  
//此时loc就为null
          
if_code->loc_conf = NULL;
          
cf->cmd_type = NGX_HTTP_SIF_CONF;

} else {
  
//否则保存对应loc_conf,这里loc_conf里面保存了我们需要的信息.
          
if_code->loc_conf = ctx->loc_conf;
          
cf->cmd_type = NGX_HTTP_LIF_CONF;
      
}
  
//解析，这时if 作用域里面的命令都会保存在if_code->loc_conf中.因为上面我们改变了cf本身的loc conf
      
rv = ngx_conf_parse(cf, NULL);
  
```

接下来来看ngx_http_rewrite_if_condition,这个函数比较长，我们就关注当if的条件是等于时的情况，其它的情况都类似。它也是会设置一个回调函数(code).

```
  
static char *
  
ngx_http_rewrite_if_condition(ngx_conf_t \*cf, ngx_http_rewrite_loc_conf_t \*lcf)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
         
if (len == 1 && p[0] == &#8216;=&#8217;) {

if (ngx_http_rewrite_value(cf, lcf, &value[last]) != NGX_CONF_OK) {
                  
return NGX_CONF_ERROR;
              
}
  
//从codes数组中得到对应的值。
              
code = ngx_http_script_start_code(cf->pool, &lcf->codes,
                                                
sizeof(uintptr_t));
              
if (code == NULL) {
                  
return NGX_CONF_ERROR;
              
}
  
//然后赋值。
              
*code = ngx_http_script_equal_code;

return NGX_CONF_OK;
          
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
}
  
```

然后来看ngx_http_script_equal_code，它主要是会判断if中声明的两个值是否相等，如果相等则设置对应的值为ngx_http_variable_true_value，否则设置为ngx_http_variable_null_value，以供后面调用ngx_http_script_if_code时判断。
  
```
  
void
  
ngx_http_script_equal_code(ngx_http_script_engine_t *e)
  
{
      
ngx_http_variable_value_t \*val, \*res;

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                     
"http script equal");

e->sp&#8211;;
      
val = e->sp;
      
res = e->sp &#8211; 1;

e->ip += sizeof(uintptr_t);
  
//比较是否相等
      
if (val->len == res->len
          
&& ngx_strncmp(val->data, res->data, res->len) == 0)
      
{
  
//相等赋值为ngx_http_variable_true_value
          
*res = ngx_http_variable_true_value;
          
return;
      
}

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                     
"http script equal: no");

*res = ngx_http_variable_null_value;
  
}
  
```

最后来看ngx_http_script_if_cod，它主要是就是根据前面的函数设置的变量来判断是否if条件成立，如果成立，则将在ngx_http_rewrite_if保存的loc conf赋值为当前的request的loc conf.这样，接下来的都会使用新的loc conf.
  
```
  
void
  
ngx_http_script_if_code(ngx_http_script_engine_t *e)
  
{
      
ngx_http_script_if_code_t *code;

code = (ngx_http_script_if_code_t *) e->ip;

ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                     
"http script if");

e->sp&#8211;;
  
//判断if的条件是否成立
      
if (e->sp->len && e->sp->data[0] != &#8216;0&#8217;) {
          
if (code->loc_conf) {
              
ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                  
"http script if: update");
  
//修改loc conf，然后update。
              
e->request->loc_conf = code->loc_conf;
              
ngx_http_update_location_config(e->request);
          
}

e->ip += sizeof(ngx_http_script_if_code_t);
          
return;
      
}
  
//否则修改ip，然后进入下面的处理
      
ngx_log_debug0(NGX_LOG_DEBUG_HTTP, e->request->connection->log, 0,
                     
"http script if: false");

e->ip += code->next;
  
}
  
```

最后来看一开始的nginx wiki中的几个if的例子。

从上面可以看到最关键的一个就是update loc conf的那段，而loc是每次在解析if指令的时候创建的，因此如果我们的指令在if之前就被解析的话，此时if中这个指令的设置就是无效的，我们来看一开始nginx wiki中的2个例子：

```
  
location /proxy-pass-uri {
              
proxy_pass http://127.0.0.1:8080/;

set $true 1;

if ($true) {
                  
\# nothing
              
}
          
}

\# try_files wont work due to if

location /if-try-files {
               
try_files /file @fallback;

set $true 1;

if ($true) {
                   
\# nothing
               
}
          
}

```

可以看到如果进入if的话，location里面的指令将不会被继承。所以对应的proxy_pass 和try_files都不会在if里面起作用.

而如果有两个if的话，第二个将会覆盖第一个，所以在下面的这个里面只有第二个会起作用.
  
```
  
location /only-one-if {
              
set $true 1;

if ($true) {
                  
add_header X-First 1;
              
}

if ($true) {
                  
add_header X-Second 2;
              
}

return 204;
          
}
  
```

不知道igor以后会不会改写if，我的想法是，把if放到core http module,然后单独做一个if作用域，它要么属于server要么属于loc，然后每次解析对应的server或者loc的时候，merge存在的if作用域就可以了。