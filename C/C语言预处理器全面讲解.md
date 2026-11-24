# C 语言预处理（Preprocessor）全面讲解

> 说明：本文覆盖的是 C 标准（C89/C90、C99、C11、C17、C23）中关于预处理器的规定，以及主流编译器（GCC / Clang / MSVC）的常见扩展行为。凡是"标准规定"的内容，我会尽量说明；凡是"编译器扩展/依赖具体实现"的内容，我会明确标注，避免混淆。

---

## 目录

1. 预处理器是什么、它在编译流程中的位置
2. `#include`：头文件包含
3. `#define`：宏定义（对象宏 / 函数宏）
4. `#undef`：取消宏定义
5. 条件编译：`#if / #ifdef / #ifndef / #elif / #else / #endif`
6. `defined` 运算符与 `#elifdef / #elifndef`
7. 预定义宏（`__FILE__`、`__LINE__`、`__DATE__` 等）
8. `#` 字符串化运算符
9. `##` 记号粘贴运算符
10. 可变参数宏（`__VA_ARGS__` / `__VA_OPT__`）
11. `#line` 指令
12. `#error` 与 `#warning`
13. `#pragma` 指令
14. 宏编写的常见陷阱与最佳实践
15. 如何查看预处理展开后的结果

---

## 1. 预处理器是什么

C 的编译过程（以 C 标准中"翻译阶段 translation phases"的说法）大致可以分成几个阶段，预处理器工作在**编译之前**，处理的是纯文本层面的替换，它本身**不理解 C 语法**，只做：

- 文件包含（把 `#include` 的文件内容"粘贴"进来）
- 宏替换（文本替换）
- 条件编译（决定哪些代码留下、哪些被删掉）
- 一些杂项指令（`#pragma`、`#line`、`#error` 等）

大致流程：

```
源代码(.c)
   │
   ▼
预处理器 (cpp)  ──►  展开后的.i文件（纯C代码，无宏、无#include）
   │
   ▼
编译器（词法/语法分析、生成汇编）
   │
   ▼
汇编器 → 目标文件(.o)
   │
   ▼
链接器 → 可执行文件
```

所有以 `#` 开头的行（前面允许有空白）都是预处理指令，例如 `#include`、`#define`、`#if` 等。

---

## 2. `#include`：头文件包含

### 2.1 基本用法

```c
#include <stdio.h>   // 尖括号：优先在"系统/标准库"路径中查找
#include "myheader.h" // 双引号：优先在"当前源文件所在目录"中查找，找不到再去系统路径
```

标准规定的区别（C11 6.10.2）：
- `<...>` 形式：查找方式由实现定义，通常用于标准库或第三方库头文件。
- `"..."` 形式：先在**包含该 `#include` 指令的源文件所在目录**中查找，找不到时按 `<...>` 的方式继续查找。

### 2.2 `#include` 的本质：文本粘贴

预处理器会把被包含文件的**全部内容**原样"贴"到 `#include` 所在的位置。举例：

`point.h`：
```c
struct Point {
    int x;
    int y;
};
```

`main.c`：
```c
#include "point.h"

int main(void) {
    struct Point p = {1, 2};
    return p.x + p.y;
}
```

预处理后，`main.c` 实际参与编译的内容等价于：
```c
struct Point {
    int x;
    int y;
};

int main(void) {
    struct Point p = {1, 2};
    return p.x + p.y;
}
```

### 2.3 头文件重复包含问题与"头文件卫士"（include guard）

如果一个头文件被间接包含多次（比如 `a.h` 和 `b.h` 都 `#include "common.h"`，而 `main.c` 又同时包含 `a.h` 和 `b.h`），`common.h` 的内容会被粘贴两次，导致结构体/函数重复定义，编译报错。

标准写法（这是**惯用法**，不是语言强制要求，但几乎所有 C 项目都这样写）：

```c
// common.h
#ifndef COMMON_H
#define COMMON_H

struct Config {
    int level;
};

void init_config(struct Config *c);

#endif // COMMON_H
```

原理：第一次包含时 `COMMON_H` 未定义，于是 `#define COMMON_H` 被执行，内容被保留；第二次再包含同一个文件时，`#ifndef COMMON_H` 判断为假，中间内容整体被跳过，从而避免重复定义。

### 2.4 `#pragma once`

这是一种**非标准但几乎所有主流编译器都支持**的替代写法，效果与头文件卫士类似，但由编译器直接根据"文件"来判断是否已包含过：

