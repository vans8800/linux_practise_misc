## 背景信息

针对某个由Cmake作为构建工具的开源项目，如OpenCV在构建时需要厘清与特定架构相关的优化选项，比如下面的情况：

```bash
loongson@loongson-pc:/usr/local/opencv4.10/build$ pwd
/usr/local/opencv4.10/build
loongson@loongson-pc:/usr/local/opencv4.10/build$ cat build.sh
cmake .. -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/usr/local/opencv410 \
    -D CPU_BASELINE=LSX \
    -D CPU_DISPATCH=LASX \
    -D BUILD_opencv_python3=ON \
    -D PYTHON3_EXECUTABLE=$(which python3) \
    -D WITH_TBB=OFF \
    -D WITH_AVIF=OFF \
    -D WITH_OPENMP=ON \
    -D WITH_EIGEN=ON \
    -D ENABLE_FAST_MATH=ON \
    -D WITH_LAPACK=ON \
    -D LAPACK_IMPL=OpenBLAS \
    -D BUILD_TESTS=OFF \
    -D BUILD_PERF_TESTS=OFF \
    -D BUILD_EXAMPLES=OFF \
    -D OPENCV_GENERATE_PKGCONFIG=ON \

```

## 常用工具
---

### camke的命令行选项

首次配置，至少要运行一次CMake来生成缓存文件CMakeCache.txt. 
```bash
mkdir build && cd build
cmake ..
```


# OpenCV CMake 编译选项完整说明

## 基础配置选项

| 选项                      | 类型   | 说明                             | 示例值                         |
| ------------------------- | ------ | -------------------------------- | ------------------------------ |
| CMAKE_BUILD_TYPE          | STRING | 构建类型，影响优化级别和调试信息 | RELEASE, Debug, RelWithDebInfo |
| CMAKE_INSTALL_PREFIX      | PATH   | 安装路径前缀                     | /usr/local, /opt/opencv        |
| CMAKE_CXX_FLAGS           | STRING | C++ 编译器额外标志               | "-march=native -O3"            |
| CMAKE_C_FLAGS             | STRING | C 编译器额外标志                 | "-march=native -O3"            |
| OPENCV_GENERATE_PKGCONFIG | BOOL   | 生成 pkg-config 文件（.pc）      | ON, OFF                        |

## Python Bindings 相关

| 选项                       | 类型 | 说明                | 示例值                   |
| -------------------------- | ---- | ------------------- | ------------------------ |
| BUILD_opencv_python3       | BOOL | 构建 Python 3 绑定  | ON, OFF                  |
| PYTHON3_EXECUTABLE         | PATH | Python 3 解释器路径 | /usr/bin/python3         |
| PYTHON3_INCLUDE_DIR        | PATH | Python 3 头文件目录 | /usr/include/python3.8   |
| PYTHON3_LIBRARY            | PATH | Python 3 库文件路径 | /usr/lib/libpython3.8.so |
| PYTHON3_NUMPY_INCLUDE_DIRS | PATH | NumPy 头文件目录    | .../numpy/core/include   |
| PYTHON3_PACKAGES_PATH      | PATH | Python 包安装路径   | .../site-packages        |

## CPU 架构优化（通用）

| 选项                 | 类型   | 说明                               | 示例值                          |
| -------------------- | ------ | ---------------------------------- | ------------------------------- |
| CPU_BASELINE         | STRING | 基准指令集（所有目标机器必须支持） | SSE4_2, AVX2, NEON, LSX, DETECT |
| CPU_DISPATCH         | STRING | 运行时分发指令集（可选优化）       | AVX,AVX2,AVX512_SKX / LASX      |
| CV_ENABLE_INTRINSICS | BOOL   | 启用 SIMD 内联汇编优化             | ON, OFF                         |
| ENABLE_FAST_MATH     | BOOL   | 启用快速数学运算（可能降低精度）   | ON, OFF                         |

## x86/x64 向量指令集优化

