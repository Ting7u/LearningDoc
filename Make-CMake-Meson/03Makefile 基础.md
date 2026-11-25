# 第 3 章 Makefile 基础

> 本章目标:掌握 Makefile 的基本语法与运行方式,理解"规则三要素"、伪目标、增量构建原理、命令前缀、注释续行,以及"每行命令一个独立 shell"这一关键特性。本章结束时,你应能为小型 C 项目写出可用的 Makefile。
>
> 本教程以 **GNU Make** 为准(`make --version` 查看,建议 4.x;macOS 自带的是 3.81,较旧,可 `brew install make` 得到 `gmake`)。

**本章结构**

1. 规则三要素
2. 第一个 Makefile(从 v1 演进到 v3)
3. 运行 make:命令行选项
4. 默认目标与 `.DEFAULT_GOAL`
5. 伪目标 `.PHONY`
6. 增量构建原理
7. 命令前缀:`@`、`-`、`+`
8. 注释与续行
9. 多命令与 shell 调用方式
10. 常见错误信息
11. 小结与练习

---

## 3.1 规则三要素:目标、依赖、命令

### 3.1.1 基本语法

```makefile
目标 ... : 依赖 ...
	命令
	命令
	...
```

| 要素 | 说明 |
|---|---|
| **目标(target)** | 要生成的文件名,或一个动作名(伪目标) |
| **依赖(prerequisites)** | 生成目标所需的文件/目标,以空格分隔 |
| **命令(recipe)** | 生成目标要执行的 shell 命令,可有多行 |

整个"目标 + 依赖 + 命令"合称一条**规则(rule)**。

### 3.1.2 必须用 Tab 缩进

**命令行必须以 Tab 字符开头**(不是空格)。这是 make 最著名的"坑"。

```makefile
app: main.o
    gcc main.o -o app      # ✘ 用了 4 个空格
```

```
Makefile:2: *** missing separator.  Stop.
```

排查办法:

```bash
cat -A Makefile        # Tab 显示为 ^I
```

编辑器提示:VS Code 右下角把缩进设为 "Tab";`vim` 中 `:set noexpandtab`。

> GNU Make 3.82+ 可用 `.RECIPEPREFIX = >` 改变命令前缀字符,但不建议,会降低可读性与兼容性。

### 3.1.3 一条规则的含义

```makefile
main.o: src/main.c include/mathutils.h
	gcc -Iinclude -c src/main.c -o main.o
```

读作:"要生成 `main.o`,先确保 `src/main.c` 和 `include/mathutils.h` 就绪;若 `main.o` 不存在或比任一依赖旧,则执行下面的命令。"

### 3.1.4 一个目标可以有多条规则(依赖合并)

```makefile
main.o: src/main.c
	gcc -Iinclude -c $< -o $@

main.o: include/mathutils.h       # 只追加依赖,没有命令
```

**只有一条规则可以有命令**(否则 make 会警告 "overriding recipe"),但其他规则可以只追加依赖。这个特性在"自动依赖生成"时非常有用(第 6 章)。

### 3.1.5 多个目标、依赖

```makefile
a.o b.o: common.h        # 等价于 a.o: common.h  和  b.o: common.h
```

---

## 3.2 第一个 Makefile:从 v1 到 v3

延用第 1 章 demo:

```
demo/
├── include/mathutils.h
├── src/main.c
└── src/mathutils.c
```

### 3.2.1 v1:最直白的写法

```makefile
app: src/main.c src/mathutils.c include/mathutils.h
	gcc -Iinclude src/main.c src/mathutils.c -o app
```

运行:

```bash
$ make
gcc -Iinclude src/main.c src/mathutils.c -o app
$ make
make: 'app' is up to date.
```

**问题**:改任何一个文件,都要把所有 `.c` 重新编译。

### 3.2.2 v2:分离编译与链接

```makefile
app: main.o mathutils.o
	gcc main.o mathutils.o -o app

main.o: src/main.c include/mathutils.h
	gcc -Iinclude -c src/main.c -o main.o

mathutils.o: src/mathutils.c include/mathutils.h
	gcc -Iinclude -c src/mathutils.c -o mathutils.o

clean:
	rm -f main.o mathutils.o app
```

