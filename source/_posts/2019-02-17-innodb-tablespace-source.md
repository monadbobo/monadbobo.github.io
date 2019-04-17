---
title: MySQL · INNODB · Tablespace分析
author: diaoliang
tags:
  - InnoDB
translate_title: mysql.-innodb.-tablespace-analysis
---

## 简介
首先来看tablespace的定义：

> A data file that can hold data for one or more InnoDB tables and associated indexes.  

这里system tablespace是一个特殊的tablespace,他包含了很多数据文件(ibdata files).而且如果没有设置 file-per-table的话，所有的新创建的表的数据以及索引信息都会保存在它里面.

下面就是ibdata可能包含的内容

> A set of files with names such as ibdata1, ibdata2, and so on, that make up the InnoDB system tablespace. These files contain metadata about InnoDB tables, (the InnoDB data dictionary), and the storage areas for one or more undo logs, the change buffer, and the doublewrite buffer.   

假设设置了file-per-table(默认打开),那么每一个表都会有自己的tablespace文件(.ibd).

> The .ibd file extension does not apply to the system tablespace, which consists of one or more ibdata files.  

还有一种就是temporary tablespace，也就是临时表的tablespace.

> InnoDB uses two types of temporary tablespace. Session temporary tablespaces store user-created temporary tables and internal temporary tables created by the optimizer.  

## 源码分析
通过上面我们可以简单的认为tablespace就是对于数据库中table的内部抽象.

既然我们已经知道tablespace是和table相关的，那么我们就按照这个逻辑来分析源码.

### shared tablespace
首先来看system tablespace的创建，在InnoDB中每一个tablespace都会有一个uint32类型的id,每一个唯一的id用来表示对应的tablespace,而system tablespace的space id就是0.

`static const space_id_t TRX_SYS_SPACE = 0;`


而由于只有system tablespace和temporary tablespace是共享的，因此他们在InnoDB中有专门的数据结构来表示他们(Tablespace表示所有的shared tablespace的基类).

``` cpp
class SysTablespace : public Tablespace {
 public:
  SysTablespace()
      : m_auto_extend_last_file(),
        m_last_file_size_max(),
        m_created_new_raw(),
        m_is_tablespace_full(false),
        m_sanity_checks_done(false) {
    /* No op */
  }
```

而对应的变量则是

``` cpp
/** The control info of the system tablespace. */
SysTablespace srv_sys_space;

/** The control info of a temporary table shared tablespace. */
SysTablespace srv_tmp_space;
```

而tablespace数据文件的创建则是在SysTablespace::open_or_create中，因此我们来看这个函数.这个函数主要功能是遍历当前system tablespace的所有文件(m_files),然后存在就打开，不存在就创建对应的文件.

``` cpp
files_t::iterator begin = m_files.begin();
  files_t::iterator end = m_files.end();

  ut_ad(begin->order() == 0);

  for (files_t::iterator it = begin; it != end; ++it) {
    if (it->m_exists) {
      err = open_file(*it);

    } else {
      err = create_file(*it);
    }
```

当打开文件之后，将会把打开的文件进行缓存，而InnoDB会将所有的tablespace缓存在fil_system中.这里要注意的是在将文件放入缓存之前会先关闭，因为最终所有的文件都是通过space(fil_space_t)对象来操作的.

``` cpp
/* Close the curent handles, add space and file info to the
  fil_system cache and the Data Dictionary, and re-open them
  in file_system cache so that they stay open until shutdown. */
  ulint node_counter = 0;
  for (files_t::iterator it = begin; it != end; ++it) {
    it->close();
    it->m_exists = true;
    if (it == begin) {
      /* First data file. */

      /* Create the tablespace entry for the multi-file
      tablespace in the tablespace manager. */
      space =
          fil_space_create(name(), space_id(), flags(),
                           is_temp ? FIL_TYPE_TEMPORARY : FIL_TYPE_TABLESPACE);
    }
    page_no_t max_size =
        (++node_counter == m_files.size()
             ? (m_last_file_size_max == 0 ? PAGE_NO_MAX : m_last_file_size_max)
             : it->m_size);

    /* Add the datafile to the fil_system cache. */
    if (!fil_node_create(it->m_filepath, it->m_size, space,
                         it->m_type != SRV_NOT_RAW, it->m_atomic_write,
                         max_size)) {
      err = DB_ERROR;
      break;
    }
  }
```

上面的代码中有两个关键调用第一个就是fil_space_create,可以看到遍历文件一开始就创建space,这里后续所有对于space的操作都是通过fil_space_t来进行的. 而这里我们可以看到对于sys_space和redo_space都是有全局变量来操作的.

