# WSL2 + Ubuntu 24.04 + RTX 4080 构建 CUDA OpenCV 完整指南

## WSL2 特定注意事项

### 驱动版本说明

**你的环境**：

- Windows NVIDIA 驱动：**577.03** (2024年最新驱动)
- 支持 CUDA：最高 **12.9**
- WSL2 直接使用 Windows 驱动，无需在 WSL2 内安装驱动

**版本关系**：

```
Windows 主机
├── NVIDIA 驱动 577.03 (显卡驱动，已安装)
└── WSL2 虚拟机
    ├── 共享 Windows 驱动 (自动)
    └── CUDA Toolkit (需手动安装，推荐 12.6)
```

### WSL2 内存配置

RTX 4080 编译 OpenCV 可能需要较多内存，建议配置：

**在 Windows 创建/编辑配置文件**：

```powershell
# 文件位置: C:\Users\<你的用户名>\.wslconfig
# 用记事本创建或编辑此文件

[wsl2]
memory=16GB          # 分配给 WSL2 的内存 (推荐至少 12GB)
processors=8         # CPU 核心数
swap=8GB            # 交换空间
localhostForwarding=true
```

**应用配置**：

```powershell
# 在 Windows PowerShell (管理员) 运行
wsl --shutdown
# 等待 8 秒后重新启动
wsl
```

### 验证 WSL2 GPU 访问

在 WSL2 Ubuntu 中运行完整检查：

```bash
# 1. 检查驱动
nvidia-smi
# 应看到 Driver Version: 577.03

# 2. 检查 CUDA 运行时
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
# 应显示: NVIDIA GeForce RTX 4080, 577.03, 16384 MiB

# 3. 测试 CUDA 编译
cat << 'EOF' > test_cuda.cu
#include <stdio.h>
__global__ void hello() {
    printf("Hello from GPU!\n");
}
int main() {
    hello<<<1,1>>>();
    cudaDeviceSynchronize();
    return 0;
}
EOF

nvcc test_cuda.cu -o test_cuda
./test_cuda
# 应输出: Hello from GPU!

# 4. 查看 GPU 详细信息
nvidia-smi -L
# 应显示: GPU 0: NVIDIA GeForce RTX 4080 (UUID: GPU-xxxx)

# 5. 检查计算能力
/usr/local/cuda/extras/demo_suite/deviceQuery || echo "deviceQuery not found, skip"
```

## 前置条件检查

### 1. 验证 WSL2 GPU 支持

```bash
# 检查 WSL 版本 (Windows PowerShell)
wsl --version
# 需要 WSL 2.0+

# 检查 WSL 状态
wsl --status

# 在 WSL2 Ubuntu 中检查 GPU
nvidia-smi
```

**你的环境输出**（Windows 端）：

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 577.03                 Driver Version: 577.03         CUDA Version: 12.9     |
```

**重要说明**：

- Windows 驱动版本：577.03（最新驱动，很好！）
- Windows CUDA 版本：12.9（显示的是驱动支持的最高 CUDA 版本）
- **WSL2 中的 CUDA 版本需要单独安装，与 Windows 独立**

在 WSL2 Ubuntu 中运行 `nvidia-smi` 应该看到类似输出：

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.xx       Driver Version: 577.03       CUDA Version: 12.9    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA GeForce RTX 4080  Off | 00000000:01:00.0 Off |                  N/A |
+-------------------------------+----------------------+----------------------+
```

**注意**：

- WSL2 中显示的 Driver Version 会匹配 Windows（577.03）
- CUDA Toolkit 需要在 WSL2 内单独安装，版本可以不同（推荐 12.4-12.6）

### 2. 安装 NVIDIA CUDA Toolkit (WSL2)

**重要：Windows 和 WSL2 的 CUDA 是独立的**

- Windows 驱动：577.03（已安装，支持 CUDA 12.9）
- WSL2 CUDA Toolkit：需要在 Ubuntu 内单独安装

