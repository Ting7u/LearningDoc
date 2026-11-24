# GDB 完整学习指南：C 语言程序调试

> 适用环境：Linux / Ubuntu / WSL2 + GCC + GDB  
> 适用语言：C 语言（部分内容同样适用于 C++）  
> 学习目标：从零掌握 GDB，能够独立调试普通程序、多线程程序、网络程序以及大型 C 项目。

---

# 一、GDB 是什么？

## 1.1 GDB 的定义

GDB（GNU Debugger）是 GNU 项目提供的源代码级调试器。

它可以让我们观察程序运行时的内部状态，包括：

- 当前执行到了哪一行代码。
    
- 某个变量当前的值是多少。
    
- 某个指针指向什么地址。
    
- 函数调用的顺序是什么。
    
- 函数调用时传递了什么参数。
    
- 程序为什么发生段错误。
    
- 哪个线程正在执行。
    
- 多个线程分别停留在哪里。
    
- 某个变量究竟在哪里被修改。
    

简单来说：

**GDB 可以让我们暂停程序，检查程序内部状态，然后决定接下来如何执行。**

## 1.2 为什么需要 GDB？

例如，我们有以下代码：

```c
#include <stdio.h>

int main(void)
{
    int a = 10;
    int b = 20;
    int c = a + b;

    printf("c = %d\n", c);

    return 0;
}
```

正常运行：

```
gcc main.c -o main
./main
```

输出：

```
c = 30
```

但是，如果程序出现问题，我们可能需要知道：

- `a` 到底是多少？
    
- `b` 是在哪里被修改的？
    
- `c` 为什么计算错误？
    
- 程序执行到哪一步出现了异常？
    

如果只使用 `printf()`，需要不断修改代码、重新编译、重新运行。

使用 GDB 则可以直接检查。

例如：

```
gdb ./main
```

然后：

```
(gdb) break main
(gdb) run
(gdb) next
(gdb) print a
```

GDB 就可以在程序运行过程中检查变量。

## 1.3 GDB 与 printf 调试的区别

|对比|printf|GDB|
|---|---|---|
|查看变量|可以|可以|
|查看内存|需要自己编写代码|直接查看|
|暂停程序|不方便|可以|
|单步执行|不可以|可以|
|查看调用栈|需要额外实现|可以|
|调试多线程|比较困难|支持|
|定位段错误|比较困难|非常方便|
|修改运行时变量|需要修改代码|可以直接修改|

两种方法并不冲突，实际开发中可以结合使用。

---

# 二、GDB 的工作原理

## 2.1 GDB 如何调试程序？

首先理解一个问题：

GDB 为什么能够知道程序内部的变量？

因为程序编译时，可以生成额外的调试信息。

正常编译：

```
gcc main.c -o main
```

加入调试信息：

```
gcc -g main.c -o main
```

这里的 `-g` 非常重要。

它会让 GCC 生成供调试器使用的信息，例如：

- 源文件信息。
    
- 源代码行号与机器指令的对应关系。
    
- 变量和类型信息。
    
- 函数及其相关信息。
    

这些信息通常保存在 ELF 文件的调试信息相关段中。

例如：

```
main
│
├── .text       程序机器指令
├── .data       已初始化的全局变量
├── .bss        未初始化的全局变量
├── .symtab     符号表（如果保留）
├── .strtab     字符串表（如果保留）
└── .debug_*    调试信息（如果生成）
```

注意：

这里是示意结构，并不是所有 ELF 文件都一定包含上述所有段。

## 2.2 GDB 与被调试程序的关系

可以理解为：

```
             用户输入 GDB 命令
                     |
                     v
              +-------------+
              |     GDB     |
              |             |
              | 断点管理    |
              | 变量检查    |
              | 单步执行    |
              | 栈回溯      |
              +-------------+
                     |
                     | 调试接口
                     v
              +-------------+
              | 被调试进程  |
              |             |
              | 线程 1      |
              | 线程 2      |
              | 线程 3      |
              +-------------+
                     |
                     v
                 CPU 执行
```

在 Linux 上，GDB 通常通过 `ptrace` 等操作系统调试机制控制被调试进程。

GDB 能够：

1. 控制程序执行。
    
2. 让程序暂停。
    
3. 读取寄存器和内存。
    
4. 检查线程状态。
    
