
> 本文基于 **Lua 5.4**。示例中的 `--> 结果` 表示 `print` 的输出。
# 第 5 节　流程控制
流程控制决定代码“**走哪条路**”和“**重复多少次**”。Lua 提供

| 类别  | 语句                                           |
| --- | -------------------------------------------- |
| 分支  | `if ... then ... elseif ... else ... end`    |
| 循环  | `while`、`repeat ... until`、数值 `for`、泛型 `for` |
| 跳转  | `break`、`goto`、`return`                      |

> 回顾：条件表达式里，**只有 `nil` 和 `false` 是假**，`0` 和 `""` 都是真。
## 5.1 if 条件分支
### 基本形式
```lua
local score = 85

if score >= 60 then
    print("及格")
end
```
### if ... else
```lua
local n = 7
if n % 2 == 0 then
    print("偶数")
else
    print("奇数")        --> 奇数
end
```
### if ... elseif ... else（多分支）
注意关键字是 `elseif`（连写，不是 `else if`）。

```lua
local score = 85

if score >= 90 then
    print("优秀")
elseif score >= 80 then
    print("良好")        --> 良好
elseif score >= 60 then
    print("及格")
else
    print("不及格")
end
```

> 分支按顺序判断，**命中第一个为真的分支后，后面的都不会再执行**。
> 所以应把条件从“严格”到“宽松”排列。
### 组合条件：`and` / `or` / `not`
```lua
local age, hasTicket = 20, true

if age >= 18 and hasTicket then
    print("可以入场")    --> 可以入场
end

if not hasTicket or age < 18 then
    print("不能入场")
end
```
### ⚠️ 易错点
**① 用 `~=` 表示不等于，不是 `!=`：**

```lua
if 1 ~= 2 then print("不相等") end
```

**② 判断变量“有没有值”：**

```lua
local name = nil
if name then print("有名字") else print("没名字") end   --> 没名字

local count = 0
if count then print("0 也是真") end                      --> 0 也是真
```

**③ 如果要判断“是否为 nil”，而不是“是否为假”，请用 `== nil`：**

```lua
local flag = false
if flag == nil then print("未设置") else print("已设置为 false") end
--> 已设置为 false
```
### Lua 没有 switch：用 if 链或“表分发”
if 链：
```lua
local cmd = "stop"
if cmd == "start" then
    print("启动")
elseif cmd == "stop" then
    print("停止")        --> 停止
else
    print("未知命令")
end
```

表分发（分支多时更清爽）：

```lua
local actions = {
    start = function() print("启动") end,
    stop  = function() print("停止") end,
}

local cmd = "stop"
local handler = actions[cmd]
if handler then
    handler()            --> 停止
else
    print("未知命令")
end
```
## 5.2 while 循环
**先判断，后执行**：条件为真就一直循环。
```lua
local i = 1
while i <= 5 do
    print(i)
    i = i + 1            -- 别忘了更新，否则死循环
end
--> 1 2 3 4 5（每个一行）
```

例子：求 1 到 100 的和：

```lua
local sum, i = 0, 1
while i <= 100 do
    sum = sum + i
    i = i + 1
end
print(sum)               --> 5050
```

例子：折半查找次数（不知道要循环几次时，`while` 最合适）：

```lua
local n, steps = 1000, 0
while n > 1 do
    n = n // 2
    steps = steps + 1
end
print(steps)             --> 9
```
## 5.3 repeat ... until 循环
**先执行，后判断**：至少执行一次，**条件为真时结束**（和 `while` 相反！）。

```lua
local i = 1
repeat
    print(i)
    i = i + 1
until i > 3
--> 1 2 3
```

对比“至少执行一次”的特点：

```lua
local x = 100

while x < 10 do
    print("while 执行了")     -- 一次也不会执行
end

repeat
    print("repeat 执行了")    --> repeat 执行了   （至少一次）
until x >= 10
```
### 特色：`until` 的条件里可以访问循环体内的局部变量
```lua
local n = 0
repeat
    local done = n >= 3       -- 循环体内声明的局部变量
    n = n + 1
until done                    -- until 里仍然能看到 done
print(n)                      --> 4
```

（这一点和 C 的 `do...while` 不同。）

典型应用：输入校验、重试直到成功：

```lua
local tries = 0
local ok
repeat
    tries = tries + 1
    ok = tries >= 3           -- 模拟：第 3 次才成功
    print("第 " .. tries .. " 次尝试", ok)
until ok
```

