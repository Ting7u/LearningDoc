# C 语言在 Linux 下的非标准输入输出（I/O）补充

> 目标：补充《C 语言输入输出分类总结》中没有涉及的内容------**不属于 ISO C 标准、但在 Linux 下非常常用的 I/O 接口**。
>
> 这些接口来自几个不同的"出身"：POSIX 标准、GNU C 库（glibc）扩展、BSD 传统、Linux 内核专有。它们都**不是标准 C**，但在 Linux 开发中几乎无处不在。

---

## 0. 先搞清楚：什么叫"非标准"？

"非标准"在这里指 **ISO C 标准之外**。进一步细分，大致有四种来源：

| 来源标记            | 含义                                               | 代表接口                                                     | 可移植性                         |
| --------------- | ------------------------------------------------ | -------------------------------------------------------- | ---------------------------- |
| **POSIX**       | 另一份独立的标准（IEEE/Open Group），Linux、macOS、BSD 等基本都支持 | `open` / `read` / `write` / `getline` / `popen` / `poll` | 在类 Unix 系统间较好；Windows 原生不支持  |
| **GNU (glibc)** | GNU C 库提供的扩展                                     | `asprintf`、`fgets_unlocked`、`%m` 格式符                     | 主要是 Linux + glibc；部分在 BSD 也有 |
| **BSD**         | 源自 BSD 系统，被 Linux 采纳                             | `err` / `warn`、`setlinebuf`                              | 类 Unix 系统间较好                 |
| **Linux 专有**    | 只有 Linux 内核提供                                    | `epoll`、`splice`、`io_uring`、`inotify`                    | 仅 Linux                      |

后文每个接口都会标注来源。

### 0.1 整体分层：标准 C I/O 与 Linux I/O 的关系

```text
你的程序
  │
  ├─ 标准 C stdio：  printf / fgets / fread / fopen ...
  │      操作对象：FILE *          有用户态缓冲区
  │          │ 内部最终调用 ↓
  │
  ├─ POSIX/Linux 系统调用：  open / read / write / close ...
  │      操作对象：int 文件描述符   没有 stdio 缓冲区
  │          │ 陷入内核 ↓
  │
  └─ Linux 内核： VFS → 页缓存(page cache) → 文件系统 → 磁盘 / 网卡 / 终端 / 管道
```

最重要的理解：

-   标准 C 的 `FILE *` 是对**文件描述符（file descriptor, fd）** 的一层带缓冲的封装。
-   文件描述符是一个小的非负整数（`int`），由内核分配，代表"一个打开的 I/O 对象"。
-   在 Linux 中，**"一切皆文件"**：普通文件、目录、终端、管道、套接字、设备，都可以通过文件描述符用 `read` / `write` 访问。

三个预先打开的标准流对应三个固定的文件描述符：

| `FILE *` | 文件描述符 | 宏（`<unistd.h>`） |
| -------- | ----- | --------------- |
| `stdin`  | 0     | `STDIN_FILENO`  |
| `stdout` | 1     | `STDOUT_FILENO` |
| `stderr` | 2     | `STDERR_FILENO` |

### 0.2 编译时的"特性测试宏"

很多非标准接口，需要声明宏之后才能使用。**这是初学者最常遇到的坑**：

```c
#define _GNU_SOURCE          /* 启用 GNU 和 Linux 扩展（包含 POSIX） */
/* 或者只要 POSIX： */
/* #define _POSIX_C_SOURCE 200809L */

#include <stdio.h>
```

注意：

-   宏必须写在所有 `#include` 之前。
-   用 `gcc` 默认的 `-std=gnu17` 时，大多数扩展已经可见；但如果使用严格模式 `-std=c11` / `-std=c17`，`getline`、`fileno`、`fdopen` 等 POSIX 函数会"消失"（编译警告：隐式声明），此时需要自己定义 `_POSIX_C_SOURCE` 或 `_GNU_SOURCE`。
-   在 32 位系统上处理大于 2 GiB 的文件，编译时加 `-D_FILE_OFFSET_BITS=64`，让 `off_t` 变成 64 位。64 位 Linux 上 `off_t` 本来就是 64 位。

查看手册：`man 7 feature_test_macros`。

---

## 1. 文件描述符 I/O：open / read / write / close

这是 Linux I/O 的**最底层常用接口**，属于 **POSIX**。

```c
#include <fcntl.h>     /* open, O_* 标志 */
#include <unistd.h>    /* read, write, close, lseek */
```

### 1.1 `open`：打开文件，得到文件描述符

```c
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

返回值：成功返回最小的未使用的文件描述符（≥ 0）；失败返回 `-1` 并设置 `errno`。

访问模式（三选一）：

| 标志         | 含义  |     |
| ---------- | --- | --- |
| `O_RDONLY` | 只读  |     |
| `O_WRONLY` | 只写  |     |
| `O_RDWR`   | 读写  |     |

常用附加标志（用 `|` 组合）：

| 标志                   | 含义                                   | 来源           |
| -------------------- | ------------------------------------ | ------------ |
| `O_CREAT`            | 不存在则创建，**此时必须提供第三个参数 `mode`**        | POSIX        |
| `O_EXCL`             | 与 `O_CREAT` 合用：文件已存在则失败（原子地"只创建新文件"） | POSIX        |
| `O_TRUNC`            | 打开时清空文件                              | POSIX        |
| `O_APPEND`           | 每次写入都原子地追加到末尾                        | POSIX        |
| `O_NONBLOCK`         | 非阻塞模式                                | POSIX        |
| `O_SYNC` / `O_DSYNC` | 每次写入等数据（及元数据）真正落盘才返回                 | POSIX        |
| `O_CLOEXEC`          | `exec` 时自动关闭该描述符，避免泄漏给子进程            | POSIX.1-2008 |
| `O_DIRECTORY`        | 路径必须是目录                              | POSIX.1-2008 |
| `O_NOFOLLOW`         | 路径最后一段是符号链接则失败                       | POSIX.1-2008 |
| `O_DIRECT`           | 绕过页缓存的直接 I/O（对缓冲区地址、长度、偏移有对齐要求）      | Linux 专有     |
| `O_TMPFILE`          | 创建没有名字的临时文件                          | Linux 专有     |
| `O_PATH`             | 只取得路径句柄，不能读写                         | Linux 专有     |
| `O_NOATIME`          | 读取时不更新访问时间                           | Linux 专有     |

`mode` 是新建文件的权限（如 `0644`），实际权限还会被进程的 `umask` 屏蔽。

与标准 C `fopen` 模式的对应关系：

| `fopen` 模式 | 大致等价的 `open` 标志                   |
| ---------- | --------------------------------- |
| `"r"`      | `O_RDONLY`                        |
| `"w"`      | `O_WRONLY \| O_CREAT \| O_TRUNC`  |
| `"a"`      | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `"r+"`     | `O_RDWR`                          |
| `"w+"`     | `O_RDWR \| O_CREAT \| O_TRUNC`    |
| `"a+"`     | `O_RDWR \| O_CREAT \| O_APPEND`   |

### 1.2 `read` 与 `write`

`ssize_t`: 有符号整数，用来装返回值。有符号是因为失败要返回 `-1`
`but`: 缓冲区的地址：`read` 把数据放进它，`write` 从它取出数据
`count`: 想读/写的最大字节数

**重点：** 返回值小于 count 不一定是错误，叫"短读/短写"。比如文件剩余内容不够 count 字节，read 就只会返回剩下的部分

**偏移量会自动移动:** 每个打开的文件有一个"当前读写位置"（偏移量）。每次 `read`/`write` 成功后，偏移量会自动往后移动实际传输的字节数。所以连续调用 `read`，读到的是文件里依次往后的内容。（想手动改位置要用 `lseek`。）

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

`read` 的返回值

| 返回值   | 含义                     |
| ----- | ---------------------- |
| `> 0` | 实际读到的字节数（可能小于 `count`） |
| `0`   | 到达文件末尾（EOF）            |
| `-1`  | 出错，查看 `errno`          |

`write` 的返回值：

| 返回值    | 含义                         |
| ------ | -------------------------- |
| `>= 0` | 实际写入的字节数（**可能小于 `count`**） |
| `-1`   | 出错，查看 `errno`              |

**关键点：`read` / `write` 都可能"短读"/"短写"。** 尤其是读写管道、套接字、终端，或者被信号中断时。可靠的代码必须循环处理，并在 `errno == EINTR` 时重试。

`write` 的可靠封装：

```c
#include <errno.h>
#include <unistd.h>