5. 修改部分运行时状态。
    
6. 让程序继续执行。
    

## 2.3 GDB 并不是 CPU

需要明确：

GDB 本身不是执行程序的 CPU。

真正执行机器指令的是 CPU。

GDB 主要负责控制和观察被调试程序。

---

# 三、安装 GDB

Ubuntu / WSL2：

```
sudo apt update
sudo apt install gdb
```

查看版本：

```
gdb --version
```

查看安装位置：

```
which gdb
```

查看帮助：

```
gdb --help
```

检查 GCC：

```
gcc --version
```

---

# 四、编译适合调试的程序

这是使用 GDB 前最重要的一步。

## 4.1 使用 -g 编译

创建 `main.c`：

```c
#include <stdio.h>

int add(int a, int b)
{
    int result = a + b;
    return result;
}

int main(void)
{
    int x = 10;
    int y = 20;

    int result = add(x, y);

    printf("result = %d\n", result);

    return 0;
}
```

编译：

```
gcc -g main.c -o main
```

启动：

```
gdb ./main
```

## 4.2 推荐使用 -O0 (O 代表 Optimization（优化）)

实际开发中，调试阶段建议：

```
gcc -g -O0 main.c -o main
```

参数含义：

|参数|含义|
|---|---|
|`-g`|生成调试信息|
|`-O0`|关闭优化|
|`-O1`|开启一级优化|
|`-O2`|开启二级优化|
|`-O3`|开启更激进的优化|

为什么推荐 `-O0`？

因为编译器优化可能导致：

- 变量被优化掉。
    
- 多条源代码对应的机器指令顺序发生变化。
    
- 函数被内联。
    
- 某些变量不再保存在预期的内存位置。
    

因此，调试时可能看到：

```
value = <optimized out>
```

这不一定代表变量不存在，而可能是编译器优化导致 GDB 无法按照预期显示它。

## 4.3 多文件项目如何编译？

例如：

```
project/
├── main.c
├── math.c
└── math.h
```

可以这样编译：

```
gcc -g -O0 main.c math.c -o app
```

如果使用 Makefile：

```
CC = gcc

CFLAGS = -Wall -Wextra -g -O0

TARGET = app

SRCS = main.c math.c

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(SRCS) -o $(TARGET)

clean:
	rm -f $(TARGET)
```

执行：

```
make
```

---

# 五、启动 GDB

## 5.1 直接启动

```
gdb ./main
```

进入后：

```
(gdb)
```

这就是 GDB 命令提示符。

## 5.2 启动时自动运行程序

```
gdb -ex run ./main
```

## 5.3 指定程序参数

假设程序：

```
#include <stdio.h>

int main(int argc, char *argv[])
{
    printf("argc = %d\n", argc);

    if (argc > 1) {
        printf("argv[1] = %s\n", argv[1]);
    }

    return 0;
}
```

启动：

```
gdb ./main
```

在 GDB 中：

```
(gdb) set args hello world
(gdb) run
```

也可以：

```
(gdb) run hello world
```

## 5.4 退出 GDB

```
(gdb) quit
```

简写：

```
(gdb) q
```

---

# 六、断点 Breakpoint

断点是 GDB 最核心的功能之一。

断点的作用：

**让程序执行到指定位置时暂停。**

## 6.1 在函数处设置断点

```
(gdb) break main
```

简写：

```
(gdb) b main
```

在 add 函数处设置：

```
(gdb) break add
```

## 6.2 在指定行设置断点

```
(gdb) break main.c:10
```

表示在 `main.c` 的第 10 行设置断点。

## 6.3 在指定地址设置断点

```
(gdb) break *0x401234
```

在指定机器指令地址设置断点。

注意：地址必须是当前程序中有效的指令地址，实际地址还可能受到 PIE 和 ASLR 等因素影响。

## 6.4 查看所有断点

```
(gdb) info breakpoints
```

简写：

```
(gdb) info b
```

## 6.5 删除断点

删除指定断点：

```
(gdb) delete 1
```

删除所有断点：

```
(gdb) delete
```

## 6.6 禁用和启用断点

禁用：

```
(gdb) disable 1
```

启用：

```
(gdb) enable 1
```

区别：

- `delete`：删除断点。
    
- `disable`：暂时关闭断点。
    
- `enable`：重新启用断点。
    

## 6.7 条件断点