```c
#pragma once

struct Config {
    int level;
};
```

需要说明：`#pragma once` 不是 ISO C 标准的一部分，属于编译器扩展（GCC、Clang、MSVC 均支持），在极少数涉及符号链接/网络文件系统的特殊场景下可能出现判断失误，而 `#ifndef` 卫士是标准保证有效的写法。

---

## 3. `#define`：宏定义

### 3.1 对象宏（object-like macro）：常量替换

```c
#define PI 3.14159
#define MAX_BUFFER_SIZE 1024
#define PROJECT_NAME "MyApp"

int buf[MAX_BUFFER_SIZE];
double area = PI * r * r;
```

预处理阶段，`MAX_BUFFER_SIZE`、`PI` 等标识符会被**原样替换**为后面的文本，这一步发生在编译器真正做类型检查之前。所以宏没有类型的概念，也不占用变量地址。

### 3.2 函数宏（function-like macro）：带参数的文本替换

```c
#define SQUARE(x) ((x) * (x))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int y = SQUARE(5);      // 展开为 ((5) * (5))
int m = MAX(3, 7);      // 展开为 ((3) > (7) ? (3) : (7))
```

注意：**函数宏的左括号必须紧跟在宏名后面**，中间不能有空格，否则会被当成对象宏：

```c
#define FOO (x) ((x)+1)   // 错误理解：这被当成对象宏 FOO，替换文本是 "(x) ((x)+1)"
```

### 3.3 为什么函数宏几乎总要加括号

先看一个反例：

```c
#define SQUARE(x) x * x

int y = SQUARE(1 + 2);   // 期望 9，实际展开为 1 + 2 * 1 + 2 = 5（乘法优先级更高）
```

正确写法必须把参数、以及整个替换结果都加上括号：

```c
#define SQUARE(x) ((x) * (x))

int y = SQUARE(1 + 2);   // 展开为 ((1 + 2) * (1 + 2)) = 9，正确
```

再看一个"整体不加括号"导致的问题：

```c
#define DOUBLE(x) (x) + (x)

int a = 3 * DOUBLE(2);   // 展开为 3 * (2) + (2) = 6 + 2 = 8，而不是期望的 3*(2+2)=12
```

改正：

```c
#define DOUBLE(x) ((x) + (x))

int a = 3 * DOUBLE(2);   // 展开为 3 * ((2) + (2)) = 12，正确
```

**结论（记住这条规则即可）**：函数宏中，每个参数使用处都加括号，宏整体替换结果也加括号。

### 3.4 宏参数副作用（side effect）陷阱

宏是纯文本替换，参数如果在展开后的文本里出现多次，副作用（比如 `++`、函数调用）会被执行多次：

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int i = 5, j = 10;
int m = MAX(i++, j++);
// 展开为: ((i++) > (j++) ? (i++) : (j++))
// i++ 被求值了两次！最终 i、j 的值和你直觉预期的不一样
```

这是宏相对于函数的**本质缺陷**——函数调用时参数只求值一次，宏展开可能让参数被求值 0 次、1 次或多次，具体取决于宏体如何使用该参数。这也是很多代码规范建议"能用内联函数就不用宏"的原因之一（见第 14 节）。

### 3.5 宏可以没有参数值（对象宏当"开关"用）

```c
#define DEBUG_MODE

#ifdef DEBUG_MODE
    printf("debug info\n");
#endif
```

这里 `DEBUG_MODE` 没有被替换成任何具体值，它只是"存在与否"这个信息本身被用来做条件编译判断（详见第 5 节）。

---

## 4. `#undef`：取消宏定义

```c
#define VERSION 1

#undef VERSION
#define VERSION 2   // 重新定义为2，没有警告

printf("%d\n", VERSION); // 输出 2
```

一个宏如果没有 `#undef` 就直接用不同内容重新 `#define`，是**未定义行为 / 编译器警告或报错**的来源（除非两次定义在标准意义上"完全相同"）。所以如果需要重定义某个宏，规范做法是先 `#undef` 再 `#define`。

典型用途：临时覆盖第三方库里的宏，用完之后恢复：

```c
#include <limits.h>

#ifdef INT_MAX
#undef INT_MAX
#endif
#define INT_MAX 100   // 为了测试临时改小
```

---

## 5. 条件编译

### 5.1 `#ifdef` / `#ifndef`

判断某个宏"是否被定义过"（不关心它的值是什么）：

