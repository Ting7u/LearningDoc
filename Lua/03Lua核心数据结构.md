
> 本文基于 **Lua 5.4**。示例中的 `--> 结果` 表示输出。
# 第 8 节　Table（表）
**table 是 Lua 唯一的数据结构**。数组、字典、集合、队列、栈、对象、模块……全都用 table 实现。
## 8.1 创建表
```lua
local empty = {}                          -- 空表

local arr = {10, 20, 30}                  -- 数组风格：键自动为 1, 2, 3

local dict = {name = "Tom", age = 20}     -- 字典风格：键值对

local mixed = {                           -- 混合
    "a", "b",                             -- [1] [2]
    x = 10,
    [100] = "hundred",                    -- 用 [表达式] 指定任意键
    ["my key"] = "含空格的键",
    f = function() return "hi" end,       -- 值可以是函数
}
print(mixed[1], mixed.x, mixed[100], mixed["my key"], mixed.f())
--> a   10   hundred   含空格的键   hi
```

小提示：
- 元素之间用 `,` 或 `;` 分隔均可，**最后一项后可以多写一个逗号**（方便增删行）
- 数组风格的元素按出现顺序编号，**从 1 开始**
## 8.2 访问与修改
```lua
local p = {name = "Tom"}

print(p.name)            --> Tom        点语法（键是合法标识符时）
print(p["name"])         --> Tom        方括号语法（任意键）
print(p.age)             --> nil        不存在的键返回 nil

p.age = 20               -- 新增
p.name = "Jerry"         -- 修改
p.age = nil              -- 删除
```
### `t.x` 与 `t[x]` 的区别
```lua
local t = {}
local key = "name"

t.key = 1                -- 等价于 t["key"] = 1，键是字符串 "key"
t[key] = 2               -- 键是变量 key 的值，即 "name"

print(t.key, t.name)     --> 1   2
```
### 键可以是什么？
**除了 `nil` 和 `NaN`，任何值都可以做键**（数字、字符串、布尔、表、函数……）。

```lua
local t = {}
t[1] = "数字键"
t["1"] = "字符串键"      -- 与数字键 1 不同！
t[true] = "布尔键"
local k = {}
t[k] = "用表作键"

print(t[1], t["1"], t[true], t[k])   --> 数字键   字符串键   布尔键   用表作键
-- t[nil] = 1                         -- ❌ 报错：index is nil
```

浮点数键如果有整数值，会自动规范成整数键：

```lua
local t = {}
t[1.0] = "x"
print(t[1])              --> x      （1.0 和 1 是同一个键）
```
## 8.3 数组用法
### 基础操作
```lua
local fruits = {"apple", "banana", "cherry"}

print(fruits[1])              --> apple     ⭐ 下标从 1 开始
print(#fruits)                --> 3
print(fruits[#fruits])        --> cherry    最后一个元素

fruits[#fruits + 1] = "date"  -- 末尾追加
print(#fruits)                --> 4
```
### 长度运算符 `#` 与“空洞”
`#t` 返回的是一个“边界”：`t[n] ~= nil` 且 `t[n+1] == nil` 的 n。**若数组中间有 `nil`，结果不可靠**。

```lua
print(#{1, 2, 3})             --> 3
print(#{})                    --> 0
print(#{n = 1})               --> 0         （只有字典部分，没有数组部分）

local t = {1, 2, 3, 4, 5}
t[3] = nil                    -- 中间挖了个洞
print(#t)                     --> 5 或 2，行为不确定！
```

> 💡 需要“可能含 nil 的列表”时，自己维护长度（如 `n` 字段，`table.pack` 就是这么做的）。**删除元素请用 `table.remove`，不要直接赋 nil。**

## 8.4 嵌套表与二维数组

```lua
local student = {
    name = "Lucy",
    scores = {math = 90, english = 85},
    friends = {"Tom", "Anna"},
}
print(student.scores.math)         --> 90
print(student.friends[2])          --> Anna
student.scores.art = 95            -- 往内层表加字段
```

二维数组（矩阵）：

```lua
local grid = {}
for i = 1, 3 do
    grid[i] = {}                   -- 先创建每一行
    for j = 1, 3 do
        grid[i][j] = i * j
    end
end

print(grid[2][3])                  --> 6

for i = 1, 3 do
    print(table.concat(grid[i], " "))
end
--> 1 2 3
--> 2 4 6
--> 3 6 9
```

> ⚠️ 访问 `a.b.c` 时，如果 `a.b` 是 `nil` 会报错：`attempt to index a nil value (field 'b')`。可以写成 `a.b and a.b.c` 防护。

## 8.5 遍历

```lua
local t = {10, 20, 30, x = 1, y = 2}

-- ipairs：只遍历 1,2,3...（数组部分），遇 nil 停止，按顺序
for i, v in ipairs(t) do print(i, v) end
--> 1 10 / 2 20 / 3 30

-- pairs：遍历全部键值对，顺序不保证
for k, v in pairs(t) do print(k, v) end
--> 1 10 / 2 20 / 3 30 / x 1 / y 2   （字典部分的顺序不固定）

-- 数值 for：需要下标、倒序或指定范围时
for i = #t, 1, -1 do print(t[i]) end   --> 30 20 10
```

