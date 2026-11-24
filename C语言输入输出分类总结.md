# C 语言输入与输出（I/O）分类总结

> 目标：把 C
> 标准库中常见的输入、输出函数按用途分清楚，理解它们的区别，并能根据场景选择合适的函数。
>
> 本文以标准 C（C17/C23
> 中长期存在的常用接口）为主。不同操作系统可能提供额外接口，但不属于标准
> C。

------------------------------------------------------------------------

## 1. 先建立整体认识：C 的 I/O 有哪些类别？

C 的输入输出函数主要来自标准头文件：

``` c
#include <stdio.h>
```

可以按数据流向和处理对象分成几类：

  ----------------------------------------------------------------------------------------------------------------
  类别                    代表函数                                                  主要用途
  ----------------------- --------------------------------------------------------- ------------------------------
  格式化输出              `printf`、`fprintf`、`sprintf`、`snprintf`                按格式把数据输出为文本

  格式化输入              `scanf`、`fscanf`、`sscanf`                               按格式从输入中解析数据

  字符 I/O                `getchar`、`putchar`、`fgetc`、`fputc`                    一次处理一个字符

  字符串/行 I/O           `fgets`、`puts`、`fputs`                                  一次读取一行或输出字符串

  二进制/块 I/O           `fread`、`fwrite`                                         按字节块读写数据

  文件管理                `fopen`、`fclose`、`fflush`、`fseek`、`ftell`、`rewind`   打开、关闭、刷新和定位文件流

  错误与状态              `feof`、`ferror`、`clearerr`、`perror`                    检查 I/O 状态和报告错误
  ----------------------------------------------------------------------------------------------------------------

最重要的理解：

**这些函数大多是在操作"流（stream）"，而不是直接操作键盘、屏幕或磁盘硬件。**

标准 C 用 `FILE *` 表示一个文件流。终端输入输出也可以看作预先打开的流：

-   `stdin`：标准输入，通常连接键盘或终端输入。
-   `stdout`：标准输出，通常连接终端。
-   `stderr`：标准错误，通常也连接终端，但用于错误信息。

例如：

``` c
printf("hello\n");
```

通常可以理解为：

``` c
fprintf(stdout, "hello\n");
```

二者都是向标准输出流写入格式化文本。

------------------------------------------------------------------------

## 2. 格式化输出：printf 家族

格式化输出的特点是：**通过格式说明符，把不同类型的数据转换成文本。**

### 2.1 `printf`：输出到标准输出

``` c
#include <stdio.h>

int main(void)
{
    int age = 20;
    double height = 1.75;

    printf("age = %d\n", age);
    printf("height = %.2f\n", height);

    return 0;
}
```

常见格式说明符：

  格式          含义                        示例
  ------------- --------------------------- ------------------------------
  `%d`          `int` 有符号十进制整数      `printf("%d", 10);`
  `%u`          `unsigned int` 十进制整数   `printf("%u", 10u);`
  `%ld`         `long`                      `printf("%ld", 100L);`
  `%lld`        `long long`                 `printf("%lld", 100LL);`
  `%f`          `double` 浮点数             `printf("%f", 3.14);`
  `%.2f`        保留两位小数                `printf("%.2f", 3.14159);`
  `%c`          字符                        `printf("%c", 'A');`
  `%s`          以 `\0` 结尾的字符串        `printf("%s", "hello");`
  `%x` / `%X`   十六进制整数                `printf("%x", 255);`
  `%p`          指针地址                    `printf("%p", (void *)ptr);`
  `%%`          输出一个百分号              `printf("80%%");`

注意：

-   `printf` 中，`float` 参数会发生默认参数提升，按 `double`
    传递，所以浮点输出使用 `%f`。
-   `%p` 对应的实参应转换为 `void *`。
-   格式说明符必须与实参类型匹配，否则可能产生未定义行为。

### 2.2 `fprintf`：输出到指定流

函数原型：

``` c
int fprintf(FILE *stream, const char *format, ...);
```

它和 `printf` 的核心区别是：多了一个 `FILE *stream`
参数，用来指定输出目标。

``` c
#include <stdio.h>

int main(void)
{
    int score = 95;

    fprintf(stdout, "score = %d\n", score);
    fprintf(stderr, "这是错误信息\n");

    return 0;
}
```

