# 第 2 章 基本语法

> 目标：掌握变量、类型、初始化、运算符、控制流这些"每行代码都会用到"的东西，并建立对**未定义行为**的警惕。

---

## 2.1 变量与基本类型

### 内置类型

| 类型 | 说明 | 大小 |
|------|------|------|
| `bool` | 布尔 | 通常 1 字节 |
| `char` | 字符/最小可寻址单元 | `sizeof(char)` **一定是 1** |
| `short` / `int` / `long` / `long long` | 整数 | 标准只保证下限：`short`、`int` 至少 16 位，`long` 至少 32 位，`long long` 至少 64 位 |
| `float` / `double` / `long double` | 浮点 | 各平台差异较大，通常 `float` 4、`double` 8 字节 |
| `wchar_t`、`char8_t`（C++20）、`char16_t`、`char32_t` | 宽字符与 Unicode 字符 | 依实现 |

以上每种整数类型都有 `unsigned` 版本。**`int` 到底几位、`long` 几位是平台相关的**（例如 64 位 Linux 上 `long` 通常是 8 字节，64 位 Windows 上通常是 4 字节）。

### 查看本机类型信息

```cpp
#include <iostream>
#include <limits>

int main() {
    std::cout << "int:       " << sizeof(int) << " bytes\n";
    std::cout << "long:      " << sizeof(long) << " bytes\n";
    std::cout << "long long: " << sizeof(long long) << " bytes\n";
    std::cout << "double:    " << sizeof(double) << " bytes\n";

    std::cout << "int max:   " << std::numeric_limits<int>::max() << '\n';
    std::cout << "int min:   " << std::numeric_limits<int>::min() << '\n';
}
```

### 固定宽度整数（`<cstdint>`）

需要确定位数时，不要靠猜 `int`/`long`：

```cpp
#include <cstdint>

std::int8_t   a = -1;
std::uint8_t  b = 255;
std::int32_t  c = 100;
std::uint64_t d = 1;
```

小陷阱：`std::uint8_t` 在常见实现中就是 `unsigned char`，用 `cout` 输出时会被当成**字符**：

```cpp
std::uint8_t x = 65;
std::cout << x << '\n';                         // 输出字符 A，而不是 65
std::cout << static_cast<int>(x) << '\n';       // 65
```

### `std::size_t`

标准库里表示大小、下标的无符号类型（`vector::size()`、`sizeof` 的结果都是它）。

### 有符号/无符号的经典坑

```cpp
#include <iostream>

int main() {
    unsigned int u = 0;
    --u;
    std::cout << u << '\n';                     // 4294967295（32 位 unsigned int）：无符号溢出是"回绕"，有明确定义

    int a = -1;
    unsigned int b = 1;
    std::cout << (a < b) << '\n';               // 0（false）！a 先被转换成 unsigned，变成一个很大的数
}
```

两个 `int`/`unsigned` 混着比较，`-Wall` 会给出 `-Wsign-compare` 警告，要重视。下面的循环是另一个常见的死循环：

```cpp
// for (unsigned i = 10; i >= 0; --i) { ... }   // i >= 0 永远为真，死循环
```

---

## 2.2 字面量

```cpp
int dec  = 42;
int hex  = 0x2A;                  // 十六进制 = 42
int oct  = 052;                   // 以 0 开头是八进制 = 42（小心前导 0）
int bin  = 0b101010;              // 二进制 = 42（C++14）
long long big = 1'000'000'000LL;  // 数字分隔符 '（C++14），LL 表示 long long

unsigned u = 10u;
long     l = 10L;
float    f = 3.14f;               // 不加 f 的 3.14 是 double
double   d = 1.5e3;               // 1500.0

char c = 'A';
const char* s = "hello";          // 类型是 const char[6] 退化成指针
bool b = true;
```

### 字符串字面量

```cpp
#include <string>

using namespace std::string_literals;

auto s1 = "hi";                   // const char*
auto s2 = "hi"s;                  // std::string（C++14 的字面量后缀 s）
auto raw = R"(C:\temp\new.txt)";  // 原始字符串：不处理转义，写路径/正则很方便
```

### 一个 C 与 C++ 的差异

```cpp
sizeof('A');    // C++ 中是 1（'A' 是 char）；C 中通常是 4（'A' 是 int）
```

---

## 2.3 初始化

C++ 有多种初始化写法：

```cpp
int a = 5;      // 拷贝初始化
int b(5);       // 直接初始化
int c{5};       // 列表初始化（C++11，推荐统一使用）
int d = {5};    // 列表初始化（带等号）
int e{};        // 值初始化：得到 0
int f;          // 默认初始化：局部变量的值不确定！
```

