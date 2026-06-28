# 📘 GitHub 完全指南

> 从原理到实战的 Git & GitHub 完整教程 — 不只是命令，更是理解。

---

## 目录

- [一、版本控制的本质](#一版本控制的本质)
- [二、Git 核心原理](#二git-核心原理)
- [三、Git 基础命令](#三git-基础命令)
- [四、Git 远程协作](#四git-远程协作)
- [五、GitHub 平台深度使用](#五github-平台深度使用)
- [六、GitHub CLI（gh）实战](#六github-cligh-实战)
- [七、工作流实战](#七工作流实战)
- [八、高级话题](#八高级话题)
- [九、踩坑集锦与最佳实践](#九踩坑集锦与最佳实践)

---

## 一、版本控制的本质

### 1.1 为什么需要版本控制

想象你正在写一篇论文：

```
论文_v1.docx
论文_v2_导师修改.docx
论文_v3_最终版.docx
论文_v3_最终版_真的最终.docx
论文_v3_最终版_再也不改了.docx
```

这就是没有版本控制的噩梦。版本控制系统（VCS，Version Control System）解决了三个核心问题：

| 问题 | 解决方案 |
|------|----------|
| **回溯** — 改了哪些、谁改的、为什么改 | 每次保存生成一个快照，附带元信息 |
| **协作** — 多人同时改同一个文件 | 分支 + 合并机制，独立工作后集成 |
| **备份** — 电脑坏了、文件丢了 | 分布式架构，每个克隆都是完整备份 |

### 1.2 Git 的设计哲学：快照 vs 差异

大多数 VCS（如 SVN）以**文件差异**（delta）方式存储历史：

```
文件 v1 → [差异] → 文件 v2 → [差异] → 文件 v3
```

Git 则完全不同 — **每次 commit 是整个项目的完整快照**：

```
commit1: 文件A(v1) 文件B(v1) 文件C(v1)
commit2: 文件A(v2) 文件B(v1) 文件C(v1)  ← 只变了一个文件，但存了整个状态
```

> **为什么要这么“浪费”？** 因为读取历史时不需要反向计算差异链，切换版本是 O(1) 操作。而且 Git 会对未变化的文件使用符号链接（通过 `.git/objects` 的 hash 去重），实际磁盘占用并不大。

### 1.3 分布式 vs 集中式

```
集中式（SVN）：               分布式（Git）：
┌──────────┐                ┌──────────┐
│ 中央仓库  │ ← 唯一真理来源    │ 远程仓库  │
└────┬─────┘                └────┬─────┘
     │                          │
┌────┴────┐              ┌──────┼──────┐
│ 工作副本  │             │ 本地仓库  │ 本地仓库  │
└─────────┘              └──────┴──────┘
```

**分布式意味着：**
- 每个开发者拥有**完整的历史副本**，不只是最新版本
- 离线可 commit、branch、merge、view log
- 没有单点故障 — 任何克隆都可以恢复整个项目
- 工作流灵活 — 不需要中央服务器也能协作（Linux 内核就是这样开发的）

---

## 二、Git 核心原理

### 2.1 `.git` 目录解剖

当你执行 `git init` 时，Git 在当前目录创建一个 `.git` 文件夹。**这是 Git 的全部** — 删除它就等于删除所有版本历史。

```
.git/
├── HEAD              # 指向当前分支的引用（如 ref: refs/heads/main）
├── config            # 仓库级别的配置
├── objects/          # 对象数据库（核心！所有内容存在这里）
│   ├── xx/           # 以 hash 前两位为目录名
│   │   └── xxxx...   # 实际对象文件（blob / tree / commit）
│   ├── info/
│   └── pack/         # 打包文件（垃圾回收后生成）
├── refs/
│   ├── heads/        # 分支引用（每个分支一个文件，内容是 commit hash）
│   ├── tags/         # 标签引用
│   └── remotes/      # 远程分支引用
├── index             # 暂存区（staging area）
└── logs/             # 引用日志（reflog）
```

### 2.2 三种状态和三个区域

Git 中的文件永远处于三种状态之一：

```
工作目录               暂存区                  Git 仓库
(Working Dir)    →    (Staging Area)    →    (Repository)
                        index                  objects/
   modified            staged                 committed
```

```bash
# 查看文件状态
git status

# 典型输出解读：
# Untracked  → 新文件，Git 不知道它的存在
# Modified   → 已跟踪的文件被修改了，但还没暂存
# Staged     → 修改已被加入暂存区，等待 commit
# Committed  → 安全存入本地数据库
```

### 2.3 四对象模型

Git 内部只有四种对象类型，理解它们就理解了 Git：

```
┌──────────────────────────────────────┐
│              commit                   │
│  tree:    <tree-hash>                │
│  parent:  <parent-commit-hash>       │
│  author:  Frank <sunangie@126.com>   │
│  message: "Add login feature"        │
└──────────┬───────────────────────────┘
           │
    ┌──────▼──────────┐
    │      tree        │
    │  src/    → <hash>│  ← 目录（tree 指向 blob 或 tree）
    │  README  → <hash>│
    └──────┬──────────┘
           │
    ┌──────▼──────────┐
    │      blob        │  ← 文件内容
    │  "print('hello') │     注意：blob 只存内容，不存文件名！
    │   world'\\"       │     文件名存在 tree 对象里
    └─────────────────┘
```

**底层验证：**

```bash
# 查看一个 commit 的内部结构
git cat-file -p HEAD

# 输出示例：
# tree 4b825dc642cb6eb9a060e54bf8993c2c4ea9a8e2
# parent 1a2b3c4d5e6f...
# author Frank <sunangie@126.com> 1700000000 +0800
# committer Frank <sunangie@126.com> 1700000000 +0800
#
# Add login feature

# 继续深入，查看 tree 对象
git cat-file -p 4b825dc6

# 输出：
# 100644 blob e69de29bb2...    README.md
# 040000 tree a1b2c3d4e5...    src

# 再深入，查看 blob 内容
git cat-file -p e69de29b
```

> **关键洞察：** commit 指向 tree，tree 指向 blob 或 tree。这是一个**内容寻址**的文件系统 — 你可以把 Git 看作一个迷你文件系统，所有内容通过 SHA-1 hash 寻址。

### 2.4 分支的本质

**分支只是一个指向 commit 的指针。**

```
# .git/refs/heads/main 的内容：
$ cat .git/refs/heads/main
1a2b3c4d5e6f7890abcdef1234567890abcdef12
```

就这么简单 — 一个 40 字符的 SHA-1 hash，存在一个文件里。

```
main
 │
 ▼
A ← B ← C ← D
         ▲
         │
      feature
```

- `main` 指向 D
- `feature` 也指向 D（刚创建的）
- 当你 checkout 到 `feature` 并 commit，`feature` 就会前移
- `main` 不动，直到 merge 或 commit

**HEAD 是什么？** HEAD 是一个特殊的指针，指向你当前所在的 commit。通常它指向一个分支（detached HEAD 时直接指向 commit）。

```bash
# HEAD 的内容（通常是符号引用）
$ cat .git/HEAD
ref: refs/heads/feature/login

# detached HEAD 时直接指向 commit
$ cat .git/HEAD
1a2b3c4d5e6f7890abcdef1234567890abcdef12
```

---

## 三、Git 基础命令

### 3.1 init / clone — 仓库的诞生

```bash
# 从零开始：在当前目录初始化
git init
git init my-project    # 在 my-project/ 下初始化

# 从远程获取：克隆完整仓库
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git          # SSH（推荐）
git clone git@github.com:user/repo.git my-dir   # 指定目录名
git clone --depth 1 git@github.com:user/repo.git # 浅克隆（只要最新版本）
git clone --branch develop git@github.com:user/repo.git # 克隆指定分支
```

> **浅克隆（--depth 1）：** 只下载最新一个 commit，不下载历史。适合 CI 环境或大型仓库。注意：浅克隆不能 push 回原仓库（历史不完整），需要先 `git fetch --unshallow`。

### 3.2 add / commit — 快照的生成

```bash
# 将修改加入暂存区
git add file.txt              # 添加单个文件
git add src/                  # 添加整个目录
git add .                     # 添加当前目录所有修改
git add -A                    # 添加所有修改（包括删除）
git add -p                    # 交互式暂存，逐块选择（推荐！）

# 提交快照
git commit -m "feat: add login feature"
git commit -a -m "fix: correct typo"        # 跳过 add，直接 commit 所有已跟踪文件的修改
git commit --amend                           # 修改最后一次 commit（慎用，见下方说明）
```

**`git add -p` 实战 — 最被低估的命令：**

```bash
$ git add -p
diff --git a/src/app.py b/src/app.py
@@ -10,6 +10,10 @@
     return data

+def login(user, password):       ← Git 展示这个块
+    if auth_service.verify(user, password):
+        return create_session(user)
+    raise AuthError()
+
 def main():
     app.run()

(1/1) Stage this hunk [y,n,q,a,d,e,?]? y
```

这样你可以**精确控制**每次 commit 包含什么 — 即使在一个文件里改了多个无关的事情，也可以分批提交。

**`--amend` 的陷阱：**

```bash
# ⚠️ 只在 commit 还没 push 时使用！
git commit --amend -m "better message"

# 如果已经 push 了，--amend 会重写历史，需要用 --force
# 这在共享分支上是灾难性的 — 会覆盖别人的工作
```

### 3.3 status / log / diff — 状态感知

```bash
# 查看工作区状态
git status
git status -s                 # 简短格式

# 查看提交历史
git log                       # 完整历史
git log --oneline             # 一行一个 commit
git log --graph --all --decorate  # 分支图（强烈推荐）
git log -3                    # 最近 3 条
git log --author="Frank"      # 按作者筛选
git log --since="2026-06-01"  # 按日期筛选
git log --grep="login"        # 搜索 commit message
git log -S "function_name"    # 搜索代码变更内容（pickaxe）
git log --stat                # 显示文件变更统计

# 查看差异
git diff                      # 工作区 vs 暂存区（还没 add 的改动）
git diff --staged             # 暂存区 vs 最新 commit（已 add 但未 commit 的）
git diff HEAD                 # 工作区 vs 最新 commit（所有未提交的改动）
git diff main..feature        # 两个分支的差异
git diff --word-diff          # 词级别差异（比行级别更细腻）
```

**推荐别名（加入 `~/.gitconfig`）：**

```ini
[alias]
    lg = log --graph --oneline --all --decorate
    lga = log --graph --oneline --all --decorate --author-date-order
    st = status -s
    df = diff --word-diff
    co = checkout
    br = branch
    ci = commit
```

### 3.4 branch / checkout / switch — 分支操作

```bash
# 查看分支
git branch                    # 本地分支列表
git branch -a                 # 包括远程分支
git branch -v                 # 显示每个分支最后一次 commit

# 创建分支
git branch feature/login      # 创建但不切换
git checkout -b feature/login # 创建并切换（经典写法）
git switch -c feature/login   # 创建并切换（Git 2.23+ 推荐）

# 切换分支
git checkout feature/login    # 经典写法
git switch feature/login      # 新写法（语义更清晰）

# 删除分支
git branch -d feature/login   # 安全删除（已 merge 的）
git branch -D feature/login   # 强制删除（未 merge 的也删）

# 重命名分支
git branch -m old-name new-name
```

> **checkout vs switch：** `git checkout` 承担了太多职责（切换分支 + 恢复文件），Git 2.23 引入了 `git switch`（只切换分支）和 `git restore`（只恢复文件），语义更清晰。

### 3.5 merge / rebase — 合并策略详解

这是 Git 中最需要理解的命令。两种方式实现同一个目标：把一个分支的改动合并到另一个分支。

**Merge（合并）— 保留真实历史：**

```
# 初始状态
     E ← feature
    /
A ← B ← C ← D ← main

# git checkout main && git merge feature
     E ← feature
    / \
A ← B ← C ← D ← M ← main  ← M 是 merge commit
```

```bash
git checkout main
git merge feature/login

# 冲突时：
# <<<<<<< HEAD
# 当前分支的代码
# =======
# 要合并进来的代码
# >>>>>>> feature/login
```

**Rebase（变基）— 重写历史使其线性：**

```
# 初始状态
     E ← feature
    /
A ← B ← C ← D ← main

# git checkout feature && git rebase main
A ← B ← C ← D ← E' ← feature  ← 注意 E' 是全新的 commit
                ↑
               main
```

```bash
git checkout feature/login
git rebase main

# 如果冲突，逐个处理：
# 1. 解决冲突文件
# 2. git add 冲突文件
# 3. git rebase --continue  （继续下个 commit）
# 4. git rebase --abort     （放弃整个 rebase）
```

**如何选择？**

| 场景 | 推荐 | 原因 |
|------|------|------|
| 公共分支（main / develop） | Merge | 不重写历史，不破坏他人工作 |
| 个人功能分支 | Rebase | 保持历史干净线性 |
| Pull Request 合并到 main | Squash Merge | 一个功能一个 commit |
| 需要保留分支开发过程细节 | Merge（--no-ff） | 保留所有 commit 和分支拓扑 |

**Interactive Rebase — 历史编辑神器：**

```bash
# 编辑最近 3 个 commit
git rebase -i HEAD~3

# 进入编辑器：
pick abc1234 feat: add login
pick def5678 fix: typo
pick ghi9012 wip: debugging

# 修改为：
pick abc1234 feat: add login
fixup def5678 fix: typo        ← squash 到上一个，丢弃 message
reword ghi9012 wip: debugging  ← 保留但修改 message
```

**可用操作：**
- `pick` — 保留
- `reword` — 保留但修改 message
- `squash` — 合并到上一个，合并 message
- `fixup` — 合并到上一个，丢弃 message
- `drop` — 删除这个 commit
- `edit` — 停下来修改这个 commit 的内容

> **黄金法则：永远不要 rebase 已经 push 到公共仓库的 commit。** 变基会产生新的 commit hash，任何基于旧 hash 的工作都会断裂。

### 3.6 reset / revert / restore — 撤销的艺术

这三个命令是 Git 中混淆最多的地方。一句话区分：

| 命令 | 做什么 | 影响历史？ | 安全程度 |
|------|--------|-----------|----------|
| `git revert` | 创建新 commit 撤销旧 commit | 否（追加） | 🟢 安全 |
| `git reset` | 移动分支指针，丢弃 commit | 是（重写） | 🔴 危险 |
| `git restore` | 恢复文件到某个版本 | 否 | 🟢 安全 |

```bash
# revert — 安全撤销（推荐用于公共分支）
git revert abc1234                    # 撤销那个 commit，生成新的 revert commit
git revert abc1234..def5678           # 撤销一个范围

# reset — 重写历史（只用在自己的分支）
git reset --soft HEAD~1               # 撤销 commit，改动留在暂存区
git reset --mixed HEAD~1              # 撤销 commit + unstage，改动留在工作区（默认）
git reset --hard HEAD~1               # 撤销 commit + unstage + 丢弃改动（⚠️ 彻底删除）

# restore — 恢复文件（Git 2.23+）
git restore file.txt                  # 丢弃工作区的修改（回到暂存区状态）
git restore --staged file.txt         # 取消暂存（unstage）
git restore --source=HEAD~2 file.txt  # 恢复到 2 个 commit 之前的版本
```

**reset 三种模式的对比：**

```
--soft:   只移动 HEAD → commit 的内容回到暂存区
--mixed:  移动 HEAD + 更新暂存区 → commit 的内容回到工作区
--hard:   移动 HEAD + 更新暂存区 + 覆盖工作区 → 一切回到指定状态
```

**典型工作流示例：**

```bash
# 场景：刚 commit 了，但发现漏了一个文件
git add forgotten-file.txt
git commit --amend --no-edit          # 追加到上一个 commit

# 场景：commit message 写错了
git commit --amend -m "correct message"

# 场景：commit 到错误的分支了
git log --oneline -1                  # 记下 commit hash
git checkout correct-branch
git cherry-pick <commit-hash>         # 把那个 commit 摘过来
git checkout wrong-branch
git reset --hard HEAD~1               # 从错误分支删除
```

### 3.7 stash / cherry-pick — 灵活工具箱

```bash
# stash — 暂存未完成的工作
git stash                             # 暂存所有改动
git stash push -m "WIP: login form"   # 附带描述
git stash list                        # 查看暂存列表
git stash pop                         # 恢复最近一次暂存并删除
git stash apply stash@{1}             # 恢复指定暂存但不删除
git stash drop stash@{0}              # 删除指定暂存
git stash show -p stash@{0}           # 查看暂存内容

# cherry-pick — 摘取指定 commit 到当前分支
git cherry-pick abc1234                # 单个 commit
git cherry-pick abc1234..def5678       # 范围（不包含 abc1234）
git cherry-pick abc1234^..def5678      # 范围（包含 abc1234）
```

### 3.8 tag — 里程碑标记

```bash
# 轻量标签（只是一个指向 commit 的指针）
git tag v1.0.0

# 附注标签（包含作者、日期、message，推荐用于发布）
git tag -a v1.0.0 -m "Release version 1.0.0"

# 查看标签
git tag                               # 列出所有标签
git show v1.0.0                       # 查看标签详情

# 推送标签
git push origin v1.0.0                # 推送单个标签
git push origin --tags                # 推送所有标签

# 删除标签
git tag -d v1.0.0                     # 删除本地标签
git push origin --delete v1.0.0       # 删除远程标签
```

> **标签 vs 分支：** 标签是固定的（指向一个永不移动的 commit），分支是移动的（随新 commit 前移）。用标签标记发布版本，用分支进行开发。

---

## 四、Git 远程协作

### 4.1 remote / fetch / pull / push — 同步机制

```bash
# 管理远程仓库
git remote -v                                    # 查看远程仓库
git remote add upstream git@github.com:org/repo.git  # 添加远程仓库
git remote rename origin upstream                # 重命名
git remote remove upstream                       # 删除

# fetch — 下载远程数据但不合并
git fetch origin                                 # 下载所有分支
git fetch origin main                            # 只下载 main 分支
git fetch --prune                                # 下载并删除本地已不存在的远程分支引用

# pull — fetch + merge（或 rebase）
git pull origin main                             # 默认是 fetch + merge
git pull --rebase origin main                    # fetch + rebase（推荐）

# push — 上传本地数据
git push origin main                             # 推送 main 分支
git push origin feature/login                    # 推送特定分支
git push -u origin feature/login                 # 推送并设置上游（下次直接 git push）
git push --force-with-lease                      # 安全强制推送（推荐替代 --force）
git push origin --delete feature/login           # 删除远程分支
```

**fetch vs pull 的区别：**

```bash
# fetch = 下载但不合并（安全，推荐）
git fetch origin
git log origin/main     # 查看远程有什么新东西
git diff main origin/main   # 看看差异
git merge origin/main   # 手动合并

# pull = 下载 + 自动合并（一步到位，但可能产生意外合并）
git pull origin main

# 推荐配置：让 pull 默认使用 rebase
git config --global pull.rebase true
```

**`--force-with-lease` 为什么更安全：**

```bash
# --force：无条件覆盖远程（如果有人在你之后 push 了，会被覆盖掉）
git push --force

# --force-with-lease：只在远程和你的本地记录一致时才推送
# 如果远程有别人推送的新内容，推送会失败，保护协作安全
git push --force-with-lease
```

### 4.2 fork vs clone vs branch — 协作模式

| 方式 | 适用场景 | 权限要求 |
|------|----------|----------|
| **Branch** | 同一团队，同仓库 | Write 权限 |
| **Fork** | 开源贡献，跨组织协作 | 无需原仓库权限 |
| **Clone** | 获取代码到本地 | 公开仓库无需权限 |

**Fork 工作流：**

```bash
# 1. 在 GitHub 上点击 Fork 按钮，fork 到自己的账号
# 2. Clone 自己的 fork
git clone git@github.com:my-account/repo.git
cd repo

# 3. 添加原始仓库为 upstream
git remote add upstream git@github.com:original-owner/repo.git

# 4. 保持同步
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 5. 创建功能分支、开发、推送
git checkout -b feature/my-feature
# ... 开发 ...
git push -u origin feature/my-feature

# 6. 在 GitHub 上从自己的 fork 创建 Pull Request 到原始仓库
```

### 4.3 冲突解决实战

```bash
# 冲突发生时：
git merge feature/login
# Auto-merging src/app.py
# CONFLICT (content): Merge conflict in src/app.py
# Automatic merge failed; fix conflicts and then commit the result.

# 文件内容：
# <<<<<<< HEAD
# def login():
#     return auth_email(email, password)
# =======
# def login():
#     return auth_oauth(provider, token)
# >>>>>>> feature/login

# 解决步骤：
# 1. 手动编辑文件，删除标记，保留正确代码
# 2. git add src/app.py
# 3. git commit（不需要 -m，Git 会生成 merge message）

# 方便的工具：
git diff --name-only --diff-filter=U    # 只列出冲突文件
git checkout --ours src/app.py           # 全选当前分支版本
git checkout --theirs src/app.py         # 全选要合并的版本
git mergetool                            # 打开可视化合并工具
```

---

## 五、GitHub 平台深度使用

### 5.1 Issues — 任务追踪

Issue 不仅是 Bug 报告，也是功能讨论、任务管理和知识库。

**Issue 模板（`.github/ISSUE_TEMPLATE/bug_report.md`）：**

```markdown
---
name: Bug Report
about: 报告一个 Bug
title: "[Bug] "
labels: bug
---

## 描述 (Description)

清晰描述 Bug 的表现。

## 复现步骤 (Steps to Reproduce)

1. 执行 '...'
2. 点击 '...'
3. 看到错误 '...'

## 期望行为 (Expected Behavior)

应该发生什么。

## 环境 (Environment)

- OS: macOS 14.0
- Browser: Chrome 120
- Version: v2.1.0
```

**Issue 高级用法：**

```markdown
# 引用 PR / commit
Closes #123          # PR 合并时自动关闭 Issue
Fixes #456           # 同上
Related to #789      # 不会自动关闭

# 任务列表
- [x] 添加登录接口
- [ ] 编写单元测试
- [ ] 更新 API 文档

# @ 提及分配
@sunailian 请 review 这个设计
```

### 5.2 Pull Requests — 代码审查流

**PR 的最佳实践：**

```markdown
## 概述 (Summary)

实现用户 OAuth 登录功能，支持 Google 和 GitHub 两种方式。

## 改动内容 (Changes)

- 新增 `OAuthService` 处理 OAuth 2.0 流程
- 新增 `config/oauth.yaml` 配置文件
- 更新 `User` 模型支持 OAuth provider 字段

## 测试 (Testing)

- [x] 单元测试：OAuthService 核心逻辑
- [x] 集成测试：Google / GitHub 登录全流程
- [x] 手动测试：开发环境完整验证

## 截图 (Screenshots)

![Login Page](https://user-images.githubusercontent.com/...)

## 关联 Issue (Related Issues)

Closes #42
```

**Code Review 命令（评论中可用）：**

```markdown
/suggest    # 提出改进建议（GitHub 会生成一个 suggestion block）
/lgtm       # Looks Good To Me
/approve    # 批准
```

### 5.3 Actions / CI — 自动化流水线

**基础工作流（`.github/workflows/ci.yml`）：**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
```

**常用的 Actions 操作：**

```yaml
# 缓存依赖
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# 发布到 GitHub Pages
- uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./dist

# Docker 构建并推送
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
```

**Secrets 管理：**

```bash
# 在 GitHub 仓库 Settings → Secrets and variables → Actions 中设置
# 然后在工作流中引用：
${{ secrets.DEPLOY_KEY }}
${{ secrets.NPM_TOKEN }}
```

### 5.4 Projects — 项目管理

GitHub Projects 是一个看板式的项目管理工具，可以跨仓库管理 Issue 和 PR。

**关键概念：**
- **视图（View）：** Table / Board / Roadmap 三种展示方式
- **字段（Field）：** 自定义状态、优先级、估算等
- **自动化：** Issue 关闭时自动移动卡片

**推荐配置：**

| Status | 含义 |
|--------|------|
| 📋 Backlog | 待办，未排期 |
| 🏗 In Progress | 开发中 |
| 👀 In Review | 代码审查中 |
| ✅ Done | 已完成 |

### 5.5 Releases — 版本发布

```bash
# 通过 gh CLI 创建 Release
gh release create v1.0.0 --title "v1.0.0" --generate-notes

# 带附件
gh release create v1.0.0 ./dist/binary --title "v1.0.0" --notes "Release notes"

# 草稿（先保存不发布）
gh release create v2.0.0-rc1 --draft --prerelease --generate-notes
```

**语义化版本（SemVer）：**

```
v2.1.3
│ │ │
│ │ └── PATCH：Bug 修复（向下兼容）
│ └──── MINOR：新功能（向下兼容）
└────── MAJOR：破坏性变更（不兼容）
```

### 5.6 安全：分支保护

在仓库 Settings → Branches → Add branch protection rule：

```json
{
  "branch": "main",
  "required_approving_review_count": 1,
  "dismiss_stale_reviews": true,
  "require_code_owner_reviews": true,
  "required_status_checks": ["ci/test", "ci/lint"],
  "require_conversation_resolution": true,
  "allow_force_pushes": false,
  "allow_deletions": false
}
```

---

## 六、GitHub CLI（gh）实战

### 6.1 安装与登录

```bash
# macOS
brew install gh

# 登录（支持浏览器和 token 两种方式）
gh auth login

# 验证
gh auth status
```

### 6.2 常用命令速查

```bash
# ==== Issues ====
gh issue list                              # 查看 Issue 列表
gh issue view 42                           # 查看详情
gh issue create --title "Bug: login fail" --body "Description"  # 创建
gh issue close 42                          # 关闭
gh issue reopen 42                         # 重新打开

# ==== PR ====
gh pr list                                 # 查看 PR 列表
gh pr view 123                             # 查看详情
gh pr create --title "feat: OAuth login" --body "..."  # 创建 PR
gh pr checkout 123                         # 检出 PR 到本地
gh pr review 123 --approve                 # 批准
gh pr review 123 --comment -b "LGTM!"     # 评论
gh pr merge 123 --squash                   # Squash 合并
gh pr merge 123 --rebase                   # Rebase 合并
gh pr merge 123 --merge                    # 普通 Merge

# ==== Repo ====
gh repo clone owner/repo                   # 克隆
gh repo create my-project --public --clone # 创建并克隆
gh repo fork owner/repo --clone            # Fork 并克隆
gh repo view                               # 查看仓库信息

# ==== Actions ====
gh run list                                # 查看运行记录
gh run watch                               # 实时查看运行日志
gh workflow run ci.yml                     # 触发工作流

# ==== Gist ====
gh gist create script.py --public          # 创建 Gist
gh gist list                               # 查看 Gist 列表
```

---

## 七、工作流实战

### 7.1 Git Flow

适合有固定发布周期的项目。

```
main     ──●────────────●────────────●──
            \          / \          /
develop  ●──●──●──●──●───●──●──●──●──
              \    /
feature   ●──●──●──●
                   \
release           ●──●──●
                         \
hotfix                    ●──●
```

**分支类型：**
- `main` — 生产代码
- `develop` — 开发主线
- `feature/*` — 新功能
- `release/*` — 发布准备
- `hotfix/*` — 紧急修复

### 7.2 GitHub Flow

GitHub 官方推荐，简单灵活。

```
main ──●────●────●────●────
        \   /    /    /
         ●─●    /    /
          feature1  /
               ●───●
                feature2
```

**规则：**
1. `main` 分支始终可部署
2. 从 `main` 创建描述性分支
3. 持续 push 到远程
4. 开 Pull Request 讨论
5. 合并后立即部署

### 7.3 Trunk-Based Development

适合持续交付和高频部署的团队。

```
main ──●─●─●─●─●─●─●──  ← 每天多次合并
        \/ / / / / /
         ● ● ● ● ●       ← 短生命周期分支（<1天）
```

**核心实践：**
- 直接在 `main` 上开发，或使用极短分支（<24 小时）
- 通过 feature flag 控制未完成功能的可见性
- 频繁集成，小步提交
- 需要高度成熟的 CI/CD 和测试

### 7.4 如何选型

| 条件 | 推荐工作流 |
|------|-----------|
| 有固定发布周期（如每两周） | Git Flow |
| 小型团队，持续部署 | GitHub Flow |
| 成熟 CI/CD，高频部署 | Trunk-Based |
| 开源项目，外部贡献者 | GitHub Flow + Fork |
| 企业内部项目 | Git Flow 或 GitHub Flow |

---

## 八、高级话题

### 8.1 `.gitignore` 与 `.gitattributes`

```gitignore
# .gitignore — 告诉 Git 哪些文件不追踪
node_modules/
*.log
.env
dist/
.DS_Store
*.pyc
__pycache__/
.vscode/

# 例外：! 开头的规则会重新包含
!important.log          # 追踪这个特定的 log 文件

# 全局 gitignore
git config --global core.excludesfile ~/.gitignore_global
```

```gitattributes
# .gitattributes — 控制 Git 如何处理文件
# 统一换行符
* text=auto

# 二进制文件不尝试 diff
*.png binary
*.jpg binary

# 大文件用 LFS 管理
*.psd filter=lfs diff=lfs merge=lfs -text
```

### 8.2 submodule / subtree

```bash
# submodule — 引用外部仓库的一个特定 commit
git submodule add git@github.com:user/lib.git libs/mylib
git submodule update --init --recursive   # 克隆后初始化子模块

# 子模块更新
cd libs/mylib
git pull origin main
cd ../..
git add libs/mylib
git commit -m "Update submodule"

# ⚠️ submodule 的痛点：需要额外命令、容易忘记更新、CI 配置复杂
# 替代方案：用 subtree 将外部代码直接纳入仓库
git subtree add --prefix=libs/mylib git@github.com:user/lib.git main --squash
```

### 8.3 reflog — 后悔药

```bash
# reflog 记录了 HEAD 的所有移动
git reflog

# 输出示例：
# abc1234 HEAD@{0}: commit: feat: add login
# def5678 HEAD@{1}: commit: fix: typo
# ghi9012 HEAD@{2}: reset: moving to HEAD~1

# 恢复到"刚才"的状态（即使做了 hard reset）
git reset --hard HEAD@{1}

# 恢复"丢失"的 commit
git checkout -b recovered-branch abc1234

# reflog 默认保留 90 天（可达对象）或 30 天（不可达对象）
```

### 8.4 bisect — 二分排错

```bash
# 已知：v1.0 工作正常，当前版本有 Bug
# 但不知道是哪个 commit 引入的

git bisect start
git bisect bad HEAD              # 标记当前版本有问题
git bisect good v1.0.0           # 标记 v1.0.0 是好的

# Git 自动 checkout 到中间某个 commit
# 测试 → 告诉 Git 结果
git bisect good                  # 或 git bisect bad

# 重复直到找到元凶
# abc1234 is the first bad commit

# 完成后退出 bisect 模式
git bisect reset
```

**原理：** 假设有 100 个可疑 commit，二分查找最多需要 7 次测试（log₂(100) ≈ 7），而不是线性遍历 100 次。

### 8.5 hooks — 自动化钩子

Git hooks 是存放在 `.git/hooks/` 下的脚本，在特定事件触发时自动执行。

```bash
# 常用 hooks：
# pre-commit  — commit 前运行（检查代码风格、运行 linter）
# commit-msg  — 验证 commit message 格式
# pre-push    — push 前运行（运行测试）
# post-merge  — merge 后运行（更新依赖）

# pre-commit 示例（检查 Python 代码风格）
#!/bin/bash
# .git/hooks/pre-commit
python3 -m ruff check .
if [ $? -ne 0 ]; then
    echo "代码风格检查失败，请修复后再 commit"
    exit 1
fi
```

**推荐使用 pre-commit 框架：**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.1.0
    hooks:
      - id: black
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
```

### 8.6 大文件处理（Git LFS）

```bash
# 安装 Git LFS
brew install git-lfs
git lfs install

# 追踪大文件类型
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.mp4"

# 正常使用
git add file.psd
git commit -m "Add design file"
git push

# Git LFS 的原理：
# 1. 仓库中只存一个指针文件（几百字节）
# 2. 实际大文件存在 LFS 服务器上
# 3. git clone 时下载指针，git checkout 时按需下载大文件
```

---

## 九、踩坑集锦与最佳实践

### 9.1 常见错误与修复

**1. `detached HEAD` 状态**

```bash
# 症状：checkout 到某个 commit 而不是分支
# 原因：直接 checkout 了一个 commit hash 或 tag
git checkout abc1234

# 修复：创建新分支
git checkout -b temp-branch
```

**2. 不小心 `add` 了不该 `add` 的文件**

```bash
# 取消暂存
git restore --staged wrong-file.txt
# 或者
git reset HEAD wrong-file.txt
```

**3. `commit` 错了分支**

```bash
git log --oneline -1           # 记下 hash
git checkout correct-branch
git cherry-pick <hash>         # 摘过去
git checkout wrong-branch
git reset --hard HEAD~1         # 删掉错的
```

**4. `push` 被拒绝（non-fast-forward）**

```bash
# 原因：远程有你的本地没有的 commit
# 不要直接 force push！
git fetch origin
git rebase origin/main          # 或者 merge
git push origin main
```

**5. 大文件已 `commit`，即使之后删除，仓库仍然很大**

```bash
# 从历史中彻底删除大文件
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/huge-file.zip" \
  --prune-empty --tag-name-filter cat -- --all

# 或者使用 BFG Repo-Cleaner（更快）
# brew install bfg
# bfg --delete-files huge-file.zip
```

### 9.2 Commit Message 规范

**Conventional Commits（推荐）：**

```
<type>(<scope>): <subject>

<body>

<footer>
```

**类型（type）：**

| Type | 含义 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档更新 |
| `style` | 代码格式（不影响逻辑） |
| `refactor` | 重构（既非新功能也非修复） |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `chore` | 构建/工具/依赖更新 |
| `ci` | CI 配置变更 |

**示例：**

```
feat(login): add OAuth 2.0 support for Google

Implemented Google OAuth 2.0 flow with PKCE extension.
Configuration via GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET env vars.

Closes #42
BREAKING CHANGE: Removed deprecated password auth endpoint
```

### 9.3 团队协作建议

**1. 保持 PR 小而专注**

```
好：一个 PR = 一个功能 / 一个修复
坏：一个 PR = 登录功能 + 首页改版 + 数据库迁移
```

小 PR 审查快、冲突少、回滚容易。

**2. 每次 commit 应该是可工作的单元**

```
好：
abc1234 feat: add login form component
def5678 feat: add login API endpoint
ghi9012 feat: wire up login form to API
jkl0123 test: add login integration tests

坏：
abc1234 wip
def5678 fix stuff
ghi9012 more fixes
```

**3. 保护 main 分支**

```bash
# 禁止直接 push
# 要求 PR + Review
# 要求 CI 通过
# 要求线性历史（禁止 merge commit）
```

**4. 分支命名规范**

```
feature/login-oauth      # 新功能
bugfix/login-timeout     # Bug 修复
hotfix/critical-auth     # 紧急修复
release/v2.0.0           # 发布准备
chore/update-deps        # 杂务
```

### 9.4 `.gitconfig` 推荐配置

```ini
[user]
    name = Frank
    email = sunangie@126.com

[core]
    editor = vim
    autocrlf = input          # macOS/Linux 使用 LF 换行
    excludesfile = ~/.gitignore_global

[alias]
    lg = log --graph --oneline --all --decorate
    lga = log --graph --oneline --all --decorate --author-date-order
    st = status -s
    df = diff --word-diff
    br = branch
    co = checkout
    ci = commit

[pull]
    rebase = true             # pull 默认使用 rebase 而非 merge

[merge]
    conflictstyle = diff3     # 冲突时显示三方对比（更好定位）

[diff]
    colorMoved = zebra        # 高亮移动的代码块

[push]
    default = current         # 默认推送当前分支
```

---

## 附录：速查表

### 撤销操作

| 想做什么 | 命令 |
|----------|------|
| 丢弃工作区修改 | `git restore file.txt` |
| 取消暂存 | `git restore --staged file.txt` |
| 修改最后一次 commit | `git commit --amend` |
| 撤销 commit（保留改动） | `git reset --soft HEAD~1` |
| 撤销 commit（丢弃改动） | `git reset --hard HEAD~1` |
| 安全撤销已 push 的 commit | `git revert abc1234` |

### 分支操作

| 想做什么 | 命令 |
|----------|------|
| 创建并切换分支 | `git switch -c feature/x` |
| 删除本地分支 | `git branch -d feature/x` |
| 删除远程分支 | `git push origin --delete feature/x` |
| 重命名分支 | `git branch -m old new` |
| 查看所有分支 | `git branch -a` |

### 远程操作

| 想做什么 | 命令 |
|----------|------|
| 下载但不合并 | `git fetch origin` |
| 下载并合并 | `git pull origin main` |
| 下载并变基 | `git pull --rebase origin main` |
| 首次推送 | `git push -u origin main` |
| 安全强制推送 | `git push --force-with-lease` |

---

> **「对抗焦虑的最好办法，是亲手构建确定性。」** — 掌握 Git，就是掌握了代码世界的确定性。

---

## 许可

MIT License