这是非常实用的功能。

例如：

```
#include <stdio.h>

int main(void)
{
    for (int i = 0; i < 100; i++) {
        printf("%d\n", i);
    }

    return 0;
}
```

如果我们只想在 `i == 50` 时暂停：

```
(gdb) break main.c:6 if i == 50
```

这样就不用每次循环都停下来。

也可以给已经设置的断点添加条件：

```
(gdb) condition 1 i == 50
```

取消条件：

```
(gdb) condition 1
```

## 6.8 忽略断点次数

```
(gdb) ignore 1 10
```

表示断点 1 接下来命中时，忽略前 10 次。

适合调试循环和重复调用的函数。

---

# 七、单步执行 Step / Next

这是 GDB 中另一个非常核心的知识点。

先理解两个概念：

- 进入函数内部执行。
    
- 不进入函数内部执行。
    

## 7.1 step：进入函数

```
(gdb) step
```

简写：

```
(gdb) s
```

作用：

执行下一行源代码，如果这一行调用了有调试信息的函数，就进入该函数。

例如：

```
int add(int a, int b)
{
    return a + b;
}

int main(void)
{
    int result = add(10, 20);

    return 0;
}
```

假设当前停留在：

```
int result = add(10, 20);
```

执行：

```
(gdb) step
```

就会进入 `add()`。

## 7.2 next：不进入函数

```
(gdb) next
```

简写：

```
(gdb) n
```

作用：

执行下一行源代码，但是不进入函数内部。

例如：

```
int result = add(10, 20);
```

执行 `next` 后，GDB 会执行整个 `add()` 函数，然后停留在调用语句之后的下一条可执行源代码位置。

## 7.3 step 和 next 的区别

|命令|含义|
|---|---|
|`step`|单步执行，遇到函数调用时进入函数|
|`next`|单步执行，遇到函数调用时不进入函数|
|`continue`|继续运行，直到下一个停止事件|
|`finish`|执行完当前函数并返回调用者|
|`until`|执行到指定位置或当前循环退出|

注意：`step` 和 `next` 是源代码级别的单步执行，不保证每条 C 语句恰好对应一条机器指令。

## 7.4 finish：执行完当前函数

```
(gdb) finish
```

假设当前处于：

```
int add(int a, int b)
{
    return a + b;
}
```

执行 `finish`：

GDB 会继续执行，直到当前函数返回。

## 7.5 continue：继续运行

```
(gdb) continue
```

简写：

```
(gdb) c
```

从当前位置继续执行，直到遇到其他断点、信号等停止事件，或者程序结束。

## 7.6 until：跳出循环

```
(gdb) until
```

常用于快速跳出当前循环。

也可以指定目标行：

```
(gdb) until 20
```

## 7.7 指令级单步

有时候我们不仅想看 C 代码，还想看 CPU 实际执行的机器指令。

这时候使用：

```
(gdb) stepi
```

简写：

```
(gdb) si
```

执行一条机器指令，并可能进入被调用函数。

不进入调用函数：

```
(gdb) nexti
```

简写：

```
(gdb) ni
```

相关命令：

|命令|作用|
|---|---|
|`step`|源代码级单步，进入函数|
|`next`|源代码级单步，不进入函数|
|`stepi`|机器指令级单步|
|`nexti`|机器指令级单步，不进入调用函数|

---

# 八、查看变量和表达式

## 8.1 print：打印变量

```
(gdb) print a
```

简写：

```
(gdb) p a
```

例如：

```
int a = 10;
int b = 20;
int c = a + b;
```

可以：

```
(gdb) p a
$1 = 10

(gdb) p b
$2 = 20

(gdb) p c
$3 = 30
```

## 8.2 打印表达式

```
(gdb) p a + b
```

```
(gdb) p a * b
```

```
(gdb) p sizeof(int)
```

GDB 可以计算很多 C 语言表达式。

## 8.3 修改变量

```
(gdb) set variable a = 100
```

或者：

```
(gdb) p a = 100
```

此时变量的运行时值就会改变。

注意：

修改的是当前被调试进程的运行时状态，不会自动修改源代码。

## 8.4 自动显示变量

```
(gdb) display a
```

之后每次程序暂停，GDB 都会自动显示 `a` 的值。

查看自动显示列表：

```
(gdb) info display
```

取消：