判断空表：

```lua
local function isEmpty(t) return next(t) == nil end
print(isEmpty({}), isEmpty({1}))       --> true   false
```

遍历中删除元素：**倒着遍历**，避免因元素前移而跳过：

```lua
local nums = {1, 2, 3, 4, 5, 6}
for i = #nums, 1, -1 do
    if nums[i] % 2 == 0 then
        table.remove(nums, i)
    end
end
print(table.concat(nums, ","))         --> 1,3,5
```

## 8.6 table 库常用函数

### table.insert / table.remove

```lua
local t = {"a", "b", "c"}

table.insert(t, "d")           -- 末尾追加         {a b c d}
table.insert(t, 1, "z")        -- 插入到位置 1     {z a b c d}（后面元素后移）
print(table.concat(t, ","))    --> z,a,b,c,d

print(table.remove(t))         --> d      删除并返回最后一个
print(table.remove(t, 1))      --> z      删除位置 1，后面元素前移
print(table.concat(t, ","))    --> a,b,c
```

### table.concat：把数组拼成字符串

```lua
print(table.concat({1, 2, 3}))                  --> 123
print(table.concat({1, 2, 3}, ", "))            --> 1, 2, 3
print(table.concat({"a", "b", "c", "d"}, "-", 2, 3))   --> b-c    （指定范围）
-- 元素必须是字符串或数字，否则报错
```

### table.sort：排序（原地修改）

```lua
local nums = {5, 2, 8, 1, 9}
table.sort(nums)                               -- 默认升序
print(table.concat(nums, ","))                 --> 1,2,5,8,9

table.sort(nums, function(a, b) return a > b end)   -- 降序
print(table.concat(nums, ","))                 --> 9,8,5,2,1

local words = {"banana", "Apple", "cherry"}
table.sort(words)                              -- 字符串按字典序（大写在前）
print(table.concat(words, ","))                --> Apple,banana,cherry

-- 按字段排序
local people = {{n = "Tom", age = 30}, {n = "Anna", age = 25}}
table.sort(people, function(a, b) return a.age < b.age end)
print(people[1].n)                             --> Anna
```

> ⚠️ 比较函数必须是严格的“小于”（`<` / `>`），**不要写 `<=` / `>=`**，否则可能报 `invalid order function for sorting`。排序不是稳定排序；元素类型混杂（数字与字符串）会报错。

### table.unpack / table.pack

```lua
print(table.unpack({1, 2, 3}))       --> 1   2   3
print(table.unpack({1, 2, 3}, 2))    --> 2   3
local p = table.pack("a", nil, "c")
print(p.n, p[1], p[3])               --> 3   a   c
```

### table.move：移动 / 复制一段元素

`table.move(源表, 起, 止, 目标起点 [, 目标表])`

```lua
local a = {1, 2, 3}
table.move(a, 1, 3, 2)                 -- 把 a[1..3] 移到 a[2..4]
print(table.concat(a, ","))            --> 1,1,2,3

local src = {"x", "y", "z"}
local copy = table.move(src, 1, #src, 1, {})   -- 复制到新表
print(table.concat(copy, ","))         --> x,y,z
```

## 8.7 引用语义与拷贝

表是**引用类型**：赋值、传参传递的都是“引用”，不会复制内容。

```lua
local a = {1, 2, 3}
local b = a
b[1] = 100
print(a[1])                  --> 100     a 和 b 是同一个表

local function modify(t) t.x = "changed" end
local obj = {}
modify(obj)
print(obj.x)                 --> changed  函数内修改会影响外面
```

### 浅拷贝

只复制第一层，内层的表仍然共享：

```lua
local function shallowCopy(t)
    local c = {}
    for k, v in pairs(t) do c[k] = v end
    return c
end

local orig = {1, 2, inner = {x = 1}}
local c = shallowCopy(orig)
c[1] = 99
c.inner.x = 42
print(orig[1], orig.inner.x)   --> 1   42   （[1] 没变；inner 是共享的）
```

### 深拷贝

递归复制所有层，并用 `seen` 表处理循环引用：

```lua
local function deepCopy(t, seen)
    if type(t) ~= "table" then return t end
    seen = seen or {}
    if seen[t] then return seen[t] end      -- 已经复制过（处理循环引用）
    local c = {}
    seen[t] = c
    for k, v in pairs(t) do
        c[deepCopy(k, seen)] = deepCopy(v, seen)
    end
    return c
end

local orig = {1, 2, inner = {x = 1}}
local c = deepCopy(orig)
c.inner.x = 42
print(orig.inner.x)            --> 1     （互不影响）
```

## 8.8 表的常见用法

### ① 集合（Set）：用键存在性表示元素是否存在

