# GNU Make 从入门到实战

> 本文单独讲解 GNU Make，不与 GCC、CMake 的教程合并。目标是从基础概念开始，逐步学会使用 Make 管理多文件 C 项目。

## 目录

1. Make 是什么
2. 第一个 Makefile
3. Makefile 基本语法
4. Make 如何判断是否重新构建
5. 变量
6. 自动变量
7. all、clean 和 .PHONY
8. 多文件 C 项目实战
9. 模式规则
10. 头文件依赖
11. 常用命令和排错
12. Make、GCC、CMake 的关系

---

# 1. Make 是什么？

假设有一个 C 项目：

```text
project/
├── main.c
├── add.c
└── sub.c
```

不用 Make 时，你可能每次都手动执行：

```bash
gcc main.c add.c sub.c -o app
```

项目文件多了，命令会越来越长。更麻烦的是，只修改了 `add.c`，却可能把所有源文件重新编译一遍。

**Make 是构建工具**，它读取 Makefile 中的规则，分析文件之间的依赖关系，判断哪些目标需要生成或更新，然后执行相应命令。

- GCC：真正执行 C 代码的编译、汇编和链接。
- Make：决定何时执行哪些命令，以及执行顺序。
- Makefile：描述项目应该怎样构建。

Make 不只可以调用 GCC，也可以调用 Clang、脚本、资源处理工具等。

# 2. 第一个 Makefile

创建目录和源文件：

```bash
mkdir make_demo
cd make_demo
```

创建 `main.c`：

```c
#include <stdio.h>

int main(void)
{
    printf("Hello Make!\\n");
    return 0;
}
```

直接用 GCC 编译：

```bash
gcc main.c -o app
./app
```

现在创建一个名为 `Makefile` 的文件：

```makefile
app:
	gcc main.c -o app
```

执行：

```bash
make
```

Make 会读取当前目录的 `Makefile`，找到默认目标 `app`，并执行下面的 GCC 命令。然后可以运行：

```bash
./app
```

也可以明确指定目标：

```bash
make app
```

**重要：** Makefile 中命令行前面必须是真正的 Tab 字符，不能用普通空格代替，否则常见报错是 `missing separator`。

# 3. Makefile 的基本语法

最基本的规则格式：

```makefile
目标: 依赖
	命令
```

例如：

```makefile
app: main.c add.c
	gcc main.c add.c -o app
```

含义：

- `app` 是目标：希望生成的文件。
- `main.c add.c` 是依赖：生成 `app` 需要的文件。
- `gcc ...` 是配方（recipe）：告诉 Make 如何生成目标。

一条规则可以有多个依赖，也可以有多条命令：

```makefile
app: main.c add.c sub.c
	gcc -Wall main.c add.c sub.c -o app
	@echo "build complete"
```

`@` 表示执行命令时不把命令本身回显出来，只显示命令输出。

# 4. Make 如何判断是否需要重新构建？

看这个规则：

```makefile
app: main.c
	gcc main.c -o app
```

第一次执行 `make` 时，`app` 不存在，Make 就执行 GCC 命令。

第二次执行时，如果 `app` 已存在，而且 `main.c` 没有比 `app` 更新，Make 通常会认为目标已经是最新的，不再执行命令。

如果修改了 `main.c`，它的修改时间就会比 `app` 新。再次执行 `make`，Make 会重新执行编译命令。

可以简化理解为：

```text
目标不存在吗？
  是 -> 执行构建命令
  否 -> 检查依赖是否比目标新
          有更新 -> 重新构建
          没更新 -> 目标是最新的
```

Make 默认主要根据文件存在情况和修改时间判断，并不会理解 C 代码的语义。头文件依赖也需要声明，或由编译器生成依赖文件。

# 5. 变量

为了避免重复写编译器和编译选项，可以使用变量：

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11

app: main.c add.c
	$(CC) $(CFLAGS) main.c add.c -o app
```

常见变量：

| 变量         | 全称                        | 常见用途               |
| ---------- | :------------------------ | ------------------ |
| `CC`       | C Compiler                | C 编译器              |
| `CXX`      | C++ (X 代表加号旋转 45°)        | C++ 编译器            |
| `CFLAGS`   | C Flags                   | C 编译选项             |
| `CXXFLAGS` | C++ Flags                 | C++ 编译选项           |
| `CPPFLAGS` | C PreProcessor Flags      | 预处理选项，例如 `-I`、`-D` |
| `LDFLAGS`  | LinkDer Flags(ld 的 flags) | 链接选项，例如 `-L`       |
| `LDLIBS`   | LinkDer Libraries         | 要链接的库，例如 `-lm`     |

示例：

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g
CPPFLAGS = -Iinclude
LDLIBS = -lm

app: main.c
	$(CC) $(CPPFLAGS) $(CFLAGS) main.c -o app $(LDLIBS)
```