```
(gdb) undisplay 1
```

## 8.5 查看变量类型

```
(gdb) ptype a
```

例如：

```
type = int
```

也可以：

```
(gdb) whatis a
```

两者都能帮助我们了解变量类型。

---

# 九、指针与内存调试

对于 C 语言程序员，这部分尤其重要。

## 9.1 查看指针地址

```
int a = 100;
int *p = &a;
```

GDB：

```
(gdb) p p
```

查看指针保存的地址。

```
(gdb) p &a
```

查看变量 `a` 的地址。

## 9.2 查看指针指向的值

```
(gdb) p *p
```

查看指针指向的对象。

## 9.3 查看内存

GDB 提供 `x` 命令。

格式：

```
x/nfu address
```

含义：

- `n`：查看多少个单位。
    
- `f`：显示格式。
    
- `u`：每个单位的大小。
    
- `address`：内存地址。
    

例如：

```
(gdb) x/4xw 0x地址
```

表示从指定地址开始，以十六进制查看 4 个 word。

常用格式：

|参数|含义|
|---|---|
|`x`|十六进制|
|`d`|十进制|
|`u`|无符号十进制|
|`o`|八进制|
|`t`|二进制|
|`c`|字符|
|`s`|字符串|
|`i`|机器指令|

常用大小：

|参数|大小|
|---|---|
|`b`|1 字节|
|`h`|2 字节|
|`w`|4 字节|
|`g`|8 字节|

例如：

```
(gdb) x/16xb p
```

从指针 `p` 指向的地址开始查看 16 个字节。

查看字符串：

```
(gdb) x/s p
```

查看机器指令：

```
(gdb) x/10i $pc
```

这里 `$pc` 是当前程序计数器对应的寄存器名称，具体名称与目标架构有关。

## 9.4 查看数组

```
int arr[5] = {10, 20, 30, 40, 50};
```

GDB：

```
(gdb) p arr
(gdb) p arr[0]
(gdb) p arr[3]
```

查看整个数组：

```
(gdb) p *arr@5
```

这里 `@` 是 GDB 提供的数组切片操作语法。

---

# 十、查看函数调用栈 Stack

这是定位复杂程序错误的关键技能。

## 10.1 什么是调用栈？

例如：

```
void func3(void)
{
    int a = 10;
}

void func2(void)
{
    func3();
}

void func1(void)
{
    func2();
}

int main(void)
{
    func1();

    return 0;
}
```

函数调用关系：

```
main()
  |
  v
func1()
  |
  v
func2()
  |
  v
func3()
```

当程序正在执行 `func3()` 时，调用栈可以帮助我们了解它是如何被调用的。

## 10.2 bt：查看调用栈

```
(gdb) bt
```

简写：

```
(gdb) backtrace
```

示意输出：

```
#0 func3()
#1 func2()
#2 func1()
#3 main()
```

这表示：

- 当前处于 `func3()`。
    
- `func3()` 是 `func2()` 调用的。
    
- `func2()` 是 `func1()` 调用的。
    
- `func1()` 是 `main()` 调用的。
    

## 10.3 查看所有调用栈

```
(gdb) bt full
```

除了调用关系，还会尽可能显示各栈帧中的局部变量。

## 10.4 切换栈帧

```
(gdb) frame 1
```

切换到第 1 个栈帧。

查看：

```
(gdb) info frame
```

查看当前栈帧的信息。

## 10.5 查看调用参数

```
(gdb) info args
```

查看当前函数的参数。

```
(gdb) info locals
```

查看当前函数的局部变量。

这两个命令在调试复杂函数时非常有用。

---

# 十一、查看源代码和汇编

## 11.1 list：查看源代码

```
(gdb) list
```

显示当前执行位置附近的源代码。

指定函数：

```
(gdb) list main
```

指定行：

```
(gdb) list 20
```

## 11.2 disassemble：查看汇编

```
(gdb) disassemble main
```

查看 `main()` 对应的机器指令。

指定范围：

```
(gdb) disassemble main
```

如果需要查看当前函数：

```
(gdb) disassemble
```

## 11.3 layout：TUI 界面

GDB 提供终端文本界面。

显示源代码：

```
(gdb) layout src
```

显示汇编：

```
(gdb) layout asm
```

同时显示源代码和汇编：

```
(gdb) layout split
```

显示寄存器：