```c
#define FEATURE_LOG

#ifdef FEATURE_LOG
    printf("日志功能已开启\n");
#endif

#ifndef FEATURE_NETWORK
    printf("网络功能未开启\n");
#endif
```

### 5.2 `#if` / `#elif` / `#else` / `#endif`：基于常量表达式的判断

`#if` 后面跟的是一个**整数常量表达式**，在预处理阶段求值：

```c
#define VERSION 3

#if VERSION == 1
    printf("v1\n");
#elif VERSION == 2
    printf("v2\n");
#else
    printf("v3 或更高\n");
#endif
```

常用于跨平台代码：

```c
#if defined(_WIN32)
    #include <windows.h>
#elif defined(__linux__)
    #include <unistd.h>
#elif defined(__APPLE__)
    #include <TargetConditionals.h>
#else
    #error "未知平台"
#endif
```

### 5.3 数值比较、逻辑运算都可以用在 `#if` 里

```c
#define A 5
#define B 10

#if (A + B) > 12 && A < B
    printf("条件成立\n");
#endif
```

注意：`#if` 表达式中，未定义的普通标识符会被当作 `0` 处理（这是标准规定的行为），所以下面这种写法是"合法但容易踩坑"的：

```c
#if UNDEFINED_MACRO > 0
    // UNDEFINED_MACRO 被当作 0，0 > 0 为假，这个分支不会进入
#endif
```

---

## 6. `defined` 运算符 与 `#elifdef` / `#elifndef`（C23）

### 6.1 `defined`

`defined(X)` 或 `defined X` 是在 `#if`/`#elif` 表达式内部使用的运算符，判断宏 `X` 是否被定义，返回 1 或 0：

```c
#if defined(DEBUG) && !defined(NDEBUG)
    printf("调试模式\n");
#endif
```

它等价于把 `#ifdef DEBUG` 这种"独立判断"变成可以和其他条件用 `&&`、`||` 组合的"表达式"，这是它相对于单独的 `#ifdef` 的优势。

### 6.2 `#elifdef` / `#elifndef`（C23 新增）

C23 标准新增了 `#elifdef` 和 `#elifndef`，用来简化"多个 `#ifdef` 链式判断"的写法：

```c
#if defined(PLATFORM_WIN)
    printf("Windows\n");
#elifdef PLATFORM_LINUX      // 等价于 #elif defined(PLATFORM_LINUX)
    printf("Linux\n");
#elifndef PLATFORM_MAC       // 等价于 #elif !defined(PLATFORM_MAC)
    printf("既不是 Linux 也没定义 mac 宏\n");
#endif
```

这是较新的 C23 特性，目前（2026年）主流编译器（较新版本的 GCC/Clang）已支持，但如果你的项目要兼容较老编译器，建议继续用 `#elif defined(...)` 的写法。

---

## 7. 预定义宏

C 标准规定了一批编译器必须提供的预定义宏，此外编译器厂商还会各自扩展一批。

### 7.1 标准预定义宏（ISO C 规定）

| 宏 | 含义 | 示例 |
|---|---|---|
| `__FILE__` | 当前源文件名（字符串） | `"main.c"` |
| `__LINE__` | 当前行号（整数） | `42` |
| `__DATE__` | 编译发生的日期（字符串） | `"Sep 25 2026"` |
| `__TIME__` | 编译发生的时间（字符串） | `"14:03:21"` |
| `__STDC__` | 是否符合 ISO C 标准，通常为 `1` | `1` |
| `__STDC_VERSION__` | 使用的 C 标准版本号 | `199901L`（C99）、`201112L`（C11）、`201710L`（C17）等 |
| `__func__` | 当前函数名（严格来说这是 C99 引入的隐式声明的字符串常量，不是宏，但用法很像） | `"main"` |

示例代码：

```c
#include <stdio.h>

void log_message(const char *msg) {
    printf("[%s:%d] %s: %s\n", __FILE__, __LINE__, __func__, msg);
}

int main(void) {
    log_message("程序启动");
    printf("当前标准版本: %ld\n", __STDC_VERSION__);
    return 0;
}
```

可能输出：
```
[main.c:5] log_message: 程序启动
当前标准版本: 201710
```

（实际输出的版本号取决于你编译时用的 `-std=` 选项和编译器默认值，这一点我无法替你确定，建议你实际编译验证。）

### 7.2 编译器扩展的预定义宏（非标准，依赖具体编译器）

这些不是 ISO C 标准规定的，而是各编译器自己加的，写跨平台代码时经常用到：

