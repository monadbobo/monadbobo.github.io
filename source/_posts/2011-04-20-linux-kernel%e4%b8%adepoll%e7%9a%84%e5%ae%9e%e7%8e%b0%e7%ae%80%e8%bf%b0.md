---
id: 264
title: linux kernel中epoll的设计和实现
author: diaoliang
layout: post
guid: 'http://www.pagefault.info/?p=264'
permalink: >-
  /2011/04/20/linux-kernel%e4%b8%adepoll%e7%9a%84%e5%ae%9e%e7%8e%b0%e7%ae%80%e8%bf%b0/
categories:
  - kernel
  - server
tags:
  - epoll
  - kernel
translate_title: design-and-implementation-of-epoll-in-linux-kernel
date: 2011-04-20 14:00:58
---
这里就不贴源码了，源码分析的话，网上一大堆，我这里只是简要的描述下epoll的实现和一些关键的代码片段。

相关的文件在 fs/eventpoll.c中,我看的是2.6.38的内核代码.

1 epoll在创建的时候会调用anon_inode_getfd新建一个file instance，也就是epoll可以看成一个文件。因此我们可以看到epoll_create会返回一个fd.
  
```
	  
error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep,
				   
O_RDWR | (flags & O_CLOEXEC));
  
```

2 epoll所管理的所有的句柄都是放在一个大的结构eventpoll(红黑树)中,而这个结构是保存在file 的private_data域中的(因为epoll本身就是一个文件).这样每次通过epoll fd就可以直接得到eventpoll.
  
```
	  
file = fget(epfd);
	  
/\* Get the "struct file \*" for the target file */
	  
tfile = fget(fd);
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
	  
ep = file->private_data;
  
```

3 每一个加入到epoll监听的句柄(也就是红黑树的一个节点)都是一个epitem.它包含了一个 struct eventpoll *ep，也就是它所属于的eventpoll(epoll实例).
  
```
	  
if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
		  
return -ENOMEM;

/\* Item initialization follow here &#8230; \*/
	  
INIT_LIST_HEAD(&epi->rdllink);
	  
INIT_LIST_HEAD(&epi->fllink);
	  
INIT_LIST_HEAD(&epi->pwqlist);
	  
epi->ep = ep;
	  
ep_set_ffd(&epi->ffd, tfile, fd);
	  
epi->event = *event;
	  
epi->nwait = 0;
	  
epi->next = EP_UNACTIVE_PTR;
  
```
  
<!--more-->


  
4 在eventpoll中包含两个wait queue，一个是被epoll_wait使用的(wq)，一个是被file->poll使用的(poll_wait).这两个都是属于eventpoll.而在epitem也有一个wait queue(pwqlist)，这个queue是fd私有的wait queue，所以它是保存在epitem中的。

```
  
struct epitem {
	  
/\* RB tree node used to link this structure to the eventpoll RB tree \*/
	  
struct rb_node rbn;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
	  
/\* List containing poll wait queues \*/
	  
struct list_head pwqlist;

/\* The "container" of this item \*/
	  
struct eventpoll *ep;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
};

struct eventpoll {
	  
/\* Protect the this structure access \*/
	  
spinlock_t lock;
	  
/*
	   
* This mutex is used to ensure that files are not removed
	   
* while epoll is using them. This is held during the event
	   
* collection loop, the file cleanup path, the epoll file exit
	   
* code and the ctl operations.
	   
*/
	  
struct mutex mtx;

/\* Wait queue used by sys_epoll_wait() \*/
	  
wait_queue_head_t wq;

/\* Wait queue used by file->poll() \*/
	  
wait_queue_head_t poll_wait;

/\* List of ready file descriptors \*/
	  
struct list_head rdllist;

/\* RB tree root used to store monitored fd structs \*/
	  
struct rb_root rbr;

/*
	   
* This is a single linked list that chains all the "struct epitem" that
	   
* happened while transfering ready events to userspace w/out
	   
* holding ->lock.
	   
*/
	  
struct epitem *ovflist;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
};
  
```

5 当我们添加一个句柄到一个epoll fd的时候，默认是会包含POLLERR和POLLHUP事件.并将epitem插入到红黑树中(eventpoll).然后会初始化一个poll_table,然后设置它的回调函数为ep_poll_callback，紧接着调用file->poll,如果是socket，则会调用tcp_poll,这个函数将会调用ep_poll_callback.
  
```
	  
switch (op) {
	  
case EPOLL_CTL_ADD:
		  
if (!epi) {
			  
epds.events |= POLLERR | POLLHUP;
			  
error = ep_insert(ep, &epds, tfile, fd);
		  
} else
			  
error = -EEXIST;
		  
break;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
```

6 ep_poll_callback这个函数主要是绑定epitem的wait queue的回调为ep_poll_callback.也就是对应fd如果有事件，则就会调用ep_poll_callback。

7 epoll中保存了一个read list(rdllist)，所有的已经有通知事件的句柄，都会放到这个list中，而对应的操作就是在ep_poll_callback中，在ep_poll_callback中主要就是由wait queue指针来取得对应的epitem，然后再取得eventpoll，并将这个epitem加入到ready list，唤醒epoll_wait(wq).这里可以看到由于一个句柄只会对应一个epitem，所以在rdllist中，也不会有重复的epitem，在ep_poll_callback会判断是否rdllist中是否已经包含了将要插入的epitem，如果包含，则直接返回.

