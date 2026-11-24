# C 语言 Socket 编程与 Rust 对比学习笔记

> 从零手写 C 语言 TCP 服务器，并全程对照 Rust 的 `TcpListener`/`TcpStream` 实现，梳理两种语言在系统编程层面的设计差异。

---

## 1. 环境搭建：为什么要用 WSL

- `sys/socket.h` 是 **POSIX（Linux/macOS）头文件**，Windows 原生（MSVC/MinGW）没有这个文件，需要改用 Winsock（`winsock2.h`）才能编译。
- 更推荐的路径是装 **WSL**（Windows Subsystem for Linux）：
  - WSL 是"让 Windows 能跑 Linux 内核"的平台技术，Ubuntu 是装在这个平台上的具体发行版，两者是"底盘"和"车身"的关系。
  - 装好后代码、系统调用与真实 Linux 服务器完全一致，不需要为了适配 Windows 修改任何代码。
  - 安装：管理员 PowerShell 执行 `wsl --install`，重启后若发现没有默认发行版，再执行 `wsl --install -d Ubuntu`。
  - 常见坑：WSL2 的 DNS 解析可能失效（`Temporary failure resolving ...`），需要手动修复 `/etc/resolv.conf`（改成 `nameserver 8.8.8.8` 并 `chattr +i` 锁定）。
  - VS Code 要装 **WSL 扩展**并 "Connect to WSL"，同时 C/C++ 扩展也要**在 WSL 侧单独装一份**（Windows 端装的扩展在 WSL 模式下不生效），否则 `Ctrl+点击` 跳转不了源码。
  - 项目文件建议放在 Linux 自己的目录（`~/xxx`），不要放在 `/mnt/c/...` 下，否则编译速度明显变慢。
  - Windows 访问 WSL 里的文件：地址栏输入 `\\wsl$\Ubuntu\home\用户名\...`，或在 WSL 终端里执行 `explorer.exe .`。

---

## 2. Socket 是什么：内核里的真实结构

Socket 不是一个"东西"，而是内核里的一块数据结构；用户进程拿到的 `fd`（文件描述符）只是一把"钥匙"。

```
进程内存                         内核内存
┌────────────┐                 ┌─────────────────────┐
│ fd 表        │                 │ socket 结构体          │
│ fd 3 → ●─────┼───────────────→│  本地地址 / 远端地址     │
└────────────┘                 │  状态（LISTENING等）    │
                                │  接收缓冲区              │
                                │  发送缓冲区              │
                                └─────────────────────┘
```

（简化）内核 socket 结构体大致包含的字段类别：

| 分类 | 字段举例 |
|---|---|
| 身份信息 | family（AF_INET）、type（SOCK_STREAM）、protocol |
| 地址信息 | 本地地址+端口、远端地址+端口 |
| 状态 | TCP 状态机：CLOSED → LISTEN → ESTABLISHED → ... |
| 缓冲区 | 接收缓冲区、发送缓冲区 |
| 仅监听 socket 有 | backlog、等待连接队列（accept queue） |
| TCP 协议状态 | 序列号、确认号、拥塞窗口、重传计时器 |
| 可配置选项 | SO_REUSEADDR、SO_KEEPALIVE、TCP_NODELAY |
| 内核内部管理 | 引用计数、锁 |

这些字段绝大部分对用户程序不可见，只能通过系统调用间接操作。

---

## 3. C 语言 Socket 编程四步（服务端）

### 3.1 `socket()`
```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
```
向内核申请分配一个新的 socket 结构体，返回一个 fd。此时地址未绑定，状态未确定。

### 3.2 `bind()`
```c
struct sockaddr_in address;
memset(&address, 0, sizeof(address));   // 清零，避免栈上垃圾数据
address.sin_family = AF_INET;
address.sin_addr.s_addr = INADDR_ANY;   // 监听本机所有网卡
address.sin_port = htons(8080);         // htons：主机字节序→网络字节序

bind(server_fd, (struct sockaddr *)&address, sizeof(address));
```
只是把地址信息填进内核那个 socket 结构体，**不会创建新 socket**。

