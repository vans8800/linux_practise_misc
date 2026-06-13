## 背景
---

第一种类型是缓存 I/O。

大多数文件系统的默认 I/O 操作都是缓存 I/O。、

对于读操作来讲，操作系统会先检查，内核的缓冲区有没有需要的数据。

若已经缓存了，那就直接从缓存中返回；否则从磁盘中读取，然后缓存在操作系统的缓存中。

对于写操作来讲，操作系统会先将数据从用户空间复制到内核空间的缓存中。

这时对用户程序来说，写操作就已经完成。

至于什么时候再写到磁盘中由操作系统决定，除非显式地调用了 sync 同步命令。

第二种类型是直接 IO，就是应用程序直接访问磁盘数据，而不经过内核缓冲区，从而减少了在内核缓存和用户程序之间数据复制。


## PageCache登场 


这里的**内核缓冲区**和**缓存 I/O** 中提到的缓存，本质上就是 **Page Cache（页缓存）**。

### 对应关系说明

文中描述的机制和 Page Cache 完全吻合：

**读操作：**

> "操作系统会先检查内核的缓冲区有没有需要的数据，若已经缓存了就直接从缓存中返回"

这正是 Page Cache 的工作方式——内核以"页"为单位缓存磁盘数据，读取时优先命中缓存，未命中才触发磁盘 I/O（缺页）。

**写操作：**

> "操作系统会先将数据从用户空间复制到内核空间的缓存中……至于什么时候再写到磁盘由操作系统决定"

这描述的是 Page Cache 的**回写（Write-back）机制**——数据先写入 Page Cache 中的脏页，由内核的 `pdflush` / `kworker` 等后台线程择机刷盘，或通过 `sync` 强制触发。

**直接 I/O 的对比：**

> "应用程序直接访问磁盘数据，而不经过内核缓冲区"

直接 I/O（`O_DIRECT`）就是**绕过 Page Cache**，数据在用户空间和磁盘之间直接传输，适合数据库等自己管理缓存的场景。

### 总结

| 文中术语                    | 实际对应   |
| --------------------------- | ---------- |
| 内核缓冲区                  | Page Cache |
| 缓存 I/O 中的"缓存"         | Page Cache |
| 直接 I/O 绕过的"内核缓冲区" | Page Cache |

> 补充：Linux 中历史上还存在独立的 **Buffer Cache**（用于裸块设备），但自 Linux 2.4 之后，Buffer Cache 已经统一整合进 Page Cache，二者本质上是同一套机制。
>
> 所以现在说"内核缓冲区"在文件系统 I/O 场景下，指的就是 Page Cache。

## PageCache 在内核中呈现
---

Page Cache 在 Linux 内核源码中的核心呈现形式如下：

------

### 1. 核心数据结构：`struct page` / `struct folio`

Page Cache 的基本存储单元是**内存页**，在源码中对应：

```c
// include/linux/mm_types.h

struct page {
    unsigned long flags;        // 页标志，如 PG_dirty（脏页）、PG_uptodate 等
    union {
        struct address_space *mapping;  // 指向所属的 address_space
        ...
    };
    pgoff_t index;              // 在文件中的页偏移量（第几页）
    atomic_t _refcount;         // 引用计数
    ...
};
```

> Linux 5.16+ 引入了 `struct folio` 作为对 `struct page` 的封装，用于支持大页（large folio），但底层原理一致。

------

### 2. 组织结构：`struct address_space`

每个文件（inode）都有一个 `address_space`，它是 Page Cache 的**索引和管理核心**：

```c
// include/linux/fs.h

struct address_space {
    struct inode            *host;        // 所属 inode（文件）
    struct xarray           i_pages;      // 核心：存储所有缓存页的基数树/XArray
    gfp_t                   gfp_mask;
    atomic_t                i_mmap_writable;
    struct rb_root_cached   i_mmap;       // 映射到此文件的所有 VMA
    unsigned long           nrpages;      // 缓存页数量
    pgoff_t                 writeback_index; // 回写起始位置
    const struct address_space_operations *a_ops;  // 操作函数表（关键！）
    ...
};
```

**关系图：**

```
inode
  └── address_space
          ├── i_pages (XArray)
          │     ├── page[0]   ← 文件第0页
          │     ├── page[1]   ← 文件第1页
          │     └── page[N]   ← 文件第N页
          └── a_ops (操作集)
```

------

### 3. 操作函数表：`address_space_operations`

定义了 Page Cache 与具体文件系统之间的接口：