| 选项              | 类型 | 说明                        | 适用架构 |
| ----------------- | ---- | --------------------------- | -------- |
| ENABLE_SSE        | BOOL | 启用 SSE 指令集             | x86/x64  |
| ENABLE_SSE2       | BOOL | 启用 SSE2 指令集            | x86/x64  |
| ENABLE_SSE3       | BOOL | 启用 SSE3 指令集            | x86/x64  |
| ENABLE_SSSE3      | BOOL | 启用 SSSE3 指令集           | x86/x64  |
| ENABLE_SSE41      | BOOL | 启用 SSE4.1 指令集          | x86/x64  |
| ENABLE_SSE42      | BOOL | 启用 SSE4.2 指令集          | x86/x64  |
| ENABLE_AVX        | BOOL | 启用 AVX 指令集             | x86/x64  |
| ENABLE_AVX2       | BOOL | 启用 AVX2 指令集            | x86/x64  |
| ENABLE_AVX512F    | BOOL | 启用 AVX-512 Foundation     | x86/x64  |
| ENABLE_AVX512_SKX | BOOL | 启用 AVX-512 Skylake-X 扩展 | x86/x64  |
| ENABLE_FMA3       | BOOL | 启用 FMA3（融合乘加）指令   | x86/x64  |
| ENABLE_POPCNT     | BOOL | 启用 POPCNT 指令（位计数）  | x86/x64  |

## ARM 架构优化

| 选项         | 类型 | 说明                 | 适用架构  |
| ------------ | ---- | -------------------- | --------- |
| ENABLE_NEON  | BOOL | 启用 ARM NEON 指令集 | ARM/ARM64 |
| ENABLE_VFPV3 | BOOL | 启用 VFPv3 浮点指令  | ARM       |

## LoongArch 架构优化

| 选项              | 类型   | 说明                              | 适用架构    |
| ----------------- | ------ | --------------------------------- | ----------- |
| CPU_BASELINE=LSX  | STRING | 使用 128-bit LSX 向量指令作为基准 | LoongArch64 |
| CPU_DISPATCH=LASX | STRING | 运行时分发 256-bit LASX 向量指令  | LoongArch64 |

## 第三方库和加速

| 选项        | 类型 | 说明                           | 用途            |
| ----------- | ---- | ------------------------------ | --------------- |
| WITH_TBB    | BOOL | 启用 Intel TBB（线程构建块）   | 多线程并行      |
| WITH_OPENMP | BOOL | 启用 OpenMP 多线程             | 多线程并行      |
| WITH_EIGEN  | BOOL | 启用 Eigen 数学库              | 线性代数优化    |
| WITH_IPP    | BOOL | 启用 Intel IPP（集成性能原语） | Intel CPU 加速  |
| WITH_CUDA   | BOOL | 启用 CUDA GPU 加速             | NVIDIA GPU 计算 |
| WITH_OPENCL | BOOL | 启用 OpenCL GPU 加速           | 通用 GPU 计算   |
| WITH_LAPACK | BOOL | 启用 LAPACK 线性代数库         | 高级线性代数    |

## LAPACK/BLAS 配置

| 选项                 | 类型   | 说明                | 示例值                        |
| -------------------- | ------ | ------------------- | ----------------------------- |
| LAPACK_IMPL          | STRING | 指定 LAPACK 实现    | OpenBLAS, ATLAS, MKL          |
| LAPACK_LIBRARIES     | PATH   | LAPACK 库文件路径   | /usr/lib/.../libopenblas.so   |
| LAPACK_CBLAS_H       | PATH   | CBLAS 头文件路径    | /usr/include/.../cblas.h      |
| LAPACK_LAPACKE_H     | PATH   | LAPACKE 头文件路径  | /usr/include/.../lapacke.h    |
| BLA_VENDOR           | STRING | BLAS 供应商         | OpenBLAS, ATLAS, Intel10_64lp |
| OpenBLAS_INCLUDE_DIR | PATH   | OpenBLAS 头文件目录 | /usr/include                  |
| OpenBLAS_LIB         | PATH   | OpenBLAS 库文件     | /usr/lib/libopenblas.so       |

## 图像/视频库支持

| 选项           | 类型 | 说明                    | 用途          |
| -------------- | ---- | ----------------------- | ------------- |
| WITH_JPEG      | BOOL | 启用 JPEG 支持          | JPEG 图像读写 |
| WITH_PNG       | BOOL | 启用 PNG 支持           | PNG 图像读写  |
| WITH_TIFF      | BOOL | 启用 TIFF 支持          | TIFF 图像读写 |
| WITH_WEBP      | BOOL | 启用 WebP 支持          | WebP 图像读写 |
| WITH_FFMPEG    | BOOL | 启用 FFmpeg 视频编解码  | 视频处理      |
| WITH_GSTREAMER | BOOL | 启用 GStreamer 视频支持 | 视频流处理    |
| WITH_V4L       | BOOL | 启用 Video4Linux 支持   | Linux 摄像头  |

