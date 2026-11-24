## 1. 最底层：总线和"一次读多少"
CPU 和内存之间通过**总线**（bus）通信，主要有三种：
- **地址总线**：CPU 说"我要访问这个地址"
- **数据总线**：真正传输数据的线，宽度决定了一次能搬多少数据
- **控制总线**：读/写信号、时钟信号等
![[Pasted image 20260924185226.png]]
现代 64 位 CPU，数据总线宽度通常是 64 位（8 字节），这就是所谓的"字长"（word size）。理论上一次总线传输能搬 8 字节
**但实际上 CPU 不是直接按字节/字跟内存打交道的**——中间隔着 Cache
### 1.1 验证"字长"和数据总线宽度
```c
#include <stdio.h>
int main()
{
    printf("int:    %zu bytes\n", sizeof(int));
    printf("long:   %zu bytes\n", sizeof(long));
    printf("void*:  %zu bytes\n", sizeof(void *));
    printf("double: %zu bytes\n", sizeof(double));
}
/*
int:    4 bytes
long:   8 bytes
void*:  8 bytes
double: 8 bytes
*/
```
`void*` 是 8 字节，这不是巧合——它直接反映了 CPU 的**地址总线/寄存器宽度**是 64 位。`long` 也是 8 字节，说明一个寄存器一次能装下一个这样的值，做加减乘除这类操作一个周期基本能搞定
### 1.2 验证 Cache Line 大小
```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    long line_size = sysconf(_SC_LEVEL1_DCACHE_LINESIZE);
    printf("L1 cache line size: %ld bytes\n", line_size);
    // 典型输出: 64
}
```
### 1.3 验证 cache line 局部性
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>


#define SIZE (64 * 1024 * 1024) // 64MB 数组

int main()
{
    int *arr = malloc(SIZE * sizeof(int));
    for (int i = 0; i < SIZE; i++)
        arr[i] = 1;

    // 实验1：连续访问（利用了 cache line 的局部性）
    clock_t start = clock();
    long sum = 0;
    for (int i = 0; i < SIZE; i++)
    {
        sum += arr[i]; // stride = 1
    }
    printf("%ld\n", sum);

    printf("连续访问耗时: %f 秒\n", (double)(clock() - start) / CLOCKS_PER_SEC);
  
    // 实验2改进版：访问次数与实验1相同，但每次跳到不同cache line
    start = clock();
    sum = 0;
    for (int i = 0; i < SIZE; i++)

    {
        int idx = (i * 16) % SIZE; // 保持循环 SIZE 次，但每次跳16个int
        sum += arr[idx];
    }

    printf("%ld\n", sum);

    printf("跨步访问耗时: %f 秒\n", (double)(clock() - start) / CLOCKS_PER_SEC);

    free(arr);
}

