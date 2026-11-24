
> 本文基于 **Lua 5.4**。示例中的 `--> 结果` 表示 `print` 的输出。
> 如何运行：保存为 `demo.lua`，命令行执行 `lua demo.lua`；或直接在 `lua` 交互环境（REPL）里逐行输入。
# 第 2 节　基本语法与注释
## 2.1 语句与分号
Lua 的语句以换行分隔，**分号可有可无**：
```lua
print("Hello")
print("World");      -- 加分号也可以
local a = 1; local b = 2   -- 同一行写多条语句，可用分号（或空格）分隔
```

> ⚠️ **小坑**：一行以 `(` 开头时，可能被当作上一行的函数调用参数。如果你写的风格不用分号，遇到这种情况在前面补一个 `;` 即可：

```lua
local s = "abc"
;(print or io.write)(s)   -- 行首加分号避免歧义
```
## 2.2 注释
### 单行注释：`--`
```lua
-- 这是一行注释
print("hi")  -- 行尾也可以写注释
```
### 多行注释：`--[[ ... ]]`
```lua
--[[
  这是多行注释
  可以写很多行
]]
print("after comment")
```
### 带等号的多行注释：`--[=[ ... ]=]`
当注释内容本身含有 `]]` 时，用等号数量一致的长括号包裹：

```
--[==[
  这里面可以出现 ]] 而不会提前结束
  local t = a[b[1]]
]==]
```
### 实用技巧：一个字符切换代码块
```lua
---[[
print("这段代码生效")      -- 开头是 ---[[ ，其实是单行注释，下面的代码正常运行
--]]

--[[
print("这段代码被注释掉")  -- 开头是 --[[ ，整块被注释
--]]
```

只要把开头的 `--[[` 改成 `---[[`（多加一个 `-`），就能快速启用 / 禁用一整段代码，调试时很方便。
## 2.3 标识符（命名规则）
标识符用来给变量、函数、表字段等命名。
**规则：**
- 由 **字母、数字、下划线** 组成，**不能以数字开头**
- **区分大小写**：`name` 和 `Name` 是两个不同的变量
- 不能使用**关键字**

```lua
local name = "Tom"      -- ✅
local _tmp = 1          -- ✅
local player2 = "B"     -- ✅
-- local 2player = 1    -- ❌ 不能以数字开头
-- local my-name = 1    -- ❌ 不能含 -
-- local end = 1        -- ❌ end 是关键字

local Name = "Jerry"
print(name, Name)       --> Tom   Jerry   （大小写不同，是不同变量）
```

**约定俗成的写法**（非强制）：

| 写法                         | 含义                         |
| -------------------------- | -------------------------- |
| `_`                        | 占位的“哑变量”，表示不关心这个值          |
| `_VERSION`、`_G`            | 下划线+大写，Lua 内部保留的特殊变量，自己别乱用 |
| `UPPER_CASE`               | 常量                         |
| `snake_case` / `camelCase` | 普通变量、函数，团队统一即可             |

```lua
print(_VERSION)         --> Lua 5.4
for _, v in ipairs({10, 20, 30}) do   -- 不关心下标，用 _ 占位
    print(v)
end
```
## 2.4 关键字
Lua 5.4 共有 22 个保留字，**不能用作标识符**：
```
and       break     do        else      elseif    end
false     for       function  goto      if        in
local     nil       not       or        repeat    return
then      true      until     while
```
## 2.5 全局变量与局部变量
### 全局变量：默认就是全局
Lua 中，**不加 `local` 声明的变量就是全局变量**，无需事先声明：
```lua
count = 10              -- 全局变量
print(count)            --> 10
print(undefined_var)    --> nil   （访问不存在的变量不会报错，得到 nil）
```

全局变量实际存放在一个叫 `_G` 的表里：

```lua
count = 10
print(_G.count)         --> 10
print(_G["count"])      --> 10
```

删除全局变量：赋值为 `nil` 即可。

```lua
count = nil
print(count)            --> nil
```
### 局部变量：用 `local` 声明
```lua
local x = 100
print(x)                --> 100
```

局部变量的作用域是**从声明处到所在代码块结束**。代码块包括：整个文件（chunk）、`do...end`、函数体、`if`/`for`/`while` 等的内部。

```lua
local a = 1

do
    local a = 2          -- 内层的新变量，遮蔽（shadow）了外层的 a
    local b = 3
    print(a, b)          --> 2   3
end

print(a)            --> 1   （外层 a 没变）
print(b)            --> nil （b 在 do...end 外不可见，访问的是全局 b，不存在）
```
### 为什么推荐优先使用 `local`？
1. **更快**：局部变量访问比全局变量快（全局要查表）
2. **更安全**：避免不同位置的同名变量互相污染
3. **更易维护**：作用域清晰

