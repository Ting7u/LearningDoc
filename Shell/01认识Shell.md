# 第 1 章　认识 Shell
## 本章目标
- 分清「终端」「Shell」「bash / zsh / sh」这几个常被混用的概念
- 看懂并写出一条命令的基本格式
- 掌握运行 Shell 脚本的几种方式，理解 shebang 的作用
## 1.1 终端、Shell、内核：三者的关系
![[Pasted image 20261122230327.png]]

| 概念           | 是什么                                 | 例子                                                          |
| :----------- | :---------------------------------- | :---------------------------------------------------------- |
| 终端（Terminal） | 一个负责「显示文字、接收键盘输入」的窗口程序              | macOS 的 Terminal.app、iTerm2、GNOME Terminal、Windows Terminal |
| Shell        | 运行在终端里的**命令解释器**：读取你输入的命令，解析后交给系统执行 | bash、zsh、sh                                                 |
| 内核（Kernel）   | 真正管理进程、文件、内存、设备的系统核心                | Linux、Darwin                                                |

要点：
- 终端只是「窗口」，Shell 才是「理解命令的程序」。同一个终端里可以换不同的 Shell。
- Shell 本身也是一个普通的可执行程序（比如 `/bin/bash`），不是什么神秘的东西。
- Shell 既是**交互式命令行**（你一条条敲），也是一门**脚本语言**（把命令写进文件批量执行）。
### Shell 执行一条命令时在做什么
你敲下 `ls -l /tmp` 并回车，Shell 大致做了这几步：
1. 读取一行输入
2. 解析：按空格切分单词，展开变量、通配符等
3. 找到 `ls` 这个程序（在 `PATH` 里查找）
4. 创建子进程去运行它
5. 等它结束，拿到退出状态码，再显示下一个提示符

> **C 语言类比**：这个过程就相当于 `fork()` + `execvp()` + `waitpid()`。
> 而且 `ls -l /tmp` 传给 `ls` 程序的，正是 C 里熟悉的 `argv`：
> `argv[0]="ls"`, `argv[1]="-l"`, `argv[2]="/tmp"`。
## 1.2 常见的 Shell

| Shell  | 简介                                                            |
| ------ | ------------------------------------------------------------- |
| `sh`   | 最早的 Bourne Shell 及其兼容实现，强调可移植性。现在 `/bin/sh` 通常是指向别的 Shell 的链接 |
| `bash` | Bourne-Again Shell，GNU 项目出品，Linux 上最常见的默认 Shell，功能比 sh 丰富     |
| `zsh`  | 功能强大、交互体验好；macOS 从 Catalina（10.15）起默认使用 zsh                   |
| `dash` | 轻量、启动快，Debian / Ubuntu 中 `/bin/sh` 通常指向它，只保证 POSIX 基础特性       |
| `fish` | 交互友好，但语法**不兼容** POSIX，一般不用来写通用脚本                              |

### 动手：看看你现在用的是哪个 Shell
```bash
echo $SHELL          # 你的"登录 Shell"（默认 Shell）
echo $0              # 当前正在运行的 Shell
cat /etc/shells      # 系统里允许使用的 Shell 列表
bash --version       # 查看 bash 版本
```

> 注意：`$SHELL` 表示默认登录 Shell，不一定等于你当前正在用的。比如默认是 zsh，
> 你在里面输入 `bash` 进入了 bash，此时 `$SHELL` 仍显示 zsh，而 `$0` 会显示 bash。

### 切换 Shell
```bash
bash          # 临时进入 bash（输入 exit 退出，回到原来的 Shell）
zsh           # 临时进入 zsh
chsh -s /bin/zsh   # 永久更改默认 Shell（需重新登录生效）
```
## 1.3 POSIX 标准
**POSIX** 是一套由 IEEE 制定的操作系统接口标准，其中也规定了 Shell 的语法和常用命令的行为。
- 只用 POSIX 规定的特性写出的脚本，能在 sh、dash、bash、zsh 等多种 Shell 上运行 —— 这叫**可移植**。
- bash 在 POSIX 之上加了很多扩展（数组、`[[ ]]`、`{1..5}` 等），这些在 `dash` 里可能不能用。

举个例子：

```bash
# bash 专有写法，用 sh（dash）运行可能报错
if [[ "$a" == "x" ]]; then echo yes; fi

# POSIX 写法，各种 Shell 都能跑
if [ "$a" = "x" ]; then echo yes; fi
```

初学阶段先学 bash 即可，第 10 章会专门讲 bash 与 POSIX sh 的差异。
## 1.4 命令的基本格式
```
命令  [选项]  [参数]
```

