# C 语言手写异步模型（二）—— 用定时器模拟 JS 的 async/await

> 承接上一份笔记（事件循环 + select 监听 stdin）。这一份讲的是同一套骨架的变体：把"监听 fd"换成"监听时间点"，模拟出 JS 里 `f1(); f2(); console.log(...)` 这种"函数提前返回、后半段延后执行"的效果。

---

## 一、从一个 JS 现象说起

在 JS 里写：

```js
async function f1() {
    console.log("f1 开始");
    await new Promise(resolve => setTimeout(resolve, 1000)); // 等1秒
    console.log("f1 恢复执行");
}

async function f2() {
    console.log("f2 开始");
    await new Promise(resolve => setTimeout(resolve, 500)); // 等0.5秒
    console.log("f2 恢复执行");
}

f1();
f2();
console.log("logxxxx");
```

实际打印顺序：

```
f1 开始
f2 开始
logxxxx
f2 恢复执行     ← 0.5秒后
f1 恢复执行     ← 1秒后
```

**关键现象**：`f1()`、`f2()` 里 `await` **之前**的代码是立刻同步执行的；执行到 `await` 那一行，函数就"提前返回"了，`await` **之后**的代码被延后到将来某个时间点才执行。所以 `logxxxx` 能抢在两个"恢复执行"之前打印。

**核心问题**：C 语言里，一行会阻塞的代码，能不能"跳过去，先执行后面的，以后再回来接着跑"？

**答案**：不能真的"跳过一行代码，以后回来接着跑剩下的部分"——CPU 执行指令是严格顺序的，没有"记住某一行还欠着"这种能力（除非借助线程、协程等专门机制）。但可以通过**把函数从"该等待的点"拆成两半**，达到同样的效果：前一半（登记"以后要干什么"）立刻返回，后一半（真正的处理逻辑）被存起来，等条件满足了才被调用。JS 的 `async/await` 正是编译器/引擎自动帮你做了这个"拆函数"的动作；下面我们在 C 里手动实现同样的效果。

---

## 二、时间基础概念（这是这一版新增的核心知识点）

### 2.1 计算机里的时间是什么

计算机内部存的"时间"不是日历格式，而是一个纯数字：

> **从 1970年1月1日 0点0分0秒（约定的"元年"）开始，到"现在"，一共过去了多少秒。**

这串数字叫 **Unix 时间戳**。本笔记为了方便手算，把"现在"假想成一个较小的数字，比如"第1000秒"，不影响理解原理。

### 2.2 `struct timeval` —— 系统提供的时间结构体

```c
struct timeval {
    long tv_sec;    // 整数秒
    long tv_usec;   // 微秒（不满一秒的零头，精确到百万分之一秒）
};
```

**这个结构体不是我们定义的**，是系统头文件（`<sys/time.h>`，通常经由 `<sys/select.h>` 间接引入）里已经写好的，我们只是 `#include` 之后拿来使用（声明变量、访问字段），从未自己写过 `struct timeval { ... };` 这几个字。

`#include` 的本质：把那个头文件的全部内容，相当于原封不动"复制粘贴"进你的 `.c` 文件里。想亲眼看这个结构体真实定义在哪个文件，可以在 WSL 里执行：

```bash
grep -rn "struct timeval" /usr/include/ 2>/dev/null | grep "tv_sec"
```

### 2.3 为什么要拆成"秒"和"微秒"两个字段

单独一个整数"秒数"，表示不了"1000秒又过了半秒"这种更精细的时刻。于是设计成两个字段配合：

- `tv_sec`：这个时间点是"第几秒"（整数部分）
- `tv_usec`：这一秒里面，又细分了多少"微秒"（1秒 = 1,000,000 微秒）

类比人民币"元 + 角/分"：

```
"1000.5秒" 拆成字段：
  tv_sec  = 1000     （整数秒部分）
  tv_usec = 500000    （零头部分：0.5秒 = 500000微秒）
```

