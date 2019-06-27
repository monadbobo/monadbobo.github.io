---
id: 375
title: nginx中cache的设计和实现(一)
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=375'
permalink: >-
  /2012/03/21/nginx%e4%b8%adcache%e7%9a%84%e8%ae%be%e8%ae%a1%e5%92%8c%e5%ae%9e%e7%8e%b0%e4%b8%80/
posturl_add_url:
  - 'yes'
categories:
  - nginx
  - server
tags:
  - nginx
  - opensource
  - web server
translate_title: design-and-implementation-of-cache-in-nginx-1
date: 2012-03-21 05:22:41
---
Nginx的cache实现比较简单，没有使用内存，全部都是使用文件来对后端的response进行cache，因此nginx相比varnish以及squid之类的专门做cache的server，可能效果不会那么好。特别如果cache内容比较大的话。不过还有一种折衷的处理，那就是挂载一个内存盘，然后让nginx cache到这个盘。

我这里的看的代码是1.1.17.

首先来看Nginx中如何开启cache，http cache主要是应用在upstream中的，因此upstream对应的两个命令来启用cache，一个是xxx_cache_path(比如proxy_cache_path)，它主要是用来创建管理cache的共享内存数据结构(红黑树和队列).一个是xxx_cache,它主要是使用前面创建的zone。

先来看第一个命令，xxx_cache_path,它会调用ngx_http_file_cache_set_slot函数，在看这个函数之前，先来看ngx_http_file_cache_t这个数据结构，它主要用来管理所有的cache文件，它本身不保存cache，只是保存管理cache的数据结构。每一个xxx_cache_path都会创建一个ngx_http_file_cache_t.
  
<!--more-->


  
```
  
typedef struct {
      
ngx_rbtree_t rbtree;
      
ngx_rbtree_node_t sentinel;
      
ngx_queue_t queue;
  
//cold表示这个cache是否已经被loader进程load过了
      
ngx_atomic_t cold;
  
//那个进程正在load这个cache
      
ngx_atomic_t loading;
  
//文件大小
      
off_t size;
  
} ngx_http_file_cache_sh_t;

struct ngx_http_file_cache_s {
      
ngx_http_file_cache_sh_t *sh;
      
ngx_slab_pool_t *shpool;
  
//cache的目录
      
ngx_path_t *path;
  
//当前的path下的所有cache文件的最大值
      
off_t max_size;
      
size_t bsize;
  
//如果多久不使用就被删除
      
time_t inactive;
  
//当前有多少个cache文件(超过loader_files之后会被清0)
      
ngx_uint_t files;
  
//这个值也就是一个阈值，当load的文件个数大于这个值之后，load进程会短暂的休眠(时间位loader_sleep)
      
ngx_uint_t loader_files;
  
//最后被manage或者loader访问的时间
      
ngx_msec_t last;
  
//和上面的loader_files配合使用，当文件个数大于loader_files，就会休眠
      
ngx_msec_t loader_sleep;
  
//配合上面的last，也就是loader遍历的休眠间隔。
      
ngx_msec_t loader_threshold;
  
//共享内存的地址
      
ngx_shm_zone_t *shm_zone;
  
};
  
```

然后这里有一个很关键的结构就是ngx_path_t,它保存了当前的cache的一些信息，以及对应的cache管理回调函数manger和loader.
  
```
  
typedef struct {
  
//cache名字
      
ngx_str_t name;
      
size_t len;
      
size_t level[3];
  
//对应的回调，以及回调数据
      
ngx_path_manager_pt manager;
      
ngx_path_loader_pt loader;
      
void *data;

u_char *conf_file;
      
ngx_uint_t line;
  
} ngx_path_t;
  
```

然后我们来看ngx_http_file_cache_set_slot函数，这个函数中可以看到上面的结构都是如何被初始化的。函数比较长，我们来看关键的部分：
  