/* 写完全部 count 个字节；成功返回 0，失败返回 -1 */
int write_all(int fd, const void *buf, size_t count)
{
    const char *p = buf;

    while (count > 0) {
        ssize_t n = write(fd, p, count);
        if (n < 0) {
            if (errno == EINTR)
                continue;        /* 被信号打断，重试 */
            return -1;
        }
        p     += n;
        count -= (size_t)n;
    }
    return 0;
}
```

### 1.3 `close` 与 `lseek`

`offset`: 偏移量，单位是字节，可以为负数
`whencce`: 偏移量的"起点"，决定 `offset` 相对于哪里

***`whence` 的三个取值**：*
- SEEK_SET  文件开头
- SEEK_CUR 当前位置
- SEEK_END 文件末尾

```c
int close(int fd);
off_t lseek(int fd, off_t offset, int whence);
```

-   `close`：关闭文件描述符。**检查返回值**，因为某些文件系统（如 NFS）的写入错误可能在 `close` 时才报告。
-   `lseek`：移动文件读写位置，`whence` 取 `SEEK_SET` / `SEEK_CUR` / `SEEK_END`（这几个宏标准 C 也有）。成功返回新位置，失败返回 `(off_t)-1`。Linux 还提供 `SEEK_DATA` / `SEEK_HOLE` 用来遍历稀疏文件。
-   `lseek` 对管道、套接字、终端无效（`errno == ESPIPE`）。

**为什么要检查返回值**：很多人觉得 `close` 不会失败，但原文说得很清楚：某些文件系统（比如 NFS 网络文件系统）会**延迟写入**。你调用 `write` 时看起来成功了，数据其实还在缓冲里，真正落盘或发给服务器时才发现出错，这个错误就会在 `close` 时报告。所以要写成：

```c
if (close(fd) == -1) {
    perror("close");
}
```

另外有一点要注意：`close` 失败后也不要再对同一个 `fd` 重试 `close`。Linux 上即使返回错误，描述符通常也已经被释放了，再关一次可能误关掉别处刚好复用了这个数字的文件。这一点各系统行为不完全一致

*常见用法：*

```c
lseek(fd, 0, SEEK_SET);    // 回到文件开头（重新读）
lseek(fd, 0, SEEK_END);    // 跳到文件末尾
lseek(fd, 10, SEEK_CUR);   // 从当前位置往后跳 10 字节
lseek(fd, -5, SEEK_END);   // 从末尾往前 5 字节的位置
off_t pos = lseek(fd, 0, SEEK_CUR);   // 不移动，只查询当前位置
```

*还有个常用技巧，获取文件大小：*

```c
// 跳到末尾，返回值就是文件总字节数
off_t size = lseek(fd, 0, SEEK_END);
```

**两个补充点:**

1. 可以跳到文件末尾之后（空洞文件）
`lseek` 可以把偏移量设到超过文件末尾的地方，这时如果再 `write`，中间没写过的部分叫"**空洞**"（hole），读出来全是 `0` 字节，而且在很多文件系统上并不真正占用磁盘空间。这样的文件叫**稀疏文件**

2. SEEK_DATA / SEEK_HOLE
这是 Linux 特有的两个选项（标准 C 里没有），专门用来遍历稀疏文件：

- `SEEK_DATA`：跳到下一段**有真实数据**的位置
- `SEEK_HOLE`：跳到下一个**空洞**的位置

用它们可以高效地只读有数据的部分，跳过空洞。初学阶段了解一下就行

*小例子：读完后回到开头再读一遍*

```c
char buf[128];
ssize_t n;

// 第一遍
while ((n = read(fd, buf, sizeof(buf))) > 0) {
    write(1, buf, n);
}

// 回到开头
if (lseek(fd, 0, SEEK_SET) == (off_t)-1) {
    perror("lseek");
}

// 第二遍
while ((n = read(fd, buf, sizeof(buf))) > 0) {
    write(1, buf, n);
}
```

### 1.4 完整例子：用系统调用复制文件

```c
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

static int write_all(int fd, const void *buf, size_t count)
{
    const char *p = buf;
    while (count > 0) {
        ssize_t n = write(fd, p, count);
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        p += n;
        count -= (size_t)n;
    }
    return 0;
}

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "用法：%s <源文件> <目标文件>\n", argv[0]);
        return 1;
    }

    int in = open(argv[1], O_RDONLY);
    if (in < 0) {
        perror("open 源文件");
        return 1;
    }

    int out = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (out < 0) {
        perror("open 目标文件");
        close(in);
        return 1;
    }

    char buf[64 * 1024];
    ssize_t n;
    int status = 0;

    while ((n = read(in, buf, sizeof buf)) != 0) {
        if (n < 0) {
            if (errno == EINTR) continue;
            perror("read");
            status = 1;
            break;
        }
        if (write_all(out, buf, (size_t)n) < 0) {
            perror("write");
            status = 1;
            break;
        }
    }

    if (close(out) < 0) {
        perror("close 目标文件");
        status = 1;
    }
    close(in);
    return status;
}
```

### 1.5 `stdio` 与 文件描述符 I/O 的对比

| 对比项 | 标准 C `stdio` | 文件描述符 I/O |
|---|---|---|
| 操作对象 | `FILE *` | `int` |
| 用户态缓冲 | 有（全缓冲 / 行缓冲 / 无缓冲） | 无，每次调用都进入内核 |
| 格式化 | 有（`printf` / `scanf`） | 无，只有字节 |
| 可移植性 | 所有 C 平台 | 类 Unix（POSIX） |
| 能否控制 `O_DIRECT`、非阻塞等 | 基本不能 | 能 |
| 典型用途 | 文本、格式化、一般文件读写 | 精细控制、网络、管道、设备、高性能 I/O |

---

## 2. `FILE *` 与文件描述符互相转换

两种世界经常需要打通。

### 2.1 `fileno`：`FILE *` → `int`（POSIX）

```c
int fileno(FILE *stream);
```

```c
int fd = fileno(stdout);    /* 得到 1 */
```

典型用途：对已经用 `fopen` 打开的文件调用 `fstat`、`fsync`、`ftruncate`、`fcntl` 等只接受文件描述符的函数。

### 2.2 `fdopen`：`int` → `FILE *`（POSIX）

```c
FILE *fdopen(int fd, const char *mode);
```

```c
int fd = open("data.txt", O_RDONLY);
FILE *fp = fdopen(fd, "r");   /* 之后就可以用 fgets / fscanf 了 */
if (fp == NULL) { perror("fdopen"); close(fd); return 1; }