看一个“全局变量污染”的典型 bug：
```lua
function sum(n)
    total = 0            -- 忘了 local，total 变成全局变量！
    for i = 1, n do
        total = total + i
    end
    return total
end

total = 999                   -- 你原本想用的全局变量
print(sum(10))                --> 55
print(total)                  --> 55   （被函数悄悄改掉了！）
```
修复：`local total = 0`。
### Lua 5.4 新增：`<const>` 和 `<close>`
```lua
local PI <const> = 3.14159    -- 常量，之后不能再赋值
-- PI = 3    -- ❌ 编译时报错：attempt to assign to const variable 'PI'
print(PI)                     --> 3.14159
```
（`<close>` 用于资源自动释放，属于进阶内容，以后讲。）
## 2.6 赋值：多重赋值与交换
Lua 支持**多重赋值**：先计算右边所有表达式，再统一赋值。
```lua
local a, b = 1, 2
print(a, b)              --> 1   2

-- 交换两个变量，不需要临时变量
a, b = b, a
print(a, b)              --> 2   1
```

左右数量不一致时的规则：

```lua
local x, y, z = 1, 2
print(x, y, z)           --> 1   2   nil    （变量多：多出的补 nil）

local m, n = 1, 2, 3
print(m, n)              --> 1   2          （值多：多出的丢弃）
```
## 2.7 小结与练习
**小结：**
- 语句分号可省略；注释 `--` 和 `--[[ ]]`
- 标识符区分大小写，不能是关键字
- 默认全局，用 `local` 限定作用域，**优先写 `local`**
- 多重赋值可以优雅地交换变量

**练习：** 下面代码输出什么？
```lua
local a = 10
do
    local a = 20
    a = a + 1
    print(a)
end
print(a)
```

<details>
<summary>答案</summary>

```
21
10
```

内层的 `local a` 是新变量，修改它不影响外层的 `a`。

</details>

---
# 第 3 节　数据类型
## 3.1 总览
Lua 是**动态类型**语言：变量本身没有类型，**值**才有类型。同一个变量可以先后存放不同类型的值。
Lua 有 **8 种基本类型**：

| 类型         | 含义             | 示例                    |
| ---------- | -------------- | --------------------- |
| `nil`      | 空值，表示“没有值”     | `nil`                 |
| `boolean`  | 布尔值            | `true` / `false`      |
| `number`   | 数字（整数 + 浮点数）   | `42`、`3.14`           |
| `string`   | 字符串            | `"hello"`             |
| `table`    | 表（Lua 唯一的数据结构） | `{1, 2, 3}`           |
| `function` | 函数             | `function() end`      |
| `userdata` | 用户数据（C 语言数据）   | 如文件句柄                 |
| `thread`   | 线程（协程）         | `coroutine.create(f)` |

用 `type()` 函数查看一个值的类型，**返回的是字符串**：
```lua
print(type(nil))                 --> nil
print(type(true))                --> boolean
print(type(42))                  --> number
print(type("hi"))                --> string
print(type({}))                  --> table
print(type(print))               --> function
print(type(io.stdout))           --> userdata
print(type(coroutine.create(print)))  --> thread

-- type 的返回值本身是 string
print(type(type(42)))            --> string

-- 变量没有类型，值才有
local v = 10
print(type(v))                   --> number
v = "ten"
print(type(v))                   --> string
```
## 3.2 nil（空值）
`nil` 表示“**没有值**”，类型是 `nil` 的唯一值就是 `nil`。

```lua
local a
print(a)                 --> nil   （声明但未赋值，默认就是 nil）
print(undefined_name)    --> nil   （访问不存在的全局变量）

local t = {name = "Tom"}
print(t.age)             --> nil   （访问表里不存在的字段）
```

**两个关键作用：**
1. 作为**条件判断**时等同于 `false`
2. 给表字段或全局变量赋 `nil` 等于**删除**它

```lua
local t = {a = 1, b = 2}
t.a = nil                -- 删除字段 a
print(t.a)               --> nil
```

比较时用 `== nil`：

```lua
local x
if x == nil then
    print("x 还没有值")  --> x 还没有值
end
```
## 3.3 boolean（布尔）
只有两个值：`true` 和 `false`。
### ⭐ 重点：Lua 的“真假”规则
> **只有 `nil` 和 `false` 是假，其他一切都是真。**