```
      
ngx_http_file_cache_t *cache;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//初始化cache对象，这两个回调后续会详细分析
      
cache->path->manager = ngx_http_file_cache_manager;
      
cache->path->loader = ngx_http_file_cache_loader;
      
cache->path->data = cache;
      
cache->path->conf_file = cf->conf_file->file.name.data;
      
cache->path->line = cf->conf_file->line;
  
//初始化这几个阈值
      
cache->loader_files = loader_files;
      
cache->loader_sleep = (ngx_msec_t) loader_sleep;
      
cache->loader_threshold = (ngx_msec_t) loader_threshold;
  
//将path添加到全局的路径管理中
      
if (ngx_add_path(cf, &cache->path) != NGX_OK) {
          
return NGX_CONF_ERROR;
      
}
  
//创建共享内存
      
cache->shm_zone = ngx_shared_memory_add(cf, &name, size, cmd->post);
      
if (cache->shm_zone == NULL) {
          
return NGX_CONF_ERROR;
      
}

if (cache->shm_zone->data) {
          
ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                             
"duplicate zone \"%V\"", &name);
          
return NGX_CONF_ERROR;
      
}

//设置初始化函数
      
cache->shm_zone->init = ngx_http_file_cache_init;
      
cache->shm_zone->data = cache;
  
//设置cache
      
cache->inactive = inactive;
      
cache->max_size = max_size;
  
```

然后是每个upstream模块的xxx_cache命令，这个命令对应xxx_cache函数，我们这里就看看ngx_http_fastcgi_cache函数。这个函数更简单,就是简单的查找出上面创建的共享内存。

```
  
static char *
  
ngx_http_fastcgi_cache(ngx_conf_t \*cf, ngx_command_t \*cmd, void *conf)
  
{
   
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//查找出共享内存
      
flcf->upstream.cache = ngx_shared_memory_add(cf, &value[1], 0,
                                                   
&ngx_http_fastcgi_module);
      
if (flcf->upstream.cache == NULL) {
          
return NGX_CONF_ERROR;
      
}

return NGX_CONF_OK;
  
}
  
```

然后来看cache系统的启动。当配置解析完毕之后就会进入进程的初始化部分也就是ngx_master_process_cycle 这个函数，而在这个函数中将会启动nginx的worker。 并且会启动对应的cache manger和cache loader，也就是说cache manger和cache loader是一对子进程，并且独立于worker。

cache manger的作用是用来定时删除无用的cache文件(引用计数为0),一般来说只有manger会删除无用的cache(特殊情况，比如在loader中分配共享内存失败可能会强制删除一些cache， 或者说 loader的时候遇到一些特殊文件).

cache loader的的主要作用是定时遍历cache目录，然后加载一些没有被加载的文件(比如nginx重启后，也就是上次遗留的文件),或者说将cache文件重新插入(因为删除是使用LRU算法),后续能够看到代码细节。

接下来就来看代码，首先在中 ngx_master_process_cycle会调用ngx_start_cache_manager_processes来启动cache子系统，我们来看这个函数的片段：
  
```
  
//启动manger子进程
      
ngx_spawn_process(cycle, ngx_cache_manager_process_cycle,
                        
&ngx_cache_manager_ctx, "cache manager process",
                        
respawn ? NGX_PROCESS_JUST_RESPAWN : NGX_PROCESS_RESPAWN);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
//启动loader子进程
      
ngx_spawn_process(cycle, ngx_cache_manager_process_cycle,
                        
&ngx_cache_loader_ctx, "cache loader process",
                        
respawn ? NGX_PROCESS_JUST_SPAWN : NGX_PROCESS_NORESPAWN);
  
```

从上面可以看到会fork两个子进程，然后子进程的回调是ngx_cache_manager_process_cycle这个函数，这个函数比较简单，就是设置定时器:
  
