## 背景

---

这里的drm和GPU驱动的关系是什么

> [!NOTE]
>
> 关于如何禁用 LoongGPU 驱动开启 loongsondrm 驱动，下面说法正确的是？（ A ） 
>
> A. 在 grub 界面，系统的启动项上按 e,然后编辑，在 linux 开头的行尾，添加如下命令：
>
>  gsgpu.LG100_support=0 loonggpu.LG100_support=0 loongson.LG100_support=1

 DRM 驱动（`loongson`）与 GPU 驱动（`LoongGPU`/`gsgpu`）是**相互独立但需要协同工作**的关系。

具体关系如下：

1. **硬件架构上的独立性**：在龙芯的硬件设计中，GPU 设备和显示控制器（Display Controller）是两个不同的 PCIe 多功能设备。因此，它们的驱动在软件结构上也是松耦合的3。
2. **功能上的协同性**：虽然驱动相互独立，但它们必须协同工作才能向用户空间提供统一的图形服务。显示控制器负责屏幕输出，而 GPU 负责图形渲染，两者配合完成完整的图形显示链路3。
3. **驱动加载的互斥性（从题目推导）**：从题目给出的禁用 LoongGPU 驱动并开启 `loongsondrm` 驱动的方法可以看出，这两套驱动在处理同一硬件（如 LG100）时可能存在功能重叠或冲突。通过在内核启动参数中分别设置 `gsgpu.LG100_support=0`、`loonggpu.LG100_support=0` 和 `loongson.LG100_support=1`，系统可以精确控制由哪一套驱动来接管和控制相关的硬件功能。

------



> [!note]
>
> LoongGPU 由用户态和内核态共同组成，请选出包含以下哪些组件？(A B C D E F G H I) 
>
> A. 用户态 firmware-loongson-graphics 
>
> B. 用户态 libldrm 
>
> C. 用户态 libloong-gpucomp 
>
> D. 用户态 loonggl 
>
> E. 用户态 loonggpu-driver 
>
> F. 用户态 loonggpu-settings 
>
> G. 用户态 xserver-xorg-video-loonggpu 
>
> H. 用户态 xserver-xorg-core 
>
> I. 内核态 loonggpu-kernel-dkms

LoongGPU 驱动包由用户态和内核态共同组成。以下是这些组件的功能梳理:

| 组件名称                        | 所属层级 | 主要功能/用途                                                |
| :------------------------------ | :------- | :----------------------------------------------------------- |
| **firmware-loongson-graphics**  | 用户态   | 龙芯显卡所需的固件（Firmware）文件，用于硬件初始化及底层功能支持。 |
| **libldrm**                     | 用户态   | 龙芯 DRM（Direct Rendering Manager）相关的用户态库，提供图形渲染管理接口。 |
| **libloong-gpucomp**            | 用户态   | 龙芯 GPU 组件库，可能用于提供 GPU 相关的底层计算或组件协同功能。 |
| **loonggl**                     | 用户态   | 龙芯的 OpenGL 实现库，为应用程序提供标准的 OpenGL 图形渲染 API。 |
| **loonggpu-driver**             | 用户态   | LoongGPU 的驱动包组（或元包），用于一键安装所有相关的驱动组件，简化部署流程。 |
| **loonggpu-settings**           | 用户态   | 龙芯 GPU 的设置工具，用于配置和管理显卡的相关参数。          |
| **xserver-xorg-video-loonggpu** | 用户态   | Xorg 显示服务器的专用 DDX（Device Dependent X）驱动，为 X11 环境提供 2D 加速和屏幕输出支持。 |
| **xserver-xorg-core**           | 用户态   | Xorg 显示服务器的核心组件，是运行图形界面（X11）的基础框架。 |
| **loonggpu-kernel-dkms**        | 内核态   | 基于 DKMS（动态内核模块支持）的内核态驱动模块，使显卡驱动能够随系统内核更新自动重新编译和适配。 |

**总结：**
这套组件构成了一个完整的图形驱动栈。

**内核态**（`loonggpu-kernel-dkms`）负责与硬件直接通信；