**特别注意：数字 `0` 和空字符串 `""` 在 Lua 里是真！**（与 C、Python、JavaScript 不同）

```lua
if 0 then print("0 是真") end            --> 0 是真
if "" then print("空字符串是真") end     --> 空字符串是真
if nil then print("不会输出") end
if false then print("不会输出") end
if {} then print("空表也是真") end       --> 空表也是真
```

快速自测：

```lua
print(not nil)      --> true
print(not false)    --> true
print(not 0)        --> false   （0 是真，取反为假）
print(not "")       --> false
```
## 3.4 number（数字）
Lua 5.3 起，number 有两种**子类型**：
- **integer（整数）**：64 位有符号整数
- **float（浮点数）**：双精度浮点数

用 `math.type()` 区分：

```lua
print(math.type(10))     --> integer
print(math.type(10.0))   --> float
print(math.type(3.14))   --> float
print(math.type("10"))   --> nil（不是数字则返回 nil；非 number 参数得到 nil）
print(type(10), type(10.0))  --> number   number   （type 看来都是 number）
```
### 数字的各种写法
```lua
print(100)         --> 100
print(3.14)        --> 3.14
print(1e3)         --> 1000.0     （科学计数法，结果是浮点数）
print(2.5e-2)      --> 0.025
print(0xff)        --> 255        （十六进制）
print(0x10)        --> 16
print(0xA.8p0)     --> 10.5       （十六进制浮点，了解即可）
```
### 整数与浮点数的运算规则
```lua
print(1 + 2)       --> 3        整数 + 整数 = 整数
print(1 + 2.0)     --> 3.0      只要有浮点，结果就是浮点
print(10 / 2)      --> 5.0      ⭐ 除法 / 永远得到浮点数
print(10 // 3)     --> 3        整除，整数
print(10 // 3.0)   --> 3.0
print(2 ^ 3)       --> 8.0      ⭐ 乘方 ^ 永远得到浮点数
```

整数和浮点数的**值相等就相等**：

```lua
print(1 == 1.0)    --> true
print(math.type(1), math.type(1.0))   --> integer   float
```
### 整数溢出：会“回绕”
```lua
print(math.maxinteger)        --> 9223372036854775807
print(math.mininteger)        --> -9223372036854775808
print(math.maxinteger + 1 == math.mininteger)   --> true（溢出回绕，不报错）
```
### 浮点数精度问题（所有语言通病）
```lua
print(0.1 + 0.2 == 0.3)       --> false
print(0.1 + 0.2)              --> 0.3        （print 默认只显示 14 位有效数字）
print(string.format("%.17f", 0.1 + 0.2))   --> 0.30000000000000004

-- 正确的比较方式：看差值是否足够小
local function almost_equal(a, b)
    return math.abs(a - b) < 1e-9
end
print(almost_equal(0.1 + 0.2, 0.3))   --> true
```
### 数字与字符串转换
```lua
-- 字符串 → 数字
print(tonumber("42"))         --> 42
print(tonumber("3.14"))       --> 3.14
print(tonumber("  10  "))     --> 10      （首尾空白允许）
print(tonumber("0xff"))       --> 255
print(tonumber("abc"))        --> nil     （转换失败返回 nil，不报错）
print(tonumber("12abc"))      --> nil
print(tonumber("ff", 16))     --> 255     （指定进制）
print(tonumber("1010", 2))    --> 10      （二进制）

-- 数字 → 字符串
print(tostring(42))           --> 42
print(tostring(3.0))          --> 3.0
print(type(tostring(42)))     --> string

-- 浮点转整数
print(math.floor(3.7))        --> 3
print(math.ceil(3.2))         --> 4
print(math.tointeger(3.0))    --> 3
print(math.tointeger(3.5))    --> nil（不能无损转换则返回 nil；在 5.4 中也可能返回 fail/nil）
```
## 3.5 string（字符串）
字符串是**不可变**的字节序列。想“修改”字符串，其实是生成了一个新字符串。
### 三种书写方式
```lua
local s1 = "双引号字符串"
local s2 = '单引号字符串'               -- 与双引号完全等价
local s3 = [[长括号字符串
可以直接跨行，
里面的 "引号" 和 '引号' 都不需要转义]]

print(s1)
print(s2)
print(s3)
```

需要在字符串中写引号时，可以换用另一种引号，或用转义：

```lua
print("He said \"hi\"")       --> He said "hi"
print('It\'s ok')             --> It's ok
print("It's ok")              --> It's ok
```
### 常用转义字符

