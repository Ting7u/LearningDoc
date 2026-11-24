# Git 学习笔记（第 4–8 章：日常使用阶段）

> 内容：基本操作 → 查看历史与比较差异 → 撤销与回退 → 分支管理 → 合并与冲突解决
> 说明：
> - 文中的提交哈希（如 `a1b2c3d`）仅为示意，实际值会不同。
> - `git switch` 和 `git restore` 是较新的命令（据我所知从 Git 2.23 引入），如果你的 Git 版本很旧，请先用 `git --version` 确认，或改用文中给出的 `checkout` 对应写法。
> - 命令输出的措辞可能随 Git 版本略有差异，以你本机为准。

---
## 第 4 章 基本操作
### 4.1 获取仓库：`git init` 与 `git clone`
**方式一：把现有目录变成仓库**

```bash
mkdir myproject
cd myproject
git init
```

会在当前目录生成一个隐藏的 `.git` 目录，它就是本地仓库，删除它就等于删除了全部历史。

**方式二：克隆远程仓库**

```bash
git clone https://github.com/用户名/仓库名.git
# 克隆到指定目录名
git clone https://github.com/用户名/仓库名.git mydir
```

克隆会把**完整历史**一起下载，并自动设置好名为 `origin` 的远程仓库。
### 4.2 查看状态：`git status`
```bash
git status        # 完整输出
git status -s     # 简短输出
```

简短输出示例：

```
 M README.md      ← 已修改，未暂存（M 在右列）
M  main.c         ← 已暂存（M 在左列）
MM app.js         ← 暂存后又修改了（两列都有）
A  new.txt        ← 新文件，已暂存
?? notes.txt      ← 未跟踪
```

左列表示暂存区的状态，右列表示工作区的状态。
### 4.3 暂存：`git add`
```bash
git add hello.txt        # 添加单个文件
git add src/             # 添加整个目录
git add .                # 添加当前目录下的所有改动
git add -A               # 添加整个仓库的所有改动（含删除）
git add -p               # 交互式选择"片段"，逐块决定是否暂存
```

> 关于 `git add .` 和 `-A` 的细节差异在不同 Git 版本中有过变化，日常使用时只要记住：在仓库根目录执行 `git add .` 或 `git add -A`，效果基本一致。

`git add -p` 很实用：同一个文件里改了两处不相关的内容，可以分两次提交。

```
Stage this hunk [y,n,q,a,d,s,e,?]?
```

常用按键：`y` 暂存这一块，`n` 跳过，`s` 把这一块再拆小，`q` 退出。
### 4.4 提交：`git commit`
```bash
git commit -m "fix: 修正登录页校验逻辑"   # 直接写提交说明
git commit                                 # 打开编辑器写较长的说明
git commit -a -m "update"                  # 自动暂存"已跟踪文件"的修改并提交
git commit --amend                         # 修改最近一次提交
```

要点：

- `-a` **只处理已被跟踪的文件**，不会包含新建的未跟踪文件。
- `--amend` 可以补充漏掉的文件或改提交说明：

```bash
git add forgotten.txt
git commit --amend --no-edit      # 把漏掉的文件并入上一次提交，说明不变
git commit --amend -m "新的说明"   # 只改说明
```

> `--amend` 会生成一个新提交来替换旧提交，**已经推送到远程的提交不要随便 amend**（第 12 章会详细讲）。

**好的提交说明**：

- 第一行简短概括"做了什么"（常见做法是控制在 50 字符左右，这是社区惯例而非硬性规定）
- 需要时空一行，再写"为什么这样改"
- 一次提交只做一件事

### 4.5 删除与移动：`git rm`、`git mv`

```bash
git rm old.txt             # 删除文件，并把删除操作加入暂存区
git rm --cached secret.txt # 只从 Git 中移除跟踪，保留磁盘上的文件
git mv a.txt b.txt         # 重命名/移动文件
```

`git mv a.txt b.txt` 等价于：

```bash
mv a.txt b.txt
git rm a.txt
git add b.txt
```

`git rm --cached` 非常常用：当你误把不该跟踪的文件提交了，用它停止跟踪，再写进 `.gitignore`。

### 4.6 `.gitignore`

