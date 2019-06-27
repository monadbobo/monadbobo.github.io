---
title: InnoDB中Redo log设计与实现(一)
author: Simon Liu
tags:
  - InnoDB
translate_title: redo-log-design-and-implementation-in-innodb-1
---

## 简介
首先来看redolog的定义:

> A disk-based data structure used during crash recovery, to correct data written by incomplete transactions.   

在redolog中最核心的两个概念，一个是lsn,一个是sn,先来看lsn.

> Acronym for “log sequence number”. This arbitrary, ever-increasing value represents a point in time corresponding to operations recorded in the redo log.  

Lsn比较好理解，就是表示已经写入到redolog的字节数，这个值会随着不断写入而不断的变大.

在看sn之前，我们需要先了解一下redolog基本的写入方式.

在InnoDB中，最小的写入单位是512字节，也就是一个block(OS_FILE_LOG_BLOCK_SIZE), 每一个block都会包含一个12字节的header(LOG_BLOCK_HDR_SIZE),以及4字节的footer(LOG_BLOCK_TRL_SIZE)组成. 

由上面所知，我们此时其实需要两个”lsn”,一个是真正应该写入文件的lsn,一个是去掉header和footer的lsn.而sn就是后一个，也就是去掉header和footer的lsn.

## 源码分析
在InnoDB中redolog的数据结构就是struct log_t，而初始化则是在log_sys_init中进行.

RedoLog主要是用来修改内存中的页以及将每次对于页的修改写入到磁盘以便于recover的时候可以恢复.

RedoLog的初始化流程是这样子的.
1. log_sys_init
2. log_start
3. log_start_background_threads

因此我们就按照这个顺序来分析代码

### 初始化
所有的初始化都是在log_sys_init这个函数中进行，接下来我会分开来介绍这个函数的实现，首先来看第一部分.

``` cpp
  /* Initialize simple value fields. */
  log.dict_persist_margin.store(0);
  log.periodical_checkpoints_enabled = false;
  log.format = LOG_HEADER_FORMAT_CURRENT;
  log.files_space_id = space_id;
  log.state = log_state_t::OK;
  log.n_log_ios_old = log.n_log_ios;
  log.last_printout_time = time(nullptr);
```

然后我们来一个个看每个域的含义.

1. **首先是dict_persist_margin**

2. periodical_checkpoints_enabled 这个变量如果被打开则InnoDB将会在每innodb_log_checkpoint_every ms执行一次checkpoint.

3. format，这个域表示了redolog的格式，这里主要是为了兼容之前的MySQL版本，这个format将会保存在LOG_HEADER_FORMAT中.

然后我们来看InnoDB都有几种format.

``` cpp
/** Supported redo log formats. Stored in LOG_HEADER_FORMAT. */
enum log_header_format_t {
  /** The MySQL 5.7.9 redo log format identifier. We can support recovery
  from this format if the redo log is clean (logically empty). */
  LOG_HEADER_FORMAT_5_7_9 = 1,

  /** Remove MLOG_FILE_NAME and MLOG_CHECKPOINT, introduce MLOG_FILE_OPEN
  redo log record. */
  LOG_HEADER_FORMAT_8_0_1 = 2,

  /** Allow checkpoint_lsn to point any data byte within redo log (before
  it had to point the beginning of a group of log records). */
  LOG_HEADER_FORMAT_8_0_3 = 3,

  /** The redo log format identifier
  corresponding to the current format version. */
  LOG_HEADER_FORMAT_CURRENT = LOG_HEADER_FORMAT_8_0_3
};
```

可以看到当前的MySQL 8.0只支持到5.7.9这个版本.

4. spaceid,之前介绍tablespace的时候我们知道redolog也是一个tablespace,因此这里他需要设置spaceid.

5. State,这个表示当前log的状态,这里只有两个状态，一个是正常一个是corrupted.
6. n_log_ios_old用于统计信息,表示上次打印统计信息的时候的io次数.
7. last_printout_time上次打印统计信息的时候的时间.

然后是第二部分
``` cpp
  log.file_size = file_size;
  log.n_files = n_files;
  log.files_real_capacity = file_size * n_files;
```

1. file_size表示每个redolog的文件大小(srv_log_file_size)
2. n_files表示redolog的个数(srv_n_log_files).
3. files_real_capacity则是总的redolog的文件大小.

第三部分

``` cpp
  log.current_file_lsn = LOG_START_LSN;
  log.current_file_real_offset = LOG_FILE_HDR_SIZE;
  log_files_update_offsets(log, log.current_file_lsn);
```