## 5.4 数值 for 循环

### 语法

```
for 变量 = 起始值, 结束值 [, 步长] do
    循环体
end
```

- 步长默认为 `1`
- **起始值、结束值都包含**（闭区间）
- 三个表达式**只在循环开始前计算一次**

```lua
for i = 1, 5 do
    io.write(i, " ")
end
print()                       --> 1 2 3 4 5

for i = 10, 1, -3 do          -- 步长为负：倒数
    io.write(i, " ")
end
print()                       --> 10 7 4 1

for i = 1, 0 do               -- 起始 > 结束，且步长为正：一次也不执行
    print("不会输出")
end

for i = 0, 1, 0.25 do         -- 浮点步长
    io.write(i, " ")
end
print()                       --> 0.0 0.25 0.5 0.75 1.0
```

### 要点

**① 循环变量是局部变量，只在循环体内有效：**

```lua
for i = 1, 3 do end
print(i)                      --> nil   （循环外访问不到）
```

**② 结束值只在开始时计算一次：**

```lua
local n = 3
for i = 1, n do
    n = 10                    -- 修改 n 不影响循环次数
    print(i)
end
--> 1 2 3
```

**③ 不要在循环体里修改循环变量**（修改的只是副本，不会影响循环进度，且容易误导读者）：

```lua
for i = 1, 3 do
    i = i * 10
    print(i)                  --> 10 20 30   （循环仍然只跑 3 次）
end
```

**④ 步长不能为 0**，否则报错：`'for' step is zero`。

### 常见应用

```lua
-- 遍历数组
local fruits = {"apple", "banana", "cherry"}
for i = 1, #fruits do
    print(i, fruits[i])
end

-- 倒序遍历
for i = #fruits, 1, -1 do
    print(fruits[i])          --> cherry banana apple
end

-- 九九乘法表
for i = 1, 9 do
    local row = {}
    for j = 1, i do
        row[#row + 1] = j .. "x" .. i .. "=" .. i * j
    end
    print(table.concat(row, "  "))
end
```

## 5.5 泛型 for 循环

用于**遍历“迭代器”**，最常见的是遍历表。

### 语法

```
for 变量1, 变量2, ... in 迭代器表达式 do
    循环体
end
```

### ipairs：按顺序遍历数组部分

从下标 `1` 开始依次遍历，**遇到第一个 `nil` 就停止**。

```lua
local colors = {"red", "green", "blue"}
for i, v in ipairs(colors) do
    print(i, v)
end
--> 1   red
--> 2   green
--> 3   blue

-- 遇到 nil 就停
local t = {10, 20, nil, 40}
for i, v in ipairs(t) do
    print(i, v)               --> 1 10 / 2 20 （到 nil 就停了，40 访问不到）
end
```

### pairs：遍历表中所有键值对

```lua
local person = {name = "Tom", age = 20, city = "Taipei"}
for k, v in pairs(person) do
    print(k, v)
end
--> name Tom / age 20 / city Taipei（顺序不保证！）
```

> ⚠️ **`pairs` 的遍历顺序是不确定的**，不要依赖它。需要固定顺序，可以先把键取出来排序：
>
> ```lua
> local keys = {}
> for k in pairs(person) do keys[#keys + 1] = k end
> table.sort(keys)
> for _, k in ipairs(keys) do
>     print(k, person[k])   --> age 20 / city Taipei / name Tom
> end
> ```

### 其他迭代器：`string.gmatch`、`io.lines` 等

```lua
-- 拆分字符串
for word in string.gmatch("apple,banana,cherry", "[^,]+") do
    print(word)
end
--> apple / banana / cherry

-- 逐个字符
for ch in ("Lua"):gmatch(".") do
    io.write(ch, "-")
end
print()                       --> L-u-a-
```

### 遍历时修改表的规则

- ✅ 遍历时可以把**已有字段**赋值为 `nil`（删除）或修改其值
- ❌ 遍历时**不要新增字段**，行为未定义

```lua
local t = {a = 1, b = 2, c = 3}
for k in pairs(t) do
    t[k] = nil                -- 清空：允许
end
print(next(t))                --> nil   （表空了）
```

### 数值 for 与 ipairs 怎么选？