```
(gdb) layout regs
```

退出当前 TUI 布局：

```
(gdb) tui disable
```

---

# 十二、监视点 Watchpoint

普通断点关注的是：

**程序执行到哪里。**

监视点关注的是：

**变量什么时候发生变化。**

这两者非常重要。

## 12.1 watch：监视变量写入

例如：

```
int count = 0;

void add(void)
{
    count++;
}
```

我们想知道：

`count` 到底在哪里被修改？

可以：

```
(gdb) watch count
```

当 GDB 检测到 `count` 的值发生变化时，程序会暂停。

## 12.2 rwatch：监视读取

```
(gdb) rwatch count
```

当程序读取该表达式时停止。

## 12.3 awatch：监视访问

```
(gdb) awatch count
```

当程序读取或写入该表达式时停止。

## 12.4 三者区别

|命令|作用|
|---|---|
|`watch`|监视写入导致的值变化|
|`rwatch`|监视读取|
|`awatch`|监视读取或写入|

注意：

监视点的具体实现和数量受到硬件调试寄存器、目标平台及 GDB 后端支持的限制。

它并不是可以无限创建的普通断点。

---

# 十三、调试多线程程序

这一部分与你学习 pthread 非常相关。

假设有三个线程：

```
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void *worker(void *arg)
{
    int id = *(int *)arg;

    while (1) {
        printf("thread %d running\n", id);
        sleep(1);
    }

    return NULL;
}

int main(void)
{
    pthread_t t1, t2, t3;

    int a = 1;
    int b = 2;
    int c = 3;

    pthread_create(&t1, NULL, worker, &a);
    pthread_create(&t2, NULL, worker, &b);
    pthread_create(&t3, NULL, worker, &c);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    return 0;
}
```

编译：

```
gcc -g -O0 main.c -o main -pthread
```

## 13.1 查看所有线程

```
(gdb) info threads
```

示意输出：

```
  Id   Target Id
* 1    Thread 0x... (LWP ...)
  2    Thread 0x... (LWP ...)
  3    Thread 0x... (LWP ...)
  4    Thread 0x... (LWP ...)
```

其中：

- `Id` 是 GDB 分配的线程编号。
    
- `Target Id` 是目标系统提供的线程标识。
    
- `*` 表示当前选中的线程。
    

注意：Linux 进程通常还包含主线程，因此这里有可能显示 4 个线程，而不只是我们创建的 3 个工作线程。

## 13.2 切换线程

```
(gdb) thread 2
```

切换到 GDB 编号为 2 的线程。

查看该线程的调用栈：

```
(gdb) bt
```

## 13.3 查看所有线程的调用栈

```
(gdb) thread apply all bt
```

查看所有线程的调用栈。

如果希望同时查看各线程的局部变量：

```
(gdb) thread apply all bt full
```

这是排查多线程死锁的重要命令。

## 13.4 线程相关断点

普通断点默认可以被所有线程触发。

如果只想让某个线程触发：

```
(gdb) break worker thread 2
```

这里的 `2` 是 GDB 显示的线程编号。

## 13.5 GDB 中的线程执行模式

GDB 通常使用 all-stop 模式：

当某个线程触发断点时，整个被调试程序中的其他线程也会暂停。

这有助于观察某一时刻的整体状态。

还存在 non-stop 模式，允许不同线程独立停止和运行。

需要注意：

调试器暂停线程会改变程序的运行时机，因此某些并发问题可能在调试时消失。

这就是多线程调试中的一个重要难点。

---

# 十四、调试段错误 Segmentation Fault

段错误是 C 语言开发中非常常见的问题。

例如：

```
#include <stdio.h>

int main(void)
{
    int *p = NULL;

    *p = 100;

    return 0;
}
```

编译：

```
gcc -g -O0 main.c -o main
```

运行：

```
./main
```

可能出现：

```
Segmentation fault (core dumped)
```

## 14.1 使用 GDB 直接调试

```
gdb ./main
```

然后：

```
(gdb) run
```

程序崩溃后，GDB 通常会显示收到 `SIGSEGV` 信号。

例如：

```
Program received signal SIGSEGV, Segmentation fault.
```

查看调用栈：

```
(gdb) bt
```

查看当前代码：

```
(gdb) list
```

查看变量：

```
(gdb) print p
```

