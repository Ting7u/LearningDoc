# C 语言变量与类型命名规范整理

> 说明：C 语言标准（C89/C99/C11/C17 等）本身**不规定**命名风格，只规定了合法标识符的语法规则（字母/下划线开头，由字母、数字、下划线组成，区分大小写，不能是关键字）。以下命名规范来自业界常见的编码风格指南（如 Linux 内核编码规范、GNU 编码规范、Google C++ 风格指南中适用于 C 的部分、MISRA C 等），不同团队/项目会有所差异，使用前请以你所在项目的实际规范为准。

---

## 1. 标识符合法性规则（C 标准硬性要求）

- 由字母（A-Z, a-z）、数字（0-9）、下划线（`_`）组成
- 不能以数字开头
- 区分大小写（`Count` 和 `count` 是不同标识符）
- 不能与 C 关键字重名（如 `int`、`return`、`for` 等）
- 以双下划线开头（`__foo`）或单下划线加大写字母开头（`_Foo`）的标识符是**保留给编译器/标准库使用的**，用户代码应避免使用
- 以单下划线开头的全局标识符（如 `_foo`）在很多规范中也建议避免

---

## 2. 变量命名

| 规范流派 | 风格 | 示例 |
|---|---|---|
| Linux 内核风格 | 小写 + 下划线（snake_case），简短 | `int max_len;` |
| GNU 风格 | 小写 + 下划线 | `int buffer_size;` |
| Windows/MFC 匈牙利命名法（较老，现较少推荐） | 类型前缀 + 驼峰 | `int nCount;` `char* szName;` |

常见共识：

- **局部变量**：推荐简短、小写、下划线分隔，如 `i`, `j`, `idx`, `buf_len`
- **全局变量**：建议加前缀以示区分（如模块名前缀），并倾向使用更具描述性的名字，如 `g_error_count` 或 `module_error_count`
- **布尔类型变量**：常用 `is_`、`has_`、`can_` 前缀，如 `is_valid`、`has_data`（C99 之后有 `<stdbool.h>` 的 `bool` 类型）
- 避免使用单字母变量名（循环计数器 `i/j/k` 除外）
- 避免使用容易和数字 `1`、字母 `l`/`O` 混淆的命名

---

## 3. 常量命名

- **宏定义常量**：全大写 + 下划线分隔，是最广泛认可的约定
  ```c
  #define MAX_BUFFER_SIZE 1024
  #define PI 3.14159
  ```
- **`const` 修饰的常量**：不同规范不一致，有的沿用全大写，有的按普通变量小写命名（更贴近类型系统的做法）
  ```c
  const int MaxRetryCount = 5;   // 部分规范
  const int max_retry_count = 5; // 另一些规范
  ```
- **枚举常量**：通常全大写，常加统一前缀标明所属枚举类型
  ```c
  enum Color { COLOR_RED, COLOR_GREEN, COLOR_BLUE };
  ```

---

## 4. 函数命名

- 通用做法：小写 + 下划线，动词开头，体现函数行为
  ```c
  int calculate_sum(int a, int b);
  void free_buffer(Buffer *buf);
  ```
- 库/模块函数常加模块名前缀，防止命名冲突（C 没有命名空间）
  ```c
  http_client_connect();
  json_parse_string();
  ```
- 返回布尔值的函数常用 `is_`、`has_` 开头
  ```c
  int is_empty(const List *list);
  ```
- 静态（文件内部私有）函数与全局导出函数有时会用不同前缀或后缀区分，视项目约定而定

---

## 5. 类型命名（`struct` / `enum` / `typedef`）

- **结构体/枚举标签名（tag）**：常用首字母大写的驼峰式（PascalCase），或与函数一致的小写下划线风格，两种都常见
  ```c
  struct Point { int x, y; };
  struct point { int x, y; };   // 另一流派
  ```
- **`typedef` 定义的新类型名**：很多规范建议加 `_t` 后缀（POSIX 系统类型多用此约定，如 `size_t`、`pid_t`）
  ```c
  typedef struct Point Point_t;
  typedef unsigned long ulong_t;
  ```
- 注意：Linux 内核编码规范其实**不鼓励**滥用 `typedef`（认为会隐藏类型信息），这是一个有争议、非统一的点，不同项目态度不同

---

## 6. 宏命名

- 普通宏常量、宏函数：全大写 + 下划线
  ```c
  #define MAX(a, b) ((a) > (b) ? (a) : (b))
  ```
- 用于条件编译的宏（头文件保护宏）：常用 `文件名_大写_H` 形式
  ```c
  #ifndef MY_HEADER_H
  #define MY_HEADER_H
  ...
  #endif
  ```

---

## 7. 文件命名

- 源文件与头文件通常使用小写 + 下划线，例如 `string_utils.c`、`string_utils.h`
- 头文件名与其中主要声明的模块名保持一致

---

## 8. 其他通用建议

- 命名要**见名知意**，避免无意义的缩写（如避免 `tmp1`, `data2` 这种）
- 项目内部风格要**保持一致**：一旦选定一种流派（如 snake_case），不要在同一项目里混用驼峰和下划线
- 避免使用容易引发歧义的缩写（除非是团队内广泛认可的，如 `idx` 表示 index、`buf` 表示 buffer、`len` 表示 length）
- 避免使用系统保留前缀（`_`、`__`、`str`、`mem`、`is` 等标准库常用前缀有时也建议规避，以防未来标准库扩展冲突）

---

## 9. 常见业界规范参考（供进一步查阅）

- Linux 内核编码规范（Linux kernel coding style）
- GNU 编码标准（GNU Coding Standards）
- MISRA C（面向汽车/嵌入式安全关键系统的编码规范，对命名有更严格要求）
- Google C++ 风格指南（部分命名理念可参考，但本身是 C++ 导向，不完全适用于纯 C）

以上内容为不同规范流派的整理归纳，并非唯一"标准答案"，实际项目请以团队/项目文档为准。