作用：告诉 Git 哪些文件**不要纳入版本管理**（编译产物、依赖目录、日志、密钥等）。

在仓库根目录创建 `.gitignore` 文件：

```gitignore
# 注释以 # 开头
*.log              # 忽略所有 .log 文件
build/             # 忽略 build 目录（末尾斜杠表示目录）
node_modules/      # 忽略依赖目录
.env               # 忽略环境变量文件（常含密钥）

!important.log     # 感叹号表示"例外"：不忽略这个文件
/TODO.txt          # 开头斜杠：只匹配仓库根目录下的 TODO.txt
doc/**/*.pdf       # ** 匹配任意层级目录
```

规则速记：

| 写法 | 含义 |
|---|---|
| `*.o` | 通配符匹配 |
| `dir/` | 匹配目录 |
| `/name` | 只匹配根目录下的 |
| `!name` | 取消忽略（例外） |
| `**/` | 匹配任意层级目录 |

**最常见的坑：已经被跟踪的文件，写进 `.gitignore` 也不会生效。**

```bash
# 假设 config.local 已经被提交过，现在想忽略它
echo "config.local" >> .gitignore
git rm --cached config.local     # 先让 Git 停止跟踪
git add .gitignore
git commit -m "chore: stop tracking config.local"
```

### 4.7 小结

- 获取仓库：`init` / `clone`
- 日常三连：`status` → `add` → `commit`
- `add -p` 可以精细拆分提交；`commit -a` 不含新文件
- `rm --cached` 用于停止跟踪；`.gitignore` 对已跟踪文件无效

---

## 第 5 章 查看历史与比较差异

先准备一个有历史的练习仓库：

```bash
mkdir log-demo && cd log-demo && git init
echo "line1" > a.txt && git add a.txt && git commit -m "add a.txt"
echo "line2" >> a.txt && git commit -am "a.txt: add line2"
echo "hello" > b.txt && git add b.txt && git commit -m "add b.txt"
```

### 5.1 `git log`

```bash
git log
```

```
commit f1e2d3c4... (HEAD -> main)
Author: 你的名字 <you@example.com>
Date:   ...

    add b.txt
...
```

常用选项：

```bash
git log --oneline              # 每个提交一行：短哈希 + 说明
git log --oneline --graph --all --decorate   # 图形化显示所有分支
git log --stat                 # 显示每次提交改了哪些文件、多少行
git log -p                     # 显示每次提交的具体改动
git log -n 3                   # 只看最近 3 条
git log --author="张三"        # 按作者过滤
git log --since="2 weeks ago"  # 按时间过滤（也可用 --until）
git log --grep="fix"           # 按提交说明关键字过滤
git log -- a.txt               # 只看影响 a.txt 的提交
git log -p -- a.txt            # 看 a.txt 的每次具体改动
```

> `--oneline --graph --all --decorate` 组合很常用，很多人会给它设置别名（第 15 章讲别名）。

### 5.2 `git show`

查看某一次提交的详情（说明 + 改动内容）：

```bash
git show              # 最新提交
git show a1b2c3d      # 指定提交
git show HEAD~1       # 上一个提交
git show HEAD:a.txt   # 查看某个提交中某文件的内容
```

### 5.3 `git diff`：搞清楚"拿谁和谁比"

这是最容易混淆的命令，关键是记住三个区域（第 3 章）：

```
  工作区  ──(git diff)──  暂存区  ──(git diff --staged)──  最新提交 HEAD
```

| 命令 | 比较的对象 | 回答的问题 |
|---|---|---|
| `git diff` | 工作区 vs 暂存区 | 我改了但**还没 add** 的是什么？ |
| `git diff --staged`（同 `--cached`） | 暂存区 vs HEAD | 我 **add 了、即将提交** 的是什么？ |
| `git diff HEAD` | 工作区 vs HEAD | 距离上次提交，我总共改了什么？ |
| `git diff A B` | 提交 A vs 提交 B | 这两个版本差在哪？ |
| `git diff main feature` | 两个分支的最新提交 | 两个分支差在哪？ |

示例：