### 默认值 vs 不确定值

```cpp
#include <iostream>

int g_global;                    // 全局/静态变量：自动零初始化 → 0

int main() {
    static int s_local;          // 同上 → 0
    int x;                       // 局部变量：值不确定，读取它是未定义行为
    int y{};                     // 0
    std::cout << g_global << s_local << y << '\n';   // 000
    // std::cout << x;           // 错误用法
}
```

### 列表初始化会拒绝"窄化转换"

窄化指可能丢失精度/数值的隐式转换（如 `double → int`、放不下的常量）。

```cpp
double pi = 3.14;
int x1 = pi;                              // 允许（通常有警告），x1 = 3
// int x2{pi};                            // 不合法：窄化
int x3{static_cast<int>(pi)};             // 显式转换，OK

char ok{65};                              // 常量且放得下：OK
// char bad{300};                         // 不合法：300 放不进 char
```

标准要求编译器对窄化给出诊断，不同编译器的处理力度略有差异（有的报错，有的是警告），以你实际编译的结果为准。优先使用 `{}` 的好处就是能提前发现这类问题。

### 容器的 `()` 与 `{}` 含义可能不同

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> a(3, 1);     // 3 个元素，每个值为 1 → {1, 1, 1}
    std::vector<int> b{3, 1};     // 初始化列表：两个元素 → {3, 1}
    std::cout << a.size() << ' ' << b.size() << '\n';   // 3 2
}
```

列表初始化时，如果类型有接受 `std::initializer_list` 的构造函数，会**优先**选它（第 12 章展开）。

### 最令人烦恼的解析（most vexing parse）

```cpp
std::string s1();      // 这不是创建对象，而是声明了一个返回 string、无参数的函数！
std::string s2;        // OK：默认构造
std::string s3{};      // OK
```

---

## 2.4 `auto`：自动类型推导

```cpp
auto i = 42;                  // int
auto d = 3.14;                // double
auto s = "hi";                // const char*
auto b = true;                // bool
```

### 推导规则（初学者需要记住的几条）

```cpp
const int ci = 1;
auto a = ci;                  // int：顶层 const 被丢弃

int x = 0;
int& rx = x;
auto b = rx;                  // int：引用被丢弃，b 是副本
auto& c = rx;                 // int&：要引用必须自己写 &
const auto& d = x;            // const int&
```

**`auto` 默认是按值拷贝，想要引用就写 `auto&` / `const auto&`。** 在范围 `for` 里这一点尤其重要（见 2.9）。

### 花括号的特殊情况

```cpp
auto il = {1, 2, 3};          // std::initializer_list<int>
auto one{1};                  // C++17 起推导为 int
```

### 适合与不适合用 `auto`

适合：类型名很长、类型显而易见。

```cpp
auto it = scores.begin();                     // 不用写长长的迭代器类型
auto p = std::make_unique<int>(5);            // 右边已经写明了类型
```

不适合：类型本身是读代码的关键信息时，写出来更清楚。

```cpp
auto n = compute();           // compute 返回什么？读者看不出来
```

`auto` 还可以用于函数返回类型推导（C++14）和 lambda 参数（C++14），后面章节再讲。

---

## 2.5 `decltype`

`decltype(表达式)` 得到表达式的类型，**不丢弃** `const` 和引用。

```cpp
int x = 0;
const int& r = x;

decltype(x) a = 1;            // int
decltype(r) b = x;            // const int&
decltype(x + 1.5) c = 0;      // double（表达式 x + 1.5 的类型）
decltype((x)) d = x;          // int&  —— 多套一层括号，x 变成"左值表达式"，结果是引用
```

最后一条是个著名的坑：`decltype(x)` 是变量声明时的类型，`decltype((x))` 是表达式类型（左值 → 引用）。

C++14 起还有 `decltype(auto)`：用 `decltype` 的规则做 `auto` 推导，在写泛型返回值时有用，之后会遇到。

---

## 2.6 `const`、`constexpr`、`static_assert` 速览

```cpp
const int a = 10;                 // 运行期或编译期常量，取决于初始值
constexpr int b = 10 * 2;         // 必须是编译期常量
constexpr double kPi = 3.14159265358979;

static_assert(sizeof(int) >= 4, "int 至少需要 4 字节");   // 编译期断言，失败直接编译错误
```

经验：**能写 `const` 就写 `const`**，真正需要编译期值（数组大小、模板参数）时用 `constexpr`。函数的 `constexpr` 见第 3 章。

---

## 2.7 运算符

### 算术运算符的细节

```cpp
#include <iostream>