| 转义 | 含义 |
|------|------|
| `\n` | 换行 |
| `\t` | 制表符 |
| `\\` | 反斜杠 |
| `\"` `\'` | 引号 |
| `\r` | 回车 |
| `\0` | 空字符 |
| `\65` | 十进制 ASCII 码（65 = `A`） |
| `\x41` | 十六进制（`A`） |
| `\u{4F60}` | Unicode 码点（`你`） |
| `\z` | 跳过其后所有空白（含换行），用于长字符串折行 |

```lua
print("第一行\n第二行")
print("a\tb\tc")                --> a   b   c
print("\65\66\67")               --> ABC
print("\x48\x69")                --> Hi
print("\u{4F60}\u{597D}")        --> 你好

-- 长括号字符串里转义不生效，原样输出
print([[a\nb]])                  --> a\nb

-- 长括号字符串的第一个换行会被忽略
local text = [[
第一行
第二行]]
print(text)
```
### 长度：`#`（字节数！）
```lua
print(#"hello")        --> 5
print(#"")             --> 0
print(#"你好")         --> 6       ⭐ 中文在 UTF-8 里一个字占 3 个字节
print(utf8.len("你好"))  --> 2     想按“字符数”计算，用 utf8.len
```
### 字符串与数字的自动转换（coercion）
算术运算时，字符串会**自动转成数字**；连接 `..` 时，数字会**自动转成字符串**：
```lua
print("10" + 5)        --> 15
print("3.5" * 2)       --> 7.0
print("0x10" + 0)      --> 16
print(10 .. 20)        --> 1020    （数字连接，结果是字符串）
print("Age: " .. 18)   --> Age: 18
-- print("abc" + 1)    -- ❌ 报错：attempt to perform arithmetic on a string value
```

> 💡 **建议**：自动转换虽方便，但容易埋 bug。需要转换时，显式使用 `tonumber()` / `tostring()` 更清晰。
### 字符串比较
```lua
print("a" == "a")      --> true
print("abc" < "abd")   --> true     （按字典序逐字符比较）
print("Z" < "a")       --> true     （大写字母 ASCII 码小于小写）
print("10" < "9")      --> true     ⚠️ 字符串比较，不是数值比较！
print(10 < 9)          --> false
```
### 常用字符串函数（先预览，第 9 章详讲）
```lua
local s = "Hello, Lua"
print(string.upper(s))         --> HELLO, LUA
print(s:lower())               --> hello, lua      （s:lower() 是 string.lower(s) 的简写）
print(s:len())                 --> 10
print(s:sub(1, 5))             --> Hello           （截取第 1~5 个字符，下标从 1 开始）
print(s:rep(2, "|"))           --> Hello, Lua|Hello, Lua
print(s:reverse())             --> auL ,olleH
print(("%d + %d = %d"):format(1, 2, 3))   --> 1 + 2 = 3
```
## 3.6 table（表）
table 是 Lua **唯一的复合数据结构**，既能当数组、字典，也能当对象、集合、模块。这里先简单认识，第 8 章详讲。
### 当“数组”用（下标从 1 开始！）
```lua
local fruits = {"apple", "banana", "cherry"}
print(fruits[1])       --> apple        ⭐ 第一个元素下标是 1，不是 0
print(fruits[3])       --> cherry
print(fruits[0])       --> nil
print(#fruits)         --> 3

fruits[#fruits + 1] = "date"     -- 在末尾追加
print(#fruits)         --> 4
```
### 当“字典”用（键值对）
```lua
local person = {
    name = "Tom",
    age  = 20,
}
print(person.name)         --> Tom          （点语法，等价于 person["name"]）
print(person["age"])       --> 20
person.city = "Taipei"     -- 添加新字段
person.age = nil           -- 删除字段
```
### 混合使用 & 嵌套
```lua
local student = {
    name = "Lucy",
    scores = {90, 85, 77},          -- 嵌套另一个表
    ["favorite color"] = "blue",    -- 键含空格，必须用 [""] 写法
}
print(student.scores[2])            --> 85
print(student["favorite color"])    --> blue
```
### 重要特性：table 是“引用类型”
变量里存的是表的“引用（地址）”，赋值不会复制整个表：

