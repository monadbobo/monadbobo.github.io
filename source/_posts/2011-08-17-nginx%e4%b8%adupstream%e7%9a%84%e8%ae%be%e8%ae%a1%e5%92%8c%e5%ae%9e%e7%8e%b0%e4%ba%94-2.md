---
id: 339
title: nginx中upstream的设计和实现(五)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=339'
permalink: >-
  /2011/08/17/nginx%e4%b8%adupstream%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0%e4%ba%94-2/
categories:
  - nginx
  - server
tags:
  - nginx
  - server
  - web server
translate_title: design-and-implementation-of-upstream-in-nginx-(5)
date: 2011-08-17 13:22:45
---
这次主要来分析upstream中的发送数据给client, 以及当buf不足，将一部分写到temp file的部分，他们对应的函数分别是ngx_event_pipe_write_to_downstream和ngx_event_pipe_write_chain_to_temp_file.

先来看ngx_event_pipe_write_to_downstream，这个函数顾名思义，就是写buf到临时文件。而所写的buf就是p->in,也就是将要发送给client的数据。
  
这个函数，它会处理两类的情况，一类是cache打开，一类是cache未打开。我们这里主要来分析cache关闭的情况。

首先来看这个函数的第一部分的代码,这部分代码主要是遍历p->in,然后计算能写多少buf到文件(temp file的size是有限制的).

```
  
//out就是将要保存到file的数据
      
if (p->buf_to_file) {
  
//cache打开的情况
          
fl.buf = p->buf_to_file;
          
fl.next = p->in;
          
out = &fl;

} else {
  
//得到数据
          
out = p->in;
      
}
  
//如果cache没有打开
      
if (!p->cacheable) {

size = 0;
          
cl = out;
          
ll = NULL;

ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                         
"pipe offset: %O", p->temp_file->offset);
  
//开始遍历out
          
do {
  
//计算大小
              
bsize = cl->buf->last &#8211; cl->buf->pos;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//看是否超过限制限制
              
if ((size + bsize > p->temp_file_write_size)
                 
|| (p->temp_file->offset + size + bsize > p->max_temp_file_size))
              
{
                  
break;
              
}

size += bsize;
              
ll = &cl->next;
              
cl = cl->next;

} while (cl);

ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0, "size: %z", size);

if (ll == NULL) {
              
return NGX_BUSY;
          
}
  
//cl存在则说明只有一部分buf能够写入到temp file，此时p->in保存剩下的chain
          
if (cl) {
             
p->in = cl;
             
*ll = NULL;

} else {
  
//否则说明所有的buf都写入到了temp file，此时p->in则设置为空
             
p->in = NULL;
             
p->last_in = &p->in;
          
}

} else {
  
//cache打开的情况，可以看到和上面类似.
          
p->in = NULL;
          
p->last_in = &p->in;
      
}
  
```
  
<!--more-->

然后是第二部分，也就是最后一部分，这部分就是写buf到temp file，然后将已经写到temp file的buf，挂载到free_raw_bufs，以供继续使用。这里有用到shadow buf，因为shadow buf的内存是已经分配好的，而当已经写入到temp file之后，这部分buf自然就可以重新使用了。

还有一个很重要的操作，那就是将已经写入到temp file的buf 挂载到p->out上.

```
  
//写out到tempfile
      
if (ngx_write_chain_to_temp_file(p->temp_file, out) == NGX_ERROR) {
          
return NGX_ABORT;
      
}
  
//遍历free_raw_bufs,以便与接下来将使用过的buf挂载到后面。
      
for (last_free = &p->free_raw_bufs;
           
*last_free != NULL;
           
last_free = &(*last_free)->next)
      
{
          
/\* void \*/
      
}

if (p->buf_to_file) {
          
p->temp_file->offset = p->buf_to_file->last &#8211; p->buf_to_file->pos;
          
p->buf_to_file = NULL;
          
out = out->next;
      
}
  
//遍历out
      
for (cl = out; cl; cl = next) {
          
next = cl->next;
          
cl->next = NULL;

b = cl->buf;
  
//可以看到重新设置buf为file。然后设置相关属性
          
b->file = &p->temp_file->file;
          
b->file_pos = p->temp_file->offset;
          
p->temp_file->offset += b->last &#8211; b->pos;
          
b->file_last = p->temp_file->offset;

b->in_file = 1;
          
b->temp_file = 1;
  
//这里就是将buf放入到p->out.
          
if (p->out) {
              
*p->last_out = cl;
          
} else {
              
p->out = cl;
          
}
  
//设置last_out
          
p->last_out = &cl->next;
  
//shadow存在，则将已经保存到file的buf挂载到free_raw_buf中。
          
if (b->last_shadow) {

tl = ngx_alloc_chain_link(p->pool);
              
if (tl == NULL) {
                  
return NGX_ABORT;
              
}
  
//可以看到使用它的shadow
              
tl->buf = b->shadow;
              
tl->next = NULL;
  
//last_free就是free_raw_buf的尾部
              
*last_free = tl;
              
last_free = &tl->next;
  
//reset buf
              
b->shadow->pos = b->shadow->start;
              
b->shadow->last = b->shadow->start;
  
//remove掉shadow。
              
ngx_event_pipe_remove_shadow_links(b->shadow);
          
}
      
}
  
```

