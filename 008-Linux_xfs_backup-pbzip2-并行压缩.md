
## 背景

如何用tar命令进行多线程并行压缩和解压缩缩大文件？
如何给xfs文件系统扩容？



## 并行压缩和解压缩问题

### 压缩问题
用pbzip2或libzip2 替代原生的bzip2


```bash
tar --use-compress-program=pbzip2 -cpf /run/media/aipc/23debf99-4ffb-4ed4-874b-03935c9f784d/home-aipctar.bz2 --exclude=.cache --exclude=*.tmp aipc/

tar -I pbzip2  -xf home-aipctar.bz2
```

### 其他版本

```bash
tar -cf - /path/to/directory | pbzip2 -p4 > output.tar.bz2
```

-  tar -cf -：将目录打包到标准输出（不生成临时 .tar文件）。
-  pbzip2 -p4：启用4线程压缩（-p后接线程数，按CPU核心数调整）。
-  output.tar.bz2：将结果重定向到目标文件


```bash
tar --use-compress-program=pbzip2 -cf output.tar.bz2 /path/to/directory
```
- lbzip2同样支持多线程



### 解压缩问题

```bash
tar -I pbzip2 -xvf your_archive.tar.bz2
```

### 其他版本

```bash
pbzip2 -d -k -p8 your_file.bz2  
```

**参数说明:**
   -d    解压模式
   -k    保留原始压缩文件
   -p8   使用8个CPU核心 (根据您的CPU内核数调整)


```bash
pbzip2 -dc your_archive.tar.bz2 | tar xvf -
```

## xfs文件系统扩容问题
---


**修改前**

<img width="1223" height="182" alt="image" src="https://github.com/user-attachments/assets/a7be1b56-b62d-4335-92e2-5fca9914bb54" />

主要针对home逻辑卷缩容300GB，然后扩容到root分区，两个分区均为xfs文件系统。

### 备份数据

建议采用tar+pbzip2 并行压缩的和解压缩的方式，其他方式如：

```bash
# 1. 备份 /home 数据到外部存储 (按实际需求选择)
rsync -avh /home/ /backup/         # 备份到本地其他分区
或
tar -czvf home_backup.tar.gz /home  # 打包压缩
```

### 在liveCD环境中卸载/home/分区并检查

```bash
# 2. 强制卸载 /home
umount -f /home

# 3. 检查文件系统一致性
xfs_repair /dev/loongnix-server/home
```
注意这里一定是liveCD 环境或其他OS为主系统

### 缩小/home 逻辑卷

```bash

# 删除原逻辑卷 (操作前确认备份!)
lvremove /dev/loongnix-server/home   # 系统会要求确认 (y)

# 创建缩小后的新 /home 卷 (612.59GB → 312.59GB)
lvcreate -L 312.59G -n home loongnix-server

# 重建 xfs 文件系统

mkfs.xfs /dev/loongnix-server/home

# 挂载新 /home 并恢复数据
mount /dev/loongnix-server/home /home

# 用tar并行解压缩，也可以用下面的方式
rsync -avh /backup/ /home/    # 恢复备份数据

```


### root分区扩容

```bash

# 查看卷组空闲空间 (应有约 300GB)
vgs loongnix-server

# 扩展 /root 逻辑卷
lvextend -L +300G /dev/loongnix-server/root

# 在线扩展 xfs 文件系统 (无需卸载 /)
xfs_growfs /dev/loongnix-server/root
```

### **可能会遇到下面的问题**

<img width="1063" height="84" alt="image" src="https://github.com/user-attachments/assets/fee0167b-6d6c-46dc-a969-d63e0a85720d" />

尽管计划释放300GB空间，但实际执行时发现卷组中只释放了76799个物理扩展单元（PE），而需求是76800个（300GB）。这意味着空间差了一个PE（通常一个PE的大小是4MiB），所以实际上差4MiB。


​​空间计算误差​​lvextend -L +300G需要​​精确76800个PE（物理扩展单元）​​，但卷组只有​​76799个可用PE​​（误差仅4MB，因PE大小通常为4MB）
```bash
Insufficient free space: 76800 extents needed, but only 76799 available
```
​​PE尺寸限制问题​​LVM的最小分配单位是PE（Physical Extent）。当剩余空间比需求少​​哪怕一个PE​​（通常4MB）时，操作就会失败。
​​
vgs命令显示：loongnix-server VFree: <300.00g这个 ​​"<"符号​​ 表示实际可用空间略小于300GB（约299.996GB），无法满足精确的300GB扩容需求。

### **解决方案**

**方案一**

```bash
lvextend -l +76799 /dev/loongnix-server/root # 注意这里是-l（小写L）表示用PE个数
```

**方案二**

```bash
lvextend -l +100%FREE /dev/loongnix-server/root
```
效果：/root 将获得约299.99GB额外空间，误差仅0.003%


**方案三**

```bash
# 首先计算PE大小 (通常4MB)
vgdisplay loongnix-server | grep "PE Size"

# 计算可扩容的精确GB数
可扩容GB = (剩余PE数 × PE大小) 
         = 76799 × 0.004GB ≈ 307.196GB

# 按安全值扩容 (如300GB预留缓冲)
lvextend -L +299G /dev/loongnix-server/root
```
