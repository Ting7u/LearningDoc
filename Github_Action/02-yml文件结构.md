# 第 2 节：yml 文件结构
> 本节目标：看懂一个 workflow 文件的"骨架"。
> 上一节讲了"有哪些角色"，这一节讲"这些角色在文件里怎么写、写在哪一层"。
> 只认识关键字的位置和作用，不深入每个关键字的用法（后面各节会展开）。

---
## 1. 先补一点 YAML 基础
workflow 是用 YAML 写的。你之前遇到的报错都是缩进问题，所以先把规则讲清楚。YAML 只有 **3 种基本结构**：
### ① 键值对（key: value）
```yaml
name: Hello World
runs-on: ubuntu-latest
```

> 冒号后面**必须有空格**。`name:Hello` ❌，`name: Hello` ✅
### ② 嵌套（靠缩进表示"属于谁"）
```yaml
jobs:                    # jobs 是父
  say-hello:             # say-hello 属于 jobs
    runs-on: ubuntu-latest   # runs-on 属于 say-hello
```

```
jobs
 └── say-hello
      └── runs-on: ubuntu-latest
```

> **缩进 = 层级**。缩进多一级，就是上一行的"子项"。
### ③ 列表（每项以 `- ` 开头
```yaml
steps:
  - name: 第一步
    run: echo 1
  - name: 第二步
    run: echo 2
```

列表里每个 `- ` 是一个独立的"条目"，一个条目内部可以有多个键。

```
steps（列表）
 ├── 条目1: { name: 第一步, run: echo 1 }
 └── 条目2: { name: 第二步, run: echo 2 }
```
### 缩进的黄金规则
```
  - name: 第一步
    run: echo 1
    ↑
    run 对齐 name，而不是对齐 "-"
```

`- ` 占 2 个字符，所以同一条目里的其他键，要和 `name` 对齐（比 `-` 多缩进 2 格）。
### 其他常用语法
```yaml
# 这是注释

run: echo "单行命令"

run: |                  # 竖线 = 多行文本，每行原样保留
  echo "第一行"
  echo "第二行"
```

**只用空格，不用 Tab；统一每级 2 个空格。**

---
## 2. workflow 文件的三层结构
整个文件分成三层，每层有自己的一套关键字：

```
┌────────────────────────────────────────────────┐
│ 第 1 层：Workflow 级（文件顶层）                  │
│   name / on / env / permissions / jobs ...     │
│                                                │
│   ┌──────────────────────────────────────────┐ │
│   │ 第 2 层：Job 级（jobs 下面的每一项）        │ │
│   │   runs-on / needs / if / steps ...       │ │
│   │                                          │ │
│   │   ┌────────────────────────────────────┐ │ │
│   │   │ 第 3 层：Step 级（steps 下的每一项） │ │ │
│   │   │   name / uses / run / with ...     │ │ │
│   │   └────────────────────────────────────┘ │ │
│   └──────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
```

**学习 yml 的关键：看到一个关键字，先问"它属于哪一层？"**

---
## 3. 最小骨架
任何 workflow 都离不开这些：

```yaml
name: 流程名字            # 第1层：可省略

on: push                  # 第1层：必须有

jobs:                     # 第1层：必须有
  job-id:                 # 第2层：一个Job，名字自己起
    runs-on: ubuntu-latest    # 第2层：必须有
    steps:                    # 第2层：必须有
      - run: echo "hi"        # 第3层：一个Step
```

必须有的只有四样：**`on`、`jobs`、`runs-on`、`steps`**。

---
## 4. 第 1 层：Workflow 级关键字
写在文件最顶层（不缩进）。

| 关键字 | 作用 | 是否必须 | 学习阶段 |
|---|---|---|---|
| `name` | 工作流名字 | 否 | 入门 |
| `on` | 触发事件 | **是** | 第 3 节 |
| `jobs` | 任务集合 | **是** | 第 4 节 |
| `env` | 全局环境变量 | 否 | 第 6 节 |
| `permissions` | 权限控制 | 否 | 进阶 |
| `concurrency` | 并发控制 | 否 | 进阶 |
| `run-name` | 每次运行的显示名 | 否 | 可忽略 |
| `defaults` | 默认设置 | 否 | 可忽略 |

**现阶段只需要关注加粗的 `on` 和 `jobs`。**

---
## 5. 第 2 层：Job 级关键字
写在 `jobs.<job-id>` 下面。

```yaml
jobs:
  build:                  # ← job-id（自己起名）
    runs-on: ...          # ← 以下都是 Job 级关键字
    needs: ...
    if: ...
    env: ...
    steps: ...
```

| 关键字               | 作用             | 是否必须  | 学习阶段  |
| ----------------- | -------------- | ----- | ----- |
| `runs-on`         | 用哪种机器          | **是** | 第 4 节 |
| `steps`           | 步骤列表           | **是** | 第 4 节 |
| `needs`           | 依赖哪个 Job（控制先后） | 否     | 第 4 节 |
| `if`              | 满足条件才运行        | 否     | 第 4 节 |
| `env`             | 本 Job 的环境变量    | 否     | 第 6 节 |
| `outputs`         | 向其他 Job 传出数据   | 否     | 第 6 节 |
| `strategy`        | 矩阵构建           | 否     | 第 7 节 |
| `timeout-minutes` | 超时时间           | 否     | 第 7 节 |
| `environment`     | 部署环境           | 否     | 第 8 节 |

---
## 6. 第 3 层：Step 级关键字
写在 `steps` 列表的每个条目里。