**推荐版本选择**：

- ✅ **CUDA 12.6** (推荐，稳定且与 OpenCV 兼容性好)
- ✅ CUDA 12.4 (也可以)
- ⚠️ CUDA 12.9 (最新，但某些库可能还不完全支持)

```bash
# 移除旧版本（如果有）
sudo apt-get --purge remove "*cublas*" "cuda*" "nsight*" 
sudo apt-get autoremove

# 添加 NVIDIA 官方源 (Ubuntu 24.04)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update

# 方案 A: 安装 CUDA 12.6 (推荐)
sudo apt-get install -y cuda-toolkit-12-6

# 方案 B: 安装 CUDA 12.4 (更保守)
# sudo apt-get install -y cuda-toolkit-12-4

# 方案 C: 安装最新版本 (与 Windows 匹配，但需测试兼容性)
# sudo apt-get install -y cuda-toolkit-12-9

# 设置环境变量
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda' >> ~/.bashrc
source ~/.bashrc

# 验证安装
nvcc --version
# 应显示: Cuda compilation tools, release 12.6 (或你安装的版本)

# 确认 WSL2 能看到 GPU
nvidia-smi
# Driver Version 应显示 577.03 (继承自 Windows)
```

**版本兼容性说明**：

| Windows 驱动 | 支持的最高 CUDA | WSL2 推荐安装 | OpenCV 兼容性 |
| ------------ | --------------- | ------------- | ------------- |
| 577.03       | 12.9            | 12.4 - 12.6   | ✅ 完美        |
| 577.03       | 12.9            | 12.9          | ⚠️ 需测试      |

**为什么不一定要匹配 12.9？**

- Windows 驱动 577.03 向下兼容所有 CUDA 12.x 版本
- OpenCV 和 cuDNN 对 12.6 及以下版本支持更成熟
- 可以避免遇到新版本的潜在 bug

### 3. 安装 cuDNN (可选但强烈推荐)

**cuDNN 版本选择**（基于你的 CUDA 版本）：

```bash
# 查看可用的 cuDNN 版本
apt-cache search cudnn | grep cuda-12

# 如果安装了 CUDA 12.6
sudo apt-get install -y libcudnn9-cuda-12 libcudnn9-dev-cuda-12

# 或针对特定 CUDA 版本
# sudo apt-get install -y libcudnn8-cuda-12-6  # CUDA 12.6 特定
```

**方法 2: 手动安装 (获取最新版本)**

1. 访问 https://developer.nvidia.com/cudnn-downloads (需要 NVIDIA 开发者账号，免费注册)
2. 选择：
   - cuDNN 9.x for CUDA 12.x
   - Linux x86_64
   - Ubuntu 24.04
3. 下载 Debian 包或 tar 文件

```bash
# 如果下载的是 .deb 包
sudo dpkg -i cudnn-local-repo-ubuntu2404-9.x.x.x_1.0-1_amd64.deb
sudo cp /var/cudnn-local-repo-ubuntu2404-9.x.x.x/cudnn-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get install -y libcudnn9-dev libcudnn9-cuda-12

# 如果下载的是 tar.xz 文件
tar -xvf cudnn-linux-x86_64-9.x.x.x_cuda12-archive.tar.xz
cd cudnn-linux-x86_64-9.x.x.x_cuda12-archive
sudo cp include/cudnn*.h /usr/local/cuda/include/
sudo cp lib/libcudnn* /usr/local/cuda/lib64/
sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
```

**验证 cuDNN 安装**：

```bash
# 方法 1: 检查头文件
cat /usr/include/cudnn_version.h | grep CUDNN_MAJOR -A 2
# 或
cat /usr/local/cuda/include/cudnn_version.h | grep CUDNN_MAJOR -A 2

# 方法 2: 检查库文件
ldconfig -p | grep cudnn

# 应该看到类似输出：
# libcudnn.so.9 (libc6,x86-64) => /usr/lib/x86_64-linux-gnu/libcudnn.so.9
```

