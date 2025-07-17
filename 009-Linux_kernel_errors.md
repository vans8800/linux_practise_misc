Linux has four varieties of kernel failure: soft lockups, hard lockups, panics, and the infamous Linux “oops.” 

## 1. Soft Lockup（软锁死）
---

### 概念：

- 软锁死指的是 内核代码长时间占用 CPU 而未让出控制权（比如超过 10 秒）。

- 系统仍有响应（比如网络服务等可能还在工作），但是该 CPU 上的某个核可能“卡住”。

### 常见触发情境：

- 死循环；

- 内核线程阻塞太久；

- 中断处理程序执行时间过长。

### 系统响应：

内核检测到该 CPU 长时间未响应调度器，会通过 watchdog 报告如下警告：

```bash
watchdog: BUG: soft lockup - CPU#0 stuck for 10s!
```

## Hard Lockup（硬锁死）
---

### 概念：

- 比软锁死更严重，指的是 CPU 完全不响应中断（如时钟中断）。

- 系统处于完全冻结状态，无响应，通常需要手动重启。

### 常见触发情境：

- 在中断上下文中死循环；

- 禁用了中断却长时间不打开；

- 内核中使用 cli()/sti() 控制中断失误。

### 系统响应

由硬件 watchdog 或 NMI watchdog 检测并触发警告：

```bash
NMI watchdog: BUG: hard LOCKUP detected on CPU#1
```

## Kernel Panic（内核崩溃）
---

### 概念：

内核遇到致命错误无法恢复时，会主动“自杀”进入 panic 状态。

panic 后系统会停止所有操作，有时会触发重启。

### 常见触发情境:


- 空指针引用；

- 页表错误；

- 显式调用 panic()；

- 内存损坏、设备驱动程序错误等。

### 系统响应：

通常屏幕上会打印大量调试信息（调用栈、寄存器等）：

```bash
Kernel panic - not syncing: Fatal exception
```

## Oops（内核 Oops）

### 概念：

- oops 是内核遇到 非致命错误 时的一种“警告性崩溃”机制。

- 表示有 bug，但系统可以部分继续运行（例如杀死某进程）。

### 常见触发情境

- NULL 指针引用；

- 非法访问用户空间地址；

- 使用释放后的内存等。

### 系统响应

会打印出详细的 oops 日志（包括进程名、调用栈、寄存器状态等）：

```bash
BUG: unable to handle kernel NULL pointer dereference at 00000000
Oops: 0002 [#1] SMP
```


## 总结对比

<img width="1068" height="309" alt="image" src="https://github.com/user-attachments/assets/1abcf19b-6567-49c4-b979-e03f7ee7f6a6" />