```bash
echo "line3" >> a.txt
git diff                # 能看到 line3（未暂存）
git add a.txt
git diff                # 没有输出了（工作区和暂存区一致）
git diff --staged       # 能看到 line3（已暂存，待提交）
```

读懂 diff 输出：

```diff
diff --git a/a.txt b/a.txt
--- a/a.txt
+++ b/a.txt
@@ -1,2 +1,3 @@
 line1
 line2
+line3
```

- `-` 开头的行：被删除
- `+` 开头的行：新增
- 空格开头的行：未变化的上下文
- `@@ -1,2 +1,3 @@`：旧文件从第 1 行起共 2 行，新文件从第 1 行起共 3 行

### 5.4 `git blame`

逐行显示"每一行最后是谁、在哪次提交中改的"：

```bash
git blame a.txt
git blame -L 1,5 a.txt     # 只看第 1~5 行
```

适合用来追问"这行代码为什么这样写"，拿到提交哈希后再 `git show` 看当时的完整改动和说明。

### 5.5 引用提交的写法

| 写法 | 含义 |
|---|---|
| `HEAD` | 当前所在提交 |
| `HEAD~1`（或 `HEAD~`） | HEAD 的上一个（父）提交 |
| `HEAD~3` | 往前数 3 代 |
| `HEAD^` | HEAD 的第一个父提交 |
| `HEAD^2` | HEAD 的第二个父提交（只对合并提交有意义） |
| `a1b2c3d` | 哈希前缀，只要足够唯一，写前 7 位通常就行 |
| `main`、`v1.0` | 分支名、标签名 |

对于只有一个父提交的普通提交，`HEAD~1` 和 `HEAD^` 效果相同；区别只在合并提交（有两个父提交）时才体现。

### 5.6 小结

- `log` 查历史，`show` 看单个提交，`blame` 追行级来源
- `diff` 不带参数 = 未暂存；`--staged` = 已暂存；`HEAD` = 全部
- `HEAD~n` 往前数 n 代

---

## 第 6 章 撤销与回退

> ⚠️ **本章有几个命令会让改动永久丢失**（尤其是 `restore <file>`、`reset --hard`、`clean`）。使用前先 `git status` 确认，必要时先备份或用 `-n` 预演。

按"你想撤销的东西处于哪个状态"来选命令：

| 想撤销的内容 | 使用的命令 |
|---|---|
| 工作区里还没 add 的修改 | `git restore <file>` |
| 已 add 但还没 commit（取消暂存） | `git restore --staged <file>` |
| 已 commit、但还没 push 的提交 | `git reset` |
| 已经 push、别人可能已拉取的提交 | `git revert` |
| 未跟踪的新文件 | `git clean` |

### 6.1 撤销工作区修改：`git restore <file>`

```bash
echo "oops" >> a.txt
git restore a.txt        # a.txt 恢复成暂存区（或最近提交）中的样子
```

⚠️ **未提交的修改会被丢弃，通常无法找回。**

从指定提交恢复某个文件：

```bash
git restore --source=HEAD~2 a.txt
```

### 6.2 取消暂存：`git restore --staged <file>`

```bash
git add a.txt
git restore --staged a.txt    # 从暂存区撤出，但工作区的修改保留
```

只是"取消 add"，**不会丢失内容**。

### 6.3 回退提交：`git reset`

`git reset <提交>` 会移动当前分支指针，使它指向指定的提交。三种模式的区别在于"同时对暂存区和工作区做什么"：

```
假设历史：A ← B ← C（HEAD，main）
执行：git reset <模式> B   → main 指针移到 B，C 不再出现在 main 的历史里
```

| 模式 | 移动分支指针 | 重置暂存区 | 重置工作区 | 效果 |
|---|---|---|---|---|
| `--soft` | ✅ | ❌ | ❌ | C 的改动**保留在暂存区**，可直接重新提交 |
| `--mixed`（默认） | ✅ | ✅ | ❌ | C 的改动**保留在工作区**，需重新 add |
| `--hard` | ✅ | ✅ | ✅ | C 的改动**全部丢弃** |

示例：