```lua
local a = {1, 2, 3}
local b = a            -- b 和 a 指向同一个表
b[1] = 100
print(a[1])            --> 100     （a 也变了）

print({} == {})        --> false   （两个不同的表，即使内容相同也不相等）
print(a == b)          --> true    （指向同一个表）
```
## 3.7 function（函数）
函数在 Lua 里是**一等公民（first-class value）**：可以赋值给变量、作为参数传递、作为返回值，也可以存进表里。
```lua
-- 常规定义
local function add(a, b)
    return a + b
end
print(add(1, 2))             --> 3

-- 等价写法：把匿名函数赋给变量
local mul = function(a, b) return a * b end
print(mul(3, 4))             --> 12

-- 作为参数传递
local function apply(f, x, y)
    return f(x, y)
end
print(apply(add, 10, 20))    --> 30
print(apply(mul, 10, 20))    --> 200

-- 存进表里
local ops = {
    add = add,
    mul = mul,
}
print(ops.add(5, 6))         --> 11

print(type(add))             --> function
```
## 3.8 userdata 与 thread（简单了解）
**userdata**：用来表示由 C/C++ 创建的数据（如文件句柄），Lua 代码里不能直接创建，只能使用。
```lua
local f = io.open("test.txt", "w")   -- 返回的文件对象就是 userdata
print(type(f))                       --> userdata
f:close()
```

**thread**：指**协程（coroutine）**，不是操作系统线程，用于实现协作式多任务（第 15 章详讲）。

```lua
local co = coroutine.create(function()
    print("in coroutine")
end)
print(type(co))                      --> thread
coroutine.resume(co)                 --> in coroutine
```
## 3.9 类型判断的实用写法
```lua
local function describe(v)
    local t = type(v)
    if t == "number" then
        return "数字: " .. v
    elseif t == "string" then
        return "字符串: " .. v
    elseif t == "boolean" then
        return "布尔: " .. tostring(v)   -- 布尔不能直接用 .. 连接，需要 tostring
    elseif t == "nil" then
        return "空值"
    else
        return "其他类型: " .. t
    end
end

print(describe(42))        --> 数字: 42
print(describe("hi"))      --> 字符串: hi
print(describe(true))      --> 布尔: true
print(describe(nil))       --> 空值
print(describe({}))        --> 其他类型: table
```
## 3.10 小结与练习
**小结：**
- 8 种类型：`nil`、`boolean`、`number`、`string`、`table`、`function`、`userdata`、`thread`
- 变量无类型，值有类型；用 `type()` 检查
- **只有 `nil` 和 `false` 为假**，`0`、`""` 都为真
- number 分整数和浮点；`/` 和 `^` 永远返回浮点
- 字符串不可变，`#` 返回字节数；`tonumber` / `tostring` 做显式转换
- table 是引用类型，下标默认从 **1** 开始

**练习：** 下面各行输出什么？

```lua
print(type(nil))
print(type(type))
print(10 / 2)
print(7 // 2)
print("5" + 3)
print(#"Lua语言")
print(0 and "yes" or "no")
```

<details>
<summary>答案</summary>

```
nil
function
5.0
3
8
9          （"Lua" 3 字节 + "语言" 6 字节）
yes        （0 是真）
```

</details>

---
# 第 4 节　运算符
## 4.1 算术运算符

| 运算符 | 含义 | 示例 | 结果 |
|--------|------|------|------|
| `+` | 加 | `5 + 3` | `8` |
| `-` | 减 | `5 - 3` | `2` |
| `*` | 乘 | `5 * 3` | `15` |
| `/` | 除（浮点除法） | `5 / 2` | `2.5` |
| `//` | 整除（向下取整） | `5 // 2` | `2` |
| `%` | 取余 | `5 % 3` | `2` |
| `^` | 乘方 | `2 ^ 3` | `8.0` |
| `-`（一元） | 取负 | `-5` | `-5` |

```lua
print(7 + 2)       --> 9
print(7 - 2)       --> 5
print(7 * 2)       --> 14
print(7 / 2)       --> 3.5
print(8 / 2)       --> 4.0     ⭐ 即使能整除，/ 也返回浮点
print(7 // 2)      --> 3
print(7 % 2)       --> 1
print(2 ^ 10)      --> 1024.0
print(-(3 + 4))    --> -7
```
### 整除 `//` 与取余 `%`：负数要小心
Lua 的整除是**向下取整（向负无穷方向）**，取余的结果**与除数同号**：
```lua
print(7 // 2)      --> 3
print(-7 // 2)     --> -4      ⭐ 不是 -3！向下取整
print(7 // -2)     --> -4

print(7 % 3)       --> 1
print(-7 % 3)      --> 2       ⭐ 结果与除数 3 同号
print(7 % -3)      --> -2
```

