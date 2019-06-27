---
title: MTR(mini-transaction)设计与实现
author: Simon Liu
tags:
  - InnoDB
translate_title: mtr-minitransaction-design-and-implementation
---

# MTR(mini-transaction)设计与实现
## 简介

首先来看MTR的定义:

> An internal phase of InnoDB processing, when making changes at the**physical**level to internal data structures during**DML**operations. A mini-transaction (mtr) has no notion of**rollback**; multiple mini-transactions can occur within a single**transaction**. Mini-transactions write information to the**redo log**that is used during**crash recovery**. A mini-transaction can also happen outside the context of a regular transaction, for example during**purge**processing by background threads.  
>   

MTR主要的目的是为了保证数据的一致性(比如多个事务或者发生数据库异常时). 因此MTR一般来说都伴随着写Redo log,因为redolog就是为了在recover的时候能够正常恢复数据.

一般来说在一个MTR中会做两个事情.

* 写redolog
* 挂载脏页到flush list.
* 
## 源码分析

### MTR的接口

一般使用逻辑如下:

> Mtr start  
> Process something  
> Mtr commit  

来看个例子，比如在btree中打印目录以及btree info.

``` cpp
  mtr_start(&mtr);

  root = btr_root_block_get(index, RW_SX_LATCH, &mtr);

  btr_print_recursive(index, root, width, &heap, &offsets, &mtr);
  if (heap) {
    mem_heap_free(heap);
  }

  mtr_commit(&mtr);

```

可以看到使用比较简单，和我们上面的描述一致。

因此我们来看对应的这两个接口. 这里要注意mtr_start不仅有正常的接口，还有一个syn和async的接口，mtr_start就是mtr_start_sync.也就是默认的mtr是同步的.而async的mtr只能做只读操作.

``` cpp
/** Start a mini-transaction. */
#define mtr_start(*m*) (m)->start()

*/** Start a synchronous mini-transaction */*
#define mtr_start_sync(*m*) (m)->start(true)

*/** Start an asynchronous read-only mini-transaction */*
#define mtr_start_ro(*m*) (m)->start(true, true)

*/** Commit a mini-transaction. */*
#define mtr_commit(*m*) (m)->commit()

```

这里的m就是 struct mtr_t ,这个结构就是mini-transaction在InnoDB中的抽象.

### 数据结构

接下来来分析mtr_t这个结构, 这个结构有一个内部的数据结构用来保存MTR的一些状态.

``` cpp
struct Impl {
    */** memo stack for locks etc. */*
    mtr_buf_t m_memo;

    */** mini-transaction log */*
    mtr_buf_t m_log;

    */** true if mtr has made at least one buffer pool page dirty */*
    bool m_made_dirty;

    */** true if inside ibuf changes */*
    bool m_inside_ibuf;

    */** true if the mini-transaction modified buffer pool pages */*
    bool m_modifications;

    */** Count of how many page initial log records have been*
*written to the mtr log */*
    ib_uint32_t m_n_log_recs;

    */** specifies which operations should be logged; default*
*value MTR_LOG_ALL */*
    mtr_log_t m_log_mode;

    */** State of the transaction */*
    mtr_state_t m_state;

    */** Flush Observer */*
    FlushObserver *m_flush_observer;

#ifdef UNIV_DEBUG
    */** For checking corruption. */*
    ulint m_magic_n;
#endif */* UNIV_DEBUG */*

    */** Owning mini-transaction */*
    mtr_t *m_mtr;
  };

```

::注释写的比较简略，这里我们来详细看几个比较重要的字段::

首先是m_state,这个表示当前MTR的状态，主要有3个状态，分别是激活，提交中以及提交完毕.
```cpp
enum mtr_state_t {
  MTR_STATE_INIT = 0,
  MTR_STATE_ACTIVE = 12231,
  MTR_STATE_COMMITTING = 56456,
  MTR_STATE_COMMITTED = 34676
};
```

然后是 m_log_mode,它主要是表示当前所需要记录的操作类型，可以看到一共分为4种操作类型.
1. MTR_LOG_ALL 表示LOG所有的操作（包括写redolog以及加脏页到flush list)
2. MTR_LOG_NONE  不记录任何操作.
3. MTR_LOG_NO_REDO 不生成REDO log,可是会加脏页到flush list
4. MTR_LOG_SHORT_INSERTS  这个也是不记录任何操作，纯粹只是使用了MTR的一些功能(只在copy page的使用).