```bash
git reset --soft HEAD~1     # 撤销最近一次提交，改动留在暂存区（常用于"重新组织这次提交"）
git reset HEAD~1            # 撤销最近一次提交，改动留在工作区
git reset --hard HEAD~1     # 撤销最近一次提交，且丢掉改动 ⚠️
```

注意：

- `reset --hard` 会丢掉**未提交**的改动，这部分通常找不回来。
- 但被 reset 掉的**已提交**内容，在一段时间内通常还能通过 `git reflog` 找回（第 13 章讲）。
- **不要对已经推送、别人可能已基于它工作的提交使用 reset**，因为会造成历史分叉。

### 6.4 安全撤销公开提交：`git revert`

`revert` 不会删除历史，而是**新建一个"反向"的提交**来抵消指定提交的改动：

```
reset：  A ← B ← C        →  A ← B          （C 消失）
revert： A ← B ← C        →  A ← B ← C ← C'  （C' 是抵消 C 的新提交）
```

```bash
git revert a1b2c3d          # 撤销指定提交，并打开编辑器写说明
git revert HEAD             # 撤销最近一次提交
git revert --no-edit HEAD   # 使用默认说明，不打开编辑器
```

因为历史只增不改，所以**对已共享的提交，用 revert 是更安全的选择**。

如果要 revert 的是一个合并提交，需要用 `-m` 指定保留哪个父提交（例如 `-m 1`），这个话题比较深，待学到合并相关内容时再展开。

### 6.5 清理未跟踪文件：`git clean`

```bash
git clean -n         # 预演：只显示"将要删除"哪些文件，不真正删除
git clean -f         # 删除未跟踪的文件 ⚠️
git clean -fd        # 同时删除未跟踪的目录 ⚠️
```

**养成习惯：先 `-n` 预演，再 `-f` 执行。** 默认不会删除被 `.gitignore` 忽略的文件；如果加 `-x` 才会连忽略的文件一起删，需要格外小心。

### 6.6 旧命令 `checkout` 与新命令的对应关系

早期 Git 中 `git checkout` 一个命令身兼数职（切换分支 + 恢复文件），容易让人困惑，所以后来拆分成了 `switch` 和 `restore`：

| 目的 | 旧写法 | 新写法 |
|---|---|---|
| 切换分支 | `git checkout dev` | `git switch dev` |
| 创建并切换分支 | `git checkout -b dev` | `git switch -c dev` |
| 丢弃工作区修改 | `git checkout -- a.txt` | `git restore a.txt` |
| 取消暂存 | `git reset HEAD a.txt` | `git restore --staged a.txt` |

看别人的教程或老项目文档时经常遇到旧写法，认得即可，自己用新写法更清晰。

### 6.7 小结

- 先判断"改动在哪个区域"，再选命令
- `reset` 改写历史（仅限未共享的提交），`revert` 追加反向提交（可用于已共享的提交）
- 带 ⚠️ 的操作先预演、先确认

---

## 第 7 章 分支管理

### 7.1 分支的本质

回顾第 3 章：**分支就是一个指向提交的可移动指针**。创建分支几乎不花成本，因为只是新建了一个指针，没有复制文件。

```
创建 feature 分支前：
  A ← B ← C
          ↑
         main
         HEAD → main

创建 feature 分支后：
  A ← B ← C
          ↑
         main, feature
         HEAD → main

切换到 feature 并提交：
  A ← B ← C ← D
          ↑   ↑
        main  feature
              HEAD → feature
```

### 7.2 基本命令

```bash
git branch                  # 列出本地分支，* 标记当前分支
git branch -a               # 同时列出远程跟踪分支
git branch -vv              # 显示每个分支最新提交，以及与上游分支的关系
git branch feature          # 创建分支（不切换）
git switch feature          # 切换到已有分支
git switch -c feature       # 创建并切换（最常用）
```

完整示例：

```bash
git switch -c feature-login
echo "login code" > login.js
git add login.js
git commit -m "feat: add login"

git switch main             # 切回 main
ls                          # 此时看不到 login.js —— 工作区随分支切换而变化
git switch feature-login
ls                          # login.js 又回来了
```

这一步能直观体会到：**切换分支 = 把工作区换成那个分支最新提交的样子。**

### 7.3 切换时有未提交的修改怎么办