用空格分隔，例如：

```bash
ls -l /tmp
```

- `ls`：命令（要运行的程序）
- `-l`：选项（改变命令的行为）
- `/tmp`：参数（命令作用的对象）
### 选项的写法

| 形式    | 例子                      | 说明                       |
| ----- | ----------------------- | ------------------------ |
| 短选项   | `-l`、`-a`               | 一个横线加一个字母                |
| 合并短选项 | `ls -la` 等价于 `ls -l -a` | 多个短选项可以连写                |
| 长选项   | `--all`、`--help`        | 两个横线加单词，可读性好（常见于 GNU 工具） |
| 带值的选项 | `-n 5` 或 `--lines=5`    | 例如 `head -n 5 file.txt`  |

> 不同系统上的命令选项可能有差异。比如 GNU 版本（Linux）和 BSD 版本（macOS）的 `ls`、`sed` 等
> 部分选项并不一致；遇到报错时，以本机的 `man 命令名` 为准。
### 双横线 `--` 的作用
`--` 表示「选项到此结束，后面都是参数」。当文件名以 `-` 开头时有用：

```bash
touch -- -weird-name.txt     # 创建名为 -weird-name.txt 的文件
rm -- -weird-name.txt        # 删除它
```
### 一行写多条命令
```bash
cd /tmp; ls; pwd      # 用分号依次执行，不管前面成功与否
cd /tmp && ls         # 前一条成功才执行下一条
cd /nonexist || echo "目录不存在"   # 前一条失败才执行下一条
```

长命令可以用反斜杠 `\` 换行：

```bash
ls -l \
   /tmp
```
## 1.5 命令的类型
你敲的「命令」其实分好几类，可以用 `type` 查看：

```bash
type cd       # cd is a shell builtin（内建命令）
type ls       # ls is /bin/ls（外部程序；macOS 与 Linux 路径可能不同）
type ll       # 若设置了别名，会显示 ll is aliased to ...
which ls      # 只显示外部程序的路径
```

| 类型            | 说明                         | 例子                          |
| ------------- | -------------------------- | --------------------------- |
| 内建命令（builtin） | Shell 自己实现的，不创建新进程         | `cd`、`echo`、`export`、`type` |
| 外部命令          | 磁盘上的可执行文件，在 `PATH` 指定的目录里找 | `ls`、`grep`、`cp`            |
| 别名（alias）     | 给命令起的外号                    | `alias ll='ls -l'`          |
| 函数            | 用户自己定义的 Shell 函数           | 第 7 章讲                      |

> 为什么 `cd` 必须是内建命令？因为「改变当前目录」只对**当前进程**有效，
> 如果用子进程去执行，子进程改了目录就退出了，对 Shell 本身毫无影响。
### PATH：Shell 去哪里找命令
```bash
echo $PATH
# 输出类似：/usr/local/bin:/usr/bin:/bin   （冒号分隔多个目录）
```

Shell 会按顺序在这些目录里找同名的可执行文件，找到第一个就执行。
找不到就报 `command not found`。
## 1.6 运行脚本
先写一个最简单的脚本。新建文件 `hello.sh`：

```bash
#!/usr/bin/env bash
echo "Hello, Shell!"
```
### 方式一：把脚本交给解释器运行
```bash
bash hello.sh
```

- 不需要执行权限
- 相当于「用 bash 这个程序去读 hello.sh」
### 方式二：直接当程序运行
```bash
chmod +x hello.sh     # 先给文件加上"可执行"权限（只需一次）
./hello.sh
```

- 需要执行权限
- 为什么要写 `./`？因为当前目录通常不在 `PATH` 里，必须明确指出路径。
- 内核会读取文件第一行的 shebang，决定用哪个解释器来运行。

常见报错：

| 报错                                           | 原因与解决                                                         |
| -------------------------------------------- | ------------------------------------------------------------- |
| `Permission denied`                          | 没有执行权限，运行 `chmod +x hello.sh`                                 |
| `command not found`                          | 直接敲了 `hello.sh`，忘了写 `./`                                      |
| `bad interpreter: No such file or directory` | shebang 路径不对，或文件是 Windows 换行（CRLF），可用 `dos2unix` 转换（如果系统里有的话） |

### 方式三：`source` 或 `.`（在当前 Shell 中执行）
```bash
source hello.sh
. hello.sh            # 点号是 source 的另一种写法
```

方式一、二会开一个**子进程**来运行脚本，脚本结束后里面设置的变量就没了；
`source` 则是在**当前 Shell 里**逐行执行，变量会保留下来。

验证一下。新建 `set_name.sh`：

```bash
#!/usr/bin/env bash
NAME="Alice"
```

```bash
bash set_name.sh
echo "NAME=[$NAME]"      # 输出 NAME=[]    （子进程里的变量，父 Shell 看不到）