| 场景 | 推荐 |
|------|------|
| 只想要元素，不关心下标，数组连续 | `ipairs` |
| 需要倒序、跳着走、指定范围 | 数值 `for` |
| 遍历字典 / 不确定的键 | `pairs` |

## 5.6 break 与 goto

### break：跳出当前循环

只会跳出**最内层**的一层循环。

```lua
for i = 1, 10 do
    if i > 3 then break end
    print(i)                  --> 1 2 3
end

-- 查找第一个满足条件的元素
local nums = {3, 8, 12, 5, 20}
local found
for i, v in ipairs(nums) do
    if v > 10 then
        found = i
        break
    end
end
print(found)                  --> 3
```

### goto：跳转到标签

语法：`goto 标签名`，标签写法为 `::标签名::`。

Lua **没有 `continue`**，可以用 `goto` 模拟：

```lua
for i = 1, 10 do
    if i % 2 == 0 then
        goto continue         -- 偶数：跳过本轮剩余代码
    end
    print(i)
    ::continue::
end
--> 1 3 5 7 9
```

用 `goto` 一次跳出多层循环：

```lua
for i = 1, 3 do
    for j = 1, 3 do
        if i * j == 4 then
            goto done         -- 一次跳出两层循环
        end
        print(i, j)
    end
end
::done::
print("结束")
--> 1 1 / 1 2 / 1 3 / 2 1 / 结束
```

### goto 的规则

- 只能跳到**同一函数内、可见的标签**
- **不能跳进**某个局部变量的作用域内（标签在块末尾除外，所以上面 `::continue::` 放在循环体最后是安全的）
- 不能跨函数跳转

> 💡 `goto` 用于“`continue` / 跳出多层循环 / 统一清理”这几类场景就够了，不要滥用。

### return：提前结束函数

`return` 只能写在**代码块的最后一条语句**；如果想在中间提前返回，用 `do return end` 包起来。

```lua
local function test(x)
    if x < 0 then
        return "负数"          -- 在 if 块末尾，没问题
    end
    return "非负数"
end
print(test(-1), test(5))      --> 负数   非负数
```

## 5.7 综合例子

**FizzBuzz：** 1~15，3 的倍数输出 Fizz，5 的倍数输出 Buzz，两者都是输出 FizzBuzz。

```lua
for i = 1, 15 do
    if i % 15 == 0 then
        print("FizzBuzz")
    elseif i % 3 == 0 then
        print("Fizz")
    elseif i % 5 == 0 then
        print("Buzz")
    else
        print(i)
    end
end
```

**找出 2~30 中的所有素数：**

```lua
for n = 2, 30 do
    local isPrime = true
    for d = 2, math.floor(math.sqrt(n)) do
        if n % d == 0 then
            isPrime = false
            break
        end
    end
    if isPrime then io.write(n, " ") end
end
print()    --> 2 3 5 7 11 13 17 19 23 29
```

## 5.8 小结与练习

**小结：**

- 分支用 `if / elseif / else`，没有 `switch`
- `while` 先判断；`repeat...until` 先执行，**条件为真时退出**
- 数值 `for` 三个参数只算一次，循环变量是局部的
- `ipairs` 遇 `nil` 停；`pairs` 顺序不确定
- 没有 `continue`，用 `goto continue`

**练习：** 下面代码输出什么？

```lua
local t = {}
for i = 1, 10 do
    if i % 3 == 0 then goto continue end
    if i > 7 then break end
    t[#t + 1] = i
    ::continue::
end
print(table.concat(t, ","))
```

<details>
<summary>答案</summary>

```
1,2,4,5,7
```

3、6 被 `continue` 跳过；到 8 时 `i > 7` 触发 `break`。

</details>

---

# 第 6 节　函数

## 6.1 定义与调用

### 基本定义

```lua
function greet(name)
    print("你好, " .. name)
end

greet("Lua")                  --> 你好, Lua
```

### 局部函数（推荐）

函数也是值，`function f() end` 本质上是把函数赋给变量 `f`。与变量一样，**优先用 `local`**：

```lua
local function add(a, b)
    return a + b
end
print(add(3, 4))              --> 7
```

### 两种等价写法

```lua
local function sq(x) return x * x end     -- 写法 1（语法糖）
local sq2 = function(x) return x * x end  -- 写法 2（把匿名函数赋给变量）
print(sq(5), sq2(5))                      --> 25   25
```

### 调用时的省略括号语法