**cuDNN 与 CUDA 版本对应**：

| CUDA 版本 | cuDNN 推荐版本 | 包名              |
| --------- | -------------- | ----------------- |
| 12.6      | cuDNN 9.x      | libcudnn9-cuda-12 |
| 12.4      | cuDNN 9.x      | libcudnn9-cuda-12 |
| 12.x      | cuDNN 8.9+     | libcudnn8-cuda-12 |

## 安装编译依赖

```bash
# 更新系统
sudo apt-get update && sudo apt-get upgrade -y

# 基础编译工具
sudo apt-get install -y build-essential cmake git pkg-config

# Python 开发环境
sudo apt-get install -y python3-dev python3-pip python3-numpy

# 图像/视频库
sudo apt-get install -y \
    libjpeg-dev libpng-dev libtiff-dev \
    libavcodec-dev libavformat-dev libswscale-dev \
    libv4l-dev libxvidcore-dev libx264-dev \
    libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev

# GUI 库
sudo apt-get install -y libgtk-3-dev libgtk2.0-dev

# 优化库
sudo apt-get install -y \
    libatlas-base-dev gfortran \
    libtbb-dev \
    libeigen3-dev \
    libopenblas-dev liblapack-dev

# OpenGL 支持 (可选)
sudo apt-get install -y libgl1-mesa-dev libglu1-mesa-dev
```

## 下载 OpenCV 源码

```bash
# 创建工作目录
mkdir -p ~/opencv_build && cd ~/opencv_build

# 下载 OpenCV 主仓库
git clone --depth 1 --branch 4.10.0 https://github.com/opencv/opencv.git

# 下载 opencv_contrib (包含 CUDA 模块)
git clone --depth 1 --branch 4.10.0 https://github.com/opencv/opencv_contrib.git

# 或使用最新主分支
# git clone https://github.com/opencv/opencv.git
# git clone https://github.com/opencv/opencv_contrib.git
```

## RTX 4080 架构信息

RTX 4080 基于 **Ada Lovelace 架构**：

- **Compute Capability**: 8.9
- **CUDA Cores**: 9728
- **Tensor Cores**: 第 4 代
- **RT Cores**: 第 3 代

## 配置 CMake (关键步骤)

```bash
cd ~/opencv_build/opencv
mkdir build && cd build

# 完整 CUDA 配置
cmake -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/usr/local \
    \
    # ===== CUDA 核心配置 =====
    -D WITH_CUDA=ON \
    -D CUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda \
    -D CUDA_ARCH_BIN=8.9 \
    -D CUDA_ARCH_PTX=8.9 \
    -D WITH_CUDNN=ON \
    -D CUDNN_INCLUDE_DIR=/usr/include \
    -D OPENCV_DNN_CUDA=ON \
    -D ENABLE_FAST_MATH=1 \
    -D CUDA_FAST_MATH=1 \
    -D WITH_CUBLAS=1 \
    -D WITH_CUFFT=ON \
    -D WITH_NVCUVID=ON \
    \
    # ===== Python 绑定 =====
    -D BUILD_opencv_python3=ON \
    -D PYTHON3_EXECUTABLE=$(which python3) \
    -D PYTHON3_INCLUDE_DIR=$(python3 -c "import sysconfig; print(sysconfig.get_path('include'))") \
    -D PYTHON3_PACKAGES_PATH=$(python3 -c "import sysconfig; print(sysconfig.get_path('purelib'))") \
    -D PYTHON3_NUMPY_INCLUDE_DIRS=$(python3 -c "import numpy; print(numpy.get_include())") \
    \
    # ===== 扩展模块 =====
    -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
    -D OPENCV_ENABLE_NONFREE=ON \
    \
    # ===== CPU 优化 =====
    -D CPU_BASELINE=SSE4_2 \
    -D CPU_DISPATCH=AVX,AVX2 \
    \
    # ===== 其他加速库 =====
    -D WITH_TBB=ON \
    -D WITH_OPENMP=ON \
    -D WITH_EIGEN=ON \
    -D WITH_LAPACK=ON \
    -D WITH_OPENCL=ON \
    \
    # ===== 视频编解码 =====
    -D WITH_FFMPEG=ON \
    -D WITH_GSTREAMER=ON \
    -D WITH_V4L=ON \
    \
    # ===== 构建选项 =====
    -D BUILD_TESTS=OFF \
    -D BUILD_PERF_TESTS=OFF \
    -D BUILD_EXAMPLES=OFF \
    -D INSTALL_PYTHON_EXAMPLES=OFF \
    -D INSTALL_C_EXAMPLES=OFF \
    -D OPENCV_GENERATE_PKGCONFIG=ON \
    ..
```