```
  
static void
  
ngx_cache_manager_process_cycle(ngx_cycle_t \*cycle, void \*data)
  
{
      
ngx_cache_manager_ctx_t *ctx = data;

void *ident[4];
      
ngx_event_t ev;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
      
ngx_memzero(&ev, sizeof(ngx_event_t));
  
//最核心的在这里了设置传递进来的ctx->handler为定时器回调
      
ev.handler = ctx->handler;
      
ev.data = ident;
      
ev.log = cycle->log;
      
ident[3] = (void *) -1;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//添加定时器
      
ngx_add_timer(&ev, ctx->delay);
  
//进入事件循环
      
for ( ;; ) {

if (ngx_terminate || ngx_quit) {
              
ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
              
exit(0);
          
}

if (ngx_reopen) {
              
ngx_reopen = 0;
              
ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
              
ngx_reopen_files(cycle, -1);
          
}

ngx_process_events_and_timers(cycle);
      
}
  
}```

上面的函数可以看到定时器的回调会设置为传递给ngx_cache_manager_process_cycle函数的data，而这个data是什么呢，就是上面ngx_start_cache_manager_processes函数中传递给spawn 的第三个参数，这里由于是两个子进程，所以有两个结构：

```
  
static ngx_cache_manager_ctx_t ngx_cache_manager_ctx = {
      
ngx_cache_manager_process_handler, "cache manager process", 0
  
};

static ngx_cache_manager_ctx_t ngx_cache_loader_ctx = {
      
ngx_cache_loader_process_handler, "cache loader process", 60000
  
};
  
```

也就是说manger 和 loader的定时器会分别调用ngx_cache_manager_process_handler和ngx_cache_loader_process_handler，不过可以看到manager的定时器初始时间是0，而loader是60000毫秒。

在看这两个handler之前，先看一个结构就是ngx_http_file_cache_node_t，一个cache文件对应一个node，这个node中主要保存了cache 的key和uniq， uniq主要是关联文件，而key是用于红黑树。

```
  
typedef struct {
  
//红黑树和queue结构
      
ngx_rbtree_node_t node;
      
ngx_queue_t queue;
  
//cache key
      
u_char key[NGX_HTTP_CACHE_KEY_LEN
                                           
&#8211; sizeof(ngx_rbtree_key_t)];
  
//引用计数
      
unsigned count:20;
  
//多少请求在使用
      
unsigned uses:10;
      
unsigned valid_msec:10;
  
//cache的状态
      
unsigned error:10;
  
//是否存在对应的cache文件
      
unsigned exists:1;
  
//是否正在更新
      
unsigned updating:1;
  
//是否正在删除
      
unsigned deleting:1;
                                       
/\* 11 unused bits \*/
  
//文件的uniq
      
ngx_file_uniq_t uniq;
  
//cache失效时间
      
time_t expire;、
  
//比如cache control中的max-age。
      
time_t valid_sec;
  
//其实应该是body大小
      
size_t body_start;
  
//文件大小
      
off_t fs_size;
  
} ngx_http_file_cache_node_t;
  
```

然后先来看manager的handler。

```
  
static void
  
ngx_cache_manager_process_handler(ngx_event_t *ev)
  
{
      
time_t next, n;
      
ngx_uint_t i;
      
ngx_path_t **path;

next = 60 * 60;

path = ngx_cycle->pathes.elts;
  
//遍历所有的cache目录
      
for (i = 0; i < ngx_cycle->pathes.nelts; i++) {

if (path[i]->manager) {
  
//调用manger回调
              
n = path[i]->manager(path[i]->data);
  
//取得下一次的定时器的时间，可以看到是取n和next的最小值
              
next = (n <= next) ? n : next;

ngx_time_update();
          
}
      
}

if (next == 0) {
          
next = 1;
      
}

ngx_add_timer(ev, next * 1000);
  
}
  
```

上面最核心的就是path[i]->manager,而这个回调是在一开始介绍的ngx_http_file_cache_set_slot中设置的，它设置manager回调为ngx_http_file_cache_manager，我们就来看这个函数：