/*
67108864
连续访问耗时: 0.108743 秒
67108864
跨步访问耗时: 0.215805 秒
*/
```
## 2. Cache Line：真正的"最小搬运单位"
现代 CPU 和内存之间还有多级缓存（L1/L2/L3）。CPU 从内存读数据，不是读你要的那 4 个字节或 8 个字节，而是**整块搬**，这个块叫 **cache line**，主流架构（x86/ARM）通常是 **64 字节**

```c
/*
内存地址空间（一条条 64 字节的 cache line）：
[0-63] [64-127] [128-191] [192-255] ...
          ↑
      你只想读一个 int（4字节，比如地址 68）
      但 CPU 会把整条 [64-127] 这 64 字节都搬进 L1 Cache
*/
```

**为什么这样设计？因为：**
1. 内存访问延迟很高（几十到上百个时钟周期），一次多搬点划算
2. 程序有"空间局部性"——你读了一个变量，附近的变量大概率马上也要用（比如数组、struct 的相邻字段）

```c
/*
CPU 要读 arr[i]
   │
   ▼
先查 L1 Cache 有没有这条 cache line
   │
   ├─ 命中(hit) → 直接从 Cache 取，几个周期搞定
   │
   └─ 未命中(miss) → 逐级往下查 L2 → L3 → 都没有
                        │
                        ▼
                  发地址总线请求，从 DRAM 搬一整条 64 字节
                  的 cache line 进 L1/L2/L3，
                  再从 Cache 里取你要的那几个字节
*/
```
### 2.1 关于地址/寄存器位宽

- x86-64 的通用寄存器、虚拟地址空间理论上是64位的，但实际上目前主流实现只用到48位（或用五级页表时57位）虚拟地址，高位必须做符号扩展（即所谓的"canonical address"）。
- 物理地址总线的实际位宽因CPU型号而异，通常在36～52位之间（这个具体数值我没有把握给出某一代CPU的精确值，因为不同型号差异较大，如果你需要某款具体CPU的物理地址位宽，建议查该CPU的规格手册）。
- 所以"64位"更多是指寄存器/虚拟地址的位宽上限，不是说"地址总线一次能传64字节数据"。地址总线传的是地址，不是数据本身。
### 2.2 关于CPU一次读取多少数据到缓存

- 现代x86-64 CPU的Cache Line（缓存行）大小通常是64字节，这是主流Intel/AMD CPU的常见实现值。
- 当CPU要读取一个int（4字节）而这个地址不在缓存中（cache miss）时，硬件会以整个cache line为单位（通常64字节）从内存加载到L1/L2/L3缓存中，然后CPU再从这64字节里定位、读取真正需要的那4字节。
- 这个64字节的cache line大小是缓存子系统的设计参数，和"x64"里的64位寄存器/地址宽度只是数值上恰好相同，没有因果关系。不同CPU架构的cache line大小可能不同（比如某些ARM实现是64字节，也有128字节的情况，这个我不确定具体是哪些型号，如果需要准确对应关系建议查具体芯片手册）
### 2.3 cache miss
"Cache miss" 指的是 **CPU的缓存（Cache）** 里的事件，不是内存或磁盘的概念。具体说：
现代CPU内部一般有多级缓存，从CPU核心往外依次是：
```c
/*
CPU 寄存器 (最快，容量最小)
    ↓
L1 Cache (每个核心私有，通常 32KB~64KB 左右，分指令/数据)
    ↓
L2 Cache (每个核心私有或部分共享，通常几百KB到几MB)
    ↓
L3 Cache (多核心共享，通常几MB到几十MB)
    ↓
主内存 DRAM (容量大，速度慢很多)
*/
```
这些容量数字是常见量级，不是某个具体型号的精确规格，如果你需要具体某款CPU的确切缓存大小，建议查该CPU的规格页面或用 `lscpu` 命令
### 2.4 cache miss 具体指什么
当CPU想读取某个地址的数据时：
1. 先去 L1 Cache 里查，这个地址对应的数据在不在里面
    - 在 →  L1 hit，直接拿，很快（大约几个时钟周期）
    - 不在 → L1 miss，往下一级找
2. 去 L2 Cache 查
    - 在 → L2 hit（但比L1慢）
    - 不在 → L2 miss，往下找
3. 去 L3 Cache查（同理）
4. 如果 L3 也没有 → 最终要去里取，**这是最慢的一步**（相对CPU时钟周期而言，可能是几十到上百个周期，具体数字因系统而异，我没有把握给出精确值）

cache miss 是"CPU的缓存"这个空间里的事件——指的是CPU想找的数据不在当前查询的这一级缓存里，需要往更下一级（更慢、更大）的存储层级继续查找，直到在某一级缓存命中，或者最终不得不去主内存取
## 3. CPU 真正"取指令/取数据"时一次拿多少

这里要分开说：
- **总线传输粒度**：cache line，64 字节（跟内存/Cache 打交道）
- **寄存器/ALU 操作粒度**：字长，64 位 = 8 字节（CPU 内部计算单位，一个整数寄存器能装 8 字节）
- **你请求的数据本身**：可以是 1/2/4/8 字节（char/short/int/long），CPU 有专门的 load 指令支持不同宽度（如 x86 的 `movb/movw/movl/movq`）

也就是说，你写 `int x = arr[3];`，CPU 执行的是一条 `mov` 指令，只把那 4 个字节从 Cache 搬进寄存器——但**这 4 字节所在的整条 cache line 早就已经因为局部性被搬进 Cache 了**
## 4. 内存对齐：为什么要对齐
### 4.1 硬件层面：跨 cache line / 跨总线传输的代价
假设一个 4 字节的 int，如果它的起始地址不是 4 的倍数，比如从地址 62 开始（占 62,63,64,65），那么这个 int 横跨了两条 cache line（\[0-63] 和 \[64-127]）
```c
/*
cache line 0: [...60][61][62][63]
cache line 1: [64][65][66]...

int 变量占 [62,63,64,65]  ← 跨界了！
*/
```
CPU 要读这个 int，就得发两次内存/Cache 访问，再把结果拼起来，这叫 **split access / unaligned access**，慢得多（有的架构甚至直接不支持，会触发异常）。

如果 int 从 64 开始（4 的倍数），它完整落在一条 cache line 里，一次访问搞定