### 关键 CUDA 选项说明

| 选项              | 值   | 说明                      |
| ----------------- | ---- | ------------------------- |
| `WITH_CUDA`       | ON   | 启用 CUDA 支持            |
| `CUDA_ARCH_BIN`   | 8.9  | RTX 4080 的计算能力       |
| `CUDA_ARCH_PTX`   | 8.9  | PTX 中间代码版本          |
| `WITH_CUDNN`      | ON   | 启用 cuDNN 深度学习加速   |
| `OPENCV_DNN_CUDA` | ON   | DNN 模块使用 CUDA         |
| `CUDA_FAST_MATH`  | 1    | CUDA 快速数学运算         |
| `WITH_CUBLAS`     | 1    | 启用 cuBLAS 线性代数      |
| `WITH_CUFFT`      | ON   | 启用 cuFFT 快速傅里叶变换 |
| `WITH_NVCUVID`    | ON   | NVIDIA 视频解码           |

### 查找 Compute Capability

如果不确定显卡的计算能力：

```bash
# 方法 1: 使用 nvidia-smi
nvidia-smi --query-gpu=compute_cap --format=csv

# 方法 2: 运行 CUDA 示例
/usr/local/cuda/extras/demo_suite/deviceQuery

# 方法 3: 查询 NVIDIA 官方文档
# https://developer.nvidia.com/cuda-gpus
# RTX 4090: 8.9
# RTX 4080: 8.9
# RTX 4070: 8.9
# RTX 3090: 8.6
# RTX 3080: 8.6
```

## 验证配置

配置完成后，检查输出：

```bash
# 查看 CUDA 配置摘要
cmake .. 2>&1 | grep -A 20 "CUDA"
```

**期望看到**：

```
--   NVIDIA CUDA:                   YES (ver 12.6, CUFFT CUBLAS FAST_MATH)
--     NVIDIA GPU arch:             89
--     NVIDIA PTX archs:            89
--   cuDNN:                         YES (ver 9.x)
```

**Python 配置**：

```
--   Python 3:
--     Interpreter:                 /usr/bin/python3 (ver 3.12.3)
--     Libraries:                   /usr/lib/x86_64-linux-gnu/libpython3.12.so
--     numpy:                       .../site-packages/numpy/core/include (ver 1.26.4)
--     install path:                lib/python3.12/site-packages/cv2/python-3.12
```

**如果 CUDA 未检测到**，检查：

```bash
# 验证 CMake 能找到 CUDA
cmake --find-package -DNAME=CUDA -DCOMPILER_ID=GNU -DLANGUAGE=CXX -DMODE=EXIST

# 手动指定 CUDA 路径
cmake -D CUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.6 ..
```

## 编译和安装