/* ... */
fclose(fp);                   /* 同时会关闭 fd，不要再 close(fd) */
```

**注意：** `fdopen` 成功后，`fd` 的所有权转给了 `FILE *`，应当只调用 `fclose(fp)`。对 `socket` 得到的描述符使用 `fdopen`，是把网络连接当作文本流来读写的常见做法。

### 2.3 `dprintf`：直接向文件描述符做格式化输出（POSIX.1-2008）

```c
int dprintf(int fd, const char *format, ...);
```

```c
dprintf(STDERR_FILENO, "错误码 = %d\n", 42);
dprintf(sockfd, "GET / HTTP/1.0\r\n\r\n");
```

它没有 `FILE *` 缓冲，每次调用直接写入。相关的还有 `vdprintf`。

### 2.4 为什么要小心混用？

```c
printf("hello");           /* 进入 stdout 的用户态缓冲区，还没有写出 */
write(STDOUT_FILENO, "world\n", 6);   /* 直接写出 */
```

输出可能是 `worldhello`。原因是 `printf` 的内容还在缓冲区里。解决办法：

-   混用之前调用 `fflush(stdout)`。
-   或者对同一个目标统一只使用一种接口。

同理，`fork` 前如果缓冲区有未写出的数据，子进程会继承这份缓冲区，导致同一段内容输出两遍。`fork` 前建议 `fflush(NULL)`（刷新所有输出流，这是标准 C 行为）。

另外：`_exit()`（POSIX 系统调用）**不会**刷新 stdio 缓冲区，而 `exit()` 和从 `main` 返回会。

---

## 3. POSIX / GNU 对 stdio 的扩展函数

这些函数仍然使用 `FILE *`，用起来和标准函数很像，但标准 C 里没有。

### 3.1 `getline` / `getdelim`：自动扩容的读行（POSIX.1-2008）

标准 `fgets` 要求自己提前指定缓冲区大小，行太长就会被截断。`getline` 会自动 `malloc` / `realloc`，是 Linux 下读取任意长度一行的**首选**方式。

```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
ssize_t getdelim(char **lineptr, size_t *n, int delim, FILE *stream);
```

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char *line = NULL;     /* 初始必须为 NULL（或指向 malloc 的内存） */
    size_t cap = 0;        /* 缓冲区当前容量，初始 0 */
    ssize_t len;

    while ((len = getline(&line, &cap, stdin)) != -1) {
        if (len > 0 && line[len - 1] == '\n')
            line[len - 1] = '\0';          /* 去掉换行符 */
        printf("读到 %zd 字节：[%s]\n", len, line);
    }

    free(line);            /* 必须自己释放 */
    return 0;
}
```

要点：

-   返回读到的字符数（包含换行符，不含结尾 `\0`）；遇到 EOF 且没有读到任何字符，或出错时返回 `-1`。
-   缓冲区由库管理，**循环中可以复用同一个 `line`**，最后只 `free` 一次。
-   `getdelim` 可以指定任意分隔符，比如用 `'\0'` 读取 `find -print0` 的输出。

### 3.2 `asprintf` / `vasprintf`：自动分配内存的 `sprintf`（GNU，BSD 也有）

```c
int asprintf(char **strp, const char *format, ...);
```

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char *msg = NULL;

    if (asprintf(&msg, "user=%s, id=%d", "tom", 1001) < 0) {
        perror("asprintf");
        return 1;
    }

    puts(msg);
    free(msg);             /* 必须释放 */
    return 0;
}
```

它根据需要的长度自动分配内存，**不会缓冲区溢出**，也不用像 `snprintf` 那样先算长度再分配。失败返回 `-1`，此时 `*strp` 的值未定义，不要使用。

### 3.3 `fmemopen` 与 `open_memstream`：把内存当作文件（POSIX.1-2008）

这两个函数返回 `FILE *`，但读写的是内存，而不是磁盘。

**`fmemopen`：把已有的内存块当作文件来读/写**

```c
FILE *fmemopen(void *buf, size_t size, const char *mode);
```

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <string.h>

int main(void)
{
    const char *text = "line1\nline2\nline3\n";
    FILE *fp = fmemopen((void *)text, strlen(text), "r");
    if (fp == NULL) { perror("fmemopen"); return 1; }

    char line[64];
    while (fgets(line, sizeof line, fp) != NULL)
        fputs(line, stdout);

    fclose(fp);
    return 0;
}
```

**`open_memstream`：向一块自动增长的内存写入**

```c
FILE *open_memstream(char **ptr, size_t *sizeloc);
```

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char *buf = NULL;
    size_t size = 0;

    FILE *mem = open_memstream(&buf, &size);
    if (mem == NULL) { perror("open_memstream"); return 1; }

    fprintf(mem, "id=%d\n", 42);
    fprintf(mem, "name=%s\n", "tom");

    fclose(mem);           /* 或 fflush(mem)；之后 buf 和 size 才有效 */

    printf("共 %zu 字节：\n%s", size, buf);
    free(buf);             /* 必须自己释放 */
    return 0;
}
```

意义：很多接收 `FILE *` 的函数（比如某个序列化函数 `dump(FILE *out)`），借助它们可以直接把输出收集到字符串里，不用改函数。

### 3.4 `popen` / `pclose`：通过管道执行 shell 命令（POSIX）

```c
FILE *popen(const char *command, const char *type);   /* type: "r" 或 "w" */
int pclose(FILE *stream);
```

```c
#include <stdio.h>

int main(void)
{
    FILE *fp = popen("ls -l", "r");     /* 读取命令的标准输出 */
    if (fp == NULL) { perror("popen"); return 1; }

    char line[256];
    while (fgets(line, sizeof line, fp) != NULL)
        fputs(line, stdout);

    int status = pclose(fp);            /* 必须用 pclose，不能用 fclose */
    printf("命令退出状态原始值：%d\n", status);
    return 0;
}
```

要点：

-   `"r"`：读取命令的标准输出；`"w"`：向命令的标准输入写数据。
-   命令通过 `/bin/sh -c` 执行，**不要把未经检查的用户输入拼进命令字符串**，否则有命令注入风险。
-   `pclose` 返回的是 `wait` 风格的状态值，需用 `WIFEXITED` / `WEXITSTATUS`（`<sys/wait.h>`）解析。
-   `popen` 只是单向管道；需要双向或更精细的控制，用 `pipe` + `fork` + `exec`（见第 5 节）。

### 3.5 `fseeko` / `ftello`：使用 `off_t` 的文件定位（POSIX）

标准 `fseek` / `ftell` 使用 `long`，在 32 位平台上限制为 2 GiB。POSIX 提供了使用 `off_t` 的版本：

```c
int   fseeko(FILE *stream, off_t offset, int whence);
off_t ftello(FILE *stream);
```

配合 `-D_FILE_OFFSET_BITS=64` 可以处理大文件。在 64 位 Linux 上，`long` 本身就是 64 位，差别不大，但 `fseeko` / `ftello` 仍然是推荐写法。

### 3.6 非加锁版本：`getc_unlocked` 等

标准 stdio 函数为了线程安全，每次调用都要加锁/解锁一次。在**单线程**或者**已手动加锁**的情况下，这是浪费。

| 函数 | 说明 | 来源 |
|---|---|---|
| `getc_unlocked` / `putc_unlocked` | 不加锁的 `getc` / `putc` | POSIX |
| `getchar_unlocked` / `putchar_unlocked` | 不加锁的 `getchar` / `putchar` | POSIX |
| `fgets_unlocked` / `fputs_unlocked` / `fread_unlocked` / `fwrite_unlocked` / `fflush_unlocked` 等 | 对应标准函数的无锁版本 | GNU |
| `flockfile` / `funlockfile` / `ftrylockfile` | 手动给 `FILE *` 加锁/解锁/尝试加锁 | POSIX |

```c
#include <stdio.h>

int main(void)
{
    int c;
    long count = 0;

    flockfile(stdin);                       /* 手动加锁一次 */
    while ((c = getc_unlocked(stdin)) != EOF) {
        if (c == '\n')
            count++;
    }
    funlockfile(stdin);

    printf("行数：%ld\n", count);
    return 0;
}
```

典型用途：算法竞赛、大量数据快速读入。但**多线程同时读写同一个流时，必须配合 `flockfile`，否则会出现数据竞争。**

### 3.7 缓冲区控制的扩展

标准 C 已有 `setvbuf` / `setbuf`。在此之外：

| 函数 | 说明 | 来源 |
|---|---|---|
| `setlinebuf(fp)` | 设置为行缓冲（等价于 `setvbuf(fp, NULL, _IOLBF, 0)`） | BSD / glibc |
| `setbuffer(fp, buf, size)` | 指定缓冲区和大小 | BSD / glibc |
| `__fpurge(fp)` | 丢弃流的缓冲区内容（`<stdio_ext.h>`） | glibc / Solaris |
| `__fpending(fp)` | 查询输出缓冲区中待写出的字节数（`<stdio_ext.h>`） | glibc / Solaris |
| `__fbufsize(fp)` | 查询缓冲区大小（`<stdio_ext.h>`） | glibc / Solaris |
| `__flbf(fp)` | 判断是否行缓冲（`<stdio_ext.h>`） | glibc / Solaris |

常见的缓冲默认行为（glibc）：

-   `stderr`：无缓冲。
-   `stdout`：连接终端时行缓冲；重定向到文件或管道时全缓冲。
-   `stdin`：连接终端时行缓冲。

**这就是为什么 `./a.out > log.txt` 和 `./a.out | less` 时，`printf` 的输出常常"延迟出现"或与 `stderr` 的顺序错乱。**

关于"清空输入缓冲区"：

```c
/* 标准、可移植的写法：读掉当前行剩余内容 */
int c;
while ((c = getchar()) != '\n' && c != EOF)
    ;