int main() {
    std::cout << 7 / 2 << '\n';        // 3：整数除法，结果截断
    std::cout << -7 / 2 << '\n';       // -3：C++11 起规定向 0 截断
    std::cout << -7 % 2 << '\n';       // -1：余数的符号与被除数相同
    std::cout << 7 / 2.0 << '\n';      // 3.5：有浮点数参与，整体按浮点计算
}
```

整数除以 0（或对 0 取模）是**未定义行为**。浮点数除以 0 的结果取决于实现（在遵循 IEEE 754 的平台上通常得到 inf 或 NaN）。

**经典陷阱：求平均值**

```cpp
int sum = 7, n = 2;
double bad  = sum / n;                                    // 3.0：先整数除法，再转 double
double good = static_cast<double>(sum) / n;               // 3.5
```

### 自增自减

```cpp
int i = 5;
int a = i++;     // a = 5，i = 6：先用后加
int b = ++i;     // i = 7，b = 7：先加后用
```

独立语句里，**优先写 `++i`**。对 `int` 没区别，但对迭代器等类型（第 7、10 章）前置版本不需要产生临时副本。

### 逻辑运算符与短路

```cpp
Node* p = find();
if (p != nullptr && p->valid) {   // && 左边为 false 时，右边不会被求值，所以不会解引用空指针
    ...
}
```

`&&`、`||` 都是短路求值。

### 位运算与移位

```cpp
unsigned x = 0b1100;
x & 0b0100;         // 与
x | 0b0001;         // 或
x ^ 0b1111;         // 异或
~x;                 // 取反
x << 2;             // 左移
x >> 1;             // 右移
```

移位位数大于等于类型位宽是未定义行为。位运算尽量用无符号类型。

### 条件运算符与逗号运算符

```cpp
int m = (a > b) ? a : b;          // 三目运算符
```

逗号运算符 `(a, b)` 依次求值并返回最后一个，不建议炫技使用。

### 优先级陷阱

```cpp
if (x & 1 == 0) { ... }           // 意图：x 的最低位是 0
                                  // 实际：== 优先级高于 &，等价于 x & (1 == 0)，恒为 0
if ((x & 1) == 0) { ... }         // 正确：加括号
```

不确定优先级时直接加括号，不丢人。

### 类型提升与"常规算术转换"

小于 `int` 的类型参与运算时，先提升成 `int`；不同类型混合运算时，向"更大/更宽"的类型转换（有符号与无符号同宽时转成无符号，这正是 2.1 里 `-1 < 1u` 为假的原因）。

---

## 2.8 表达式的求值顺序与未定义行为（UB）

### 求值顺序

```cpp
int i = 0;
int r = i++ + ++i;                // 未定义行为：同一个变量在没有顺序保证的情况下被多次修改
```

```cpp
int f();
int g();
int sum = f() + g();              // f 和 g 谁先调用？标准未规定（不过两者不会交错执行）
```

C++ 中**保证**从左到右求值的运算符：`&&`、`||`、`?:`、`,`。

C++17 额外规定了一批顺序：`a << b` 与 `a >> b` 左边先于右边；赋值 `a = b`（以及 `a op= b`）右边先于左边；`a[b]`、`a.b`、`a->b` 前者先于后者；函数调用时，被调用的函数表达式先于参数求值。**但函数的各个实参之间求值顺序仍然未指定。**

写代码的原则：**不要在同一个表达式里对同一个变量既读又改，也不要依赖实参的求值顺序。**

### 三类"不确定"行为

| 类别 | 含义 | 例子 |
|------|------|------|
| **未定义行为（UB）** | 标准对结果**没有任何要求**，程序可以崩溃、输出错误结果，甚至"看起来正常" | 有符号整数溢出、空指针解引用、数组越界、读未初始化变量、数据竞争 |
| **未指定行为** | 有几种合法结果，编译器选哪种不必说明 | 函数实参求值顺序 |
| **实现定义行为** | 由编译器/平台决定，**文档里必须写明** | `sizeof(int)`、`char` 是否有符号 |

### UB 的危险

UB 的可怕之处在于编译器**可以假设 UB 永远不会发生**，并据此做优化，导致结果和你的直觉完全不符。比如：

```cpp
#include <climits>
#include <iostream>