> 恒等式：`a == (a // b) * b + a % b`
> 验证：`-7 == (-4) * 3 + 2` ✔
### 浮点数的整除与取余
```lua
print(7.5 // 2)    --> 3.0
print(7.5 % 2)     --> 1.5
print(5.3 % 1)     --> 0.3      （取小数部分，实际有浮点误差）
```
### 除以 0
```lua
print(1 / 0)       --> inf      （正无穷）
print(-1 / 0)      --> -inf
print(0 / 0)       --> -nan 或 nan   （非数字，不同平台显示不同）
print(1 // 0.0)    --> inf
-- print(1 // 0)   -- ❌ 整数整除 0：报错 attempt to perform 'n//0'
-- print(1 % 0)    -- ❌ 整数取余 0：报错 attempt to perform 'n%%0'
```
### 实用例子
```lua
-- 判断奇偶
local n = 7
if n % 2 == 0 then print("偶数") else print("奇数") end   --> 奇数

-- 秒数转时分秒
local total = 3725
local h = total // 3600
local m = total % 3600 // 60
local s = total % 60
print(string.format("%02d:%02d:%02d", h, m, s))   --> 01:02:05

-- 开平方：用 ^ 0.5 或 math.sqrt
print(16 ^ 0.5)        --> 4.0
print(math.sqrt(16))   --> 4.0
```
## 4.2 位运算符（Lua 5.3+）
位运算只对**整数**有效（浮点数若能无损转为整数也行）。

| 运算符 | 含义 | 示例 | 结果 |
|--------|------|------|------|
| `&` | 按位与 | `5 & 3` | `1` |
| `\|` | 按位或 | `5 \| 3` | `7` |
| `~`（二元） | 按位异或 | `5 ~ 3` | `6` |
| `~`（一元） | 按位取反 | `~5` | `-6` |
| `<<` | 左移 | `1 << 4` | `16` |
| `>>` | 右移（逻辑右移，补 0） | `256 >> 4` | `16` |

```lua
print(5 & 3)       --> 1     （0101 & 0011 = 0001）
print(5 | 3)       --> 7     （0101 | 0011 = 0111）
print(5 ~ 3)       --> 6     （0101 ^ 0011 = 0110）
print(~0)          --> -1
print(1 << 4)      --> 16
print(256 >> 4)    --> 16

-- 实用：用位标志表示权限
local READ, WRITE, EXEC = 1, 2, 4      -- 001, 010, 100
local perm = READ | WRITE              -- 011 = 3
print(perm & WRITE ~= 0)               --> true   ⚠️ 看优先级！见下文
```

> ⚠️ 上面最后一行：在 Lua 中 `~=`（不等）的优先级**低于** `&`，所以 `perm & WRITE ~= 0` 恰好等价于 `(perm & WRITE) ~= 0`，是对的。但为了可读性，**建议总是加括号**。
## 4.3 关系运算符
结果永远是 **boolean**。

| 运算符 | 含义 |
|--------|------|
| `==` | 等于 |
| `~=` | **不等于**（注意不是 `!=`） |
| `<` | 小于 |
| `>` | 大于 |
| `<=` | 小于等于 |
| `>=` | 大于等于 |

```lua
print(1 == 1)      --> true
print(1 ~= 2)      --> true
print(3 > 5)       --> false
print(3 <= 3)      --> true
```
### `==` 的比较规则
**① 类型不同，一定不相等**（不会自动转换）：

```lua
print(1 == "1")        --> false   ⭐ 数字和字符串不会自动转换
print(0 == false)      --> false
print(nil == false)    --> false
print(1 == 1.0)        --> true    （同为 number，值相等）
```

**② 字符串按内容比较：**

```lua
print("abc" == "abc")  --> true
```

**③ table、function 等按“引用”比较：**

```lua
local a = {1, 2}
local b = {1, 2}
local c = a
print(a == b)          --> false   （不同对象）
print(a == c)          --> true    （同一个对象）
```
### `<` `>` 的比较规则
- 两个**数字**：按数值比较
- 两个**字符串**：按字典序比较
- 其他情况（如数字与字符串）：**报错**

```lua
print(1 < 2)           --> true
print("a" < "b")       --> true
-- print(1 < "2")      -- ❌ 报错：attempt to compare number with string
```
## 4.4 逻辑运算符：`and` `or` `not`

| 运算符 | 含义 |
|--------|------|
| `and` | 与 |
| `or` | 或 |
| `not` | 非（永远返回 boolean） |

### 基础用法
```lua
print(true and false)    --> false
print(true or false)     --> true
print(not true)          --> false
print(not nil)           --> true
```
### ⭐ 重点：`and` / `or` 返回的是“操作数本身”，并且短路求值
规则（记住这两句即可）：

- **`a and b`**：如果 `a` 为假，返回 `a`；否则返回 `b`
- **`a or b`**：如果 `a` 为真，返回 `a`；否则返回 `b`