1. current_file_lsn表示当前的lsn，而current_file_real_offset则表示当前lsn下在redolog的偏移.
这里这两个区别在于，有可能会有多个redolog,而lsn是全局的，因此我们需要通过lsn来得到当前的lsn所在的redolog file的偏移.

```
#define OS_FILE_LOG_BLOCK_SIZE 512
constexpr lsn_t LOG_START_LSN = 16 * OS_FILE_LOG_BLOCK_SIZE;
constexpr uint32_t LOG_FILE_HDR_SIZE = 4 * OS_FILE_LOG_BLOCK_SIZE;

```

2. log_files_update_offsets 用来update对应的offset.

第四部分，这部分主要是用来初始化event，这里有这么多event,主要是因为redolog会有好几个线程(后续介绍),然后这些线程之间会有交互，因此这些event就是用来做这些事情.

``` cpp
  log.checkpointer_event = os_event_create("log_checkpointer_event");
  log.closer_event = os_event_create("log_closer_event");
  log.write_notifier_event = os_event_create("log_write_notifier_event");
  log.flush_notifier_event = os_event_create("log_flush_notifier_event");
  log.writer_event = os_event_create("log_writer_event");
  log.flusher_event = os_event_create("log_flusher_event");
```

第五部分是初始化sn_lock,这个log主要是保护sn.

``` cpp
  log.sn_lock.create(
#ifdef UNIV_PFS_RWLOCK
      log_sn_lock_key,
#else
      PSI_NOT_INSTRUMENTED,
#endif
      SYNC_LOG_SN, 64);
```


第六部分是初始化每个线程使用的锁.

``` cpp
  mutex_create(LATCH_ID_LOG_CHECKPOINTER, &log.checkpointer_mutex);
  mutex_create(LATCH_ID_LOG_CLOSER, &log.closer_mutex);
  mutex_create(LATCH_ID_LOG_WRITER, &log.writer_mutex);
  mutex_create(LATCH_ID_LOG_FLUSHER, &log.flusher_mutex);
  mutex_create(LATCH_ID_LOG_WRITE_NOTIFIER, &log.write_notifier_mutex);
  mutex_create(LATCH_ID_LOG_FLUSH_NOTIFIER, &log.flush_notifier_mutex);
```

第七部分是初始化每个线程需要使用的buffer.
* log.buf是上层应用写入到InnoDB的第一站,每一次mtr提交都是先写入到这里.
	* srv_log_buffer_size
* log.write_ahead_buf ?
	* srv_log_write_ahead_size
* log. checkpoint_buf 主要是用于checkpoint
	* OS_FILE_LOG_BLOCK_SIZE
	* 可以看到大小也就是redo log中一个block的大小.


``` cpp
  /* Allocate buffers. */
  log_allocate_buffer(log);
  log_allocate_write_ahead_buffer(log);
  log_allocate_checkpoint_buffer(log);
  log_allocate_recent_written(log);
  log_allocate_recent_closed(log);
  log_allocate_flush_events(log);
  log_allocate_write_events(log);
  log_allocate_file_header_buffers(log);
```

最后一部分是计算buf大小以及checkpoint相关.

``` cpp
log_calc_buf_size(log);

  if (!log_calc_max_ages(log)) {
........................................
    return (false);
  }
```

### 根据checkpoint来再次初始化对应的数据

首先是初始化一些统计信息，write_to_file_requests_total表示redolog对于io的请求数量，而write_to_file_requests_interval表示平均的请求数量(ms)。

``` cpp
  log.write_to_file_requests_total.store(0);
  log.write_to_file_requests_interval.store(0);
```


然后是初始化一些lsn的信息.
1. recovered_lsn表示recover的lsn.
2. last_checkpoint_lsn表示最新的进行过checkpoint的lsn.
3. next_checkpoint_no表示下一个checkpoint number.
4. available_for_checkpoint_lsn表示可以进行checkpoint的lsn

``` cpp
  log.recovered_lsn = start_lsn;
  log.last_checkpoint_lsn = checkpoint_lsn;
  log.next_checkpoint_no = checkpoint_no;
  log.available_for_checkpoint_lsn = checkpoint_lsn;
```

紧接着就是根据传递进来的start_lsn来得到正确的偏移，这里start_lsn有两种偏移需要处理，一种是start_lsn刚好是一个新的block的开始，那么此时lsn则需要跳过block头，还有一种是start_lsn刚好是处于一个block的结尾(不包括footer),那么此时就需要跳过footer+header.

