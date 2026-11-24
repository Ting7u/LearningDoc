# C 语言 pthread 速查手册

POSIX Threads(pthread)是 Unix/Linux 下 C 语言多线程编程的标准接口。本手册按功能分类整理,每篇都带可直接编译的示例。

## 文档目录

| 文档 | 内容 |
|---|---|
| [01-线程管理.md](01-线程管理.md) | 创建、等待、分离、退出、线程属性、栈大小 |
| [02-互斥锁.md](02-互斥锁.md) | `pthread_mutex_*`、锁类型、trylock/timedlock、死锁 |
| [03-条件变量.md](03-条件变量.md) | `pthread_cond_*`、生产者消费者、超时等待 |
| [04-读写锁-自旋锁-屏障.md](04-读写锁-自旋锁-屏障.md) | `pthread_rwlock_*`、`pthread_spin_*`、`pthread_barrier_*` |
| [05-线程私有数据-once-取消-信号.md](05-线程私有数据-once-取消-信号.md) | TSD、`pthread_once`、取消与清理、信号、CPU 亲和性 |
| [06-综合实战与避坑.md](06-综合实战与避坑.md) | 线程池实例、常见坑、函数总表 |

## 编译方式

```bash
gcc -Wall -Wextra -O2 -pthread demo.c -o demo
```

- 使用 `-pthread`(而不是只写 `-lpthread`),它会同时设置预处理宏和链接选项。
- glibc 2.34+ 已把 pthread 合并进 libc,但仍建议保留 `-pthread`,保证可移植。
- 调试工具:`-fsanitize=thread`(TSan)检测数据竞争;`valgrind --tool=helgrind` 检测锁问题。

## 通用约定

1. **返回值**:几乎所有 pthread 函数成功返回 `0`,失败返回**错误码**(正数 errno 值),**不会设置 `errno`**。要打印错误信息请用 `strerror(rc)`,不能用 `perror`。
2. **头文件**:`#include <pthread.h>`。
3. **`_np` 后缀**:表示 non-portable(GNU 扩展),使用时需要 `#define _GNU_SOURCE`,且不保证在 macOS/BSD 可用。
4. **可选特性**:`pthread_barrier_*`、`pthread_spin_*` 在 POSIX 中是可选的,macOS 不支持。

推荐的错误检查宏:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define CHECK(call)                                                   \
    do {                                                              \
        int _rc = (call);                                             \
        if (_rc != 0) {                                               \
            fprintf(stderr, "%s:%d %s failed: %s\n",                  \
                    __FILE__, __LINE__, #call, strerror(_rc));        \
            exit(EXIT_FAILURE);                                       \
        }                                                             \
    } while (0)
```

后续示例为了简洁,多数省略了错误检查,实际项目中应加上。

## 知识地图

```
pthread
├── 线程生命周期  create / join / detach / exit / cancel / self / equal
├── 同步原语
│   ├── 互斥  mutex / spinlock
│   ├── 读写  rwlock
│   ├── 等待  cond
│   └── 汇合  barrier
├── 线程局部存储  key(TSD) / __thread
├── 一次性初始化  once
├── 属性  attr / mutexattr / condattr / rwlockattr / barrierattr
└── 其它  sigmask / kill / atfork / affinity / sched
```