```lua
print(1 and 2)           --> 2        （1 为真，返回后者）
print(nil and 2)         --> nil      （nil 为假，直接返回 nil）
print(false and error("不会执行"))   --> false   （短路：右边不会被计算）

print(1 or 2)            --> 1        （1 为真，直接返回 1）
print(nil or 2)          --> 2        （nil 为假，返回后者）
print(false or nil)      --> nil
print(nil or false)      --> false
```
### 常见惯用法
**① 设置默认值：`x = x or default`**

```lua
local function greet(name)
    name = name or "陌生人"       -- name 为 nil/false 时使用默认值
    print("你好, " .. name)
end
greet("Tom")             --> 你好, Tom
greet()                  --> 你好, 陌生人
```

> ⚠️ 注意：如果参数可能合法地传 `false`，`x or default` 会把 `false` 也替换掉：

```lua
local function setFlag(flag)
    flag = flag or true         -- ❌ 传入 false 也会变成 true
    return flag
end
print(setFlag(false))           --> true   （不是想要的结果）

-- 正确写法：`if flag == nil then flag = true end`
```

**② 模拟三元运算符：`cond and a or b`**

```lua
local age = 20
local label = age >= 18 and "成年" or "未成年"
print(label)             --> 成年
```

> ⚠️ **陷阱**：当 `a` 为 `nil` 或 `false` 时，这个写法会失效：

```lua
local r = true and false or "default"
print(r)  --> default   （本来想得到 false）
-- 此时请改用 `if ... then ... else ... end`
```

**③ 保护式取值（避免对 nil 取字段报错）：**

```lua
local user = nil
-- print(user.name)                    -- ❌ 报错：attempt to index a nil value
print(user and user.name)              --> nil   （user 为 nil，短路返回 nil）

local user2 = {name = "Tom"}
print(user2 and user2.name)            --> Tom
```
### `not` 总是返回布尔值
```lua
print(not 0)             --> false
print(not "hello")       --> false
print(not not nil)       --> false   （双重否定，常用来转成布尔）
print(not not "x")       --> true
```
## 4.5 字符串连接运算符：`..`
用于拼接字符串，**数字会自动转为字符串**。
```lua
print("Hello" .. ", " .. "Lua")     --> Hello, Lua
print("Age: " .. 18)                --> Age: 18
print("Pi: " .. 3.14)               --> Pi: 3.14
```
### 要点
**① 数字与 `..` 之间要加空格**，否则会被当成小数点而报错：

```lua
print(1 .. 2)          --> 12
-- print(1..2)         -- ❌ 语法错误：malformed number near '1..2'
```

**② 只有字符串和数字能直接连接，其他类型需先转换：**

```lua
-- print("flag: " .. true)           -- ❌ 报错：attempt to concatenate a boolean value
print("flag: " .. tostring(true))    --> flag: true
-- print("val: " .. nil)             -- ❌ 报错
print("val: " .. tostring(nil))      --> val: nil
```

**③ 连接是右结合的：** `a .. b .. c` 等价于 `a .. (b .. c)`

**④ 性能提示：循环中大量拼接，请用 `table.concat`**

字符串不可变，每次 `..` 都会创建新字符串。拼接次数多时效率低：

```lua
-- ❌ 低效：每次循环都创建新字符串
local s = ""
for i = 1, 5 do
    s = s .. i .. ","
end
print(s)                             --> 1,2,3,4,5,

-- ✅ 高效：先放进表里，最后一次性连接
local parts = {}
for i = 1, 5 do
    parts[#parts + 1] = i
end
print(table.concat(parts, ","))      --> 1,2,3,4,5
```
## 4.6 长度运算符：`#`
```lua
-- 字符串：返回字节数
print(#"hello")            --> 5
print(#"你好")              --> 6

-- 表（序列）：返回元素个数
local arr = {10, 20, 30, 40}
print(#arr)                --> 4

-- 常见用法：在末尾追加元素
arr[#arr + 1] = 50
print(#arr)                --> 5

-- 取最后一个元素
print(arr[#arr])           --> 50
```

> ⚠️ **含有“空洞”（nil）的表，`#` 的结果不可靠：**