当函数只有**一个参数，且是字符串字面量或表构造器**时，括号可以省略：

```lua
print "Hello"                 --> Hello
print [[多行
字符串]]

local function show(t) print(#t) end
show {1, 2, 3}                --> 3
```

### 参数个数不匹配

规则和多重赋值一样：**缺的补 `nil`，多的丢弃**。

```lua
local function f(a, b, c)
    print(a, b, c)
end

f(1)                          --> 1   nil   nil
f(1, 2, 3, 4)                 --> 1   2   3     （4 被丢弃）
```

利用这一点实现**默认参数**：

```lua
local function connect(host, port)
    host = host or "localhost"
    port = port or 8080
    print(host .. ":" .. port)
end

connect()                     --> localhost:8080
connect("example.com")        --> example.com:8080
connect("example.com", 443)   --> example.com:443
```

### 表中的函数

```lua
local calc = {}

function calc.add(a, b) return a + b end    -- 等价于 calc.add = function(a, b) ... end
function calc.sub(a, b) return a - b end

print(calc.add(1, 2), calc.sub(5, 3))       --> 3   2
```

冒号语法 `obj:method()` 会自动把 `obj` 作为第一个参数 `self` 传入（面向对象章节详讲）：

```lua
local counter = {n = 0}

function counter:inc()        -- 等价于 function counter.inc(self)
    self.n = self.n + 1
end

counter:inc()                 -- 等价于 counter.inc(counter)
counter:inc()
print(counter.n)              --> 2
```

## 6.2 多返回值

Lua 的函数可以一次返回**多个值**。

```lua
local function minmax(a, b)
    if a < b then return a, b else return b, a end
end

local lo, hi = minmax(9, 3)
print(lo, hi)                 --> 3   9
```

标准库中就有大量多返回值函数：

```lua
print(string.find("hello world", "wor"))   --> 7   9      （起止位置）
print(math.modf(3.7))                      --> 3   0.7    （整数部分、小数部分）
```

### ⭐ 多返回值的“调整”规则（重要）

函数调用返回多个值时，**结果会被怎样截取，取决于它出现的位置**：

```lua
local function f() return 1, 2, 3 end

-- ① 作为赋值的右侧：按左边变量数量接收
local a, b = f()
print(a, b)                   --> 1   2
local x, y, z, w = f()
print(x, y, z, w)             --> 1   2   3   nil

-- ② 放在表达式列表的【最后】：全部展开
print(f())                    --> 1   2   3
print(10, f())                --> 10  1   2   3
local t = {f()}
print(#t)                     --> 3

-- ③ 放在表达式列表的【中间/开头】：只取第 1 个值
print(f(), 10)                --> 1   10
local t2 = {f(), f()}
print(#t2)                    --> 4   （第一个 f() 只取 1 个，第二个全展开：1,1,2,3）

-- ④ 用括号包起来：强制只取第 1 个值
print((f()))                  --> 1
```

一句话记忆：**只有出现在列表末尾的函数调用才会展开全部返回值**，其他位置只取第一个。

### 实用例子：返回“结果 + 错误信息”

这是 Lua 里处理错误的惯用模式：

```lua
local function divide(a, b)
    if b == 0 then
        return nil, "除数不能为 0"
    end
    return a / b
end

local r, err = divide(10, 0)
if not r then
    print("出错: " .. err)    --> 出错: 除数不能为 0
end

print(divide(10, 4))          --> 2.5
```

## 6.3 可变参数 `...`

参数列表末尾写 `...` 表示可以接收任意个参数。

```lua
local function show(...)
    print("参数个数:", select("#", ...))
    print(...)
end

show()                        --> 参数个数: 0
show(1, 2, 3)                 --> 参数个数: 3   然后输出  1   2   3
```

### 三种使用 `...` 的方式

**① 放进表里：`{...}`**

```lua
local function sum(...)
    local s = 0
    for _, v in ipairs({...}) do
        s = s + v
    end
    return s
end
print(sum(1, 2, 3, 4))        --> 10
```

**② `select("#", ...)` 取个数；`select(n, ...)` 取第 n 个及之后的所有参数**

```lua
local function demo(...)
    print(select("#", ...))        -- 参数总个数
    print(select(2, ...))          -- 从第 2 个开始的所有参数
    print((select(2, ...)))        -- 只要第 2 个（括号截断）
    print(select(-1, ...))         -- 最后一个
end

demo("a", "b", "c")
--> 3
--> b   c
--> b
--> c
```