**硬性规则**：`tv_usec` 只能存 `0 ~ 999999`。一旦累加后凑够了 1000000（凑满1秒），必须**进位**到 `tv_sec` 上——跟"攒够100分换1块钱"是同一个道理。这个规则是后面 `add_seconds` 里那段 `if` 判断存在的原因。

---

## 三、完整代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/select.h>
#include <sys/time.h>

// ---------- 定时器系统：模拟 JS 的 setTimeout ----------

typedef void (*callback_t)(void);

#define MAX_TIMERS 10

typedef struct {
    struct timeval expire_at;  // 绝对到期时间点
    callback_t callback;       // 到期了调用谁
    int active;                // 这条登记是否还有效（触发过一次就作废）
} timer_t_;

timer_t_ timers[MAX_TIMERS];
int timer_count = 0;

static struct timeval now(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv;
}

static struct timeval add_seconds(struct timeval base, double seconds) {
    long usec_offset = (long)(seconds * 1000000);
    base.tv_sec  += usec_offset / 1000000;
    base.tv_usec += usec_offset % 1000000;
    if (base.tv_usec >= 1000000) {
        base.tv_sec += 1;
        base.tv_usec -= 1000000;
    }
    return base;
}

void set_timeout(double delay_sec, callback_t cb) {
    timers[timer_count].expire_at = add_seconds(now(), delay_sec);
    timers[timer_count].callback  = cb;
    timers[timer_count].active    = 1;
    timer_count++;
}

void event_loop(void) {
    while (1) {
        int earliest = -1;
        for (int i = 0; i < timer_count; i++) {
            if (!timers[i].active) continue;
            if (earliest == -1 || timercmp(&timers[i].expire_at, &timers[earliest].expire_at, <)) {
                earliest = i;
            }
        }

        if (earliest == -1) break;

        struct timeval now_tv = now();
        struct timeval remaining;
        timersub(&timers[earliest].expire_at, &now_tv, &remaining);
        if (remaining.tv_sec < 0) {
            remaining.tv_sec = 0;
            remaining.tv_usec = 0;
        }

        select(0, NULL, NULL, NULL, &remaining);

        struct timeval now2 = now();
        for (int i = 0; i < timer_count; i++) {
            if (timers[i].active && timercmp(&timers[i].expire_at, &now2, <=)) {
                timers[i].active = 0;
                timers[i].callback();
            }
        }
    }
}

// ---------- 业务代码：对应 JS 里的 f1() / f2() ----------

void f1_rest(void) { printf("f1 恢复执行\n"); }

void f1(void) {
    printf("f1 开始\n");
    set_timeout(1.0, f1_rest);
}

void f2_rest(void) { printf("f2 恢复执行\n"); }

void f2(void) {
    printf("f2 开始\n");
    set_timeout(0.5, f2_rest);
}