```lua
local set = {}
for _, v in ipairs({"a", "b", "a", "c", "b"}) do
    set[v] = true                  -- 自动去重
end
print(set["a"], set["z"])          --> true   nil

for k in pairs(set) do io.write(k, " ") end   -- 顺序不定
print()
```

### ② 计数器

```lua
local count = {}
for ch in ("hello"):gmatch(".") do
    count[ch] = (count[ch] or 0) + 1
end
print(count.l, count.h)            --> 2   1
```

### ③ 栈（后进先出）

```lua
local stack = {}
table.insert(stack, "A")           -- push
table.insert(stack, "B")
print(table.remove(stack))         --> B    pop
print(table.remove(stack))         --> A
```

### ④ 队列（先进先出）

`table.remove(t, 1)` 要整体前移，数据量大时慢。用头尾两个下标更高效：

```lua
local Queue = {first = 1, last = 0, items = {}}

function Queue.push(q, v)
    q.last = q.last + 1
    q.items[q.last] = v
end

function Queue.pop(q)
    if q.first > q.last then return nil end
    local v = q.items[q.first]
    q.items[q.first] = nil
    q.first = q.first + 1
    return v
end

Queue.push(Queue, "x"); Queue.push(Queue, "y")
print(Queue.pop(Queue), Queue.pop(Queue), Queue.pop(Queue))   --> x   y   nil
```

### ⑤ 用表做“枚举”

```lua
local Color = {RED = 1, GREEN = 2, BLUE = 3}
local c = Color.GREEN
if c == Color.GREEN then print("绿色") end   --> 绿色
```

### ⑥ 用函数表代替 switch

见第 5 节“表分发”。

## 8.9 小结与练习

**小结：**

- 表可以当数组（下标从 **1** 开始）、字典、对象、集合
- `ipairs` 遍历数组部分，`pairs` 遍历全部（无序）
- 数组避免出现空洞；删除用 `table.remove`
- 表是引用类型，复制要自己写浅拷贝 / 深拷贝
- 常用库函数：`insert` `remove` `concat` `sort` `unpack` `pack` `move`

**练习：** 下面输出什么？

```lua
local t = {1, 2, 3}
table.insert(t, 4)
table.insert(t, 1, 0)
print(table.remove(t))
print(table.remove(t, 2))
print(table.concat(t, ","))
```

<details>
<summary>答案</summary>

```
4
1
0,2,3
```

过程：`{1,2,3}` → `{1,2,3,4}` → `{0,1,2,3,4}`；删最后一个返回 4；删位置 2（值 1）返回 1；剩 `{0,2,3}`。

</details>

---

# 第 9 节　字符串处理

字符串是**不可变**的：所有“修改”函数都返回**新字符串**，原字符串不变。

```lua
local s = "hello"
local u = s:upper()
print(s, u)                  --> hello   HELLO
```

> 调用方式：`string.upper(s)` 与 `s:upper()` 完全等价。字符串字面量要加括号：`("abc"):upper()`。

## 9.1 基础函数

```lua
local s = "Hello, Lua"

print(#s, s:len())                 --> 10   10
print(s:upper())                   --> HELLO, LUA
print(s:lower())                   --> hello, lua
print(s:reverse())                 --> auL ,olleH
print(("ab"):rep(3))               --> ababab
print(("ab"):rep(3, "-"))          --> ab-ab-ab
```

### sub：截取子串

`s:sub(i, j)` 取第 i 到第 j 个字节（闭区间），**负数表示从末尾数起**（`-1` 是最后一个）。

```lua
local s = "hello"
print(s:sub(2, 4))        --> ell
print(s:sub(2))           --> ello      省略 j：到末尾
print(s:sub(-3))          --> llo       最后 3 个
print(s:sub(2, -2))       --> ell       从第 2 个到倒数第 2 个
print(s:sub(3, 3))        --> l         取单个字符
print(s:sub(10))          -->           （空字符串，不报错）
```

遍历每个字符：

```lua
local s = "Lua"
for i = 1, #s do
    print(i, s:sub(i, i))
end
--> 1 L / 2 u / 3 a
```

### byte / char：字符与编码互转

```lua
print(("A"):byte())              --> 65
print(("ABC"):byte(1, -1))       --> 65   66   67
print(string.char(72, 105))      --> Hi
print(string.char(("a"):byte() + 1))   --> b     字母“后移”一位
```

## 9.2 string.format：格式化

和 C 的 `printf` 类似：