```c
#if defined(__GNUC__)
    printf("使用 GCC，版本 %d.%d\n", __GNUC__, __GNUC_MINOR__);
#elif defined(_MSC_VER)
    printf("使用 MSVC，版本号 %d\n", _MSC_VER);
#elif defined(__clang__)
    printf("使用 Clang\n");
#endif
```

这类宏的具体名称和取值在不同编译器/不同版本间可能有差异，如果要在生产代码里依赖它们做精确判断，建议查阅对应编译器的官方文档确认，我这里不确保覆盖了所有情况。

---

## 8. `#` 字符串化运算符

在函数宏定义中，`#参数名` 会把传入的实际参数**转换成字符串字面量**（在替换之前，参数本身不会先展开成别的宏，除非用了下面 9.3 提到的间接层技巧）。

```c
#define STR(x) #x

printf("%s\n", STR(hello));      // 输出: hello
printf("%s\n", STR(1 + 2));      // 输出: 1 + 2  （注意：是文本"1 + 2"，不是计算结果3）
```

常见实用场景——把宏值和它的文本名一起打印，方便调试：

```c
#define PRINT_VAR(v) printf(#v " = %d\n", v)

int count = 42;
PRINT_VAR(count);   // 展开为 printf("count" " = %d\n", count);
                     // 相邻字符串字面量会被编译器自动拼接，等价于 printf("count = %d\n", count);
                     // 输出: count = 42
```

---

## 9. `##` 记号粘贴运算符

`##` 用于把两个"记号（token）"拼接成一个新的记号，常用于批量生成变量名/函数名。

### 9.1 基本用法

```c
#define CONCAT(a, b) a##b

int CONCAT(my, Var) = 10;   // 展开为 int myVar = 10;
```

### 9.2 结合循环式生成代码（一种常见套路：生成一组同类型函数）

```c
#define DECLARE_GETTER_SETTER(type, name)      \
    static type name##_value;                  \
    void set_##name(type v) { name##_value = v; } \
    type get_##name(void) { return name##_value; }

DECLARE_GETTER_SETTER(int, age)
DECLARE_GETTER_SETTER(double, score)

int main(void) {
    set_age(20);
    set_score(88.5);
    printf("%d %f\n", get_age(), get_score());
    return 0;
}
```

上面这段宏展开之后大致等价于手写了：

```c
static int age_value;
void set_age(int v) { age_value = v; }
int get_age(void) { return age_value; }

static double score_value;
void set_score(double v) { score_value = v; }
double get_score(void) { return score_value; }
```

### 9.3 一个常见"坑"：宏参数在 `#` / `##` 旁边不会先展开

```c
#define VALUE 100
#define STR(x) #x
#define STR_EXPAND(x) STR(x)   // 多包一层，让参数先展开再字符串化

printf("%s\n", STR(VALUE));         // 输出: "VALUE"（没有先展开宏！）
printf("%s\n", STR_EXPAND(VALUE));  // 输出: "100"（VALUE先被替换成100，再字符串化）
```

原理：`#` 和 `##` 运算符会让紧邻它们的参数"原样"参与替换，不会先对参数做宏展开；而普通的宏参数传递，在替换前是会先展开的。所以如果你希望"参数先展开，再字符串化/拼接"，通常需要像 `STR_EXPAND` 这样**多包一层间接宏**。这是编写通用宏工具库时非常经典的技巧。

---

## 10. 可变参数宏

### 10.1 `__VA_ARGS__`（C99 标准引入）

```c
#include <stdio.h>

#define LOG(fmt, ...) printf("[LOG] " fmt "\n", __VA_ARGS__)

int main(void) {
    LOG("value = %d", 42);
    // 展开为: printf("[LOG] " "value = %d" "\n", 42);
    return 0;
}
```

### 10.2 没有额外参数时的问题

上面这个宏如果调用时不传任何可变参数，会出问题：

```c
LOG("hello");
// 展开为: printf("[LOG] " "hello" "\n", );   // 多了一个逗号，编译错误（标准C99/C11下）
```

### 10.3 GNU 扩展：`##__VA_ARGS__`（非标准，但被 GCC/Clang 广泛支持）

```c
#define LOG(fmt, ...) printf("[LOG] " fmt "\n", ##__VA_ARGS__)

LOG("hello");           // 正常展开，没有多余逗号（GCC扩展会在可变参数为空时把前面的逗号也去掉）
LOG("value=%d", 42);    // 正常展开
```