**用户态**则涵盖了从底层固件、DRM 库、OpenGL 实现（`loonggl`），到 X11 显示服务（`xserver-xorg-*`）以及配置管理工具（`settings`、`driver`包组）的方方面面，确保操作系统能够正常调用显卡进行图形渲染和桌面显示。

------



### **📊 loongsondrm 驱动组件梳理**

| 组件名称                        | 所属层级 | 主要功能/用途                                                |
| :------------------------------ | :------- | :----------------------------------------------------------- |
| **loongson-drm-dkms**           | 内核态   | 核心的内核模块，基于 DKMS 构建。它负责管理显示控制器（Display Controller），处理模式设置（KMS）和内存管理（GEM），是屏幕能点亮的基础。 |
| **libdrm**                      | 用户态   | 龙芯适配版的通用 DRM 库。它是用户空间程序与内核态 DRM 驱动通信的桥梁，提供底层接口支持。 |
| **libglx-loongson**             | 用户态   | 龙芯实现的 GLX 库。它负责在 X Server 和 OpenGL 之间进行接口转换，让 X11 环境能够调用 OpenGL 进行渲染。 |
| **libegl-loongson**             | 用户态   | 龙芯实现的 EGL 库。用于管理图形上下文、表面和缓冲区的绑定，是 OpenGL ES 等 API 与原生窗口系统之间的接口。 |
| **xserver-xorg-video-loongson** | 用户态   | Xorg 显示服务器的 DDX 驱动。专门为 X11 环境提供 2D 图形加速和显示输出支持，与之前的 `xserver-xorg-video-loonggpu` 对应。 |

### **🧐 核心区别与联系**

为了让你更直观地理解这两者的关系，做一个简单的对比：

| 特性            | LoongGPU 驱动栈                        | loongsondrm 驱动栈                                   |
| :-------------- | :------------------------------------- | :--------------------------------------------------- |
| **核心定位**    | 侧重于 **GPU 渲染** 与显示控制的结合   | 侧重于 **显示控制器** 管理，兼容标准 DRM 架构        |
| **内核模块**    | `loonggpu-kernel-dkms`                 | `loongson-drm-dkms`                                  |
| **OpenGL 实现** | `loonggl` (自研/特定实现)              | 通常配合标准的 Mesa 或特定的 `libglx/egl-loongson`   |
| **Xorg 驱动**   | `xserver-xorg-video-loonggpu`          | `xserver-xorg-video-loongson`                        |
| **适用场景**    | 早期或特定的 LG100 等 GPU 硬件加速场景 | 较新的架构，强调与上游 Linux 内核 DRM 子系统的兼容性 |

**总结来说：**

- **LoongGPU** 更像是一个包含固件、专用 OpenGL 库和内核模块的“全家桶”式私有驱动方案。
- **loongsondrm** 则更趋向于标准的 Linux 开源驱动模式（类似 i915 或 amdgpu），将显示控制（DRM/KMS）与渲染（通常交给 Mesa）解耦得更清晰。

通过参数 `loongson.LG100_support=1` 来切换驱动，因为虽然它们服务的硬件可能相同（如 LG100），但调用的内核模块和上层库是完全不同的两套体系。

## 使用的loongGPU 还是longsonDRM驱动栈 

----

在 Debian 系统下，要确定当前使用的是 `loongGPU` 还是 `loongsonDRM` 驱动栈，可以通过以下几个层面的命令进行排查。

核心思路是检查**内核模块**、**Xorg 驱动**以及**用户态库**。

### **1. 检查内核态驱动（最核心）**

由于这两套驱动栈的内核模块名称完全不同，这是最直接的判断方法：

```bash
lsmod | grep -E "loonggpu|loongson"
```

- 若输出中包含 `loonggpu` 相关模块（如 `loonggpu_ko`），说明使用的是 **LoongGPU 驱动栈**。
- 若输出中包含 `loongson` 相关模块（如 `loongson_drm`），说明使用的是 **loongsonDRM 驱动栈**。

### **2. 检查 Xorg 显示服务器驱动**

查看 X 服务器当前加载的 DDX 驱动模块：

```bash
lspci -k | grep -A3 VGA
```

或者直接查看 Xorg 日志：

```bash
grep -i "driver" /var/log/Xorg.0.log
```

