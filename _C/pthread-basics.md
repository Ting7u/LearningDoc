# C 语言多线程基础梳理

> 承接上一篇《Socket 编程核心概念梳理》，这一篇讲：怎么用多线程处理并发连接、`pthread_create` 的参数含义、`void *` 和 `NULL` 的本质。

---

## 一、为什么需要多线程

### 单线程服务器的问题

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
    handle_client(client_fd);   // 如果这里处理很慢……
    close(client_fd);
}
```

`accept()` 是在循环里**顺序**调用的。如果 `handle_client()` 处理某个客户端耗时很久：

- 主线程会一直卡在这一行，**没空回到循环头去调用下一次 `accept()`**
- 这期间新来的客户端，TCP 握手依然能正常完成（握手是内核自动处理的），但**已完成握手、还没被 `accept()` 取走的连接**只能先存在 `listen(server_fd, N)` 设置的等待队列里
- 队列长度有限，队列满了之后，再来的新连接会被拒绝或超时

**结论**：不是"请求被阻塞"，而是"主线程被某个慢客户端拖住，导致它没空及时 `accept()` 新连接"。

### 多线程的解决思路

```
主线程：只做一件事——循环、快速地 accept()，不参与耗时业务处理
子线程：每接受一个连接就丢给一个新线程，各自独立处理，互不阻塞主线程
```

---

## 二、线程是什么

单线程程序只有**一条执行路径**，所有代码排队执行。

**线程**：让程序同时拥有**多条执行路径**，它们可以"并发"运行，互不等待。

打比方：
- 单线程 = 一个服务员，一次只能招待一桌
- 多线程 = 多个服务员，各自负责一桌，同时忙碌

---

## 三、最简单的线程示例

```c
#include <stdio.h>
#include <unistd.h>     // sleep()
#include <pthread.h>    // 使用线程必须包含

void *say_hello(void *arg)
{
    printf("你好，我是一个新线程！\n");
    return NULL;
}

int main()
{
    pthread_t tid;                                   // 线程的"句柄"/编号
    pthread_create(&tid, NULL, say_hello, NULL);      // 创建新线程
    printf("我是主线程\n");
    sleep(1);   // 等一下，确保子线程有机会执行完
    return 0;
}
```

编译需加 `-lpthread`：

```bash
gcc test.c -o test -lpthread
```

**输出顺序不确定**（可能主线程先打印，也可能子线程先打印）——这正是"并发"的本质：多条执行路径谁先谁后，由操作系统调度决定，不是代码写的顺序决定的。这也是为什么示例里用 `sleep(1)` 强制主线程等一下，否则主线程可能在子线程还没来得及执行时就已经 `return` 了，导致整个进程连带子线程一起结束，你可能根本看不到子线程的输出。

---

## 四、`pthread_create()` 四个参数详解

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                    void *(*start_routine)(void *), void *arg);
```

调用示例：

```c
pthread_create(&tid, NULL, say_hello, NULL);
```

| 参数 | 含义 |
|---|---|
| 参数1 `&tid` | **输出参数**。创建成功后，新线程的编号会被写进这里，之后要操作这个线程（等它结束/分离），都靠这个编号 |
| 参数2 `NULL` | 线程属性（栈大小、调度策略等），新手阶段传 `NULL` 表示"用默认属性" |
| 参数3 `say_hello` | 新线程要执行的函数，告诉系统"这个线程从这个函数开始跑" |
| 参数4 `NULL` | 传给线程函数的参数（数据），不需要传数据就填 `NULL` |

一句话理解：**"新线程编号写到哪（1）？用什么属性（2，默认即可）？执行哪个函数（3）？给这个函数传什么数据（4）？"**

---

## 五、为什么线程函数必须是 `void *func(void *arg)`

`pthread_create` 第三个参数要求的类型是：

```c
void *(*start_routine)(void *)
```

即：**必须接收一个 `void *` 参数，返回一个 `void *`**，格式固定，不能随意改成 `int func(int)` 之类。

**原因**：`pthread_create` 要能创建"任意类型"的线程，执行各种不同参数、不同返回值类型的函数。C 语言函数指针要求固定签名，无法为每种类型单独写一个版本，所以统一用"万能容器" `void *` 来兼容所有情况。

### `void *` 是什么

无类型指针——**只是一个地址**，本身不携带"这块内存里存的是什么类型数据"的信息。类比：一个没贴标签的快递箱，箱子能装任何东西，但外面没写是什么，收件人得自己知道该怎么打开。

### 传参与取参

**传参**（具体类型指针 → `void *`，这个方向是自动、安全的）：

```c
int number = 42;
pthread_create(&tid, NULL, my_func, &number);  // int* 自动转 void*
```

**取参**（`void *` → 具体类型指针，必须手动强转，编译器不检查对不对）：

```c
void *my_func(void *arg)
{
    int value = *(int *)arg;   // 强转回 int*，再解引用取值
    printf("收到的数字: %d\n", value);
    return NULL;
}
```

### 返回值同理

```c
void *my_func(void *arg)
{
    int *result = malloc(sizeof(int));
    *result = 100;
    return result;   // int* 自动转成 void*
}
```