```bash
# 查看将要编译的模块
make help | grep opencv_

# 编译 (使用所有 CPU 核心)
make -j$(nproc)

# 如果内存不足 (WSL2 可能限制内存)
make -j4  # 使用 4 个线程

# 安装
sudo make install

# 更新链接器缓存
sudo ldconfig

# 更新 Python 路径 (如果需要)
echo 'export PYTHONPATH=/usr/local/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
source ~/.bashrc
```

### 编译时间估算

RTX 4080 + 现代 CPU:

- 全模块编译：20-40 分钟
- 仅核心模块：10-20 分钟

## 验证安装

### 1. 验证 C++ 库

```bash
# 检查 OpenCV 版本
pkg-config --modversion opencv4

# 检查 CUDA 模块
pkg-config --libs opencv4 | grep cuda
```

### 2. 验证 Python 绑定

创建 `test_cuda.py`:

```python
import cv2
import numpy as np
print(f"OpenCV Version: {cv2.__version__}")
print(f"CUDA Enabled: {cv2.cuda.getCudaEnabledDeviceCount() > 0}")

if cv2.cuda.getCudaEnabledDeviceCount() > 0:
    print(f"CUDA Devices: {cv2.cuda.getCudaEnabledDeviceCount()}")
    print(f"CUDA Device Name: {cv2.cuda.getDevice()}")
    
    # 打印 CUDA 构建信息
    build_info = cv2.getBuildInformation()
    for line in build_info.split('\n'):
        if 'CUDA' in line or 'cuDNN' in line:
            print(line)
    
    # 测试 CUDA 操作
    print("\n=== Testing CUDA Operations ===")
    
    # 上传到 GPU
    cpu_mat = np.random.rand(1000, 1000).astype(np.float32)
    gpu_mat = cv2.cuda_GpuMat()
    gpu_mat.upload(cpu_mat)
    
    # GPU 高斯模糊
    gpu_result = cv2.cuda.createGaussianFilter(
        cv2.CV_32F, cv2.CV_32F, (5, 5), 1.5
    ).apply(gpu_mat)
    
    # 下载结果
    cpu_result = gpu_result.download()
    print(f"✓ CUDA Gaussian Blur successful: {cpu_result.shape}")
    
else:
    print("❌ CUDA not available!")
    print("\nBuild Information:")
    print(cv2.getBuildInformation())
```

运行测试：

```bash
python3 test_cuda.py
```

### 3. 性能对比测试

创建 `benchmark_cuda.py`:

```python
import cv2
import numpy as np
import time

print(f"OpenCV: {cv2.__version__}")
print(f"CUDA Devices: {cv2.cuda.getCudaEnabledDeviceCount()}\n")

# 创建测试图像
img = np.random.randint(0, 255, (4000, 4000, 3), dtype=np.uint8)

# ===== CPU 测试 =====
print("=" * 50)
print("CPU Performance")
print("=" * 50)

start = time.time()
for _ in range(50):
    result_cpu = cv2.GaussianBlur(img, (5, 5), 0)
cpu_time = time.time() - start
print(f"GaussianBlur (4000x4000, 50 iterations): {cpu_time:.3f}s")
print(f"Average: {cpu_time/50*1000:.1f}ms per iteration")

# ===== GPU 测试 =====
if cv2.cuda.getCudaEnabledDeviceCount() > 0:
    print("\n" + "=" * 50)
    print("GPU Performance")
    print("=" * 50)
    
    # 上传到 GPU
    gpu_img = cv2.cuda_GpuMat()
    gpu_img.upload(img)
    
    # 创建滤波器
    gaussian = cv2.cuda.createGaussianFilter(
        cv2.CV_8UC3, cv2.CV_8UC3, (5, 5), 1.5
    )
    
    # 预热
    for _ in range(5):
        gpu_result = gaussian.apply(gpu_img)
    
    # 测试
    start = time.time()
    for _ in range(50):
        gpu_result = gaussian.apply(gpu_img)
        cv2.cuda.Stream.waitForCompletion(cv2.cuda.Stream())
    gpu_time = time.time() - start
    
    print(f"GaussianBlur (4000x4000, 50 iterations): {gpu_time:.3f}s")
    print(f"Average: {gpu_time/50*1000:.1f}ms per iteration")
    
    # 加速比
    speedup = cpu_time / gpu_time
    print(f"\n🚀 Speedup: {speedup:.2f}x faster on GPU")
else:
    print("\n❌ GPU not available")
```