- 若看到加载的是 `xserver-xorg-video-loonggpu`，则为 **LoongGPU**。
- 若看到加载的是 `xserver-xorg-video-loongson`，则为 **loongsonDRM**。

### **3. 检查用户态包安装情况**

通过 Debian 的包管理工具 `dpkg` 检查系统中安装了哪套用户态库：

```bash
dpkg -l | grep -E "loonggpu|loongson-drm|libldrm|libglx-loongson"
```

- 若存在 `libldrm`、`loonggl`、`loonggpu-driver` 等包，属于 **LoongGPU 栈**。
- 若存在 `libglx-loongson`、`libegl-loongson`、`loongson-drm-dkms` 等包，属于 **loongsonDRM 栈**。

### **4. 检查内核启动参数**

若系统是通过 GRUB 参数来切换和指定驱动的，可以查看当前的内核命令行：

```bash
cat /proc/cmdline
```

- 若参数中包含 `loonggpu.LG100_support=1`，则强制启用了 **LoongGPU**。
- 若参数中包含 `loongson.LG100_support=1` 且 `loonggpu.LG100_support=0`，则启用了 **loongsonDRM**。

### **5. 检查 OpenGL 渲染器（验证实际生效情况）**

若系统有图形界面，可以查看当前实际用于 3D 渲染的驱动：

```bash
glxinfo | grep "OpenGL renderer"
```

*(注：需要安装 `mesa-utils` 包)*
输出的渲染器名称通常会直接暴露底层使用的是哪套驱动架构。

**总结建议**：
最快的方法是直接运行 `lsmod | grep -E "loonggpu|loongson"` 和 `cat /proc/cmdline`，这两步基本就能准确判定当前系统使用的是哪一套驱动栈。

---

## GPU栈切换

---

在 Debian 系统下，从 `LoongGPU` 切换到 `loongsonDRM` 驱动栈的核心操作是**修改 GRUB 启动参数**。

以下是具体的操作步骤：

### **1. 临时测试（推荐先做这一步）**

在确认切换后系统能正常进入图形界面前，建议先通过临时修改参数进行测试，避免改错导致黑屏无法开机：

1. 重启系统，在 GRUB 引导菜单界面，选中你要启动的 Debian 系统，按下 **`e`** 键进入编辑模式。

2. 找到以 `linux` 开头的那一行。

3. 在该行的**末尾**（确保与前面的内容有空格），添加以下参数：

   ```bash
   gsgpu.LG100_support=0 loonggpu.LG100_support=0 loongson.LG100_support=1
   ```

   *(注：这三行参数的作用是彻底禁用原有的 GPU 驱动，并强制启用 loongson DRM 驱动)*

4. 按下 **`Ctrl + X`** 或 **`F10`** 保存并引导系统。

5. 观察是否能正常进入桌面，并使用我们之前提到的命令（如 `lsmod | grep loongson`）验证是否生效。

### **2. 永久生效（测试成功后执行）**

若临时测试一切正常，就可以将其写入配置文件使其永久生效：

1. 打开终端，使用 root 权限编辑 GRUB 配置文件：

   ```bash
   sudo nano /etc/default/grub
   ```

2. 找到 `GRUB_CMDLINE_LINUX_DEFAULT` 这一行，在引号内的末尾追加上述参数。例如：

   ```bash
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash gsgpu.LG100_support=0 loonggpu.LG100_support=0 loongson.LG100_support=1"
   ```

3. 按 `Ctrl + O` 保存，`Ctrl + X` 退出。

4. **非常重要**：更新 GRUB 配置使其生效：

   ```bash
   sudo update-grub
   ```

5. 重启系统：

   ```bash
   sudo reboot
   ```

### **⚠️ 注意事项**

- **包依赖**：切换驱动栈前，请确保你已经安装了 `loongsonDRM` 相关的用户态包（如 `libdrm`、`libglx-loongson` 等）。若内核加载了 DRM 驱动但缺少用户态库，图形界面可能会无法启动。
- **回退方案**：若切换后遇到黑屏或花屏，可以在 GRUB 编辑界面将参数改回（如 `loonggpu.LG100_support=1 loongson.LG100_support=0`），即可恢复原来的驱动。
