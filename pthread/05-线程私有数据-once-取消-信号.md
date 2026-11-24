# 05 线程私有数据、once、取消、信号与其它

## 一、线程私有数据(TSD / TLS)

让每个线程拥有自己的"全局变量"副本,典型用途:每线程的缓冲区、`errno`、上下文对象。

### 函数一览

| 函数 | 作用 |
|---|---|
| `pthread_key_create(&key, destructor)` | 创建键,可指定析构函数 |
| `pthread_setspecific(key, value)` | 设置当前线程在该键下的值 |
| `pthread_getspecific(key)` | 获取当前线程在该键下的值,未设置返回 `NULL` |
| `pthread_key_delete(key)` | 删除键(不会调用析构函数) |

线程退出时,若某个键的值非 `NULL`,系统会以该值调用析构函数,用于释放资源。

### 示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>

static pthread_key_t key;
static pthread_once_t key_once = PTHREAD_ONCE_INIT;

static void destructor(void *p) {
    printf("destructor frees buffer of thread: %s\n", (char *)p);
    free(p);
}

static void make_key(void) {
    pthread_key_create(&key, destructor);
}

static char *get_buffer(void) {
    pthread_once(&key_once, make_key);           // 保证 key 只创建一次
    char *buf = pthread_getspecific(key);
    if (!buf) {
        buf = malloc(64);
        pthread_setspecific(key, buf);
    }
    return buf;
}

static void *worker(void *arg) {
    snprintf(get_buffer(), 64, "thread-%ld", (long)arg);   // 每个线程独立的 buffer
    printf("%s uses its own buffer\n", get_buffer());
    return NULL;
}

int main(void) {
    pthread_t t[3];
    for (long i = 0; i < 3; i++) pthread_create(&t[i], NULL, worker, (void *)i);
    for (int i = 0; i < 3; i++) pthread_join(t[i], NULL);
    return 0;
}
```

### 更简单的方式:`__thread` / `_Thread_local`

```c
#include <stdio.h>
#include <pthread.h>

static _Thread_local int tls_counter = 0;        // C11;GCC 也可写 __thread int

static void *worker(void *arg) {
    long id = (long)arg;
    for (int i = 0; i < 3; i++) tls_counter++;
    printf("thread %ld: tls_counter = %d\n", id, tls_counter);   // 每个线程都是 3
    return NULL;
}

int main(void) {
    pthread_t t[2];
    for (long i = 0; i < 2; i++) pthread_create(&t[i], NULL, worker, (void *)i);
    for (int i = 0; i < 2; i++) pthread_join(t[i], NULL);
    return 0;
}
```

| 对比 | `pthread_key_*` | `__thread` / `_Thread_local` |
|---|---|---|
| 析构函数 | 支持 | 不支持(C 中) |
| 动态数量 | 可运行时创建 | 编译期固定 |
| 速度 | 较慢(函数调用) | 快(直接寻址) |
| 适用 | 需要自动清理、库代码 | 简单变量、性能敏感 |

---

## 二、一次性初始化 `pthread_once`

```c
int pthread_once(pthread_once_t *once, void (*init_routine)(void));
```

无论多少线程同时调用,`init_routine` 只执行一次,且其它线程会等它执行完。适合懒初始化单例、全局资源、TSD 键。

```c
#include <stdio.h>
#include <pthread.h>

static pthread_once_t once = PTHREAD_ONCE_INIT;
static int resource;

static void init_resource(void) {
    printf("init resource (only once)\n");
    resource = 42;
}

static void *worker(void *arg) {
    (void)arg;
    pthread_once(&once, init_resource);
    printf("resource = %d\n", resource);
    return NULL;
}

int main(void) {
    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
    return 0;
}
```

> C11 也提供了 `call_once`(`<threads.h>`),但 glibc 的 `<threads.h>` 支持较新,pthread 版本兼容性更好。

---

## 三、线程取消与清理

### 函数一览

| 函数 | 作用 |
|---|---|
| `pthread_cancel(tid)` | 向目标线程发送取消请求(异步请求,不保证立即终止) |
| `pthread_setcancelstate(state, &old)` | `PTHREAD_CANCEL_ENABLE`(默认)/ `PTHREAD_CANCEL_DISABLE` |
| `pthread_setcanceltype(type, &old)` | `PTHREAD_CANCEL_DEFERRED`(默认)/ `PTHREAD_CANCEL_ASYNCHRONOUS` |
| `pthread_testcancel()` | 手动设置一个取消点 |
| `pthread_cleanup_push(fn, arg)` / `pthread_cleanup_pop(execute)` | 注册/弹出清理处理函数(**是宏,必须在同一作用域成对出现**) |

### 取消的工作方式

- **延迟取消(默认)**:线程运行到"取消点"时才响应。取消点包括 `sleep`、`read`、`write`、`pthread_cond_wait`、`pthread_join`、`pthread_testcancel` 等阻塞型函数。
- **异步取消**:任意时刻都可能被终止,极易导致锁未释放、内存泄漏,**几乎不要使用**。
- 被取消的线程会依次执行清理处理函数,然后以返回值 `PTHREAD_CANCELED` 结束(`pthread_join` 可获得)。

### 示例:取消一个循环线程并正确清理

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

static pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;

static void cleanup_unlock(void *arg) {
    printf("cleanup: unlock mutex\n");
    pthread_mutex_unlock(arg);
}

static void cleanup_free(void *arg) {
    printf("cleanup: free memory\n");
    free(arg);
}

static void *worker(void *arg) {
    (void)arg;
    char *buf = malloc(128);
    pthread_cleanup_push(cleanup_free, buf);         // 后进先出执行

    pthread_mutex_lock(&mu);
    pthread_cleanup_push(cleanup_unlock, &mu);

    for (;;) {
        printf("working...\n");
        sleep(1);                                    // sleep 是取消点
    }

    pthread_cleanup_pop(1);                          // 参数非 0:弹出并执行
    pthread_cleanup_pop(1);                          // 必须配对,即使代码不可达
    return NULL;
}

int main(void) {
    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    sleep(2);

    pthread_cancel(t);

    void *ret;
    pthread_join(t, &ret);
    if (ret == PTHREAD_CANCELED)
        printf("thread was canceled\n");
    return 0;
}
```