``` cpp
*/** Logging modes for a mini-transaction */*
enum mtr_log_t {
  */** Default mode: log all operations modifying disk-based data */*
  MTR_LOG_ALL = 21,

  */** Log no operations and dirty pages are not added to the flush list */*
  MTR_LOG_NONE = 22,

  */** Don't generate REDO log but add dirty pages to flush list */*
  MTR_LOG_NO_REDO = 23,

  */** Inserts are logged in a shorter form */*
  MTR_LOG_SHORT_INSERTS = 24
};

```

m_log 则是当前的MTR提交的log内容,后面我们回来分析m_log的格式.

然后我们来看mtr_t这个结构里面仅有的几个字段. 可以看到m_impl就是上面我们介绍的Impl,而m_commit_lsn表示在commit的时候(commit)的lsn, 这里还有一个比较关键的数据结构，那就是Command,这个数据结构主要是抽象了MTR的具体操作。也就是说对于Redo log的修改其实是在Command这个结构中执行的.

```cpp
 private:
  Impl m_impl;

  */** LSN at commit time */*
  lsn_t m_commit_lsn;

  */** true if it is synchronous mini-transaction */*
  bool m_sync;

  class Command;

  friend class Command;
```


### mtr_t::start
接下来我们来看MTR的启动函数mtr_t::start.这个函数包含两个参数，第一个sync表示是否当前的mtr是同步，第二个是read_only,这个表示当前mtr 是否只读.默认情况下sync=true, read_only=false. 

这里的初始化可以看到state会被初始化为MTR_STATE_ACTIVE,其他的参数都是初始化为默认值.

``` cpp
void mtr_t::start(bool sync, bool read_only) {
  UNIV_MEM_INVALID(this, sizeof(*this));

  UNIV_MEM_INVALID(&m_impl, sizeof(m_impl));

  m_sync = sync;

  m_commit_lsn = 0;

  new (&m_impl.m_log) mtr_buf_t();
  new (&m_impl.m_memo) mtr_buf_t();

  m_impl.m_mtr = this;
  m_impl.m_log_mode = MTR_LOG_ALL;
  m_impl.m_inside_ibuf = false;
  m_impl.m_modifications = false;
  m_impl.m_made_dirty = false;
  m_impl.m_n_log_recs = 0;
  m_impl.m_state = MTR_STATE_ACTIVE;
  m_impl.m_flush_observer = NULL;

  ut_d(m_impl.m_magic_n = MTR_MAGIC_N);
}

```

Start完毕之后，就是提交修改了(commit).

### mtr_t::commit

1. 首先会设置m_state为COMMITTING状态
2. 然后进行判断是否需要执行所做的修改.

这里可以看到只要满足下面两个条件之一就会去执行MTR.
1. m_n_log_recs 大于0 也就是将要写入到mar log的页的个数.
2. 当前mtr修改buffer pool pages并且不生成redolog操作.

``` cpp
*/** Commit a mini-transaction. */*
void mtr_t::commit() {
  ut_ad(is_active());
  ut_ad(!is_inside_ibuf());
  ut_ad(m_impl.m_magic_n == MTR_MAGIC_N);
  m_impl.m_state = MTR_STATE_COMMITTING;

  Command cmd(this);

  if (m_impl.m_n_log_recs > 0 ||
      (m_impl.m_modifications && m_impl.m_log_mode == MTR_LOG_NO_REDO)) {
    ut_ad(!srv_read_only_mode || m_impl.m_log_mode == MTR_LOG_NO_REDO);

    cmd.execute();
  } else {
    cmd.release_all();
    cmd.release_resources();
  }
}

```

### Command::execute

因此我们来看最终的执行方法 execute 

``` cpp
void mtr_t::Command::execute() {
  ut_ad(m_impl->m_log_mode != MTR_LOG_NONE);

  ulint len;

#ifndef UNIV_HOTBACKUP
  len = prepare_write();

  if (len > 0) {
    mtr_write_log_t write_log;

    write_log.m_left_to_write = len;

    auto handle = log_buffer_reserve(*log_sys, len);

    write_log.m_handle = handle;
    write_log.m_lsn = handle.start_lsn;
    write_log.m_rec_group_start_lsn = handle.start_lsn;

    m_impl->m_log.for_each_block(write_log);

    ut_ad(write_log.m_left_to_write == 0);
    ut_ad(write_log.m_lsn == handle.end_lsn);

    log_wait_for_space_in_log_recent_closed(*log_sys, handle.start_lsn);

    DEBUG_SYNC_C("mtr_redo_before_add_dirty_blocks");

    add_dirty_blocks_to_flush_list(handle.start_lsn, handle.end_lsn);

    log_buffer_close(*log_sys, handle);

    m_impl->m_mtr->m_commit_lsn = handle.end_lsn;

  } else {
    DEBUG_SYNC_C("mtr_noredo_before_add_dirty_blocks");

    add_dirty_blocks_to_flush_list(0, 0);
  }
#endif */* !UNIV_HOTBACKUP */*

  release_all();
  release_resources();
}


```

