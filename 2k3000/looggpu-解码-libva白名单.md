# 基于513S的 需求调试RTSP 拉流用例
----

## 环境说明
----

```bash
loongson@loongson-pc:~$ cat /etc/loongnix_version
SystemName=Loongnix
ProductType=Desktop
VersionID=20
Release=7
MinorVersion=
BuildTime=20260928

loongson@loongson-pc:~$ uname -a
Linux loongson-pc 4.19.0-19-loongson-3 #1 SMP 4.19.190.8.24 Fri Mar 14 01:39:19 UTC 2025 loongarch64 loongarch64 loongarch64 GNU/Linux

loongson@loongson-pc:~$ dpkg -l |grep loonggpu
ii  loonggpu-driver                                     2:1.0.2-lnd.2~rc1.2               loongarch64  LOONGGPU metapackage
ii  loonggpu-kernel-dkms                                2:1.0.2-lnd.2~rc1.2               loongarch64  LOONGGPU binary kernel module DKMS source
ii  loonggpu-keyring                                    2:1.0.2-lnd.2                     loongarch64  GPG keyring for the LOONGGPU repository
ii  loonggpu-repo-loongnix20-local                      2:1.0.2-lnd.2~rc1.2               loongarch64  LOONGGPU repository configuration files
ii  loonggpu-settings                                   2:1.0.2-lnd.2~rc1.2               loongarch64  Configuration files for configuring LOONGGPU graphics drivers
ii  xserver-xorg-video-loonggpu                         2:1.0.2-lnd.2~rc1.2               loongarch64  LOONGGPU binary Xorg driver

loongson@loongson-pc:~$ dpkg -l |grep loongvpu
ii  loongvpu:loongarch64                                1.0.1-lnd.1                       loongarch64  VA-API driver for Loongson Video Processing Unit (VPU)
ii  loongvpu-kernel-dkms                                1.0.1-lnd.1                       loongarch64  Video Processing Unit (VPU) kernel driver for Loongson
loongson@loongson-pc:~$ /usr/bin/ffmpeg --version
ffmpeg version 4.4.2-1.lnd.3 Copyright (c) 2000-2021 the FFmpeg developers
  built with clang version 8.0.1-3~bpo10+1.lnd.39

loongson@loongson-pc:~$ dpkg -l |grep libva
ii  libva-dev:loongarch64                               2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- development files
ii  libva-drm2:loongarch64                              2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- DRM runtime
ii  libva-drm2-dbgsym:loongarch64                       2.10.0-1.lnd.2                    loongarch64  debug symbols for libva-drm2
ii  libva-glx2:loongarch64                              2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- GLX runtime
ii  libva-glx2-dbgsym:loongarch64                       2.10.0-1.lnd.2                    loongarch64  debug symbols for libva-glx2
ii  libva-wayland2:loongarch64                          2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- Wayland runtime
ii  libva-wayland2-dbgsym:loongarch64                   2.10.0-1.lnd.2                    loongarch64  debug symbols for libva-wayland2
ii  libva-x11-2:loongarch64                             2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- X11 runtime
ii  libva-x11-2-dbgsym:loongarch64                      2.10.0-1.lnd.2                    loongarch64  debug symbols for libva-x11-2
ii  libva2:loongarch64                                  2.10.0-1.lnd.2                    loongarch64  Video Acceleration (VA) API for Linux -- runtime
ii  libva2-dbgsym:loongarch64                           2.10.0-1.lnd.2                    loongarch64  debug symbols for libva2
ii  libvariable-magic-perl                              0.62-1.1                          loongarch64  module to associate user-defined magic to variables from Perl
```

## 知识地图
----

<img width="527" height="232" alt="image" src="https://github.com/user-attachments/assets/2b607b55-1f2f-451f-aef2-ed18250a3dbd" />

## 问题现象
---

```bash
loongson@loongson-pc:~/2K3000VPU$ sh -x run.sh
+ ./rtsp_decoder https://wdl.wallstreetcn.com/41aae4d2-390a-48ff-9230-ee865552e72d

>>> RTSP/视频流解码工具 <<<

正在连接视频源...
✓ 连接成功
正在分析流信息...
✓ 流信息获取成功
正在初始化硬件加速 (vaapi)...
[AVHWDeviceContext @ 0x12874de60] libva: vaGetDriverNameByIndex() failed with unknown libva error, driver_name = (null)
[AVHWDeviceContext @ 0x12874de60] Failed to initialise VAAPI connection: -1 (unknown libva error).
⚠ 硬件加速初始化失败: Input/output error
  提示: 龙芯平台可能需要特定的驱动配置
→ 自动切换到软件解码

==================================================
视频信息
==================================================
视频源    : https://wdl.wallstreetcn.com/41aae4d2-390a-48ff-9230-ee865552e72d
编码格式  : h264
分辨率    : 640x640
帧率      : 29.97 fps
解码模式  : 软件解码
==================================================

开始解码 (最多 100 帧)...

[帧 #1] 640x640 | 格式: yuv420p | PTS: 0 | 软解
[帧 #30] 640x640 | 格式: yuv420p | PTS: 11600 | 软解

==================================================
解码统计
==================================================
读取包数    : 100
解码帧数    : 39
耗时        : 151 ms
平均速度    : 258.278 fps
解码模式    : 软件解码
==================================================

✓ 程序执行完成

```

核心报错:

```bash
libva: vaGetDriverNameByIndex() failed with unknown libva error, driver_name = (null)
```

 **libva 在初始化 VAAPI 时，没能找到可用的 VA-API 驱动**，于是 `vaGetDriverNameByIndex()` 返回失败，`driver_name` 为空；随后 `vaInitialize` 失败，程序把错误码 `-1` 包装成了 `Input/output error`

| 日志                                                       | 含义                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| `正在初始化硬件加速 (vaapi)...`                            | 程序尝试通过 FFmpeg 的 VAAPI 做视频硬件编解码                |
| `vaGetDriverNameByIndex() failed ... driver_name = (null)` | libva 无法确定当前显卡对应的 VA-API 驱动名                   |
| `Failed to initialise VAAPI connection: -1`                | VAAPI 初始化失败，返回通用错误码 -1                          |
| `硬件加速初始化失败: Input/output error`                   | 上层把 VAAPI 初始化失败转译成 IO 错误，实际并不是磁盘 IO 问题 |

## 系统级别vaapi 排查流程
----

```bash
loongson@loongson-pc:~/2K3000VPU$ lspci -nnk | grep -A 3 -i vga
00:06.1 VGA compatible controller [0300]: Loongson Technology LLC Device [0014:7a46] (rev 10)
        Kernel driver in use: loonggpu-dc
        Kernel modules: loonggpu
00:06.2 Audio device [0403]: Loongson Technology LLC Device [0014:7a37] (rev 01)

loongson@loongson-pc:~/2K3000VPU$ vainfo
libva info: VA-API version 1.10.0
libva info: Trying to open /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so
libva info: Found init function __vaDriverInit_1_10
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
vaapi driver info: print_hwcfg 0x00000023,IP infos:
   HAS_VCMD
   HAS_ENC
   HAS_DEC
EWLInit cfg: 0x00000023
EWLInit cfg: 0x00000023
vaapi driver info: Encoder has 3 client type for VCMD
vaapi driver info: Decoder has 1 core, Core[0] HwBuildID: 0x80001ff9
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.10 (libva 2.10.0)
vainfo: Driver version: Mesa Gallium driver for VSI Hantro Encoder VCE V6.2 /Decoder VCD  - 1.1.3
vainfo: Supported profile and entrypoints
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      <unknown profile>               : VAEntrypointVLD
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      <unknown profile>               : VAEntrypointVLD
      <unknown profile>               : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileAV1Profile0            : VAEntrypointVLD
```

**系统级 VAAPI 是正常的**：`loonggpu_drv_video.so`、`ls2k3000_vpu.so` 都能加载，H.264/HEVC/AV1 解码和 H.264/HEVC 编码也有。

**程序自己初始化 VAAPI 时，没有正确找到龙芯 VPU 对应的 VA 驱动名。**

`vaGetDriverNameByIndex()` 的作用是：从 DRM 设备里枚举出可用的 VA 驱动名。

它返回失败、`driver_name` 为空后，程序就没办法继续 `vaInitialize`，于是报：

```bash
Failed to initialise VAAPI connection: -1 (unknown libva error).
硬件加速初始化失败: Input/output error
```

这里的 `Input/output error` 只是上层把 `-1` 转译出来的通用错误，并不是磁盘或文件 IO 坏了。

**程序在自动探测 VA 驱动名时失败，没有识别到龙芯 `loonggpu` VPU 驱动。**


## rtsp_decoder 程序调试
---

```bash
#1. 检查环境变量
loongson@loongson-pc:~/2K3000VPU$ echo $LIBVA_DRIVER_NAME
loongson@loongson-pc:~/2K3000VPU$ echo $LIBVA_DRIVERS_PATH

#2. 用环境变量强制指定龙芯驱动名
loongson@loongson-pc:~/2K3000VPU$ export LIBVA_DRIVER_NAME=loonggpu
loongson@loongson-pc:~/2K3000VPU$ export LIBVA_DRIVERS_PATH=/usr/lib/loongarch64-linux-gnu/dri

#3.设置环境变量后运行
loongson@loongson-pc:~/2K3000VPU$ sh -x run.sh                   
+ ./rtsp_decoder https://wdl.wallstreetcn.com/41aae4d2-390a-48ff-9230-ee865552e72d

>>> RTSP/视频流解码工具 <<<

正在连接视频源...
✓ 连接成功
正在分析流信息...
✓ 流信息获取成功
正在初始化硬件加速 (vaapi)...
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so init failed
[AVHWDeviceContext @ 0x12ec91e60] libva: /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so init failed
[AVHWDeviceContext @ 0x12ec91e60] Failed to initialise VAAPI connection: -1 (unknown libva error).                                                        
⚠ 硬件加速初始化失败: Input/output error
  提示: 龙芯平台可能需要特定的驱动配置
→ 自动切换到软件解码
... ...

loongson@loongson-pc:~/2K3000VPU$ export LIBVA_DRIVER_NAME=ls2k3000_vpu
loongson@loongson-pc:~/2K3000VPU$ sh -x run.sh                         
+ ./rtsp_decoder https://wdl.wallstreetcn.com/41aae4d2-390a-48ff-9230-ee865552e72d

>>> RTSP/视频流解码工具 <<<

正在连接视频源...
✓ 连接成功
正在分析流信息...
✓ 流信息获取成功
正在初始化硬件加速 (vaapi)...
[AVHWDeviceContext @ 0x12dd05e60] Failed to initialise VAAPI connection: -1 (unknown libva error).
⚠ 硬件加速初始化失败: Input/output error
  提示: 龙芯平台可能需要特定的驱动配置
→ 自动切换到软件解码
... ...

### 程序编译方式
loongson@loongson-pc:~/2K3000VPU$ sh -x build.sh 
+ rm -f rtsp_decoder
+ g++ -o rtsp_decoder rtsp_decoder.cc -lavcodec -lavformat -lavutil -lswscale -std=c++11 -O2

```