常见赋值方式：

```makefile
A = hello       # 递归展开：使用时再展开
B := hello      # 立即展开：赋值时展开
C ?= hello      # 只有 C 尚未定义时才赋值
CFLAGS += -Wextra  # 追加内容
```

入门阶段先熟悉 `=`、`+=` 和 `$(变量名)` 即可。

# 6. 自动变量

Make 提供特殊变量，自动引用当前规则中的目标和依赖：

| 自动变量 | 含义 |
|---|---|
| `$@` | 当前规则的目标 |
| `$<` | 当前规则的第一个依赖 |
| `$^` | 当前规则的所有依赖（通常会去重） |
| `$?` | 比目标新的依赖 |

例如：

```makefile
app: main.c add.c
	gcc $^ -o $@
```

效果相当于：

```bash
gcc main.c add.c -o app
```

再看：

```makefile
main.o: main.c
	gcc -c $< -o $@
```

相当于：

```bash
gcc -c main.c -o main.o
```

# 7. all、clean 和 .PHONY

通常会提供一个默认入口 `all`，以及清理构建产物的 `clean`：

```makefile
.PHONY: all clean

all: app

app: main.c
	gcc main.c -o app

clean:
	rm -f app
```

执行：

```bash
make        # 默认构建第一个目标 all
make all    # 构建 all
make clean  # 删除生成的 app
```

`.PHONY` 表示这些目标是伪目标，不对应需要生成的普通文件。比如目录里刚好有一个叫 `clean` 的文件，如果没有 `.PHONY: clean`，Make 可能会误以为 `clean` 已经是最新的，从而不执行清理命令。

# 8. 多文件 C 项目实战

创建如下结构：

```text
calculator/
├── Makefile
├── include/
│   └── calc.h
└── src/
    ├── main.c
    ├── add.c
    └── sub.c
```

`include/calc.h`：

```c
#ifndef CALC_H
#define CALC_H

int add(int a, int b);
int sub(int a, int b);

#endif
```

`src/add.c`：

```c
#include "calc.h"

int add(int a, int b)
{
    return a + b;
}
```

`src/sub.c`：

```c
#include "calc.h"

int sub(int a, int b)
{
    return a - b;
}
```

`src/main.c`：

```c
#include <stdio.h>
#include "calc.h"

int main(void)
{
    printf("10 + 5 = %d\\n", add(10, 5));
    printf("10 - 5 = %d\\n", sub(10, 5));
    return 0;
}
```

第一版 Makefile，直接编译所有源文件：

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11 -g
CPPFLAGS = -Iinclude

TARGET = calculator
SRCS = src/main.c src/add.c src/sub.c

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(SRCS)
	$(CC) $(CPPFLAGS) $(CFLAGS) $(SRCS) -o $@

clean:
	rm -f $(TARGET)
```

执行：

```bash
make
./calculator
make clean
```

这版可以工作，但只要任意源文件变化，链接命令就会重新编译所有源文件。项目较大时，可以把每个 `.c` 单独编译为 `.o`。

# 9. 模式规则：分别生成 .o 文件

构建过程变成：

```text
main.c -> main.o --+
add.c  -> add.o  --+--> calculator
sub.c  -> sub.o  --+
```

`.o` 是目标文件，通常包含编译后的机器代码，但还没有完成最终程序的链接。

Makefile：

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11 -g
CPPFLAGS = -Iinclude

TARGET = calculator
OBJS = src/main.o src/add.o src/sub.o

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $@

src/%.o: src/%.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(TARGET) $(OBJS)
```

这里的：

```makefile
src/%.o: src/%.c
```

是模式规则。`%` 匹配相同的部分，例如 `src/add.o` 对应 `src/add.c`。命令中的 `$@` 是目标（例如 `src/add.o`），`$<` 是第一个依赖（例如 `src/add.c`）。

现在试试：

```bash
make
make
```