常见用法：

``` c
printf("输出到标准输出\n");

fprintf(stdout, "也输出到标准输出\n");

fprintf(stderr, "输出到标准错误\n");
```

记忆：

-   `printf`：默认输出到 `stdout`。
-   `fprintf`：输出到指定的流。

### 2.3 `sprintf`：格式化到字符数组

函数原型：

``` c
int sprintf(char *str, const char *format, ...);
```

它不是把内容输出到屏幕，而是把格式化后的文本写入内存中的字符数组。

``` c
#include <stdio.h>

int main(void)
{
    char buffer[100];

    int id = 7;
    double price = 19.9;

    sprintf(buffer, "id=%d, price=%.1f", id, price);

    printf("%s\n", buffer);

    return 0;
}
```

**重要：`sprintf`
不知道目标数组有多大，可能造成缓冲区溢出。实际编程中通常优先使用
`snprintf`。**

### 2.4 `snprintf`：有长度限制的格式化输出

函数原型：

``` c
int snprintf(char *str, size_t size, const char *format, ...);
```

``` c
#include <stdio.h>

int main(void)
{
    char buffer[16];

    int n = snprintf(buffer, sizeof buffer, "number=%d", 12345);

    printf("buffer: %s\n", buffer);
    printf("理论上需要的字符数（不含结尾的 \\0）：%d\n", n);

    return 0;
}
```

重点：

-   `size` 是目标缓冲区的总容量，包含结尾的 `\0` 所占空间。
-   只要 `size` 大于 0，函数会保证结果以 `\0` 结尾。
-   返回值是**如果空间足够，本来需要写入的字符数**，不包括结尾的 `\0`。
-   如果返回值大于或等于 `size`，说明结果被截断了。
-   `size == 0` 时不写入字符，目标指针可以是空指针。

判断是否被截断：

``` c
int n = snprintf(buffer, sizeof buffer, "hello %d", 123);

if (n < 0) {
    /* 格式化发生错误 */
} else if ((size_t)n >= sizeof buffer) {
    /* 缓冲区不够，结果被截断 */
} else {
    /* 完整写入 */
}
```

### 2.5 四个输出函数放在一起比较

  函数         输出位置        是否格式化   是否指定长度
  ------------ --------------- ------------ --------------
  `printf`     `stdout`        是           不适用
  `fprintf`    指定 `FILE *`   是           不适用
  `sprintf`    字符数组        是           否，风险较高
  `snprintf`   字符数组        是           是

------------------------------------------------------------------------

## 3. 格式化输入：scanf 家族

格式化输入的特点是：**根据格式说明符，从输入流或字符串中解析数据。**

### 3.1 `scanf`：从标准输入读取

``` c
#include <stdio.h>

int main(void)
{
    int age;
    double height;

    printf("请输入年龄和身高：");

    if (scanf("%d %lf", &age, &height) == 2) {
        printf("age = %d, height = %.2f\n", age, height);
    } else {
        printf("输入格式不正确\n");
    }

    return 0;
}
```

这里有两个重要细节：

**第一，为什么要写 `&age`？**

`scanf` 需要把读取到的值写进变量，因此需要变量的地址。

**第二，为什么 `double` 使用 `%lf`？**

在 `scanf` 中：

-   `%f`：对应 `float *`。
-   `%lf`：对应 `double *`。

不要把 `printf` 和 `scanf` 的浮点格式混淆：

  场景       `float`                         `double`
  ---------- ------------------------------- ----------------------
  `printf`   `%f`（传参时提升为 `double`）   `%f`
  `scanf`    `%f`，传 `float *`              `%lf`，传 `double *`

### 3.2 `scanf` 的返回值必须检查

`scanf` 返回成功匹配并赋值的项目数；遇到输入失败可能返回 `EOF`。

``` c
int a, b;

int result = scanf("%d %d", &a, &b);

if (result == 2) {
    printf("a=%d, b=%d\n", a, b);
} else {
    printf("读取失败或输入不完整\n");
}
```

不要默认用户一定输入了正确的数据。

### 3.3 `fscanf`：从指定流读取

函数原型：

``` c
int fscanf(FILE *stream, const char *format, ...);
```

例如从文件读取两个整数：