**③ `table.pack(...)` 与 `table.unpack(t)`**

```lua
local t = table.pack(10, nil, 30)
print(t.n)                         --> 3     （n 字段记录真实个数）

print(table.unpack({1, 2, 3}))     --> 1   2   3
local a, b = table.unpack({"x", "y", "z"})
print(a, b)                        --> x   y
```

### ⚠️ 参数中含有 nil 时，别用 `#{...}` 数个数

```lua
local function count(...)
    return #{...}, select("#", ...)
end
print(count(1, nil, nil))          --> 1   3     （#{...} 不可靠，select("#") 才准确）
```

### 向其他函数“透传”参数

```lua
local function log(fmt, ...)
    print(string.format("[LOG] " .. fmt, ...))
end
log("用户 %s 登录, 年龄 %d", "Tom", 20)   --> [LOG] 用户 Tom 登录, 年龄 20
```

### 用表模拟“命名参数”

参数很多时，传一个表更清晰（并配合 5.1 里省略括号的写法）：

```lua
local function createWindow(opts)
    local w = opts.width  or 800
    local h = opts.height or 600
    local title = opts.title or "Untitled"
    print(title, w, h)
end

createWindow {title = "主窗口", width = 1024}   --> 主窗口   1024   600
```

## 6.4 匿名函数

没有名字的函数，常用于**临时传递**。

```lua
-- 直接调用
print((function(a, b) return a + b end)(3, 4))   --> 7

-- 作为回调参数
local function repeatDo(n, action)
    for i = 1, n do action(i) end
end

repeatDo(3, function(i)
    print("第 " .. i .. " 次")
end)
```

### 典型场景一：`table.sort` 自定义排序

```lua
local people = {
    {name = "Tom",  age = 30},
    {name = "Anna", age = 25},
    {name = "Bob",  age = 35},
}

table.sort(people, function(a, b) return a.age < b.age end)

for _, p in ipairs(people) do
    print(p.name, p.age)
end
--> Anna 25 / Tom 30 / Bob 35
```

### 典型场景二：高阶函数（map / filter）

函数是一等公民，可以接收函数、返回函数：

```lua
local function map(t, f)
    local r = {}
    for i, v in ipairs(t) do r[i] = f(v) end
    return r
end

local function filter(t, pred)
    local r = {}
    for _, v in ipairs(t) do
        if pred(v) then r[#r + 1] = v end
    end
    return r
end

local nums = {1, 2, 3, 4, 5, 6}
local squares = map(nums, function(x) return x * x end)
local evens   = filter(nums, function(x) return x % 2 == 0 end)

print(table.concat(squares, ","))   --> 1,4,9,16,25,36
print(table.concat(evens, ","))     --> 2,4,6
```

## 6.5 递归

函数调用自己。**局部函数要用 `local function` 形式定义，才能在函数体内引用自己**：

```lua
local function fact(n)
    if n <= 1 then return 1 end
    return n * fact(n - 1)
end
print(fact(5))                      --> 120
```

> ⚠️ **易错点**：
>
> ```lua
> local fact = function(n)
>     if n <= 1 then return 1 end
>     return n * fact(n - 1)     -- ❌ 此时 local fact 还未生效，这里的 fact 是全局变量（nil）
> end
> print(fact(5))                 -- 报错：attempt to call a nil value (global 'fact')
> ```
> 原因：`local x = 表达式` 中，新变量 `x` 要等整条语句结束后才可见。
> 修复一：改写成 `local function fact(n)`；
> 修复二：先 `local fact`，再 `fact = function ... end`。

斐波那契数列：

```lua
local function fib(n)
    if n <= 2 then return 1 end
    return fib(n - 1) + fib(n - 2)
end
for i = 1, 10 do io.write(fib(i), " ") end
print()                             --> 1 1 2 3 5 8 13 21 34 55
```

## 6.6 小结与练习

**小结：**

- 函数是值，可赋值、传参、返回、存进表里；优先用 `local function`
- 参数：缺的补 `nil`，多的丢弃
- 多返回值：**只有位于列表末尾的调用才展开全部**；用 `(f())` 截成一个
- 可变参数 `...`：配合 `select("#", ...)`、`table.pack`、`table.unpack`
- 递归函数要用 `local function` 声明

**练习 1：** 下面输出什么？

