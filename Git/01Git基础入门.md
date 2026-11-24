
> 内容：版本控制与 Git 简介 → 安装与初始配置 → 核心概念
> 说明：文中出现的提交哈希（如 `a1b2c3d`）仅为示意，你实际操作时的值会不同。
## 第 1 章 认识版本控制与 Git
### 1.1 为什么需要版本控制
先看一个没有版本控制时的典型场景：

```
毕业论文.docx
毕业论文_改1.docx
毕业论文_最终版.docx
毕业论文_最终版_真的最终.docx
毕业论文_最终版_真的最终_导师修改.docx
```

**这种方式的问题很明显：**
- 不知道每个版本之间到底改了什么
- 想回到三天前的某个状态，只能靠翻文件
- 多人同时修改，很容易互相覆盖
- 想尝试一个大胆的改动，又怕改坏了回不去

版本控制系统（VCS）解决的就是这几件事：

| 需求  | 版本控制提供的能力            |
| --- | -------------------- |
| 回溯  | 随时回到历史上任意一个版本        |
| 对比  | 精确知道每次改了哪些行、谁改的、为什么改 |
| 协作  | 多人并行开发，再把成果合并到一起     |
| 备份  | 完整历史保存在多个位置，不怕单点丢失   |
| 实验  | 通过分支安全地试验新想法，失败了直接丢弃 |

### 1.2 集中式与分布式
**集中式（如 SVN）**

```
        ┌──────────────┐
        │  中央服务器    │  ← 完整历史只在这里
        └──────────────┘
         ↑    ↑    ↑
       开发者A 开发者B 开发者C   ← 本地只有当前版本的文件
```

- 提交、查看历史等操作基本都要连接服务器
- 服务器故障或断网时，很多操作无法进行

**分布式（Git）**

```
 ┌────────────┐      ┌────────────┐
 │ 开发者A 本地 │ ←──→ │ 远程仓库    │ ←──→ 开发者B 本地
 │ 完整历史     │      │ 完整历史    │      完整历史
 └────────────┘      └────────────┘
```

- 每个人的本地仓库都有**完整的历史**
- 提交、查看日志、切换分支、回退等绝大多数操作在**本地完成**，不需要联网
- 只有"同步到远程"（push）和"从远程获取"（fetch / pull）才需要网络
- 任何一份克隆在理论上都可以充当备份
### 1.3 Git、GitHub、GitLab、Gitee 的关系
最容易混淆的一点：**Git 是工具，GitHub 等是托管平台**。

| 名称 | 是什么 |
|---|---|
| **Git** | 运行在你电脑上的版本控制软件（命令行工具），不依赖任何网站 |
| **GitHub** | 基于 Git 的代码托管平台，提供远程仓库、Pull Request、Issues、Actions 等 |
| **GitLab** | 类似平台，可以自建（私有部署），也提供云服务 |
| **Gitee（码云）** | 国内的代码托管平台，国内访问通常更稳定 |

类比：Git 像是"Word 软件"，GitHub 像是"网盘 + 协作平台"。没有网盘，Word 照样能用；没有平台，Git 也能完整地管理本地项目。
### 1.4 小结
- 版本控制 = 回溯 + 对比 + 协作 + 备份 + 实验
- Git 是分布式的，每个本地仓库都有完整历史
- Git 是工具，GitHub / GitLab / Gitee 是托管平台
## 第 2 章 安装与初始配置
### 2.1 安装 Git
**Windows**：从 Git 官网下载安装包（https://git-scm.com），安装后会得到 Git Bash 终端。安装过程中大部分选项保持默认即可。

**macOS**：

```bash
# 方式一：使用 Homebrew
brew install git

# 方式二：首次在终端输入 git，系统可能提示安装命令行开发工具
git --version
```

**Linux**：

```bash
# Debian / Ubuntu
sudo apt install git

# Fedora
sudo dnf install git
```

**验证安装**：

```bash
git --version
```

输出类似（版本号因人而异）：

```
git version 2.43.0
```
### 2.2 首次必做的配置：用户信息
每次提交都会记录"谁提交的"，所以**必须先设置用户名和邮箱**：

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
```

**注意：**
- 这里的名字和邮箱会写进每一次提交，**不是登录账号**
- 如果要让 GitHub 把提交关联到你的账号，邮箱需要与平台上绑定的邮箱一致（具体规则以平台设置页面为准）
### 2.3 其他常用配置
```bash
# 设置默认分支名为 main（新建仓库时生效）
git config --global init.defaultBranch main

# 设置默认编辑器（例如 VS Code）
git config --global core.editor "code --wait"

# 开启命令行彩色输出
git config --global color.ui auto
```

查看配置：

```bash
# 查看所有配置及其来源文件
git config --list --show-origin