- `htons`（host to network short）：不同 CPU 存多字节数字的顺序不同（大端/小端），网络传输统一规定用大端，所以端口号、IP 地址在填入前要转换字节序。
- `(struct sockaddr *)&address` 的强转：`bind()` 是通用函数，能接受多种地址类型，靠这层"基类指针"兼容不同协议族的地址结构体。

### 3.3 `listen()`
```c
listen(server_fd, 10);
```
状态变为 `LISTENING`，并分配一个**等待队列**（backlog），存放"已完成三次握手但还没被 `accept()` 取走"的连接。

### 3.4 `accept()`
```c
struct sockaddr_in client_addr;
socklen_t addr_len = sizeof(client_addr);
int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &addr_len);
```
- 从等待队列取出一个连接，**内核为它单独新建一个全新的 socket 结构体**（状态 ESTABLISHED，记录对端地址）。
- 原来的 `server_fd`（listening socket）完全不受影响，继续 LISTENING，等待下一个新客户端。
- `server_fd` 好比"前台"，只负责一直站着迎接新客人；`accept()` 返回的新 fd 好比"专属服务员"，只服务这一位客人，生命周期与前台完全脱钩。

---

## 4. C 与 Rust 的对应关系

| C | Rust | 说明 |
|---|---|---|
| `socket()` + `bind()` + `listen()` 三步 | `TcpListener::bind(addr)` | Rust 把三步合并成一次调用 |
| `server_fd` | `listener`（`TcpListener`） | 全程只有一个，负责监听 |
| `accept(server_fd, ...)` | `listener.incoming()` 每次迭代内部调用 | 语义相同 |
| `accept()` 返回的新 `client_fd` | `for` 循环每次拿到的新 `stream`（`TcpStream`） | 都是独立的新连接对象 |
| `while(1) { accept(...); ... }` | `for stream in listener.incoming() { ... }` | `for...in` 是 `Iterator` 的语法糖，本质等价于 `loop { match iter.next() {...} }` |
| 阻塞发生在 `accept()` 这一行（字面可见） | 阻塞发生在 `for` 循环背后隐式调用的 `.next()`（本质也是 `accept()`） | 阻塞机制完全相同，只是 Rust 把调用点藏进了迭代器协议里 |

### `for stream in listener.incoming()` 展开后的样子
```rust
let mut iter = listener.incoming();
loop {
    match iter.next() {          // 阻塞点在这里，内部调用 accept()
        Some(result) => { let stream = result; /* 循环体 */ }
        None => break,
    }
}
```

---

## 5. 并发模型：多线程处理连接

C（`pthread`）和 Rust（`std::thread`）用的是同一种骨架——**thread-per-connection**：主循环阻塞在 accept/incoming，来一个连接开一个线程处理，主线程立刻回去接下一个。

### 5.1 跨线程传递数据：C 的 `malloc` vs Rust 的 `move`

**C：为什么必须 malloc**
```c
int *fd_ptr = malloc(sizeof(int));
*fd_ptr = client_fd;
pthread_create(&tid, NULL, handle_client, fd_ptr);
```
- `pthread_create` 只能传一个 `void *` 指针。
- 如果直接传主线程栈上 `client_fd` 的地址：主线程会立刻进入下一轮循环，**复用同一块栈内存**，新线程可能读到已被覆盖的错误值（真实的竞态条件）。
- 解决办法：在**堆**上单独 `malloc` 一块内存，值拷贝进去，传这块堆内存的地址；生命周期不受栈帧影响，子线程用完后自己 `free()`。