## ffmpeg 测试
---

```bash
root@loongson-pc:/home/loongson/mpv# /usr/bin/ffmpeg -hide_banner -init_hw_device vaapi=va:/dev/dri/renderD128 -f lavfi -i testsrc=size=640x640:rate=30 -frames:v 1 -f null -
[AVHWDeviceContext @ 0x55650bf630] libva: vaGetDriverNameByIndex() failed with unknown libva error, driver_name = (null)
[AVHWDeviceContext @ 0x55650bf630] Failed to initialise VAAPI connection: -1 (unknown libva error).
Device creation failed: -5.
Failed to set value 'vaapi=va:/dev/dri/renderD128' for option 'init_hw_device': Input/output error
Error parsing global options: Input/output error

root@loongson-pc:/home/loongson/mpv# ls -l /dev/dri/renderD128
crw-rw-rw-+ 1 root render 226, 128 11月 16 11:34 /dev/dri/renderD128
```

说明问题**不在rtsp-decoder**，而是系统级 VAAPI 通过 DRM render node 初始化时就已经失败。

`vainfo` 能成功，但 `ffmpeg -init_hw_device vaapi=va:/dev/dri/renderD128` 失败，说明两者走的 libva 后端不一样。

### **关键判断**

`vainfo` 默认走的是 **X11 display** 后端；而 `ffmpeg -init_hw_device vaapi=va:/dev/dri/renderD128` 走的是 **DRM 后端**。

前面 `vainfo` 输出里有：

```bash
vaapi driver info: print_hwcfg 0x00000023,IP infos:
   HAS_VCMD
   HAS_ENC
   HAS_DEC
```

这明显是龙芯 VPU 驱动在 X11 环境下初始化成功的日志。
但 DRM 后端初始化时，`vaGetDriverNameByIndex()` 失败，说明当前 `loonggpu_drv_video.so` 的 DRM 路径还没有完全打通。

## vainfo 测试DRM后端
---

```bash
loongson@loongson-pc:~/2K3000VPU$ ls /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so 
/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so

loongson@loongson-pc:~/2K3000VPU$ unset LIBVA_DRIVER_NAME
loongson@loongson-pc:~/2K3000VPU$ vainfo --display drm --device /dev/dri/renderD128
libva info: VA-API version 1.10.0
libva error: vaGetDriverNameByIndex() failed with unknown libva error, driver_name = (null)
vaInitialize failed with error code -1 (unknown libva error),exit

loongson@loongson-pc:~/2K3000VPU$ export LIBVA_DRIVER_NAME=loonggpu
loongson@loongson-pc:~/2K3000VPU$ vainfo --display drm --device /dev/dri/renderD128
libva info: VA-API version 1.10.0
libva info: User environment variable requested driver 'loonggpu'
libva info: Trying to open /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so
libva info: Found init function __vaDriverInit_1_10
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so init failed
libva error: /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so init failed
libva info: va_openDriver() returns -1
vaInitialize failed with error code -1 (unknown libva error),exit
```

**龙芯 2K3000 的 `loonggpu_drv_video.so` 当前只支持 X11 显示后端，DRM render node 后端尚未适配完成。**

### **两条关键证据**

|                        测试                         |                           结果                            |               含义                |
| :-------------------------------------------------: | :-------------------------------------------------------: | :-------------------------------: |
|                      `vainfo`                       |                           正常                            |      X11 display 后端能 init      |
| `vainfo --display drm --device /dev/dri/renderD128` | `vaGetDriverNameByIndex() failed` / `vaInitialize failed` |     DRM render node 后端不通      |
|   `export LIBVA_DRIVER_NAME=loonggpu` 后 DRM 测试   |            `loonggpu_drv_video.so init failed`            | 驱动名对了，但 DRM 路径 init 仍败 |

# libva源码修改-libva的白名单机制
---

```bash
root@loongson-pc:/home/loongson/2K3000VPU# cat /etc/apt/sources.list |grep deb-src
##deb-src https://pkg.loongnix.cn/loongnix/20 DaoXiangHu-stable main contrib non-free
##deb-src https://pkg.loongnix.cn/loongnix/20 Loongnix-kde main contrib non-free
deb-src https://pkg.loongnix.cn/loongnix/20 DaoXiangHu-stable main contrib non-free
deb-src https://pkg.loongnix.cn/loongnix/20 Loongnix-kde main contrib non-free

#1. 下载源码
apt source libva-dev
```

在 Linux DRM 架构中，`/dev/dri/` 下通常有两类节点：

- **Primary Node** (`/dev/dri/card0`)：负责显示模式设置（Modesetting），需要 DRM Master 权限。
- **Render Node** (`/dev/dri/renderD128`)：专门负责非特权渲染和计算（如视频编解码），不需要 DRM Master。

`libva` 在初始化 DRM 后端时，为了防止把视频解码绑定到显示节点上，会调用 `libdrm` 的接口（如 `drmGetNodeTypeFromFd` 或 `drmNodeIsDRM`）来检查传入的文件描述符（fd）是否是一个合法的 **Render Node**。