``` c
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("data.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    int id;
    int score;

    if (fscanf(fp, "%d %d", &id, &score) == 2) {
        printf("id=%d, score=%d\n", id, score);
    } else {
        printf("文件内容格式不正确或读取失败\n");
    }

    fclose(fp);
    return 0;
}
```

记忆：

-   `scanf`：从 `stdin` 读取。
-   `fscanf`：从指定的 `FILE *` 读取。

### 3.4 `sscanf`：从字符串解析数据

函数原型：

``` c
int sscanf(const char *str, const char *format, ...);
```

它不从键盘或文件读取，而是解析已经存在于内存中的字符串。

``` c
#include <stdio.h>

int main(void)
{
    const char *text = "100 3.14 hello";

    int id;
    double value;
    char word[32];

    int n = sscanf(text, "%d %lf %31s", &id, &value, word);

    if (n == 3) {
        printf("id=%d, value=%.2f, word=%s\n", id, value, word);
    }

    return 0;
}
```

`%31s` 是为了给 `word[32]` 留出一个 `\0` 的位置。

### 3.5 三个输入函数比较

  函数       输入来源
  ---------- ------------------
  `scanf`    标准输入 `stdin`
  `fscanf`   指定的文件流
  `sscanf`   内存中的字符串

### 3.6 `scanf` 常见陷阱：读取字符串和换行符

``` c
char name[20];
scanf("%19s", name);
```

`%s`
会读取连续的非空白字符，遇到空格、换行或制表符就停止。因此它不能直接读取带空格的整行文本。

另外，`scanf("%d", &n)`
读取整数后，输入中的换行符通常仍留在输入流里。接下来如果使用 `getchar()`
或 `fgets()`，可能立刻读到这个换行符。

初学阶段可以记住：

-   读取单个数字：`scanf` 可以用，但要检查返回值。
-   读取一整行文本：优先使用 `fgets`。
-   混用 `scanf` 和 `fgets` 时，要明确处理输入流中残留的字符。

------------------------------------------------------------------------

## 4. 字符 I/O：一次读取或写入一个字符

字符 I/O 适合逐字符处理文本、统计字符、扫描输入内容等场景。

### 4.1 `getchar`：从标准输入读取一个字符

``` c
#include <stdio.h>

int main(void)
{
    int ch;

    printf("请输入一个字符：");
    ch = getchar();

    printf("你输入的是：");
    putchar(ch);
    putchar('\n');

    return 0;
}
```

**为什么用 `int`，而不是 `char`？**

因为 `getchar()` 需要同时表示所有可能的 `unsigned char`
值，以及特殊返回值 `EOF`。使用 `int` 才能可靠地区分字符和文件结束标志。

### 4.2 `putchar`：向标准输出写入一个字符

``` c
putchar('A');
putchar('\n');
```

它相当于面向 `stdout` 的字符输出函数。

### 4.3 `fgetc` 和 `fputc`：针对指定流操作字符

``` c
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("data.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    int ch;
    while ((ch = fgetc(fp)) != EOF) {
        putchar(ch);
    }

    if (ferror(fp)) {
        perror("读取文件");
    }

    fclose(fp);
    return 0;
}
```

写入字符：

``` c
FILE *fp = fopen("output.txt", "w");
if (fp == NULL) {
    perror("fopen");
    return 1;
}

fputc('A', fp);
fputc('\n', fp);

fclose(fp);
```

### 4.4 字符函数比较

  函数              操作对象
  ----------------- --------------------------
  `getchar()`       从 `stdin` 读取一个字符
  `putchar(ch)`     向 `stdout` 写入一个字符
  `fgetc(fp)`       从指定流读取一个字符
  `fputc(ch, fp)`   向指定流写入一个字符

`getchar()` 可以理解为 `fgetc(stdin)` 的常用简写；`putchar(ch)`
可以理解为 `fputc(ch, stdout)` 的常用简写。

------------------------------------------------------------------------

## 5. 字符串与行 I/O：fgets、puts、fputs

这组函数在读取文本时非常实用。

### 5.1 `fgets`：读取一行或一段字符

函数原型：

``` c
char *fgets(char *str, int n, FILE *stream);
```

示例：