然后来看ngx_event_pipe_write_to_downstream，也就是发送数据到client的部分，这个函数整体是一个大循环， 而在这个大循环内分为三部分，其中第一部分处理upstream已经发送完毕(比如断开连接，出错等)时的情况，第二部分是对发送前的buf进行一些处理(比如busy buf，p->in,p->out等),然后循环发送,第三部分就是调用发送接口（output_filter)发送数据，然后update chain.

ok，接下来我们就来看这三部分。先来看第一部分，这部分就是处理upstream端已经发送完毕的情况，此时需要我们立即发送数据到downstream。

这里需要注意的是在upstream发送的时候，对于buf的选择有一个顺序，通过前面一篇blog，我们能看到当拷贝从upstream的数据的时候，就有一个顺序。
  
这个顺序是这样子的，首先是p->out,然后是p->in,因为p->out保存了一些buf在文件中的buf。

然后时buf的recycle属性，这个属性主要目的是这样子的，由于在p->input_filter中，nginx制造了一个buf充当已经读取的buf的shadow(last_shadow = 1)，而在以后，nginx操作的都是这个buf，这个buf就被设置为recycled，也就是可以循环利用。而且当设置了recycled，则也将会在发送数据的时候，立即将buf发送出去，而不会缓存(可以看ngx_http_write_filter中的判断).

```
  
//判断upstream状态
          
if (p->upstream_eof || p->upstream_error || p->upstream_done) {

/\* pass the p->out and p->in chains to the output filter \*/
  
//取消掉recycled设置，
              
for (cl = p->busy; cl; cl = cl->next) {
                  
cl->buf->recycled = 0;
              
}
  
//首先发送p->out.
              
if (p->out) {
                  
ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                                 
"pipe write downstream flush out");
  
//取消掉recycled设置，因为已经不要回收这个buf了(后端不会有数据过来)。
                  
for (cl = p->out; cl; cl = cl->next) {
                      
cl->buf->recycled = 0;
                  
}
  
//调用filter函数
                  
rc = p->output_filter(p->output_ctx, p->out);
  
//如果发送失败，则设置downstream_error,并且回收chain
                  
if (rc == NGX_ERROR) {
                      
p->downstream_error = 1;
                      
return ngx_event_pipe_drain_chains(p);
                  
}

p->out = NULL;
              
}
  
//发送p->in
              
if (p->in) {
                  
ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                                 
"pipe write downstream flush in");
  
//取消recycled设置
                  
for (cl = p->in; cl; cl = cl->next) {
                      
cl->buf->recycled = 0;
                  
}
  
//发送数据
                  
rc = p->output_filter(p->output_ctx, p->in);
  
//同上
                  
if (rc == NGX_ERROR) {
                      
p->downstream_error = 1;
                      
return ngx_event_pipe_drain_chains(p);
                  
}

p->in = NULL;
              
}
  
//cache相关设置
              
if (p->cacheable && p->buf_to_file) {

file.buf = p->buf_to_file;
                  
file.next = NULL;

if (ngx_write_chain_to_temp_file(p->temp_file, &file)
                      
== NGX_ERROR)
                  
{
                      
return NGX_ABORT;
                  
}
              
}

ngx_log_debug0(NGX_LOG_DEBUG_EVENT, p->log, 0,
                             
"pipe write downstream done");

/\* TODO: free unused bufs \*/

p->downstream_done = 1;
              
break;
          
}
  
```