运行基准测试：

```bash
python3 benchmark_cuda.py
```

**预期结果**：GPU 应该比 CPU 快 5-20 倍（取决于操作类型）

## 常见问题和解决方案

### 问题 1: nvcc not found

```bash
# 检查 CUDA 安装
which nvcc
ls -la /usr/local/cuda/bin/nvcc

# 如果找不到，设置路径
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

# 永久添加
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### 问题 2: WSL2 中 nvidia-smi 显示 "NVIDIA-SMI has failed"

**原因**: Windows 驱动未正确传递到 WSL2

**解决方案**:

```powershell
# 在 Windows PowerShell (管理员) 运行
wsl --shutdown
# 更新 WSL2 内核
wsl --update
# 重启 WSL2
wsl

# 在 WSL2 中验证
nvidia-smi
```

如果还是失败：

1. 确保 Windows NVIDIA 驱动是最新的 (你的 577.03 已经是最新)
2. 检查 WSL2 版本：`wsl --version` (需要 2.0.0+)
3. 重新安装 WSL2 GPU 支持：

```powershell
# Windows PowerShell
wsl --install
```

### 问题 3: CUDA 版本不匹配警告

如果看到类似警告：

```
CUDA driver version is insufficient for CUDA runtime version
```

**原因**: WSL2 CUDA Toolkit 版本高于 Windows 驱动支持的版本

**解决方案**:

```bash
# 卸载当前 CUDA
sudo apt-get --purge remove "*cuda*"

# 安装与驱动兼容的版本 (你的 577.03 支持 12.9，所以任何 12.x 都可以)
sudo apt-get install -y cuda-toolkit-12-6
```

### 问题 4: CMake 找不到 CUDA

```bash
# 手动指定 CUDA 路径
cmake -D CUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.6 \
      -D CUDA_CUDA_LIBRARY=/usr/local/cuda/lib64/stubs/libcuda.so \
      ..

# 或设置环境变量
export CUDA_HOME=/usr/local/cuda
export CUDA_PATH=/usr/local/cuda
```

### 问题 5: CUDA 编译错误 "unsupported GPU architecture"

```bash
# 检查显卡计算能力
nvidia-smi --query-gpu=compute_cap --format=csv
# RTX 4080 应该显示 8.9

# 重新配置，确保使用正确的架构
cd build
rm -rf *
cmake -D WITH_CUDA=ON \
      -D CUDA_ARCH_BIN=8.9 \
      -D CUDA_ARCH_PTX=8.9 \
      ..
```

### 问题 6: cuDNN 版本不兼容

如果遇到 "cuDNN version mismatch" 错误：

```bash
# 检查当前 cuDNN 版本
cat /usr/include/cudnn_version.h | grep CUDNN_MAJOR -A 2

# 卸载并重新安装
sudo apt-get --purge remove "*cudnn*"
sudo apt-get install -y libcudnn9-cuda-12 libcudnn9-dev-cuda-12

# 清理 CMake 缓存并重新配置
cd opencv/build
rm -rf *
cmake -D WITH_CUDA=ON -D WITH_CUDNN=ON ..
```

### 问题 7: Python 找不到 cv2 模块

```bash
# 查找安装位置
find /usr/local -name "cv2*.so"

# 添加到 Python 路径
export PYTHONPATH=/usr/local/lib/python3.12/site-packages:$PYTHONPATH