| 格式 | 含义 | 示例 | 结果 |
|------|------|------|------|
| `%d` | 整数 | `("%d"):format(42)` | `42` |
| `%5d` | 宽度 5，右对齐 | `("%5d|"):format(42)` | `   42|` |
| `%-5d` | 左对齐 | `("%-5d|"):format(42)` | `42   |` |
| `%05d` | 前面补 0 | `("%05d"):format(42)` | `00042` |
| `%f` | 浮点数 | `("%f"):format(3.14)` | `3.140000` |
| `%.2f` | 保留 2 位小数 | `("%.2f"):format(3.14159)` | `3.14` |
| `%e` | 科学计数法 | `("%e"):format(12345.678)` | `1.234568e+04` |
| `%s` | 字符串（任意值会 `tostring`） | `("%s"):format(true)` | `true` |
| `%10s` | 宽度 10 右对齐 | `("%10s|"):format("hi")` | `        hi|` |
| `%-10s` | 左对齐 | `("%-10s|"):format("hi")` | `hi        |` |
| `%x` / `%X` | 十六进制 | `("%x"):format(255)` | `ff` |
| `%o` | 八进制 | `("%o"):format(8)` | `10` |
| `%c` | 字符 | `("%c"):format(65)` | `A` |
| `%%` | 百分号本身 | `("%d%%"):format(50)` | `50%` |

```lua
print(string.format("姓名: %s, 年龄: %d, 身高: %.1f", "Tom", 20, 175.5))
--> 姓名: Tom, 年龄: 20, 身高: 175.5

print(string.format("%02d:%02d:%02d", 1, 5, 9))   --> 01:05:09
print(string.format("%d", 3.0))                   --> 3     （能无损转整数则可以）
-- print(string.format("%d", 3.5))                -- ❌ 报错：number has no integer representation
```

## 9.3 查找：string.find

`s:find(pattern [, init [, plain]])`

- 找到：返回**起始位置、结束位置**（以及捕获内容，见后）
- 没找到：返回 `nil`
- 第三个参数 `init`：从哪个位置开始找
- 第四个参数 `plain = true`：**按普通文本查找，不当作模式**

```lua
local s = "hello world"
print(s:find("o"))             --> 5   5
print(s:find("o", 6))          --> 8   8     从第 6 个字符开始找
print(s:find("wor"))           --> 7   9
print(s:find("xyz"))           --> nil

-- 判断是否包含
if s:find("world") then print("包含") end     --> 包含
```

**查找带特殊符号的普通文本，务必用 plain：**

```lua
print(("a.b"):find("."))            --> 1   1    ⚠️ "." 在模式里表示“任意字符”
print(("a.b"):find(".", 1, true))   --> 2   2    ✅ 按普通文本找
```

## 9.4 模式匹配（Lua 的“正则”）

> ⭐ **Lua 的模式（pattern）不是正则表达式**：它更简单、更小巧。**没有 `|` 或运算，没有 `{n,m}` 计数**，但够用。

### 字符类

| 写法 | 匹配 | 反义（大写） |
|------|------|-------------|
| `.` | 任意字符 | — |
| `%a` | 字母 | `%A` 非字母 |
| `%d` | 数字 | `%D` 非数字 |
| `%l` | 小写字母 | `%L` |
| `%u` | 大写字母 | `%U` |
| `%s` | 空白（空格、制表、换行等） | `%S` 非空白 |
| `%w` | 字母或数字 | `%W` |
| `%p` | 标点符号 | `%P` |
| `%x` | 十六进制字符 | `%X` |
| `%c` | 控制字符 | `%C` |
| `[abc]` | a、b、c 之一 | |
| `[a-z]` | 范围 | |
| `[^abc]` | 除 a、b、c 外的字符 | |
| `%.` `%%` `%(` … | `%` 加符号 = 转义，匹配符号本身 | |

### 量词（放在单个字符类后）

| 量词 | 含义 |
|------|------|
| `*` | 0 次或多次（贪婪，尽量多） |
| `+` | 1 次或多次（贪婪） |
| `-` | 0 次或多次（**懒惰**，尽量少） |
| `?` | 0 次或 1 次 |

### 锚点与捕获

| 写法 | 含义 |
|------|------|
| `^` | 模式开头：必须从字符串开头匹配 |
| `$` | 模式结尾：必须匹配到字符串末尾 |
| `( )` | 捕获：把括号内匹配到的内容单独返回 |
| `%1` `%2` | 反向引用：再次匹配第 1、2 个捕获的内容 |
| `%b()` | 匹配成对平衡的括号 `( )`（可换成其他字符对） |
| `%f[集合]` | 边界匹配（单词边界等） |

### string.match：提取第一个匹配

没有捕获时返回整个匹配；有捕获时返回各个捕获。**没找到返回 `nil`。**

```lua
print(("abc123def"):match("%a+"))          --> abc
print(("abc123def"):match("%d+"))          --> 123
print(("abc123"):match("^%a+$"))           --> nil    （不是整串都是字母）
print(("hello"):match("^h"))               --> h

-- 捕获
print(("key=value"):match("(%w+)=(%w+)"))  --> key   value

local y, m, d = ("2026-11-21"):match("^(%d+)-(%d+)-(%d+)$")
print(y, m, d)                             --> 2026   11   21   （都是字符串）
print(tonumber(y) + 1)                     --> 2027

-- 去除首尾空白（trim）
local function trim(s)
    return s:match("^%s*(.-)%s*$")
end
print("[" .. trim("   hello  ") .. "]")    --> [hello]
```

### 贪婪与懒惰：`*` / `+` 对比 `-`