int main() {
    int x = INT_MAX;
    int y = x + 1;                // 有符号整数溢出：未定义行为
    std::cout << y << '\n';
}
```

在你机器上它可能打印 `-2147483648`，换个编译选项或编译器就可能不同。用 UBSan 可以抓到：

```bash
g++ -std=c++17 -g -fsanitize=undefined ub.cpp -o ub && ./ub
# 运行时会提示类似 "signed integer overflow" 的错误信息
```

**学习期的建议**：编译时始终带上 `-Wall -Wextra`，再加上 `-fsanitize=address,undefined`（第 0 章、第 18 章）。

---

## 2.9 控制流

### `if` / `else`，以及 C++17 的初始化语句

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> score{{"Alice", 90}, {"Bob", 75}};

    // C++17：if (初始化; 条件)，it 的作用域仅限于这个 if/else
    if (auto it = score.find("Alice"); it != score.end()) {
        std::cout << "Alice: " << it->second << '\n';
    } else {
        std::cout << "not found\n";
    }
    // 这里 it 已经不可见，不会污染外层作用域
}
```

### `switch`

```cpp
#include <iostream>

void describe(char grade) {
    switch (grade) {
    case 'A':
    case 'B':                                     // 多个 case 共用一段代码
        std::cout << "good\n";
        break;
    case 'C': {                                   // 需要定义变量时，必须加花括号
        int bonus = 1;
        std::cout << "ok, bonus=" << bonus << '\n';
        break;
    }
    case 'D':
        std::cout << "warn\n";
        [[fallthrough]];                          // C++17：明确表示"故意不 break"，消除警告
    default:
        std::cout << "check again\n";
        break;
    }
}
```

要点：
- `switch` 的条件只能是整数或枚举类型，**不能是字符串**。
- 忘写 `break` 会"贯穿"到下一个 `case`；故意贯穿时写 `[[fallthrough]];`。
- 在 `case` 里直接定义带初始化的变量而不加花括号，会因为"跳过初始化"而编译失败。
- C++17 的 `switch` 同样支持初始化语句：`switch (int v = read(); v) { ... }`。

### 循环

```cpp
// while
int n = 3;
while (n > 0) { --n; }

// do-while：至少执行一次
do { ++n; } while (n < 3);

// 传统 for：循环变量的作用域仅在循环内
for (int i = 0; i < 5; ++i) {
    if (i == 2) continue;       // 跳过本次
    if (i == 4) break;          // 退出循环
}
```

### 范围 `for`（C++11）

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v{1, 2, 3, 4};

    for (int x : v) {                  // 每个元素按值拷贝，修改 x 不会影响 v
        std::cout << x << ' ';
    }
    std::cout << '\n';

    for (int& x : v) {                 // 引用：可以修改元素
        x *= 2;
    }

    for (const auto& x : v) {          // 只读遍历，且避免拷贝（元素是大对象时尤其重要）
        std::cout << x << ' ';
    }
    std::cout << '\n';                 // 2 4 6 8

    int arr[] = {10, 20, 30};
    for (int x : arr) {                // 原生数组也可以
        std::cout << x << ' ';
    }
    std::cout << '\n';
}
```

经验：遍历大对象（如 `std::string`）用 `const auto&`，要修改元素用 `auto&`，基本类型直接按值即可。

注意：遍历过程中不要对容器做插入/删除，那会导致迭代器失效（第 10 章）。

C++ 没有带标签的 `break`，要跳出多层循环，可以用函数 `return`、标志变量，或者 `goto`。

### `goto`

```cpp
for (int i = 0; i < 10; ++i) {
    for (int j = 0; j < 10; ++j) {
        if (i * j == 42) goto found;   // 一次跳出两层循环
    }
}
found:
// ...
```

C 里 `goto cleanup;` 是常见的统一清理资源的写法；C++ 里这个需求由 **RAII**（第 5 章）更好地解决，所以 `goto` 用得很少。另外，C++ 禁止 `goto` 跳过变量的初始化：

```cpp
// goto end;
// int x = 5;       // 如果从上面直接跳到下面的 end，会跳过 x 的初始化 → 编译错误
// end: ;
```

---

## 2.10 结构化绑定（C++17）

一次性把"聚合体"拆成多个变量，代码更清晰。

```cpp
#include <iostream>
#include <map>
#include <string>
#include <tuple>
#include <utility>

struct Point { int x; int y; };