这就是第 2 章依赖图的直接翻译。验证增量:

```bash
$ make
gcc -Iinclude -c src/main.c -o main.o
gcc -Iinclude -c src/mathutils.c -o mathutils.o
gcc main.o mathutils.o -o app

$ touch src/main.c
$ make
gcc -Iinclude -c src/main.c -o main.o      # 只重编 main.o
gcc main.o mathutils.o -o app              # 然后重新链接
```

**仍有问题**:`clean` 如果恰好存在一个名为 `clean` 的文件会怎样?(见 3.5 节)

### 3.2.3 v3:加上 `.PHONY`、默认目标

```makefile
.PHONY: all clean

all: app

app: main.o mathutils.o
	gcc main.o mathutils.o -o app

main.o: src/main.c include/mathutils.h
	gcc -Iinclude -c src/main.c -o main.o

mathutils.o: src/mathutils.c include/mathutils.h
	gcc -Iinclude -c src/mathutils.c -o mathutils.o

clean:
	rm -f main.o mathutils.o app
```

> 此版本命令里重复出现 `gcc`、`-Iinclude`,下一章用变量、第 5 章用模式规则与自动变量来消除重复。

---

## 3.3 运行 make:命令行选项

### 3.3.1 基本用法

```bash
make                  # 构建默认目标(Makefile 里第一个目标)
make app              # 构建指定目标
make clean            # 执行 clean 目标
make main.o mathutils.o     # 一次指定多个目标
make CC=clang         # 命令行传变量(第 4 章)
```

make 按顺序查找 Makefile 文件:`GNUmakefile` → `makefile` → `Makefile`。推荐用 `Makefile`。

### 3.3.2 常用选项速查

| 选项 | 全称 | 含义 |
|---|---|---|
| `-f FILE` | `--file` | 指定 Makefile 文件名,可多次使用 |
| `-C DIR` | `--directory` | 先切换到 DIR 再执行(递归 make 常用) |
| `-j [N]` | `--jobs` | 并行执行 N 个任务;不带数字则不限 |
| `-n` | `--dry-run` / `--just-print` | 只打印命令,不执行(**调试利器**) |
| `-k` | `--keep-going` | 某目标失败后继续构建其他不依赖它的目标 |
| `-B` | `--always-make` | 无条件重建所有目标 |
| `-s` | `--silent` / `--quiet` | 不打印命令 |
| `-p` | `--print-data-base` | 打印内置规则与变量(含 Makefile 中所有) |
| `-r` | `--no-builtin-rules` | 禁用内置隐含规则 |
| `-R` | `--no-builtin-variables` | 禁用内置变量 |
| `-q` | `--question` | 不执行,只用退出码表示是否 up to date(0 表示已最新) |
| `-t` | `--touch` | 只 touch 目标而不重建(骗过 make,慎用) |
| `-W FILE` | `--what-if` | 假设 FILE 刚被修改过 |
| `-e` | `--environment-overrides` | 环境变量覆盖 Makefile 中的变量 |
| `-i` | `--ignore-errors` | 忽略所有命令错误 |
| `-d` / `--debug[=FLAGS]` | | 打印调试信息 |
| `-w` | `--print-directory` | 打印进入/离开目录信息 |
| `--no-print-directory` | | 不打印目录信息 |
| `-O[TYPE]` | `--output-sync` | 并行时同步输出(`-Otarget`、`-Oline`、`-Orecurse`) |
| `-l [LOAD]` | `--load-average` | 系统负载超过 LOAD 时不启动新任务 |
| `--warn-undefined-variables` | | 引用未定义变量时警告 |
| `-v` | `--version` | 版本 |

### 3.3.3 几个必用实例

**① `-n`:先看看会做什么**

```bash
$ make -n
gcc -Iinclude -c src/main.c -o main.o
gcc -Iinclude -c src/mathutils.c -o mathutils.o
gcc main.o mathutils.o -o app
```

**② `-j`:并行**