```lua
local s = "<a><b>"
print(s:match("<(.*)>"))     --> a><b     贪婪：尽可能多
print(s:match("<(.-)>"))     --> a        懒惰：尽可能少
```

### 其他特殊用法

```lua
-- 反向引用：找连续重复的字母
print(("hello"):match("(%a)%1"))            --> l

-- 平衡括号
print(("f(a(b)c) + g"):match("%b()"))       --> (a(b)c)

-- 位置捕获：空括号 () 返回当前位置
print(("hello"):match("()ll"))              --> 3

-- find 同样支持捕获：先返回起止位置，再返回捕获
print(("THE (quick) fox"):find("%((%a+)%)"))   --> 5   11   quick
```

### string.gmatch：遍历所有匹配

返回一个迭代器，用于泛型 `for`：

```lua
-- 取出所有单词
for w in ("the quick brown fox"):gmatch("%a+") do
    io.write(w, "|")
end
print()                                     --> the|quick|brown|fox|

-- 解析键值对
local s = "a=1, b=2, c=3"
for k, v in s:gmatch("(%w+)=(%w+)") do
    print(k, v)
end
--> a 1 / b 2 / c 3

-- 提取所有数字
local sum = 0
for n in ("10 apples, 20 pears, 5 figs"):gmatch("%d+") do
    sum = sum + tonumber(n)
end
print(sum)                                  --> 35
```

实现 `split`（按分隔符拆分）：

```lua
local function split(s, sep)
    local result = {}
    for piece in (s .. sep):gmatch("(.-)" .. sep) do   -- 末尾补一个分隔符，方便统一处理
        result[#result + 1] = piece
    end
    return result
end

local parts = split("a,b,,c", ",")
print(#parts)                                --> 4
print(table.concat(parts, "|"))              --> a|b||c
```

> 注意：`sep` 会被当作**模式**。如果分隔符是 `.` `|` `+` 等特殊字符，需要转义（如 `%.`）。

## 9.5 替换：string.gsub

`s:gsub(pattern, repl [, n])` → 返回**新字符串**和**替换次数**（两个值）。

`repl` 可以是：**字符串**、**表**、**函数**。

### ① repl 是字符串

在替换串里 `%1`、`%2` 引用捕获，`%0` 引用整个匹配，`%%` 表示百分号：

```lua
print(("hello world"):gsub("o", "0"))              --> hell0 w0rld   2
print(("hello world"):gsub("(%w+)", "<%1>"))       --> <hello> <world>   2
print(("hello world"):gsub("%w+", "%0 %0"))        --> hello hello world world   2
print(("aaa"):gsub("a", "b", 2))                   --> bba   2        最多替换 2 次
print(("a b  c"):gsub("%s+", " "))                 --> a b c   2      压缩多余空白
```

只想要字符串、不要次数时，用括号截断或赋给单个变量：

```lua
local s = ("hello"):gsub("l", "L")     -- s 只接收第一个返回值
print(s)                               --> heLLo
print((("hello"):gsub("l", "L")))      --> heLLo
```

### ② repl 是表：用匹配内容作键去查表

找不到（值为 `nil` / `false`）则保留原文：

```lua
local vars = {name = "Tom", age = 20}
print(("Hi $name, you are $age"):gsub("%$(%w+)", vars))
--> Hi Tom, you are 20   2

print(("$a and $b"):gsub("%$(%w+)", {a = "1"}))
--> 1 and $b   2
```

### ③ repl 是函数：对每个匹配调用，返回值作为替换

```lua
print(("hello world"):gsub("%w+", string.upper))      --> HELLO WORLD   2

print(("1 2 3"):gsub("%d+", function(n)
    return tonumber(n) * 2
end))                                                  --> 2 4 6   3

-- 单词首字母大写
local title = ("hello lua world"):gsub("(%a)([%w_']*)", function(first, rest)
    return first:upper() .. rest:lower()
end)
print(title)                                           --> Hello Lua World
```

### 统计出现次数

```lua
local _, count = ("banana"):gsub("a", "a")    -- 替换成自己，只关心次数
print(count)                                   --> 3
```

### 转义特殊字符

模式中的特殊字符：`^ $ ( ) % . [ ] * + - ?`。想匹配它们本身，前面加 `%`。

```lua
print(("1+1=2"):find("1+1"))       --> nil     ⚠️ 这里 + 是量词（“1 个或多个 1”），不是加号
print(("1+1=2"):find("1%+1"))      --> 1  3    ✅

-- 通用转义函数
local function escape(s)
    return (s:gsub("[%^%$%(%)%%%.%[%]%*%+%-%?]", "%%%0"))
end
print(escape("1+1=2"))             --> 1%+1=2
```

## 9.6 UTF-8 与中文处理

Lua 的字符串是**字节序列**，`#` 和 `sub` 都按字节计算。中文在 UTF-8 中通常占 3 字节。

```lua
local s = "你好Lua"
print(#s)                          --> 9      3 + 3 + 3
print(s:sub(1, 3))                 --> 你     按字节切，恰好是第一个汉字
print(s:sub(1, 2))                 --> 乱码   切到了汉字中间
```