**Rust：为什么不需要 malloc**
```rust
thread::spawn(move || {
    handle_connection(stream)
});
```
- `move` 把 `stream` 的**所有权**整体搬进闭包，而不是只传一个地址。
- 编译器在编译期保证：一旦 `move`，外层变量立刻失效，不允许再访问，因此不可能出现"两边同时读写同一块内存"的情况。
- `thread::spawn` 要求闭包满足 `'static` 生命周期，`move` 把"借用"换成"整体转移"，脱钩外层作用域的生命周期，满足这个要求。
- 实际上闭包捕获的数据最终会被移动到**堆**上（因为要跨线程独立存活），这一步与 C 的 `malloc` 本质上解决的是同一个问题，只是由编译器自动完成。

**所有权流动的正确心智模型：接力棒，不是并存的多份拷贝**
```
for 循环变量 stream（0x1000, fd:3）
   │ move（一移动，旧变量立即失效）
   ▼
match Ok(stream) 解构出的 stream（0x3000, fd:3）
   │ move 进闭包（thread::spawn 要求）
   ▼
闭包捕获环境（临时栈位置）
   │ 真正搬到堆上
   ▼
闭包在堆上的最终位置 —— 新线程独立拥有
```
- 循环下一轮，`for` 变量通常复用同一块栈地址（如仍是 0x1000），装入新连接的 fd；这是安全的，因为上一轮的值早已被 move 到堆上，覆盖旧地址不会丢数据。
- 这正好是 C 语言那个竞态条件问题的"安全版"：Rust 编译器保证"搬家"一定发生在"新数据写入"之前，顺序错乱的写法根本编译不过。

### 5.2 局部变量 vs 全局变量：什么时候需要加锁

| 变量 | 声明位置 | 存储位置 | 是否需要加锁 |
|---|---|---|---|
| `request_count`（每条连接处理了几个请求） | 函数内部局部变量 | 每个线程各自独立的栈 | **不需要**，各线程互不相通 |
| `thread_count`（当前活跃线程总数） | 函数外部全局变量 | 所有线程共享的一块内存 | **需要**，不加锁会有竞态条件（多个线程同时读改写同一地址） |

判断标准只有一个：**这块内存是否被多个线程同时能访问到**。

---

## 6. 资源清理：手动 close vs 自动 Drop

- C：必须显式 `close(client_fd)`，忘记写就是**文件描述符泄漏**——不会立刻崩溃，但长期运行后会因 fd 耗尽而拒绝新连接。
- Rust：`TcpStream` 实现了 `Drop` trait，变量离开作用域时编译器自动插入清理代码（底层依然是调用 `close()`），不需要手写。

---

## 7. 错误处理：返回值判断 vs Result 类型

- C：函数返回 `-1` 表示出错，是"约定俗成"，**编译器不强制检查**，忘记判断也能正常编译，运行时才会出现诡异问题。
- Rust：`Result<TcpStream, io::Error>` 是类型系统的一部分，不处理（`match`/`?`/`.unwrap()`）**根本无法编译通过**——是编译期强制检查 vs 运行时可能被疏忽的规范。

---

## 8. 请求解析与 keep-alive

### 8.1 `read()` 的真实行为

```c
int n = read(client_fd, buffer, sizeof(buffer) - 1);
```
- 第三个参数只是"上限"，不是"一定读这么多"；`n`（实际读到的字节数）可能远小于这个上限。
- 只要内核接收缓冲区里有数据（哪怕只有几个字节），`read()` 就会立刻返回，不会等凑够指定字节数。
- TCP 是字节流协议，没有"消息边界"概念：一个 HTTP 请求可能被拆成多次 `read()` 才能收全，也可能一次 `read()` 里混着下一个请求的开头。
- 返回值含义：`> 0` 读到数据；`= 0` 对端正常关闭；`< 0` 出错。

Rust 的 `BufReader` 内部自动循环调用底层 `read()`，直到凑够"一行"或满足解析条件才返回给上层，业务代码不必关心"这次有没有读全"。

### 8.2 `write()` 的真实行为