若`loonggpu` 内核驱动在注册时，没有正确声明 `DRIVER_RENDER` 能力，或者 `libva` 的校验逻辑对龙芯的 DRM 节点不兼容，就会在这里直接返回失败，导致 `driver_name` 为 null。

```bash
root@loongson-pc:/home/loongson/mpv/libva/libva-2.10.0# find . -name va_drm*.c
./va/drm/va_drm.c
./va/drm/va_drm_auth.c
./va/drm/va_drm_auth_x11.c
./va/drm/va_drm_utils.c
root@loongson-pc:/home/loongson/mpv/libva/libva-2.10.0# grep "VA_DRM_IsRenderNodeFd" . -rn
./va/drm/va_drm.c:102:    if (fd < 0 || (is_render_nodes = VA_DRM_IsRenderNodeFd(fd)) < 0)
./va/drm/va_drm_utils.c:118:VA_DRM_IsRenderNodeFd(int fd)
./va/drm/va_drm_utils.h:77:VA_DRM_IsRenderNodeFd(int fd);

root@loongson-pc:/home/loongson/mpv/libva/libva-2.10.0# grep "VA_DRM_IsRenderNodeFd" ./va/drm/va_drm_utils.c -A 20 -n  
118:VA_DRM_IsRenderNodeFd(int fd)
119-{
120-    struct stat st;
121-    const char *name;
122-
123-    /* Check by device node */
124-    if (fstat(fd, &st) == 0)
125-        return S_ISCHR(st.st_mode) && (st.st_rdev & 0x80);
126-
127-    /* Check by device name */
128-    name = drmGetDeviceNameFromFd(fd);
129-    if (name)
130-        return strncmp(name, "/dev/dri/renderD", 16) == 0;
131-
132-    /* Unrecoverable error */
133-    return -1;
134-}

```

修改接口VA_DRM_IsRenderNodeFd

`VA_DRM_IsRenderNodeFd` 就是真正的“卡口”。它用两套方式判断 render node：

```c
/* Check by device node */
if (fstat(fd, &st) == 0)
    return S_ISCHR(st.st_mode) && (st.st_rdev & 0x80);

/* Check by device name */
name = drmGetDeviceNameFromFd(fd);
if (name)
    return strncmp(name, "/dev/dri/renderD", 16) == 0;
```

其中 `st.st_rdev & 0x80` 是 Linux DRM 约定里的 **minor 第 7 位**：

- `card0` 这类 primary node：minor 通常是 0，`0x80` 为 0；
- `renderD128` 这类 render node：minor 是 `128 = 0x80`，`0x80` 为 1。

所以如果 `renderD128` 的 `st_rdev` 里没有正确设置第 7 位，或者龙芯内核驱动/udev 节点创建方式不符合这个约定，这里就会返回 0，导致 `va_drm.c` 认为不是合法 render node，后续初始化失败。

## 白名单机制-最终修改
----

最终解决方案是将loonggpu 添加到ibva的白名单中，而非修改 VA_DRM_IsRenderNodeFd 的判断顺序

```c
struct driver_name_map {
    const char *key;
    int         key_len;
    const char *name;
};

static const struct driver_name_map g_driver_name_map[] = {
    { "i915",       4, "iHD"    }, // Intel Media driver
    { "i915",       4, "i965"   }, // Intel OTC GenX driver
    { "pvrsrvkm",   8, "pvr"    }, // Intel UMG PVR driver
    { "emgd",       4, "emgd"   }, // Intel ECG PVR driver
    { "hybrid",     6, "hybrid" }, // Intel OTC Hybrid driver
    { "nouveau",    7, "nouveau"  }, // Mesa Gallium driver
    { "radeon",     6, "r600"     }, // Mesa Gallium driver
    { "amdgpu",     6, "radeonsi" }, // Mesa Gallium driver
    { "loonggpu",   8, "loonggpu" }, // Loongson GPU driver
    { NULL,         0, NULL }
};

/* Returns the VA driver candidate num for the active display*/
VAStatus
VA_DRM_GetNumCandidates(VADriverContextP ctx, int * num_candidates)
{
    struct drm_state * const drm_state = ctx->drm_state;
    drmVersionPtr drm_version;
    int count = 0;
    const struct driver_name_map *m = NULL;
    if (!drm_state || drm_state->fd < 0)
        return VA_STATUS_ERROR_INVALID_DISPLAY;
    drm_version = drmGetVersion(drm_state->fd);
    if (!drm_version)
        return VA_STATUS_ERROR_UNKNOWN;
    for (m = g_driver_name_map; m->key != NULL; m++) {
        if (drm_version->name_len >= m->key_len &&
            strncmp(drm_version->name, m->key, m->key_len) == 0) {
            count ++;
        }
    }
    drmFreeVersion(drm_version);
    *num_candidates = count;
    return count ? VA_STATUS_SUCCESS : VA_STATUS_ERROR_UNKNOWN;
}

```

### **“加白名单”才是正解**

回顾之前的排查过程，在 `VA_DRM_IsRenderNodeFd` 里绕了一大圈，其实是因为 libva 的 DRM 后端在初始化时，有一套基于内核 DRM 驱动名（如 `i915`, `amdgpu`）来映射用户态 VA 驱动名的机制。