# 查看单项
git config user.name
```
### 2.4 配置的三个层级

| 层级 | 参数 | 作用范围 | 常见存储位置* |
|---|---|---|---|
| 系统级 | `--system` | 本机所有用户 | `/etc/gitconfig`（Linux/macOS） |
| 全局级 | `--global` | 当前用户的所有仓库 | `~/.gitconfig` 或 `~/.config/git/config` |
| 仓库级 | `--local`（默认） | 仅当前仓库 | 仓库内的 `.git/config` |

*具体路径因操作系统和安装方式而异，可用 `git config --list --show-origin` 确认你机器上的实际位置。

**优先级：仓库级 > 全局级 > 系统级**（范围越小，优先级越高）。

典型用法：全局配置个人邮箱，但某个公司项目要用公司邮箱：

```bash
# 在公司项目的目录里执行（不加 --global，即仓库级）
cd company-project
git config user.email "me@company.com"
```

这样只有这个仓库的提交会使用公司邮箱，其他仓库不受影响。
### 2.5 远程认证：HTTPS 与 SSH
连接 GitHub 等远程仓库时，有两种常见认证方式：

| 方式 | 远程地址形式 | 特点 |
|---|---|---|
| HTTPS | `https://github.com/用户名/仓库.git` | 上手简单；GitHub 已不支持账号密码推送，通常需要使用个人访问令牌（PAT）或凭据管理器 |
| SSH | `git@github.com:用户名/仓库.git` | 配好密钥后无需反复输入凭据，长期使用更方便 |

**配置 SSH 的基本流程**：

```bash
# 1. 生成密钥对（推荐 ed25519 算法）
ssh-keygen -t ed25519 -C "you@example.com"
# 一路回车即可，默认保存在 ~/.ssh/id_ed25519

# 2. 查看公钥内容（注意是 .pub 结尾的公钥）
cat ~/.ssh/id_ed25519.pub

# 3. 把公钥内容复制到 GitHub：
#    Settings → SSH and GPG keys → New SSH key

# 4. 测试连接
ssh -T git@github.com
```

成功时会看到类似"Hi 用户名! You've successfully authenticated..."的提示。

> **安全提醒**：只把 `.pub` 公钥交给平台；**私钥（无 .pub 后缀的文件）绝不能泄露**。
### 2.6 小结
- 安装后第一件事：配置 `user.name` 和 `user.email`
- 配置分三层，范围越小优先级越高
- 认证方式二选一：HTTPS（令牌）或 SSH（密钥）
## 第 3 章 核心概念
这一章是整个 Git 的地基，建议动手跟着做一遍。
### 3.1 三个区域
Git 管理的文件分布在三个地方：
![[Pasted image 20261122165842.png]]

| 区域 | 是什么 | 类比 |
|---|---|---|
| **工作区** | 你在磁盘上看到、直接编辑的文件 | 书桌上正在写的稿子 |
| **暂存区** | 下一次提交要包含的内容清单 | 打包前先放进纸箱的东西 |
| **本地仓库** | 保存所有提交历史的数据库（位于 `.git` 目录） | 封箱后存进仓库的档案 |

**为什么要有暂存区？** 它让你可以**选择性提交**。比如你同时改了 A、B 两个文件，但只想把 A 作为一次提交，就只 `add A` 再提交，B 留到下一次。这样每次提交都能保持"只做一件事"，历史更清晰。
### 3.2 动手实验：走完一遍完整流程
```bash
# 创建练习目录并初始化仓库
mkdir git-demo
cd git-demo
git init
```

输出（默认分支名取决于你的配置）：

```
Initialized empty Git repository in /.../git-demo/.git/
```

**第 1 步：新建文件，状态为"未跟踪"**

```bash
echo "hello git" > hello.txt
git status
```

```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        hello.txt

nothing added to commit but untracked files present (use "git add" to track)
```

`Untracked files` 表示 Git 发现了这个文件，但还没有在管理它。

**第 2 步：`git add`，放入暂存区，状态为"已暂存"**

```bash
git add hello.txt
git status
```

```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   hello.txt
```

`Changes to be committed` 表示：它已在暂存区，下次提交会包含它。

**第 3 步：`git commit`，进入仓库，状态为"已提交"**

```bash
git commit -m "add hello.txt"
```

```
[main (root-commit) a1b2c3d] add hello.txt
 1 file changed, 1 insertion(+)
 create mode 100644 hello.txt
```

```bash
git status
```

```
On branch main
nothing to commit, working tree clean
```

`working tree clean` 表示工作区与最新提交一致，没有任何待处理的改动。

**第 4 步：修改文件，状态为"已修改"**

```bash
echo "second line" >> hello.txt
git status
```