紧接着是第二部分，这部分主要是处理upstream的数据并没有发送完全，此时nginx会尽量发送最大的可能的数据到client。

这里busy就保存了上次发送没有发送完毕的chain，它主要是为了方便统计。

它的步骤是这样子的，首先会计算busy chain的大小，因为我们有busy chain的限制(有busy buf的命令).然后计算p->in/p->out,最后得到一个最大的chain，然后发送。这里要注意，我们发送的每个buf大小是不会大于busy buf的大小的。

```
  
//判断是否需要退出循环，这里比较关键的就是downstream->write->ready，当发送返回again，就会从这里退出
          
if (downstream->data != p->output_ctx
              
|| !downstream->write->ready
              
|| downstream->write->delayed)
          
{
              
break;
          
}

prev = NULL;
          
bsize = 0;
  
//首先计算busy的chain的大小
          
for (cl = p->busy; cl; cl = cl->next) {

if (cl->buf->recycled) {
  
//如果是相同的chain，则跳过计算
                  
if (prev == cl->buf->start) {
                      
continue;
                  
}
  
//计算大小
                  
bsize += cl->buf->end &#8211; cl->buf->start;
                  
prev = cl->buf->start;
              
}
          
}

ngx_log_debug1(NGX_LOG_DEBUG_EVENT, p->log, 0,
                         
"pipe write busy: %uz", bsize);

out = NULL;
  
//如果bsize大于我们设置的busy buf size,则直接发送数据
          
if (bsize >= (size_t) p->busy_size) {
              
flush = 1;
              
goto flush;
          
}

flush = 0;
          
ll = NULL;
          
prev_last_shadow = 1;
  
//然后开始处理p->out和p->in
          
for ( ;; ) {
              
if (p->out) {
                  
cl = p->out;
  
//计算将要发送的buf是否大于我们设置的busy_size，而cl->buf->last &#8211; cl->buf->pos是肯定小于等于busy_size.
                  
if (cl->buf->recycled
                      
&& bsize + cl->buf->last &#8211; cl->buf->pos > p->busy_size)
                  
{
  
//此时当前的cl就不能发送，所以设置flush，然后立即发送
                      
flush = 1;
                      
break;
                  
}

p->out = p->out->next;
  
//将shadow buf 放到free_raw_bufs中，以便后面使用
                  
ngx_event_pipe_free_shadow_raw_buf(&p->free_raw_bufs, cl->buf);

} else if (!p->cacheable && p->in) {
                  
cl = p->in;

ngx_log_debug3(NGX_LOG_DEBUG_EVENT, p->log, 0,
                                 
"pipe write buf ls:%d %p %z",
                                 
cl->buf->last_shadow,
                                 
cl->buf->pos,
                                 
cl->buf->last &#8211; cl->buf->pos);
  
//类似上面的，也是需要计算buf大小，这里可以看到我们只操作影子(shadow)buf，
                  
if (cl->buf->recycled
                      
&& cl->buf->last_shadow
                      
&& bsize + cl->buf->last &#8211; cl->buf->pos > p->busy_size)
                  
{
                      
if (!prev_last_shadow) {
  
//设置out chain
                          
p->in = p->in->next;

cl->next = NULL;

if (out) {
                              
*ll = cl;
                          
} else {
                              
out = cl;
                          
}
                      
}

flush = 1;
                      
break;
                  
}

prev_last_shadow = cl->buf->last_shadow;

p->in = p->in->next;

} else {
                  
break;
              
}
  
//如果cl是recycled，则说明这个buf会被发送，因此bsize更新
              
if (cl->buf->recycled) {
                  
bsize += cl->buf->last &#8211; cl->buf->pos;
              
}

cl->next = NULL;
  
//将对应的chain绑定到out上，接下来就会发送out。
              
if (out) {
                  
*ll = cl;
              
} else {
                  
out = cl;
              
}
              
ll = &cl->next;
          
}
  
```

然后是最后一部分，也就是发送chain，然后update chain的操作(类似chain output的操作)

这里用有一个很重要的操作，通过上面的分析我们知道，在upstream中，基本上每个buf都会有一个shadow，而我们发送的时候，用的是shadow(last_shadow=1),当发送完毕，这个buf会被放到p->free中，此时还有一个buf，那就是原始的buf(这个buf是有分配内存的,b->start不为null)，因此这里就可以重用这个buf，在nginx中会将发送完毕后的这个buf放到free_raw_buf中。