source set_name.sh
echo "NAME=[$NAME]"      # 输出 NAME=[Alice]
```

这个区别在后面写配置文件、加载环境变量时会反复用到。
### 三种方式对比

| 方式 | 命令 | 需要 x 权限 | 在哪里执行 | 脚本里的变量保留？ |
|------|------|:---:|-----------|:---:|
| 交给解释器 | `bash a.sh` | 否 | 子进程 | 否 |
| 直接执行 | `./a.sh` | 是 | 子进程（由 shebang 决定解释器） | 否 |
| source | `source a.sh` | 否 | 当前 Shell | 是 |

## 1.7 Shebang（`#!`）
脚本第一行以 `#!` 开头，后面写解释器的路径，叫 **shebang**：

```bash
#!/bin/bash              # 直接指定 bash 的路径
#!/usr/bin/env bash      # 通过 env 在 PATH 里查找 bash（更灵活）
#!/bin/sh                # 用 sh，写 POSIX 脚本时用
#!/usr/bin/env python3   # 不仅限于 Shell，别的脚本语言也能用
```

- 它只在**直接执行**（`./a.sh`）时起作用，决定用哪个解释器。
- 用 `bash a.sh` 运行时，shebang 会被忽略（因为那一行以 `#` 开头，对 bash 来说是注释）。
- 为什么推荐 `#!/usr/bin/env bash`？因为 bash 并不总在 `/bin/bash`
  （比如有些系统上装了更新版本的 bash 在别处），`env` 会按 `PATH` 去找。
- 在 shebang 之后的 `#` 开头的内容都是注释。
## 1.8 综合例子：一个会读参数的脚本
新建 `args.sh`：

```bash
#!/usr/bin/env bash

# 这是注释
echo "脚本名:     $0"
echo "第一个参数:  $1"
echo "第二个参数:  $2"
echo "参数个数:    $#"
```

运行：

```bash
chmod +x args.sh
./args.sh foo bar
```

输出：

```
脚本名:     ./args.sh
第一个参数: foo
第二个参数: bar
参数个数:   2
```

> **C 语言类比**：`$0` 对应 `argv[0]`，`$1` 对应 `argv[1]`，`$#` 对应 `argc - 1`。
> 注意 Shell 的 `$#` **不含**脚本名本身。
## 本章小结
- 终端是窗口，Shell 是命令解释器，内核才是真正干活的；Shell 通过 fork + exec 运行外部命令。
- bash 是学习首选；zsh 是 macOS 默认；写通用脚本要注意 POSIX 兼容。
- 命令格式：`命令 [选项] [参数]`；用 `type` 区分内建命令和外部命令。
- 运行脚本三种方式：`bash a.sh`、`./a.sh`（需 `chmod +x`）、`source a.sh`（在当前 Shell 执行）。
- 脚本第一行写 shebang，推荐 `#!/usr/bin/env bash`。
## 练习
1. 用 `echo $SHELL` 和 `echo $0` 查看你当前的 Shell，思考两者为什么可能不同。
2. 用 `type` 分别查看 `cd`、`echo`、`ls`、`grep`，判断各自是内建命令还是外部命令。
3. 写一个 `hello.sh`，分别用 `bash hello.sh` 和 `./hello.sh` 运行，观察 `Permission denied` 并解决它。
4. 写一个 `set_name.sh`，对比 `bash set_name.sh` 与 `source set_name.sh` 之后变量是否存在。
5. 写一个 `args.sh`，打印前三个参数和参数总数，用不同数量的参数运行它。
6. 思考：为什么 `cd` 不能做成外部命令？
### 参考答案要点
1. 如果默认是 zsh、你在其中手动执行了 `bash`，`$SHELL` 仍是 zsh，而 `$0` 是 bash。
2. `cd`、`echo` 在 bash 中是内建命令（`echo` 同时也存在外部版本，但 Shell 优先用内建的）；`ls`、`grep` 是外部命令。
3. 用 `chmod +x hello.sh` 加权限后再运行。
4. 只有 `source` 之后，当前 Shell 才能看到 `NAME`。
5. 用 `$1 $2 $3 $#`，参数不足时对应位置为空。
6. 子进程改变工作目录不会影响父进程，所以 `cd` 必须由 Shell 自己执行。
