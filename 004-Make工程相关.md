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


​​表达式拆解分析​​

1. (1) ^[a-zA-Z0-9]
​​
- ^​​：匹配行首，确保匹配从行开始。
​​
- [a-zA-Z0-9]​​：匹配​​单个字母或数字​​，要求键必须以字母或数字开头（排除以符号开头的无效键）。

2. (2) [^$#\/\t=]*
​​
[^...]​​：否定字符类，匹配​​不包含指定字符​​的任意字符。

​​排除的字符​​：

- $ 和 #：常见注释符号（如 # Comment 或 $VAR=value）。

- /：路径符号（如 /etc/config）。

- \t：制表符（避免键含空白符）。

- =：防止键名包含赋值符号（如 key=value 是无效键）。
​​
- *​​：匹配前一部分 ​​0 次或多次​​，允许键名长度可变（如 key1 或 long_key_name）。


(3) : ​​字面冒号​​：作为键值对的分隔符，必须存在（如 key:value）。

(4) ([^=]|$)

​​分组逻辑​​：
​​
-  [^=]​​：匹配​​非等号字符​​（确保值不以 = 开头，避免与赋值混淆）。

​​-  $​​：匹配行尾（允许值为空，如 key:）。
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