```
  
static int ep_poll_callback(wait_queue_t \*wait, unsigned mode, int sync, void \*key)
  
{
	  
struct epitem *epi = ep_item_from_wait(wait);
	  
struct eventpoll *ep = epi->ep;
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
	  
/\* If this file is already in the ready list we exit soon \*/
	  
if (!ep_is_linked(&epi->rdllink))
		  
list_add_tail(&epi->rdllink, &ep->rdllist);
  
```

8 当系统调用epoll_wait,如果ready list(rdllist)为空，则休眠等待被唤醒。当被唤醒之后(第7条),将rdllist复制(指针)到一个新的list(主要是针对LT),然后调用ep_send_events_proc对这个新的list进行遍历(对应的会从rdllist中删除这个epitem).遍历完毕后，最终会返回给用户对应的数据.
  
```
	  
if (list_empty(&ep->rdllist)) {
		  
/*
		   
* We don&#8217;t have any available event to return to the caller.
		   
* We need to sleep here, and we will be wake up by
		   
* ep_poll_callback() when events will become available.
		   
*/
		  
init_waitqueue_entry(&wait, current);
		  
__add_wait_queue_exclusive(&ep->wq, &wait);

for (;;) {
			  
/*
			   
* We don&#8217;t want to sleep if the ep_poll_callback() sends us
			   
* a wakeup in between. That&#8217;s why we set the task state
			   
* to TASK_INTERRUPTIBLE before doing the checks.
			   
*/
			  
set_current_state(TASK_INTERRUPTIBLE);
			  
if (!list_empty(&ep->rdllist) || timed_out)
				  
break;
			  
if (signal_pending(current)) {
				  
res = -EINTR;
				  
break;
			  
}

spin_unlock_irqrestore(&ep->lock, flags);
			  
if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				  
timed_out = 1;

spin_lock_irqsave(&ep->lock, flags);
		  
}
		  
__remove_wait_queue(&ep->wq, &wait);

set_current_state(TASK_RUNNING);
	  
}
  
```

9 LT和ET的区别是在ep_send_events_proc中处理的，如果是LT，不但会将对应的数据返回给用户，并且会将当前的epitem再次加入到rdllist中。这样子，如果下次再次被唤醒就会给用户空间再次返回事件.
  
```
  
static int ep_send_events_proc(struct eventpoll \*ep, struct list_head \*head,
			         
void *priv)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
			  
if (__put_user(revents, &uevent->events) ||
			      
__put_user(epi->event.data, &uevent->data)) {
				  
list_add(&epi->rdllink, head);
				  
return eventcnt ? eventcnt : -EFAULT;
			  
}
			  
eventcnt++;
			  
uevent++;
			  
if (epi->event.events & EPOLLONESHOT)
				  
epi->event.events &= EP_PRIVATE_BITS;
			  
else if (!(epi->event.events & EPOLLET)) {
				  
/*
				   
* If this file has been added with Level
				   
* Trigger mode, we need to insert back inside
				   
* the ready list, so that the next call to
				   
* epoll_wait() will check again the events
				   
* availability. At this point, noone can insert
				   
* into ep->rdllist besides us. The epoll_ctl()
				   
* callers are locked out by
				   
* ep_scan_ready_list() holding "mtx" and the
				   
* poll callback will queue them in ep->ovflist.
				   
*/
				  
list_add_tail(&epi->rdllink, &ep->rdllist);
			  
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
}
  
```

10 eventpol还有一个list，叫做ovflist，主要是解决当内核在传输数据给用户空间(ep_send_events_proc)时的锁(eventpoll->mtx)，此时epoll就是将这个时候传递上来的事件保存到ovflist中。
  
```
  
static int ep_poll_callback(wait_queue_t \*wait, unsigned mode, int sync, void \*key)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;..
	  
/*
	   
* If we are trasfering events to userspace, we can hold no locks
	   
* (because we&#8217;re accessing user memory, and because of linux f_op->poll()
	   
* semantics). All the events that happens during that period of time are
	   
* chained in ep->ovflist and requeued later on.
	   
*/
	  
if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
		  
if (epi->next == EP_UNACTIVE_PTR) {
			  
epi->next = ep->ovflist;
			  
ep->ovflist = epi;
		  
}
		  
goto out_unlock;
	  
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
  
}
  
```

11 当从ep_send_events_proc返回(ep_scan_ready_list)后，会遍历ovflist，然后将ready的epitem保存到rdllist,以便与下次再次被唤醒时进行操作.
  
```
  
static int ep_scan_ready_list(struct eventpoll *ep,
			        
int (\*sproc)(struct eventpoll \*,
					     
struct list_head \*, void \*),
			        
void *priv)
  
{
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;.
	  
/*
	   
* During the time we spent inside the "sproc" callback, some
	   
* other events might have been queued by the poll callback.
	   
* We re-insert them inside the main ready-list here.
	   
*/
	  
for (nepi = ep->ovflist; (epi = nepi) != NULL;
	       
nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
		  
/*
		   
* We need to check if the item is already in the list.
		   
* During the "sproc" callback execution time, items are
		   
* queued into ->ovflist but the "txlist" might already
		   
* contain them, and the list_splice() below takes care of them.
		   
*/
		  
if (!ep_is_linked(&epi->rdllink))
			  
list_add_tail(&epi->rdllink, &ep->rdllist);
	  
}
  
&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;&#8230;
  
}
  
```