这样就能定位发生错误的位置。

## 14.2 常见段错误原因

- 空指针解引用。
    
- 野指针访问。
    
- 数组越界。
    
- 已释放内存再次访问。
    
- 栈内存访问越界。
    
- 错误的指针类型转换。
    
- 非法内存写入。
    

---

# 十五、Core Dump 核心转储

Core Dump 是程序崩溃时保存下来的进程状态信息。

可以理解为：

**程序崩溃瞬间的现场记录。**

它可以帮助我们在程序已经退出之后，分析当时发生了什么。

## 15.1 开启 Core Dump

查看限制：

```
ulimit -c
```

如果输出：

```
0
```

说明当前 Shell 的 Core Dump 大小限制为 0。

临时设置：

```
ulimit -c unlimited
```

然后运行程序：

```
./main
```

如果发生符合 Core Dump 条件的异常，系统可能生成核心转储文件。

注意：是否生成还受到系统 Core Dump 配置、权限、磁盘空间等因素影响。

## 15.2 使用 Core 文件调试

假设生成了：

```
core
```

使用：

```
gdb ./main core
```

进入后：

```
(gdb) bt
```

查看崩溃时的调用栈。

```
(gdb) info registers
```

查看寄存器状态。

```
(gdb) list
```

查看对应源代码。

## 15.3 systemd 系统中的 Core Dump

Ubuntu 等系统可能使用 systemd-coredump 管理核心转储。

可以查看：

```
coredumpctl list
```

查看某个进程的转储：

```
coredumpctl info PID
```

使用 GDB 打开：

```
coredumpctl debug PID
```

具体是否可用取决于系统是否安装和启用了对应组件。

---

# 十六、调试正在运行的进程

有时候程序已经运行了，我们不想重新启动。

这时可以让 GDB 附加到进程。

## 16.1 查找 PID

```
ps aux | grep main
```

或者：

```
pgrep main
```

假设 PID 是：

```
12345
```

## 16.2 附加进程

```
gdb -p 12345
```

或者：

```
(gdb) attach 12345
```

附加成功后，GDB 可以检查进程当前状态。

例如：

```
(gdb) info threads
(gdb) thread apply all bt
```

## 16.3 解除附加

```
(gdb) detach
```

解除调试器与进程的连接。

通常进程会继续运行。

## 16.4 注意事项

附加进程需要满足系统权限要求。

例如：

- 进程属于当前用户。
    
- 系统 ptrace 权限策略允许。
    
- 容器或安全策略没有禁止调试。
    

另外，附加操作通常会暂时暂停目标进程。

---

# 十七、调试动态库

C 项目中经常使用动态库：

```
libc.so
libpthread.so
libcurl.so
libssl.so
```

有时候问题不在自己的代码中，而在动态库调用过程中。

可以使用：

```
(gdb) info sharedlibrary
```

查看当前加载的共享库。

如果需要调试共享库内部代码，需要相应的调试符号和源代码。

设置共享库函数断点：

```
(gdb) break function_name
```

如果函数尚未加载，可以考虑：

```
(gdb) set breakpoint pending on
```

然后再设置断点。

---

# 十八、GDB 常用命令速查表

## 18.1 启动与退出

|命令|作用|
|---|---|
|`gdb ./main`|启动 GDB|
|`run`|运行程序|
|`run arg1 arg2`|带参数运行|
|`set args ...`|设置程序参数|
|`start`|从 main 函数附近开始调试|
|`quit`|退出 GDB|
|`kill`|终止当前被调试程序|
|`attach PID`|附加进程|
|`detach`|解除附加|

## 18.2 断点

|命令|作用|
|---|---|
|`break main`|在 main 设置断点|
|`break file.c:10`|在指定行设置断点|
|`break func`|在函数处设置断点|
|`break *address`|在指定地址设置断点|
|`info breakpoints`|查看断点|
|`delete 1`|删除断点|
|`disable 1`|禁用断点|
|`enable 1`|启用断点|
|`condition 1 expr`|设置条件断点|
|`ignore 1 10`|忽略前 10 次命中|

## 18.3 执行控制

|命令|作用|
|---|---|
|`next`|单步，不进入函数|
|`step`|单步，进入函数|
|`continue`|继续运行|
|`finish`|执行完当前函数|
|`until`|执行到指定位置或循环退出|
|`stepi`|执行一条机器指令|
|`nexti`|执行一条机器指令，不进入调用|
|`advance`|运行到指定位置|