int main(void) {
    f1();
    f2();
    printf("logxxxx\n");
    event_loop();
    return 0;
}
```

实测输出（在 WSL 里编译运行）：

```
f1 开始
f2 开始
logxxxx
f2 恢复执行
f1 恢复执行
```

---

## 四、逐个类型/变量详解

### 4.1 `callback_t` 签名的变化

```c
typedef void (*callback_t)(void);
```

对比上一版：

```c
typedef void (*callback_t)(int fd);   // 旧版本：回调需要一个int参数（要处理哪个fd）
```

这次改成 `(void)`（不带参数），因为定时器回调不需要知道"是哪个fd"，只需要在到期时执行该做的事。

### 4.2 `timer_t_` 结构体 —— 新的"登记卡"

```c
typedef struct {
    struct timeval expire_at;
    callback_t callback;
    int active;
} timer_t_;
```

对比上一版的 `event_t { int fd; callback_t callback; }`，多了两处：

- `expire_at`：不是"关心哪个fd"，而是"到几点几分几秒算到期"——**是一个绝对时间点，不是"还要等多久"这种相对值**。存绝对值的原因：每一轮循环都要重新计算"现在距离到期还有多久"，如果存的是相对值，每次都会被"用掉"一部分，容易出错；存绝对值，每次用"到期时刻 − 当前时刻"现算，永远准确。
- `active`：标志位，记录"这条登记是否还有效"。fd 的登记是可以反复触发的（旧模型每轮都检查），但定时器是"一次性"的——触发一次就该作废，不然会被重复选中、重复执行。

内存布局（假设 `struct timeval` 占16字节，指针8字节，`int` 4字节，算上对齐大约32字节一格，具体数字仅为示意）：

```
一个 timer_t_ 变量的内存布局（假设起始地址 0x4000）
┌────────┬─────────────────────┬──────┬──────────┐
│ 0x4000 │ expire_at.tv_sec     │ 8字节 │   ???     │
│ 0x4008 │ expire_at.tv_usec    │ 8字节 │   ???     │
│ 0x4010 │ callback              │ 8字节 │   ???     │
│ 0x4018 │ active                │ 4字节 │   ???     │
│ 0x401C │ (对齐填充)             │ 4字节 │  (无意义)  │
└────────┴─────────────────────┴──────┴──────────┘
```

### 4.3 `timers` 数组与 `timer_count`

```c
timer_t_ timers[MAX_TIMERS];
int timer_count = 0;
```

和上一版的 `events` 数组是同一套机制：一段连续内存，每格是一个 `timer_t_`；`timer_count` 是记账变量，因为 C 数组本身不知道自己"实际用了多少格"，必须手动记录。

---

## 五、`now()` 详解——函数定义与内存过程

### 5.1 这是函数，不是结构体

容易混淆的一点：`struct timeval` 是**类型**（结构体），`now` 是**函数**，它的返回类型恰好是 `struct timeval` 而已——"定义一个类型"和"写一个返回该类型的函数"是两件不同的事。

```c
static struct timeval now(void)
```

拆开读：

| 片段 | 含义 |
|---|---|
| `static` | 只能在本文件内部被调用 |
| `struct timeval` | **返回类型**：这个函数执行完会返回一个 `struct timeval` 值 |
| `now` | 函数名 |
| `(void)` | 不需要任何参数 |

### 5.2 函数体与内存过程

```c
static struct timeval now(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv;
}
```

- `struct timeval tv;`：在栈上声明一个变量 `tv`，此刻内部两个字段是垃圾数据。
- `gettimeofday(&tv, NULL);`：又一次"陷入内核"的系统调用——用户态程序自己不知道系统时钟，只有内核能准确告诉你"现在几点"。传的是 `&tv`（地址），因为内核需要把结果**写回**这块内存，不能只传值的拷贝。
- `return tv;`：把 `tv` 的值**整个拷贝一份**返回给调用者（结构体按值返回，是完整拷贝，不是返回地址）。

```
调用 gettimeofday(&tv, NULL) 前（栈上，假设tv在0x8000）：
┌────────┬──────────────┬────────────┐
│ 0x8000 │ tv.tv_sec    │  ???（垃圾） │
│ 0x8008 │ tv.tv_usec   │  ???（垃圾） │
└────────┴──────────────┴────────────┘

内核查完系统时钟后，直接改写这块内存：
┌────────┬──────────────┬────────────┐
│ 0x8000 │ tv.tv_sec    │  1758268800 │
│ 0x8008 │ tv.tv_usec   │  123456     │
└────────┴──────────────┴────────────┘

return tv; 把这16字节内容整个拷贝一份，交给调用者（比如调用者的变量result）：
┌────────┬──────────────┬────────────┐
│ 0x9000 │ result.tv_sec │ 1758268800  │  ← 独立的副本
│ 0x9008 │ result.tv_usec│ 123456      │
└────────┴──────────────┴────────────┘