# 或创建符号链接
SITE_PACKAGES=$(python3 -c "import sysconfig; print(sysconfig.get_path('purelib'))")
sudo ln -sf /usr/local/lib/python3.12/site-packages/cv2 $SITE_PACKAGES/cv2
```

### 问题 8: WSL2 内存不足 (编译时崩溃)

**症状**: 编译到一半进程被杀死，或系统卡死

**解决方案 1**: 增加 WSL2 内存 (推荐)

在 Windows 创建 `C:\Users\<用户名>\.wslconfig`:

```ini
[wsl2]
memory=16GB
processors=8
swap=8GB
```

然后重启：

```powershell
wsl --shutdown
wsl
```

**解决方案 2**: 减少编译并发数

```bash
# 不要用 make -j$(nproc)
make -j4  # 只用 4 个线程

# 或者更保守
make -j2
```

**解决方案 3**: 分步编译

```bash
# 只编译核心模块
make -j4 opencv_core opencv_imgproc
# 等完成后再编译其他
make -j4
```

### 问题 9: CUDA out of memory (运行时)

```python
# 监控 GPU 内存使用
import cv2
print(f"Total Memory: {cv2.cuda.DeviceInfo().totalMemory() / 1024**3:.2f} GB")
print(f"Free Memory: {cv2.cuda.DeviceInfo().freeMemory() / 1024**3:.2f} GB")

# 处理大图像时分块处理
# 不要一次性加载整个 4K/8K 图像到 GPU
```

### 问题 10: 驱动版本 577.03 特定问题

最新驱动可能有新 bug，如果遇到问题：

```powershell
# Windows 上回退到稳定版本驱动 (可选)
# 访问 NVIDIA 官网下载 560.x 或 570.x 系列
```

或在 WSL2 中使用较旧的 CUDA Toolkit：

```bash
# 使用更稳定的 CUDA 12.4
sudo apt-get install -y cuda-toolkit-12-4
```

### 问题 11: WSL2 性能不如原生 Linux

**优化建议**：

1. 将项目文件放在 Linux 文件系统 (`~/`) 而不是 Windows 挂载点 (`/mnt/c/`)
2. 使用 WSL2 内部的编辑器，避免跨文件系统操作
3. 禁用 Windows Defender 对 WSL2 目录的实时扫描

```powershell
# Windows PowerShell (管理员)
# 排除 WSL2 虚拟磁盘
Add-MpPreference -ExclusionPath "C:\Users\<用户名>\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu*\LocalState\ext4.vhdx"
```

### 调试技巧

**查看详细的 CMake 配置过程**:

```bash
cmake --debug-output -D WITH_CUDA=ON .. 2>&1 | tee cmake_debug.log
grep -i "cuda\|cudnn" cmake_debug.log
```

**查看编译详细输出**:

```bash
make VERBOSE=1 2>&1 | tee build.log
```

**测试 CUDA 是否真的在工作**:

```bash
# 使用 NVIDIA Nsight Systems 性能分析
nsys profile -o opencv_test python3 test_cuda.py
```

## 高级优化选项

### 针对 RTX 4080 的优化配置

```bash
cmake -D CMAKE_BUILD_TYPE=RELEASE \
    # Tensor Core 支持 (Ada Lovelace)
    -D CUDA_ARCH_BIN="8.9" \
    -D CUDA_ARCH_PTX="8.9" \
    \
    # 编译器优化
    -D CMAKE_CXX_FLAGS="-O3 -march=native" \
    -D CMAKE_C_FLAGS="-O3 -march=native" \
    \
    # CUDA 优化
    -D CUDA_NVCC_FLAGS="-O3 --use_fast_math -Xcompiler -march=native" \
    \
    # 减少编译产物大小
    -D CUDA_ARCH_PTX="" \  # 只保留 BIN
    \
    # 最大并行度
    -D CV_PARALLEL_FRAMEWORK=TBB \
    ..