## 18.4 变量和内存

|命令|作用|
|---|---|
|`print a`|查看变量|
|`print &a`|查看变量地址|
|`print *p`|查看指针指向的值|
|`ptype a`|查看类型定义|
|`whatis a`|查看变量类型|
|`set variable a=10`|修改变量|
|`display a`|自动显示变量|
|`undisplay 1`|取消自动显示|
|`x/16xb p`|查看 16 个字节|
|`x/s p`|查看字符串|
|`x/10i $pc`|查看当前机器指令|

## 18.5 函数调用栈

|命令|作用|
|---|---|
|`bt`|查看调用栈|
|`bt full`|查看调用栈和局部变量|
|`frame 1`|切换栈帧|
|`up`|切换到上一层调用者|
|`down`|切换到下一层被调用者|
|`info frame`|查看栈帧信息|
|`info args`|查看函数参数|
|`info locals`|查看局部变量|

## 18.6 多线程

|命令|作用|
|---|---|
|`info threads`|查看所有线程|
|`thread 2`|切换线程|
|`thread apply all bt`|查看所有线程调用栈|
|`thread apply all bt full`|查看所有线程的完整调用栈|
|`break func thread 2`|指定线程断点|

## 18.7 其他

|命令|作用|
|---|---|
|`list`|查看源代码|
|`disassemble`|查看汇编|
|`info registers`|查看寄存器|
|`info program`|查看程序状态|
|`info sharedlibrary`|查看动态库|
|`help`|查看帮助|
|`apropos keyword`|搜索相关命令|

---

# 十九、GDB 实战案例

这一节把前面的知识串联起来。

## 案例一：定位变量计算错误

代码：

```
#include <stdio.h>

int calculate(int a, int b)
{
    int result = a * b;
    result += 10;

    return result;
}

int main(void)
{
    int x = 10;
    int y = 20;

    int result = calculate(x, y);

    printf("%d\n", result);

    return 0;
}
```

编译：

```
gcc -g -O0 main.c -o main
```

启动：

```
gdb ./main
```

调试：

```
(gdb) break main
(gdb) run
(gdb) next
(gdb) next
(gdb) next
```

进入函数：

```
(gdb) step
```

查看参数：

```
(gdb) info args
```

查看局部变量：

```
(gdb) info locals
```

查看计算结果：

```
(gdb) print result
```

继续：

```
(gdb) next
```

执行完函数：

```
(gdb) finish
```

查看返回结果：

```
(gdb) print result
```

## 案例二：定位变量被意外修改

代码：

```
#include <stdio.h>

int main(void)
{
    int count = 0;

    count = 10;
    count = 20;
    count = 30;

    printf("%d\n", count);

    return 0;
}
```

我们不知道哪一步把 `count` 修改成了 30。

可以：

```
(gdb) break main
(gdb) run
(gdb) watch count
(gdb) continue
```

之后 GDB 会在监视表达式发生变化时暂停。

查看：

```
(gdb) print count
(gdb) bt
```

这样就能追踪变量的修改过程。

## 案例三：定位多线程程序死锁

假设程序有两个线程：

```
线程 A：
    获取 mutex1
    获取 mutex2

线程 B：
    获取 mutex2
    获取 mutex1
```

如果两个线程分别持有一把锁，然后等待另一把锁，就可能形成死锁。

使用 GDB：

```
gdb ./main
```

程序卡住后按：

```
Ctrl + C
```

然后：

```
(gdb) info threads
```

查看所有线程：

```
(gdb) thread apply all bt
```

重点观察：

- 哪个线程停在 `pthread_mutex_lock()`。
    
- 哪个线程正在等待锁。
    
- 两个线程分别处于哪个函数。
    
- 是否存在循环等待。
    

这是一种非常实用的死锁定位方法。

---

# 二十、GDB 与其他调试工具的关系

GDB 并不是唯一的调试工具。

C 语言开发中经常需要组合使用多种工具。

|工具|主要用途|
|---|---|
|GDB|运行时调试、断点、单步、线程检查|
|Valgrind|内存错误检测、内存泄漏分析|
|AddressSanitizer|检测越界、Use-After-Free 等内存问题|
|ThreadSanitizer|检测数据竞争|
|strace|跟踪系统调用|
|ltrace|跟踪动态库调用|
|perf|性能分析|
|objdump|反汇编、查看 ELF 信息|
|readelf|查看 ELF 文件结构|
|nm|查看符号信息|