```
  
static time_t
  
ngx_http_file_cache_manager(void *data)
  
{
      
ngx_http_file_cache_t *cache = data;

off_t size;
      
time_t next, wait;
  
//如果有超时的cache，就超时对应的cache
      
next = ngx_http_file_cache_expire(cache);
  
//最后访问时间
      
cache->last = ngx_current_msec;
      
cache->files = 0;

for ( ;; ) {
          
ngx_shmtx_lock(&cache->shpool->mutex);

size = cache->sh->size;

ngx_shmtx_unlock(&cache->shpool->mutex);

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ngx_cycle->log, 0,
                         
"http file cache size: %O", size);
  
//如果没有超过最大的大小限制，则直接返回。
          
if (size < cache->max_size) {
              
return next;
          
}
  
//否则遍历所有的cache文件，然后删除过期的cache.
          
wait = ngx_http_file_cache_forced_expire(cache);

if (wait > 0) {
              
return wait;
          
}

if (ngx_quit || ngx_terminate) {
              
return next;
          
}
      
}
  
}
  
```

上面的函数中主要调用了两个函数，一个是ngx_http_file_cache_expire,一个是ngx_http_file_cache_forced_expire，他们有什么区别呢，主要区别是这样子，前一个只有过期的cache才会去尝试删除它(引用计数为0),而后一个不管有没有过期，只要引用计数为0，就会去清理。来详细看这两个函数的实现。

首先是ngx_http_file_cache_expire，这里注意nginx使用了LRU，也就是队列最尾端保存的是最长时间没有被使用的,并且这个函数返回的就是一个wait值，这个值的计算不知道为什么nginx会设置为10ms，我觉得这个值设置为可调或许更好。

```
  
static time_t
  
ngx_http_file_cache_expire(ngx_http_file_cache_t *cache)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..

now = ngx_time();

ngx_shmtx_lock(&cache->shpool->mutex);

for ( ;; ) {
  
//如果cache队列为空，则直接退出返回
          
if (ngx_queue_empty(&cache->sh->queue)) {
              
wait = 10;
              
break;
          
}
  
//从最后一个开始
          
q = ngx_queue_last(&cache->sh->queue);

fcn = ngx_queue_data(q, ngx_http_file_cache_node_t, queue);

wait = fcn->expire &#8211; now;
  
//如果没有超时，则退出循环
          
if (wait > 0) {
              
wait = wait > 10 ? 10 : wait;
              
break;
          
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
//如果引用计数为0，则删除这个cache节点
          
if (fcn->count == 0) {
              
ngx_http_file_cache_delete(cache, q, name);
              
continue;
          
}
  
//如果当前节点正在删除，则退出循环
          
if (fcn->deleting) {
              
wait = 1;
              
break;
          
}

p = ngx_hex_dump(key, (u_char *) &fcn->node.key,
                           
sizeof(ngx_rbtree_key_t));
          
len = NGX_HTTP_CACHE_KEY_LEN &#8211; sizeof(ngx_rbtree_key_t);
          
(void) ngx_hex_dump(p, fcn->key, len);

/*
           
* abnormally exited workers may leave locked cache entries,
           
* and although it may be safe to remove them completely,
           
* we prefer to just move them to the top of the inactive queue
           
*/
  
//将当前节点放入队列最前端
          
ngx_queue_remove(q);
          
fcn->expire = ngx_time() + cache->inactive;
          
ngx_queue_insert_head(&cache->sh->queue, &fcn->queue);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
      
}

ngx_shmtx_unlock(&cache->shpool->mutex);

ngx_free(name);

return wait;
  
}
  
```

然后是ngx_http_file_cache_forced_expire，顾名思义，就是强制删除cache 节点，它的返回值也是wait time。它的遍历也是从后到前的。