```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   hello.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

注意区分两个很像的标题：

- `Changes to be committed` → **已暂存**，下次提交会包含
- `Changes not staged for commit` → **已修改但未暂存**，下次提交不会包含

再走一遍 `git add hello.txt` → `git commit -m "add second line"`，又回到干净状态。
### 3.3 文件的四种状态
![[Pasted image 20261122165943.png]]

| 状态 | 含义 | `git status` 中的表现 |
|---|---|---|
| 未跟踪（Untracked） | 新文件，Git 还没管理它 | `Untracked files` |
| 已修改（Modified） | 已被管理的文件，内容有改动但未暂存 | `Changes not staged for commit` |
| 已暂存（Staged） | 改动已放入暂存区，等待提交 | `Changes to be committed` |
| 已提交（Committed） | 改动已安全保存进仓库 | 不出现（`working tree clean`） |

**一个容易困惑的情况：同一个文件可以同时处于"已暂存"和"已修改"两种状态。**

```bash
echo "v1" > a.txt
git add a.txt          # 暂存了 v1
echo "v2" >> a.txt     # 之后又改了
git status
```

此时 `a.txt` 会同时出现在 `Changes to be committed` 和 `Changes not staged for commit` 下。原因：暂存区里存的是 `add` 那一刻的内容（v1），后来的修改（v2）还没暂存。此时提交，只会提交 v1；想把 v2 也包含进去，需要再 `git add` 一次。
### 3.4 提交（commit）、HEAD 与分支
**提交（commit）**
一次提交就是历史中的一个节点，包含：

- 项目在那一刻的完整快照
- 作者、提交时间
- 提交说明（message）
- 指向**父提交**的引用（第一次提交没有父提交）

每个提交有唯一的哈希值作标识（如 `a1b2c3d...`，完整为 40 位十六进制字符，日常常用前 7 位简写）。

查看一下：

```bash
git log --oneline
```

```
f4e5d6c add second line
a1b2c3d add hello.txt
```

提交之间通过"父指针"串成一条链：

```
a1b2c3d  ←──  f4e5d6c
(第一次)       (第二次，父提交是 a1b2c3d)
```

**分支（branch）**

分支本质上只是**一个指向某个提交的可移动指针**，并不是把代码复制了一份。默认分支通常叫 `main`（旧版本默认名是 `master`）。每次在当前分支上提交，这个分支指针就自动向前移动到新提交。

```
a1b2c3d ← f4e5d6c
              ↑
            main
```

**HEAD**

`HEAD` 是一个特殊指针，表示"**你当前所在的位置**"。通常它指向当前分支，再由分支指向最新提交：

```
HEAD → main → f4e5d6c
```

当你切换分支，HEAD 就改指向另一个分支。更多细节会在后面"分支管理"一章展开，这里先建立印象：

- **提交** = 历史节点
- **分支** = 指向提交的标签（指针）
- **HEAD** = 指向"当前所在分支"的标签

可以亲自验证：

```bash
cat .git/HEAD
```

```
ref: refs/heads/main
```
### 3.5 理解"快照"而非"差异"
很多版本控制系统（如 SVN）按"**差异**"存储：记录每个版本相对上一个版本改了什么。

```
文件A: 版本1 → Δ1 → Δ2 → Δ3
```

Git 的思路是存"**快照**"：每次提交，相当于给整个项目拍一张照片并保存。

```
提交1: [A1, B1, C1]
提交2: [A2, B1, C1]   ← 只有 A 变了，B、C 没变，就直接引用之前的存储，不重复保存
提交3: [A2, B2, C1]
```

要点：

- 每个提交都是项目的**完整状态**，因此切换到任意版本非常快，不需要把一连串差异依次"回放"
- 没变的文件不会被重复存储，所以并不会因为"存快照"而占用成倍的空间
- 你在 `git diff`、`git log -p` 里看到的"差异"，是 Git **在需要时通过比较两份快照临时算出来的**，并不是它存储的内容
### 3.6 小结

| 概念 | 一句话总结 |
|---|---|
| 工作区 | 你正在编辑的文件 |
| 暂存区 | 下一次提交的内容清单，让你可以选择性提交 |
| 本地仓库 | 保存全部历史的数据库（`.git` 目录） |
| 四种状态 | 未跟踪 → 已修改 → 已暂存 → 已提交 |
| 提交 | 项目在某一时刻的完整快照 + 元信息 + 父指针 |
| 分支 | 指向某个提交的可移动指针 |
| HEAD | 指向当前所在位置（通常是当前分支） |
| 存储思路 | 存快照，差异是临时算出来的 |

## 本阶段练习清单
1. 安装 Git，并完成 `user.name`、`user.email`、`init.defaultBranch` 的配置
2. 用 `git config --list --show-origin` 查看各配置来自哪个文件
3. （可选）生成 SSH 密钥，添加到 GitHub，并用 `ssh -T git@github.com` 测试
4. 按 3.2 的步骤，亲手走一遍"未跟踪 → 已暂存 → 已提交 → 已修改"，每一步都执行 `git status` 对照输出
5. 复现 3.3 中"同一文件同时已暂存和已修改"的情况，并思考此时提交会包含哪个版本
6. 用 `git log --oneline` 和 `cat .git/HEAD` 观察提交链与 HEAD
## 自测问题
- Git 和 GitHub 的区别是什么？
- 为什么说 Git 是分布式的？断网时哪些操作还能做？
- `--system`、`--global`、`--local` 谁的优先级最高？
- 暂存区的作用是什么？
- `Changes to be committed` 和 `Changes not staged for commit` 有何区别？
- 分支是"复制了一份代码"吗？HEAD 又是什么？
