# GCC 常用命令速查

## 1. 基本编译

### 1.1 直接编译并生成可执行文件
```bash
gcc hello.c -o hello
```
将 `hello.c` 编译并链接，生成可执行文件 `hello`。若省略 `-o`，默认输出为 `a.out`。

### 1.2 只编译不链接
```bash
gcc -c hello.c -o hello.o
```
生成目标文件（`.o`），不进行链接，常用于多文件项目分步编译。

### 1.3 多文件编译
```bash
gcc main.c utils.c -o app
```
或先分别生成 `.o` 再统一链接：
```bash
gcc -c main.c -o main.o
gcc -c utils.c -o utils.o
gcc main.o utils.o -o app
```

---

## 2. 编译各阶段的中间产物

GCC 编译流程大致为：预处理 → 编译 → 汇编 → 链接。

### 2.1 只做预处理
```bash
gcc -E hello.c -o hello.i
```
展开宏、头文件包含等，输出预处理后的源码。

### 2.2 只编译到汇编代码
```bash
gcc -S hello.c -o hello.s // AT&T
gcc -S -masm=intel test.c // Inter
```
生成汇编代码文件（`.s`），不进行汇编和链接。

### 2.3 只汇编不链接
```bash
gcc -c hello.s -o hello.o
```
将汇编代码转成目标文件。

---

## 3. 警告与调试

### 3.1 开启常用警告
```bash
gcc -Wall -Wextra hello.c -o hello
```
`-Wall` 开启大部分常见警告，`-Wextra` 开启更多额外警告，写代码时建议默认加上。

### 3.2 将警告当作错误处理
```bash
gcc -Wall -Werror hello.c -o hello
```
出现任何警告都会导致编译失败，便于强制代码规范。

### 3.3 生成调试信息（配合 gdb 使用）
```bash
gcc -g hello.c -o hello
```
保留调试符号信息，之后可用 `gdb ./hello` 调试。

---

## 4. 优化选项

```bash
gcc -O0 hello.c -o hello   # 不优化（默认，便于调试）
gcc -O1 hello.c -o hello   # 基本优化
gcc -O2 hello.c -o hello   # 常用的生产环境优化级别
gcc -O3 hello.c -o hello   # 更激进的优化，可能增大体积
gcc -Os hello.c -o hello   # 优化代码体积
```
一般开发调试用 `-O0 -g`，发布构建用 `-O2`。

---

## 5. 头文件与库

### 5.1 指定头文件搜索路径
```bash
gcc hello.c -I/usr/local/include -o hello
```
`-I` 添加头文件（`.h`）的查找目录。

### 5.2 指定库文件搜索路径与链接库
```bash
gcc hello.c -L/usr/local/lib -lm -o hello
```
- `-L` 指定库文件（`.so`/`.a`）的查找目录
- `-lm` 链接数学库 `libm`（`-l` 后跟库名，去掉 `lib` 前缀和后缀）

常见示例：链接 pthread 库
```bash
gcc thread_demo.c -lpthread -o thread_demo
```

### 5.3 静态链接
```bash
gcc hello.c -static -o hello
```
生成不依赖动态库的可执行文件（体积更大，但部署更方便）。

---

## 6. 生成与使用库文件

### 6.1 生成静态库
```bash
gcc -c utils.c -o utils.o
ar rcs libutils.a utils.o
```
`ar rcs` 将目标文件打包成静态库 `libutils.a`。

### 6.2 生成动态库（共享库）
```bash
gcc -fPIC -c utils.c -o utils.o
gcc -shared utils.o -o libutils.so
```
- `-fPIC` 生成位置无关代码（动态库必需）
- `-shared` 生成共享库

### 6.3 使用自建库编译
```bash
gcc main.c -L. -lutils -o app
```
`-L.` 表示在当前目录下查找库文件。

---

## 7. 宏定义

### 7.1 编译时定义宏
```bash
gcc -DDEBUG hello.c -o hello
```
等价于在代码里写 `#define DEBUG`。

### 7.2 定义带值的宏
```bash
gcc -DVERSION=\"1.0.0\" hello.c -o hello
```

---

## 8. 指定标准

```bash
gcc -std=c11 hello.c -o hello
gcc -std=c99 hello.c -o hello
gcc -std=gnu11 hello.c -o hello   # GNU 扩展版 C11
```
指定使用的 C 语言标准版本。

---

## 9. 查看依赖与预处理信息

### 9.1 生成头文件依赖关系（常用于 Makefile）
```bash
gcc -M hello.c
```
输出该文件依赖的所有头文件，格式适合放入 Makefile。