```

`__fpurge(stdin)` 在 glibc 下可用，但依赖实现，**能用上面的标准写法时不要用它**。

### 3.8 `fopen` 的扩展模式字符（glibc）

在模式字符串中追加这些字符：

| 追加 | 含义 |
|---|---|
| `e` | 等价于给底层 `open` 加 `O_CLOEXEC`（例：`"re"`） |
| `x` | 与 `"w"` 合用，文件已存在则失败（类似 `O_EXCL`；C11 已将它纳入标准） |
| `m` | 读取时尝试用 `mmap`（例：`"rm"`） |
| `,ccs=编码名` | 指定字符编码转换（例：`"r,ccs=UTF-8"`） |

### 3.9 其他实用的 stdio 扩展与相关函数

| 函数 | 作用 | 来源 |
|---|---|---|
| `fcloseall()` | 关闭所有打开的流 | GNU |
| `getw` / `putw` | 读写一个 `int`（已过时，不建议使用） | 传统 Unix |
| `tmpfile`（标准）/ `mkstemp` | `mkstemp`：按模板（末尾 `XXXXXX`）安全创建临时文件，返回 `fd` | POSIX |
| `mkdtemp` | 按模板创建临时目录 | POSIX |

`mkstemp` 示例：

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(void)
{
    char path[] = "/tmp/myapp-XXXXXX";    /* 必须是可修改的数组，不能是字符串字面量指针 */
    int fd = mkstemp(path);               /* path 被改写为真实文件名 */
    if (fd < 0) { perror("mkstemp"); return 1; }

    printf("临时文件：%s\n", path);
    write(fd, "hello\n", 6);

    close(fd);
    unlink(path);                         /* 用完删除 */
    return 0;
}
```

---

## 4. `printf` / `scanf` 的 POSIX / GNU 格式扩展

### 4.1 `%m`（glibc）：直接输出 `strerror(errno)`

```c
#include <errno.h>
#include <stdio.h>

int main(void)
{
    errno = ENOENT;
    printf("操作失败：%m\n");      /* 输出：操作失败：No such file or directory */
    return 0;
}
```

注意：`%m` 不消耗任何参数。这是 glibc 的扩展，别的 C 库不一定支持。

### 4.2 `%n$`（POSIX）：按位置指定参数

```c
printf("%2$s %1$s\n", "world", "hello");     /* 输出：hello world */
```

适用于国际化（不同语言的语序不同），`gettext` 就依赖这个特性。一条格式串中要么全用位置参数，要么全不用，不能混用。

### 4.3 `'` 标志（POSIX）：按本地化规则加千位分隔符

```c
#include <locale.h>
#include <stdio.h>

int main(void)
{
    setlocale(LC_ALL, "");               /* 采用环境的 locale */
    printf("%'d\n", 1234567);            /* 例如 en_US 下输出 1,234,567 */
    return 0;
}
```

效果取决于系统是否安装了对应 locale。

### 4.4 `scanf` 的 `m` 修饰符（POSIX.1-2008）：自动分配内存

标准 `scanf("%s")` 必须提前准备好足够大的数组，容易溢出。POSIX 提供了 `%ms`，由库自动分配：

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    char *word = NULL;

    if (scanf("%ms", &word) == 1) {      /* 参数是 char ** */
        printf("读到：%s\n", word);
        free(word);                      /* 必须释放 */
    }
    return 0;
}
```

---

## 5. 文件描述符的复制、管道与控制

### 5.1 `dup` / `dup2` / `dup3`：复制文件描述符（POSIX / dup3 为 Linux）

```c
int dup(int oldfd);                 /* 返回最小的可用新描述符，指向同一个打开文件 */
int dup2(int oldfd, int newfd);     /* 让 newfd 指向与 oldfd 相同的对象（必要时先关闭 newfd） */
int dup3(int oldfd, int newfd, int flags);   /* Linux：可附加 O_CLOEXEC */
```

最经典的用途：**输出重定向**。

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    fflush(stdout);                          /* 重定向前先清空缓冲区 */
    int saved = dup(STDOUT_FILENO);          /* 备份原来的标准输出 */

    dup2(fd, STDOUT_FILENO);                 /* 现在 fd 1 指向 out.txt */
    close(fd);

    printf("这一行会进入 out.txt\n");
    fflush(stdout);                          /* 恢复前先刷新 */

    dup2(saved, STDOUT_FILENO);              /* 恢复到原来的终端 */
    close(saved);

    printf("这一行回到终端\n");
    return 0;
}
```

Shell 里的 `cmd > file`、`cmd 2>&1` 本质上就是 `dup2`。

### 5.2 `pipe` / `pipe2`：创建管道（POSIX / pipe2 为 Linux）

```c
int pipe(int pipefd[2]);                   /* pipefd[0] 读端，pipefd[1] 写端 */
int pipe2(int pipefd[2], int flags);       /* Linux：可加 O_CLOEXEC / O_NONBLOCK */
```

典型模式：`pipe` + `fork` + `dup2` + `exec`，在父子进程间传递输出：

```c
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void)
{
    int fds[2];
    if (pipe(fds) == -1) { perror("pipe"); return 1; }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {                              /* 子进程 */
        close(fds[0]);                           /* 不用读端 */
        dup2(fds[1], STDOUT_FILENO);             /* 标准输出接到管道写端 */
        close(fds[1]);
        execlp("ls", "ls", "-l", (char *)NULL);
        perror("execlp");
        _exit(127);
    }

    close(fds[1]);                               /* 父进程不用写端（否则读不到 EOF） */

    char buf[256];
    ssize_t n;
    while ((n = read(fds[0], buf, sizeof buf)) > 0)
        write(STDOUT_FILENO, buf, (size_t)n);

    close(fds[0]);
    waitpid(pid, NULL, 0);
    return 0;
}
```

注意：**必须关闭不用的那一端。** 如果父进程不关闭写端，读端永远等不到 EOF。

### 5.3 `fcntl`：修改文件描述符属性（POSIX）

```c
int fcntl(int fd, int cmd, ... /* arg */);
```

| 命令 | 作用 |
|---|---|
| `F_GETFL` / `F_SETFL` | 获取/设置文件状态标志（如 `O_NONBLOCK`、`O_APPEND`） |
| `F_GETFD` / `F_SETFD` | 获取/设置描述符标志（如 `FD_CLOEXEC`） |
| `F_DUPFD` / `F_DUPFD_CLOEXEC` | 复制描述符 |
| `F_SETLK` / `F_SETLKW` / `F_GETLK` | 记录锁（见 5.5） |
| `F_SETPIPE_SZ` / `F_GETPIPE_SZ` | 修改/查询管道容量（Linux） |

把描述符设为非阻塞：

```c
int flags = fcntl(fd, F_GETFL, 0);
if (flags == -1) { perror("F_GETFL"); return 1; }

if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
    perror("F_SETFL");
    return 1;
}
```

非阻塞模式下，没有数据可读时 `read` 不会等待，而是返回 `-1` 且 `errno` 为 `EAGAIN`（或等价的 `EWOULDBLOCK`）。