``` cpp
fil_space_t *fil_space_create(const char *name, space_id_t space_id,
                              ulint flags, fil_type_t purpose) {
.....................................
  fil_system->mutex_acquire_all();

  auto shard = fil_system->shard_by_id(space_id);

  auto space = shard->space_create(name, space_id, flags, purpose);
...........................................

  /* Cache the system tablespaces, avoid looking them up during IO. */

  if (space->id == TRX_SYS_SPACE) {
    ut_a(fil_space_t::s_sys_space == nullptr ||
         fil_space_t::s_sys_space == space);

    fil_space_t::s_sys_space = space;

  } else if (space->id == dict_sys_t::s_log_space_first_id) {
    ut_a(fil_space_t::s_redo_space == nullptr ||
         fil_space_t::s_redo_space == space);

    fil_space_t::s_redo_space = space;
  }
.............................
  return (space);
}
```

而fil_node_create就是将当前的文件加入到创建好的space中并且缓存，在InnoDB中所有的数据文件都会统一管理，其中redo/undo会做特殊处理，而其他的tablespace则会根据他们的spaceid进行缓存.

``` cpp
char *fil_node_create(const char *name, page_no_t size, fil_space_t *space,
                      bool is_raw, bool atomic_write, page_no_t max_pages) {
  auto shard = fil_system->shard_by_id(space->id);

  fil_node_t *file;

  file = shard->create_node(name, size, space, is_raw,
                            IORequest::is_punch_hole_supported(), atomic_write,
                            max_pages);

  return (file == nullptr ? nullptr : file->name);
}
```


此时的疑问就是m_files是何时被初始化的，也就是system tablespace文件的初始化，这里system tablespace的文件初始化是在数据库init的时候被初始化的，因此我们来看相关代码 .

InnoDB中有一个变量叫做innobase_data_file_path，这个变量是一个字符串，这个字符串包含了所有的system tablespace需要创建 的文件以及一些属性，这个字符串默认值是在InnoDB引擎初始化的时候初始化的.

``` cpp
  /* Set default InnoDB temp data file size to 12 MB and let it be
  auto-extending. */
  if (!innobase_data_file_path) {
    innobase_data_file_path = (char *)"ibdata1:12M:autoextend";
  }
```

可以看到默认只创建ibdata1文件，并且大小为12m,自动扩展，而innobase_data_file_path的格式如下

> innodb_data_file_path=datafile_spec1[;datafile_spec2]..  
> file_name:file_size[:autoextend[:max:max_file_size]]  

因此system tablespace创建文件也会根据这个配置来创建，也就是当InnoDB参数解析完毕之后进入system tablespace的文件创建.

``` cpp
  if (int error = innodb_init_params()) {
    DBUG_RETURN(error);
  }

  /* After this point, error handling has to use
  innodb_init_abort(). */

  if (!srv_sys_space.parse_params(innobase_data_file_path, true)) {
    ib::error(ER_IB_MSG_545)
        << "Unable to parse innodb_data_file_path=" << innobase_data_file_path;
    DBUG_RETURN(innodb_init_abort());
  }
```

parse_params这个函数将会解析innobase_data_file_path,然后根据不同的属性创建对应的文件，先来看参数解析.

``` cpp
/*---------------------- PASS 1 ---------------------------*/
  /* First calculate the number of data files and check syntax. */
  while (*ptr != '\0') {
    filepath = ptr;

    ptr = parse_file_name(ptr);

    if (ptr == filepath) {
...........................
      return (false);
    }

    if (*ptr == '\0') {
.................................

      return (false);
    }

    ptr++;

    size = parse_units(ptr);

    if (size == 0) {
.........................
      return (false);
    }

    if (0 == strncmp(ptr, ":autoextend", (sizeof ":autoextend") - 1)) {
      ptr += (sizeof ":autoextend") - 1;

      if (0 == strncmp(ptr, ":max:", (sizeof ":max:") - 1)) {
        ptr += (sizeof ":max:") - 1;

        page_no_t max = parse_units(ptr);

        if (max < size) {
          goto invalid_size;
        }
      }

      if (*ptr == ';') {
..................................
        return (false);
      }
    }

    if (0 == strncmp(ptr, "new", (sizeof "new") - 1)) {
      ptr += (sizeof "new") - 1;
    }

    if (0 == strncmp(ptr, "raw", (sizeof "raw") - 1)) {
...............................
        return (false);
      }

      ptr += (sizeof "raw") - 1;
    }

    ++n_files;

    if (*ptr == ';') {
      ptr++;
    } else if (*ptr != '\0') {
.......................
      return (false);
    }
  }

  if (n_files == 0) {
......................
    return (false);
  }
```

然后第二步就是存储对应的文件名以及属性到m_files中，这里有一个重要的数据结构就是Datafile,每一个数据文件在InnoDB中的抽象就是Datafile.