now() 函数结束后，它自己栈帧（0x8000那块）被回收，
但因为值已经拷贝走了，result不受影响——这是"按值返回"的关键性质。
```

---

## 六、`add_seconds()` 详解——时间点推移的完整计算过程

```c
// 计算 base 时间点，往后推 seconds 秒，得到新的时间点
static struct timeval add_seconds(struct timeval base, double seconds) {
    long usec_offset = (long)(seconds * 1000000);
    base.tv_sec  += usec_offset / 1000000;
    base.tv_usec += usec_offset % 1000000;
    if (base.tv_usec >= 1000000) {
        base.tv_sec += 1;
        base.tv_usec -= 1000000;
    }
    return base;
}
```

### 6.1 函数头

```c
static struct timeval add_seconds(struct timeval base, double seconds)
```

- 参数1：`struct timeval base`——一个时间点，**按值传递**，函数内部拿到的是一份独立拷贝，修改不影响调用者原来的变量
- 参数2：`double seconds`——要往后推移多少秒，可以带小数（比如 `0.5`）
- 返回类型：`struct timeval`——推移之后的新时间点

### 6.2 完整数值演算（用一组会触发"进位"的数字）

假设调用 `add_seconds({1000, 800000}, 0.5)`，也就是"从 1000.8 秒这个时刻，往后推0.5秒"，预期结果应该是 `1001.3` 秒。

**第①行**：把"要推移的秒数"换算成微秒

```c
long usec_offset = (long)(seconds * 1000000);
```
- `seconds * 1000000` = `0.5 * 1000000` = `500000.0`（此刻还是 `double` 类型）
- `(long)(...)`：**强制类型转换**，砍掉小数部分，转成整数 → `500000`

```
usec_offset = 500000
```

**第②行**：把"整秒"部分累加进 `tv_sec`

```c
base.tv_sec += usec_offset / 1000000;
```
- `500000 / 1000000`：**整数除法**（两边都是整数类型时，C 会直接砍掉余数，只保留商），结果是 `0`
- `base.tv_sec += 0` → 还是 `1000`

**第③行**：把"零头微秒"部分累加进 `tv_usec`

```c
base.tv_usec += usec_offset % 1000000;
```
- `%` 是**取余运算符**：`500000 % 1000000` = `500000`（500000除以1000000商0余500000）
- `base.tv_usec` 原本是 `800000`
- `800000 + 500000 = 1300000` —— **超过了999999的上限！**

```
此刻：base.tv_sec = 1000, base.tv_usec = 1300000（不合法，待修正）
```

**第④行**：进位处理

```c
if (base.tv_usec >= 1000000) {
    base.tv_sec += 1;
    base.tv_usec -= 1000000;
}
```
- `1300000 >= 1000000`，条件成立
- `base.tv_sec += 1` → `1000 + 1 = 1001`
- `base.tv_usec -= 1000000` → `1300000 - 1000000 = 300000`

```
最终：base.tv_sec = 1001, base.tv_usec = 300000
```

**验证**：`1000.8 + 0.5 = 1001.3` 秒，算出来的 `{1001, 300000}` 正好就是 1001.3 秒——完全对上。

### 6.3 完整内存过程

```
调用 add_seconds({1000, 800000}, 0.5) 时：

栈帧（假设起始地址 0x9500）
┌────────┬──────────────────┬────────────┐
│ 0x9500 │ base.tv_sec       │  1000       │ ← 参数拷贝进来
│ 0x9508 │ base.tv_usec      │  800000     │
│ 0x9510 │ seconds           │  0.5        │
└────────┴──────────────────┴────────────┘

第①行后：
│ 0x9518 │ usec_offset       │  500000     │

第②行后：
│ 0x9500 │ base.tv_sec       │  1000       │  （+0，不变）

第③行后：
│ 0x9508 │ base.tv_usec      │  1300000    │  （超标，待修正）

第④行 if 成立，进位处理后：
│ 0x9500 │ base.tv_sec       │  1001       │
│ 0x9508 │ base.tv_usec      │  300000     │

