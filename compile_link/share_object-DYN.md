
## MISC1
---

<img width="625" height="110" alt="image" src="https://github.com/user-attachments/assets/50ab6aef-e9cc-4e43-b965-ba3a709f47b2" />=

### ORIGIN

$ORIGIN​ 是一个由动态链接器在程序运行时识别的特殊变量。

$ORIGIN代表的是正在执行的程序（即 a.out）自身所在的目录路径。

$ORIGIN不是一个由 Shell 设置的系统环境变量（如 $PATH或 $HOME），而是链接器（ld）能够理解的一个特殊占位符。当程序启动，动态链接器需要寻找其依赖的共享库（如 libexample.so）时，它会将 rpath中出现的 $ORIGIN替换为这个程序二进制文件（例如 /home/user/projects/myapp/a.out）所在的实际目录路径（例如 /home/user/projects/myapp）。

在命令中的作用：
- -Wl,-rpath=选项用于将库的搜索路径（即 rpath）直接嵌入到可执行文件中。

- -Wl,-rpath='$ORIGIN':/opt这条指令告诉动态链接器：

“当运行 a.out时，请首先在 a.out文件所在的同一目录下寻找 libexample.so，如果没找到，然后再去 /opt目录下寻找。”

#### 为什么使用单引号 

在命令中，'$ORIGIN'使用了单引号。这是至关重要的，目的是防止当前 Shell 在调用 gcc之前就解释 $ORIGIN这个变量。单引号确保了字符串 $ORIGIN会被原封不动地传递给链接器，由链接器在将来运行时再去解析它。

#### 实际应用示例

假设你的项目目录结构如下，并且你在 /home/user/projects/myapp目录下执行了图片中的编译命令：
/home/user/projects/myapp/
├── libexample.so  (编译生成的库)
├── example.c
├── main.c
└── a.out         (编译生成的可执行文件)
当你运行 ./a.out时，动态链接器会：

1. 遇到 $ORIGIN，将其解析为 a.out所在的目录，即 /home/user/projects/myapp。

2. 首先在 /home/user/projects/myapp目录下查找 libexample.so并成功加载。

3. 如果在该目录下没找到，它才会继续尝试在 /opt目录中查找。

### 总结

简而言之，$ORIGIN代表的是可执行文件自身的目录路径。这是一种非常实用的技术，常用于制作可以自包含的可执行文件，使其能够优先加载与其位于同一目录下的本地共享库，从而实现程序的便捷分发和部署，而不必依赖系统全局库路径的设置。