- 如果这些修改与目标分支**不冲突**，Git 会带着修改一起切换过去。
- 如果**会冲突**，Git 会拒绝切换并报错，提示你先提交或暂存修改。
- 最稳妥的做法：切换前先 commit；暂时不想提交就用 `git stash`（第 13 章讲）。

### 7.4 删除与重命名

```bash
git branch -d feature       # 删除已合并的分支（未合并会拒绝，起保护作用）
git branch -D feature       # 强制删除，不管是否合并 ⚠️
git branch -m old new       # 重命名分支
git branch -m new           # 重命名当前分支
```

注意：

- 不能删除你当前所在的分支，需要先切换到别的分支。
- 删除分支只是删掉指针；如果那些提交没有被任何其他分支或标签引用，之后可能被垃圾回收。强制删除前先确认。

### 7.5 分离头指针（detached HEAD）

正常情况下 `HEAD` 指向某个分支。如果直接切换到一个**提交**（而不是分支），就进入"分离头指针"状态：

```bash
git switch --detach a1b2c3d    # 或老写法：git checkout a1b2c3d
```

```
You are in 'detached HEAD' state. ...
```

```
正常：   HEAD → main → C
分离：   HEAD → B        （HEAD 直接指向一个提交，不属于任何分支）
```

适合的用途：**临时查看**某个历史版本、做一次性的试验。

风险：在这个状态下新提交的内容**没有分支指向它**，一旦切走，这些提交就"悬空"了，容易丢失。

如果在分离状态下已经做了想保留的提交，**立刻创建分支**把它挂住：

```bash
git switch -c rescue-branch
```

如果只是看看就想回去：

```bash
git switch main
```

### 7.6 小结

- 分支 = 指针，创建/切换/删除都很轻量
- 日常用 `git switch -c <name>` 开新分支，一个功能一个分支
- 删除用 `-d`（安全）优先，`-D` 慎用
- detached HEAD 不是错误，但新提交要用新分支保存

---

## 第 8 章 合并与冲突解决

### 8.1 `git merge` 基本用法

把 `feature` 分支的改动合并到 `main`：

```bash
git switch main          # 先切换到"接收方"分支
git merge feature        # 把 feature 合并进来
```

**记住：merge 是把"别人"合并进"当前所在的分支"。**

### 8.2 两种合并方式

**① 快进合并（fast-forward）**

条件：`main` 自从分出 `feature` 后**没有新提交**，`feature` 是 `main` 的直接"后代"。

```
合并前：
  A ← B            ← main
       ↖
         C ← D     ← feature

合并后（main 指针直接向前移动）：
  A ← B ← C ← D    ← main, feature
```

```
Updating b2c3d4e..e5f6a7b
Fast-forward
 login.js | 1 +
 1 file changed, 1 insertion(+)
```

没有产生新的提交，只是指针前进。

**② 三方合并（three-way merge）**

条件：两个分支**各自都有新提交**，历史发生了分叉。Git 会找到两者的共同祖先，结合"共同祖先、main 最新、feature 最新"三个点，生成一个**合并提交**：

```
合并前：
  A ← B ← E        ← main
       ↖
         C ← D     ← feature

合并后：
  A ← B ← E ← M    ← main
       ↖     ↗
         C ← D     ← feature
  （M 是合并提交，有两个父提交）
```

```
Merge made by the 'ort' strategy.
 login.js | 1 +
 ...
```

（策略名称如 `ort` 因 Git 版本而异，较老版本可能显示 `recursive`。）

**强制保留合并记录**：即使可以快进，也想留下一个合并提交，用 `--no-ff`：

```bash
git merge --no-ff feature
```

这样历史里能清楚看到"这个功能分支存在过"，是否使用取决于团队约定。

### 8.3 冲突：产生原因

当**两个分支修改了同一个文件的同一块区域**（或一边修改、一边删除），Git 无法自动判断保留哪边，就会停下来让你手动决定。

> 修改的是不同文件或同一文件的不同区域时，一般可以自动合并，不会冲突。

### 8.4 动手实验：制造并解决一次冲突

**准备：**

