## 背景
---

基于win11 蓝图笔记本构建的WSL+Ubuntu作为基础验证环境，来构建大模型微调、推理等环境。这里以总览视角掌握wsl命令的常见使用方法。

## 快捷操作
---

Windows Subsystem for Linux (WSL) 的命令可以帮助你高效地管理Linux子系统。下面这个表格整理了最常见的WSL命令及其用法，方便你快速查阅。


| 功能类别         | 命令示例                                          | 参数说明及用途                                               |
| :--------------- | :------------------------------------------------ | :----------------------------------------------------------- |
| **🔍 发行版管理** | `wsl --list --online`                             | 列出所有可从在线应用商店安装的Linux发行版。                  |
|                  | `wsl --list --verbose`(或 `wsl -l -v`)            | 详细列出已安装的所有发行版，包括其**名称、状态和运行的WSL版本（1或2）**。 |
|                  | `wsl --install -d <发行版名称>`                   | 安装指定的Linux发行版，例如 `wsl --install -d Ubuntu`。      |
|                  | `wsl --set-default <发行版名称>`                  | 将指定发行版设置为**默认发行版**，之后直接输入`wsl`命令会进入该系统。 |
|                  | `wsl --unregister <发行版名称>`                   | **卸载**指定的发行版，会删除其所有数据和配置。               |
| **🚀 运行与控制** | `wsl`                                             | **启动默认的Linux发行版**并进入其Shell环境。                 |
|                  | `wsl --distribution <发行版名称>`(或 `wsl -d`)    | 启动指定的发行版，例如 `wsl -d Ubuntu-22.04`。               |
|                  | `wsl --shutdown`                                  | **立即终止所有正在运行的发行版和WSL 2虚拟机**，相当于彻底关闭WSL服务。 |
|                  | `wsl --terminate <发行版名称>`(或 `wsl -t`)       | 终止（关闭）某个特定的发行版。                               |
| **👤 用户与权限** | `wsl --user <用户名>`(或 `wsl -u`)                | 以指定的用户身份运行Linux发行版，例如 `wsl -u root`以**root管理员身份**运行。 |
|                  | `<发行版名称>.exe config --default-user <用户名>` | 更改某个发行版的**默认登录用户**。例如，`ubuntu2204.exe config --default-user root`。 |
| **💻 执行命令**   | `wsl <Linux命令>`                                 | 在Windows命令提示符或PowerShell中**直接执行单条Linux命令**。例如：`wsl ls -l`。 |
|                  | `wsl --cd <目录路径>`                             | 设置命令执行的**当前工作目录**。可以接受Linux路径（如`/home`）或Windows绝对路径（如`C:\Users`）。 |
| **💾 磁盘与文件** | `wsl --export <发行版名称> <文件路径>`            | 将指定发行版**导出**为一个tar格式的备份文件，用于备份或迁移。 |
|                  | `wsl --import <发行版名称> <安装位置> <备份文件>` | 将之前导出的备份文件**导入**为一个新的发行版。               |
|                  | `wslpath "C:\Users\..."`                          | 将Windows文件路径转换为WSL中的路径格式，方便在子系统内访问Windows文件。 |
| **⚙️ 系统维护**   | `wsl --update`                                    | **手动更新WSL内核**到最新版本。                              |
|                  | `wsl --status`                                    | 查看WSL的**总体状态**，包括默认版本、内核版本和最后一次更新日期等信息。 |
|                  | `wsl --set-default-version 2`                     | 设置**新安装发行版的默认WSL版本**（1或2），推荐设置为WSL 2以获得更完整的Linux体验。 |
|                  | `wsl --set-version <发行版名称> 2`                | 将**已安装的发行版从WSL 1转换为WSL 2**，这个过程可能需要几分钟。 |


### 💡 使用技巧与注意事项

- **获取帮助**：任何时候忘记命令，都可以在终端输入 `wsl --help`来查看完整的帮助信息


- **管理员权限**：部分命令（如安装、导入导出发行版或修改系统级设置）可能需要以**管理员身份**运行Windows终端（PowerShell或命令提示符）


- **WSL 2的优势**：WSL 2使用真正的Linux内核，在性能（尤其是文件I/O和 Docker 兼容性）上远超WSL 1。建议将新安装的发行版设置为WSL 2版本


- **文件系统互访**：你可以在WSL中通过`/mnt/c/`这样的路径访问Windows的C盘，反之亦然，这种集成非常方便。