```

### 仅构建 CUDA 相关模块

```bash
cmake -D BUILD_LIST=core,imgproc,imgcodecs,videoio,highgui,cuda,cudaarithm,cudafilters,cudaimgproc,cudawarping ..
```

## 卸载

```bash
cd ~/opencv_build/opencv/build
sudo make uninstall

# 或手动删除
sudo rm -rf /usr/local/include/opencv4
sudo rm -rf /usr/local/lib/libopencv*
sudo rm -rf /usr/local/lib/python3.12/site-packages/cv2
```

## 总结

完整构建流程：

1. ✅ 安装 CUDA Toolkit (12.x)
2. ✅ 安装 cuDNN
3. ✅ 安装编译依赖
4. ✅ 下载 OpenCV + contrib
5. ✅ 配置 CMake (CUDA_ARCH_BIN=8.9)
6. ✅ 编译安装
7. ✅ 验证 CUDA 功能

关键参数：

- **CUDA_ARCH_BIN=8.9** (RTX 4080)
- **WITH_CUDNN=ON**
- **OPENCV_DNN_CUDA=ON**



## openCV+cuda 构建备份
---


```bash
# 完整 CUDA 配置
#
#
# https://zread.ai/opencv/opencv/3-building-opencv-from-source
#
cmake .. -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/usr/local/opencv410 \
    \
    -D WITH_CUDA=ON \
    -D CUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.6 \
    -D CUDA_ARCH_BIN=8.9 \
    -D CUDA_ARCH_PTX=8.9 \
    \
    -D WITH_CUDNN=ON \
    -D CUDNN_INCLUDE_DIR=/usr/include/x86_64-linux-gnu \
    -D OPENCV_DNN_CUDA=ON \
    -D CUDNN_LIBRARY=/usr/lib/x86_64-linux-gnu/libcudnn.so \
    \
    -D ENABLE_FAST_MATH=1 \
    -D CUDA_FAST_MATH=1 \
    \
    -D WITH_CUBLAS=ON \
    -D WITH_CUFFT=ON \
    -D WITH_NVCUVID=ON \
    \
    -D BUILD_opencv_python3=ON \
    -D PYTHON3_EXECUTABLE=$(which python3) \
    -D PYTHON3_INCLUDE_DIR=$(python3 -c "import sysconfig; print(sysconfig.get_path('include'))") \
    -D PYTHON3_PACKAGES_PATH=$(python3 -c "import sysconfig; print(sysconfig.get_path('purelib'))") \
    -D PYTHON3_NUMPY_INCLUDE_DIRS=$(python3 -c "import numpy; print(numpy.get_include())") \
    \
    -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
    -D OPENCV_ENABLE_NONFREE=ON \
    \
    -D CPU_BASELINE=SSE4_2 \
    -D CPU_DISPATCH=AVX,AVX2 \
    \
    -D WITH_TBB=ON \
    -D WITH_OPENMP=ON \
    -D WITH_EIGEN=ON \
    -D WITH_LAPACK=ON \
    -D WITH_OPENCL=ON \
    -D WITH_FFMPEG=ON \
    -D WITH_GSTREAMER=ON \
    -D WITH_V4L=ON \
    \
    -D BUILD_TESTS=OFF \
    -D BUILD_PERF_TESTS=OFF \
    -D BUILD_EXAMPLES=OFF \
    -D INSTALL_PYTHON_EXAMPLES=ON \
    -D INSTALL_C_EXAMPLES=ON \
    \
    -D WITH_NVCUVID=OFF \
    -D WITH_NVCUVENC=OFF \
    \
    -D ENABLE_OPTIMIZATION=ON \
    -D ENABLE_INSTRUMENTATION=ON \
    \
    -D BUILD_ZLIB=OFF \
    -D BUILD_JPEG=OFF \
    -D BUILD_PNG=OFF \
    -D BUILD_TIFF=OFF \
    -D WITH_OEPNGL=ON  \
    -D OPENCV_GENERATE_PKGCONFIG=ON
```