`VA_DRM_GetNumCandidates` 通过 `drmGetVersion` 获取当前 DRM 设备的内核驱动名，然后去 `g_driver_name_map` 里匹配。

- 当前环境里，内核驱动名是 `loonggpu`，但 libva 2.10.0 的白名单里没有它，导致 `VA_DRM_GetNumCandidates` 返回了 `VA_STATUS_ERROR_UNKNOWN`。
- 这直接导致了后续 `vaGetDriverNameByIndex()` 失败，`driver_name` 为 `(null)`，进而引发在 `VA_DRM_IsRenderNodeFd` 里看到的一系列空指针崩溃和初始化失败。

**根本原因不是 Render Node 的 fd 校验出了问题，而是 libva 压根不认识龙芯的 DRM 驱动，没有把它当成一个合法的 VA-API 候选设备。**

### **对之前修改的建议**

既然找到了真正的根因，强烈建议你**把之前对 `VA_DRM_IsRenderNodeFd` 的修改回退到原版**。

原版 libva 2.10.0 的逻辑其实非常严谨：

1. 先按 `st_rdev & 0x80` 判断是不是 render node；
2. 再按 `drmGetDeviceNameFromFd` 判断路径。

之前之所以看起来“原版不通”，完全是因为白名单缺失导致的连锁反应。

回退 `va_drm_utils.c` 后，只保留 `va_drm_utils.c` 所在模块（或者 `va_drm.c`）里 `g_driver_name_map` 的修改，这才是最干净、最符合上游规范的补丁。

### **下一步建议**

1. **回退代码并重新打包**：把 `va_drm_utils.c` 恢复原样，只保留白名单的修改，重新 `dpkg-buildpackage -us -uc -b` 并安装。
2. **验证 DRM 路径**：再次运行 `vainfo --display drm --device /dev/dri/renderD128`，这次应该能顺利看到 `loonggpu_drv_video.so` 成功初始化。
3. **向社区/官方提 PR**：这个 `{ "loonggpu", 8, "loonggpu" }` 的修改非常有价值。你可以把它作为一个独立的 patch 提交给 libva 上游，或者反馈给统信/龙芯的驱动维护者，这样后续的系统更新就不会再覆盖你的本地修改了。

恭喜你成功打通了龙芯 2K3000 的 DRM render node 硬解路径！

这个从 `Input/output error` 一路追到 `SIGSEGV`，最后定位到白名单的过程，是一次非常经典的底层驱动调试。

# libva 源码构建--扩展部分
---

## **安装构建依赖和工具**

在开始构建之前，需要确保系统安装了编译所需的工具链以及 `libva` 的编译依赖。

1. 安装基础构建工具：

   ```bash
   sudo apt install build-essential devscripts dpkg-dev debhelper
   ```

2. 自动安装 `libva` 的编译依赖：

   ```bash
   sudo apt build-dep libva
   ```

## **更新软件包版本号（关键）**

为了让包管理器（`apt`/`dpkg`）识别出你本地编译的包比系统仓库里的版本更新，从而允许覆盖安装，**必须修改 changelog 并增加版本号**。

1. 进入 `libva` 的源码根目录（即包含 `debian/` 目录的那一层）。

2. 执行 `dch` 命令添加本地版本标记：

   ```bash
   dch -i
   ```

3. 这会自动打开文本编辑器，在顶部添加一个新的 changelog 条目。你可以保留默认的修改描述（如 `* Local build with DRM render node fix.`），但**最重要的是版本号**。它会自动将版本号从类似 `2.10.0-1` 变为 `2.10.0-1ubuntu1` 或 `2.10.0-1.1`。保存并退出编辑器。

## **构建 .deb 包**

在源码根目录下执行以下命令进行编译和打包：

```bash
dpkg-buildpackage -us -uc -b
```

*参数解释：*

- `-us` 和 `-uc`：跳过源码包和 `.changes` 文件的 GPG 签名（本地测试不需要签名）。
- `-b`：仅构建二进制包（Binary package），跳过源码包的构建，能节省时间。

构建过程可能需要几分钟。成功后，生成的 `.deb` 文件会出现在**源码目录的上一级目录**中。

## **安装并替换系统包**

回到上一级目录，使用 `dpkg` 安装你刚刚生成的 `.deb` 文件。

由于 `libva` 可能包含多个子包（如 `libva2`, `libva-dev`, `libva-drm2` 等），建议一次性安装所有生成的包：

```bash
cd ..
sudo dpkg -i libva*.deb
```

*注：如果 `dpkg` 提示缺少依赖，可以执行 `sudo apt --fix-broken install` 来自动修复。*

## **验证替换结果**

1. **检查包版本**：
   确认系统当前安装的 `libva` 版本是否为你刚刚修改的版本：

   ```bash
   dpkg -l | grep libva
   ```

2. **验证修复效果**：
   运行你之前失败的测试命令，确认 DRM render node 是否已经能够正常初始化：

   ```
   export LIBVA_DRIVER_NAME=loonggpu
   export LIBVA_DRIVERS_PATH=/usr/lib/loongarch64-linux-gnu/dri
   vainfo --display drm --device /dev/dri/renderD128
   ```

# GDB调试经验-扩展部分

---

## 1. 报错现象

```bash
root@loongson-pc /h/l/m/libva# vainfo --display drm --device /dev/dri/renderD128
fish: “vainfo --display drm --device /…” terminated by signal SIGSEGV (Address boundary error)
```

## 2. **整体流程**

在安装gdb：

