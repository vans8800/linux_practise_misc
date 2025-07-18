## 如何用Make命令查看Makefile的构建目标
---


在使用一个标准的 Makefile 时，想要列出它支持的构建目标（targets）以及构建这些目标所需的 make 命令，有以下几种常见方法：

### ✅ 方法一：查看 Makefile 中定义的显式目标


直接在终端中运行下面命令，列出所有在 Makefile 中定义的目标：

```bash
make -npr | grep -E '^[a-zA-Z0-9][^$#\/\t=]*:([^=]|$)' | cut -d: -f1 | sort -u
```


```bash

##笔者验证

[aippc@localhost searxng]$ make -npr | grep -E '^[a-zA-Z0-9][^$#\/\t=]*:([^=]|$)' | cut -d: -f1 | sort -u
all
clean
docker.build
docker.buildx
docs
docs.clean
docs.gh-pages
docs.live
docs.prebuild
gecko.driver
help
lxc.clean
lxc-purge
Makefile
make-help
node.clean
node.env
node.env.dev
nvm.clean
nvm.nodejs
podman.build
py.build
py.clean
pyenv
run
search.checker
search.checker.%
themes
themes.all
themes.fix

```
make -npr：打印 make 的内部规则，不执行。

grep ...：提取形式为 target: 的行（排除变量定义等）。

cut -d: -f1：只取目标名。

sort -u：去重并排序。

### 正则表达式解析
---

整体功能​​： 该正则表达式主要用于 ​​匹配配置文件中合法的键值对格式的行​​，具体场景包括：

- ​​键（Key）的合法性验证​​：确保键以字母或数字开头，且不包含特殊字符。

​​- 分隔符和值（Value）的约束​​：键后必须有冒号分隔符，且值部分不能以等号开头（避免与赋值语句混淆）。

​- ​过滤无效行​​：自动跳过注释行（以 # 或 $ 开头）、空行或格式错误的行。


​​**表达式拆解分析​**​

1. ^[a-zA-Z0-9]
​​
- ^​​：匹配行首，确保匹配从行开始。
​​
- [a-zA-Z0-9]​​：匹配​​单个字母或数字​​，要求键必须以字母或数字开头（排除以符号开头的无效键）。

2. [^$#\/\t=]*
​​
[^...]​​：否定字符类，匹配​​不包含指定字符​​的任意字符。


**​​排除的字符​​：**

- $ 和 #：常见注释符号（如 # Comment 或 $VAR=value）。

- /：路径符号（如 /etc/config）。

- \t：制表符（避免键含空白符）。

- =：防止键名包含赋值符号（如 key=value 是无效键）。
​​
- *​​：匹配前一部分 ​​0 次或多次​​，允许键名长度可变（如 key1 或 long_key_name）。


**其他内容**

(3) : ​​字面冒号​​：作为键值对的分隔符，必须存在（如 key:value）。


(4) ([^=]|$)


​​分组逻辑​​：
​​
-  [^=]​​：匹配​​非等号字符​​（确保值不以 = 开头，避免与赋值混淆）。

-  $​​：匹配行尾（允许值为空，如 key:）。
​​
-  |​​：逻辑“或”，满足任一条件即可（值非空或行结束）。


### ✅ 方法二：让 Makefile 自动打印所有目标

许多 Makefile 中会定义一个特殊目标 help，你可以试试运行：

```bash
make help

```

如果 Makefile 编写得规范且包含 help 目标，它通常会列出类似以下内容：

```bash
Usage:
  make all        # Build all components
  make clean      # Remove all built files
  make install    # Install binaries
```

如果你自己编写 Makefile，也可以添加如下 help 目标：

```makefile

help:
	@echo "Available targets:"
	@grep -E '^[a-zA-Z0-9_-]+:.*?## ' Makefile | \
	  awk -F ':|##' '{printf "  %-15s %s\n", $$1, $$3}'

```


然后你可以使用 make help 来列出所有带 ## 注释的目标说明。

### ✅ 方法三：使用 awk 或 sed 直接分析 Makefile

例如，可以列出所有没有前缀的顶级目标：

```bash

awk -F':' '/^[a-zA-Z0-9][^$#\/\t=]*:/{print $1}' Makefile | sort | uniq


```

### ✅ 方法四：结合 make -n 查看目标命令

若已知目标名称（如 clean），你可以查看该目标执行什么命令：

```bash

make -n clean
```

这会显示 make clean 时会执行的具体命令，但不会实际执行（安全预览）。



##  正则表达解析


🧩 正则表达式结构详解

```bash
^[a-zA-Z0-9][^$#\/\t=]*:([^=]|$)

```


1. ^
匹配行首。

2. [a-zA-Z0-9]
行首第一个字符必须是 字母或数字。

不包括下划线 _ 或其他标点。

3. [^$#\/\t=]*
非捕获字符集合：匹配任意数量的 不包含 下面字符的字符：

- $ （美元符）

- \# （井号） 注意：这里因为#在MD文件中表示不同的段落等级，所以这里用\# 代替 # 仅为格式统一。

- / （斜杠）

- \t（Tab制表符）

- = （等号）

✅ 示例能匹配的字符串部分, 如：

```makefile

configValue:
section1:
foo_bar:

```

❌ 不会匹配的, 如：

```makefile

$VAR:
name#:
name/:
name=:  ← 注意这里冒号后有等号

```

4. :（冒号）

要求前面的内容后面紧跟着一个冒号。

5. ([^=]|$)

接着匹配冒号之后的一个字符，这个字符不是等号 =，或者就是行尾。

意图是：避免匹配诸如 name:=value 这类结构。

📌 整体含义总结

这个正则表达式的匹配条件是： 匹配以字母或数字开头，后跟任意不含 $, #, /, Tab 或 = 的字符，紧跟一个冒号 :，且该冒号后面不能是 = 的整行文本。

✅ 可能匹配的示例行：

```makefile 
CONFIG_OPTION: value
name: some text
section1: another section
abc: 123
```

❌ 不会匹配的行：

```makefile

# 注释行（以#开头）
=key: value（不是以字母数字开头）
name:=value（冒号后紧跟等号）
name/#:value（包含非法字符/）

```

### 📚 典型应用场景

这种正则一般用于 解析配置文件中的“标签头部”（如 Makefile, .ini 文件，或某种结构化文本），目的是：

**识别合法的段标签或键名：**比如 section1:

**排除注释、变量赋值、特殊行：**避免如 VAR:=value、#comment、/path: 之类干扰


### 括号

🧠 首先：() 在正则中的作用

在正则表达式中，() 表示：

捕获组（Capturing Group）：用于提取匹配内容，供后续引用（如 \1, $1，或 grep -oP 等工具中使用）。

也可以用于逻辑分组（让 | 作用于多个字符块）。

📌 在你给出的正则中：

```bash
^[a-zA-Z0-9][^$#\/\t=]*:([^=]|$)

```
其中：

```bash
([^=]|$)
```

表示“冒号后面跟的要么不是等号 =, 要么是行尾 $”，即：

不允许像 abc:=value 这样的格式出现。

但允许 abc: 或 abc: something

这个子表达式中的括号 () 在这里只是 创建了一个捕获组，但如果你不需要引用该组结果（比如不在 sed, awk, grep -o 等中使用 $1），那么：

✅ 可以写成不带括号的版本：

``` bash

^[a-zA-Z0-9][^$#\/\t=]*:[^=]|\:$

```

这个版本逻辑等价，意思是：

匹配：

冒号后不是等号： [^=]

或者是冒号后直接行尾： :$