``` cpp
while (*ptr != '\0') {
    filepath = ptr;

    ptr = parse_file_name(ptr);

    if (*ptr == ':') {
      /* Make filepath a null-terminated string */
      *ptr = '\0';
      ptr++;
    }

    size = parse_units(ptr);
    ut_ad(size > 0);

    if (0 == strncmp(ptr, ":autoextend", (sizeof ":autoextend") - 1)) {
      m_auto_extend_last_file = true;

      ptr += (sizeof ":autoextend") - 1;

      if (0 == strncmp(ptr, ":max:", (sizeof ":max:") - 1)) {
        ptr += (sizeof ":max:") - 1;

        m_last_file_size_max = parse_units(ptr);
      }
    }

    m_files.push_back(Datafile(filepath, flags(), size, order));
    Datafile *datafile = &m_files.back();
    datafile->make_filepath(path(), filepath, NO_EXT);

    if (0 == strncmp(ptr, "new", (sizeof "new") - 1)) {
      ptr += (sizeof "new") - 1;
    }

    if (0 == strncmp(ptr, "raw", (sizeof "raw") - 1)) {
      ut_a(supports_raw);

      ptr += (sizeof "raw") - 1;

      /* Initialize new raw device only during initialize */
      m_files.back().m_type =
#ifndef UNIV_HOTBACKUP
          opt_initialize ? SRV_NEW_RAW : SRV_OLD_RAW;
#else  /* !UNIV_HOTBACKUP */
          SRV_OLD_RAW;
#endif /* !UNIV_HOTBACKUP */
    }

    if (*ptr == ';') {
      ++ptr;
    }
    order++;
  }
```


### 非shared tablespace

然后我们来看一般的tablespace的创建,一般来说每创建一个表都会创建一个新的ibd文件，也就是tablespace. 而这种tablespace以及Redolog/undolog都是属于fil_space_t这个结构体.通过上面的代码我们知道system tablespace最终也是创建一个fil_space_t然后再接入整个系统的tablespace的管理的.

这里先来看两个特殊的tablespace,也就是REDO log和UNDO log，由于这两个log也都是磁盘上的文件，因此在InnoDB中会讲这两个log文件作为一种特殊的tablespace，来看他们的初始化.

``` cpp
static dberr_t create_log_files(char *logfilename, size_t dirnamelen, lsn_t lsn,
                                char *&logfile0, lsn_t &checkpoint_lsn) {
........................
    /* Disable the doublewrite buffer for log files. */
    fil_space_t *log_space = fil_space_create(
        "innodb_redo_log", dict_sys_t::s_log_space_first_id,
        fsp_flags_set_page_size(0, univ_page_size), FIL_TYPE_LOG);
.......................
}

```

下面的undo log.

``` cpp
static dberr_t srv_undo_tablespace_open(space_id_t space_id) {
....................................
    space = fil_space_create(undo_name, space_id, flags, FIL_TYPE_TABLESPACE);
..............
}
```

以后我们详细分析undolog/redolog的时候会再来分析这个地方

接下来我们就来看当一般表的创建的时候会发生什么，对于一般的表，创建tablespace是跟随着ibd文件一起创建的(上面的都是在初始化的时候创建)。

``` cpp
static bool dd_create_hardcoded(space_id_t space_id, const char *filename) {
  page_no_t pages = FIL_IBD_FILE_INITIAL_SIZE;

  dberr_t err = fil_ibd_create(space_id, dict_sys_t::s_dd_space_name, filename,
                               predefined_flags, pages);

  if (err == DB_SUCCESS) {
    mtr_t mtr;
    mtr.start();

    bool ret = fsp_header_init(space_id, pages, &mtr, true);

    mtr.commit();

    if (ret) {
      btr_sdi_create_index(space_id, false);
      return (false);
    }
  }

  return (true);
}

```

### Tablespace pool

在MySQL 8.0中，由于临时表也全部使用了InnoDB,因此创建代价会比5.6(myisam)大，所以在8.0中session临时表会有一个pool来控制，在启动的时候会创建这个pool。

``` cpp
  err = ibt::open_or_create(create_new_db);
  if (err != DB_SUCCESS) {
    return (srv_init_abort(err));
  }

```


在open_or_create函数中会先创建对应的临时表的目录，然后再创建对应的pool,最后初始化pool.
* 这里初始的tablespace pool的大小是10(INIT_SIZE)

``` cpp
dberr_t open_or_create(bool create_new_db) {
  dberr_t err = create_temp_dir();
  if (err != DB_SUCCESS) {
    return (err);
  }

  tbsp_pool = UT_NEW_NOKEY(Tablespace_pool(INIT_SIZE));
  if (tbsp_pool == nullptr) {
    return (DB_OUT_OF_MEMORY);
  }

  err = tbsp_pool->initialize(create_new_db);

  return (err);
}

```