```
  
static time_t
  
ngx_http_file_cache_forced_expire(ngx_http_file_cache_t *cache)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.

path = cache->path;
      
len = path->name.len + 1 + path->len + 2 * NGX_HTTP_CACHE_KEY_LEN;

name = ngx_alloc(len + 1, ngx_cycle->log);
      
if (name == NULL) {
          
return 10;
      
}

ngx_memcpy(name, path->name.data, path->name.len);

wait = 10;
  
//删除节点尝试次数
      
tries = 20;

ngx_shmtx_lock(&cache->shpool->mutex);
  
//遍历队列
      
for (q = ngx_queue_last(&cache->sh->queue);
           
q != ngx_queue_sentinel(&cache->sh->queue);
           
q = ngx_queue_prev(q))
      
{
          
fcn = ngx_queue_data(q, ngx_http_file_cache_node_t, queue);

&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//如果引用计数为0则删除cache
          
if (fcn->count == 0) {
              
ngx_http_file_cache_delete(cache, q, name);
              
wait = 0;

} else {
  
//否则尝试20次
              
if (&#8211;tries) {
                  
continue;
              
}

wait = 1;
          
}

break;
      
}

ngx_shmtx_unlock(&cache->shpool->mutex);

ngx_free(name);

return wait;
  
}
  
```

上面的分析中还有一个函数就是ngx_http_file_cache_delete，这个函数这里就不分析了，它主要有2个功能，一个是删除cache文件，一个是删除cache管理节点。

然后来看loader的handle，ngx_http_file_cache_loader这个函数，

```
  
static void
  
ngx_http_file_cache_loader(void *data)
  
{
      
ngx_http_file_cache_t *cache = data;

ngx_tree_ctx_t tree;

if (!cache->sh->cold || cache->sh->loading) {
          
return;
      
}

if (!ngx_atomic_cmp_set(&cache->sh->loading, 0, ngx_pid)) {
          
return;
      
}

&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
//设置回调，后续会介绍这几个回调的含义
      
tree.init_handler = NULL;
      
tree.file_handler = ngx_http_file_cache_manage_file;
      
tree.pre_tree_handler = ngx_http_file_cache_noop;
      
tree.post_tree_handler = ngx_http_file_cache_noop;
      
tree.spec_handler = ngx_http_file_cache_delete_file;
  
//回调数据就是cache
      
tree.data = cache;
      
tree.alloc = 0;
      
tree.log = ngx_cycle->log;
  
//last为最后load时间
      
cache->last = ngx_current_msec;
      
cache->files = 0;
  
//开始遍历
      
if (ngx_walk_tree(&tree, &cache->path->name) == NGX_ABORT) {
          
cache->sh->loading = 0;
          
return;
      
}

cache->sh->cold = 0;
      
cache->sh->loading = 0;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
  
}
  
```

上面有一个很核心的数据结构，就是ngx_tree_ctx_t，它的结构在nginx里面有注释，我们可以看下注释：

> * ctx->init_handler() &#8211; see ctx->alloc
   
> * ctx->file_handler() &#8211; file handler
   
> * ctx->pre_tree_handler() &#8211; handler is called before entering directory
   
> * ctx->post_tree_handler() &#8211; handler is called after leaving directory
   
> * ctx->spec_handler() &#8211; special (socket, FIFO, etc.) file handler
   
> *
   
> * ctx->data &#8211; some data structure, it may be the same on all levels, or
   
> * reallocated if ctx->alloc is nonzero
   
> *
   
> * ctx->alloc &#8211; a size of data structure that is allocated at every level
   
> * and is initilialized by ctx->init_handler()
   
> *
   
> * ctx->log &#8211; a log 

不过这里要注意在nginx的当前cache实现中，只有file_handler和spec_handler被设置了，其他的都是空，因此我们着重来看ngx_http_file_cache_manage_file，这里ngx_walk_tree就不分析了，这个函数主要是遍历所有的cache目录，然后对于每一个cache文件调用file_handler回调。