```bash
make -j4              # 4 个并行任务
make -j$(nproc)       # Linux 取 CPU 核数
make -j$(sysctl -n hw.ncpu)    # macOS
```

并行构建的前提是依赖写得完整,否则会随机失败(第 8 章详述)。

**③ `-k`:尽可能多地继续**

```bash
make -k               # 某个 .c 编译失败,仍编译其余 .c,最后汇总报错
```

适合一次看到所有编译错误。

**④ `-B`:强制重建**

```bash
make -B               # 等价于先 clean 再 make(但不删除多余文件)
```

**⑤ `-C`:在其他目录里运行**

```bash
make -C build         # 等价于 cd build && make
make -C libs/foo
```

**⑥ `-f`:使用非默认名**

```bash
make -f build.mk
make -f common.mk -f project.mk       # 多个 Makefile 合并读取
```

**⑦ `-p`:查看 make 的"大脑"**

```bash
make -p -f /dev/null | less           # 只看内置规则和变量
make -pn | grep -A3 '^app'            # 看某目标的最终规则
```

**⑧ `-q`:脚本中判断是否最新**

```bash
make -q app && echo "up to date" || echo "needs rebuild"
```

### 3.3.4 退出码与错误

- 命令返回非 0 → make 报错 `Error N` 并停止(除非命令前加 `-` 或使用 `-k`/`-i`)。
- `make` 本身退出码:0 成功,1(用 `-q` 时表示需要重建),2 出错。

### 3.3.5 输出里常见的提示

```
make: Nothing to be done for 'all'.     # 目标已最新,且没有命令要执行
make: 'app' is up to date.              # 目标文件已最新
make: *** No rule to make target 'xxx'. Stop.     # 找不到生成 xxx 的规则
```

---

## 3.4 默认目标与 `.DEFAULT_GOAL`

### 3.4.1 默认目标

直接运行 `make` 时,构建**Makefile 中出现的第一个"普通目标"**(不以 `.` 开头的、非模式规则的目标)。

```makefile
clean:                    # ← 第一个目标!直接 make 会执行 clean!
	rm -f *.o app

app: main.o
	gcc $^ -o $@
```

这是常见陷阱。惯例:**第一个目标写成 `all`**(最常见)或 `default`。

```makefile
all: app tests            # 作为默认目标,聚合其他目标
```

### 3.4.2 用 `.DEFAULT_GOAL` 显式指定(GNU make 3.81+)

```makefile
.DEFAULT_GOAL := app      # 无论 app 写在哪,直接 make 就构建 app

clean:
	rm -f ...

app: ...
```

可通过变量 `$(.DEFAULT_GOAL)` 查询当前默认目标;可在 Makefile 中多次赋值,最后一次生效;赋空值(`.DEFAULT_GOAL :=`)则重置,取之后出现的第一个目标。

### 3.4.3 `.DEFAULT` 特殊目标(区分!)

```makefile
.DEFAULT:                 # 当 make 找不到规则构建某目标时,执行此规则
	@echo "不知道如何构建 $@"
```

`.DEFAULT` 与 `.DEFAULT_GOAL` 不同:前者是"兜底规则",后者是"默认构建谁"。

---

## 3.5 伪目标 `.PHONY`

### 3.5.1 问题:目标并非真实文件

```makefile
clean:
	rm -f *.o app
```

如果目录中恰好有名为 `clean` 的**文件**,且没有依赖,make 认为 `clean` 目标"已是最新",于是不执行:

```bash
$ touch clean
$ make clean
make: 'clean' is up to date.      # 没有清理!
```

### 3.5.2 解决:声明伪目标

```makefile
.PHONY: clean
clean:
	rm -f *.o app
```

`.PHONY` 目标**永远视为过期,总是执行**,且 make 不会为它去找隐含规则(效率略高)。

### 3.5.3 常见伪目标

| 名称 | 约定用途 |
|---|---|
| `all` | 构建所有主要产物(默认目标) |
| `clean` | 删除构建产物 |
| `distclean` | 比 clean 更彻底,恢复到解包后状态 |
| `install` / `uninstall` | 安装/卸载 |
| `test` / `check` | 运行测试 |
| `dist` | 打包源码 |
| `help` | 显示帮助 |
| `run` | 构建后运行 |
| `format` / `lint` | 代码格式化/静态检查 |