## conda操作
---

| 操作类别                      | 功能描述                                                     | 命令示例                                                | 说明/注意事项                                        |
| :---------------------------- | :----------------------------------------------------------- | :------------------------------------------------------ | ---------------------------------------------------- |
| **基础检查**                  | 验证安装                                                     | `conda --version`                                       | 确认 conda 已正确安装并配置到 PATH。                 |
|                               | 查看当前所有环境                                             | `conda env list`                                        |                                                      |
| 或 `conda info -e`            | **星号 (`\*`)** 标记的是当前激活的环境。`base` 是默认根环境。 |                                                         |                                                      |
| **创建环境**                  | 创建指定 Python 版本的新环境                                 | `conda create -n <env_name> python=<version>`           | 例：`conda create -n myproj python=3.10`             |
| `-n` 指定环境名称。           |                                                              |                                                         |                                                      |
|                               | 创建环境并一次性安装包                                       | `conda create -n <env_name> python=<ver> <pkg1> <pkg2>` | 例：`conda create -n ai_env python=3.9 numpy pandas` |
|                               | **关键步骤**: 确认安装                                       | 输入 `y` 并回车                                         | 执行创建命令后，Conda 会列出待安装的包，需确认。     |
| **激活/退出**                 | 激活虚拟环境                                                 | `conda activate <env_name>`                             | 激活后，命令行提示符前会出现 `(env_name)`。          |
|                               | 退出当前环境                                                 | `conda deactivate`                                      | 返回到上一级环境（通常是 `base`）。                  |
| **包管理**                    | 在当前环境中安装包                                           | `conda install <package_name>`                          | 例：`conda install pytorch`                          |
| 优先使用 conda 安装二进制包。 |                                                              |                                                         |                                                      |
|                               | 使用 pip 安装包                                              | `pip install <package_name>`                            | 如果 conda 源没有该包，可在激活环境后使用 pip。      |
|                               | 更新某个包                                                   | `conda update <package_name>`                           | 例：`conda update numpy`                             |
|                               | 更新 conda 本身                                              | `conda update -n base -c defaults conda`                | 建议定期更新 conda 核心工具。                        |
|                               | 查看已安装的包                                               | `conda list`                                            | 列出当前激活环境中所有包及其版本。                   |
|                               | 搜索包                                                       | `conda search <package_name>`                           | 在仓库中搜索可用的包版本。                           |
| **环境维护**                  | 删除指定环境                                                 | `conda remove -n <env_name> --all`                      | **慎用**：会彻底删除该环境及其所有包。               |
|                               | 清理未使用的包和缓存                                         | `conda clean -all`                                      | 释放磁盘空间，删除下载的安装包缓存和无用包。         |
| **高级操作**                  | 导出环境配置                                                 | `conda env export > environment.yml`                    | 生成 YAML 文件，包含所有包及版本，用于复现环境。     |
|                               | 从文件创建环境                                               | `conda env create -f environment.yml`                   | 根据 YAML 文件还原一个完全一致的环境。               |
|                               | 克隆环境                                                     | `conda create --clone <source_env> -n <new_env>`        | 快速复制一个现有环境（如备份或实验）。               |

Miniconda 是 Conda 的轻量版，非常适合通过命令行灵活管理多个 Python 虚拟环境。


### 💡 给新手的特别提示 (Linux 环境)

**初始化 Shell (重要)：**

如果你安装后直接在终端输入 conda 提示“command not found”，可能是因为 shell 尚未初始化。运行以下命令（根据你的 shell 类型，通常是 bash 或 zsh）：
```bash
# 对于 bash
~/miniconda3/bin/conda init bash

# 对于 zsh
~/miniconda3/bin/conda init zsh
```

运行完后，关闭并重新打开终端，或者执行 source ~/.bashrc (或 ~/.zshrc) 使其生效。

**关于 Base 环境：**

默认情况下，每次打开终端都会自动激活 base 环境。如果你希望保持纯净，只在需要时激活，可以运行：
```bash
conda config --set auto_activate_base false
```
**国内镜像加速：**

如果在国内下载包速度较慢，建议配置清华或中科大镜像源：

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

**环境隔离原则：**
不要在 base 环境中安装项目特定的包。

每个新项目最好创建一个独立的虚拟环境（例如：conda create -n project_a python=3.9），这样可以避免不同项目间的依赖冲突（即“依赖地狱”）。
