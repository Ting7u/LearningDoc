# GCC 常用命令完全整理（分类 + 实战）

> 适用场景：Linux 下的 C/C++ 开发、系统编程、后端开发和嵌入式入门。\
> 目标：理解 GCC
> 的编译流程，掌握常用参数，并能独立编译多文件项目、链接库和调试程序。

------------------------------------------------------------------------

## 目录

1.  [GCC 编译流程](#一gcc-编译流程)
2.  [基础编译命令](#二基础编译命令)
3.  [控制编译阶段](#三控制编译阶段)
4.  [语言标准与警告](#四语言标准与警告)
5.  [调试、优化与错误检测](#五调试优化与错误检测)
6.  [头文件与库文件](#六头文件与库文件)
7.  [静态库与动态库](#七静态库与动态库)
8.  [宏定义与依赖关系](#八宏定义与依赖关系)
9.  [常用辅助命令](#九常用辅助命令)
10. [C++ 与交叉编译](#十c与交叉编译)
11. [真实项目常用命令组合](#十一真实项目常用命令组合)
12. [参数速查表](#十二参数速查表)
13. [完整实战练习](#十三完整实战练习)
14. [学习建议与官方文档](#十四学习建议与官方文档)

------------------------------------------------------------------------

## 一、GCC 编译流程

GCC
不只是一个编译器入口，它会协调多个工具完成从源代码到可执行程序的过程。

``` text
main.c
  |
  v
预处理（Preprocessing）
  |  处理头文件、宏定义、条件编译
  v
main.i
  |
  v
编译（Compilation）
  |  将预处理后的 C 代码转换成汇编
  v
main.s
  |
  v
汇编（Assembly）
  |  将汇编代码转换成机器指令
  v
main.o
  |
  v
链接（Linking）
  |  将目标文件和需要的库组合起来
  v
可执行文件
```

四个阶段：

  阶段     输入       输出         主要工作
  -------- ---------- ------------ --------------------------
  预处理   `.c`       `.i`         展开头文件、宏和条件编译
  编译     `.i`       `.s`         生成汇编代码
  汇编     `.s`       `.o`         生成目标文件
  链接     `.o`、库   可执行文件   解析符号并组合目标代码

GCC 默认会完成整个流程：

``` bash
gcc main.c -o main
```

运行：

``` bash
./main
```

## 二、基础编译命令

### 1. 编译一个 C 文件

准备 `main.c`：

``` c
#include <stdio.h>

int main(void)
{
    printf("Hello GCC!\n");
    return 0;
}
```

编译：

``` bash
gcc main.c
```

默认生成 `a.out`，运行：

``` bash
./a.out
```

### 2. 指定输出文件名

``` bash
gcc main.c -o main
./main
```

参数说明：

-   `gcc`：调用 GCC 编译器驱动程序。
-   `main.c`：源代码文件。
-   `-o`：指定输出文件名称。
-   `main`：输出的可执行文件。

### 3. 编译多个源文件

项目结构：

``` text
project/
├── main.c
├── add.c
└── add.h
```

`add.h`：

``` c
#ifndef ADD_H
#define ADD_H

int add(int a, int b);

#endif
```

`add.c`：

``` c
#include "add.h"

int add(int a, int b)
{
    return a + b;
}
```

`main.c`：

``` c
#include <stdio.h>
#include "add.h"

int main(void)
{
    printf("%d\n", add(10, 20));
    return 0;
}
```

编译：

``` bash
gcc main.c add.c -o main
./main
```

输出：

``` text
30
```

**注意：** 头文件通常不需要单独放进编译命令。它通过 `#include`
被预处理器引入。

## 三、控制编译阶段

  参数            作用                 常见输出
  --------------- -------------------- ---------------
  `-E`            只进行预处理         预处理文本
  `-S`            编译到汇编阶段       `.s`
  `-c`            编译、汇编，不链接   `.o`
  `-o`            指定输出文件         自定义
  `-save-temps`   保留中间文件         `.i`、`.s` 等

### 1. `-E`：只进行预处理

``` bash
gcc -E main.c -o main.i
```

查看结果：

``` bash
less main.i
```

可以观察到：

-   `#include` 引入的头文件内容已经展开。
-   `#define` 定义的宏已经替换。
-   条件编译已经处理。

例如：

``` c
#define NUM 100

int main(void)
{
    return NUM;
}
```

预处理后，`NUM` 会被替换为 `100`。

### 2. `-S`：生成汇编代码

``` bash
gcc -S main.c -o main.s
```

查看：

``` bash
less main.s
```

生成的是汇编代码，不是目标文件。

### 3. `-c`：生成目标文件

``` bash
gcc -c main.c -o main.o
```

生成 `main.o`。它是目标文件，通常还需要链接才能成为可执行程序。

### 4. 手动完成四个阶段

``` bash
# 第一步：预处理
gcc -E main.c -o main.i

# 第二步：编译成汇编
gcc -S main.i -o main.s

# 第三步：汇编成目标文件
gcc -c main.s -o main.o

# 第四步：链接
gcc main.o -o main

# 运行
./main
```

### 5. 保存中间文件

``` bash
gcc -save-temps main.c -o main
```

适合学习和排查编译过程问题。

## 四、语言标准与警告

### 1. 指定 C 语言标准：`-std`

``` bash
gcc -std=c11 main.c -o main
```

常见标准：（stdandard）

  参数         标准
  ------------ ------
  `-std=c89`   C89
  `-std=c99`   C99
  `-std=c11`   C11
  `-std=c17`   C17
  `-std=c23`   C23

实际可用标准取决于 GCC 版本。

### 2. `-Wall`：开启常见警告
Warning all（所有警告）

``` bash
gcc -Wall main.c -o main
```

`-Wall` 会开启一组常见警告，但它并不意味着开启了 GCC 的全部警告。

### 3. `-Wextra`：开启额外警告
Warning extra（额外警告）

``` bash
gcc -Wall -Wextra main.c -o main
```

### 4. `-Wpedantic`：提示标准相关问题
Warning pedantic（吹毛求疵的）
代码不符合 C 语言标准时给出警告

``` bash
gcc -std=c11 -Wall -Wextra -Wpedantic main.c -o main
```

### 5. `-Werror`：将警告视为错误
Warning → error（警告变错误）

``` bash
gcc -Wall -Wextra -Werror main.c -o main
```

启用后，警告会被当作错误处理，导致编译失败。

### 6. `-fsyntax-only`：只做语法和语义检查
flag + syntax only（只检查语法）

``` bash
gcc -fsyntax-only main.c
```

不生成目标文件或可执行文件。

**日常开发推荐：**

``` bash
gcc -std=c11 -Wall -Wextra -Wpedantic main.c -o main
```

## 五、调试、优化与错误检测

### 1. `-g`：生成调试信息

``` bash
gcc -g main.c -o main
```

之后可以使用 GDB：

``` bash
gdb ./main
```

常用 GDB 命令：

``` gdb
break main
run
next
step
print 变量名
continue
quit
```

### 2. 优化等级

  参数    作用                 常见用途
  ------- -------------------- --------------------
  `-O0`   不进行常规优化       调试、学习
  `-O1`   基础优化             需要一定优化
  `-O2`   较全面的优化         常见发布构建
  `-O3`   更积极的优化         经测试确认有收益时
  `-Os`   优先考虑代码体积     体积受限场景
  `-Og`   兼顾调试体验的优化   开发调试

调试构建：

``` bash
gcc -g -O0 -Wall -Wextra main.c -o main
```

常见发布构建：

``` bash
gcc -O2 -DNDEBUG -Wall -Wextra main.c -o main
```

`-DNDEBUG` 会定义 `NDEBUG` 宏，通常用于关闭 `assert`
断言。只有确认发布构建不需要这些断言时才使用。

### 3. `-fsanitize=address`：检测部分内存错误

``` bash
gcc -g -O1 \
    -fsanitize=address \
    -fno-omit-frame-pointer \
    main.c -o main
```

运行：

``` bash
./main
```

AddressSanitizer 可以帮助发现多种内存错误，例如：

-   堆内存越界。
-   栈内存越界。
-   Use-after-free。
-   部分非法内存访问。

它不能代替完整测试，也不能保证发现所有问题。

## 六、头文件与库文件

这是实际项目中非常重要的一部分。例如使用 Redis 的 C 客户端
hiredis、Kafka 的 C 客户端 librdkafka
时，通常都需要处理头文件和库文件路径。

### 1. `-I`：指定头文件搜索目录

项目结构：

``` text
project/
├── main.c
└── include/
    └── mylib.h
```

代码：

``` c
#include "mylib.h"

int main(void)
{
    return 0;
}
```

编译：

``` bash
gcc main.c -I./include -o main
```

`-I` 告诉编译器额外到指定目录搜索头文件。

### 2. `-L`：指定库文件搜索目录

假设目录中有：

``` text
project/
├── main.c
└── lib/
    └── libmylib.so
```

编译：

``` bash
gcc main.c -L./lib -lmylib -o main
```

-   `-L./lib`：告诉链接器到 `./lib` 搜索库文件。
-   `-lmylib`：链接名为 `mylib` 的库。

通常 `-lmylib` 对应 `libmylib.so` 或 `libmylib.a`。

### 3. `-l`：指定链接库

例如使用数学库：

``` c
#include <stdio.h>
#include <math.h>

int main(void)
{
    printf("%f\n", sqrt(25.0));
    return 0;
}
```

编译：

``` bash
gcc main.c -lm -o main
```

`-lm` 表示链接数学库 `libm`。

### 4. 组合使用 `-I`、`-L`、`-l`

``` bash
gcc main.c \
    -I./include \
    -L./lib \
    -lmylib \
    -o main
```

### 5. `-pthread`：POSIX 线程支持

示例：

``` c
#include <pthread.h>
#include <stdio.h>

void *task(void *arg)
{
    (void)arg;
    printf("Hello thread!\n");
    return NULL;
}

int main(void)
{
    pthread_t tid;

    pthread_create(&tid, NULL, task, NULL);
    pthread_join(tid, NULL);

    return 0;
}
```

编译：

``` bash
gcc main.c -pthread -o main
```

建议使用 `-pthread`，而不是只写 `-lpthread`。它还会设置相应的编译选项。

## 七、静态库与动态库

### 1. 基本概念

| 类型  | 常见后缀  | 基本特点               |
| --- | ----- | ------------------ |
| 静态库 | `.a`  | 链接时将需要的目标代码纳入可执行文件 |
| 动态库 | `.so` | 程序运行时由动态链接器加载共享库   |

### 2. 创建静态库

准备 `add.c`：

``` c
int add(int a, int b)
{
    return a + b;
}
```

第一步：编译为目标文件。

``` bash
gcc -c add.c -o add.o
```

第二步：使用 `ar` 创建静态库。

``` bash
ar rcs libadd.a add.o
```

生成：

``` text
libadd.a
```

第三步：编写 `main.c`：

``` c
#include <stdio.h>

int add(int a, int b);

int main(void)
{
    printf("%d\n", add(10, 20));
    return 0;
}
```

第四步：链接静态库。

``` bash
gcc main.c -L. -ladd -o main
```

运行：

``` bash
./main
```

输出：

``` text
30
```

### 3. 创建动态库

第一步：生成位置无关目标代码。

``` bash
gcc -fPIC -c add.c -o add.o
```

第二步：生成动态库。

``` bash
gcc -shared add.o -o libadd.so
```

第三步：链接程序。

``` bash
gcc main.c -L. -ladd -o main
```

第四步：运行时让动态链接器找到库。

``` bash
LD_LIBRARY_PATH=. ./main
```

`LD_LIBRARY_PATH`
适合临时测试。正式部署应合理配置系统动态库搜索路径或程序运行环境。

## 八、宏定义与依赖关系

### 1. `-D`：从命令行定义宏

``` bash
gcc -DDEBUG main.c -o main
```

等价于在代码中定义：

``` c
#define DEBUG
```

也可以指定值：

``` bash
gcc -DVERSION=3 main.c -o main
```

代码中可以这样使用：

``` c
#ifdef DEBUG
printf("debug mode\n");
#endif
```

### 2. `-U`：取消宏定义

``` bash
gcc -UDEBUG main.c -o main
```

用于取消指定宏的定义。

### 3. `-M`：生成头文件依赖信息

``` bash
gcc -M main.c
```

会输出源文件及其包含的头文件依赖，通常包括系统头文件。

### 4. `-MM`：只列出用户头文件依赖

``` bash
gcc -MM main.c
```

通常用于生成 Makefile 所需的依赖信息，不列出系统头文件。

## 九、常用辅助命令

  命令                               用途
  ---------------------------------- --------------------------
  `gcc --version`                    查看 GCC 版本
  `gcc -v`                           查看详细编译过程及配置
  `gcc --help`                       查看命令帮助
  `gcc -dumpmachine`                 查看编译器目标平台三元组
  `gcc -print-search-dirs`           查看 GCC 搜索目录
  `gcc -print-file-name=libgcc.a`    查询指定库文件路径
  `gcc -M main.c`                    输出完整头文件依赖
  `gcc -MM main.c`                   输出用户头文件依赖
  `gcc -save-temps main.c -o main`   保存中间文件

示例：

``` bash
gcc --version
gcc -v
gcc -dumpmachine
gcc -print-search-dirs
gcc -print-file-name=libgcc.a
```

注意：`ldd` 不是 GCC 参数，而是 Linux
系统工具。它可以查看程序依赖的动态库：

``` bash
ldd ./main
```

## 十、C++ 与交叉编译

### 1. GCC 与 G++

| 命令    | 用途                  |
| ----- | ------------------- |
| `gcc` | GCC 编译器驱动程序，常用于 C   |
| `g++` | GCC 编译器驱动程序，常用于 C++ |

C 程序：

``` bash
gcc main.c -o main
```

C++ 程序：

``` bash
g++ -std=c++17 -Wall -Wextra -g main.cpp -o main
```

C++ 项目建议使用 `g++`，它会自动处理 C++ 标准库链接。

### 2. 交叉编译

普通编译：

``` bash
gcc main.c -o main
```

交叉编译示例：

``` bash
arm-linux-gnueabihf-gcc main.c -o main
```

常见工具链名称：

``` text
arm-linux-gnueabihf-gcc
aarch64-linux-gnu-gcc
riscv64-linux-gnu-gcc
```

这些工具链需要提前安装。目标架构、ABI、目标系统库和运行环境必须匹配。

## 十一、真实项目常用命令组合

### 场景一：普通 C 程序

``` bash
gcc -std=c11 -Wall -Wextra -g main.c -o main
```

### 场景二：多文件项目

``` bash
gcc -std=c11 -Wall -Wextra -g \
    main.c add.c sub.c \
    -o main
```

### 场景三：多线程项目

``` bash
gcc -std=c11 -Wall -Wextra -g \
    main.c \
    -pthread \
    -o main
```

### 场景四：链接第三方库

``` bash
gcc main.c \
    -I/usr/local/include \
    -L/usr/local/lib \
    -lhiredis \
    -o main
```

这里以 hiredis
为例，实际头文件路径、库目录和库名称应以本机安装情况为准。

### 场景五：调试版本

``` bash
gcc -g -O0 -Wall -Wextra main.c -o main
```

### 场景六：发布版本

``` bash
gcc -O2 -DNDEBUG -Wall -Wextra main.c -o main
```

### 场景七：检查动态库依赖

``` bash
gcc main.c -o main
ldd ./main
```

## 十二、参数速查表

  分类       参数                   作用
  ---------- ---------------------- --------------------------------
  基础       `-o`                   指定输出文件
  阶段       `-E`                   只预处理
  阶段       `-S`                   生成汇编
  阶段       `-c`                   生成目标文件，不链接
  阶段       `-save-temps`          保存中间文件
  标准       `-std=c11`             指定 C 标准
  警告       `-Wall`                开启一组常见警告
  警告       `-Wextra`              开启额外警告
  警告       `-Werror`              警告视为错误
  警告       `-Wpedantic`           提示标准相关问题
  调试       `-g`                   生成调试信息
  优化       `-O0`                  不进行常规优化
  优化       `-O2`                  较全面的优化
  优化       `-O3`                  更积极的优化
  优化       `-Os`                  优先考虑程序体积
  头文件     `-I`                   指定头文件目录
  链接       `-L`                   指定库搜索目录
  链接       `-l`                   指定链接库
  链接       `-shared`              生成共享库
  链接       `-static`              请求静态链接
  链接       `-fPIC`                生成位置无关代码
  多线程     `-pthread`             启用 POSIX 线程支持
  宏定义     `-D`                   定义宏
  宏定义     `-U`                   取消宏定义
  检测       `-fsanitize=address`   启用 AddressSanitizer
  依赖       `-M`                   生成完整头文件依赖
  依赖       `-MM`                  生成用户头文件依赖
  目标平台   `-march`               指定目标指令集架构
  目标平台   `-mcpu`                指定目标 CPU（依目标后端而定）

## 十三、完整实战练习：构建多文件 C 项目

这个练习把头文件、多个源文件、目标文件、链接和调试串联起来。

### 第一步：创建项目目录

``` bash
mkdir gcc_demo
cd gcc_demo

mkdir include src build
```

目录结构：

``` text
gcc_demo/
├── include/
│   └── math_utils.h
├── src/
│   ├── main.c
│   └── math_utils.c
└── build/
```

### 第二步：编写头文件

`include/math_utils.h`：

``` c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

int add(int a, int b);
int sub(int a, int b);

#endif
```

### 第三步：编写实现文件

`src/math_utils.c`：

``` c
#include "math_utils.h"

int add(int a, int b)
{
    return a + b;
}

int sub(int a, int b)
{
    return a - b;
}
```

### 第四步：编写主程序

`src/main.c`：

``` c
#include <stdio.h>
#include "math_utils.h"

int main(void)
{
    int a = 100;
    int b = 30;

    printf("add = %d\n", add(a, b));
    printf("sub = %d\n", sub(a, b));

    return 0;
}
```

### 第五步：分别编译两个源文件

``` bash
gcc -std=c11 -Wall -Wextra -g \
    -I./include \
    -c src/main.c \
    -o build/main.o
```

``` bash
gcc -std=c11 -Wall -Wextra -g \
    -I./include \
    -c src/math_utils.c \
    -o build/math_utils.o
```

### 第六步：链接目标文件

``` bash
gcc build/main.o build/math_utils.o -o build/app
```

### 第七步：运行程序

``` bash
./build/app
```

预期输出：

``` text
add = 130
sub = 70
```

### 第八步：使用 GDB 调试

``` bash
gdb ./build/app
```

在 GDB 中执行：

``` gdb
break main
run
next
print a
print b
continue
quit
```

至此，完成了一个简单的多文件 C 项目构建和调试。

## 十四、学习建议与官方文档

### 1. 参数学习优先级

**第一优先级：必须熟练**

``` text
gcc
-o
-c
-E
-S
-I
-L
-l
-Wall
-g
```

**第二优先级：项目开发常用**

``` text
-std
-O0
-O2
-pthread
-shared
-fPIC
-D
```

**第三优先级：深入学习时掌握**

``` text
-fsanitize
-march
-mcpu
-static
-M
-MM
```

### 2. 后续学习顺序

建议按下面的顺序学习构建工具：

1.  **GCC**：理解预处理、编译、汇编和链接。
2.  **Make**：通过 Makefile 自动执行编译命令。
3.  **CMake**：管理更大型的 C/C++ 项目，并生成 Make 或 Ninja
    等构建系统。

三者不是同一种工具：

-   GCC：负责编译和链接。
-   Make：根据构建规则执行命令。
-   CMake：描述项目构建方式并生成构建系统。

### 3. 官方文档

-   [GCC 官方文档](https://gcc.gnu.org/onlinedocs/)
-   [GCC
    参数分类速查](https://gcc.gnu.org/onlinedocs/gcc/Option-Summary.html)
-   [GCC
    编译流程及基础参数](https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html)
-   [GCC
    预处理参数](https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html)

------------------------------------------------------------------------

**学习建议：**
不需要一次性背完所有参数。先熟练掌握基础编译、多文件编译、`-I/-L/-l`、调试和链接库，再通过
Make 和 CMake 把这些命令组织成可复用的项目构建流程。