### 3.5.4 一个实用的 `help` 目标

```makefile
.PHONY: help
help:
	@echo "可用目标:"
	@echo "  make        构建程序"
	@echo "  make clean  清理"
	@echo "  make run    运行"
```

### 3.5.5 伪目标可以有依赖,形成"聚合"

```makefile
.PHONY: all build test
all: build test           # all 依赖其他伪目标
build: app
test: app
	./app --selftest
```

### 3.5.6 其他有趣的用法:强制重建(`FORCE` 惯用法)

对不支持 `.PHONY` 的老 make,可以写:

```makefile
clean: FORCE
	rm -f ...
FORCE:
```

(`FORCE` 是个没有命令也没有依赖的目标,而且不存在同名文件,因此总是"更新",带动 `clean` 也被视为过期。)

---

## 3.6 增量构建原理

### 3.6.1 make 决定是否重建的算法

对每个目标 T,递归地:

1. 先**递归构建 T 的所有依赖**。
2. 若 T 是文件目标:
   - T **不存在** → 执行规则;
   - 任一依赖的修改时间 **晚于** T → 执行规则;
   - 否则 → 跳过。
3. 若 T 是伪目标 → 总是执行规则。

### 3.6.2 例子推演

```
main.c  10:00     main.o  10:01     app  10:02
```

1. `make`:`app` 的依赖 `main.o`;`main.o` 的依赖 `main.c`(叶子);`main.c`(10:00)不晚于 `main.o`(10:01)→ 跳过;`main.o`(10:01)不晚于 `app`(10:02)→ 跳过 → "up to date"。
2. `touch main.c`(现在 10:30):`main.c` 晚于 `main.o` → 重编 `main.o`(10:30);于是 `main.o` 晚于 `app` → 重新链接。

### 3.6.3 缺失的依赖 → 没有规则

```
make: *** No rule to make target 'foo.h', needed by 'main.o'.  Stop.
```

依赖本身必须**要么存在,要么有规则可生成**。

### 3.6.4 观察 make 的决策过程

```bash
make --debug=b       # basic:只显示重建原因
make -d | less       # 全部细节
make -n              # 干跑
make -W src/main.c -n    # 假设 main.c 刚改过,看会重编什么
```

### 3.6.5 时间戳带来的陷阱

| 场景 | 后果 |
|---|---|
| 时钟不同步(NFS、虚拟机、容器) | 目标/依赖时间错乱,漏编或过度重编 |
| `git checkout` 回退到旧版本文件 | 文件 mtime 变为"现在",触发大量重编;或旧提交文件 mtime 反而更旧而不重编 |
| 命令生成了比预期更旧的文件 | 每次都重编 |
| 命令没有真正产出目标文件 | 每次都重编,且下游也永远过期 |

规则:**规则的命令必须真正生成(或更新)名为目标的文件。**

---

## 3.7 命令前缀:`@`、`-`、`+`

### 3.7.1 默认行为:回显命令

make 在执行每条命令前,默认把它打印出来:

```makefile
hello:
	echo "Hello"
```

```
$ make hello
echo "Hello"
Hello
```

### 3.7.2 `@`:不回显该命令

```makefile
hello:
	@echo "Hello"
```

```
$ make hello
Hello
```

常用于 `echo`、`printf` 等输出提示的命令;也可以全局用 `make -s` 或 `.SILENT:`。

**技巧**:让构建输出简洁,类似内核风格:

```makefile
%.o: %.c
	@echo "  CC    $@"
	@$(CC) $(CFLAGS) -c $< -o $@
```

可再用变量控制是否回显:

```makefile
V ?= 0
ifeq ($(V),0)
  Q := @
else
  Q :=
endif

%.o: %.c
	$(Q)$(CC) $(CFLAGS) -c $< -o $@
```

用 `make V=1` 显示完整命令。

