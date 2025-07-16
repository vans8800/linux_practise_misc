## 概念
---


在Linux内核中，“bookkeeping scheme（记账机制或记账方案）”是一种通用术语，不是特指某一个模块，而是描述内核如何跟踪和管理资源使用情况的一套机制或方法。

这些“记账”行为在操作系统内部是核心工作之一，确保内核能够正确、及时、安全地进行资源分配、调度和回收。


## 常见的Bookkeeping用途包括
---

|  领域	           |       Bookkeeping内容                                      |
|------------------|-----------------------------------------------------------|
| 内存管理          | 	跟踪页的分配与释放、slab缓存、NUMA节点的使用统计等            |
| 进程管理	         |  跟踪进程创建、状态切换、CPU使用时间、上下文切换次数等          |
| 调度器           | 为公平调度算法（如CFS）记录任务的虚拟运行时间（vruntime）等     |
| 文件系统          | inode使用、缓存的页数、文件读写统计等                          |
| 网络子系统        |	跟踪socket的使用、数据包发送/接收数量、丢包统计等              |
| 时间子系统        |	记录系统运行时间、tick计数、中断次数等                        |
| cgroup控制组      |	跟踪每个cgroup的资源使用情况（CPU、内存、IO等）               |


### 示例：内存子系统中的 Bookkeeping


在内核中，内存并非只是简单的malloc/free，而是有严格的记账机制：

- struct page 结构用于每个物理页的元数据记录；

- vm_area_struct 记录进程虚拟地址区间；

- zone->managed_pages 记录某个内存zone中可管理页数量；

- Slab allocator（如kmalloc）会跟踪每个cache中对象的使用状态。

这些结构和字段共同完成对内存使用情况的详细记账。

### 📌 示例：调度器中的 Bookkeeping

CFS（完全公平调度器）中的调度bookkeeping机制包括：

每个task_struct有一个 vruntime（虚拟运行时间）字段，用于确保CPU时间公平分配；

使用红黑树来记录所有运行队列任务的vruntime；

每次调度都更新运行任务的时间账本。

### 📌 示例：cgroup中的 Bookkeeping

当你使用 cgroup（control groups）限制进程资源使用时，内核会进行详细bookkeeping：

每个cgroup控制器会记录子进程组的CPU时间、内存使用量；

支持以文件形式导出，例如 /sys/fs/cgroup/memory/.../memory.usage_in_bytes；

可用于限制、审计、统计多租户或容器环境中的资源使用。


### 🧠 总结

Bookkeeping Scheme 在 Linux 内核中是一类通用的机制，用于：

“跟踪和维护系统资源状态，以支持调度、限制、审计和优化等功能。”

它是Linux内核高效运行和可靠资源管理的基础，尽管形式多样，但核心思想是一致的。