### 5.4 `fsync` / `fdatasync` / `sync`：强制落盘（POSIX）

`write` 成功只表示数据进入了**内核页缓存**，不保证已经写到磁盘。断电可能丢数据。

| 函数 | 作用 |
|---|---|
| `fsync(fd)` | 把该文件的数据**和元数据**都刷到磁盘 |
| `fdatasync(fd)` | 只刷数据和必要的元数据（通常更快） |
| `sync()` | 请求把所有缓存的修改写出 |
| `syncfs(fd)` | 刷新 `fd` 所在文件系统（Linux） |

使用 `stdio` 时，需要**两步**：

```c
fprintf(fp, "重要数据\n");
fflush(fp);                  /* 第一步：用户态缓冲 → 内核 */
fsync(fileno(fp));           /* 第二步：内核页缓存 → 磁盘 */
```

写配置文件、数据库日志等要求"掉电不丢"的场景必须这样做。

### 5.5 文件锁（多进程协调）

这些都是**建议性锁（advisory lock）**，只对遵守锁协议的进程有效，不会阻止没有加锁的进程读写。

| 接口 | 说明 | 来源 |
|---|---|---|
| `flock(fd, LOCK_SH / LOCK_EX / LOCK_UN)` | 对整个文件加共享/独占锁，简单易用 | BSD / Linux |
| `fcntl(fd, F_SETLK / F_SETLKW, &fl)` | 可以锁文件的某一段字节范围 | POSIX |
| `fcntl(fd, F_OFD_SETLK, &fl)` | 与打开文件描述（open file description）关联的锁，避免 POSIX 记录锁的陷阱 | Linux 3.15+ |
| `lockf(fd, F_LOCK, len)` | `fcntl` 记录锁的简化封装 | POSIX |

最简单的用法（`flock`）：

```c
#include <sys/file.h>

flock(fd, LOCK_EX);        /* 阻塞直到获得独占锁 */
/* ... 读写文件 ... */
flock(fd, LOCK_UN);        /* 释放 */
```

### 5.6 `ftruncate` / `truncate`：改变文件大小（POSIX）

```c
int ftruncate(int fd, off_t length);
int truncate(const char *path, off_t length);
```

缩短会丢弃末尾数据；扩大则新增部分读出来全是 `0`（可能形成稀疏文件）。

---

## 6. 位置相关的读写：`pread` / `pwrite` 与分散/聚集 I/O

### 6.1 `pread` / `pwrite`：指定偏移量，且不移动文件位置（POSIX）

```c
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

-   等价于 `lseek` + `read`/`write`，但**是原子的**，且**不改变**文件当前位置。
-   多个线程共用同一个文件描述符、读写文件不同位置时，非常有用，不会互相干扰。
-   对管道、套接字无效（`ESPIPE`）。
-   注意：以 `O_APPEND` 打开时，Linux 上 `pwrite` 仍然会追加到末尾，而忽略 `offset`。

```c
char header[16];
pread(fd, header, sizeof header, 0);        /* 读文件开头 16 字节 */
pwrite(fd, "OK", 2, 1024);                  /* 在偏移 1024 处写 2 字节 */
```

### 6.2 `readv` / `writev`：分散读、聚集写（POSIX）

一次系统调用，读写多块**不连续**的内存，避免多次调用和额外的拷贝。

```c
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
    void  *iov_base;    /* 内存起始地址 */
    size_t iov_len;     /* 长度 */
};
```

```c
#include <string.h>
#include <sys/uio.h>
#include <unistd.h>

int main(void)
{
    struct iovec iov[3];

    iov[0].iov_base = "Hello, ";   iov[0].iov_len = 7;
    iov[1].iov_base = "Linux";     iov[1].iov_len = 5;
    iov[2].iov_base = "!\n";       iov[2].iov_len = 2;

    writev(STDOUT_FILENO, iov, 3);   /* 一次调用输出 "Hello, Linux!\n" */
    return 0;
}
```

典型用途：网络协议中，"包头 + 数据体"分别放在不同的内存里，一次发出。

相关的还有 `preadv` / `pwritev`（带偏移）和 Linux 专有的 `preadv2` / `pwritev2`（可带额外标志）。

---

## 7. 内存映射 I/O：`mmap`

把文件**映射到进程地址空间**，之后像访问数组一样访问文件，不用显式 `read` / `write`。属于 **POSIX**。

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int   munmap(void *addr, size_t length);
int   msync(void *addr, size_t length, int flags);
```

常用参数：

| 参数 | 常见取值 |
|---|---|
| `prot` | `PROT_READ`、`PROT_WRITE`、`PROT_EXEC`、`PROT_NONE` |
| `flags` | `MAP_SHARED`（修改会写回文件，别的进程可见）、`MAP_PRIVATE`（写时复制，不影响文件）、`MAP_ANONYMOUS`（匿名映射，不对应文件；Linux 常用） |

示例：把整个文件映射后输出：

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(void)
{
    int fd = open("data.txt", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    struct stat st;
    if (fstat(fd, &st) < 0) { perror("fstat"); close(fd); return 1; }

    if (st.st_size == 0) {                  /* 长度为 0 的文件无法映射 */
        close(fd);
        return 0;
    }

    char *p = mmap(NULL, (size_t)st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (p == MAP_FAILED) {                  /* 失败返回 MAP_FAILED，不是 NULL */
        perror("mmap");
        close(fd);
        return 1;
    }

    close(fd);                              /* 映射建立后即可关闭描述符 */

    fwrite(p, 1, (size_t)st.st_size, stdout);

    munmap(p, (size_t)st.st_size);
    return 0;
}
```

优缺点：

-   优点：省去用户态/内核态之间的一次数据拷贝；随机访问方便；可用于多进程共享内存（`MAP_SHARED`）。
-   缺点：映射期间如果别的进程把文件截短，访问越界部分会收到 `SIGBUS`；不适合小文件频繁映射；错误处理比 `read` 更麻烦；对管道、终端等不可用。
-   使用 `MAP_SHARED` + `PROT_WRITE` 修改后，需要 `msync` 才能确保写回磁盘。

相关辅助：`madvise`（给内核访问模式的建议）、`mlock`（锁定内存，防止换出）。

---

## 8. 内核层面的高效数据搬运

这些接口让数据**不经过用户态**（零拷贝或少拷贝），常用于文件复制、静态文件服务、代理转发。

| 接口 | 作用 | 来源 |
|---|---|---|
| `sendfile(out_fd, in_fd, &off, count)` | 在两个描述符之间直接搬运数据 | Linux |
| `copy_file_range(fd_in, off_in, fd_out, off_out, len, flags)` | 在文件之间在内核内复制数据，部分文件系统可优化为快速克隆 | Linux 4.5+ |
| `splice(fd_in, off_in, fd_out, off_out, len, flags)` | 在文件与管道之间移动数据；**两端至少有一端是管道** | Linux |
| `tee(fd_in, fd_out, len, flags)` | 复制管道数据，但不消耗输入端 | Linux |
| `vmsplice` | 把用户内存页"拼接"进管道 | Linux |

### 8.1 `sendfile` 示例

```c
#include <sys/sendfile.h>

off_t off = 0;
while (off < st.st_size) {
    ssize_t n = sendfile(out_fd, in_fd, &off, (size_t)(st.st_size - off));
    if (n <= 0) break;       /* 出错或没有更多数据，应检查 errno */
}
```

-   `in_fd` 必须是支持类似 `mmap` 的普通文件。
-   Linux 2.6.33 之前，`out_fd` 必须是套接字；之后可以是任意文件。
-   典型用途：Web 服务器把磁盘上的文件直接发给套接字。

### 8.2 `copy_file_range` 示例

```c
#define _GNU_SOURCE
#include <unistd.h>

ssize_t n;
while ((n = copy_file_range(in_fd, NULL, out_fd, NULL, 1 << 20, 0)) > 0)
    ;                        /* 偏移传 NULL，表示使用并推进各自的文件位置 */
if (n < 0)
    perror("copy_file_range");
```

比"循环 `read` + `write`"更简洁，也更快，复制大文件很合适。

### 8.3 给内核的 I/O 提示与预分配

| 接口 | 作用 | 来源 |
|---|---|---|
| `posix_fadvise(fd, off, len, advice)` | 告知访问模式（`POSIX_FADV_SEQUENTIAL` 顺序读、`POSIX_FADV_RANDOM` 随机读、`POSIX_FADV_DONTNEED` 不再需要缓存等） | POSIX |
| `readahead(fd, off, count)` | 提前把数据读入页缓存 | Linux |
| `posix_fallocate(fd, off, len)` | 预先分配磁盘空间，确保后续写入不会因空间不足失败 | POSIX |
| `fallocate(fd, mode, off, len)` | 功能更丰富的预分配/打洞/插入空间 | Linux |

---

## 9. I/O 多路复用：同时等待多个文件描述符

**问题：** 一个程序要同时处理多个输入源（比如键盘 + 多个网络连接），如果对每个都阻塞 `read`，就只能等其中一个。I/O 多路复用让你"一次等多个，哪个就绪处理哪个"。

三代接口：

| 接口 | 头文件 | 特点 | 来源 |
|---|---|---|---|
| `select` / `pselect` | `<sys/select.h>` | 最古老；`fd_set` 受 `FD_SETSIZE`（通常 1024）限制；每次调用都要重新设置集合 | POSIX |
| `poll` / `ppoll` | `<poll.h>` | 没有 1024 限制；每次调用要把整个数组交给内核 | POSIX（`ppoll` 为 Linux） |
| `epoll` | `<sys/epoll.h>` | 大量连接时效率高，事件由内核维护，只返回就绪的 | Linux 专有 |

### 9.1 `poll` 示例：带超时地等待标准输入

```c
#include <poll.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    struct pollfd pfd = { .fd = STDIN_FILENO, .events = POLLIN };

    printf("3 秒内请输入内容……\n");
    int r = poll(&pfd, 1, 3000);              /* 超时单位：毫秒；-1 表示一直等 */

    if (r < 0) {
        perror("poll");
    } else if (r == 0) {
        printf("超时\n");
    } else if (pfd.revents & POLLIN) {
        char buf[128];
        ssize_t n = read(STDIN_FILENO, buf, sizeof buf - 1);
        if (n > 0) {
            buf[n] = '\0';
            printf("读到：%s", buf);
        }
    }
    return 0;
}
```

常见事件：`POLLIN`（可读）、`POLLOUT`（可写）、`POLLERR`（出错）、`POLLHUP`（对端挂断）。

### 9.2 `epoll` 基本流程

三个核心函数：

```c
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);   /* op: EPOLL_CTL_ADD / MOD / DEL */
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