例如：

GDB：

```
gdb ./main
```

AddressSanitizer：

```
gcc -g -O1 -fsanitize=address -fno-omit-frame-pointer main.c -o main
./main
```

ThreadSanitizer：

```
gcc -g -O1 -fsanitize=thread main.c -o main -pthread
./main
```

注意：ASan 和 TSan 通常需要分别构建，不能简单地把两者同时启用。

---

# 二十一、推荐的 GDB 学习路线

建议按照下面的顺序学习，不要一开始就钻研复杂的底层调试机制。

## 第一阶段：基本调试

掌握：

- 安装 GDB
    
- `-g` 编译参数
    
- `-O0` 编译参数
    
- 启动 GDB
    
- 设置断点
    
- 运行程序
    
- 单步执行
    
- 查看变量
    
- 继续运行
    

目标：能够独立调试一个简单的 C 程序。

## 第二阶段：深入程序内部

掌握：

- 指针调试
    
- 数组调试
    
- 内存查看
    
- 栈帧
    
- 函数调用栈
    
- 查看寄存器
    
- 查看汇编
    

目标：能够理解程序运行时的内存和函数调用过程。

## 第三阶段：复杂程序调试

掌握：

- 条件断点
    
- 监视点
    
- Core Dump
    
- 段错误定位
    
- 调试正在运行的进程
    
- 动态库调试
    

目标：能够独立定位普通 C 项目中的运行时错误。

## 第四阶段：多线程调试

掌握：

- 查看线程
    
- 切换线程
    
- 查看所有线程调用栈
    
- 线程断点
    
- 定位死锁
    
- 分析线程间数据竞争
    

目标：能够调试 pthread 线程池、生产者消费者模型等程序。

---

# 二十二、最重要的命令：建议先记住这些

如果暂时不想记住全部命令，先掌握下面这 15 个：

```
break main              # 设置断点
run                     # 运行
next                    # 单步，不进入函数
step                    # 单步，进入函数
continue                # 继续运行
finish                  # 执行完当前函数
print a                 # 查看变量
set variable a = 10     # 修改变量
watch a                 # 监视变量变化
info locals             # 查看局部变量
info args               # 查看函数参数
bt                      # 查看调用栈
info threads            # 查看所有线程
thread apply all bt     # 查看所有线程调用栈
quit                    # 退出
```

---

# 二十三、总结

学习 GDB，真正需要掌握的不是背诵命令，而是建立一套程序调试思维。

当程序出现问题时，应该能够回答：

1. 程序在哪里出错？
    
2. 出错时执行到了哪一行？
    
3. 当前函数是谁调用的？
    
4. 当前变量是什么值？
    
5. 指针指向什么地址？
    
6. 内存中的内容是什么？
    
7. 哪个线程正在执行？
    
8. 哪个线程正在等待锁？
    
9. 是谁修改了这个变量？
    
10. 程序崩溃时究竟发生了什么？
    

能够回答这些问题，就具备了比较扎实的 C 语言调试能力。

**建议把 GDB 当作日常开发工具，而不是只有程序出错时才使用的工具。**

每写一个 C 程序，都可以尝试用 GDB 走一遍程序的执行过程。这样不仅能学会调试，也能加深对 C 语言、内存、函数调用、线程和操作系统的理解。

---

# 官方参考资料

1. GNU GDB 官方手册：[https://sourceware.org/gdb/current/onlinedocs/gdb.html](https://sourceware.org/gdb/current/onlinedocs/gdb.html)
    
2. GDB 命令参考：[https://sourceware.org/gdb/current/onlinedocs/gdb/Commands.html](https://sourceware.org/gdb/current/onlinedocs/gdb/Commands.html)
    
3. GDB 断点与监视点：[https://sourceware.org/gdb/current/onlinedocs/gdb/Breakpoints.html](https://sourceware.org/gdb/current/onlinedocs/gdb/Breakpoints.html)
    
4. Linux GDB 手册：[https://man7.org/linux/man-pages/man1/gdb.1.html](https://man7.org/linux/man-pages/man1/gdb.1.html)