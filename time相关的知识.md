>头文件 `time.h` 
## 1. time_t
时间在程序里的存储形式
`time_t` 是 `<time.h>` 里定义的整数类型。在 Linux 上（POSIX 规定），它的值是*从 1970-01-01 00:00:00 UTC 起经过的秒数*，这个起点叫 Unix 纪元
C 标准本身没规定它的起点和单位，只有 POSIX 系统才保证是秒数。你在 Linux 上可以放心这样理解
64 位 Linux 上 `time_t` 是 64 位，不用担心 2038 年溢出（那是 32 位 `time_t` 的问题）

流水里存 `time_t`，而不存 `"2026-11-19 12:00:00"` 这样的字符串，原因是：占 8 字节、可以直接比较大小排序、可以做减法算间隔，需要显示时再格式化成任何样子
## 2. time(NULL)
取当前时间
```c
/* timer 如果不是 NULL，当前时间同时写到它指向的变量里；传 NULL 就只用返回值 */
time_t time(time_t *timer);
```
返回当前时间；失败时返回 `(time_t)(-1)`，实际很少发生
## 3. `struct tm`：拆开的时间
两个坑：月份从 0 开始，年份是减去 1900 的。要手动显示时，写 `tm_year + 1900` 和 `tm_mon + 1`
```c
struct tm {
    int tm_sec;    /* 秒 0-60（60 是闰秒） */
    int tm_min;    /* 分 0-59 */
    int tm_hour;   /* 时 0-23 */
    int tm_mday;   /* 日 1-31 */
    int tm_mon;    /* 月 0-11，注意从 0 开始 */
    int tm_year;   /* 从 1900 起的年数 */
    int tm_wday;   /* 星期 0-6，0 是周日 */
    int tm_yday;   /* 一年中的第几天 0-365 */
    int tm_isdst;  /* 夏令时标记 */
};
```
## 4. localtime
秒数拆成本地时间
```c
/* timer  time_t 变量的地址（不是值），只读，所以是 const */
struct tm *localtime(const time_t *timer);
```

返回 `struct tm *`，已按本机时区换算。对应有个 `gmtime`，用法相同，但得到的是 UTC 时间

**要注意两点：**
- 返回的指针指向库内部的静态缓冲区。 你不需要 `free`，也不能 `free`。但下一次调用 `localtime` 或 `gmtime` 会覆盖同一块内存，所以不要保存这个指针，用完立刻处理。多线程下也不安全，POSIX 提供了线程安全版 `localtime_r`
- 理论上可能返回 `NULL`（比如时间值大到无法换算）。我们的代码没检查，因为这里的时间来自 `time(NULL)`，不会触发。以后处理外部读进来的时间戳时，应该检查，否则把 `NULL` 传给 `strftime` 会崩

## 5. strftime
格式化成字符串
```c
size_t strftime(char *s, size_t maxsize, const char *format, const struct tm *tm);
```

| 参数        | 含义                                    |
| --------- | ------------------------------------- |
| `s`       | 输出缓冲区，结果写到这里                          |
| `maxsize` | 缓冲区总大小，包括结尾的 `'\0'`；我们传 `sizeof(buf)` |
| `format`  | 格式串，`%` 开头的是占位符，其余原样输出                |
| `tm`      | 要格式化的时间                               |
**返回值**：写入的字符数，不含结尾的 `'\0'`。如果结果（加上 `'\0'`）放不进 `maxsize` 个字节，返回 `0`，并且此时 `s` 的内容不确定，不能再用

## 6. 常用占位符：
| 占位符  | 含义                  | 例子   |
| ---- | ------------------- | ---- |
| `%Y` | 四位年份                | 2026 |
| `%m` | 月（01-12，已经 +1 处理好了） | 11   |
| `%d` | 日（01-31）            | 19   |
| `%H` | 时（00-23）            | 09   |
| `%M` | 分                   | 05   |
| `%S` | 秒                   | 07   |
所以 `"%Y-%m-%d %H:%M:%S"` 得到 `2026-11-19 09:05:07`，共 19 个字符加 `'\0'` 是 20 字节，我们的 `buf[32]` 够用

注意，`strftime` 会帮你处理年份 +1900、月份 +1，只有你自己直接读 `struct tm` 字段时才需要手动换算
## 7. 测试
```c
#include <stdio.h>
#include <time.h>

int main(void) {
    time_t now = time(NULL);
    /*  
	    这里用 (long long)now 强转再用 %lld 打印，
	    是因为 time_t 具体是哪种整数类型因平台而异，转成 long long 最稳妥 
    */
    printf("秒数: %lld\n", (long long)now);

    struct tm *p = localtime(&now);
    if (p == NULL) {
        return 1;
    }
    printf("年 %d  月 %d  日 %d\n", p->tm_year + 1900, p->tm_mon + 1, p->tm_mday);
    printf("时 %d  分 %d  秒 %d\n", p->tm_hour, p->tm_min, p->tm_sec);
    printf("星期 %d（0 表示周日）\n", p->tm_wday);

    char buf[32];
    size_t n = strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", p);
    printf("%s（写入 %zu 个字符）\n", buf, n);

    /* 缓冲区太小的情况 */
    char small[8];
    n = strftime(small, sizeof(small), "%Y-%m-%d %H:%M:%S", p);
    printf("缓冲区太小时返回: %zu\n", n);

    return 0;
}
```