## GUI 和显示

| 选项         | 类型 | 说明                  | 平台    |
| ------------ | ---- | --------------------- | ------- |
| WITH_GTK     | BOOL | 启用 GTK+ GUI 支持    | Linux   |
| WITH_QT      | BOOL | 启用 Qt GUI 支持      | 跨平台  |
| WITH_WIN32UI | BOOL | 启用 Windows 原生 GUI | Windows |
| WITH_COCOA   | BOOL | 启用 Cocoa GUI        | macOS   |

## 扩展模块

| 选项                      | 类型 | 说明                        | 内容           |
| ------------------------- | ---- | --------------------------- | -------------- |
| OPENCV_EXTRA_MODULES_PATH | PATH | opencv_contrib 扩展模块路径 | 额外算法和功能 |
| OPENCV_ENABLE_NONFREE     | BOOL | 启用非自由（专利）算法      | SIFT, SURF 等  |

## 构建选项

| 选项                    | 类型 | 说明                   | 建议           |
| ----------------------- | ---- | ---------------------- | -------------- |
| BUILD_SHARED_LIBS       | BOOL | 构建共享库（.so/.dll） | ON（默认）     |
| BUILD_TESTS             | BOOL | 构建单元测试           | OFF（发布版）  |
| BUILD_PERF_TESTS        | BOOL | 构建性能测试           | OFF（发布版）  |
| BUILD_EXAMPLES          | BOOL | 构建示例代码           | OFF（发布版）  |
| BUILD_opencv_world      | BOOL | 构建单一合并库文件     | ON（加速链接） |
| INSTALL_PYTHON_EXAMPLES | BOOL | 安装 Python 示例       | OFF            |
| INSTALL_C_EXAMPLES      | BOOL | 安装 C/C++ 示例        | OFF            |

## 调试和开发

| 选项                       | 类型 | 说明               | 用途     |
| -------------------------- | ---- | ------------------ | -------- |
| ENABLE_PROFILING           | BOOL | 启用性能分析       | 开发调试 |
| ENABLE_COVERAGE            | BOOL | 启用代码覆盖率分析 | 测试     |
| OPENCV_WARNINGS_ARE_ERRORS | BOOL | 将警告视为错误     | 严格编译 |

## 特定平台选项

### Android

| 选项                     | 类型   | 说明                                       |
| ------------------------ | ------ | ------------------------------------------ |
| ANDROID_ABI              | STRING | Android ABI 类型（arm64-v8a, armeabi-v7a） |
| ANDROID_NATIVE_API_LEVEL | INT    | Android API 级别                           |

### iOS

| 选项     | 类型   | 说明                     |
| -------- | ------ | ------------------------ |
| IOS_ARCH | STRING | iOS 架构（arm64, armv7） |

## 环境变量（运行时控制）

| 变量                 | 说明                    | 示例                             |
| -------------------- | ----------------------- | -------------------------------- |
| OPENCV_CPU_DISABLE   | 运行时禁用特定 CPU 优化 | OPENCV_CPU_DISABLE=AVX2,AVX512F  |
| OPENCV_OPENCL_DEVICE | 指定 OpenCL 设备        | OPENCV_OPENCL_DEVICE=Intel:GPU:0 |

## 常用配置组合

### 通用桌面（兼容性优先）

```bash
-D CMAKE_BUILD_TYPE=RELEASE
-D CPU_BASELINE=SSE4_2
-D CPU_DISPATCH=AVX,AVX2
-D WITH_TBB=ON
-D WITH_OPENMP=ON
-D BUILD_opencv_python3=ON
```

### 高性能（新硬件）

```bash
-D CMAKE_BUILD_TYPE=RELEASE
-D CPU_BASELINE=AVX2
-D CPU_DISPATCH=AVX512_SKX
-D ENABLE_FAST_MATH=ON
-D WITH_IPP=ON
-D WITH_TBB=ON
```

### LoongArch64 龙芯

```bash
-D CMAKE_BUILD_TYPE=RELEASE
-D CPU_BASELINE=LSX
-D CPU_DISPATCH=LASX
-D WITH_TBB=ON
-D WITH_OPENMP=ON
```

### ARM 设备