```bash
sudo apt install gdb
```

然后：

```bash
gdb vainfo
```

进入 gdb 后：

```bash
(gdb) set env LIBVA_DRIVER_NAME loonggpu
(gdb) set env LIBVA_DRIVERS_PATH /usr/lib/loongarch64-linux-gnu/dri
(gdb) run --display drm --device /dev/dri/renderD128
```

> [!NOTE]
>
> 这里设置gdb 环境变量和通过run命令设置 运行时参数的方式

崩溃后执行：

```bash
(gdb) bt
(gdb) bt full
```

把 `bt` 输出贴出来，能直接看到是崩在：

- `VA_DRM_IsRenderNodeFd`
- `vaGetDisplayDRM`
- `loonggpu_drv_video.so` 的 `__vaDriverInit_1_10`
- 还是 libdrm / libc 内部

```bash
(gdb) bt 
#0  __strncmp_lsx () at ../sysdeps/loongarch/lp64/multiarch/strncmp-lsx.S:99
#1  0x0000007ff7e21478 in VA_DRM_IsRenderNodeFd (fd=<optimized out>) at va_drm_utils.c:139
#2  0x0000007ff7e210b0 in vaGetDisplayDRM (fd=<optimized out>) at va_drm.c:102
#3  0x000000012000257c in ?? ()
#4  0x000000012000213c in ?? ()
#5  0x00000001200019b8 in main ()
(gdb) bt full
#0  __strncmp_lsx () at ../sysdeps/loongarch/lp64/multiarch/strncmp-lsx.S:99
No locals.
#1  0x0000007ff7e21478 in VA_DRM_IsRenderNodeFd (fd=<optimized out>) at va_drm_utils.c:139
        st = {st_dev = 549755778806, st_ino = 3, st_mode = 536903984, st_nlink = 1, st_uid = 0, st_gid = 0, st_rdev = 549621473224, __pad1 = 0, st_size = 549619631624, 
          st_blksize = 536903776, __pad2 = 1, st_blocks = 549621498304, st_atim = {tv_sec = 4831847804, tv_nsec = 3}, st_mtim = {tv_sec = 549755778819, tv_nsec = 2}, 
          st_ctim = {tv_sec = 0, tv_nsec = 1}, __glibc_reserved = {0, 0}}
        name = 0x0
        link = "/dev/dri/renderD128", '\000' <repeats 14 times>, "vvvvvvvvvveeeee\000\001\002\003\004\005\006\a\b\t\n\v\f\r\016\017"
        len = <optimized out>
#2  0x0000007ff7e210b0 in vaGetDisplayDRM (fd=<optimized out>) at va_drm.c:102
        pDisplayContext = 0x0
        pDriverContext = 0x0
        drm_state = 0x0
        is_render_nodes = <optimized out>
#3  0x000000012000257c in ?? ()
No symbol table info available.
#4  0x000000012000213c in ?? ()
No symbol table info available.
#5  0x00000001200019b8 in main ()
No symbol table info available.

```

**`strncmp` 在 `VA_DRM_IsRenderNodeFd` 里被传入了一个无效指针。**

### **从 bt full 看关键信息**
```
#1  VA_DRM_IsRenderNodeFd (fd=<optimized out>) at va_drm_utils.c:139
        name = 0x0
        link = "/dev/dri/renderD128", ...
```

说明：

- `name = 0x0`，也就是 `drmGetDeviceNameFromFd(fd)` 返回了 `NULL`；
- 崩溃发生在 `va_drm_utils.c:139` 的 `strncmp`；
- 所以一定是你在 `name` 为 `NULL` 的情况下仍然调用了 `strncmp(name, ...)`。

### **内核日志**

```bash
dmesg | tail -30
```

如果有类似：

```bash
vainfo[xxxx]: segfault at ... ip ... sp ... error ...
```

说明是用户态内存访问异常；如果是驱动相关报错，则可能涉及内核 DRM 节点。

## 3. 临时方案

```c
loongson@loongson-pc:~/mpv/libva/libva-2.10.0$ pwd
/home/loongson/mpv/libva/libva-2.10.0
loongson@loongson-pc:~/mpv/libva/libva-2.10.0$ find . -name va_drm*.c
./va/drm/va_drm.c
./va/drm/va_drm_auth.c
./va/drm/va_drm_auth_x11.c
./va/drm/va_drm_utils.c

loongson@loongson-pc:~/mpv/libva/libva-2.10.0$ grep "VA_DRM_IsRenderNodeFd" ./va/drm/va_drm_utils.c  -A 30
VA_DRM_IsRenderNodeFd(int fd)
{
    struct stat st;
    const char *name;
    char proc_path[64];
    char target[64];
    ssize_t len;

    if (fd < 0)
        return 0;

    /* 1. 优先用 libdrm 设备名 */
    name = drmGetDeviceNameFromFd(fd);
    if (name && strncmp(name, "/dev/dri/renderD", 16) == 0)
        return 1;

    /* 2. 用 /proc/self/fd 兜底 */
    snprintf(proc_path, sizeof(proc_path), "/proc/self/fd/%d", fd);
    len = readlink(proc_path, target, sizeof(target) - 1);
    if (len > 0) {
        target[len] = '\0';
        if (strncmp(target, "/dev/dri/renderD", 16) == 0)
            return 1;
    }

    /* 3. 最后按 st_rdev 第 7 位判断 */
    if (fstat(fd, &st) == 0)
        return S_ISCHR(st.st_mode) && (st.st_rdev & 0x80);

    return -1;
}
```