int main() {
    // 1) pair
    std::pair<int, std::string> p{1, "one"};
    auto [id, name] = p;               // 拷贝
    std::cout << id << ' ' << name << '\n';

    auto& [rid, rname] = p;            // 引用：修改会影响 p
    rid = 100;
    std::cout << p.first << '\n';      // 100

    // 2) 结构体（所有非静态成员为 public）
    Point pt{3, 4};
    auto [px, py] = pt;
    std::cout << px << ',' << py << '\n';

    // 3) 数组
    int arr[3] = {1, 2, 3};
    auto [a, b, c] = arr;
    std::cout << a + b + c << '\n';

    // 4) tuple
    std::tuple<int, double, std::string> t{1, 2.5, "x"};
    auto [i, d, s] = t;
    std::cout << i << ' ' << d << ' ' << s << '\n';

    // 5) 遍历 map：最常见的用法
    std::map<std::string, int> m{{"Alice", 90}, {"Bob", 75}};
    for (const auto& [key, value] : m) {
        std::cout << key << " -> " << value << '\n';
    }

    // 6) 接收 insert 的返回值（pair<iterator, bool>）
    auto [it, inserted] = m.insert({"Carol", 88});
    std::cout << it->first << ' ' << std::boolalpha << inserted << '\n';
}
```

规则：
- 方括号里的名字**个数必须和元素个数一致**，不能只取一部分；不想用的变量可以加 `[[maybe_unused]]` 避免未使用警告。
- 前面写 `auto` 是拷贝，写 `auto&` / `const auto&` 是引用。
- 适用对象：数组、`std::pair`/`std::tuple`，以及成员全是 public 的结构体/类。

---

## 2.11 综合示例：成绩统计

把本章的内容串起来（`vector` 和 `map` 属于第 10 章内容，这里只作使用示范）：

```cpp
#include <iostream>
#include <map>
#include <vector>

char to_grade(int s) {
    if (s >= 90) return 'A';
    if (s >= 80) return 'B';
    if (s >= 60) return 'C';
    return 'D';
}

int main() {
    const std::vector<int> scores{95, 82, 67, 45, 88, 91, 73};

    // 求平均：注意先转成 double，否则变成整数除法
    int sum = 0;
    for (int s : scores) {
        sum += s;
    }
    const double avg = static_cast<double>(sum) / static_cast<double>(scores.size());
    std::cout << "average = " << avg << '\n';

    // 统计每个等级的人数
    std::map<char, int> count;
    for (int s : scores) {
        ++count[to_grade(s)];
    }

    // 结构化绑定遍历
    for (const auto& [grade, n] : count) {
        std::cout << grade << ": " << n << '\n';
    }

    // if 初始化语句
    if (auto it = count.find('A'); it != count.end()) {
        std::cout << "A 等级有 " << it->second << " 人\n";
    } else {
        std::cout << "没有 A\n";
    }
}
```

编译运行：

```bash
g++ -std=c++17 -Wall -Wextra -g -fsanitize=address,undefined main.cpp -o main && ./main
```

---

## 2.12 小结

- 内置类型的大小是**平台相关**的，要定宽就用 `<cstdint>`。
- 局部变量不初始化，值就是不确定的；优先用 `{}` 初始化，还能挡住窄化转换。
- `auto` 默认按值，`auto&` / `const auto&` 才是引用；`decltype((x))` 比 `decltype(x)` 多一层引用。
- 整数除法截断、有符号溢出是 UB、无符号会回绕、有符号与无符号混比较会出错。
- 不要在同一个表达式里既读又改同一个变量，也不要依赖函数实参的求值顺序。
- `if`/`switch` 可以带初始化语句（C++17），范围 `for` 遍历大对象用 `const auto&`。
- 结构化绑定让 `pair`、`tuple`、`struct`、数组、`map` 遍历更简洁。
- UB 是 C++ 最大的坑之一：学习期开 `-Wall -Wextra` 和 sanitizers。

## 练习

1. 打印你机器上 `char`、`short`、`int`、`long`、`long long`、`float`、`double` 的大小和各自的最大/最小值。
2. 解释下面每一行的结果，再编译运行验证：`7 / 2`、`-7 / 2`、`-7 % 2`、`7 / 2.0`、`(-1 < 1u)`。
3. 写出 `auto` 推导结果：`const int a = 1; auto b = a; auto& c = a; const auto& d = 5;`，再用 `decltype` 验证。
4. 用 `switch` 写一个简易计算器：读入 `a op b`，支持 `+ - * /`，除数为 0 时给出提示。
5. 写 FizzBuzz（1 到 100），分别用传统 `for` 和 `while` 实现。
6. 用 `std::map<std::string, int>` 做一个单词计数程序，遍历时使用结构化绑定，并用 `if` 初始化语句查询某个词。
7. 故意写一个有符号整数溢出的程序，分别用 `-fsanitize=undefined` 运行和不用它运行，对比现象。