```bash
-D CMAKE_BUILD_TYPE=RELEASE
-D CPU_BASELINE=NEON
-D ENABLE_NEON=ON
-D WITH_TBB=ON
```

### 最小化构建

```bash
-D CMAKE_BUILD_TYPE=RELEASE
-D BUILD_TESTS=OFF
-D BUILD_PERF_TESTS=OFF
-D BUILD_EXAMPLES=OFF
-D WITH_FFMPEG=OFF
-D WITH_GTK=OFF
```

## CMake 查询命令详解

### cmake 命令行选项

| 选项 | 完整形式                   | 说明                             | 示例              |
| ---- | -------------------------- | -------------------------------- | ----------------- |
| -L   | --list-cache               | 列出缓存变量（cache variables）  | `cmake -L ..`     |
| -LA  | --list-cache-advanced      | 列出所有缓存变量（包括高级选项） | `cmake -LA ..`    |
| -LAH | --list-cache-advanced-help | 列出所有缓存变量及其帮助文档     | `cmake -LAH ..`   |
| -N   | --no-warn-unused-cli       | 不配置，仅列出变量               | `cmake -N -LA ..` |

#### 详细说明

**-L (List cache)**

- 只显示非高级（非 ADVANCED）的 CMake 缓存变量
- 输出格式：`VARIABLE_NAME:TYPE=VALUE`
- 适合查看基本配置选项

**-LA (List all cache)**

- 显示所有缓存变量，包括标记为 ADVANCED 的选项
- ADVANCED 选项通常是高级配置，普通用户不常修改
- 输出变量数量远多于 -L

**-LAH (List all cache with help)**

- 显示所有缓存变量及其帮助文档说明
- 每个变量会附带描述信息
- 最详细的查询方式，适合探索可用选项

#### 使用示例

```bash
# 基础变量列表
cmake -L .. | grep PYTHON

# 所有变量（包括高级）
cmake -LA .. | grep -i "cpu\|simd"

# 带说明文档（最详细）
cmake -LAH .. | grep -A 2 "CMAKE_BUILD_TYPE"

# 只列出变量，不执行配置
cmake -N -LAH .. > cmake_options.txt

# 按类别筛选
cmake -LAH .. | grep -E "^(BUILD_|WITH_|ENABLE_)" | less

# 保存完整配置信息
cmake -LAH .. > full_options.txt 2>&1
```

#### 输出格式示例

```
// 标准格式
CMAKE_BUILD_TYPE:STRING=RELEASE

// 带帮助文档的格式 (-H)
// Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel ...
CMAKE_BUILD_TYPE:STRING=RELEASE

// Bool 类型
BUILD_EXAMPLES:BOOL=OFF

// 路径类型
CMAKE_INSTALL_PREFIX:PATH=/usr/local
```

### ccmake 交互式配置工具

**ccmake** 是 CMake 的文本用户界面（TUI），提供交互式配置体验。

#### 基本用法

```bash
# 在构建目录中启动
cd opencv/build
ccmake ..

# 或直接指定源码目录
ccmake /path/to/opencv
```

#### 常用快捷键

| 快捷键         | 功能              | 说明                     |
| -------------- | ----------------- | ------------------------ |
| `[Enter]`      | 编辑选项          | 修改当前高亮的选项值     |
| `[t]`          | 切换高级模式      | 显示/隐藏 ADVANCED 选项  |
| `[c]`          | 配置（Configure） | 执行配置，检测系统环境   |
| `[g]`          | 生成（Generate）  | 生成构建文件并退出       |
| `[q]`          | 退出              | 不保存退出（配置前）     |
| `[h]`          | 帮助              | 显示帮助信息             |
| `[/]`          | 搜索              | 搜索选项（部分版本支持） |
| `↑/↓`          | 导航              | 上下移动光标             |
| `Page Up/Down` | 翻页              | 快速浏览                 |
| `[d]`          | 删除缓存条目      | 删除选中的缓存变量       |

#### 界面布局

```
┌─────────────────────────────────────────────────────────┐
│ CMake Version X.XX.X                                    │
│ Press [enter] to edit option                            │
│ Press [c] to configure   Press [g] to generate and exit│
│ Press [t] to toggle advanced mode (Currently Off)      │
│ Press [q] to quit without generating                   │
├─────────────────────────────────────────────────────────┤
│ BUILD_EXAMPLES           *OFF                           │
│ BUILD_opencv_python3     *ON                            │
│ CMAKE_BUILD_TYPE         *RELEASE                       │
│ CMAKE_INSTALL_PREFIX     */usr/local                    │
│ CPU_BASELINE             *SSE4_2                        │
│ ...                                                     │
└─────────────────────────────────────────────────────────┘
```