第二次通常不会重新编译。只修改 `src/add.c` 再运行 `make`，通常只重新编译 `add.o`，然后重新链接程序。这就是增量构建。

# 10. 头文件依赖

上面的模式规则只声明了 `.o` 依赖 `.c`，没有声明它也依赖 `calc.h`。如果只修改头文件，Make 可能不知道哪些目标文件需要重新编译。

小项目可以手动写：

```makefile
src/main.o: include/calc.h
src/add.o: include/calc.h
src/sub.o: include/calc.h
```

但头文件多了，手动维护很麻烦。GCC 可以自动生成依赖文件：

- `-MMD`：生成用户头文件依赖信息，写入 `.d` 文件。
- `-MP`：为头文件生成空规则，降低头文件被删除后出现缺失规则错误的风险。

改进版完整 Makefile：

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -std=c11 -g
CPPFLAGS = -Iinclude

TARGET = calculator
SRCS = src/main.c src/add.c src/sub.c
OBJS = $(SRCS:.c=.o)
DEPS = $(OBJS:.o=.d)

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $@

src/%.o: src/%.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -MMD -MP -c $< -o $@

-include $(DEPS)

clean:
	rm -f $(TARGET) $(OBJS) $(DEPS)
```

`$(SRCS:.c=.o)` 是变量替换引用，把列表中的 `.c` 后缀替换为 `.o`。`$(OBJS:.o=.d)` 同理。

`-include` 表示尝试包含这些依赖文件；第一次构建时 `.d` 文件尚不存在，也不会因此直接报错。构建过程中 GCC 会生成 `.d` 文件，之后 Make 就能据此追踪头文件依赖。

# 11. 常用命令和排错

| 命令 | 作用 |
|---|---|
| `make` | 构建默认目标 |
| `make all` | 构建 all 目标 |
| `make clean` | 执行 clean 目标 |
| `make app` | 构建指定目标 |
| `make -j4` | 最多使用 4 个并行任务 |
| `make -j` | 尽可能并行构建 |
| `make -n` | 只打印将要执行的命令，不真正执行 |
| `make -B` | 强制认为目标需要重新构建 |
| `make -f xxx.mk` | 指定 Makefile 文件 |
| `make -C dir` | 先进入指定目录再执行 |

常见错误：

### 11.1 `missing separator`

通常是命令行前面用了空格而不是 Tab。检查 Makefile 的配方行。

### 11.2 `No rule to make target`

常见原因是文件名或路径写错、文件不存在，或者没有提供生成该依赖的规则。核对目录结构和 Makefile 中的路径。

### 11.3 修改头文件却没有重新编译

检查是否声明了头文件依赖，或是否使用 `-MMD -MP` 生成并包含 `.d` 文件。

### 11.4 `make clean` 没有执行

检查是否声明了 `.PHONY: clean`，以及目标名是否写错。

# 12. Make、GCC、CMake 的关系

常见组合如下：

```text
你编写的 C/C++ 源代码
          |
          v
        CMake
  （可选：生成构建系统）
          |
          v
       Makefile
  （描述构建规则）
          |
          v
         Make
  （读取规则、组织执行）
          |
          v
         GCC
  （编译、汇编、链接）
          |
          v
       可执行程序
```

这只是常见组合，不是唯一方式：

- 可以直接写 Makefile，不使用 CMake。
- CMake 可以生成 Makefile，也可以生成 Ninja 等构建系统。
- Make 可以调用 GCC、Clang 或其他命令。
- GCC 不依赖 Make 才能工作，你也可以手动执行 GCC 命令。

# 13. 建议的学习顺序

1. 能写出 `目标: 依赖` 规则。
2. 理解 Make 根据文件时间戳判断是否需要重建。
3. 学会 `CC`、`CFLAGS` 等变量。
4. 理解 `$@`、`$<`、`$^`。
5. 学会 `all`、`clean`、`.PHONY`。
6. 能把多个 `.c` 分别编译成 `.o`。
7. 理解 `%.o: %.c` 模式规则。
8. 学会自动生成头文件依赖 `.d`。
9. 能读懂并修改真实项目的 Makefile。

## 最后记住

Make 的核心不是记住一堆命令，而是理解依赖关系：

```makefile
目标: 依赖
	生成目标的命令
```

Make 根据依赖关系决定构建顺序，再根据文件状态决定哪些命令需要执行。理解这个模型，变量、自动变量、模式规则和多文件项目就能串起来了。