``` cpp
  if ((start_lsn + LOG_BLOCK_TRL_SIZE) % OS_FILE_LOG_BLOCK_SIZE == 0) {
    start_lsn += LOG_BLOCK_TRL_SIZE + LOG_BLOCK_HDR_SIZE;
  } else if (start_lsn % OS_FILE_LOG_BLOCK_SIZE == 0) {
    start_lsn += LOG_BLOCK_HDR_SIZE;
  }
```

然后就是更新两个关键的数据结构recent_written以及recent_closed,这里简单的介绍下

``` cpp
  log.recent_written.add_link(0, start_lsn);
  log.recent_written.advance_tail();
  ut_a(log_buffer_ready_for_write_lsn(log) == start_lsn);

  log.recent_closed.add_link(0, start_lsn);
  log.recent_closed.advance_tail();
  ut_a(log_buffer_dirty_pages_added_up_to_lsn(log) == start_lsn);
```


最后则是写入第一个block,这里可以看到block的写入是直接写到log.buf中.这里之所以需要写入第一个block主要是为了补齐，因为在redolog中写入最小单位就是block,因此这里start_lsn如果不是512对其，那么需要跳过非对其的位置到log.buf,

``` cpp
lsn_t block_lsn;
  byte *block;

  block_lsn = ut_uint64_align_down(start_lsn, OS_FILE_LOG_BLOCK_SIZE);

  ut_a(block_lsn % log.buf_size + OS_FILE_LOG_BLOCK_SIZE <= log.buf_size);

  block = static_cast<byte *>(log.buf) + block_lsn % log.buf_size;

  log_block_set_hdr_no(block, log_block_convert_lsn_to_no(block_lsn));

  log_block_set_flush_bit(block, true);

  log_block_set_data_len(block, start_lsn - block_lsn);

  log_block_set_first_rec_group(block, start_lsn % OS_FILE_LOG_BLOCK_SIZE);
```


下面就是第一个block的内存内容.

1. header_no 表示当前的redolog的no. 4个字节
``` cpp
inline void log_block_set_hdr_no(byte *log_block, uint32_t n) {
  ut_a(n > 0);
  ut_a(n < LOG_BLOCK_FLUSH_BIT_MASK);
  ut_a(n <= LOG_BLOCK_MAX_NO);

  mach_write_to_4(log_block + LOG_BLOCK_HDR_NO, n);
}
```
2. flush_bit. 表示是否flush，这里需要注意，最终flush_bit是保存在header_no的最高位.

``` cpp
inline void log_block_set_flush_bit(byte *log_block, bool value) {
  uint32_t field = mach_read_from_4(log_block + LOG_BLOCK_HDR_NO);

  ut_a(field != 0);

  if (value) {
    field = field | LOG_BLOCK_FLUSH_BIT_MASK;
  } else {
    field = field & ~LOG_BLOCK_FLUSH_BIT_MASK;
  }

  mach_write_to_4(log_block + LOG_BLOCK_HDR_NO, field);
}
```

3. 数据长度。2个字节

``` cpp
inline void log_block_set_data_len(byte *log_block, ulint len) {
  mach_write_to_2(log_block + LOG_BLOCK_HDR_DATA_LEN, len);
}
```

4. 最后是设置rec group。也是2个字节.?
 
```cpp
inline void log_block_set_first_rec_group(byte *log_block, uint32_t offset) {
  mach_write_to_2(log_block + LOG_BLOCK_FIRST_REC_GROUP, offset);
}
```

### 启动后台工作线程

log_start_background_threads这个函数主要就是用来启动后台线程.

1. 设置对应的标记位置.
```cpp
  log.closer_thread_alive.store(true);
  log.checkpointer_thread_alive.store(true);
  log.writer_thread_alive.store(true);
  log.flusher_thread_alive.store(true);
  log.write_notifier_thread_alive.store(true);
  log.flush_notifier_thread_alive.store(true);

  log.should_stop_threads.store(false);
```

2. 启动线程
这里一共有6个线程.

* log_checkpointer 做checkpoint的线程
* log_closer 关闭整个写入操作的线程
* log_writer 从log.buf写入到磁盘的线程
* log_flusher 刷新redolog到磁盘的线程(sync).
* log_write_notifier 唤醒等到写入的线程
* log_flush_notifier 唤醒等待刷新的线程

``` cpp
os_thread_create(log_checkpointer_thread_key, log_checkpointer, &log);

  os_thread_create(log_closer_thread_key, log_closer, &log);

  os_thread_create(log_writer_thread_key, log_writer, &log);

  os_thread_create(log_flusher_thread_key, log_flusher, &log);

  os_thread_create(log_write_notifier_thread_key, log_write_notifier, &log);

  os_thread_create(log_flush_notifier_thread_key, log_flush_notifier, &log);
```