```c
// include/linux/fs.h

struct address_space_operations {
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*read_folio)(struct file *, struct folio *);   // 从磁盘读一页进缓存
    int (*writepages)(struct address_space *, struct writeback_control *);
    bool (*dirty_folio)(struct address_space *, struct folio *); // 标记脏页
    int (*readahead)(struct readahead_control *);       // 预读
    int (*write_begin)(struct file *, struct address_space *,
                       loff_t, unsigned, struct page **, void **);
    int (*write_end)(struct file *, struct address_space *,
                     loff_t, unsigned, unsigned, struct page *, void *);
    ...
};
```

------

### 4. 读写核心路径源码

#### 读路径（`filemap_read`）

```c
// mm/filemap.c

// 用户 read() 系统调用最终进入：
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter, ...)
{
    ...
    // 1. 在 address_space->i_pages 中查找页
    folio = filemap_get_folio(mapping, index);
    
    // 2. 缓存命中 → 直接拷贝到用户空间
    // 3. 缓存未命中 → 触发 page_cache_sync_readahead() 从磁盘读取
    //    → 调用 a_ops->read_folio() → 提交 bio 请求
    ...
}
```

#### 写路径（`generic_perform_write`）

```c
// mm/filemap.c

ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
    ...
    // 1. 找到或分配对应的缓存页
    a_ops->write_begin(..., &page);
    
    // 2. 将数据从用户空间拷贝到 Page Cache 的页中
    copy_page_from_iter_atomic(page, offset, bytes, i);
    
    // 3. 标记为脏页（不立即写盘）
    a_ops->write_end(..., page);
    // → 内部调用 folio_mark_dirty()
    ...
}
```

------

### 5. 脏页回写机制

```c
// 脏页标志在 page flags 中体现：
#define PG_dirty        4    // include/linux/page-flags.h

// 后台线程触发回写：
// fs/fs-writeback.c
void wakeup_flusher_threads(enum wb_reason reason);

// 最终调用链：
// writeback_sb_inodes()
//   → __writeback_single_inode()
//     → do_writepages()
//       → a_ops->writepages()   ← 具体文件系统实现刷盘
```

------

### 整体结构总览

```
用户进程 read()/write()
        │
        ▼
  VFS 层（sys_read/sys_write）
        │
        ▼
  filemap_read / generic_perform_write
        │
        ▼
  address_space->i_pages (XArray)     ← Page Cache 核心索引
     ├── 命中 → 直接操作内存页
     └── 未命中 → a_ops->read_folio()
                      │
                      ▼
               提交 bio 到块设备层
                      │
                      ▼
                    磁盘 I/O
```

------

### 关键源码文件索引

| 文件                         | 作用                                |
| ---------------------------- | ----------------------------------- |
| `mm/filemap.c`               | Page Cache 读写核心实现             |
| `mm/page-writeback.c`        | 脏页回写控制                        |
| `fs/fs-writeback.c`          | writeback 线程调度                  |
| `include/linux/fs.h`         | `address_space` 等结构定义          |
| `include/linux/mm_types.h`   | `struct page` / `struct folio` 定义 |
| `include/linux/page-flags.h` | PG_dirty 等页标志定义               |


##  PageCache解惑
---

### Page Cache 既是逻辑概念，**也有**实际对应的数据结构

#### 准确的理解应该是：

**Page Cache 是一个有明确源码实体支撑的功能模块**，并非纯粹的逻辑抽象。它由**一组协同工作的数据结构共同构成**，只是没有一个单独叫做 `struct page_cache` 的结构体而已。

------

### 类比理解

就像"进程"这个概念：

```
"进程" 这个概念
    └── 对应源码实体是 struct task_struct
            ├── mm_struct    （内存）
            ├── files_struct （文件）
            ├── signal_struct（信号）
            └── ...
```

Page Cache 也是同理：

```
"Page Cache" 这个概念
    └── 对应源码实体是 struct address_space
            ├── i_pages (XArray)   ← 真实存储缓存页的容器
            ├── struct page/folio  ← 真实的缓存页
            ├── a_ops              ← 操作接口
            └── nrpages            ← 缓存页计数
```

------

### 你可能产生误解的原因

| 误解来源                          | 实际情况                                          |
| --------------------------------- | ------------------------------------------------- |
| 没有 `struct page_cache` 这个名字 | 概念由 `address_space` + `page/folio` 共同承载    |
| "内核缓冲区"听起来很抽象          | 它就是真实分配的物理内存页，由 `struct page` 描述 |
| 多个结构体共同实现                | 这是内核模块化设计，不代表没有实体                |

------

### 一句话总结

> Page Cache / 内核缓冲区 **不是纯逻辑概念**。它是以 `struct address_space`（索引和管理） + `struct page/folio`（实际存储数据的物理内存页）为核心实体，在源码中有清晰对应的**真实内存数据结构**。

你可以认为它是一个**"有具体实现的架构概念"**——概念层面叫 Page Cache，实现层面就是那组数据结构协同工作的结果。