### 3.7.3 `-`:忽略该命令的错误

```makefile
clean:
	-rm main.o         # 即使文件不存在导致 rm 失败,也继续
	-rm app
```

否则 `rm` 返回非 0 → make 会终止。

更推荐直接用 `rm -f`;`-` 前缀的含义是"失败也无所谓",只在必要时用。

### 3.7.4 `+`:即使在 `-n`/`-q`/`-t` 下也执行

主要用于**递归调用 make**(`$(MAKE)` 已自动带 `+` 效果):

```makefile
sub:
	+$(MAKE) -C subdir
```

### 3.7.5 前缀可组合

```makefile
	@-rm -f temp       # 不回显且忽略错误
```

### 3.7.6 相关特殊目标

| 目标 | 作用 |
|---|---|
| `.SILENT:` | 所有命令不回显(可指定目标,如 `.SILENT: clean`) |
| `.IGNORE:` | 忽略所有命令错误 |
| `.DELETE_ON_ERROR:` | 命令失败时删除可能不完整的目标文件(**强烈建议开启**) |

```makefile
.DELETE_ON_ERROR:
```

---

## 3.8 注释与续行

### 3.8.1 注释

以 `#` 开头到行尾。

```makefile
# 这是注释
CC = gcc        # 行尾注释:注意 gcc 后面的空格也会被计入变量值!
```

> **陷阱**:`CC = gcc # comment` 中 `gcc` 与 `#` 之间的空格属于变量值(即 `"gcc "`),通常无害,但在拼接路径/文件名时会出问题。保险做法是把注释放在单独一行。

**命令行中的 `#`**:

- 命令行(Tab 开头)中的 `#` **不是** make 的注释,而是传给 shell,由 shell 处理(在 shell 里 `#` 也是注释)。make 会把它当命令的一部分打印出来:

```makefile
all:
	echo hi   # 这个注释会被 shell 吃掉,但 make 回显时仍会显示
```

- 若要在变量值或规则中使用字面 `#`,用反斜杠转义:`\#`。

### 3.8.2 续行

行尾加反斜杠 `\`,下一行继续。

```makefile
SRCS = main.c \
       mathutils.c \
       util.c
```

**注意**:

- `\` 后面不能有空格。
- 在**变量定义**里,续行会被替换为单个空格。
- 在**命令**里,反斜杠+换行会原样交给 shell(shell 同样识别为续行),所以可以把一条长命令拆成多行:

```makefile
app: $(OBJS)
	$(CC) $(LDFLAGS) \
	      $(OBJS) \
	      $(LDLIBS) \
	      -o $@
```

- 续行里的缩进 Tab 会被 make 去掉第一个 Tab(命令续行中)。

### 3.8.3 空行与行格式

- 空行被忽略。
- 一行以 Tab 开头,且上文刚出现过规则(或在规则的命令上下文中)→ 命令行。
- 否则一行以 Tab 开头 → 若没有规则上下文,可能被报 `commands commence before first target`。

---

## 3.9 多命令与 shell 调用方式

### 3.9.1 每条命令一个独立的 shell 进程

**这是 make 最重要的特性之一。**make 对规则中的每一行命令,都用 `/bin/sh -c "命令行"` **单独启动**一个 shell。

```makefile
test:
	cd src
	pwd
```

```
$ make test
cd src
pwd
/home/me/demo          # ← 仍然是原目录!cd 没影响下一行
```

因为 `cd src` 在第一个 shell 里执行完就退出了。

### 3.9.2 正确写法:放在同一行(用 `&&` 或 `;`)

```makefile
test:
	cd src && pwd
```

或者用 `\` 把多行连成一个 shell 命令:

```makefile
test:
	cd src && \
	pwd && \
	ls
```

**注意:`;` 与 `&&` 的区别**

```makefile
	cd nonexist ; rm -rf *      # 危险!cd 失败仍执行 rm,删掉当前目录!
	cd nonexist && rm -rf *     # 安全:cd 失败则不执行 rm