``` c
#include <stdio.h>

int main(void)
{
    char line[128];

    printf("请输入一行文字：");

    if (fgets(line, sizeof line, stdin) != NULL) {
        printf("读取结果：%s", line);
    }

    return 0;
}
```

`fgets` 的行为：

1.  最多读取 `n - 1` 个字符。
2.  如果在达到上限前遇到换行符，会把换行符也保存到数组中。
3.  最后自动添加 `\0`。
4.  成功时返回目标数组指针；遇到 EOF
    且没有读到字符，或发生读取错误时返回 `NULL`。

例如输入：

``` text
hello world
```

读取结果通常包含末尾的换行符：

``` text
hello world\n\0
```

注意：如果一行太长，`fgets`
可能只读到这一行的一部分。需要继续读取剩余内容时，应当循环处理。

### 5.2 `puts`：输出字符串并自动换行

``` c
puts("hello world");
```

它会输出字符串，并在末尾追加一个换行符。

### 5.3 `fputs`：向指定流输出字符串

``` c
fputs("hello world", stdout);
fputs("\n", stdout);
```

`fputs` 不会自动追加换行符。

### 5.4 三个函数比较

  函数      作用                     是否自动添加换行
  --------- ------------------------ ------------------------------------
  `fgets`   从流读取一段字符         读取到的换行符会保留（如果读到了）
  `puts`    向 `stdout` 输出字符串   是
  `fputs`   向指定流输出字符串       否

### 5.5 读取后去掉末尾换行符

实际开发中，经常希望得到不带换行符的字符串：

``` c
#include <stdio.h>
#include <string.h>

int main(void)
{
    char name[100];

    if (fgets(name, sizeof name, stdin) == NULL) {
        return 1;
    }

    name[strcspn(name, "\n")] = '\0';

    printf("name = %s\n", name);

    return 0;
}
```

`strcspn(name, "\n")`
返回字符串中第一个换行符的位置；如果没有换行符，则返回字符串长度。这样写不会越界。

------------------------------------------------------------------------

## 6. 二进制/块 I/O：fread 和 fwrite

前面很多函数主要面向文本。`fread` 和 `fwrite`
则适合按固定大小的块读写数据。

### 6.1 `fwrite`：把内存数据写入流

函数原型：

``` c
size_t fwrite(
    const void *ptr,
    size_t size,
    size_t count,
    FILE *stream
);
```

示例：写入一个结构体对象的内存表示。

``` c
#include <stdio.h>

struct User {
    int id;
    double score;
};

int main(void)
{
    struct User user = {1, 98.5};

    FILE *fp = fopen("user.bin", "wb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    size_t written = fwrite(&user, sizeof user, 1, fp);

    if (written != 1) {
        perror("fwrite");
        fclose(fp);
        return 1;
    }

    fclose(fp);
    return 0;
}
```

参数含义：

-   `ptr`：数据在内存中的起始地址。
-   `size`：每个数据项的字节数。
-   `count`：要写入多少个数据项。
-   `stream`：写入哪个流。

返回值是成功写入的完整数据项数量，不是字节数。

### 6.2 `fread`：从流读取数据到内存

``` c
#include <stdio.h>

struct User {
    int id;
    double score;
};

int main(void)
{
    struct User user;

    FILE *fp = fopen("user.bin", "rb");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    size_t n = fread(&user, sizeof user, 1, fp);

    if (n == 1) {
        printf("id=%d, score=%.1f\n", user.id, user.score);
    } else if (feof(fp)) {
        printf("文件数据不足或已经到达文件末尾\n");
    } else if (ferror(fp)) {
        perror("fread");
    }

    fclose(fp);
    return 0;
}
```

### 6.3 `fread` 和 `fwrite` 比较

  函数       数据方向    返回值
  ---------- ----------- --------------------------
  `fread`    流 → 内存   成功读取的完整数据项数量
  `fwrite`   内存 → 流   成功写入的完整数据项数量

注意：

-   二进制文件通常使用 `"rb"`、`"wb"` 等模式打开。
-   直接写结构体的内存表示适合特定程序内部的简单场景，但不保证跨平台、跨编译器或跨版本兼容。
-   结构体可能包含填充字节；整数和浮点数的字节序、表示方式也可能不同。
-   如果要设计长期保存或跨平台交换的数据格式，应明确规定字段宽度、字节序和编码方式，而不是直接依赖结构体内存布局。

