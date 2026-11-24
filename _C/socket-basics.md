# Socket 编程核心概念梳理

> 一句话理解：所有这些系统调用（`socket` / `bind` / `listen` / `accept` / `read` / `write`）都不是在操作"网络"本身，而是在指挥**内核**去创建、修改、读写一些**内核内部维护的数据结构**。你的程序全程只是在跟内核打交道，从没直接碰过网卡或网线。

---

## 一、总览：谁负责什么

| 层面 | 负责的角色 |
|---|---|
| 你的进程 | 只持有一个个整数句柄（fd），通过系统调用"指挥"内核干活 |
| 内核（OS） | 真正创建、维护 socket 对象，管理缓冲区、连接队列、协议状态 |
| 网卡 + TCP/IP 协议栈 | 真正完成数据的收发、拆包、握手、重传等物理/协议层工作 |

**核心心智模型**：`fd`（文件描述符）只是一个"门牌号"，真正的对象和状态都在内核里，你的程序看不见摸不着，只能通过系统调用去读写它。

---

## 二、`socket()` —— 创建一个内核对象

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
```

- 你的进程向内核发起系统调用
- 内核在自己的内存空间里创建一个"通信端点"对象，记录：
  - 协议族（IPv4 / IPv6）
  - 类型（TCP / UDP）
  - 状态（此时是"未绑定"）
  - 一对缓冲区（此时为空，还未启用）
- 内核把这个对象登记进你进程的**文件描述符表**，返回一个整数编号（`server_fd`）

```
你的进程                         内核
┌────────────┐            ┌────────────────────┐
│ server_fd=3 │ ─────────▶ │ fd表[3] → socket对象 │
└────────────┘            │  { 协议族, 类型,       │
                           │    状态: 未绑定 }      │
                           └────────────────────┘
```

```c
// 这不是真实的内核代码，是帮助理解的简化模型
struct socket_内核对象 {

    // ===== 1. 身份信息（socket() 调用时就确定，之后不变）=====
    int family;      // AF_INET (IPv4) / AF_INET6 (IPv6) / AF_UNIX (本机进程间) ...
    int type;         // SOCK_STREAM (TCP) / SOCK_DGRAM (UDP)
    int protocol;     // 具体协议号，通常自动推导

    // ===== 2. 地址信息（bind()/connect()/accept() 会填）=====
    struct sockaddr_in local_addr;    // 本地地址+端口，bind() 填的就是这个
    struct sockaddr_in remote_addr;   // 对端地址+端口，只有"已连接"的 socket 才有值
                                        // 一个 LISTENING 状态的 socket，这里是空的

    // ===== 3. 状态（TCP 状态机，一直在变）=====
    enum tcp_state state;   
    // CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → FIN_WAIT → ... → CLOSED
    // 我们图里画的"LISTENING"/"ESTABLISHED"，就是这个字段的值

    // ===== 4. 缓冲区（收发数据真正存放的地方）=====
    ring_buffer recv_buffer;   // 接收缓冲区，read() 从这里取数据
    ring_buffer send_buffer;   // 发送缓冲区，write() 的数据先堆在这里
    size_t recv_buf_size;      // 缓冲区容量，对应 SO_RCVBUF 这个选项
    size_t send_buf_size;      // 对应 SO_SNDBUF

    // ===== 5. 只有"监听中"的 socket 才有的字段 =====
    int backlog;                       // listen() 第二个参数，队列最大长度
    queue<connection> accept_queue;    // 已握手完成、等 accept() 取走的连接队列

    // ===== 6. TCP 协议本身需要的一堆状态（真实内核里非常庞大）=====
    uint32_t seq_num;        // 序列号，保证数据按顺序到达
    uint32_t ack_num;        // 确认号
    int cwnd;                 // 拥塞窗口，控制发送速度，避免网络拥堵
    int rtt;                  // 往返时延估计，用来算超时重传的时间
    timer retransmit_timer;   // 超时没收到 ACK，触发重传

    // ===== 7. 一些可配置选项（setsockopt() 用来改）=====
    bool reuse_addr;   // SO_REUSEADDR，允许快速重新绑定同一个地址
    bool keep_alive;   // SO_KEEPALIVE，定期探测连接是否还活着
    bool no_delay;     // TCP_NODELAY，关闭 Nagle 算法，减少小包延迟