处理字符请用 `utf8` 库：

```lua
local s = "你好Lua"
print(utf8.len(s))                 --> 5      字符数

-- 遍历每个字符（码点）
for pos, code in utf8.codes("你好") do
    print(pos, code, utf8.char(code))
end
--> 1   20320   你
--> 4   22909   好

-- 用 utf8.charpattern 逐字符匹配
for ch in s:gmatch(utf8.charpattern) do
    io.write(ch, "|")
end
print()                            --> 你|好|L|u|a|

-- 取第 n 个字符：先把字符放进表里
local chars = {}
for ch in s:gmatch(utf8.charpattern) do chars[#chars + 1] = ch end
print(chars[2])                    --> 好

-- 反转含中文的字符串
print(table.concat(chars):len())   --> 9
local rev = {}
for i = #chars, 1, -1 do rev[#rev + 1] = chars[i] end
print(table.concat(rev))           --> auL好你
```

## 9.7 性能提示：字符串拼接

字符串不可变，循环里反复 `s = s .. x` 会不断创建新字符串。大量拼接用 `table.concat`：

```lua
-- ❌ 慢
local s = ""
for i = 1, 10000 do s = s .. i .. "," end

-- ✅ 快
local buf = {}
for i = 1, 10000 do buf[#buf + 1] = i end
local s2 = table.concat(buf, ",")
```

## 9.8 小结与练习

**小结：**

- 字符串不可变，下标从 1 开始，`sub` 支持负数下标
- `format` 做格式化；`find` 找位置；`match` 取内容；`gmatch` 遍历；`gsub` 替换
- 模式 ≠ 正则：没有 `|` 和 `{n,m}`；`-` 是懒惰量词；`%` 用来转义和表示字符类
- `find` 想按普通文本查找，传第 4 个参数 `true`
- 中文用 `utf8` 库处理

**练习 1：** 下面输出什么？

```lua
print(("Hello, World"):gsub("o", "0"))
print(("a1b22c333"):match("%d+"))
print(("a1b22c333"):gsub("%d+", "#"))
print(("key = value"):match("^(%w+)%s*=%s*(%w+)$"))
```

<details>
<summary>答案</summary>

```
Hell0, W0rld   2
1
a#b#c#   3
key   value
```

</details>

**练习 2：** 统计字符串 `"Programming in Lua"` 中元音字母（a e i o u，不分大小写）的个数。

<details>
<summary>参考答案</summary>

```lua
local s = "Programming in Lua"
local _, n = s:gsub("[aeiouAEIOU]", "")
print(n)    --> 6
```

</details>

**练习 3：** 写一个函数 `parseDate(s)`，把 `"2026-11-21"` 解析成表 `{year=2026, month=11, day=21}`，格式不对返回 `nil`。

<details>
<summary>参考答案</summary>

```lua
local function parseDate(s)
    local y, m, d = s:match("^(%d%d%d%d)-(%d%d)-(%d%d)$")
    if not y then return nil end
    return {year = tonumber(y), month = tonumber(m), day = tonumber(d)}
end

local r = parseDate("2026-11-21")
print(r.year, r.month, r.day)     --> 2026   11   21
print(parseDate("2026/11/21"))    --> nil
```

</details>

---

# 第 10 节　标准库速览

Lua 的标准库很精简。常用的库：

| 库 | 用途 |
|----|------|
| 基础函数（`_G`） | `print` `type` `pairs` `pcall` `require` `setmetatable` 等，无需前缀 |
| `string` | 字符串（第 9 节） |
| `table` | 表操作（第 8 节） |
| `math` | 数学 |
| `os` | 操作系统：时间、环境变量、退出 |
| `io` | 输入输出、文件 |
| `utf8` | UTF-8 支持 |
| `coroutine` | 协程（第 15 章） |
| `debug` / `package` | 调试 / 模块加载 |

## 10.1 基础函数速览

```lua
print(type(42), tostring(42), tonumber("42"))   --> number   42   42

-- assert：条件为假时抛出错误，为真则原样返回参数
local n = assert(tonumber("12"), "不是数字")
print(n)                                         --> 12

-- select / next / rawget / rawset / rawequal / rawlen：见前面章节与元表章节

-- load：把字符串当代码执行
local f = load("return 1 + 2")
print(f())                                       --> 3
print(load("return +"))                          --> nil   [string "return +"]:1: unexpected symbol near <eof>

-- pcall：保护调用，捕获错误（第 14 章详讲）
print(pcall(error, "boom"))                      --> false   boom
print(pcall(function() return "ok" end))         --> true   ok

-- collectgarbage："count" 返回当前内存使用量（KB）
print(collectgarbage("count") > 0)               --> true
```

## 10.2 math 库

### 常量

```lua
print(math.pi)               --> 3.1415926535898
print(math.huge)             --> inf         正无穷
print(-math.huge)            --> -inf
print(math.maxinteger)       --> 9223372036854775807
print(math.mininteger)       --> -9223372036854775808
```

