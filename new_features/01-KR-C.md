# K&R C（1978）

> 来源：Brian Kernighan 与 Dennis Ritchie 的《The C Programming Language》第一版。当时没有正式标准，这本书就是事实标准（"K&R C"）。

## 核心特性

| 特性 | 说明 |
|---|---|
| 基本类型 | `char` `int` `float` `double`，以及 `short` `long` `unsigned` 修饰 |
| 结构体 `struct` | 自定义复合类型（结构体不能整体赋值、传参，只能通过指针） |
| 预处理器 | `#define`、`#include` |
| 复合赋值 | `+=`、`-=` 等（早期写法是 `=+`，K&R 书中改为 `+=`） |
| 指针与数组 | 指针算术、数组名退化为指针 |
| 标准 I/O 库 | `stdio` 的 `printf` / `getchar` / `putchar` 等 |

## 典型的"老式"写法

### 1. 老式函数定义（参数类型写在括号外）

```c
/* K&R 风格 */
int add(a, b)
int a;
int b;
{
    return a + b;
}
```

### 2. 没有函数原型

调用时编译器不检查参数个数和类型：

```c
main()                 /* 隐式 int 返回类型 */
{
    int r = add(1, 2, 3);   /* 多传了参数，编译器也不报错 */
    printf("%d\n", r);      /* 未声明 printf，隐式声明 */
}
```

### 3. 隐式 int

```c
static x;        /* 等价于 static int x; */
foo() { return 1; }  /* 返回类型默认 int */
```

## 局限（后来被标准化解决）

- 没有 `void`、`const`、`volatile`、`enum`（这些在 C89 引入或固定）
- 没有函数原型，类型检查弱
- 结构体按值传递/赋值在不同编译器上不一致
- 标准库没有统一规范，各平台差异大

> 这些问题直接推动了 ANSI C（C89）的诞生。