```c
write(client_fd, response, strlen(response));
```
- 把数据从进程内存拷贝进该 socket 的**发送缓冲区**，通常拷贝完就返回，不等数据真正经网卡发出（内核异步处理发送）。
- 同样**不保证一次写完全部字节**（对方接收缓冲区可能已满），严谨写法需要循环写入直到全部发送完成。
- 用 `strlen()` 计算长度的风险：如果响应体是二进制数据（图片等），中间恰好出现字节值 0，会被提前截断。更严谨的做法是从构造响应开始就显式记录字节数，而不是依赖扫描 `\0`。

### 8.3 keep-alive 的判断逻辑（C 与 Rust 对应）

```
client_wants_keep_alive = 请求头里没有 "Connection: close"
server_forces_close     = 这条连接已处理的请求数 >= 上限（如 100）
keep_alive = client_wants_keep_alive && !server_forces_close
```
两边都同意留，才真的留；任何一边想关，就关。

| Rust | C |
|---|---|
| `loop { ... }` | `while (1) { ... }` |
| `Request::parse(&mut reader)?` | `read(client_fd, buffer, ...)`（简化：假设一次读完，实际需处理多次读取拼接） |
| `should_keep_alive(&req)` | `should_keep_alive(buffer)` |
| `response.write_to(...)` | `write(client_fd, response, ...)` |
| `stream` 离开作用域自动 `Drop` | 循环结束后手动 `close(client_fd)` |
| `request_count` 声明在 `loop` 外，跨轮次累加 | `request_count` 声明在 `while` 外，同理 |

---

## 9. 内存与指针细节回顾

### 9.1 字符串结尾符 `\0` 需要手动补

`read()` 读进来的是原始字节，不会自带结尾符：
```c
buffer[n] = '\0';
```
不补的话，`%s`、`strstr` 等字符串函数会一直往后读，直到内存里偶然出现的 0 字节，可能读出乱码或导致崩溃。

### 9.2 指针减法：`line_end - buffer`

`strstr(buffer, "\r\n")` 返回指向 `\r\n` 起始位置的指针（`line_end`）。指针相减得到两个地址之间隔了多少个元素：

```
地址:   0x1000 ... 0x100D(最后一个'1')  0x100E('\r')  0x100F('\n')
buffer 指向 0x1000，line_end 指向 0x100E
line_end - buffer = 14  → 正好是请求行 "GET / HTTP/1.1" 的字符数
```

### 9.3 数组名退化为指针，`[]` 是 `*(s+n)` 的语法糖

```c
s[count]  等价于  *(s + count)
```
- `s + count`：地址运算，算出第 count 个字符的地址。
- `*(...)`：解引用，取出该地址存的值。
- `s[count]` 拿到的是**字符值**，不是地址；`while (s[count] != '\0')` 比较的是"取出来的字符"与 `'\0'`，不是地址比较。

`strlen` 内部本质：
```c
int strlen(const char *s) {
    int count = 0;
    while (s[count] != '\0') count++;
    return count;
}
```

### 9.4 字符串字面量存在只读数据段

```c
const char *body = "Hello from C server!";
```
- `"Hello..."` 存在**只读数据段**，程序加载时就固定好，不在栈也不在堆。
- `body`（指针变量本身）在栈上，占 8 字节，存的值是这段字符串的起始地址。
- `const` 提醒/强制不能通过 `body` 修改这些字符，强行写会导致运行时崩溃（段错误）。

### 9.5 栈上数组大小是"拍脑袋定的"，需警惕缓冲区溢出

```c
char request_line[256];
strncpy(request_line, buffer, line_len);
```
- `request_line` 是局部变量，在**栈**上开辟，函数/作用域结束自动回收。
- `256` 只是经验值，`strncpy` **不会自动检查目标数组是否够大**；如果 `line_len > 256`，会直接覆盖相邻栈内存（缓冲区溢出，C 语言最常见的安全隐患之一）。
- 严谨写法：拷贝前判断 `line_len` 是否超过容量，超过则拒绝处理。
- 对比 Rust：`String`/`Vec<u8>` 长度动态、自动扩容，不需要预先猜测"够不够用"。

---

## 10. 完整 C 服务器示例（含 keep-alive，简化版）