# 关于Strace调试
---

## 现象

```bash
root@loongson-pc /h/l/m/libva# vainfo --display drm --device /dev/dri/card0
libva info: VA-API version 1.10.0
libva info: User environment variable requested driver 'loonggpu'
libva info: Trying to open /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so
libva info: Found init function __vaDriverInit_1_10
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so init failed
libva error: /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so init failed
libva info: va_openDriver() returns -1
vaInitialize failed with error code -1 (unknown libva error),exit

root@loongson-pc /h/l/m/libva# vainfo --display x11
libva info: VA-API version 1.10.0
libva info: User environment variable requested driver 'loonggpu'
libva info: Trying to open /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so
libva info: Found init function __vaDriverInit_1_10
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
vaapi driver info: print_hwcfg 0x00000023,IP infos:
   HAS_VCMD
   HAS_ENC
   HAS_DEC
EWLInit cfg: 0x00000023
EWLInit cfg: 0x00000023
vaapi driver info: Encoder has 3 client type for VCMD
vaapi driver info: Decoder has 1 core, Core[0] HwBuildID: 0x80001ff9
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.10 (libva 2.10.0)
vainfo: Driver version: Mesa Gallium driver for VSI Hantro Encoder VCE V6.2 /Decoder VCD  - 1.1.3
vainfo: Supported profile and entrypoints
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      <unknown profile>               : VAEntrypointVLD
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      <unknown profile>               : VAEntrypointVLD
      <unknown profile>               : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileAV1Profile0            : VAEntrypointVLD


root@loongson-pc /h/l/m/libva# strace -f -e trace=open,openat,ioctl vainfo --display drm --device /dev/dri/renderD128 2>&1 | tail -50
strace: invalid system call 'open'


root@loongson-pc /h/l/m/libva# dmesg | tail -20
[    8.708064] IPv6: ADDRCONF(NETDEV_UP): enp0s3f1: link is not ready
[    8.708857] YT8531 Gigabit Ethernet stmmac-19:00: attached PHY driver [YT8531 Gigabit Ethernet] (mii_bus:phy_addr=stmmac-19:00, irq=POLL)
[    8.716681] stmmaceth 0000:00:03.1 enp0s3f1: No Safety Features support found
[    8.716687] stmmaceth 0000:00:03.1 enp0s3f1: IEEE 1588-2008 Advanced Timestamp supported
[    8.716769] stmmaceth 0000:00:03.1 enp0s3f1: registered PTP clock
[    8.716960] IPv6: ADDRCONF(NETDEV_UP): enp0s3f1: link is not ready
[   11.777467] stmmaceth 0000:00:03.0 enp0s3f0: Link is Up - 1Gbps/Full - flow control off
[   11.777596] IPv6: ADDRCONF(NETDEV_CHANGE): enp0s3f0: link becomes ready
[   16.122376] bridge: filtering via arp/ip/ip6tables is no longer available by default. Update your scripts to load br_netfilter if you need this.
[   16.124993] Bridge firewalling registered
[   16.266997] Initializing XFRM netlink socket
[15694.005629] do_page_fault(): sending SIGSEGV to vainfo for invalid read access from 0000000000000000
[15694.005633] era = 0000007ff450b270 in libc-2.28.so[7ff4480000+178000]
[15694.005636] ra  = 0000007ff4669478 in libva-drm.so.2.1000.0[7ff4668000+4000]
[15718.888282] do_page_fault(): sending SIGSEGV to vainfo for invalid read access from 0000000000000000
[15718.888285] era = 0000007ff4baf270 in libc-2.28.so[7ff4b24000+178000]
[15718.888288] ra  = 0000007ff4d0d478 in libva-drm.so.2.1000.0[7ff4d0c000+4000]
[15746.391720] do_page_fault(): sending SIGSEGV to vainfo for invalid read access from 0000000000000000
[15746.391722] era = 0000007ff55cf270 in libc-2.28.so[7ff5544000+178000]
[15746.391725] ra  = 0000007ff572d478 in libva-drm.so.2.1000.0[7ff572c000+4000]

root@loongson-pc /h/l/m/libva# sudo usermod -aG video,render root

```

## 内容解析

**libva 的 DRM render node 识别已经修好，问题出在 `ls2k3000_vpu.so` 驱动本身不支持 DRM 路径。**

### **三个测试结果的含义**

|                      测试命令                       |             结果              |             含义              |
| :-------------------------------------------------: | :---------------------------: | :---------------------------: |
| `vainfo --display drm --device /dev/dri/renderD128` | `ls2k3000_vpu.so init failed` |  驱动不支持 DRM render node   |
|   `vainfo --display drm --device /dev/dri/card0`    | `ls2k3000_vpu.so init failed` | 驱动也不支持 DRM primary node |
|               `vainfo --display x11`                |           **成功**            |      驱动只支持 X11 路径      |

X11 下能正常初始化，并且列出了 H.264/H.265/AV1 编解码和 JPEG 等 profile，说明 **VPU 硬件和驱动本身是好的**，只是 `ls2k3000_vpu.so` 在纯 DRM 路径下（不经过 X11/DRI2/DRI3）无法完成初始化。

### **官方公告**