------------------------------------------------------------------------

## 7. 文件操作：把输入输出连接到文件

### 7.1 `fopen`：打开文件

``` c
FILE *fopen(const char *filename, const char *mode);
```

常见模式：

  模式                     含义
  ------------------------ --------------------------------
  `"r"`                    读取，文件必须存在
  `"w"`                    写入；不存在则创建，存在则清空
  `"a"`                    追加；不存在则创建
  `"r+"`                   读写，文件必须存在
  `"w+"`                   读写；不存在则创建，存在则清空
  `"a+"`                   读取和追加写入；不存在则创建
  `"rb"`、`"wb"`、`"ab"`   对应的二进制模式

在某些平台上，文本模式和二进制模式的区别比较明显；在另一些平台上差别较小。需要可移植地处理二进制数据时，明确使用
`b`。

### 7.2 `fclose`：关闭文件

``` c
if (fclose(fp) == EOF) {
    perror("fclose");
}
```

文件使用完毕应及时关闭。关闭流时，尚未写出的缓冲数据也会尝试写出，因此
`fclose` 本身也可能失败。

### 7.3 `fflush`：刷新输出缓冲区

``` c
printf("请输入：");
fflush(stdout);
```

输出流通常带有缓冲。`fflush(stdout)`
请求把用户态缓冲的输出送往宿主环境。

要点：

-   `fflush` 主要用于刷新输出流。
-   不要把它当作清空键盘输入缓冲区的方法。
-   对输入流调用 `fflush` 在 ISO C
    中没有可移植的通用定义；不要依赖这种写法。

### 7.4 `fseek`、`ftell`、`rewind`：文件位置控制

``` c
fseek(fp, 0, SEEK_SET);   /* 移到文件开头 */
fseek(fp, 0, SEEK_END);   /* 移到文件末尾 */
long pos = ftell(fp);     /* 获取当前位置 */
rewind(fp);               /* 回到文件开头，并清除 EOF 标志 */
```

`fseek` 的第三个参数常见取值：

-   `SEEK_SET`：相对于文件开头。
-   `SEEK_CUR`：相对于当前位置。
-   `SEEK_END`：相对于文件末尾。

注意：`ftell` 返回 `long`，失败时返回
`-1L`。对大型文件或需要精确可移植定位的场景，应了解目标平台提供的扩展接口及其限制。

------------------------------------------------------------------------

## 8. EOF、feof、ferror：如何判断读取结束或出错？

### 8.1 `EOF` 是什么？

`EOF` 是 `stdio.h`
中定义的一个负整数常量，用于表示输入函数遇到文件结束或某些读取失败情况。

例如：

``` c
int ch;

while ((ch = getchar()) != EOF) {
    putchar(ch);
}
```

这里必须用 `int ch`，不能用 `char ch` 来可靠地区分所有字符值和 `EOF`。

### 8.2 不要这样判断文件是否结束

不推荐：

``` c
while (!feof(fp)) {
    /* 读取数据 */
}
```

原因：`feof`
只有在一次读取操作已经尝试读取并遇到文件末尾后，才会设置对应的 EOF
指示器。它不是"预先预测下一次读取是否会结束"。

正确方式是：**先读取，再根据读取函数的返回值判断是否成功。**

字符读取：

``` c
int ch;

while ((ch = fgetc(fp)) != EOF) {
    putchar(ch);
}

if (ferror(fp)) {
    perror("读取文件");
}
```

块读取：

``` c
size_t n;

while ((n = fread(buffer, 1, sizeof buffer, fp)) > 0) {
    /* 处理 buffer 中的 n 个字节 */
}

if (ferror(fp)) {
    perror("读取文件");
}
```

### 8.3 状态函数

  函数             用途
  ---------------- -------------------------------
  `feof(fp)`       检查流的 EOF 指示器是否已设置
  `ferror(fp)`     检查流的错误指示器是否已设置
  `clearerr(fp)`   清除 EOF 和错误指示器
  `perror(msg)`    根据当前 `errno` 输出错误说明

`feof` 和 `ferror` 返回非零表示对应状态已设置。

------------------------------------------------------------------------

## 9. 一个完整小例子：读取文本文件并逐行输出