#### 使用流程

1. **启动 ccmake**

   ```bash
   cd opencv/build
   ccmake ..
   ```

2. **首次配置**

   - 按 `[c]` 执行配置
   - CMake 会检测系统环境
   - 等待配置完成

3. **查看/修改选项**

   - 按 `[t]` 显示高级选项
   - 用 `↑/↓` 浏览
   - 按 `[Enter]` 编辑

4. **编辑选项**

   - BOOL 类型：按 `[Enter]` 切换 ON/OFF
   - STRING 类型：输入新值后按 `[Enter]`
   - PATH 类型：输入路径或按 `[Tab]` 自动补全

5. **重新配置**

   - 修改后按 `[c]` 重新配置
   - 可能需要多次配置直到没有新选项出现

6. **生成构建文件**

   - 按 `[g]` 生成 Makefile 并退出
   - 只有配置无错误时才能生成

#### 选项标记说明

| 标记         | 含义                        |
| ------------ | --------------------------- |
| `*`          | 已修改的选项                |
| `(空)`       | 默认值                      |
| `(ADVANCED)` | 高级选项（按 [t] 切换显示） |

#### 常见选项分组

在 ccmake 中，选项按字母顺序排列，常见前缀：

- `BUILD_*` - 控制构建内容（测试、示例、模块等）
- `CMAKE_*` - CMake 标准选项（构建类型、安装路径等）
- `CPU_*` - CPU 架构优化选项
- `ENABLE_*` - 启用特定特性（SIMD 指令集等）
- `WITH_*` - 启用第三方库支持
- `OPENCV_*` - OpenCV 特定配置
- `PYTHON*` - Python 绑定相关
- `INSTALL_*` - 安装选项

#### ccmake 命令行选项

```bash
# 基本用法
ccmake [options] <source-dir>

# 常用选项
ccmake -C <initial-cache-file> ..    # 加载预设缓存文件
ccmake -D<var>=<value> ..            # 设置初始变量
ccmake -G <generator> ..             # 指定生成器
ccmake --help                        # 显示帮助
```

### cmake-gui 图形界面工具

如果你更喜欢图形界面，可以使用 `cmake-gui`（需要单独安装）：

```bash
# 安装 cmake-gui
sudo apt-get install cmake-qt-gui  # Ubuntu/Debian
brew install --cask cmake          # macOS

# 启动
cmake-gui
# 或指定源码目录
cmake-gui /path/to/opencv
```

**cmake-gui 优势**：

- 更直观的图形界面
- 支持搜索过滤
- 可按组分类显示
- 支持鼠标操作
- 实时帮助文档显示

### 实用查询脚本

```bash
#!/bin/bash
# 保存为 query_cmake_options.sh

echo "=== OpenCV CMake 配置选项查询 ==="
echo ""

cd opencv/build

echo "1. Python 相关选项:"
cmake -LAH .. 2>/dev/null | grep -A 1 "PYTHON"
echo ""

echo "2. CPU 优化选项:"
cmake -LAH .. 2>/dev/null | grep -A 1 -E "(CPU_|ENABLE_.*SSE|ENABLE_.*AVX|ENABLE_.*NEON)"
echo ""

echo "3. 第三方库:"
cmake -LAH .. 2>/dev/null | grep -A 1 "^WITH_"
echo ""

echo "4. 构建选项:"
cmake -LAH .. 2>/dev/null | grep -A 1 "^BUILD_"
echo ""

echo "5. 当前配置摘要:"
cmake -LA .. 2>/dev/null | grep -E "CMAKE_BUILD_TYPE|CMAKE_INSTALL_PREFIX|CPU_BASELINE|CPU_DISPATCH"
```

### 快速参考卡片

```bash
# 快速查看所有选项
cmake -LAH .. | less

# 搜索特定关键词
cmake -LAH .. | grep -i "keyword"

# 导出完整配置
cmake -LAH .. > cmake_config.txt

# 交互式配置
ccmake ..
# 然后: [t] 显示高级选项 → [c] 配置 → [g] 生成

# 图形界面
cmake-gui
```