这是 GNU C 的扩展语法，不是 ISO C 标准的一部分。如果你要求代码严格标准可移植，请使用下面 C23 引入的 `__VA_OPT__`。

### 10.4 `__VA_OPT__`（C23 标准新增，此前是 C++20 特性）

```c
#define LOG(fmt, ...) printf(fmt __VA_OPT__(,) __VA_ARGS__)

LOG("hello\n");            // 没有可变参数时，__VA_OPT__(,) 不产生逗号
LOG("value=%d\n", 42);     // 有可变参数时，__VA_OPT__(,) 产生一个逗号
```

`__VA_OPT__(content)` 的含义是：只有当可变参数部分（`__VA_ARGS__`）**非空**时，才把 `content` 保留在展开结果里，否则整体删除。这是标准做法，比 GNU 的 `##__VA_ARGS__` 扩展更"正规"，但要求编译器支持 C23（或早期作为 C2x 草案特性被部分编译器提前实现）。

---

## 11. `#line` 指令

`#line` 用来**手动修改**预处理器内部记录的"当前行号"和"当前文件名"，主要用于代码生成工具（比如把其他语言/模板生成成 C 代码时，让报错信息指向原始模板文件而不是生成的 C 文件）。

```c
#line 100 "generated_from_template.tpl"

int main(void) {
    int x;  // 假设这里有个错误
    return 0;
}
```

如果这一行往下第 2 行触发编译警告/错误，编译器提示的"文件名:行号"会显示为类似 `generated_from_template.tpl:101`，而不是实际的 `.c` 文件名和真实行号。这是一个相对冷门、主要给代码生成工具使用的指令，日常业务代码基本用不到。

---

## 12. `#error` 与 `#warning`

### 12.1 `#error`（ISO C 标准，从 C89 就有）

用于在预处理阶段主动让编译失败，并输出一条自定义信息：

```c
#if !defined(__STDC_VERSION__) || __STDC_VERSION__ < 199901L
    #error "本项目要求 C99 或更高标准，请添加编译选项如 -std=c99"
#endif
```

### 12.2 `#warning`

`#warning` 的效果和 `#error` 类似，但**只输出警告信息，不会让编译失败**。需要说明的是：`#warning` 在 C23 之前属于编译器扩展（GCC、Clang、MSVC 等长期都支持这个扩展，但不是标准的一部分），**从 C23 标准开始它被正式纳入 ISO C 标准**。

```c
#ifdef USE_DEPRECATED_API
    #warning "USE_DEPRECATED_API 已过时，将在下个版本移除"
#endif
```

如果你的项目需要兼容非常严格/较老的标准模式（比如某些 `-std=c99 -pedantic-errors` 的极端设置），要留意 `#warning` 在旧标准下可能被当作扩展而报"非标准用法"的提示。

---

## 13. `#pragma` 指令

`#pragma` 是"给编译器的特殊指令"，具体支持哪些 `#pragma`、含义是什么，**完全由各编译器自行定义**，标准只规定了语法形式和"编译器不认识的 `#pragma` 应该被忽略"这一行为，不规定具体语义。

一些常见例子（均为编译器扩展，不属于语言核心标准语义）：

```c
#pragma once              // 前面提到过：防止头文件被重复包含（GCC/Clang/MSVC支持）

#pragma pack(push, 1)     // 通知编译器结构体按1字节对齐（常用于网络协议/文件格式解析）
struct PacketHeader {
    unsigned char type;
    unsigned int  length;
};
#pragma pack(pop)         // 恢复默认对齐方式

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wunused-variable"
int unused_var;           // 这行不会触发"未使用变量"警告
#pragma GCC diagnostic pop
```

### 13.1 `_Pragma` 运算符（C99 标准，宏内可用）

`#pragma` 是一整行指令，不能直接写进宏定义里；如果想在宏里生成 `#pragma` 效果，标准提供了 `_Pragma` 运算符：

```c
#define DISABLE_UNUSED_WARNING _Pragma("GCC diagnostic ignored \"-Wunused-variable\"")

DISABLE_UNUSED_WARNING
int temp;
```

---

## 14. 宏编写的常见陷阱与最佳实践

### 14.1 多语句宏要用 `do { ... } while (0)` 包裹

反例：

```c
#define SWAP(a, b) { int t = a; a = b; b = t; }

if (x > y)
    SWAP(x, y);
else
    printf("no swap\n");
```

展开后：
```c
if (x > y)
    { int t = x; x = y; y = t; };   // 注意这里多了一个分号！
else                                  // 于是这个 else 找不到匹配的 if，编译报错
    printf("no swap\n");
```