这个例子把 `fopen`、`fgets`、`fputs`、`ferror` 和 `fclose` 串起来：

``` c
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("data.txt", "r");
    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    char line[256];

    while (fgets(line, sizeof line, fp) != NULL) {
        fputs(line, stdout);
    }

    if (ferror(fp)) {
        perror("读取文件");
        fclose(fp);
        return 1;
    }

    if (fclose(fp) == EOF) {
        perror("fclose");
        return 1;
    }

    return 0;
}
```

执行过程：

1.  `fopen` 打开文件，得到 `FILE *`。
2.  `fgets` 每次读取一段文本。
3.  `fputs` 把读取到的文本输出到标准输出。
4.  读取循环结束后，用 `ferror` 检查是否发生读取错误。
5.  `fclose` 关闭文件。

------------------------------------------------------------------------

## 10. 该怎么选？常见场景速查

  需求                       优先考虑
  -------------------------- -----------------------
  在终端输出带变量的文本     `printf`
  把格式化文本写入文件       `fprintf`
  把格式化结果放进字符数组   `snprintf`
  从终端读取简单格式化数据   `scanf`，并检查返回值
  从文件解析格式化数据       `fscanf`
  从已有字符串中解析数据     `sscanf`
  从终端读取一个字符         `getchar`
  向终端输出一个字符         `putchar`
  逐字符读取文件             `fgetc`
  读取一行文本               `fgets`
  输出字符串且自动换行       `puts`
  向指定流输出字符串         `fputs`
  读写二进制数据块           `fread` / `fwrite`
  打开和关闭文件             `fopen` / `fclose`
  刷新标准输出               `fflush(stdout)`

------------------------------------------------------------------------

## 11. 函数家族的命名规律

很多函数名字相似，是因为它们在做同一类工作，只是目标流或数据来源不同。

### 11.1 `printf` 家族

  函数         含义
  ------------ --------------------------------
  `printf`     格式化输出到标准输出
  `fprintf`    格式化输出到指定流
  `sprintf`    格式化输出到字符数组
  `snprintf`   限制长度的格式化输出到字符数组

### 11.2 `scanf` 家族

  函数       含义
  ---------- ----------------------
  `scanf`    从标准输入格式化读取
  `fscanf`   从指定流格式化读取
  `sscanf`   从字符串格式化解析

### 11.3 字符 I/O 家族

  函数        含义
  ----------- --------------------
  `getchar`   从标准输入读取字符
  `putchar`   向标准输出写入字符
  `fgetc`     从指定流读取字符
  `fputc`     向指定流写入字符

### 11.4 字符串/行 I/O 家族

  函数      含义
  --------- ------------------------------
  `fgets`   从指定流读取一段字符
  `puts`    向标准输出写字符串并换行
  `fputs`   向指定流写字符串，不自动换行

### 11.5 块 I/O 家族

  函数       含义
  ---------- ----------------------
  `fread`    从流读取数据块到内存
  `fwrite`   从内存写入数据块到流

------------------------------------------------------------------------

## 12. 初学者最值得先掌握的组合

不需要一开始就把所有函数背下来。建议按下面的顺序练习：

1.  **终端格式化输出：** `printf`
2.  **终端简单输入：** `scanf`，重点练习返回值检查和 `&`
3.  **字符输入输出：** `getchar`、`putchar`
4.  **安全地读取一行：** `fgets`
5.  **字符串输出：** `puts`、`fputs`
6.  **文件文本读写：** `fopen`、`fgets`、`fputs`、`fclose`
7.  **格式化文件读写：** `fprintf`、`fscanf`
8.  **二进制文件读写：** `fread`、`fwrite`
9.  **缓冲、定位和错误处理：**
    `fflush`、`fseek`、`ftell`、`feof`、`ferror`

### 最后记住三个核心概念

-   **流（stream）**：输入输出数据的通道，标准 C 用 `FILE *` 表示。
-   **格式化 I/O**：`printf` / `scanf`
    这一类，负责把文本和各种数据类型互相转换。
-   **字符或块 I/O**：`fgetc` / `fgets` / `fread`
    等，按字符、字符串片段或字节块处理数据。

把这三个概念分清楚，C 语言的输入输出函数就不会再显得杂乱。