```
  
static ngx_int_t
  
ngx_http_file_cache_manage_file(ngx_tree_ctx_t \*ctx, ngx_str_t \*path)
  
{
      
ngx_msec_t elapsed;
      
ngx_http_file_cache_t *cache;

cache = ctx->data;
  
//将文件添加进cache
      
if (ngx_http_file_cache_add_file(ctx, path) != NGX_OK) {
          
(void) ngx_http_file_cache_delete_file(ctx, path);
      
}
  
//如果文件个数太大，则休眠并清理files计数
      
if (++cache->files >= cache->loader_files) {
          
ngx_http_file_cache_loader_sleep(cache);

} else {
          
ngx_time_update();
  
//否则看loader时间是不是过长，如果过长则又进入休眠
          
elapsed = ngx_abs((ngx_msec_int_t) (ngx_current_msec &#8211; cache->last));

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ngx_cycle->log, 0,
                         
"http file cache loader time elapsed: %M", elapsed);

if (elapsed >= cache->loader_threshold) {
              
ngx_http_file_cache_loader_sleep(cache);
          
}
      
}

return (ngx_quit || ngx_terminate) ? NGX_ABORT : NGX_OK;
  
}
  
```

上面有两个函数没有分析，分别是ngx_http_file_cache_add_file和ngx_http_file_cache_loader_sleep。其中sleep函数比较简单，就是休眠,并重置对应的域：
  
```
  
static void
  
ngx_http_file_cache_loader_sleep(ngx_http_file_cache_t *cache)
  
{
      
ngx_msleep(cache->loader_sleep);

ngx_time_update();

cache->last = ngx_current_msec;
      
cache->files = 0;
  
}
  
```

然后来看ngx_http_file_cache_add_file,它主要是通过文件名计算hash，然后调用ngx_http_file_cache_add将这个文件加入到cache管理中(也就是添加红黑树以及队列),因此我们就主要来看ngx_http_file_cache_add。

```
  
static ngx_int_t
  
ngx_http_file_cache_add(ngx_http_file_cache_t \*cache, ngx_http_cache_t \*c)
  
{
      
ngx_http_file_cache_node_t *fcn;

ngx_shmtx_lock(&cache->shpool->mutex);
  
//首先查找
      
fcn = ngx_http_file_cache_lookup(cache, c->key);

if (fcn == NULL) {
  
//如果不存在，则新建结构
          
fcn = ngx_slab_alloc_locked(cache->shpool,
                                      
sizeof(ngx_http_file_cache_node_t));
          
if (fcn == NULL) {
              
ngx_shmtx_unlock(&cache->shpool->mutex);
              
return NGX_ERROR;
          
}

ngx_memcpy((u_char *) &fcn->node.key, c->key, sizeof(ngx_rbtree_key_t));

ngx_memcpy(fcn->key, &c->key[sizeof(ngx_rbtree_key_t)],
                     
NGX_HTTP_CACHE_KEY_LEN &#8211; sizeof(ngx_rbtree_key_t));
  
//插入红黑树
          
ngx_rbtree_insert(&cache->sh->rbtree, &fcn->node);

fcn->uses = 1;
          
fcn->count = 0;
          
fcn->valid_msec = 0;
          
fcn->error = 0;
          
fcn->exists = 1;
          
fcn->updating = 0;
          
fcn->deleting = 0;
          
fcn->uniq = 0;
          
fcn->valid_sec = 0;
          
fcn->body_start = 0;
          
fcn->fs_size = c->fs_size;

cache->sh->size += c->fs_size;

} else {
  
//否则删除queue，后续会重新插入
          
ngx_queue_remove(&fcn->queue);
      
}

fcn->expire = ngx_time() + cache->inactive;
  
//重新插入
      
ngx_queue_insert_head(&cache->sh->queue, &fcn->queue);

ngx_shmtx_unlock(&cache->shpool->mutex);

return NGX_OK;
  
}
  
```

下一篇我将会介绍在upstream中如何使用cache的。