```lua
local function f() return 1, 2, 3 end
print(f(), f())
print((f()), f())
print({f(), f()}[1], #{f(), f()})
```

> 提示：第三行语法上不能直接对表构造器取下标，请改成 `local t = {f(), f()}; print(t[1], #t)`。

<details>
<summary>答案</summary>

```
1   1   2   3        第一个 f() 在中间只取 1 个，第二个展开
1   1   2   3        (f()) 取 1 个，后面的 f() 展开
1   4                t = {1, 1, 2, 3}
```

</details>

**练习 2：** 写一个函数 `average(...)`，返回任意个数的平均值；没有参数时返回 `nil`。

<details>
<summary>参考答案</summary>

```lua
local function average(...)
    local n = select("#", ...)
    if n == 0 then return nil end
    local s = 0
    for i = 1, n do
        s = s + (select(i, ...))
    end
    return s / n
end

print(average(1, 2, 3, 4))   --> 2.5
print(average())             --> nil
```

</details>

---

# 第 7 节　作用域与闭包

## 7.1 词法作用域（Lexical Scoping）

Lua 采用**词法作用域**：变量能在哪里被访问，由它在**代码中的书写位置**决定，而不是调用时的情况。

规则：

1. 局部变量从**声明之后**开始可见，到所在**代码块**结束为止
2. 内层代码块可以访问外层的局部变量
3. 内层同名变量会**遮蔽（shadow）**外层变量

```lua
local x = 10                   -- 外层 x

local function show()
    print(x)                   -- 能访问外层的 x
end

do
    local x = 20               -- 遮蔽外层的 x，只在 do...end 内有效
    print(x)                   --> 20
    show()                     --> 10   ⭐ show 看到的是它定义处能看到的 x，而不是调用处的 x
end

print(x)                       --> 10
```

> 最后一个例子体现了词法作用域：`show` 里的 `x` 在 `show` **书写的位置**就已经确定指向外层 `x = 10`。

### 声明顺序

局部变量必须**先声明后使用**，否则访问到的是全局变量：

```lua
local function a()
    return b()                 -- 此时 b 还没声明为局部，这里引用的是全局 b
end

local function b() return "b" end

-- print(a())                  -- 报错：attempt to call a nil value (global 'b')
```

修复：提前声明。

```lua
local b                        -- 先声明
local function a() return b() end
b = function() return "b" end  -- 再赋值（不要再写 local）
print(a())                     --> b
```

## 7.2 Upvalue（上值）

当一个函数**引用了外层函数的局部变量**，这个被引用的外层局部变量就叫该函数的 **upvalue**。

```lua
local function outer()
    local count = 0                  -- outer 的局部变量
    local function inner()
        count = count + 1            -- count 对 inner 来说是 upvalue
        return count
    end
    return inner
end
```

对 `inner` 来说：

- 它自己的局部变量 / 参数：普通局部变量
- 来自外层函数的局部变量（`count`）：**upvalue**
- 全局变量：既不是局部也不是 upvalue，要通过 `_ENV`（`_G`）访问

## 7.3 闭包（Closure）

**闭包 = 函数 + 它所引用的 upvalue。**

关键现象：外层函数返回之后，它的局部变量本来该消失了，但**只要内层函数还在，被它引用的变量就会被一直保留**。

```lua
local function counter()
    local count = 0
    return function()
        count = count + 1
        return count
    end
end

local c1 = counter()
print(c1())                    --> 1
print(c1())                    --> 2
print(c1())                    --> 3

local c2 = counter()           -- 再次调用，得到全新的 count
print(c2())                    --> 1   （与 c1 互不影响）
print(c1())                    --> 4
```

要点：

- **每次调用 `counter()` 都会创建一个新的闭包**，各自拥有独立的 `count`
- `count` 是被保存下来的“私有状态”，外部无法直接访问

### 多个闭包共享同一个 upvalue

```lua
local function makeAccount(balance)
    local function deposit(n)  balance = balance + n end
    local function withdraw(n)
        if n > balance then return false, "余额不足" end
        balance = balance - n
        return true
    end
    local function get() return balance end
    return deposit, withdraw, get
end

local deposit, withdraw, get = makeAccount(100)
deposit(50)
print(get())                   --> 150
print(withdraw(200))           --> false   余额不足
print(withdraw(30))            --> true
print(get())                   --> 120
-- 外部没有办法直接修改 balance，只能通过这三个函数：这就是“封装”
```