```lua
local t = {1, 2, nil, 4}
print(#t)      --> 可能是 4，也可能是 2，行为不确定！
-- 想保证 `#` 可靠，请让数组保持**连续**（中间不要出现 nil）
```
## 4.7 运算符优先级
**从高到低：**

| 优先级 | 运算符 | 结合性 |
|:------:|--------|:------:|
| 1（最高） | `^` | 右结合 |
| 2 | `not`  `#`  `-`（一元）  `~`（一元） | 右结合 |
| 3 | `*`  `/`  `//`  `%` | 左结合 |
| 4 | `+`  `-` | 左结合 |
| 5 | `..` | **右结合** |
| 6 | `<<`  `>>` | 左结合 |
| 7 | `&` | 左结合 |
| 8 | `~`（二元，异或） | 左结合 |
| 9 | `\|` | 左结合 |
| 10 | `<` `>` `<=` `>=` `~=` `==` | 左结合 |
| 11 | `and` | 左结合 |
| 12（最低） | `or` | 左结合 |

### 例子
```lua
print(2 + 3 * 4)            --> 14      先乘后加
print((2 + 3) * 4)          --> 20      括号优先

print(-2 ^ 2)               --> -4.0    ⭐ ^ 比一元负号优先级高：-(2^2)
print((-2) ^ 2)             --> 4.0

print(2 ^ 3 ^ 2)            --> 512.0   ⭐ ^ 右结合：2^(3^2) = 2^9

print(1 .. 2 .. 3)          --> 123     .. 右结合，结果相同
print("a" .. "b" == "ab")   --> true    .. 优先于 ==

print(not 1 == 2)           --> false   ⭐ 等价于 (not 1) == 2 → false == 2 → false
print(not (1 == 2))         --> true    想要“1 不等于 2”，加括号（或直接写 1 ~= 2）

print(1 < 2 and 2 < 3)      --> true    比较优先于 and
print(false and true or true)  --> true  and 优先于 or：(false and true) or true

print(#"abc" + 1)           --> 4       # 优先于 +
print(#"abc" .. "!")        --> 3!      # 优先于 ..
```

> 💡 **实战建议**：记不清优先级时，**直接加括号**。可读性比“炫技”重要得多。
## 4.8 综合练习
**1.** 下面各行输出什么？
```lua
print(7 // 2 * 2 + 7 % 2)
print(not nil and 1 or 2)
print(2 ^ 2 == 4)
print(10 .. 20 == "1020")
print(nil or false)
print(false or nil)
print(-3 % 5)
```

<details>
<summary>答案与解析</summary>

```
7        7//2=3 → 3*2=6 → 7%2=1 → 6+1=7
1        not nil 为 true → true and 1 → 1 → 1 or 2 → 1
true     2^2=4.0，4.0 == 4 为 true（数值相等）
true     .. 优先于 ==，"1020" == "1020"
false    nil 为假，返回后者 false
nil      false 为假，返回后者 nil
2        -3 % 5：结果与除数同号，(-3) = (-1)*5 + 2
```

</details>

**2.** 写一个函数 `max3(a, b, c)`，用 `and` / `or` 求三个数的最大值（不使用 `math.max`）。

<details>
<summary>参考答案</summary>

```lua
local function max3(a, b, c)
    local m = a > b and a or b      -- a、b 为数字，a 不会是 false/nil，三元写法安全
    return m > c and m or c
end

print(max3(3, 9, 5))     --> 9
print(max3(10, 2, 7))    --> 10
```

</details>

**3.** 给定秒数 `s = 100000`，用 `//` 和 `%` 求出是几天几小时几分几秒。

<details>
<summary>参考答案</summary>

```lua
local s = 100000
local d = s // 86400
local h = s % 86400 // 3600
local m = s % 3600 // 60
local sec = s % 60
print(d .. "天" .. h .. "小时" .. m .. "分" .. sec .. "秒")
--> 1天3小时46分40秒
```

</details>

---

# 本部分速查表

**注释：** `-- 单行`　`--[[ 多行 ]]`

**变量：** 默认全局；`local` 声明局部；`local X <const> = 1` 常量

**真假：** 只有 `nil`、`false` 为假

**类型：** `nil` `boolean` `number` `string` `table` `function` `userdata` `thread`，用 `type()` 查看

**数字：** `/` 和 `^` 返回浮点；`//` 向下取整；`%` 结果与除数同号

**比较：** 不等于是 `~=`；不同类型 `==` 恒为 `false`

**逻辑：** `a and b`、`a or b` 返回操作数本身；`x = x or default`；`cond and a or b` 要注意 `a` 为假的陷阱

**连接：** `..`；数字前后加空格；大量拼接用 `table.concat`

**长度：** `#`；字符串是字节数；表中避免空洞

**优先级口诀：** `^` > 一元运算 > 乘除 > 加减 > `..` > 位运算 > 比较 > `and` > `or`；不确定就加括号

---

*下一节预告：第二部分《流程控制与函数》——`if`、循环、函数定义、多返回值与可变参数。*