return base，把 {1001, 300000} 拷贝一份返回给调用者
```

### 6.4 一句话总结

> `add_seconds` 把一个时间点往后推移指定的秒数：先把秒数拆成"整秒部分"和"零头微秒部分"分别累加，如果零头累加后超过1秒的上限（100万微秒），就把多出的这1秒"搬"到整秒那边去，始终保持 `tv_usec` 只存"不满一秒的零头"这条规矩，跟"凑够100分换1块钱"是同一个道理。

---

## 七、`set_timeout()` 详解——"登记"动作的完整内存过程

```c
void set_timeout(double delay_sec, callback_t cb) {
    timers[timer_count].expire_at = add_seconds(now(), delay_sec);
    timers[timer_count].callback  = cb;
    timers[timer_count].active    = 1;
    timer_count++;
}
```

人话翻译：**"从现在开始往后推 `delay_sec` 秒，算出那个准确的时间点，把这个时间点和'到时候调用哪个函数'一起，记在 `timers` 数组的下一个空位上"**。

跟踪调用 `set_timeout(1.0, f1_rest)`（对应 `f1()` 函数体里那一行），假设此刻 `timer_count == 0`：

### 第①行：算出到期时间点

```c
timers[timer_count].expire_at = add_seconds(now(), delay_sec);
```

等价于拆开的三步：

```c
struct timeval t1 = now();                     // 假设 t1 = {1000, 0}
struct timeval t2 = add_seconds(t1, 1.0);       // 算出 t2 = {1001, 0}
timers[0].expire_at = t2;                       // 存进数组
```

```
timers 数组（此刻，赋值后）
┌──────────┬───────────────────────────┬───────────┐
│ 0x5000   │ timers[0].expire_at.tv_sec │   1001     │
│ 0x5008   │ timers[0].expire_at.tv_usec│    0       │
└──────────┴───────────────────────────┴───────────┘
```

### 第②行：记下到期后要调用谁

```c
timers[timer_count].callback = cb;
```

`cb` 存的是 `f1_rest` 这个函数的地址（假设是 `0x1200`）：

```
┌──────────┬──────────────────────┬────────┐
│ 0x5010   │ timers[0].callback    │ 0x1200  │
└──────────┴──────────────────────┴────────┘
```

### 第③行：标记这张卡片有效

```c
timers[timer_count].active = 1;
```

```
┌──────────┬────────────────────┬────────┐
│ 0x5018   │ timers[0].active    │   1     │
└──────────┴────────────────────┴────────┘
```

### 第④行：账本计数 +1

```c
timer_count++;
```

```
┌──────────┬──────────────┬────────┐
│ 0x5100   │ timer_count   │   1     │  ← 从0变成1
└──────────┴──────────────┴────────┘
```

之后再有谁调用 `set_timeout`，就会填进 `timers[1]`，不会覆盖 `timers[0]`。

### 一句话总结

> `set_timeout` 不是真的"等" `delay_sec` 秒，而是做三件记账的事：算出未来的准确时间点、记下到时候调用谁、标记有效，然后立刻返回——真正的"等待"，被推迟到了 `event_loop()` 里才发生。这正是"函数拆成两半"里的**前一半**：立刻返回，不卡住调用者。

---

## 八、`event_loop()` 详解——完整走两轮，配内存图

承接上文，假设 `f1()`、`f2()` 都调用完了，状态是：

```
timers[0]: expire_at={1001, 0}       callback=f1_rest  active=1
timers[1]: expire_at={1000, 500000}  callback=f2_rest  active=1
timer_count = 2
```

### 8.1 第一轮：找出最早到期的定时器

```c
int earliest = -1;
for (int i = 0; i < timer_count; i++) {
    if (!timers[i].active) continue;
    if (earliest == -1 || timercmp(&timers[i].expire_at, &timers[earliest].expire_at, <)) {
        earliest = i;
    }
}
```

`earliest` 是局部变量（初始 `-1`，意思是"还没选出任何一个"）。

**`i=0`**：`timers[0].active` 为真，不跳过。`earliest == -1` 成立（逻辑或 `||` 短路，右边不再判断），`earliest = 0`。

**`i=1`**：`timers[1].active` 为真，不跳过。这次 `earliest == -1` 是假（已经是0了），必须判断右边：

```c
timercmp(&timers[1].expire_at, &timers[0].expire_at, <)
// 即比较 {1000, 500000} < {1001, 0} ？
// 先比 tv_sec：1000 < 1001 → 成立
```

条件成立，`earliest = 1`。

`timercmp` 是 `<sys/time.h>` 提供的宏，专门比较两个 `struct timeval`（因为结构体不能直接用 `<` 比较，必须先比 `tv_sec`，相等再比 `tv_usec`，这套逻辑被封装进了这个宏里）。

循环结束，`earliest = 1`——`timers[1]`（f2 的定时器）最早到期，符合预期（f2 只等0.5秒，比f1的1秒快）。

### 8.2 检查"是否全部处理完"

```c
if (earliest == -1) break;
```

`earliest` 是 `1`，不成立，继续往下走。（这一句存在的意义：如果所有定时器都已触发过、`active` 全是0，`earliest` 会一直停在 `-1`，这时候该退出循环，不然会死循环空转。）

### 8.3 算出"还要等多久"

```c
struct timeval now_tv = now();
struct timeval remaining;
timersub(&timers[earliest].expire_at, &now_tv, &remaining);
```

假设 `now_tv = {1000, 0}`（刚好是注册定时器那一刻，方便手算）：

```c
timersub(&timers[1].expire_at, &now_tv, &remaining);
// remaining = {1000, 500000} - {1000, 0} = {0, 500000}
```

`timersub` 也是宏：**"用第一个参数减去第二个参数，结果写进第三个参数指向的内存"**。

```
remaining.tv_sec  = 0
remaining.tv_usec = 500000     （意思：还要等0.5秒）
```

```c
if (remaining.tv_sec < 0) {
    remaining.tv_sec = 0;
    remaining.tv_usec = 0;
}
```

`remaining.tv_sec` 是 `0`，不小于0，跳过。（这段防御性代码只在"算出来已经是负数"——比如循环处理耗费了点时间——这种极端情况下才会触发。）

### 8.4 `select` 被当"精确睡眠"工具用

```c
select(0, NULL, NULL, NULL, &remaining);
```

- `nfds` 传 `0`，`readfds` 传 `NULL`：告诉内核"不关心任何fd"
- `&remaining`：把"还要等多久"的地址传给内核

这种用法下，`select` 唯一作用变成"睡够 `remaining` 这段时长，时间一到无条件唤醒"——一个常见的技巧：没有专门"睡到某精确时刻"的简单API时，借用 `select` 的超时机制实现。内核层面的"陷入内核、栈帧冻结、超时后唤醒"机制，跟前一版监听fd时完全一样，只是这次没有任何fd参与判断。

线程在这里真正阻塞了约0.5秒。

### 8.5 醒来后，检查所有真正到期的定时器

```c
struct timeval now2 = now();
for (int i = 0; i < timer_count; i++) {
    if (timers[i].active && timercmp(&timers[i].expire_at, &now2, <=)) {
        timers[i].active = 0;
        timers[i].callback();
    }
}
```

假设醒来这一刻 `now2 = {1000, 500010}`（比到期时间稍晚一点点，属于正常的微小延迟）。

**`i=0`**：`timers[0].active` 为真。`timercmp({1001,0}, {1000,500010}, <=)`：先比 `tv_sec`：`1001 <= 1000`？不成立。整体条件为假，`timers[0]` **不触发**（还没到它的点）。

**`i=1`**：`timers[1].active` 为真。`timercmp({1000,500000}, {1000,500010}, <=)`：`tv_sec` 相等，比 `tv_usec`：`500000 <= 500010`？成立！整体条件为真：

```c
timers[1].active = 0;     // 标记作废，防止以后被重复选中
timers[1].callback();     // 等价于调用 f2_rest() → 打印"f2 恢复执行"
```

```
内存变化：
timers[1]: expire_at={1000,500000}  callback=f2_rest  active=0   ← 从1变成0
```

**注意这里和上一版 `FD_ISSET` 检查的不同**：这次是遍历所有定时器、用"醒来时刻的当前时间"重新跟每个 `expire_at` 比较，而不是像 `readfds` 那样直接读"内核标记好的结果"——因为 `select` 只负责"睡够指定时长"，它并不知道这是"定时器"，**判断"谁到期了"完全是我们自己在用户态代码里做的**，内核没有参与这部分逻辑。

### 8.6 第二轮循环

回到 `while(1)` 开头，重新"找最早到期"：

```
i=0: timers[0].active 是1，不跳过，earliest=0
i=1: timers[1].active 是0，continue，跳过（不会被再选中）
```

`earliest` 停在 `0`（f1的定时器）。后续流程一样：算 `remaining`（约剩0.5秒）→ `select` 睡这么久 → 醒来 → 检查发现 `timers[0]` 到期 → `active`置0、调用 `f1_rest()` → 打印"f1 恢复执行"。

### 8.7 第三轮：退出

再次回到循环开头："找最早到期"时，`timers[0]` 和 `timers[1]` 的 `active` 都是 `0`，两个都被跳过，`earliest` 从头到尾停在 `-1`：

```c
if (earliest == -1) break;
```

成立，跳出 `while(1)`，`event_loop()` 返回，控制权回到 `main()`，`main` 里 `event_loop();` 的下一行 `return 0;` 执行，程序结束。

### 8.8 一句话总结 `event_loop`

> 每一轮循环：在"还没触发过"的定时器里挑出最早到期的那个，只睡到那个时间点为止（而不是傻等所有定时器），醒来后逐个检查有没有到期的（可能不止一个，全部触发），处理完回到循环开头重新挑选——直到所有定时器都触发过、挑不出新的为止，才真正退出。

---

## 九、和上一版（监听 stdin）模型的核心区别

| 对比点 | 上一版（events，监听fd） | 这一版（timers，监听时间点） |
|---|---|---|
| 登记的是什么 | `fd`（一种持续存在的资源） | `expire_at`（一个只会发生一次的未来时间点） |
| 是否需要 `active` 字段 | 不需要，fd 的登记默认一直有效，每轮都检查 | 需要，触发一次就必须作废，防止重复触发 |
| select 的用途 | 同时等一批 fd 就绪（或超时兜底） | 完全不关心fd，纯粹当"精确睡眠"工具用 |
| 醒来后怎么判断"谁触发了" | 读内核直接改写好的 `readfds` 位图（`FD_ISSET`） | 自己在用户态重新比较当前时间和每个 `expire_at`（`timercmp`），内核不知道这是定时器语义 |
| 循环退出条件 | 没写退出逻辑（无限循环，服务器场景本该如此） | `earliest == -1`（所有任务都处理完了）就主动退出 |

---

## 十、和 JS async/await 的最终对应关系

| JS | C（这一版实现） |
|---|---|
| `async function f1() { ... }` | 拆成 `f1()`（前半段） + `f1_rest()`（`await`之后的部分） |
| 函数体执行到 `await`，函数"提前返回" | `f1()` 里调用 `set_timeout(...)` 后自然执行完、返回 |
| `await` 之后的代码，被引擎包装成回调，等时机成熟再调用 | `f1_rest` 被存进 `timers` 数组的 `callback` 字段，等 `event_loop` 判断到期后手动调用 |
| JS 引擎自带、自动运行的事件循环 | 我们自己在 `main()` 最后手动调用一次 `event_loop()` |
| `setTimeout(callback, ms)` | `set_timeout(delay_sec, callback)` |

**核心结论（贯穿两份笔记的主线）**：不管是 C 手写的这套模型，还是 JS 的 `async/await`，本质上做的都是同一件事——**把一个函数从"该等待的点"切开，前半段立刻返回，后半段被打包成回调、交给一个不断轮询的事件循环，在条件满足时才被调用**。区别只在于：C 需要你手动完成"拆函数"和"调用事件循环"这两步，JS 的语法糖帮你把这两步自动化、隐藏了起来。

---

## 十一、可以继续深挖的方向

1. **把这一版的定时器系统，和上一版的 stdin 监听合并**：让 `select` 同时监听 fd 事件和维护一个最早到期时间的超时值，做出一个更接近真实 Node.js / libuv 的事件循环
2. **升级到 `epoll`**：理解 select 在 fd 数量多时为什么慢，epoll 怎么解决
3. **深入协程（coroutine）**：理解 Rust/JS 的 async/await 底层，是怎么做到"函数真正挂起、精确恢复到断点"的（这比纯回调模型更进一步，需要理解"保存/恢复执行上下文"这套机制）