```yaml
steps:
  - name: 拉取代码           # ← 以下都是 Step 级关键字
    uses: actions/checkout@v4

  - name: 打印
    run: echo "hi"
```

| 关键字    | 作用           | 说明               |
| ------ | ------------ | ---------------- |
| `name` | 步骤名字         | 显示在日志里，建议写       |
| `run`  | 执行 shell 命令  | 与 `uses` **二选一** |
| `uses` | 调用现成 Action  | 与 `run` **二选一**  |
| `with` | 给 Action 传参数 | 配合 `uses` 使用     |
| `env`  | 本步骤的环境变量     | 第 6 节            |
| `if`   | 满足条件才执行      | 第 4 节            |
| `id`   | 给步骤起编号       | 取输出时用            |

> 一个 Step 里，`run` 和 `uses` 只能出现一个。

---
## 7. 同名关键字：`env` 和 `if` 出现在多层
有些关键字在多层都能用，**范围不同**：

```yaml
env:                       # 作用于整个 workflow
  A: 1

jobs:
  job1:
    env:                   # 只作用于 job1
      B: 2
    steps:
      - run: echo $C
        env:               # 只作用于这一个 step
          C: 3
```

```
范围：  workflow 级  ⊃  job 级  ⊃  step 级
        (最大)                    (最小)
```

---
## 8. 一个完整示例：逐块拆解
```yaml
name: CI 流程                          # ━━ 第1层 ━━

on:                                    # 第1层：触发条件
  push:
    branches: [main]

jobs:                                  # 第1层：任务集合

  test:                                # ── 第2层：Job "test" ──
    runs-on: ubuntu-latest             # 第2层：机器
    steps:                             # 第2层：步骤列表
      - name: 拉取代码                  #   第3层：Step 1
        uses: actions/checkout@v4

      - name: 安装 Node                 #   第3层：Step 2
        uses: actions/setup-node@v4
        with:                          #   第3层：Action 的参数
          node-version: 20

      - name: 运行测试                  #   第3层：Step 3
        run: npm test

  deploy:                              # ── 第2层：Job "deploy" ──
    needs: test                        # 第2层：等 test 成功后才运行
    runs-on: ubuntu-latest
    steps:
      - name: 部署
        run: echo "部署中..."
```

用树形图看更清楚：

```
Workflow: CI 流程
│
├── on: push 到 main 分支时触发
│
└── jobs
     ├── test
     │    ├── runs-on: ubuntu-latest
     │    └── steps
     │         ├── 1. 拉取代码   (uses checkout)
     │         ├── 2. 安装 Node  (uses setup-node, with 版本)
     │         └── 3. 运行测试   (run npm test)
     │
     └── deploy
          ├── needs: test        ← 依赖关系：test 之后才跑
          ├── runs-on: ubuntu-latest
          └── steps
               └── 1. 部署       (run echo)
```

执行顺序：

```
        触发（push 到 main）
              │
              ▼
          ┌───────┐
          │ test  │  先跑
          └───┬───┘
              │ 成功后
              ▼
          ┌────────┐
          │ deploy │  再跑
          └────────┘
```

> 这个例子里你可能有些关键字没见过（`branches`、`with`、`needs`），**没关系**，现在只要能看出"这是三层结构"就达标了。

---
## 9. 常见错误对照
### 错误 1：键和 `-` 对齐（你之前遇到的）
```yaml
# ❌
steps:
  - name: A
  run: echo 1

# ✅
steps:
  - name: A
    run: echo 1
```

### 错误 2：用了 Tab 或缩进不统一

```yaml
# ❌ 有的 2 格，有的 4 格，混着来
jobs:
  build:
      runs-on: ubuntu-latest
```

### 错误 3：冒号后没有空格

```yaml
# ❌
runs-on:ubuntu-latest
# ✅
runs-on: ubuntu-latest
```

### 错误 4：关键字放错层级

```yaml
# ❌ runs-on 是 Job 级，不能放顶层
name: Test
runs-on: ubuntu-latest

# ✅
jobs:
  test:
    runs-on: ubuntu-latest
```

### 错误 5：一个 step 同时写 run 和 uses

```yaml
# ❌
- uses: actions/checkout@v4
  run: echo 1
```

---
## 10. 本节小结
1. YAML 三要素：**键值对、缩进嵌套、`-` 列表**，缩进用 2 个空格
2. workflow 文件分 **三层**：Workflow 级 → Job 级 → Step 级
3. 最小骨架只需要：`on`、`jobs`、`runs-on`、`steps`
4. 看到关键字先问"它属于哪一层"
5. `env`、`if` 等可出现在多层，范围从大到小
## 自测
- [ ] `runs-on` 属于哪一层？（Job 级）
- [ ] `uses` 和 `run` 属于哪一层？（Step 级）
- [ ] `on` 和 `jobs` 属于哪一层？（Workflow 级）
- [ ] 同一个 Step 里能同时写 `run` 和 `uses` 吗？（不能）
- [ ] 一个 step 里的 `run` 应该和什么对齐？（和 `name` 对齐）

---
## 学习路线

| 文件 | 内容 |
|---|---|
| ✅ 01-整体框架 | 全景图 |
| ✅ 02-yml文件结构 | 三层骨架（本节） |
| 03-事件on | 什么时候运行 |
| 04-Job与Step | 任务与步骤、依赖关系 |
| 05-Action | 现成插件的用法 |
| 06-变量与密钥 | env / secrets |
| 07-进阶特性 | matrix / cache 等 |
| 08-实战 | 完整 CI/CD |
