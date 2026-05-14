## 背景信息
---

因不当操作导致loongnix-desktop无法正常启动，具体原因与initramfs根文件系统有关，由此需要重建Linux启动过程中的根文件系统。

## 操作流程

采用USB制作LiveCD环境进行修复，已知系统安装过程中采用自动分区，分区以及挂载点关系如下：

| 分区      | 挂载点    |
| --------- | --------- |
| /dev/sda1 | /boot/efi |
| /dev/sda2 | /boot     |
| /dev/sda3 | /         |
| /dev/sda4 | backupfs  |
| /dev/sda5 | /data     |
| /dev/sda6 | swap      |

### chroot环境创建

```bash
## 挂载已有分区
sudo mount /dev/sda3 /mnt
sudo mount /dev/sda2 /mnt/boot/
sudo mount /dev/sda1 /mnt/boot/efi
sudo mount /dev/sda4 /mnt/restore
sudo mount /dev/sda5 /mnt/data

## 将 LiveCD 的系统关键目录绑定挂载过去，否则 chroot 后的环境无法识别硬件和进程：

sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run

##进入chroot环境修复

sduo chroot /mnt

sudo dracut -f /boot/initrd.img-4.19.0-19-loongson-3 4.19.0-19-loongson-3

sudo exit
```

> [TIP]
>
> 这里使用dracut是因为chroot环境中无法apt联网使用。