正确写法：

```c
#define SWAP(a, b) do { int t = (a); (a) = (b); (b) = t; } while (0)

if (x > y)
    SWAP(x, y);
else
    printf("no swap\n");
```

`do { ... } while (0)` 让整个宏体在语法上表现得像"单条语句"，可以安全地配合 `if/else`、后面加分号使用，是 C 语言里编写"多语句宏"的标准套路。

### 14.2 尽量用 `const` / `enum` / `static inline` 替代简单宏

```c
// 不推荐（没有类型检查，调试时看不到符号，编译器报错信息里全是数字/展开文本）
#define MAX_RETRY 3

// 更推荐（有类型、有作用域、调试器里能看到符号名）
static const int MAX_RETRY = 3;
```

对于类似 `MAX(a, b)` 这种简单函数宏，C99 之后可以用 `static inline` 函数替代，从而避免"参数多次求值"的副作用问题：

```c
static inline int max_int(int a, int b) {
    return a > b ? a : b;
}

int m = max_int(i++, j++);  // i、j 各自只求值一次，行为符合直觉
```

代价是：`inline` 函数需要指定具体类型（不像宏那样对任意类型都"泛型"适用），如果需要多种类型通用，C11 之后可以配合 `_Generic` 实现类型分派，这属于另一个话题，这里不展开。

### 14.3 X-Macro 技巧：用一份数据驱动多处代码生成

这是一个比较进阶、但在枚举/字符串表/协议字段等场景非常实用的模式：

```c
// 定义一份“数据表”宏，只在这里维护一次
#define COLOR_LIST \
    X(RED)          \
    X(GREEN)        \
    X(BLUE)

// 用同一份数据表分别生成枚举
typedef enum {
#define X(name) COLOR_##name,
    COLOR_LIST
#undef X
} Color;

// 再用同一份数据表生成对应的字符串数组
static const char *color_names[] = {
#define X(name) #name,
    COLOR_LIST
#undef X
};

// 使用
int main(void) {
    Color c = COLOR_GREEN;
    printf("%s\n", color_names[c]);  // 输出: GREEN
    return 0;
}
```

好处：以后新增一种颜色，只需要在 `COLOR_LIST` 里加一行 `X(YELLOW)`，枚举定义和字符串表会自动同步更新，避免两处手动维护导致不一致。

### 14.4 宏命名习惯

- 宏名通常全大写、下划线分隔（如 `MAX_BUFFER_SIZE`），这样在阅读代码时能一眼看出"这是宏，不是变量/函数"，从而对它"文本替换、无类型检查、参数可能被多次求值"这些特性保持警惕。这是一种被广泛采用的约定，不是语言强制要求。
- 避免用宏覆盖常见的语言关键字或者太短、太容易和普通标识符混淆的名字（比如不要 `#define MAX 100` 之后又在别处把 `MAX` 当普通变量名用）。

---

## 15. 如何查看预处理展开后的结果

在调试宏、排查"宏到底展开成了什么"时，最直接的办法是让编译器只做预处理、不继续往下编译，输出中间结果，这个技巧对理解本文前面所有内容都很有帮助。

以 GCC / Clang 为例：

```bash
# 只做预处理，把结果输出到标准输出
gcc -E main.c

# 只做预处理，并去掉多余的行号标记信息（可读性更好）
gcc -E -P main.c

# 输出到文件，方便查看
gcc -E main.c -o main.i
```

例如对下面这段代码：

```c
#define SQUARE(x) ((x) * (x))
int y = SQUARE(1 + 2);
```

执行 `gcc -E -P a.c` 大致会得到：

```c
int y = ((1 + 2) * (1 + 2));
```

这是排查宏相关 bug（尤其是运算符优先级、多次求值、`##`/`#` 拼接结果）最可靠的手段，比"靠脑子在纸上展开"要准确得多，建议遇到复杂宏时优先用这个方法验证。

---

## 附：本文覆盖范围说明

本文涵盖的是 ISO C 标准（C89 ~ C23）中预处理器相关的核心内容，以及 GCC/Clang/MSVC 常见的扩展用法。个别标注为"扩展/非标准"的内容（如 `#pragma once`、`##__VA_ARGS__` 的 GNU 用法），在不同编译器、不同版本之间可能存在细微差异，如果你的项目对可移植性要求很高，建议以你实际使用的编译器官方文档为准做二次确认。