示例（监听标准输入）：

```c
#include <stdio.h>
#include <sys/epoll.h>
#include <unistd.h>

int main(void)
{
    int epfd = epoll_create1(0);
    if (epfd < 0) { perror("epoll_create1"); return 1; }

    struct epoll_event ev = { .events = EPOLLIN, .data.fd = STDIN_FILENO };
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev) < 0) {
        perror("epoll_ctl");
        close(epfd);
        return 1;
    }

    struct epoll_event events[8];
    int n = epoll_wait(epfd, events, 8, -1);      /* -1：无限等待 */

    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == STDIN_FILENO) {
            char buf[128];
            ssize_t len = read(STDIN_FILENO, buf, sizeof buf - 1);
            if (len > 0) {
                buf[len] = '\0';
                printf("读到：%s", buf);
            }
        }
    }

    close(epfd);
    return 0;
}
```

要点：

-   **水平触发（LT，默认）**：只要还有数据可读，每次 `epoll_wait` 都会报告。
-   **边沿触发（ET，`EPOLLET`）**：仅在状态**变化**时通知一次，必须配合**非阻塞** I/O，并且收到通知后要一直读到 `EAGAIN`，否则会丢事件。
-   `epoll` **不支持普通磁盘文件**（`epoll_ctl` 会返回 `EPERM`），普通文件总被认为"随时可读写"。
-   `select` / `poll` / `epoll` 只告诉你"可以读/写了"，真正的读写还是用 `read` / `write` 完成。

---

## 10. 异步 I/O

多路复用是"告诉我什么时候**能**读"；异步 I/O 是"**你帮我读完**，读完再告诉我"。

| 方案 | 说明 | 来源 |
|---|---|---|
| POSIX AIO（`aio_read` / `aio_write` / `aio_error` / `aio_return` / `aio_suspend`，`<aio.h>`） | glibc 内部通过**用户态线程池**模拟，并非真正的内核异步；老版本 glibc 链接需 `-lrt` | POSIX |
| Linux 原生 AIO（`io_setup` / `io_submit` / `io_getevents`，常用 `libaio`） | 内核级异步，但基本只对 `O_DIRECT` 的文件 I/O 才真正异步 | Linux |
| **`io_uring`** | 现代 Linux 异步 I/O 框架：提交队列（SQ）和完成队列（CQ）通过共享内存与内核交互，可大幅减少系统调用；支持文件、网络等多种操作 | Linux 5.1+ |

`io_uring` 通常通过 `liburing` 库使用（编译时加 `-luring`），一次读操作的最小流程：

```c
#include <fcntl.h>
#include <liburing.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    int fd = open("data.txt", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    struct io_uring ring;
    if (io_uring_queue_init(8, &ring, 0) < 0) {       /* 队列深度 8 */
        fprintf(stderr, "io_uring_queue_init 失败\n");
        return 1;
    }

    char buf[4096];

    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);        /* 取一个提交项 */
    io_uring_prep_read(sqe, fd, buf, sizeof buf, 0);           /* 描述：从偏移 0 读 */
    io_uring_submit(&ring);                                    /* 提交给内核 */

    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);                            /* 等待完成 */

    if (cqe->res < 0)
        fprintf(stderr, "读取失败：%d\n", cqe->res);           /* 失败时 res 是 -errno */
    else
        printf("读到 %d 字节\n", cqe->res);

    io_uring_cqe_seen(&ring, cqe);                             /* 标记该完成项已处理 */
    io_uring_queue_exit(&ring);
    close(fd);
    return 0;
}
```

初学阶段先了解它们的定位即可：**普通程序用 `read` / `write` + `poll`/`epoll` 已经足够；追求极致吞吐的存储、网络服务才需要 `io_uring`。**

---

## 11. 终端 I/O：termios 与终端控制

标准 C 的 `getchar` 在终端上是**行缓冲**的：用户必须按回车，程序才能收到输入；输入的字符也会自动回显。想实现"按一个键立即响应""输入密码不回显"，就要用 POSIX 的终端接口 `termios`。

### 11.1 常用函数

| 函数 | 作用 | 来源 |
|---|---|---|
| `isatty(fd)` | 判断是否连接终端 | POSIX |
| `tcgetattr(fd, &t)` / `tcsetattr(fd, when, &t)` | 获取/设置终端属性 | POSIX |
| `cfmakeraw(&t)` | 把属性设为"原始模式" | BSD / glibc |
| `ttyname(fd)` | 获得终端设备名（如 `/dev/pts/0`） | POSIX |
| `ioctl(fd, TIOCGWINSZ, &ws)` | 获取终端窗口大小 | Linux / BSD |

关键标志（`c_lflag`）：

| 标志 | 含义 |
|---|---|
| `ICANON` | 规范（行缓冲）模式；关闭后按字节立即返回 |
| `ECHO` | 回显输入字符；关闭后不显示 |

非规范模式下，`c_cc[VMIN]` 和 `c_cc[VTIME]` 控制 `read` 的返回条件（最少读几个字节、等多久）。

### 11.2 示例：不用回车，读取一个按键（不回显）

