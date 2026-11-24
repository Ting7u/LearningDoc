# C 标准库头文件速查表

按功能分类整理，涵盖 C89/C99/C11 标准库常用头文件。每个头文件都标注了缩写全称，方便记忆。

## 输入输出

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `stdio.h` | Standard Input/Output | 标准输入输出：`printf`/`scanf`、`fopen`/`fclose`/`fread`/`fwrite`、`fprintf`/`fscanf`、`getchar`/`putchar`、`FILE`、`EOF`、`fseek`/`ftell` |

## 内存与通用工具

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `stdlib.h` | Standard Library | 动态内存（`malloc`/`calloc`/`realloc`/`free`）、程序控制（`exit`/`abort`/`atexit`）、字符串转数字（`atoi`/`atof`/`strtol`/`strtod`）、随机数（`rand`/`srand`）、排序查找（`qsort`/`bsearch`）、`abs`/`div` |

## 字符串与内存操作

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `string.h` | String | 字符串处理（`strlen`/`strcpy`/`strncpy`/`strcat`/`strcmp`/`strstr`/`strtok`）、内存操作（`memcpy`/`memmove`/`memset`/`memcmp`）、错误信息（`strerror`） |

## 数学运算

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `math.h` | Mathematics | 三角函数（`sin`/`cos`/`tan`）、幂与根（`pow`/`sqrt`/`cbrt`）、对数指数（`log`/`log2`/`exp`）、取整（`floor`/`ceil`/`round`）、`fabs`、`fmod`（链接需加 `-lm`） |
| `complex.h` | Complex Number | C99 复数运算：`complex`、`creal`、`cimag`、`cabs` |

## 字符处理

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `ctype.h` | Character Type | 字符分类（`isdigit`/`isalpha`/`isspace`/`isupper`）、转换（`toupper`/`tolower`） |
| `wctype.h` | Wide Character Type | 宽字符版本的 `ctype.h`（`iswdigit` 等） |

## 类型与常量

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `stdbool.h` | Standard Boolean | `bool`/`true`/`false`（C99 引入，本质是宏） |
| `stdint.h` | Standard Integer | 固定宽度整数类型：`int8_t`/`uint32_t`/`int64_t`，以及 `INT32_MAX` 等边界宏 |
| `inttypes.h` | Integer Types | 配合 `stdint.h` 的格式化宏（如 `PRId64`）及扩展的整数转换函数 |
| `limits.h` | Limits | 整数类型的边界值：`INT_MAX`/`CHAR_BIT`/`LONG_MIN` 等 |
| `float.h` | Floating Point | 浮点类型的边界与精度：`FLT_EPSILON`/`DBL_MAX` 等 |
| `stddef.h` | Standard Definitions | 基础类型定义：`size_t`/`ptrdiff_t`/`NULL`/`offsetof` |

## 可变参数与函数指针

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `stdarg.h` | Standard Argument | 可变参数函数支持：`va_list`/`va_start`/`va_arg`/`va_end`（类似 `printf` 的实现方式） |

## 时间与日期

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `time.h` | Time | `time()`/`clock()`、`struct tm`、`strftime`/`localtime`/`gmtime`、`difftime` |

## 错误处理与调试

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `errno.h` | Error Number | 全局错误码变量 `errno`，以及 `EINVAL`/`ENOMEM` 等宏 |
| `assert.h` | Assert | `assert()` 断言宏，调试时检查条件是否成立 |

## 控制流与信号

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `setjmp.h` | Set Jump | 非局部跳转：`setjmp`/`longjmp`（模拟异常处理，较少在日常代码中使用） |
| `signal.h` | Signal | 信号处理：`signal()`、`raise()`，处理 `SIGINT`/`SIGSEGV` 等 |

## 本地化

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `locale.h` | Locale | 本地化设置：`setlocale()`，影响数字格式、日期格式等 |

## 宽字符与多字节

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `wchar.h` | Wide Character | 宽字符字符串处理：`wprintf`/`wcslen`/`wcscpy` |
| `uchar.h` | Unicode Character | C11 引入，Unicode 字符类型：`char16_t`/`char32_t` |

## 并发（C11 引入）

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `threads.h` | Threads | C11 线程支持：`thrd_create`/`mtx_lock`（实际项目中更常用 POSIX 的 `pthread.h`） |
| `stdatomic.h` | Standard Atomic | C11 原子操作类型与函数 |