### 临时屏蔽取消

```c
int old;
pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, &old);
/* 关键操作:不会被取消 */
pthread_setcancelstate(old, NULL);
pthread_testcancel();                                // 恢复后主动检查是否有挂起的取消请求
```

### 更推荐的做法:协作式退出

实际工程中,比起 `pthread_cancel`,更常用**原子标志 + 条件变量**通知线程自行退出,这样退出点可控、没有隐藏的资源泄漏:

```c
#include <stdio.h>
#include <stdatomic.h>
#include <stdbool.h>
#include <unistd.h>
#include <pthread.h>

static atomic_bool stop = false;

static void *worker(void *arg) {
    (void)arg;
    while (!atomic_load(&stop)) {
        printf("working...\n");
        usleep(300 * 1000);
    }
    printf("worker exits gracefully\n");
    return NULL;
}

int main(void) {
    pthread_t t;
    pthread_create(&t, NULL, worker, NULL);
    sleep(1);
    atomic_store(&stop, true);
    pthread_join(t, NULL);
    return 0;
}
```

---

## 四、线程与信号

规则要点:

- 信号处理函数(handler)是**进程级别**共享的,但信号掩码(mask)是**线程级别**的。
- 发给进程的信号(如 `kill`、Ctrl+C)会由内核挑选一个**未屏蔽该信号**的线程来处理。
- 多线程程序处理信号的推荐模式:**主线程屏蔽所有信号,专设一个线程用 `sigwait` 同步等待**,避免在异步 handler 里做复杂操作。

### 函数一览

| 函数 | 作用 |
|---|---|
| `pthread_sigmask(how, set, old)` | 设置当前线程的信号掩码(多线程中用它代替 `sigprocmask`) |
| `pthread_kill(tid, sig)` | 向指定线程发送信号 |
| `sigwait(set, &sig)` | 同步等待信号集中的任一信号 |

### 示例:专用信号处理线程

```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
#include <pthread.h>

static void *signal_thread(void *arg) {
    sigset_t *set = arg;
    int sig;
    for (;;) {
        sigwait(set, &sig);                          // 同步等待,可以安全地调用任何函数
        if (sig == SIGINT) {
            printf("\ngot SIGINT, shutting down\n");
            return NULL;
        } else if (sig == SIGUSR1) {
            printf("got SIGUSR1, reload config\n");
        }
    }
}

int main(void) {
    static sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGUSR1);

    pthread_sigmask(SIG_BLOCK, &set, NULL);          // 必须在创建其它线程之前!子线程继承掩码

    pthread_t st;
    pthread_create(&st, NULL, signal_thread, &set);

    printf("pid=%d, try: kill -USR1 %d, or press Ctrl+C\n", getpid(), getpid());
    pthread_join(st, NULL);
    return 0;
}
```

---

## 五、fork 与 pthread_atfork

多线程程序 `fork` 后,子进程**只有调用 fork 的那个线程**,其它线程凭空消失,若它们当时持有锁,子进程中该锁将永远无人释放。

```c
int pthread_atfork(void (*prepare)(void),   // fork 前,在父进程调用
                   void (*parent)(void),    // fork 后,在父进程调用
                   void (*child)(void));    // fork 后,在子进程调用
```

典型用法:`prepare` 里把关键锁都加上,`parent` 和 `child` 里解锁,保证 `fork` 时锁处于一致状态。最稳妥的原则仍然是:**多线程程序中 fork 之后立刻 exec**。

---

## 六、CPU 亲和性与调度(Linux)

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <sched.h>
#include <pthread.h>

static void *worker(void *arg) {
    (void)arg;
    printf("running on CPU %d\n", sched_getcpu());
    return NULL;
}

int main(void) {
    pthread_t t;
    pthread_attr_t attr;
    pthread_attr_init(&attr);

    cpu_set_t cpus;
    CPU_ZERO(&cpus);
    CPU_SET(0, &cpus);                               // 绑定到 CPU 0
    pthread_attr_setaffinity_np(&attr, sizeof cpus, &cpus);

    pthread_create(&t, &attr, worker, NULL);
    pthread_join(t, NULL);

    // 也可在运行中绑定:pthread_setaffinity_np(t, sizeof cpus, &cpus);
    pthread_attr_destroy(&attr);
    return 0;
}
```

调度策略相关:

| 函数 | 作用 |
|---|---|
| `pthread_setschedparam(tid, policy, &param)` | 设置策略和优先级(`SCHED_FIFO`/`SCHED_RR` 需要 root 或 `CAP_SYS_NICE`) |
| `pthread_getschedparam` | 获取 |
| `pthread_setschedprio` | 仅修改优先级 |
| `pthread_getcpuclockid` | 获取线程 CPU 时间时钟,配合 `clock_gettime` 统计线程 CPU 用时 |

---

## 小结

- 线程独有数据:简单变量用 `_Thread_local`,需要析构用 `pthread_key_*`。
- 单次初始化用 `pthread_once`。
- 避免 `pthread_cancel`(尤其异步取消),使用协作式退出;必须用时配合 `pthread_cleanup_push/pop`。
- 多线程里信号:屏蔽 + 专用线程 `sigwait`。
- 多线程中谨慎 `fork`。