```c
#include <stdio.h>
#include <termios.h>
#include <unistd.h>

int main(void)
{
    struct termios old, raw;

    if (tcgetattr(STDIN_FILENO, &old) < 0) { perror("tcgetattr"); return 1; }

    raw = old;
    raw.c_lflag &= ~(tcflag_t)(ICANON | ECHO);   /* 关闭行缓冲和回显 */
    raw.c_cc[VMIN]  = 1;                         /* 至少读到 1 个字节才返回 */
    raw.c_cc[VTIME] = 0;                         /* 不设超时 */

    if (tcsetattr(STDIN_FILENO, TCSANOW, &raw) < 0) { perror("tcsetattr"); return 1; }

    printf("请按任意键……\n");

    char c;
    if (read(STDIN_FILENO, &c, 1) == 1)          /* 直接用 read，避开 stdio 缓冲 */
        printf("你按下的是：%c\n", c);

    tcsetattr(STDIN_FILENO, TCSANOW, &old);      /* 一定要恢复！ */
    return 0;
}
```

注意：

-   **程序退出前一定要恢复原设置**，否则终端会处于异常状态（可在 shell 中执行 `reset` 或 `stty sane` 修复）。实际程序还应处理 `SIGINT` 等信号，在退出路径上恢复。
-   这种场景下建议直接用 `read`，避免和 stdio 的缓冲互相干扰。

### 11.3 获取终端大小

```c
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(void)
{
    struct winsize ws;

    if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == 0)
        printf("终端大小：%d 行 x %d 列\n", ws.ws_row, ws.ws_col);
    else
        perror("ioctl");
    return 0;
}
```

### 11.4 更高层的终端库（第三方库，非系统调用）

| 库 | 用途 |
|---|---|
| `ncurses` | 全屏文本界面（菜单、窗口、颜色、按键处理） |
| `readline` | 带行编辑、历史记录、自动补全的输入（Bash 就在用它） |

---

## 12. 文件元信息与目录 I/O

虽然不是"读写数据"，但处理文件时几乎总要用到。

### 12.1 `stat` / `fstat` / `lstat`：查询文件信息（POSIX）

```c
#include <sys/stat.h>

int stat(const char *path, struct stat *buf);      /* 符号链接会被跟随 */
int lstat(const char *path, struct stat *buf);     /* 不跟随符号链接 */
int fstat(int fd, struct stat *buf);               /* 通过描述符查询 */
```

`struct stat` 中常用字段：`st_size`（大小）、`st_mode`（类型和权限）、`st_mtime`（修改时间）、`st_uid` / `st_gid`（属主）。

判断文件类型的宏：

| 宏 | 含义 |
|---|---|
| `S_ISREG(m)` | 普通文件 |
| `S_ISDIR(m)` | 目录 |
| `S_ISLNK(m)` | 符号链接 |
| `S_ISFIFO(m)` | 管道（FIFO） |
| `S_ISSOCK(m)` | 套接字 |
| `S_ISCHR(m)` / `S_ISBLK(m)` | 字符设备 / 块设备 |

```c
struct stat st;
if (stat("data.txt", &st) == 0) {
    printf("大小：%lld 字节\n", (long long)st.st_size);
    if (S_ISDIR(st.st_mode))
        printf("这是一个目录\n");
}
```

### 12.2 目录读取：`opendir` / `readdir` / `closedir`（POSIX）

```c
#include <dirent.h>
#include <stdio.h>

int main(void)
{
    DIR *d = opendir(".");
    if (d == NULL) { perror("opendir"); return 1; }

    struct dirent *e;
    while ((e = readdir(d)) != NULL)
        puts(e->d_name);                 /* 包含 "." 和 ".." */

    closedir(d);
    return 0;
}
```

### 12.3 `*at` 系列：相对于目录描述符的操作（POSIX.1-2008）

`openat`、`fstatat`、`unlinkat`、`mkdirat`、`renameat` 等，路径是相对于一个**目录文件描述符**解析的。

-   避免"检查-使用"之间路径被替换导致的竞态（TOCTOU 安全问题）。
-   多线程程序里不依赖进程全局的"当前工作目录"。

```c
int dirfd = open("/var/data", O_RDONLY | O_DIRECTORY);
int fd = openat(dirfd, "file.txt", O_RDONLY);
```

---

## 13. 特殊用途的文件描述符（Linux 专有）

Linux 把很多内核事件也包装成文件描述符，从而可以统一交给 `poll` / `epoll` 处理。

| 接口 | 作用 |
|---|---|
| `inotify_init1` / `inotify_add_watch` | **监控文件或目录变化**（创建、修改、删除、移动） |
| `eventfd` | 轻量的事件通知计数器，常用于线程/进程间唤醒 |
| `timerfd_create` / `timerfd_settime` | 定时器，到期时描述符变为可读 |
| `signalfd` | 把信号变成可读取的描述符 |
| `pidfd_open` | 代表一个进程的描述符，可用 `poll` 等待其退出 |
| `memfd_create` | 创建只存在于内存的匿名文件 |

统一的思路：**把各种事件变成文件描述符 → 用 `epoll` 在一个事件循环里统一等待。**

---

## 14. 网络套接字 I/O（概览）

套接字也是文件描述符，因此 `read` / `write` / `close` / `poll` / `epoll` 都能用在它上面。此外还有专用接口：

| 函数 | 作用 |
|---|---|
| `socket` | 创建套接字 |
| `bind` / `listen` / `accept` | 服务端：绑定地址、监听、接受连接 |
| `connect` | 客户端：发起连接 |
| `send` / `recv` | 带标志的收发（面向连接） |
| `sendto` / `recvfrom` | 收发数据报（UDP），可指定对端地址 |
| `sendmsg` / `recvmsg` | 最通用的收发，支持分散/聚集和辅助数据 |
| `shutdown` | 关闭连接的读/写方向 |
| `getsockopt` / `setsockopt` | 获取/设置套接字选项 |

这部分内容属于网络编程，此处只说明它在 I/O 体系中的位置。

注意：向已经被对端关闭的套接字或管道写入，会收到 `SIGPIPE` 信号，默认**终止进程**。网络程序通常会忽略它（`signal(SIGPIPE, SIG_IGN)`），或在 `send` 时加 `MSG_NOSIGNAL`，然后通过返回值 `-1` 且 `errno == EPIPE` 处理错误。

---

## 15. 错误处理与日志相关的扩展

### 15.1 `errno` 与常见错误码

系统调用失败通常返回 `-1` 并设置 `errno`（`<errno.h>`，标准 C 中有 `errno`，但具体错误码名称多数来自 POSIX）：

| 错误码 | 含义 | 常见处理 |
|---|---|---|
| `EINTR` | 系统调用被信号中断 | 通常重试 |
| `EAGAIN` / `EWOULDBLOCK` | 非阻塞 I/O 暂时没有数据/空间 | 稍后重试，或等待 `poll`/`epoll` |
| `EPIPE` | 向已关闭的管道/套接字写入 | 视为连接已断开 |
| `ENOENT` | 文件或目录不存在 | 检查路径 |
| `EACCES` / `EPERM` | 权限不足 | 检查权限 |
| `EMFILE` / `ENFILE` | 进程/系统打开的文件过多 | 关闭不用的描述符，或调大限制（`ulimit -n`） |
| `ENOSPC` | 磁盘空间不足 | 清理空间 |
| `EBADF` | 无效的文件描述符 | 检查是否已 `close` 或传错 |

### 15.2 `strerror` 与 `strerror_r`

```c
char *strerror(int errnum);                          /* 标准 C，但不一定线程安全 */
int   strerror_r(int errnum, char *buf, size_t n);   /* 线程安全版本（POSIX 版本返回 int） */
```

注意：glibc 在定义了 `_GNU_SOURCE` 时，`strerror_r` 会变成返回 `char *` 的 GNU 版本，签名不同。可移植代码需要留意，或者用 `snprintf` 配合 `%m`。

### 15.3 `err` / `warn` 系列（BSD，glibc 提供）