### 取整与绝对值、极值

```lua
print(math.floor(3.7), math.floor(-3.5))   --> 3   -4     向下取整
print(math.ceil(3.2), math.ceil(-3.5))     --> 4   -3     向上取整
print(math.abs(-5))                        --> 5
print(math.max(3, 9, 2))                   --> 9
print(math.min(3, 9, 2))                   --> 2
print(math.modf(3.7))                      --> 3.0   0.7   整数部分、小数部分

-- 四舍五入（Lua 没有内置 round）
local function round(x) return math.floor(x + 0.5) end
print(round(2.5), round(2.4), round(-2.5)) --> 3   2   -2
```

### 幂、根、对数、三角函数

```lua
print(math.sqrt(16))         --> 4.0
print(2 ^ 10)                --> 1024.0
print(math.exp(0))           --> 1.0
print(math.log(8, 2))        --> 3.0       log(x, 底)
print(math.log(100, 10))     --> 2.0
print(math.log(math.exp(1))) --> 1.0       省略底：自然对数
print(math.sin(math.pi / 2)) --> 1.0
print(math.fmod(-7, 3))      --> -1        fmod 的余数符号与被除数相同（对比 -7 % 3 == 2）
```

### 类型相关

```lua
print(math.type(1), math.type(1.0), math.type("1"))   --> integer   float   nil
print(math.tointeger(3.0))                            --> 3
print(math.tointeger(3.5))                            --> nil
```

### 随机数

```lua
print(math.random())         -- [0, 1) 的随机浮点数
print(math.random(6))        -- 1~6 的随机整数（模拟骰子）
print(math.random(10, 20))   -- 10~20 的随机整数

math.randomseed(42)          -- 设置种子：相同种子得到相同序列（便于复现）
```

> Lua 5.4 启动时会自动用随机种子；Lua 5.3 及更早版本需要手动 `math.randomseed(os.time())`，否则每次运行结果相同。

洗牌算法（Fisher–Yates）：

```lua
local function shuffle(t)
    for i = #t, 2, -1 do
        local j = math.random(i)
        t[i], t[j] = t[j], t[i]
    end
    return t
end

local cards = {1, 2, 3, 4, 5}
print(table.concat(shuffle(cards), ","))   --> 例如 3,5,1,4,2（每次不同）
```

## 10.3 os 库

### 时间与日期

```lua
print(os.time())                      --> 当前时间戳（秒），例如 1795180665

print(os.date("%Y-%m-%d"))            --> 例如 2026-11-21
print(os.date("%H:%M:%S"))            --> 例如 14:30:05
print(os.date("%Y-%m-%d %H:%M:%S"))   --> 例如 2026-11-21 14:30:05

-- os.date("*t") 返回表
local now = os.date("*t")
print(now.year, now.month, now.day, now.hour, now.min, now.sec)
print(now.wday, now.yday)             -- 星期几（周日=1）、一年中的第几天
```

常用格式符：`%Y` 四位年、`%m` 月、`%d` 日、`%H` 时（24 小时）、`%M` 分、`%S` 秒、`%A` 星期全称、`%B` 月份全称。

### 由日期构造时间戳、计算时间差

```lua
local t1 = os.time{year = 2026, month = 1, day = 1, hour = 0}
local t2 = os.time{year = 2026, month = 1, day = 2, hour = 0}
print(os.difftime(t2, t1))            --> 86400.0   （秒）
print((t2 - t1) // 3600)              --> 24        （小时）
```

### 计时：os.clock()

`os.clock()` 返回程序已使用的 CPU 时间（秒），适合做性能测试：

```lua
local start = os.clock()
local s = 0
for i = 1, 1000000 do s = s + i end
print(string.format("耗时 %.4f 秒", os.clock() - start))
```

### 其他常用函数

```lua
print(os.getenv("HOME"))         -- 读取环境变量（不存在返回 nil；Windows 可试 "USERPROFILE"）
os.rename("old.txt", "new.txt")  -- 重命名文件
os.remove("new.txt")             -- 删除文件（失败返回 nil, 错误信息）
os.exit(0)                       -- 退出程序，参数是退出码（0 表示正常）
```

## 10.4 io 库

### 输出：print 与 io.write

```lua
print("a", "b", 1)         --> a   b   1     多个参数用制表符分隔，自动换行，自动 tostring
io.write("a", "b", 1)      --> ab1           不加分隔符，不换行；只接受字符串和数字
io.write("\n")
```

`io.write` 更适合精确控制输出格式：

```lua
for i = 1, 5 do io.write(i, i < 5 and ", " or "\n") end   --> 1, 2, 3, 4, 5
```

### 输入：io.read

```lua
io.write("请输入姓名: ")
local name = io.read()          -- 读一行（不含换行符），等同于 io.read("l")
io.write("请输入年龄: ")
local age = io.read("n")        -- 读一个数字
print(name, age)
```

`io.read` 的格式：