Tablespace pool的实现比较粗糙,它包含了两个链表，一个保存已经使用的tablespace,一个保存了还未使用的tablespace.

下面我们来看下Tablespace_pool这个类的字段.
* m_free和m_active分别保存了未使用和已使用的.
* m_mutex用来保证并发的存取pool

``` cpp
class Tablespace_pool {
 public:
  using Pool = std::list<Tablespace *, ut_allocator<Tablespace *>>;
............................
 private:
  */** True after the pool has been initialized */*
  bool m_pool_initialized;
  */** Initial size of pool */*
  size_t m_init_size;
  */** Vector of tablespaces that are unused */*
  Pool *m_free;
  */** Vector of tablespaces that are being used */*
  Pool *m_active;
  */** Mutex to protect concurrent operations on the pool */*
  ib_mutex_t m_mutex;
};

```

接下来我们来看tablespace_pool的初始化.

* 初始化m_active和m_free两个链表
* 调用expand来扩展pool.

``` cpp
dberr_t Tablespace_pool::initialize(bool create_new_db) {
  if (m_pool_initialized) {
    return (DB_SUCCESS);
  }

  ut_ad(m_active == nullptr && m_free == nullptr);

  m_active = UT_NEW_NOKEY(Pool());
  if (m_active == nullptr) {
    return (DB_OUT_OF_MEMORY);
  }

  m_free = UT_NEW_NOKEY(Pool());
  if (m_free == nullptr) {
    return (DB_OUT_OF_MEMORY);
  }

  delete_old_pool(create_new_db);
  dberr_t err = expand(m_init_size);
  if (err != DB_SUCCESS) {
    return (err);
  }

  m_pool_initialized = true;
  return (DB_SUCCESS);
}

```

这样的话，所有的初始化其实都在expand中进行.
* 创建Tablespace对象(非shared的tablespace)
* 加入到m_free链表中.

``` cpp
dberr_t Tablespace_pool::expand(size_t size) {
  ut_ad(!m_pool_initialized || mutex_own(&m_mutex));
  for (size_t i = 0; i < size; i++) {
    Tablespace *ts = UT_NEW_NOKEY(Tablespace());

    if (ts == nullptr) {
      return (DB_OUT_OF_MEMORY);
    }

    dberr_t err = ts->create();

    if (err == DB_SUCCESS) {
      m_free->push_back(ts);
    } else {
      UT_DELETE(ts);
      return (err);
    }
  }
  return (DB_SUCCESS);
}

```

最后我们来看tablespace_pool的读取以及删除方法.首先来看get

* 如果m_free为空，也就是pool中的所有的tablespace都已经被使用了，此时InnoDB会再次扩展这个pool.而再次扩展的大小也是10(POOL_EXPAND_SIZE)
* 每次进来都会使用mutex锁来控制
* 使用很简单就是从m_free中得到一个ts,然后再将这个ts加入到m_active中.

``` cpp
Tablespace *Tablespace_pool::get(my_thread_id id, enum tbsp_purpose purpose) {
  DBUG_EXECUTE_IF("ibt_pool_exhausted", return (nullptr););

  Tablespace *ts = nullptr;
  acquire();
  if (m_free->size() == 0) {
    */* Free pool is empty. Add more tablespaces by expanding it */*
    dberr_t err = expand(POOL_EXPAND_SIZE);
    if (err != DB_SUCCESS) {
      */* Failure to expand the pool means that there is no disk space*
*available to create .IBT files */*
      release();
      ib::error() << "Unable to expand the temporary tablespace pool";
      return (nullptr);
    }
  }

  ts = m_free->back();
  m_free->pop_back();
  m_active->push_back(ts);
  ts->set_thread_id_and_purpose(id, purpose);

  release();
  return (ts);
}

```

最后就是释放使用过的tablespace到pool中.

* 可以看到所有文件都是由fil_space管理
* 从m_active中遍历然后查找出对应的ts
	* 线性查找，可以看到如果临时表如果同时创建很多的话，这里性能会很成问题
* 如果查找到则从m_active中删除，最后加入到m_free中.

``` cpp
void Tablespace_pool::free_ts(Tablespace *ts) {
  space_id_t space_id = ts->space_id();
  fil_space_t *space = fil_space_get(space_id);
  ut_ad(space != nullptr);

  if (space->size != FIL_IBT_FILE_INITIAL_SIZE) {
    ts->truncate();
  }

  acquire();

  Pool::iterator it = std::find(m_active->begin(), m_active->end(), ts);
  if (it != m_active->end()) {
    m_active->erase(it);
  } else {
    ut_ad(0);
  }

  m_free->push_back(ts);

  release();
}

```