函数不长，我们分段来看。

首先在excecute中会先进行写之前的操作，主要是进行一些校验以及最终返回将要写入的redolog长度,这个函数就是(prepare_write).

#### prepare_write

先来看几个变量.
1. m_log.size() 这个返回当前m_log buffer的字节长度.
2. m_n_log_recs 这个表示当前mtr将要写入的页的个数.

因此prepare_write这个函数就是根据m_n_log_recs来判断是否是多个record，从而来设置不同的标记.
* 如果是单个record(n_recs == 1), 则设置到m_log的最高位为1.
* 如果是多个record(n_recs > 1),则多写一个字节到record的最末.

最终只有多个redord才会更改len,不然默认就是m_log.size().

``` cpp
ulint mtr_t::Command::prepare_write() {
...............................................
  */* An ibuf merge could happen when loading page to apply log*
*records during recovery. During the ibuf merge mtr is used. */*

  ut_a(!recv_recovery_is_on() || !recv_no_ibuf_operations);

  ulint len = m_impl->m_log.size();
  ut_ad(len > 0);

  ulint n_recs = m_impl->m_n_log_recs;
  ut_ad(n_recs > 0);

  ut_ad(log_sys != nullptr);

  ut_ad(m_impl->m_n_log_recs == n_recs);

  */* This was not the first time of dirtying a*
*tablespace since the latest checkpoint. */*

  ut_ad(n_recs == m_impl->m_n_log_recs);

  if (n_recs <= 1) {
    ut_ad(n_recs == 1);

    */* Flag the single log record as the*
*only record in this mini-transaction. */*

    *m_impl->m_log.front()->begin() |= MLOG_SINGLE_REC_FLAG;

  } else {
    */* Because this mini-transaction comprises*
*multiple log records, append MLOG_MULTI_REC_END*
*at the end. */*

    mlog_catenate_ulint(&m_impl->m_log, MLOG_MULTI_REC_END, MLOG_1BYTE);
    ++len;
  }

  ut_ad(m_impl->m_log_mode == MTR_LOG_ALL);
  ut_ad(m_impl->m_log.size() == len);
  ut_ad(len > 0);

  return (len);
}
```

计算完毕之后，我们就进入真正的执行阶段了(Command. execute)，这里流程是这样子的:
1. 首先需要构造一个mtr_write_log_t 结构.
 `  mtr_write_log_t write_log; `
	1. 这个结构主要是将mtr中的内容写入到redo log 中.这里先来看他的字段.
	2. m_handle 这个主要是用来保存从redolog得到的一些字段
		1. lock_no 表示shared锁
		2. start_lsn表示当前mtr的起始lsn
		3. end_lsn表示当前mtr的结束lsn.
```cpp
typedef size_t log_lock_no_t;

struct Log_handle {
  log_lock_no_t lock_no;

  lsn_t start_lsn;

  lsn_t end_lsn;
};
```
	2. m_lsn 起始lsn
	3. m_rec_group_start_lsn ?
	4. m_left_to_write 表示还需要写入到redolog的内容的长度，因此这个值默认就是len.

``` cpp
struct mtr_write_log_t {
..................................
  Log_handle m_handle;
  lsn_t m_lsn;
  lsn_t m_rec_group_start_lsn;
  ulint m_left_to_write;
};

```

2. 分配log buf以及初始化write_log.

``` cpp
void mtr_t::Command::execute() {
..........................
  if (len > 0) {
    mtr_write_log_t write_log;

    write_log.m_left_to_write = len;

    auto handle = log_buffer_reserve(*log_sys, len);

    write_log.m_handle = handle;
    write_log.m_lsn = handle.start_lsn;
    write_log.m_rec_group_start_lsn = handle.start_lsn;
.........................................
}

```


#### log_buffer_reserve

在执行mtr的时候会首先调用log_buffer_reserve在redolog中分配对应的buf长度. 这个函数主要是计算sn以及lsn ,而sn和lsn的区别是在于数据写入到redolog的时候，redolog是按照block来写的，而每一个block都会有header和footer，因此这里sn是写入者看到的lsn，而lsn则是在磁盘上的真正的lsn.
下面就是代码:

``` cpp
Log_handle log_buffer_reserve(log_t &log, size_t len) {
  Log_handle handle;

  handle.lock_no = log_buffer_s_lock_enter(log);
......................................
  srv_stats.log_write_requests.inc();

  ut_a(srv_shutdown_state <= SRV_SHUTDOWN_FLUSH_PHASE);
  ut_a(len > 0);

  */* Reserve space in sequence of data bytes: */*
  const sn_t start_sn = log.sn.fetch_add(len);

  */* Ensure that redo log has been initialized properly. */*
  ut_a(start_sn > 0);
......................................

  */* Headers in redo blocks are not calculated to sn values: */*
  const sn_t end_sn = start_sn + len;
...........................................
  */* Translate sn to lsn (which includes also headers in redo blocks): */*
  handle.start_lsn = log_translate_sn_to_lsn(start_sn);
  handle.end_lsn = log_translate_sn_to_lsn(end_sn);
...................................................

  return (handle);
}

```

3. 然后就是从mtr的buffer中写入内容到redolog的buffer.

``` cpp
void mtr_t::Command::execute() {
.....................
m_impl->m_log.for_each_block(write_log);
.......................
}

  template <typename Functor>
  bool for_each_block(Functor &functor) const {
    for (const block_t *block = UT_LIST_GET_FIRST(m_list); block != NULL;
         block = UT_LIST_GET_NEXT(m_node, block)) {
      if (!functor(block)) {
        return (false);
      }
    }

    return (true);
  }

```
 从上面的代码我们可以看到写入最红柿会遍历m_log的buf，然后再调用write_log类的方法来真正写入block.因此我们需要再次回到mtr_write_log_t这个结构.

这里是通过重载()来实现函数调用的，参数block表示将要写入redolog的内容.这里看到一个循环勒啊些股

1. 首先是调用 log_buffer_write来写入block到redolog.
2. 更新m_left_to_write.
3. 如果内容写完则 log_buffer_set_first_record_group ？
4. 调用 log_buffer_write_completed完成buffer的写入
5. 更新m_lsn,以便于下次使用.

``` cpp
struct mtr_write_log_t {
  */** Append a block to the redo log buffer.*
*@return whether the appending should continue */*
  bool operator()(const mtr_buf_t::block_t *block) {
    lsn_t start_lsn;
    lsn_t end_lsn;

    ut_ad(block != nullptr);

    if (block->used() == 0) {
      return (true);
    }

    start_lsn = m_lsn;

    end_lsn = log_buffer_write(*log_sys, m_handle, block->begin(),
                               block->used(), start_lsn);

    ut_a(end_lsn % OS_FILE_LOG_BLOCK_SIZE <
         OS_FILE_LOG_BLOCK_SIZE - LOG_BLOCK_TRL_SIZE);

    m_left_to_write -= block->used();

    if (m_left_to_write == 0
        && m_rec_group_start_lsn / OS_FILE_LOG_BLOCK_SIZE !=
               end_lsn / OS_FILE_LOG_BLOCK_SIZE) {
      log_buffer_set_first_record_group(*log_sys, m_handle, end_lsn);
    }

    log_buffer_write_completed(*log_sys, m_handle, start_lsn, end_lsn);

    m_lsn = end_lsn;

    return (true);
  }
.........................
};

```

通过上面的代码，可以看到核心就是两个调用，一个是log_buffer_write, 一个是log_buffer_write_completed.

#### log_buffer_write
先来看log_buffer_write,这个函数主要是写入到redo log buffer(log->buf).
1. 这里首先根据start_lsn(也就是前一次写入之后的lsn),来计算当前的redolog block的偏移(也就是上一次写入之后的可写位置).
2. 然后得到当前的block剩余的大小
3. 如果当前block可以写入在直接copy到当前的block
4. 否则只copy部分(left)内容到当前block,然后再次进入循环再写入一个新的block.
5. 最后则是返回最终写入完毕后的lsn.

``` cpp
lsn_t log_buffer_write(log_t &log, const Log_handle &handle, const byte *str,
                       size_t str_len, lsn_t start_lsn) {
............................

  const lsn_t end_sn = log_translate_lsn_to_sn(start_lsn) + str_len;

  byte *buf_end = log.buf + log.buf_size;

  byte *ptr = log.buf + (start_lsn % log.buf_size);

  lsn_t lsn = start_lsn;

  while (true) {
    const auto offset = lsn % OS_FILE_LOG_BLOCK_SIZE;

    const auto left = OS_FILE_LOG_BLOCK_SIZE - LOG_BLOCK_TRL_SIZE - offset;

    ut_a(left > 0);
    ut_a(left < OS_FILE_LOG_BLOCK_SIZE);

    size_t len, lsn_diff;

    if (left > str_len) {

      len = str_len;

      lsn_diff = str_len;

    } else {

      len = left;

      lsn_diff = left + LOG_BLOCK_TRL_SIZE + LOG_BLOCK_HDR_SIZE;
    }
..............................
    std::memcpy(ptr, str, len);

    str_len -= len;
    str += len;
    lsn += lsn_diff;
    ptr += lsn_diff;

    if (ptr >= buf_end) {

      ptr -= log.buf_size;
    }

    if (lsn_diff > left) {
......................................
      log_block_set_first_rec_group(
          reinterpret_cast<byte *>(uintptr_t(ptr) &
                                   ~uintptr_t(LOG_BLOCK_HDR_SIZE)),
          0);

      if (str_len == 0) {

        break;
      }

    } else {
      break;
    }
  }

...............................
  return (lsn);
}

```