### ⭐ 循环中的闭包

`for` 循环里，**每一轮迭代都会创建一个新的循环变量**，所以闭包捕获到的是各自独立的值：

```lua
local fns = {}
for i = 1, 3 do
    fns[i] = function() return i end
end
print(fns[1](), fns[2](), fns[3]())   --> 1   2   3
```

但如果捕获的是循环**外部**声明的变量，大家共享同一个：

```lua
local fns = {}
local j = 1
while j <= 3 do
    fns[j] = function() return j end   -- 捕获的是同一个 j
    j = j + 1
end
print(fns[1](), fns[2](), fns[3]())   --> 4   4   4   ⭐ 共享的 j 最后是 4
```

修复：在循环体内再声明一个局部变量来“固定”当前值：

```lua
local fns = {}
local j = 1
while j <= 3 do
    local k = j                        -- 每轮新建一个 k
    fns[j] = function() return k end
    j = j + 1
end
print(fns[1](), fns[2](), fns[3]())   --> 1   2   3
```

## 7.4 闭包的常见应用

### ① 函数工厂：生成“定制化”的函数

```lua
local function multiplier(factor)
    return function(x) return x * factor end
end

local double = multiplier(2)
local triple = multiplier(3)
print(double(10), triple(10))         --> 20   30
```

```lua
local function makeGreeter(greeting)
    return function(name) return greeting .. ", " .. name .. "!" end
end
local hello = makeGreeter("Hello")
local hi    = makeGreeter("你好")
print(hello("Lua"))                   --> Hello, Lua!
print(hi("世界"))                      --> 你好, 世界!
```

### ② 私有状态 / 数据封装

前面的计数器、账户就是例子。再看一个“ID 生成器”：

```lua
local nextId = (function()
    local id = 0
    return function()
        id = id + 1
        return id
    end
end)()

print(nextId(), nextId(), nextId())   --> 1   2   3
```

### ③ 缓存（memoize）

```lua
local function memoize(f)
    local cache = {}                   -- 闭包保存的缓存表
    return function(x)
        local v = cache[x]
        if v == nil then
            v = f(x)
            cache[x] = v
        end
        return v
    end
end

local slowSquare = function(x)
    print("计算 " .. x)
    return x * x
end

local fastSquare = memoize(slowSquare)
print(fastSquare(4))                  --> 计算 4  然后  16
print(fastSquare(4))                  --> 16      （命中缓存，不再计算）
```

### ④ 自定义迭代器

泛型 `for` 每次会调用迭代器函数，直到它返回 `nil`。闭包正好能保存“当前进度”：

```lua
local function range(n)
    local i = 0
    return function()
        i = i + 1
        if i <= n then return i end    -- 超出范围时隐式返回 nil，循环结束
    end
end

for v in range(3) do
    print(v)                           --> 1 2 3
end
```

再来一个遍历数组的迭代器：

```lua
local function values(t)
    local i = 0
    return function()
        i = i + 1
        return t[i]
    end
end

for v in values({"a", "b", "c"}) do
    io.write(v, " ")
end
print()                                --> a b c
```

### ⑤ 回调与事件处理

```lua
local function onClick(name)
    local times = 0
    return function()
        times = times + 1
        print(name .. " 被点击了 " .. times .. " 次")
    end
end

local btn = onClick("确定按钮")
btn()                                  --> 确定按钮 被点击了 1 次
btn()                                  --> 确定按钮 被点击了 2 次
```

## 7.5 尾调用（Tail Call）

### 什么是尾调用

当函数的**最后一个动作**是 `return 另一个函数调用` 时，这就是尾调用：

```lua
local function f(x)
    return g(x)               -- ✅ 尾调用
end
```

### Lua 支持“正确的尾调用”

尾调用**不会占用新的栈帧**，因此**即使递归非常深也不会栈溢出**。

对比：求 1 到 n 的和

```lua
-- ❌ 非尾递归：每层都要等下一层返回后再做加法，栈会越来越深
local function sumBad(n)
    if n == 0 then return 0 end
    return n + sumBad(n - 1)           -- 返回前还要做 n + ...，不是尾调用
end
-- print(sumBad(1000000))              -- 报错：stack overflow

-- ✅ 尾递归：把中间结果通过参数 acc（累加器）传下去
local function sumGood(n, acc)
    acc = acc or 0
    if n == 0 then return acc end
    return sumGood(n - 1, acc + n)     -- 最后一步就是调用自己 → 尾调用
end
print(sumGood(1000000))                --> 500000500000
```