| 格式 | 含义 |
|------|------|
| `"l"` | 读一行，去掉换行符（默认） |
| `"L"` | 读一行，保留换行符 |
| `"n"` | 读一个数字 |
| `"a"` | 读取全部剩余内容 |
| 数字 `k` | 读取 k 个字节 |

### 读写文件（第 16 章会深入讲）

```lua
-- 写文件
local f, err = io.open("test.txt", "w")
if not f then
    print("打开失败: " .. err)
else
    f:write("line1\n", "line2\n", 123, "\n")
    f:close()                              -- 用完记得关闭
end

-- 逐行读
for line in io.lines("test.txt") do
    print(line)
end
--> line1 / line2 / 123

-- 一次读完
local f2 = io.open("test.txt", "r")
local content = f2:read("a")
f2:close()
print(#content)                            --> 16

-- 打开失败：返回 nil、错误信息、错误码
print(io.open("not_exist.txt"))
--> nil   not_exist.txt: No such file or directory   2
```

打开模式：

| 模式 | 含义 |
|------|------|
| `"r"` | 读（默认），文件须存在 |
| `"w"` | 写，**清空**原内容，不存在则创建 |
| `"a"` | 追加到末尾 |
| `"r+"` | 读写，文件须存在 |
| `"w+"` | 读写，清空原内容 |
| `"a+"` | 读写，写入追加到末尾 |
| 加 `b` | 二进制模式，如 `"rb"`、`"wb"` |

标准流：`io.stdout`（标准输出）、`io.stderr`（标准错误）、`io.stdin`（标准输入）：

```lua
io.stderr:write("这是错误信息\n")
io.stdout:write("这是普通输出\n")
```

## 10.5 综合实战：词频统计

综合运用 table、string、math 的小例子：

```lua
local text = "the quick brown fox jumps over the lazy dog the end"

-- 1. 统计词频
local freq = {}
for w in text:gmatch("%a+") do
    local lw = w:lower()
    freq[lw] = (freq[lw] or 0) + 1
end

-- 2. 把单词取出来放进数组
local words = {}
for w in pairs(freq) do words[#words + 1] = w end

-- 3. 按频率降序排，频率相同按字母升序
table.sort(words, function(a, b)
    if freq[a] ~= freq[b] then return freq[a] > freq[b] end
    return a < b
end)

-- 4. 输出前 3 名
for i = 1, 3 do
    print(string.format("%-6s %d", words[i], freq[words[i]]))
end
--> the    3
--> brown  1
--> dog    1
```

## 10.6 小结与练习

**小结：**

- 基础函数常用：`type` `tostring` `tonumber` `assert` `load` `pcall`
- `math`：取整 `floor/ceil`、极值 `max/min`、随机数 `random`、常量 `pi/huge`
- `os`：时间 `time/date/clock/difftime`、`getenv`、`exit`
- `io`：`write/read/open/lines`；用完文件要 `close`
- Lua 没有内置 `round`、`split`、`trim`，需要自己写（前面都给了实现）

**练习 1：** 写一个程序，把 1~5 写入文件 `nums.txt`（每行一个），再读回来求和。

<details>
<summary>参考答案</summary>

```lua
local f = assert(io.open("nums.txt", "w"))
for i = 1, 5 do
    f:write(i, "\n")
end
f:close()

local sum = 0
for line in io.lines("nums.txt") do
    sum = sum + tonumber(line)
end
print(sum)       --> 15
os.remove("nums.txt")
```

</details>

**练习 2：** 模拟掷两个骰子 1000 次，统计各点数之和（2~12）出现的次数。

<details>
<summary>参考答案</summary>

```lua
local count = {}
for s = 2, 12 do count[s] = 0 end

for _ = 1, 1000 do
    local s = math.random(6) + math.random(6)
    count[s] = count[s] + 1
end

for s = 2, 12 do
    print(string.format("%2d: %s", s, string.rep("*", count[s] // 10)))
end
```

7 的出现次数通常最多。

</details>

---

# 本部分速查表

**Table：** 下标从 1 开始；`ipairs` 数组、`pairs` 全部；`#` 遇到空洞不可靠；引用类型

**table 库：** `insert` `remove` `concat` `sort` `unpack` `pack` `move`

**字符串：** 不可变；`sub(i,j)` 支持负数；`find` 位置、`match` 提取、`gmatch` 遍历、`gsub` 替换

**模式：** `%a %d %s %w %p`（大写取反）；`* + - ?`；`^ $`；`( )` 捕获；`%b()`；用 `%` 转义特殊字符

**格式化：** `string.format("%d %5.2f %s %05d %x %%", ...)`

**中文：** `#` 是字节数；用 `utf8.len`、`utf8.codes`、`utf8.charpattern`

**math：** `floor ceil abs max min sqrt log random randomseed pi huge tointeger type`

**os：** `time date clock difftime getenv exit remove rename`

**io：** `write read open lines close`；模式 `r w a r+ w+ a+`

---

*下一节预告：第四部分《进阶特性》——元表与元方法、面向对象编程、模块与包、错误处理、协程。*