## 泛型编程（C11 引入）

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `tgmath.h` | Type-Generic Math | 类型泛型数学宏，自动根据参数类型调用 `math.h` 或 `complex.h` 中对应函数 |

## 对齐（C11 引入）

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `stdalign.h` | Standard Align | 内存对齐相关：`alignas`/`alignof` |

## 其他

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `iso646.h` | ISO 646（字符集标准编号） | 提供 `and`/`or`/`not` 等作为 `&&`/`\|\|`/`!` 的替代拼写（很少用） |
| `stdnoreturn.h` | Standard No-Return | C11 引入的 `noreturn` 宏，标记不会返回的函数 |

---

## 高频使用（日常写代码最常打交道）

```
stdio.h    stdlib.h    string.h    math.h    ctype.h
```

## 补充：POSIX / 系统相关（不属于 C 标准库，Linux/Unix 开发常用）

### 进程与系统调用

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `unistd.h` | Unix Standard | 最常用的 POSIX 头文件：`read`/`write`/`close`、`fork`/`exec`系列、`sleep`/`usleep`、`getpid`/`getppid`、`chdir`、`access` |
| `sys/types.h` | System Types | POSIX 基础数据类型：`pid_t`/`uid_t`/`size_t`/`off_t` 等 |
| `sys/wait.h` | System Wait | 进程等待：`wait`/`waitpid`，配合`fork`使用 |
| `spawn.h` | Spawn（POSIX Spawn） | `posix_spawn`，创建子进程（比`fork+exec`更轻量的接口） |

### 文件与I/O

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `fcntl.h` | File Control | 文件控制：`open`、`fcntl`（设置文件描述符属性）、`O_RDONLY`等标志位 |
| `sys/stat.h` | System Status（文件状态） | 文件状态信息：`stat`/`fstat`/`lstat`、`struct stat`、`chmod`、`mkdir` |
| `dirent.h` | Directory Entry | 目录遍历：`opendir`/`readdir`/`closedir`、`struct dirent` |
| `poll.h` | Poll | I/O 多路复用：`poll()` |
| `sys/select.h` | System Select | I/O 多路复用：`select()`、`fd_set` |
| `sys/mman.h` | System Memory Map | 内存映射：`mmap`/`munmap`，共享内存 |

### 多线程与同步

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `pthread.h` | POSIX Thread | POSIX 线程：`pthread_create`/`pthread_join`、互斥锁`pthread_mutex_*`、条件变量`pthread_cond_*` |
| `semaphore.h` | Semaphore | 信号量：`sem_init`/`sem_wait`/`sem_post` |

### 网络编程

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `sys/socket.h` | System Socket | Socket 核心：`socket`/`bind`/`listen`/`accept`/`connect`/`send`/`recv` |
| `netinet/in.h` | Network Internet | 网络地址结构：`struct sockaddr_in`、端口/地址转换相关宏 |
| `arpa/inet.h` | ARPA Internet（网络地址转换） | 地址转换函数：`inet_addr`/`inet_ntoa`/`inet_pton`/`inet_ntop` |
| `netdb.h` | Network Database | 域名解析：`gethostbyname`、`getaddrinfo` |
| `sys/un.h` | System Unix（Unix域套接字） | Unix 域套接字（本机进程间通信）：`struct sockaddr_un` |

### 进程间通信（IPC）

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `sys/ipc.h` | System Inter-Process Communication | System V IPC 通用定义，配合下面三个使用 |
| `sys/shm.h` | System Shared Memory | 共享内存：`shmget`/`shmat`/`shmdt` |
| `sys/msg.h` | System Message | 消息队列：`msgget`/`msgsnd`/`msgrcv` |
| `sys/sem.h` | System Semaphore | System V 信号量：`semget`/`semop` |
| `mqueue.h` | Message Queue | POSIX 消息队列（比 System V 版本更现代）：`mq_open`/`mq_send` |

### 终端与信号

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `termios.h` | Terminal I/O Settings | 终端控制：设置终端为原始模式、波特率等（串口编程常用） |
| `sys/ioctl.h` | System I/O Control | 设备控制：`ioctl()`，操作底层设备参数 |

### 时间

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `sys/time.h` | System Time | 更高精度的时间：`gettimeofday`、`struct timeval` |

### 动态库

| 头文件 | 全称 | 主要内容 |
|---|---|---|
| `dlfcn.h` | Dynamic Linking Functions | 动态链接库操作：`dlopen`/`dlsym`/`dlclose` |