### 哪些写法才是尾调用？

```lua
return f(x)           -- ✅ 尾调用
return f(x) + 1       -- ❌ 调用后还有运算
return (f(x))         -- ❌ 括号截断返回值，需要额外处理，不是尾调用
return f(x), 1        -- ❌ 还有其他返回值
f(x)                  -- ❌ 这只是普通调用语句，没有 return
```

### 应用：状态机

用尾调用写状态机，状态切换时不会无限增长调用栈：

```lua
local stateB                           -- 提前声明，因为 stateA 里要引用它

local function stateA(n)
    if n <= 0 then return "结束于 A" end
    return stateB(n - 1)               -- 尾调用
end

function stateB(n)
    if n <= 0 then return "结束于 B" end
    return stateA(n - 1)               -- 尾调用
end

print(stateA(1000001))                 --> 结束于 B   （深度超过百万也没问题）
```

> 注意：这里 `stateB` 前面已经 `local stateB` 声明过，所以后面 `function stateB(n)` 是给这个局部变量赋值，而不是新建全局函数。

## 7.6 小结与练习

**小结：**

- Lua 是**词法作用域**：内层函数可以访问外层的局部变量
- **upvalue** = 被内层函数引用的外层局部变量
- **闭包** = 函数 + 它的 upvalue；每次创建都是独立的一份
- `for` 循环每轮都有新的循环变量；`while` 外部变量则是共享的
- 闭包的应用：函数工厂、私有状态、缓存、迭代器、回调
- `return f(...)` 是**尾调用**，不增长调用栈，适合深度递归

**练习 1：** 下面输出什么？

```lua
local function make()
    local n = 0
    return function() n = n + 1; return n end,
           function() n = n - 1; return n end
end

local inc, dec = make()
print(inc(), inc(), dec())
local inc2 = make()
print(inc2())
```

<details>
<summary>答案</summary>

```
1   2   1
1
```

`inc` 与 `dec` 共享同一个 `n`；再次调用 `make()` 得到全新的 `n`。

</details>

**练习 2：** 写一个函数 `once(f)`，返回一个新函数：第一次调用时执行 `f` 并记住结果，之后再调用都直接返回第一次的结果。

<details>
<summary>参考答案</summary>

```lua
local function once(f)
    local called, result = false, nil
    return function(...)
        if not called then
            called = true
            result = f(...)
        end
        return result
    end
end

local init = once(function() print("初始化..."); return 42 end)
print(init())        --> 初始化...  然后  42
print(init())        --> 42   （不再执行）
```

</details>

**练习 3：** 用尾递归写一个 `factorial(n, acc)`，计算 `n!`。

<details>
<summary>参考答案</summary>

```lua
local function factorial(n, acc)
    acc = acc or 1
    if n <= 1 then return acc end
    return factorial(n - 1, acc * n)
end
print(factorial(10))     --> 3628800
```

</details>

---

# 本部分速查表

**分支：** `if ... then ... elseif ... then ... else ... end`；没有 `switch`，用 if 链或表分发

**循环：**

| 语句 | 特点 |
|------|------|
| `while cond do ... end` | 先判断后执行 |
| `repeat ... until cond` | 先执行后判断，**cond 为真时退出** |
| `for i = a, b, step do` | 闭区间，三个值只算一次 |
| `for k, v in pairs(t) do` | 遍历所有键值，顺序不定 |
| `for i, v in ipairs(t) do` | 从 1 开始，遇 `nil` 停止 |

**跳转：** `break`；`goto 标签` + `::标签::`；`continue` 用 `goto continue` 模拟

**函数：** 优先 `local function`；参数缺补 `nil` 多丢弃；多返回值只有**末尾**的调用才展开，`(f())` 截成一个

**可变参数：** `...`、`select("#", ...)`、`select(n, ...)`、`table.pack`、`table.unpack`

**闭包：** 函数 + upvalue；每次创建独立；`for` 每轮新变量，`while` 外部变量共享

**尾调用：** `return f(...)` 不增长栈；`return f(...) + 1` 或 `return (f(...))` 不是

---

*下一节预告：第三部分《核心数据结构》——Table（表）、字符串处理与模式匹配、标准库速览。*
