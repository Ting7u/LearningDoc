# C 语言最新特性速览（C23）

> 最新正式标准：**C23**（ISO/IEC 9899:2024）
> 编译选项：`-std=c23`（旧版本编译器用 `-std=c2x`）
> 检查版本：`gcc --version`，版本较老时部分特性可能不支持

---

## 1. 基础语法更现代

### 1.1 `bool` / `true` / `false` 成为关键字

不再需要 `#include <stdbool.h>`。

```c
bool ok = true;
if (!ok) { /* ... */ }
```

### 1.2 `nullptr`：专用空指针

比 `NULL`（本质是 `0` 或 `(void*)0`）类型更明确、更安全。

```c
int *p = nullptr;
if (p == nullptr) { puts("空指针"); }
```

### 1.3 `auto`：类型推导

根据初始值自动推断类型（必须有初始化）。

```c
auto x = 3.14;     // double
auto n = 10;       // int
```

### 1.4 `typeof`：获取表达式类型

以前是 GCC 扩展，现在进入标准。

```c
int a = 5;
typeof(a) b = 10;  // b 也是 int
```

### 1.5 `constexpr`：编译期常量

C 里的 `const` 并不是真正的常量，`constexpr` 才是。

```c
constexpr int N = 10;
int arr[N];        // 合法，N 是编译期常量
```

### 1.6 空初始化 `{}`

```c
int arr[10] = {};          // 全部清零，不必再写 {0}
struct Point p = {};
```

---

## 2. 数字与整数

### 2.1 二进制字面量 + 数字分隔符

```c
int flags = 0b1010'0101;       // 二进制
long big  = 1'000'000'000;     // 分隔符只为方便阅读
```

### 2.2 `_BitInt(N)`：指定位宽的整数

适合硬件寄存器、嵌入式、密码学。

```c
_BitInt(12) reg = 100;         // 12 位整数
unsigned _BitInt(128) huge;    // 128 位无符号整数
```

### 2.3 枚举指定底层类型

```c
enum Color : unsigned char { RED, GREEN, BLUE };
```

---

## 3. 属性 `[[...]]`

给代码附加提示，让编译器帮你检查。

```c
[[nodiscard]] int open_file(const char *path);   // 返回值不能忽略
[[maybe_unused]] int debug_flag;                 // 没用到也不警告
[[deprecated("请使用 new_api")]] void old_api(void);

switch (x) {
    case 1:
        do_a();
        [[fallthrough]];                         // 故意不写 break
    case 2:
        do_b();
        break;
}
```

调用 `open_file("a.txt");` 却不使用返回值时，编译器会给出警告。

---

## 4. 预处理器

### 4.1 `#embed`：直接嵌入文件

```c
const unsigned char logo[] = {
    #embed "logo.png"
};
```

不再依赖 `xxd -i` 之类的外部工具。（GCC 15+ 才支持，较老环境可能不可用）

### 4.2 `#elifdef` / `#elifndef`

```c
#ifdef WIN32
    // Windows
#elifdef __linux__
    // Linux
#else
    // 其他
#endif
```

### 4.3 其他

- `#warning` 正式标准化
- `__has_include("x.h")` 检查头文件是否存在

---

## 5. 标准库新增

### 5.1 `<stdckdint.h>`：溢出检查的算术

```c
#include <stdckdint.h>

int r;
if (ckd_add(&r, INT_MAX, 1)) {
    puts("加法溢出了");
}
```

### 5.2 `<stdbit.h>`：位操作

```c
#include <stdbit.h>

unsigned x = 0b10110000;
printf("1 的个数: %u\n", stdc_count_ones(x));
printf("前导 0 个数: %u\n", stdc_leading_zeros(x));
```

### 5.3 `memset_explicit`

清零操作**不会被编译器优化掉**，适合擦除密码、密钥。

```c
char password[64];
// ... 使用 ...
memset_explicit(password, 0, sizeof password);
```

### 5.4 其他

- `strdup`、`strndup`、`memccpy` 正式纳入标准
- `unreachable()`：告诉编译器此处不会执行到
- `printf` 支持 `%b` 输出二进制

---

## 6. 清理与修正

| 变化 | 说明 |
|---|---|
| `void f()` | 现在明确表示**没有参数**（以前是"参数未指定"） |
| 删除 K&R 老式函数定义 | `int f(a, b) int a, b; {...}` 不再合法 |
| 删除三字符组 trigraphs | 如 `??=` 不再有效 |
| `static_assert` 可省略消息 | `static_assert(sizeof(int) == 4);` |
| 函数定义允许参数不起名 | `void f(int) { }` |
| 标签后可直接跟声明 | 不再要求必须是语句 |

---

## 7. 综合小例子

```c
#include <stdio.h>
#include <stdckdint.h>

[[nodiscard]] bool safe_add(int a, int b, int *out) {
    return !ckd_add(out, a, b);       // 溢出返回 false
}

int main(void) {
    constexpr int LIMIT = 1'000;
    int arr[LIMIT] = {};              // 空初始化
    auto sum = 0;                     // 类型推导为 int
    int *p = nullptr;                 // 空指针

    if (safe_add(LIMIT, 20, &sum)) {
        printf("sum = %d\n", sum);
    }
    return p == nullptr ? 0 : 1;
}
```

编译运行：

```bash
gcc -std=c23 -Wall -Wextra -Wpedantic main.c -o app
./app
```

---

## 8. 下一代：C2y（草案阶段）

正在讨论的内容包括 `defer`、`if` 里直接声明变量、`_Countof`（获取数组元素个数）等，细节可能变化，暂不需要关注。

---

## 学习建议

- 日常学习用 `-std=c11` 或 `-std=c17` 已经足够
- 基础稳固后，优先尝试这几个最实用的：`bool` 关键字、`nullptr`、`auto`、`constexpr`、`[[nodiscard]]`
- 遇到报错 `unknown option` 或 `unrecognized keyword`，通常是编译器版本太旧，升级 GCC 或改用 `-std=c2x` 试试