```
      
flush:
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//发送数据到client
          
rc = p->output_filter(p->output_ctx, out);
  
//错误的话，设置downstream_error
          
if (rc == NGX_ERROR) {
              
p->downstream_error = 1;
              
return ngx_event_pipe_drain_chains(p);
          
}
  
//update chain,将已经完全发送的chain保存到free，还没发送的保存到busy.
          
ngx_chain_update_chains(&p->free, &p->busy, &out, p->tag);
  
//遍历free chain,
          
for (cl = p->free; cl; cl = cl->next) {

if (cl->buf->temp_file) {
                  
if (p->cacheable || !p->cyclic_temp_file) {
                      
continue;
                  
}

/\* reset p->temp_offset if all bufs had been sent \*/

if (cl->buf->file_last == p->temp_file->offset) {
                      
p->temp_file->offset = 0;
                  
}
              
}

/\* TODO: free buf if p->free_bufs && upstream done \*/

/\* add the free shadow raw buf to p->free_raw_bufs \*/
  
//将shadow的buf放到p->free_raw_buf中.
              
if (cl->buf->last_shadow) {
  
//可以看到这里操作的是cl->buf->shadow,也就是我们在p->input_filter中，拷贝的那个原始buf。
                  
if (ngx_event_pipe_add_free_buf(p, cl->buf->shadow) != NGX_OK) {
                      
return NGX_ABORT;
                  
}

cl->buf->last_shadow = 0;
              
}

cl->buf->shadow = NULL;
          
}
  
```

上面的分析，还遗漏了一个函数，那就是ngx_event_pipe_drain_chains，这个函数被调用，说明client出错，此时则需要将对应的shadow buf放到free_raw_buf中(调用(ngx_event_pipe_add_free_buf).

最后还有一个问题没解决，那就是当client断开连接时(当接收完毕所有的header)，nginx如何来处理，通过上面的代码，我们能够看到，当发送数据失败时，只是调用ngx_event_pipe_drain_chains然后返回，那么其实这里处理和nginx处理一般的http请求是一样的，那就是暂时忽略client的断开，也就是不管client的状态，当upstream发送完毕，才会认为request处理完成，这样子能简化代码的处理。

并且，这里只有cache没有打开的时候，才会finalize 当前的request，这是因为当cache打开的时候，当前的request 是需要被cache，然后下次请求再次到达，就可以发送cache的了。所以此时需要接受完后端所有的数据。

下面我们就来看这部分代码。

```
  
static void
  
ngx_http_upstream_process_request(ngx_http_request_t *r)
  
{
      
ngx_uint_t del;
      
ngx_temp_file_t *tf;
      
ngx_event_pipe_t *p;
      
ngx_http_upstream_t *u;

u = r->upstream;
      
p = u->pipe;
  
//当finalize upstream之后，connection将会被赋值为NULL
      
if (u->peer.connection) {

if (u->store) {

del = p->upstream_error;

tf = u->pipe->temp_file;

if (p->upstream_eof || p->upstream_done) {

if (u->headers_in.status_n == NGX_HTTP_OK
                      
&& (u->headers_in.content_length_n == -1
                          
|| (u->headers_in.content_length_n == tf->offset)))
                  
{
                      
ngx_http_upstream_store(r, u);

} else {
                      
del = 1;
                  
}
              
}

if (del && tf->file.fd != NGX_INVALID_FILE) {

if (ngx_delete_file(tf->file.name.data) == NGX_FILE_ERROR) {

ngx_log_error(NGX_LOG_CRIT, r->connection->log, ngx_errno,
                                    
ngx_delete_file_n " \"%s\" failed",
                                    
u->pipe->temp_file->file.name.data);
                  
}
              
}
          
}
  
//如果upstream端发送完毕(断开等),则finalize request。
          
if (p->upstream_done || p->upstream_eof || p->upstream_error) {
              
ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                             
"http upstream exit: %p", p->out);
              
ngx_http_upstream_finalize_request(r, u, 0);
              
return;
          
}
      
}
  
//如果downstream error设置，则进入下面的处理
      
if (p->downstream_error) {
          
ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                         
"http upstream downstream error");
  
//这里可以看到如果cache没有打开，则finalize 当前的request
          
if (!u->cacheable && !u->store && u->peer.connection) {
              
ngx_http_upstream_finalize_request(r, u, 0);
          
}
      
}
  
}
  
```