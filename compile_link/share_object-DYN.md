
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

<img width="507" height="181" alt="image" src="https://github.com/user-attachments/assets/c4d9d885-e8b9-4f06-93c8-1f9e1365a165" />

当你运行 ./a.out时，动态链接器会：

1. 遇到 $ORIGIN，将其解析为 a.out所在的目录，即 /home/user/projects/myapp。

2. 首先在 /home/user/projects/myapp目录下查找 libexample.so并成功加载。

3. 如果在该目录下没找到，它才会继续尝试在 /opt目录中查找。

### 总结

简而言之，$ORIGIN代表的是可执行文件自身的目录路径。这是一种非常实用的技术，常用于制作可以自包含的可执行文件，使其能够优先加载与其位于同一目录下的本地共享库，从而实现程序的便捷分发和部署，而不必依赖系统全局库路径的设置。


## fPIC 与 shared
---

动态库的编译必须要加上-fPIC编译选项，-fPIC选项告诉编译器生成位置无关代码，命令如下：

```bash
gcc -fPIC -shared -o libexample.so example.c
```


命令 `gcc -fPIC -shared -o libexample.so example.c`中，**`-shared`参数是生成动态链接库（.so 文件）所必需的**。



| 参数          | 主要作用                                                     | 是否必需     |
| :------------ | :----------------------------------------------------------- | :----------- |
| **`-shared`** | **链接器行为**：指示编译器在链接阶段生成一个**共享对象文件（动态库）** 而不是可执行文件。 | **是**       |
| **`-fPIC`**   | **编译器行为**：指示编译器生成**位置无关代码**，这是动态库中代码的理想特性。 | **强烈推荐** |



### 💡 深入理解参数的作用

- **`-shared`的核心作用**： 改变编译器的链接目标。若不存在，`gcc`的默认行为是尝试生成一个可执行文件。而添加 `-shared`后，它会告诉链接器：“请将输入的目标文件打包成一个可以被多个程序在运行时动态加载的共享库”，而不是一个独立的、可以直接运行的程序 。因此，它是区分生成动态库和可执行文件的关键指令。

  

- **`-fPIC`的重要性**：虽然在某些简单情况下，不加 `-fPIC`也可能成功生成 `.so`文件，但**强烈建议始终加上**。位置无关代码意味着生成的代码不依赖于在内存中的**固定加载地址**。这使得同一个动态库可以被多个进程共享，从而节省内存，并且保证了库的灵活性和安全性 。如果不使用 `-fPIC`，代码可能会包含绝对地址，当被多个进程加载时，每个进程可能都需要一份该库的副本，并且可能需要进行代码修复，这违背了动态库的共享初衷并可能引发问题 。



### 🛠️ 动态库的典型创建步骤

一个规范的动态库创建过程通常分为两步，这能更清晰地展示参数的作用：

1. **编译为目标文件**：首先使用 `-fPIC`选项将源文件编译成位置无关的目标文件（`.o`文件）。

   ```
   gcc -c -fPIC example.c -o example.o
   ```

2. **链接为动态库**：然后使用 `-shared`选项将这些目标文件链接成最终的动态库。

   ```
   gcc -shared example.o -o libexample.so
   ```

 `gcc -fPIC -shared -o libexample.so example.c`是将以上两步合并的简洁形式 



 ## 原文内容

https://mp.weixin.qq.com/s/f11qJv3MMj8uUV61n32h2w

 