    // ===== 8. 内核内部管理用（跟"业务逻辑"没关系，但真实存在）=====
    int ref_count;     // 引用计数：有几个 fd 指向这个 socket（比如 fork 之后父子进程共享）
    spinlock lock;      // 保护这个结构体不被多个 CPU 核心同时修改出错
};
```

---

## 三、`bind()` —— 把地址"记"到内核对象上

```c
bind(server_fd, (struct sockaddr *)&address, sizeof(address));
```

- **谁绑定**：进程发起调用，内核执行
- **绑定的对象**：`server_fd` 指向的那个 socket 对象
- **绑定到什么**：`address` 里描述的 IP + 端口（如 `0.0.0.0:8080`）

```
调用前： socket对象 { 本地地址: 空 }
调用后： socket对象 { 本地地址: 0.0.0.0:8080 }
```

内核会顺带检查端口有没有被占用，占用则 `bind` 失败（"Address already in use"）。

**关于强制类型转换 `(struct sockaddr *)`**：
`bind()` 接口设计成接收通用地址类型 `sockaddr *`（因为要同时支持 IPv4/IPv6/Unix Socket），而你实际填的是具体类型 `sockaddr_in`。C 语言约定这些具体地址结构体的**开头字段内存布局**与通用结构体保持一致，所以可以放心强转指针类型，再靠 `sizeof(address)` 告诉内核这块内存到底多大，帮它正确解析。这是 C 语言里手工模拟"多态"的经典手法。

---

## 四、`listen()` —— 把对象状态改为"监听中"

```c
listen(server_fd, 10);
```

- **谁监听**：内核
- **监听什么**：本机 8080 端口上，有没有新的 TCP 连接请求（三次握手）到达
- 第二个参数 `10`：等待队列的长度上限（已完成握手、但还未被 `accept()` 取走的连接数上限）

```
socket对象 {
  状态: LISTEN
  本地地址: 0.0.0.0:8080
  等待队列: []   ← 新握手完成的连接会被放进这里
}
```

之后只要有客户端发起连接，**网卡 + 内核协议栈**会自动完成三次握手，无需应用层介入。

---

## 五、`accept()` —— 从队列里取出一个连接

```c
int client_fd = accept(server_fd, NULL, NULL);
```

- **不创建线程**，本身只是一次普通的（默认阻塞的）系统调用
- 内核检查等待队列：
  - 队列为空 → 阻塞，程序卡在这一行，直到有新连接
  - 队列非空 → 取出一个，**创建一个新的 socket 对象**，专门服务这个客户端，返回新 fd（`client_fd`）

调用完成后，你手里有两个 socket：

```
server_fd → 状态 LISTEN，继续监听 8080，等下一个新客户
client_fd → 状态 ESTABLISHED，专门跟这一个客户端收发数据
```

### 并发模型是应用层自己的选择

`accept()` 只负责"取出一个连接"，**至于怎么同时处理多个客户端，是你自己的程序设计问题**，常见模式：

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
    pthread_create(&tid, NULL, handle_client_thread, &client_fd); // 每连接一线程
    // 主线程立刻回去 accept 下一个
}
```

对应 Rust：

| C | Rust |
|---|---|
| `socket+bind+listen` | `TcpListener::bind()` |
| 一次 `accept()` | `listener.incoming()` 迭代出的一个 `stream` |
| `pthread_create()` | `thread::spawn()` |
| `client_fd`（整数句柄） | `TcpStream`（安全封装对象） |

> 生产环境更常用**异步 I/O**（`epoll` / `io_uring`，对应 Rust 的 `tokio`）而非"每连接一线程"，因为线程开销在高并发下会成为瓶颈。

---

## 六、`read()` / `write()` —— 读写的是内核缓冲区，不是"网络"本身

`accept()` 返回的 `client_fd` 对应的 socket 对象，内部包含**两个独立缓冲区**：

```
client_fd 指向的内核对象：
┌─────────────────────────────┐
│ 接收缓冲区 receive buffer     │ ← 对方发来的数据，暂存在这
├─────────────────────────────┤
│ 发送缓冲区 send buffer        │ ← 我方要发的数据，暂存在这
└─────────────────────────────┘
```

- `read(client_fd, buf, ...)`：把**接收缓冲区**的数据**拷贝**到你程序的 `buf` 内存。数据是网卡收到后由**内核协议栈**自动拆包放进去的，与你的程序无关。
- `write(client_fd, data, ...)`：把你程序 `data` 内存的数据**拷贝**进**发送缓冲区**，内核再在合适时机通过网卡发出。

### 完整数据流向图

```
服务端进程          内核(服务端)            内核(客户端)          客户端进程
┌────────┐      ┌──────────────┐      ┌──────────────┐      ┌────────┐
│  buf   │◀read─│接收缓冲区      │◀网络──│发送缓冲区      │◀write─│request │
│response│─write▶│发送缓冲区      │─网络─▶│接收缓冲区      │─read─▶│  buf   │
└────────┘      └──────────────┘      └──────────────┘      └────────┘
```

客户端和服务端是完全对称的结构：都各自持有一个 socket fd，内核都为其维护一套"接收 + 发送"缓冲区。

---

## 七、统一规律总结

| 系统调用 | 表面看起来做什么 | 实际在做什么 |
|---|---|---|
| `socket()` | 创建一个 socket | 内核**创建**一个通信端点对象，返回 fd |
| `bind()` | 绑定地址 | 把地址信息**写进**内核对象的字段 |
| `listen()` | 开始监听 | 把内核对象状态**改为** LISTEN，准备等待队列 |
| `accept()` | 接受连接 | 从队列**取出**已握手连接，**新建**一个内核对象 |
| `read()` | 读数据 | 从内核对象的接收缓冲区**拷贝**到用户内存 |
| `write()` | 写数据 | 把用户内存的数据**拷贝**进内核对象的发送缓冲区 |

**Berkeley Socket API 的本质**：把复杂的网络协议栈（握手、分包、重传、拥塞控制）抽象封装成"像操作文件一样 open/read/write/close"的统一接口，这也是为什么 Linux 里 socket 也被视为一种特殊的"文件"。

---

## 附：调试踩坑记录

1. **`close()` 报 "implicit declaration"**：忘记 `#include <unistd.h>`。
2. **`printf("hello")` 不换行，终端看起来"卡住"**：`stdout` 连终端时是**行缓冲**，没有 `\n` 触发刷新时，内容会一直憋在缓冲区里不显示。解决：加 `\n`，或手动 `fflush(stdout)`。
3. **浏览器报 `ERR_EMPTY_RESPONSE`**：说明 TCP 连接其实已经成功建立，只是服务端没有返回任何 HTTP 格式的数据就 `close()` 了连接——这是正常现象，因为还没写响应逻辑，不代表网络故障。
4. **WSL2 中 Windows 浏览器访问不到服务**：可尝试
   - 先在 WSL 内部用 `curl localhost:8080` 自测
   - 用 `hostname -I` 拿到 WSL 的实际 IP，浏览器直接访问该 IP
   - 检查 Windows 防火墙是否拦截了入站连接
