# XFS 文件系统 I/O 错误排查与修复实战总结

## 1. 故障现象与初步排查

### 1.1 故障表现

在执行 `xfs_repair` 修复 `/dev/mapper/ao-root` 时，无论是否添加 `-L` 参数，均报错退出：

```
Phase 1 - find and verify superblock...
superblock read failed, offset 0, size 524288, ag 0, rval -1
fatal error -- Input/output error
```

### 1.2 初步排查命令

- `pvs -a`：查看物理卷状态，发现 `/dev/sdb3` 状态为 `ao`。
- `lvs`：查看逻辑卷状态，确认 `ao-root` 存在且大小正常。

### 1.3 尝试重新激活LVM逻辑卷

若底层磁盘硬件检测正常，可能是 LVM 逻辑卷状态异常导致读取失败。

```bash
# 1. 停用逻辑卷
lvchange -an /dev/mapper/ao-root

# 2. 重新激活逻辑卷
lvchange -ay /dev/mapper/ao-root

# 3. 再次尝试只读检查（不要加 -L 参数）
xfs_repair -n /dev/mapper/ao-root
```

尝试修复，并查看<span style="color:red"> **底层物理磁盘和LVM逻辑卷的状态**</span>

```bash
[root@localhost user01]# xfs_repair -n /dev/mapper/ao-root
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 2
        - agno = 1
        - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.

[root@localhost user01]# pvdisplay /dev/sdb3
  --- Physical volume ---
  PV Name               /dev/sdb3
  VG Name               ao
  PV Size               235.47 GiB / not usable 0   
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              60281
  Free PE               0
  Allocated PE          60281
  PV UUID               NByv1I-ALXY-2sbb-JH6t-ZeOr-20a3-0vTZ6q
   
[root@localhost user01]# vgdisplay ao
  --- Volume group ---
  VG Name               ao
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               235.47 GiB
  PE Size               4.00 MiB
  Total PE              60281
  Alloc PE / Size       60281 / 235.47 GiB
  Free  PE / Size       0 / 0   
  VG UUID               QtL0EY-hmXD-40GK-lt2S-pkT2-w3pf-zgw31G
   
```

**底层物理磁盘和 LVM 逻辑卷目前都是健康的**，之前的 `Input/output error` 很可能是由于系统运行时进程占用导致的读取冲突。

#### **1. 状态分析：硬件与 LVM 均正常**

- **物理卷 (PV) 与 卷组 (VG) 健康**：`pvdisplay` 和 `vgdisplay` 均能正常读取元数据，且状态为 `read/write`、`resizable`。这说明 `/dev/sdb3` 磁盘本身没有发生物理损坏，LVM 结构完好。
- **文件系统扫描通过**：`xfs_repair -n`（只读检查模式）成功完成了 7 个阶段的扫描，没有报错退出。这说明文件系统没有发生灾难性的结构损坏。

#### **2. 下一步操作建议**

虽然 `-n` 模式没有报错，但这并不代表文件系统 100% 没有问题（只读模式不会修复任何潜在的逻辑错误）。

为了彻底解决之前的 I/O 错误并修复文件系统，建议执行以下操作：

##### **第一步：卸载文件系统（关键）**

`xfs_repair` 必须在文件系统**未挂载**的状态下才能进行真正的修复。请执行：

```bash
umount /dev/mapper/ao-root
```

*(注：若提示 target is busy，说明有进程正在使用该目录，可以使用 `lsof /dev/mapper/ao-root` 查找并结束相关进程，或者重启进入单用户模式/救援模式进行修复。)*

##### **第二步：执行真正的修复**

卸载成功后，去掉 `-n` 参数执行修复：

```bash
xfs_repair /dev/mapper/ao-root
```

若修复过程中提示日志脏（Dirty log）需要清理，可以加上 `-L` 参数强制清空日志（注意：这可能导致最近未同步的少量数据丢失，但在 I/O 报错的情况下通常是必要的）：

```
xfs_repair -L /dev/mapper/ao-root
```

## **2. 修复结果**

----



```bash
xfs_repair /dev/mapper/ao-root
... ....
... .... 
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
done
```

### **1. 修复结果解读**

- **Phase 5 (重建 AG 头部和树)**：`reset superblock...` 表示修复工具成功重置并重建了超级块，这是解决之前 I/O 错误和文件系统损坏的关键一步。
- **Phase 6 (检查 inode 连通性)**：`moving disconnected inodes to lost+found ...` 表示在修复过程中，发现了一些与主目录树断开连接的文件或目录（孤儿文件）。修复工具已经将它们安全地移动到了 `lost+found` 目录中。
- **Phase 7 (验证并修正链接计数)**：`done` 表示修复顺利完成。



## 4. 完整修复流程串联

----

### 4.1 卸载文件系统

确保目标分区处于未挂载状态：

```bash
umount /dev/mapper/ao-root
```

若提示 `target is busy`，使用 `lsof /dev/mapper/ao-root` 查找占用进程或重启进入救援模式。

### 4.2 执行修复

去掉 `-n` 参数，执行标准修复：

```bash
xfs_repair /dev/mapper/ao-root
```

若提示日志脏（Dirty log）且确认需要清理，可加 `-L`：

```bash
xfs_repair -L /dev/mapper/ao-root
```

### 4.3 修复结果确认

成功修复的日志特征：

- **Phase 5**：`reset superblock...`（超级块重建成功）
- **Phase 6**：`moving disconnected inodes to lost+found ...`（孤儿文件已归档）
- **Phase 7**：`done`（链接计数校验完成）

### 4.4 后续操作

1. **重新挂载**：

   ```bash
   mount /dev/mapper/ao-root /your/mount/point
   ```

2. **检查 lost+found**：

   ```bash
   ls -la /your/mount/point/lost+found
   ```

   检查是否有重要业务文件被断开连接，必要时手动归位。

3. **验证空间**：

   ```bash
   df -h /your/mount/point
   ```

---

## 5. 核心经验总结

1. **I/O 错误先查硬件**：永远不要跳过 `smartctl` 和 `dmesg` 直接修文件系统。
2. **禁止带病强修**：在 I/O 报错时，严禁直接使用 `xfs_repair -L`。
3. **卸载是前提**：`xfs_repair` 必须在 `umount` 后执行，否则极易因设备占用导致误判或修复失败。
4. **只读预检保平安**：养成先跑 `xfs_repair -n` 的习惯，确认结构完整性后再动手修复。
5. **关注 lost+found**：修复后的断连文件是数据完整性的最后一道防线，务必人工复核。