```c
#include <err.h>

err(1, "打开 %s 失败", path);     /* 打印"程序名: 打开 xxx 失败: 错误描述"并以状态 1 退出 */
errx(1, "参数错误");              /* 不附带 errno 的描述，并退出 */
warn("读取 %s 出错", path);       /* 同 err，但不退出 */
warnx("警告：配置缺失");          /* 同 errx，但不退出 */
```

适合写小工具时快速、统一地报告错误。

### 15.4 `syslog`：写入系统日志（POSIX）

```c
#include <syslog.h>
#include <unistd.h>

int main(void)
{
    openlog("myapp", LOG_PID, LOG_USER);          /* 程序标识、选项、设施 */
    syslog(LOG_INFO, "程序启动，pid=%d", (int)getpid());
    syslog(LOG_ERR,  "发生错误：%m");             /* syslog 的格式串里也支持 %m */
    closelog();
    return 0;
}
```

日志级别从高到低：`LOG_EMERG`、`LOG_ALERT`、`LOG_CRIT`、`LOG_ERR`、`LOG_WARNING`、`LOG_NOTICE`、`LOG_INFO`、`LOG_DEBUG`。在使用 systemd 的系统上，可以用 `journalctl -t myapp` 查看。

守护进程（没有终端）无法使用 `printf` 输出，`syslog` 是标准的日志手段。

---

## 16. 标准 C 与 Linux 扩展：对照速查

### 16.1 读取一行

| 需求 | 标准 C | Linux / POSIX 扩展 |
|---|---|---|
| 读取一行（定长缓冲） | `fgets` | ------ |
| 读取任意长度的一行 | 需要自己循环 + `realloc` | `getline` |
| 指定分隔符 | 需要自己实现 | `getdelim` |

### 16.2 格式化输出到字符串

| 需求 | 标准 C | Linux / GNU 扩展 |
|---|---|---|
| 带长度限制 | `snprintf` | ------ |
| 自动分配内存 | 先 `snprintf(NULL, 0, …)` 算长度再 `malloc` | `asprintf` |
| 输出到内存文件流 | ------ | `open_memstream` |
| 直接输出到文件描述符 | ------ | `dprintf` |

### 16.3 打开与读写文件

| 需求 | 标准 C | Linux / POSIX 扩展 |
|---|---|---|
| 打开文件 | `fopen` | `open`（可设置 `O_NONBLOCK`、`O_CLOEXEC`、`O_DIRECT` 等） |
| 读写数据 | `fread` / `fwrite` | `read` / `write` / `pread` / `pwrite` |
| 定位 | `fseek` / `ftell` | `lseek` / `fseeko` / `ftello` |
| 刷新到磁盘 | `fflush`（只到内核） | `fsync` / `fdatasync` |
| 文件大小 | `fseek` + `ftell` | `fstat` |
| 执行命令并读输出 | `system`（拿不到输出） | `popen` |

### 16.4 终端交互

| 需求 | 标准 C | Linux / POSIX 扩展 |
|---|---|---|
| 读一个字符（需回车） | `getchar` | ------ |
| 立即读取按键、不回显 | ------ | `termios` |
| 判断是否在终端 | ------ | `isatty` |
| 获取终端大小 | ------ | `ioctl(TIOCGWINSZ)` |

### 16.5 按"场景"选择高级接口

| 场景 | 优先考虑 |
|---|---|
| 普通文本/配置文件读写 | 标准 `stdio`（`fopen` / `fgets` / `fprintf`） |
| 读取任意长度的行 | `getline` |
| 同时等待多个输入源 | `poll`；连接数很多时 `epoll` |
| 多线程读写同一文件的不同位置 | `pread` / `pwrite` |
| 高速复制大文件 | `copy_file_range` / `sendfile` |
| 大文件随机访问、多进程共享数据 | `mmap` |
| 要求掉电不丢数据 | `fflush` + `fsync` |
| 重定向标准输出 | `dup2` |
| 父子进程间通信 | `pipe` + `fork` |
| 执行 shell 命令并读取输出 | `popen`（输入可信时） |
| 监控目录变化 | `inotify` |
| 网络服务的高并发 I/O | 非阻塞套接字 + `epoll`，或 `io_uring` |
| 守护进程日志 | `syslog` |
| 交互式字符界面 | `termios`，或 `ncurses` |

---

## 17. 常见陷阱汇总

1.  **忘记定义特性测试宏**：用 `-std=c11` 等严格模式编译时，`getline`、`fileno`、`fdopen`、`popen` 等会"未声明"。
2.  **把 `read` / `write` 当成一定读写完**：它们可能短读/短写，要循环处理，并处理 `EINTR`。
3.  **混用 `stdio` 和 `read`/`write`**：缓冲区没刷新，导致输出顺序错乱；`fork` 前没 `fflush(NULL)` 导致重复输出。
4.  **`fdopen` 后又手动 `close(fd)`**：`fclose` 会关闭它，重复关闭会出问题。
5.  **`popen` 用了 `fclose` 关闭**：必须用 `pclose`，否则会泄漏子进程。
6.  **`getline` / `asprintf` / `open_memstream` / `%ms` 分配的内存忘记 `free`**。
7.  **管道的无用端没有关闭**：读端永远收不到 EOF，程序卡死。
8.  **`fflush` 当作落盘手段**：`fflush` 只把数据交给内核，不保证写到磁盘；需要 `fsync`。
9.  **`mmap` 失败检查错了**：失败返回 `MAP_FAILED`（即 `(void *)-1`），不是 `NULL`；空文件无法映射。
10. **`epoll` 边沿触发没配合非阻塞**：可能丢事件，或者卡死在阻塞的 `read` 上。
11. **修改终端属性后没恢复**：程序异常退出会让终端无回显、行为怪异。
12. **向已关闭的套接字/管道写入没处理 `SIGPIPE`**：进程被默默杀死。
13. **`open` 使用 `O_CREAT` 却漏了 `mode` 参数**：权限随机，是未定义的栈垃圾值。
14. **`O_DIRECT` 缓冲区未对齐**：会返回 `EINVAL`；需要用 `posix_memalign` / `aligned_alloc` 分配对齐的缓冲区。

---

## 18. 学习路线建议

在已经掌握标准 C I/O 的基础上，推荐按以下顺序扩展：

1.  **`getline`**：先解决"读任意长度的行"这个最常见的问题。
2.  **`open` / `read` / `write` / `close`**：理解文件描述符，写出可靠的 `write_all`。
3.  **`fileno` / `fdopen` / `dup2` / `pipe`**：理解 `FILE *` 与描述符的关系，学会重定向和进程间管道。
4.  **`fstat` / `opendir` / `readdir`**：能获取文件信息、遍历目录。
5.  **`fsync` 与缓冲问题**：理解"数据到底写到哪一层了"。
6.  **`poll` → `epoll`**：理解 I/O 多路复用，能写简单的事件循环。
7.  **`mmap`、`sendfile` / `copy_file_range`**：理解内存映射与零拷贝。
8.  **`termios`**：做交互式终端程序。
9.  **`io_uring`、`inotify` 等**：按需深入。

### 查阅手册

Linux 自带的 `man` 手册是最权威的参考：

```bash
man 2 open          # 第 2 节：系统调用
man 3 getline       # 第 3 节：C 库函数
man 7 epoll         # 第 7 节：概念性说明
man 7 pipe
man 7 feature_test_macros
man 7 io_uring
```

### 最后记住三个核心概念

-   **文件描述符（fd）**：Linux I/O 的统一入口，文件、管道、终端、套接字都是它。
-   **缓冲层次**：`printf` 的数据 → stdio 用户态缓冲 → 内核页缓存 → 磁盘；每一层都有对应的"刷新"手段（`fflush` → `fsync`）。
-   **阻塞与事件驱动**：简单程序用阻塞 I/O；需要同时处理多个来源时，用 `poll` / `epoll` 等待就绪事件，再去读写。

把这三个概念分清楚，Linux 下的各种 I/O 接口就有了清晰的脉络。