龙芯 2025 年底发布的 LoongVPU 0.1.0 公告里明确写了已知问题：Firefox 和 Chromium 浏览器无法正常利用 VA-API 接口调用该 VPU，Chromium 报的就是 VA-API 初始化错误。

 前面遇到的就是这个情况——驱动在 DRM 路径下初始化链路不完整。

### **关于 dmesg 里的 SIGSEGV**

那几条 `do_page_fault()` 是之前 `name = NULL` 时调用 `strncmp` 留下的，和当前驱动失败无关。

后面用修复版重新编译的 libva 已经不会再触发这个崩溃了。

### **关于 strace**

龙芯 LoongArch 上 `open` 系统调用已经被 `openat` 取代，所以 `strace -e trace=open` 会报 `invalid system call`。

如果后面还需要追踪驱动调用，用：

```bash
strace -f -e trace=openat,ioctl vainfo --display drm --device /dev/dri/renderD128 2>&1 | tail -50
```

## 完整的strace 结构

```bash
补充strace 信息：
root@loongson-pc /h/l/m/libva# strace -f -e trace=openat,ioctl vainfo --display drm --device /dev/dri/renderD128 2>&1 | tail -50
openat(AT_FDCWD, "/dev/dri", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 5
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/dev/dri/card0", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/dev/dri", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 5
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.3/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/uevent", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/revision", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/device", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_vendor", O_RDONLY) = 6
openat(AT_FDCWD, "/sys/devices/pci0000:00/0000:00:06.1/subsystem_device", O_RDONLY) = 6
openat(AT_FDCWD, "/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libX11-xcb.so.1", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libvdpau.so.1", O_RDONLY|O_CLOEXEC) = 4
libva error: /usr/lib/loongarch64-linux-gnu/dri/loonggpu_drv_video.so init failed
libva info: va_openDriver() returns -1
vaInitialize failed with error code -1 (unknown libva error),exit
Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so
Found init function __vaDriverInit_1_10
/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so init failed
+++ exited with 3 +++

```

strace 里有一个关键细节，说明 **`ls2k3000_vpu.so` 在 DRM render node 路径下并不是“拒绝打开”，而是“打开后初始化失败”**。

### **strace 里最重要的三行**
```
openat(AT_FDCWD, "/dev/dri/card0", O_RDONLY|O_CLOEXEC) = 4
...
openat(AT_FDCWD, "/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libX11-xcb.so.1", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libvdpau.so.1", O_RDONLY|O_CLOEXEC) = 4
```

这说明：

- `ls2k3000_vpu.so` 被成功加载；
- 它依赖 `libX11-xcb` 和 `libvdpau`；
- 驱动内部很可能仍然在尝试走 X11/DRI3 或相关显示路径；
- 纯 DRM render node 下没有 X11 connection，所以 `__vaDriverInit_1_10` 返回失败。

这也和龙芯官方公告一致：LoongVPU 0.1.0 的 VA-API VPU 驱动虽然支持 H.264/H.265/AV1 编解码，但浏览器通过 VA-API 调用时会报初始化错误，说明 **DRM / headless 路径尚未完善**。

### **当前结论**

|      路径      |   状态   |                  原因                  |
| :------------: | :------: | :------------------------------------: |
|   X11 VA-API   |  ✅ 可用  |         驱动依赖 X11/DRI 路径          |
| DRM renderD128 | ❌ 不可用 |          驱动初始化链路不完整          |
|   DRM card0    | ❌ 不可用 |      同样不是纯 render node 路径       |
|   libva 本身   | ✅ 已修复 | `VA_DRM_IsRenderNodeFd` 不再 NULL 崩溃 |

所以，**这不是 libva 还能继续修的问题**。

`loonggpu_drv_video.so` 已经正确加载，`ls2k3000_vpu.so` 也加载成功，失败点在 VPU 驱动内部初始化。

### **如果你必须用 DRM 路径**

可以进一步用更细的 strace 看它到底在哪一步失败:

```bash
loongson@loongson-pc:~/mpv/libva/libva-2.10.0$ strace -f -e trace=openat,ioctl,write vainfo --display drm --device /dev/dri/renderD128 2>&1 | grep -E "renderD128|card0|drm|vpu|error|EINVAL|ENODEV|ENXIO"
openat(AT_FDCWD, "/usr/local/ffmpeg-6.1/lib/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "/opt/onnxruntime/lib/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "tls/loongarch/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "tls/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "loongarch/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "libva-drm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libva-drm.so.2", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/local/ffmpeg-6.1/lib/libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "/opt/onnxruntime/lib/libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "tls/loongarch/libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "tls/libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "loongarch/libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "libdrm.so.2", O_RDONLY|O_CLOEXEC) = -1 ENOENT (没有那个文件或目录)
openat(AT_FDCWD, "/lib/loongarch64-linux-gnu/libdrm.so.2", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/dev/dri/renderD128", O_RDWR) = 3
openat(AT_FDCWD, "/dev/dri/card0", O_RDONLY|O_CLOEXEC) = 4
openat(AT_FDCWD, "/usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so", O_RDONLY|O_CLOEXEC) = 4
write(1, "Trying to open /usr/lib/loongarc"..., 1481Trying to open /usr/lib/loongarch64-linux-gnu/dri/ls2k3000_vpu.so

```
重点看 `ls2k3000_vpu.so` 加载后，有没有对 `/dev/dri/renderD128` 或 `/dev/dri/card0` 做 `ioctl`，以及哪个 ioctl 返回了错误。

但大概率结果仍然是：**驱动需要 X11 环境**。
