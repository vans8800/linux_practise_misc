## DKMS vs vermagic

---



### 背景

> [!NOTE]
>
> Linux 内核中dkms是做什么的？
>
> 如何解决基于kernel-5.4 构建出来的网卡驱动(net.ko)，在kernel-6.6 内核环境直接使用，而不去检查Magic number版本的问题



### **1. DKMS 的作用**

DKMS（Dynamic Kernel Module Support，动态内核模块支持）是一个用于在 Linux 操作系统中构建和维护内核模块的框架。

它的核心作用是允许设备驱动的源代码独立于 Linux 内核源代码树存在。

### **2. 解决跨内核版本（5.4 到 6.6）直接使用驱动且绕过 Magic Number 检查的方法**

Linux 内核在加载模块时会进行严格的安全检查，其中 `vermagic` 字符串用于验证模块是否为当前运行的内核编译。

若 `net.ko` 在 5.4 下编译，其 `vermagic` 会包含 "5.4" 的字样，在 6.6 内核下加载时会因版本不匹配而报 "Invalid module format" 错误。

要强制在 6.6 内核下加载 5.4 编译的 `net.ko` 并绕过该检查，有以下三种主要方案：

#### **方案一：使用模块加载参数（最安全、最推荐）**

现代 Linux 内核（如 6.6）在 `finit_module` 系统调用中提供了忽略版本检查的标志。

若你是通过代码或支持该参数的工具加载模块，可以传入 `MODULE_INIT_IGNORE_VERMAGIC` 标志来强制忽略 vermagic 字符串的校验。



#### **方案二：修改模块二进制文件中的 vermagic 字符串（适合手动加载）**

若你必须使用 `insmod` 命令手动加载，可以使用专门的脚本直接修改 `net.ko` 文件中的 vermagic 字符串，使其与 6.6 内核完全一致：

1. 获取当前 6.6 内核的 vermagic 字符串：

   ```bash
   modinfo /lib/modules/$(uname -r)/kernel/drivers/net/xxx.ko | grep vermagic
   ```

2. 使用 `setvermagic.sh` 等工具修改你的驱动：

   ```bash
   ./setvermagic.sh net.ko "6.6.x-xxxxx SMP mod_unload aarch64"
   ```

   这会生成一个修改后的 `net-modified.ko`，此时再使用 `insmod net-modified.ko` 即可绕过版本检查。

#### **方案三：在内核源码层面强制允许加载（不推荐用于生产环境）**

若你的 6.6 内核是自己编译的，可以在内核配置（Kconfig）中开启 `CONFIG_MODULE_FORCE_LOAD` 选项。

开启后，内核将允许加载所有格式不合法或版本不匹配的模块。但这会破坏内核的安全机制，极易导致系统崩溃。

### **⚠️ 严重风险提示**

虽然上述方法可以绕过 Magic Number 的检查，但**强烈不建议**将 5.4 编译的驱动直接用于 6.6 内核。

从 5.4 到 6.6 跨越了十几个大版本，Linux 内核的内部 API、数据结构（如网络栈的 `sk_buff`、设备模型等）发生了大量变更。

即使成功绕过了版本检查强行加载，由于接口不兼容，极大概率会导致**内核崩溃（Kernel Panic）**或**严重的内存破坏**。

正确的做法仍然是使用 DKMS 或手动针对 6.6 内核的头文件重新编译该网卡驱动。

------

## DMKS的用法

---

### **1. 源代码独立于内核树的意义**

允许设备驱动的源代码独立于 Linux 内核源代码树（Out-of-Tree）存在，是 Linux 生态系统能够繁荣发展的核心设计之一。

其重要意义主要体现在以下四个方面：

1. **解耦与独立发布周期**：内核的发布周期是固定的（通常每 2-3 个月一个大版本），而硬件厂商开发新驱动或修复 Bug 的速度远快于此。独立源码允许厂商随时发布驱动更新，无需等待内核官方合并，极大缩短了硬件支持周期。
2. **保护商业机密（闭源驱动）**：Linux 内核主体采用 GPL 协议，要求开源。但许多硬件厂商（如 NVIDIA、某些网卡/显卡厂商）不愿公开其核心算法。独立源码允许厂商以二进制模块（Binary Blob）或专有协议的形式提供驱动，既满足了商业保密需求，又保证了 Linux 系统的可用性。
3. **避免污染主线代码**：内核主线需要保持代码的高质量、通用性和安全性。大量特定硬件的驱动若全部塞入内核树，会导致内核体积膨胀、维护困难。独立源码使得内核树保持精简，同时满足了小众或新硬件的需求。
4. **支持 DKMS 等自动化工具**：正是因为驱动代码独立存在，DKMS 等工具才能在系统升级内核时，自动找到这些源码并针对新内核的头文件进行重新编译，实现驱动的无缝升级。

------

### **2. DKMS 完整配置示例**

假设有一个名为 `net-driver` 的网卡驱动，源码包含 `net.c` 和 `Makefile`。

以下是将其接入 DKMS 的标准流程：

#### **第一步：准备源码目录结构**

在 `/usr/src/` 下创建以 `模块名-版本号` 命名的目录：

```bash
sudo mkdir -p /usr/src/net-driver-1.0.0
```

将你的驱动源码（`net.c`）和构建文件放入该目录。

#### **第二步：编写 DKMS 配置文件**

在 `/usr/src/net-driver-1.0.0/` 目录下创建名为 `dkms.conf` 的文件，内容如下：

```ini
PACKAGE_NAME="net-driver"
PACKAGE_VERSION="1.0.0"
BUILT_MODULE_NAME[0]="net"          # 编译生成的 .ko 文件名（不带后缀）
DEST_MODULE_LOCATION[0]="/kernel/drivers/net" # 安装到内核模块树的路径
AUTOINSTALL="yes"                   # 允许系统升级内核时自动编译
MAKE[0]="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build modules"
CLEAN="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build clean"
```

#### **第三步：注册并安装模块**

执行以下命令将驱动注册到 DKMS 并编译安装：

```bash
# 1. 添加源码到 DKMS 树
sudo dkms add -m net-driver -v 1.0.0

# 2. 针对当前运行的内核进行编译和安装
sudo dkms build -m net-driver -v 1.0.0
sudo dkms install -m net-driver -v 1.0.0

# 或者使用一条命令完成 build 和 install：
# sudo dkms install -m net-driver -v 1.0.0
```

#### **第四步：验证与管理**

```bash
# 查看 DKMS 状态
dkms status

# 卸载模块
sudo dkms remove -m net-driver -v 1.0.0 --all
```

**配置说明**：当系统安装新内核（例如从 6.6 升级到 6.7）时，内核的 post-install 脚本会自动触发 DKMS。

DKMS 会读取 `dkms.conf`，使用新内核的头文件（`${kernel_source_dir}`）重新编译 `net.c`，并将生成的 `net.ko` 放入新内核对应的 `/lib/modules/6.7.x/` 目录下，从而实现驱动的自动平滑过渡。