要点提示：以下代码将请求解析简化为"假设一次 `read()` 能读到完整请求"，未处理请求被拆成多次到达的情况，仅用于演示 keep-alive 循环结构。

```c
#define _GNU_SOURCE   // strcasestr 需要，必须放在所有 #include 之前
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define MAX_THREADS 10
#define PORT 8080
#define BUFFER_SIZE 4096
#define MAX_REQUESTS_PER_CONN 100

int thread_count = 0;
pthread_mutex_t count_lock = PTHREAD_MUTEX_INITIALIZER;

int should_keep_alive(const char *request_headers)
{
    if (strcasestr(request_headers, "Connection: close") != NULL) return 0;
    return 1;
}

void handle_connection(int client_fd)
{
    char buffer[BUFFER_SIZE];
    int request_count = 0;

    while (1)
    {
        memset(buffer, 0, sizeof(buffer));
        int n = read(client_fd, buffer, sizeof(buffer) - 1);
        if (n <= 0) break;

        request_count++;

        int client_wants_keep_alive = should_keep_alive(buffer);
        int server_forces_close = (request_count >= MAX_REQUESTS_PER_CONN);
        int keep_alive = client_wants_keep_alive && !server_forces_close;

        char response[BUFFER_SIZE];
        const char *body = "Hello from C server!";
        int body_len = strlen(body);
        const char *connection_header = keep_alive
            ? "Connection: keep-alive\r\n"
            : "Connection: close\r\n";

        snprintf(response, sizeof(response),
                 "HTTP/1.1 200 OK\r\n"
                 "Content-Length: %d\r\n"
                 "%s"
                 "\r\n"
                 "%s",
                 body_len, connection_header, body);

        write(client_fd, response, strlen(response));

        if (!keep_alive) break;
    }

    close(client_fd);
}

void *handle_client(void *arg)
{
    int client_fd = *(int *)arg;
    free(arg);

    handle_connection(client_fd);

    pthread_mutex_lock(&count_lock);
    thread_count--;
    pthread_mutex_unlock(&count_lock);

    return NULL;
}

int main()
{
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) { perror("socket 创建失败"); return 1; }

    struct sockaddr_in address;
    memset(&address, 0, sizeof(address));
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) == -1) {
        perror("bind 失败"); return 1;
    }
    if (listen(server_fd, 10) == -1) { perror("listen 失败"); return 1; }

    while (1)
    {
        struct sockaddr_in client_addr;
        socklen_t addr_len = sizeof(client_addr);
        int client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &addr_len);
        if (client_fd == -1) { perror("accept 失败"); continue; }

        pthread_mutex_lock(&count_lock);
        if (thread_count >= MAX_THREADS) {
            pthread_mutex_unlock(&count_lock);
            close(client_fd);
            continue;
        }
        thread_count++;
        pthread_mutex_unlock(&count_lock);

        int *fd_ptr = malloc(sizeof(int));
        *fd_ptr = client_fd;

        pthread_t tid;
        pthread_create(&tid, NULL, handle_client, fd_ptr);
        pthread_detach(tid);
    }

    close(server_fd);
    return 0;
}
```

---

## 11. 核心心得

C 语言把所有内存管理、并发安全、缓冲区大小、错误检查的细节，全部交给程序员自己负责——写起来繁琐，但每一步发生了什么都清清楚楚，不存在"看不懂的黑盒"。

Rust 把大量容易出错的手动管理（缓冲区、跨线程数据传递、资源释放）封装进类型系统和标准库内部（`BufReader`、所有权/`move`、`Drop`），换来更简洁、编译期就能拦截大量错误的代码，但代价是标准库内部具体怎么实现，需要额外去读源码才能真正理解，不像 C 那样"写出来的就是全部真相"。

两种语言用的并发骨架（thread-per-connection）、协议细节（TCP 三次握手、HTTP keep-alive 语义）本质上是完全一致的——语言只是提供了不同的"心智负担该由谁来背"的取舍。