```

### 3.9.3 shell 变量与 `$$`

make 先对命令行做一遍**变量展开**,把 `$` 当作 make 变量引用符。要把 `$` 传给 shell,需写成 `$$`:

```makefile
list:
	for f in *.c; do echo $$f; done
	echo "PID is $$$$"           # 传给 shell 的是 $$ ,即 shell 进程 PID
```

对比:

```makefile
bad:
	for f in *.c; do echo $f; done     # make 把 $f 当成变量 f(空值) → echo 空
```

> `$f` 在 make 中会被解释为 `$(f)`(单字符变量名可以省略括号),这是常见坑。**shell 变量一律写 `$$var`。**

### 3.9.4 变量不跨行

```makefile
x:
	FOO=bar
	echo $$FOO          # 空!FOO 只存在于上一行的 shell 中
```

修复:

```makefile
x:
	FOO=bar; echo $$FOO
```

### 3.9.5 `.ONESHELL`:整个规则一个 shell(GNU make 3.82+)

```makefile
.ONESHELL:
test:
	cd src
	pwd                # 此时 pwd 显示 src 目录
	FOO=bar
	echo $$FOO         # 输出 bar
```

注意副作用:

- 所有命令共享一个 shell,前缀 `@` 只对第一行生效(整个脚本);
- 遇错不会立刻停止,除非开头 `set -e`(或设 `.SHELLFLAGS = -e -c`)。

### 3.9.6 指定 shell:`SHELL` 与 `.SHELLFLAGS`

默认使用 `/bin/sh`,即使你的登录 shell 是 bash/zsh。因此命令里用 bash 特性(`[[ ]]`、数组、`{1..3}`)可能失败(尤其在 Ubuntu,`/bin/sh` 是 dash)。

```makefile
SHELL := /bin/bash
.SHELLFLAGS := -eu -o pipefail -c       # 严格模式:遇错即停、未定义变量报错、管道任一失败即失败
```

注意:**`SHELL` 不会从环境变量继承**,必须在 Makefile 里设置。

### 3.9.7 管道、重定向、通配符

命令里可以自由使用 shell 特性:

```makefile
count:
	ls *.c | wc -l > count.txt
	@echo "done"
```

通配符分两处展开:

| 位置 | 谁展开 | 示例 |
|---|---|---|
| 命令里 | shell | `rm -f *.o` |
| 目标/依赖里 | make | `app: *.o`(需注意变量中不会自动展开,需 `$(wildcard)`,第 7 章) |

### 3.9.8 命令太长/太多:可读性建议

```makefile
install: app
	install -d $(DESTDIR)/usr/bin
	install -m 755 app $(DESTDIR)/usr/bin/app
```

多行 shell 脚本可以写在 `define` 里(第 4 章),或抽成独立脚本文件调用。

---

## 3.10 常见错误信息速查

| 错误信息 | 常见原因 | 处理 |
|---|---|---|
| `*** missing separator. Stop.` | 命令行没用 Tab;或语法错误 | `cat -A` 检查,Tab 缩进 |
| `*** No rule to make target 'X'` | 文件 X 不存在且无规则可生成;文件名拼错;缺少依赖文件 | 检查路径、拼写、是否忘了 `include` |
| `make: Nothing to be done for 'X'` | 目标已最新(或没有命令的聚合目标) | 正常 |
| `make: 'X' is up to date.` | 目标文件已最新 | 正常,可 `touch` 源文件或 `make -B` |
| `*** commands commence before first target` | 以 Tab 开头的行出现在任何规则之前 | 去掉缩进或补目标 |
| `*** target pattern contains no '%'` | 静态模式规则写法错误(第 5 章) | |
| `Circular X <- Y dependency dropped.` | 循环依赖 | 修正依赖图 |
| `warning: overriding recipe for target 'X'` | 同一目标定义了多份命令 | 保留一份 |
| `*** [Makefile:N: X] Error 1` | 命令返回非 0,N 是行号 | 看上面编译器的实际错误 |
| `Makefile:N: *** unterminated call to function` | 函数括号未闭合 | 检查 `$( )` |
| `*** recipe commences before first target` | 同上(新版措辞) | |

---

## 3.11 综合示例:一个"够用"的小型 Makefile

```makefile
# ---------- 基本配置 ----------
.DEFAULT_GOAL := all
.DELETE_ON_ERROR:

# ---------- 目标 ----------
.PHONY: all clean run help

all: app

app: main.o mathutils.o
	@echo "  LINK  $@"
	@gcc main.o mathutils.o -o app

main.o: src/main.c include/mathutils.h
	@echo "  CC    $@"
	@gcc -Wall -Iinclude -c src/main.c -o main.o

mathutils.o: src/mathutils.c include/mathutils.h
	@echo "  CC    $@"
	@gcc -Wall -Iinclude -c src/mathutils.c -o mathutils.o

run: app
	./app

clean:
	rm -f main.o mathutils.o app

help:
	@echo "targets: all (default), run, clean, help"
```

试一试:

```bash
make            # 构建
make            # Nothing to be done for 'all'.
make run
make -n clean
make -B         # 强制重建
```

---

## 3.12 本章小结

1. 规则 = `目标: 依赖` + **Tab 缩进**的命令;make 按**时间戳**比较决定是否执行。
2. 默认目标是第一个目标,惯例写 `all`;也可用 `.DEFAULT_GOAL`。
3. 不对应文件的目标必须声明为 `.PHONY`(`all`、`clean`、`install`、`test`...)。
4. 命令前缀:`@` 不回显、`-` 忽略错误、`+` 在 `-n` 下仍执行。
5. **每行命令一个独立 shell**:`cd` 与变量不跨行,要么写同一行(`&&`、`\`),要么 `.ONESHELL`;传给 shell 的 `$` 要写 `$$`。
6. 默认 shell 是 `/bin/sh`;用 bash 特性需显式设置 `SHELL`。
7. 常用调试组合:`make -n`、`make -p`、`make --debug=b`、`make -d`。
8. 建议开启 `.DELETE_ON_ERROR:`,避免失败后残留不完整的目标文件被当作最新。

## 3.13 练习

1. 手写第 3.2.2 节的 v2 Makefile,依次执行 `make`、`touch include/mathutils.h && make`、`touch src/main.c && make`,说明每次重编了什么。
2. 故意把一条命令的 Tab 换成空格,看错误信息;用 `cat -A` 找出问题。
3. 创建名为 `clean` 的文件,验证没有 `.PHONY` 时 `make clean` 失效;加上 `.PHONY` 后恢复。
4. 把 `clean` 放在 Makefile 第一位,执行 `make` 观察后果;再用 `.DEFAULT_GOAL` 修复。
5. 写一个目标,先 `cd /tmp` 再 `pwd`,比较"两行写法"、`&&` 写法与 `.ONESHELL` 写法的输出。
6. 写一个目标,用 shell 循环打印 `src/*.c` 的每个文件名;分别写 `$f` 与 `$$f`,体会区别。
7. 用 `V=1`/`V=0` 实现可切换的"安静/详细"输出模式(参见 3.7.2)。
8. 使用 `make -n`、`make -W src/main.c -n`、`make --debug=b`,预测并验证重建行为。
9. 用 `make -p -f /dev/null` 找出 `CC`、`CXX`、`RM`、`AR` 的默认值,以及 `%.o: %.c` 对应的内置规则。
10. (思考)为什么命令没有真正生成目标文件会导致每次都重建?在什么场景下会遇到(提示:`touch` 之外的"标记文件"用法)?

## 3.14 速查卡

```makefile
目标: 依赖1 依赖2          # 规则(命令必须 Tab 缩进)
	命令

.PHONY: all clean           # 伪目标
.DEFAULT_GOAL := all        # 默认目标
.DELETE_ON_ERROR:           # 失败时删除半成品
.ONESHELL:                  # 整个规则一个 shell
SHELL := /bin/bash          # 指定 shell

@cmd     不回显          -cmd    忽略错误          +cmd    -n 下仍执行
$$var    传给 shell 的 $        \  行尾续行        #       注释
```

```bash
make [目标] [VAR=值] -f FILE -C DIR -j N -n -k -B -s -p -q
```