#### log_buffer_write_completed

然后就是log_buffer_write_completed函数，这个函数主要是用来更新recent_written字段，这个字段主要是用来track已经写入到log buffer的lsn(后续分析redolog的时候会详细分析).

逻辑很简单，就是判断是否recent_written是否还有空间，如果没有则等待，否则加入到recent_written.

``` cpp
void log_buffer_write_completed(log_t &*log*, const Log_handle &*handle*,
                                lsn_t *start_lsn*, lsn_t *end_lsn*) {

  uint64_t wait_loops = 0;

  while (!log.recent_written.has_space(start_lsn)) {
    ++wait_loops;
    os_thread_sleep(20);
  }

  if (unlikely(wait_loops != 0)) {
    MONITOR_INC_VALUE(MONITOR_LOG_ON_RECENT_WRITTEN_WAIT_LOOPS, wait_loops);
  }
  std::atomic_thread_fence(std::memory_order_release);

  log.recent_written.add_link(start_lsn, end_lsn);
}

```

### log_wait_for_space_in_log_recent_closed

然后我们来看log_wait_for_space_in_log_recent_closed，到达这里的话，则说明我们已经写完log buffer,然后等待加脏页到flush list,而在InnoDB中log.recent_closed用来track在flush list中的脏页，因此这里在加脏页之前需要判断是否link buf已满。

``` cpp
void log_wait_for_space_in_log_recent_closed(log_t &*log*, lsn_t *lsn*) {
  ut_a(log_lsn_validate(lsn));

  ut_ad(lsn >= log_buffer_dirty_pages_added_up_to_lsn(log));

  uint64_t wait_loops = 0;

  while (!log.recent_closed.has_space(lsn)) {
    ++wait_loops;
    os_thread_sleep(20);
  }

  if (unlikely(wait_loops != 0)) {
    MONITOR_INC_VALUE(MONITOR_LOG_ON_RECENT_CLOSED_WAIT_LOOPS, wait_loops);
  }
}

```

### add_dirty_blocks_to_flush_list

然后就是加脏页到flush list中.

``` cpp
void mtr_t::Command::add_dirty_blocks_to_flush_list(lsn_t *start_lsn*,
                                                    lsn_t *end_lsn*) {
  Add_dirty_blocks_to_flush_list add_to_flush(start_lsn, end_lsn,
                                              m_impl->m_flush_observer);

  Iterate<Add_dirty_blocks_to_flush_list> iterator(add_to_flush);

  m_impl->m_memo.for_each_block_in_reverse(iterator);
}

```

这里有几个点要注意的.
* m_impl->m_memo
	* 这个里面保存了需要加入到flush list的block
	* 也就是说在使用mtr的时候，需要自己挂载block到这个数据结构
* Add_dirty_blocks_to_flush_list
	* 核心是这个数据结构，所有的操作都在这个结构里面

m_memo.for_each_block_in_reverse比较简单，就是从尾部开始遍历，然后调用iterator的()，因此这里我们来看Add_dirty_blocks_to_flush_list的().

这里可以看到最终就是调用add_dirty_page_to_flush_list来吧对应的block加入到flush list,这里不详细分析这块，以后我们分析buffer pool相关代码的时候会再来看这块.

``` cpp
  bool operator()(mtr_memo_slot_t **slot*) const {
    if (slot->object != NULL) {
      if (slot->type == MTR_MEMO_PAGE_X_FIX ||
          slot->type == MTR_MEMO_PAGE_SX_FIX) {
        add_dirty_page_to_flush_list(slot);

      } else if (slot->type == MTR_MEMO_BUF_FIX) {
        buf_block_t *block;
        block = reinterpret_cast<buf_block_t *>(slot->object);
        if (block->made_dirty_with_no_latch) {
          add_dirty_page_to_flush_list(slot);
          block->made_dirty_with_no_latch = false;
        }
      }
    }

```

#MySQL/InnoDB/MTR