```bash
mkdir conflict-demo && cd conflict-demo && git init
echo "title: hello" > app.txt
git add app.txt && git commit -m "init"

# 在 feature 分支改第一行
git switch -c feature
echo "title: hello from feature" > app.txt
git commit -am "feature: change title"

# 回到 main，也改第一行
git switch main
echo "title: hello from main" > app.txt
git commit -am "main: change title"
```

**触发冲突：**

```bash
git merge feature
```

```
Auto-merging app.txt
CONFLICT (content): Merge conflict in app.txt
Automatic merge failed; fix conflicts and then commit the result.
```

**查看状态：**

```bash
git status
```

```
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   app.txt
```

**打开文件，会看到冲突标记：**

```
<<<<<<< HEAD
title: hello from main
=======
title: hello from feature
>>>>>>> feature
```

含义：

| 标记 | 含义 |
|---|---|
| `<<<<<<< HEAD` 到 `=======` | **当前分支**（main）的版本 |
| `=======` 到 `>>>>>>> feature` | **被合并分支**（feature）的版本 |

**解决步骤：**

1. 手动编辑文件，决定最终内容：保留一边、保留另一边，或者两边融合；
2. **删除全部冲突标记**（`<<<<<<<`、`=======`、`>>>>>>>` 三行都要删）；
3. 保存文件，例如最终内容改为：

   ```
   title: hello from main and feature
   ```

4. 标记为已解决并提交：

   ```bash
   git add app.txt
   git commit              # 会预填合并说明，保存退出即可
   ```

5. 用 `git log --oneline --graph` 查看合并后的历史。

> 一定要确认文件里不再残留冲突标记，否则这些标记会被当成普通文本提交进去。

### 8.5 放弃合并：`git merge --abort`

合并过程中发现冲突太复杂，或者想重来：

```bash
git merge --abort
```

会回到合并开始之前的状态。

### 8.6 合并工具：`git mergetool`

可以用图形化工具辅助解决冲突：

```bash
git mergetool
```

需要事先配置一个合并工具（例如 VS Code、Beyond Compare 等），具体配置项请参考你所用工具的文档和 `git help mergetool`，这里不展开。

### 8.7 减少冲突的习惯

- 分支生命周期不要太长，隔一段时间就把主分支的新改动合并进来
- 一个分支只做一件事，一次提交只做一件事
- 团队内约定好代码格式化规范，避免因为空格、换行引起的大面积差异
- 合并前先 `git status` 确认工作区干净

### 8.8 小结

- `merge` 把别的分支合并进**当前所在**分支
- 能快进就快进；分叉了就生成合并提交（`--no-ff` 可强制保留合并提交）
- 冲突 = 同一区域被两边改动；流程是"编辑 → 删标记 → `add` → `commit`"
- 搞不定就 `merge --abort`

---

## 第二阶段练习清单

1. 建一个练习仓库，用 `git add -p` 把同一文件中的两处改动拆成两次提交
2. 创建 `.gitignore`，并复现"已跟踪文件忽略不生效"，再用 `git rm --cached` 修复
3. 对同一个改动，分别观察 `git diff`、`git diff --staged`、`git diff HEAD` 的输出差异
4. 在练习仓库里分别体验 `reset --soft`、`--mixed`、`--hard`，每次执行后用 `git status` 和 `git log --oneline` 对照
5. 对一个提交使用 `git revert`，观察历史的变化，并与 `reset` 对比
6. 创建分支、切换分支，观察工作区文件的变化；练习进入和退出 detached HEAD
7. 按 8.4 的步骤，亲手制造并解决一次冲突；再试一次 `merge --abort`
8. 分别构造一次"快进合并"和一次"三方合并"，并用 `git log --oneline --graph --all` 观察形状差异

## 自测问题

- `git add .` 之后，`git diff` 为什么没有输出？该用什么命令查看已暂存的改动？
- `reset --soft`、`--mixed`、`--hard` 分别保留了什么？
- 什么时候用 `reset`，什么时候用 `revert`？
- 什么是 detached HEAD？在里面做了提交该怎么保住？
- 快进合并和三方合并的区别是什么？
- 冲突标记里 `HEAD` 一侧和另一侧分别是谁的版本？
- 解决冲突后，为什么还要 `git add`？