（这个"线程返回值"机制在 socket 服务器场景基本用不上，用到再深入。）

---

## 六、`NULL` 到底是什么

### 本质：一个宏，代表"空指针"

`NULL` 不是关键字，是头文件里用 `#define` 定义出来的宏，常见定义类似：

```c
#define NULL ((void *)0)
```

（具体底层实现因编译器/标准库而异，不需要深究，只需记住它代表"空指针"这个语义。）

### 为什么可以在指针上下文里直接用它

C 语言规定：整数字面量 `0` 可以被赋值给任何指针类型，编译器会把它理解成"空指针"，而非数值 0。所以以下写法语义等价：

```c
int *p1 = NULL;
int *p2 = 0;          // 效果一样，但不推荐这么写
```

### 同一个 `NULL`，不同位置代表不同类型的"空"

```c
pthread_create(&tid, NULL, say_hello, NULL);
//              参数2 ↑              ↑ 参数4
```

- 参数2的 `NULL`：类型是 `const pthread_attr_t *`，意思是"不指定线程属性，用默认"
- 参数4的 `NULL`：类型是 `void *`，意思是"不传数据给线程函数"

两处写法一样，但因为函数签名规定了各自期望的指针类型，`NULL` 会被理解成对应类型的空指针。

### 注意

`NULL` 的"空指针"语义**只在指针上下文里生效**。如果是普通整数变量用到 `0`，就纯粹是数字 0，跟"空指针"无关。

---

## 七、实战：来一个连接开一个线程，最多同时 10 个

### 完整思路

```c
#define MAX_THREADS 10

int thread_count = 0;                                    // 当前活跃线程数（共享变量）
pthread_mutex_t count_lock = PTHREAD_MUTEX_INITIALIZER;  // 保护 thread_count 的锁

void *handle_client(void *arg)
{
    int client_fd = *(int *)arg;
    free(arg);                     // arg 是 malloc 出来的，用完释放

    printf("hello\n");
    close(client_fd);

    pthread_mutex_lock(&count_lock);
    thread_count--;                // 线程结束，计数减一，腾出名额
    pthread_mutex_unlock(&count_lock);

    return NULL;
}

int main()
{
    // ... socket() / bind() / listen() 保持不变 ...

    while (1)
    {
        int client_fd = accept(server_fd, NULL, NULL);

        pthread_mutex_lock(&count_lock);
        if (thread_count >= MAX_THREADS) {
            pthread_mutex_unlock(&count_lock);
            printf("已达最大连接数，拒绝本次连接\n");
            close(client_fd);
            continue;
        }
        thread_count++;
        pthread_mutex_unlock(&count_lock);

        int *fd_ptr = malloc(sizeof(int));   // 单独分配内存，避免多线程共用同一个变量
        *fd_ptr = client_fd;

        pthread_t tid;
        pthread_create(&tid, NULL, handle_client, fd_ptr);
        pthread_detach(tid);                 // 不需要手动 join，线程结束自动回收
    }
    return 0;
}
```

### 关键点解释

**1）为什么 `client_fd` 要单独 `malloc`，不能直接传地址？**

`client_fd` 是循环里的局部变量。如果直接传它的地址给线程，等新线程真正去读这个值时，主线程可能已经进入下一轮循环、把变量值改成了别的连接的 fd——多个线程可能读到错误、甚至相同的值。这是经典的**竞态条件（race condition）**。解决办法：每个连接单独 `malloc` 一块内存，各用各的，互不干扰。

**2）为什么 `thread_count` 要加锁？**

`thread_count` 是**所有线程共享**的变量（每个线程结束都要去减它）。如果不加锁，可能出现"两个线程同时读到 9，都以为可以变成 10，结果实际上错误地变成了 11"这种数据错乱——这也是竞态条件的一种。`pthread_mutex_lock`/`unlock` 保证"同一时刻只允许一个线程操作这个变量"。

**3）为什么用 `pthread_detach` 而不是 `pthread_join`？**

- `pthread_join()` 会**阻塞**调用者，直到目标线程执行完——如果主线程对每个新线程都 `join`，主线程又会被卡住，等于白写了多线程
- `pthread_detach()` 告诉系统："这个线程结束后自己回收资源，不需要任何人等它"，主线程创建完线程可以**立刻**回去 `accept()` 下一个连接

**4）整体流程**

```
主线程死循环 accept()
  → 检查当前线程数是否 < 10
    → 未达上限：计数+1，开新线程处理，主线程立刻继续 accept 下一个
    → 已达上限：直接拒绝这个连接（close），继续循环等下一个
  → 子线程处理完、close 完连接后，计数-1，腾出名额
```

---

## 八、待深入的话题（暂不展开）

- **多个线程共享数据**（不只是简单计数器，而是复杂的共享结构）需要用 `mutex` 等机制保护，避免数据竞争
- **每连接一线程 vs 异步 I/O（epoll/io_uring）**：高并发场景下，线程创建/切换开销会成为瓶颈，生产级服务器（如 Nginx、Node.js）更常用异步事件驱动模型
- **线程返回值**（通过 `pthread_join` 接收 `void *` 类型的返回值）