### 9.2 查看编译器版本
```bash
gcc --version
```

### 9.3 查看默认包含路径等详细信息
```bash
gcc -v hello.c -o hello
```

---

## 10. 常用组合示例

### 10.1 开发调试常用组合
```bash
gcc -Wall -Wextra -g -O0 -std=c11 main.c -o main
```

### 10.2 发布构建常用组合
```bash
gcc -Wall -O2 -std=c11 main.c -o main
```

### 10.3 多文件项目 + 头文件路径 + 数学库
```bash
gcc -Wall -Iinclude src/*.c -lm -o app
```

---
## 11. nm
```c
gcc -c person.c student.c   # 只编译，生成 person.o 和 student.o，不链接
nm student.o
nm person.o
```
`nm` 是一个命令行工具，作用是**列出目标文件（`.o`）或可执行文件里的符号**，也就是函数名、全局变量名，以及它们的类型和地址。它属于 GNU binutils，装了 gcc 的 Linux 通常都自带
![[Pasted image 20260929201217.png]]
![[Pasted image 20260929201242.png]]![[Pasted image 20260929201249.png]]
## 附：常用选项一览表

| 选项 | 作用 |
|------|------|
| `-o <file>` | 指定输出文件名 |
| `-c` | 只编译/汇编，不链接 |
| `-E` | 只预处理 |
| `-S` | 只编译到汇编代码 |
| `-g` | 生成调试信息 |
| `-Wall` | 开启常用警告 |
| `-Wextra` | 开启额外警告 |
| `-Werror` | 警告视为错误 |
| `-O0`~`-O3`/`-Os` | 优化级别 |
| `-I<dir>` | 添加头文件搜索路径 |
| `-L<dir>` | 添加库文件搜索路径 |
| `-l<name>` | 链接指定库 |
| `-D<macro>` | 定义宏 |
| `-std=<standard>` | 指定语言标准 |
| `-static` | 静态链接 |
| `-fPIC` | 生成位置无关代码 |
| `-shared` | 生成共享库 |
| `-v` | 显示详细编译过程 |
| `--version` | 查看版本 |
## 12. readelf

```c
extern int buf[];
int *bufp0 = &buf[0];
int *bufp1;
void swap()
{
    int temp;
    bufp1 = &buf[1];
    temp = *bufp0;
    *bufp0 = *bufp1;
    *bufp1 = temp;
}
```
**readelf -s swap.o**
![[Pasted image 20261003163327.png]]
**readelf -S swap.o**
![[Pasted image 20261003163359.png]]

| Nr  | 节名                  | 类型       | Size（字节）  | 文件偏移  | Flags | 注释                                                   |
| :-: | :------------------ | :------- | :-------- | :---- | :---- | :--------------------------------------------------- |
|  0  | （空）                 | NULL     | 0         | 0x00  |       | 固定的空条目，占位用                                           |
|  1  | `.text`             | PROGBITS | 0x43（67）  | 0x40  | AX    | `swap` 的机器码；A=加载进内存，X=可执行                            |
|  2  | `.rela.text`        | RELA     | 0x90（144） | 0x1d8 | I     | `.text` 的重定位条目（书里叫 `.rel.text`）；Info=1 表示作用于 `.text` |
|  3  | `.data`             | PROGBITS | 0         | 0x83  | WA    | 普通已初始化数据，这里是空的；W=可写                                  |
|  4  | `.bss`              | NOBITS   | 8         | 0x88  | WA    | 放 `bufp1`（未初始化），NOBITS 表示**文件里不占空间**                 |
|  5  | `.data.rel`         | PROGBITS | 8         | 0x88  | WA    | 放 `bufp0`（初值是地址 `&buf[0]`，需要重定位）                     |
|  6  | `.rela.data.rel`    | RELA     | 0x18（24）  | 0x268 | I     | `.data.rel` 的重定位条目，对应书里的 `.rel.data`；Info=5          |
|  7  | `.comment`          | PROGBITS | 0x27      | 0x90  | MS    | 编译器版本信息，不用管                                          |
|  8  | `.note.GNU-stack`   | PROGBITS | 0         | 0xb7  |       | 标记栈是否可执行，不用管                                         |
|  9  | `.note.gnu.pr[...]` | NOTE     | 0x20      | 0xb8  | A     | GNU 属性说明，不用管                                         |
| 10  | `.eh_frame`         | PROGBITS | 0x38      | 0xd8  | A     | 异常/栈回溯信息，不用管                                         |
| 11  | `.rela.eh_frame`    | RELA     | 0x18      | 0x280 | I     | `.eh_frame` 的重定位条目                                   |
