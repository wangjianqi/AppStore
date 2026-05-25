# Git与版本控制基础

> 🎯 **本章目标**：帮助零基础开发者理解版本控制的概念，学会安装和配置 Git，掌握日常开发中最常用的 Git 操作，并能结合 GitHub 完成代码托管与协作。读完本章后，你将能够独立使用 Git 管理项目代码，不再害怕"改坏了回不去"的问题。

---

## 什么是版本控制

### 生活中的版本控制

你可能有过这样的经历——写毕业论文时，文件名变成了这样：

```
论文初稿.docx
论文修改版.docx
论文修改版2.docx
论文修改版最终版.docx
论文修改版最终版2.docx
论文修改版最终版打死不改版.docx
论文修改版最终版打死不改版导师又改了.docx
```

这其实就是一种"原始的版本控制"——通过文件名来区分不同版本。但这种方式问题很大：

| 问题 | 说明 |
|------|------|
| 版本混乱 | 到底哪个是最终版？ |
| 无法追溯 | 三天前删了哪段内容？不知道 |
| 无法对比 | 这版和上一版改了什么？看不出来 |
| 无法协作 | 两个人同时改，怎么合并？ |
| 占用空间 | 每个版本都是完整副本 |

**版本控制系统（Version Control System, VCS）** 就是专门解决这些问题的工具。它能帮你：

- 📸 **记录每一次修改**：谁、什么时候、改了什么，一目了然
- ⏪ **随时回退**：改坏了？一键回到之前的版本
- 🔀 **并行开发**：多人在不同分支上工作，互不干扰
- 🔗 **合并协作**：把不同人的修改合并到一起

### 版本控制的发展历程

```
第一代：本地版本控制（1980s）
  只在本机记录文件变化，无法协作
  代表：RCS

第二代：集中式版本控制（1990s-2000s）
  有一个中央服务器存储所有版本，开发者从中获取代码
  代表：SVN、CVS、Perforce
  缺点：服务器挂了，所有人都无法工作

第三代：分布式版本控制（2005-至今）
  每个人都有完整的代码仓库副本，包括所有历史记录
  代表：Git、Mercurial
  优点：离线也能工作、速度更快、分支更灵活
```

### 为什么选择 Git

Git 是目前全球最流行的版本控制系统，由 Linux 之父 Linus Torvalds 在 2005 年创建。以下是选择 Git 的理由：

| 优势 | 说明 |
|------|------|
| **分布式** | 每个开发者都有完整的仓库副本，离线也能提交 |
| **速度快** | 大部分操作都在本地完成，不需要联网 |
| **分支轻量** | 创建和切换分支几乎瞬间完成 |
| **社区庞大** | GitHub 上有超过 3 亿个仓库 |
| **行业标准** | 几乎所有科技公司都在使用 |
| **Xcode 内置** | Apple 开发工具链原生支持 Git |

> 💡 **提示**：在 iOS 开发领域，Git 是唯一的版本控制标准。你不需要考虑其他选项，直接学 Git 就对了。

---

## Git 安装与初始配置

### 验证 Git 是否已安装

macOS 在安装 Xcode Command Line Tools 时会自动安装 Git。打开终端输入：

```bash
git --version
```

如果输出类似 `git version 2.39.5 (Apple Git-158)`，说明已经安装好了。

> ⚠️ **警告**：如果提示"git 不是可用命令"，说明 Xcode Command Line Tools 未安装，执行以下命令：
> ```bash
> xcode-select --install
> ```
> 弹出安装窗口后，点击"安装"即可。详见 [安装Xcode](./安装Xcode.md)。

### 安装最新版 Git（可选）

macOS 自带的 Git 版本可能较旧。如果你想使用最新版，可以通过 Homebrew 安装：

```bash
brew install git
```

安装后，重启终端，再次检查版本：

```bash
git --version
```

> 💡 **提示**：Homebrew 安装的 Git 版本通常比系统自带的更新。如果你已经安装了 Homebrew，建议使用 Homebrew 版本。

### 配置用户名和邮箱

Git 每次提交代码时，都会记录是谁提交的。这是最基本的配置，必须设置：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
```

💡 **提示**：
- 这里的名字不一定要用真名，但建议使用能让协作者认出你的名字
- 邮箱建议使用你注册 GitHub 的邮箱，这样提交记录能正确关联到你的 GitHub 账号
- `--global` 表示全局配置，对所有仓库生效。如果只想对某个仓库使用不同的名字/邮箱，可以在仓库目录下不加 `--global` 重新配置

验证配置：

```bash
git config --global --list
```

输出中应该能看到你刚设置的 `user.name` 和 `user.email`。

### 配置默认分支名

从 Git 2.28 开始，可以设置默认分支名。目前业界推荐使用 `main` 替代传统的 `master`：

```bash
git config --global init.defaultBranch main
```

> 💡 **提示**：GitHub 从 2020 年 10 月起已将默认分支名改为 `main`。设置这个配置可以保持一致，避免混淆。

### 其他推荐配置

```bash
git config --global core.editor "code --wait"
```

这条命令将 Git 的默认编辑器设为 VS Code。当你需要编写提交信息时，会自动打开 VS Code。

```bash
git config --global core.quotepath false
```

这条命令解决中文文件名显示为转义字符的问题（如 `\345\274\200\345\217\221`）。

### 生成 SSH Key

SSH Key 是一种加密认证方式，配置后你可以免密码推送代码到 GitHub / GitLab，而不需要每次都输入密码。

**生成新的 SSH Key：**

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

执行后会依次提示：

| 提示 | 建议操作 |
|------|---------|
| `Enter file in which to save the key` | 直接按 **回车**（使用默认路径 `~/.ssh/id_ed25519`） |
| `Enter passphrase` | 直接按 **回车**（不设密码），或输入一个密码增加安全性 |
| `Enter same passphrase again` | 再次按 **回车**，或重复输入密码 |

**查看公钥内容：**

```bash
cat ~/.ssh/id_ed25519.pub
```

输出类似：

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... 你的邮箱@example.com
```

把整行内容**完整复制**，后面配置 GitHub 时需要用到。

### 配置 GitHub SSH Key

1. 打开 [GitHub](https://github.com)，登录你的账号
2. 点击右上角头像 → **Settings**
3. 左侧菜单选择 **SSH and GPG keys**
4. 点击 **New SSH key**
5. Title 随便填（如"MacBook Pro"），Key 粘贴刚才复制的公钥
6. 点击 **Add SSH key**

**验证连接是否成功：**

```bash
ssh -T git@github.com
```

首次连接会提示 `Are you sure you want to continue connecting (yes/no/[fingerprint])?`，输入 `yes` 回车。如果看到：

```
Hi 你的用户名! You've successfully authenticated, but GitHub does not provide shell access.
```

说明配置成功！

> ⚠️ **警告**：如果连接失败，请检查：
> - 公钥是否完整复制（包括开头的 `ssh-ed25519` 和结尾的邮箱）
> - 是否在 GitHub 上正确添加了 SSH Key
> - 网络是否正常（国内可能需要代理）

### 配置 GitLab SSH Key（如需要）

流程与 GitHub 类似：

1. 登录 GitLab
2. 点击头像 → **Preferences**（或 **Edit Profile**）
3. 左侧选择 **SSH Keys**
4. 粘贴公钥，点击 **Add key**

验证：

```bash
ssh -T git@gitlab.com
```

---

## 核心概念

### Git 的四个区域

理解 Git 的关键在于理解它的四个区域，这是所有操作的基础：

```
工作区（Working Directory）     你实际编辑文件的地方
        │
        │ git add
        ▼
暂存区（Staging Area）          下次提交要包含的修改
        │
        │ git commit
        ▼
本地仓库（Local Repository）    保存在本地的所有提交历史
        │
        │ git push
        ▼
远程仓库（Remote Repository）   保存在 GitHub 等服务器上的仓库
```

用一个生活类比来理解：

| 区域 | 类比 | 说明 |
|------|------|------|
| 工作区 | 你的书桌 | 你正在修改文件的地方 |
| 暂存区 | 购物车 | 你挑选好准备结账的商品 |
| 本地仓库 | 你的日记本 | 已经记录下来的内容 |
| 远程仓库 | 云端同步 | 上传到云端，其他人也能看到 |

### 仓库（Repository）

仓库就是 Git 管理的项目目录，里面包含了所有文件和完整的修改历史。仓库分为两种：

| 类型 | 说明 | 创建方式 |
|------|------|---------|
| 本地仓库 | 存在你电脑上的仓库 | `git init` 或 `git clone` |
| 远程仓库 | 存在 GitHub 等服务器上的仓库 | 在 GitHub 上创建 |

每个仓库根目录下都有一个隐藏的 `.git` 文件夹，里面存储了所有的版本信息：

```bash
# 查看 .git 目录结构
ls -la .git
```

输出：

```
total 24
drwxr-xr-x   12 wjq  staff   384 May 25 10:00 .
drwxr-xr-x    8 wjq  staff   256 May 25 10:00 ..
drwxr-xr-x   13 wjq  staff   416 May 25 10:00 hooks
-rw-r--r--    1 wjq  staff    73 May 25 10:00 HEAD
drwxr-xr-x    3 wjq  staff    96 May 25 10:00 info
drwxr-xr-x    4 wjq  staff   128 May 25 10:00 objects
-rw-r--r--    1 wjq  staff   336 May 25 10:00 config
drwxr-xr-x    4 wjq  staff   128 May 25 10:00 refs
```

> ⚠️ **警告**：**绝对不要**手动修改 `.git` 目录里的内容，也不要删除它！删除 `.git` 就等于删除了整个版本历史。

### 提交（Commit）

提交是 Git 的核心概念。每次提交都是一个"快照"，记录了项目在某一时刻的完整状态。

一个提交包含以下信息：

| 信息 | 说明 | 示例 |
|------|------|------|
| 哈希值 | 提交的唯一标识 | `a1b2c3d` |
| 作者 | 谁做的提交 | `张三 <zhangsan@example.com>` |
| 日期 | 什么时候提交的 | `2026-05-25 10:30:00` |
| 提交信息 | 描述这次提交做了什么 | `feat: 添加用户登录功能` |
| 父提交 | 上一个提交的哈希值 | `e4f5g6h` |

提交信息的重要性——好的提交信息能让你快速了解项目演变历史：

```bash
# 好的提交信息
feat: 添加用户登录功能
fix: 修复登录页面密码显示错误
docs: 更新 README 安装说明
refactor: 重构网络请求模块

# 不好的提交信息
update
fix bug
修改
aaa
```

### 分支（Branch）

分支是 Git 最强大的功能之一。你可以把分支理解为"平行宇宙"——在一条分支上的修改不会影响其他分支。

```
主分支 main:    A --- B --- C --- D --- E --- M（合并）
                       \                     /
功能分支 feature:        F --- G --- H --- I
```

常见分支策略：

| 分支名 | 用途 | 生命周期 |
|--------|------|---------|
| `main` | 生产代码，随时可发布 | 永久 |
| `develop` | 开发集成分支 | 永久 |
| `feature/xxx` | 新功能开发 | 功能完成后删除 |
| `fix/xxx` | Bug 修复 | 修复完成后删除 |
| `release/xxx` | 发布准备 | 发布完成后删除 |

> 💡 **提示**：对于个人项目或小团队，使用 `main` + `feature/xxx` 的简单策略就足够了。不需要一开始就采用复杂的分支模型。

### 远程（Remote）

远程仓库是代码的"云端备份"和"协作中心"。最常见的远程仓库托管平台：

| 平台 | 特点 | 适用场景 |
|------|------|---------|
| **GitHub** | 全球最大的代码托管平台，社区活跃 | 开源项目、国际协作 |
| **GitLab** | 支持 CI/CD，可私有化部署 | 企业内部项目 |
| **Gitee（码云）** | 国内平台，访问速度快 | 国内项目、网络受限 |
| **Bitbucket** | 与 Jira/Confluence 集成 | 使用 Atlassian 工具链的团队 |

---

## 日常操作

### 初始化仓库：git init

将一个普通目录变成 Git 仓库：

```bash
# 创建项目目录
mkdir MyFirstApp
cd MyFirstApp

# 初始化 Git 仓库
git init
```

输出：

```
Initialized empty Git repository in /Users/wjq/MyFirstApp/.git/
```

此时仓库是空的，还没有任何提交。

### 克隆仓库：git clone

从远程仓库复制一份到本地：

```bash
# 通过 SSH 克隆（推荐，已配置 SSH Key）
git clone git@github.com:username/MyApp.git

# 通过 HTTPS 克隆
git clone https://github.com/username/MyApp.git

# 克隆到指定目录
git clone git@github.com:username/MyApp.git my-local-dir
```

> 💡 **提示**：推荐使用 SSH 方式克隆，配置好 SSH Key 后无需每次输入密码。如果尚未配置 SSH Key，可以暂时使用 HTTPS 方式。

### 查看状态：git status

这是最常用的命令之一，用于查看当前仓库的状态：

```bash
git status
```

输出示例：

```
On branch main
No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        AppDelegate.swift
        SceneDelegate.swift
        ViewController.swift

nothing added to commit but untracked files present (use "git add" to track)
```

文件状态说明：

| 状态 | 含义 | 标识 |
|------|------|------|
| Untracked | 新文件，Git 还没跟踪 | 红色显示 |
| Modified | 已跟踪的文件被修改了 | 红色显示 |
| Staged | 修改已添加到暂存区 | 绿色显示 |
| Committed | 已提交到本地仓库 | 不再显示 |

### 添加到暂存区：git add

将工作区的修改添加到暂存区，准备提交：

```bash
# 添加单个文件
git add AppDelegate.swift

# 添加多个文件
git add AppDelegate.swift SceneDelegate.swift

# 添加所有修改
git add .

# 添加所有 .swift 文件
git add "*.swift"

# 交互式添加（可以选择部分修改）
git add -p
```

> 💡 **提示**：`git add .` 是最常用的方式，它会添加当前目录及子目录下的所有修改。注意 `.` 代表当前目录，如果你在项目根目录执行，会添加整个项目的修改。

> ⚠️ **警告**：`git add .` 会添加所有修改，包括你可能不想提交的文件。建议先执行 `git status` 查看有哪些修改，再决定添加哪些。

### 提交：git commit

将暂存区的内容提交到本地仓库：

```bash
# 提交并写提交信息
git commit -m "feat: 添加用户登录功能"

# 提交所有已跟踪文件的修改（跳过 git add）
git commit -a -m "fix: 修复登录页面布局问题"

# 打开编辑器编写详细的提交信息
git commit
```

**提交信息规范（Conventional Commits）**：

这是业界广泛使用的提交信息格式，让提交历史更易读：

```
<类型>: <简短描述>

[可选的详细描述]

[可选的关联信息]
```

常用类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 添加用户注册功能` |
| `fix` | Bug 修复 | `fix: 修复登录密码验证错误` |
| `docs` | 文档修改 | `docs: 更新 API 文档` |
| `style` | 格式调整（不影响逻辑） | `style: 统一缩进为 4 空格` |
| `refactor` | 重构（不是新功能也不是修复） | `refactor: 重构网络请求模块` |
| `test` | 添加或修改测试 | `test: 添加登录功能单元测试` |
| `chore` | 构建过程或辅助工具变动 | `chore: 更新 SPM 依赖版本` |

> 💡 **提示**：养成写好提交信息的习惯，未来的你会感谢现在的自己。尤其是团队协作时，清晰的提交信息是高效沟通的基础。

### 推送：git push

将本地提交推送到远程仓库：

```bash
# 推送当前分支到远程
git push

# 首次推送新分支到远程（设置上游分支）
git push -u origin feature/login

# 推送所有分支
git push --all

# 推送标签
git push origin v1.0.0

# 推送所有标签
git push --tags
```

> ⚠️ **警告**：`git push --force`（强制推送）会覆盖远程仓库的历史，**除非你确切知道自己在做什么，否则不要使用**。在团队协作中，强制推送可能导致其他人的工作丢失。

### 拉取：git pull

从远程仓库获取最新代码并合并到本地：

```bash
# 拉取当前分支的最新代码
git pull

# 拉取指定分支
git pull origin main

# 拉取并使用 rebase 方式合并（推荐，保持线性历史）
git pull --rebase
```

`git pull` 实际上是两个操作的组合：

```
git pull = git fetch + git merge
```

- `git fetch`：从远程下载最新内容，但不修改工作区
- `git merge`：将下载的内容合并到当前分支

> 💡 **提示**：推荐使用 `git pull --rebase` 替代 `git pull`，这样可以避免产生不必要的合并提交，保持提交历史的线性。

### 查看历史：git log

查看提交历史记录：

```bash
# 查看完整历史
git log

# 单行显示
git log --oneline

# 图形化显示分支
git log --oneline --graph

# 显示最近 5 条
git log -5

# 查看某个文件的修改历史
git log -- AppDelegate.swift

# 查看某个作者的提交
git log --author="张三"

# 搜索提交信息
git log --grep="登录"
```

输出示例（`--oneline` 格式）：

```
a1b2c3d feat: 添加用户登录功能
e4f5g6h fix: 修复首页布局问题
i7j8k9l docs: 更新 README
m0n1o2p feat: 初始化项目
```

### 查看差异：git diff

查看文件的具体修改内容：

```bash
# 查看工作区和暂存区的差异
git diff

# 查看暂存区和上次提交的差异
git diff --staged

# 查看两个提交之间的差异
git diff a1b2c3d e4f5g6h

# 查看某个文件的差异
git diff AppDelegate.swift

# 统计修改的文件和行数
git diff --stat
```

输出示例：

```diff
diff --git a/AppDelegate.swift b/AppDelegate.swift
index a1b2c3d..e4f5g6h 100644
--- a/AppDelegate.swift
+++ b/AppDelegate.swift
@@ -10,6 +10,8 @@ class AppDelegate {

     var window: UIWindow?

+    var userManager: UserManager?
+
     func application(_ application: UIApplication,
                      didFinishLaunchingWithOptions launchOptions:
```

### 撤销操作

撤销是 Git 中容易混淆的部分，下面逐一说明：

```bash
# 撤销工作区的修改（还没 add）
git checkout -- AppDelegate.swift

# 或者使用新语法（Git 2.23+）
git restore AppDelegate.swift

# 撤销暂存区的修改（已 add，还没 commit）
git reset HEAD AppDelegate.swift

# 或者使用新语法
git restore --staged AppDelegate.swift

# 修改最近一次提交的信息
git commit --amend -m "新的提交信息"

# 回退到上一个提交（保留工作区修改）
git reset --soft HEAD~1

# 回退到上一个提交（丢弃所有修改）
git reset --hard HEAD~1
```

⚠️ **警告**：
- `git reset --hard` 会**永久丢失**未提交的修改，使用前务必确认
- `git commit --amend` 会修改最近一次提交的哈希值，如果已经 push 到远程，不要使用 amend
- 撤销操作前，建议先 `git status` 确认当前状态

### 暂存工作：git stash

当你正在开发一个功能，突然需要切换到另一个分支修 Bug，但又不想提交半成品代码时，可以使用 stash：

```bash
# 暂存当前修改
git stash

# 暂存时添加说明
git stash save "正在开发登录功能"

# 查看暂存列表
git stash list

# 恢复最近的暂存
git stash pop

# 恢复指定的暂存
git stash pop stash@{2}

# 恢复暂存但不删除暂存记录
git stash apply

# 删除暂存记录
git stash drop stash@{0}

# 清空所有暂存
git stash clear
```

> 💡 **提示**：`git stash pop` = `git stash apply` + `git stash drop`。如果你想在多个分支上应用同一份暂存，使用 `apply` 而不是 `pop`。

---

## 分支管理

### 创建分支

```bash
# 创建新分支（但不切换）
git branch feature/login

# 创建并切换到新分支（最常用）
git checkout -b feature/login

# 或者使用新语法（Git 2.23+）
git switch -c feature/login
```

### 切换分支

```bash
# 切换到已有分支
git checkout main

# 或者使用新语法
git switch main

# 切换到上一个分支
git switch -
```

> 💡 **提示**：切换分支前，确保当前分支的修改已经提交或暂存（stash），否则 Git 可能会阻止切换，或者将修改带到目标分支。

### 查看分支

```bash
# 查看本地分支
git branch

# 查看所有分支（本地 + 远程）
git branch -a

# 查看分支及最后一次提交
git branch -v

# 查看已合并到当前分支的分支
git branch --merged

# 查看未合并到当前分支的分支
git branch --no-merged
```

输出示例：

```
* main
  develop
  feature/login
  fix/homepage-bug
```

`*` 号表示当前所在的分支。

### 合并分支

当功能开发完成后，需要将分支合并回主分支：

```bash
# 先切换到目标分支
git switch main

# 合并 feature/login 分支到当前分支
git merge feature/login
```

合并方式对比：

| 方式 | 命令 | 特点 |
|------|------|------|
| **Merge** | `git merge feature/login` | 保留完整的分支历史，会产生合并提交 |
| **Rebase** | `git rebase feature/login` | 将提交重新"播放"到目标分支上，历史更线性 |
| **Squash** | `git merge --squash feature/login` | 将分支的所有提交压缩为一个提交 |

> 💡 **提示**：对于个人项目，使用 `git merge` 即可。对于团队项目，推荐使用 Squash Merge（在 GitHub PR 界面选择），保持主分支历史整洁。

### 冲突解决

当两个分支修改了同一文件的同一位置时，Git 无法自动合并，就会产生冲突。

**冲突标记：**

```
<<<<<<< HEAD
这是当前分支的内容
=======
这是要合并的分支的内容
>>>>>>> feature/login
```

**解决冲突的步骤：**

1. **查看冲突文件**

```bash
# 查看哪些文件有冲突
git status
```

2. **打开冲突文件，手动编辑**

选择保留哪部分内容，删除冲突标记（`<<<<<<<`、`=======`、`>>>>>>>`）。

3. **标记冲突已解决**

```bash
git add 冲突文件名
```

4. **完成合并**

```bash
git commit
```

**在 Xcode 中解决冲突：**

Xcode 提供了可视化的冲突解决工具：

1. 当合并产生冲突时，Xcode 会弹出冲突提示
2. 选择 **File → Source Control → Conflicts** 查看冲突列表
3. 对于每个冲突，选择"保留我的"、"保留他们的"或"手动合并"
4. 保存文件，提交合并

> ⚠️ **警告**：解决冲突时务必仔细检查，不要盲目选择一方。建议运行项目确认合并后的代码能正常工作。

### 删除分支

```bash
# 删除已合并的本地分支
git branch -d feature/login

# 强制删除分支（即使未合并）
git branch -D feature/login

# 删除远程分支
git push origin --delete feature/login
```

> 💡 **提示**：养成"用完即删"的习惯。功能分支合并后就应该删除，避免分支堆积。

---

## GitHub 使用

### 创建远程仓库

1. 登录 [GitHub](https://github.com)
2. 点击右上角 **+** → **New repository**
3. 填写仓库信息：

| 设置项 | 说明 | 建议 |
|--------|------|------|
| Repository name | 仓库名称 | 与项目名一致，如 `MyFirstApp` |
| Description | 仓库描述 | 简要说明项目用途 |
| Public / Private | 公开/私有 | 学习项目选 Public，商业项目选 Private |
| README | 初始化 README 文件 | ✅ 建议勾选 |
| .gitignore | 忽略文件配置 | 选择 Swift 或 Xcode 模板 |
| License | 开源协议 | 学习项目可选 MIT |

4. 点击 **Create repository**

### 关联本地仓库与远程仓库

如果你先在本地创建了仓库，需要关联到远程：

```bash
# 在本地仓库中添加远程仓库
git remote add origin git@github.com:username/MyFirstApp.git

# 推送代码到远程
git push -u origin main
```

如果你先在 GitHub 上创建了仓库，可以直接克隆：

```bash
git clone git@github.com:username/MyFirstApp.git
cd MyFirstApp
```

### 远程仓库管理

```bash
# 查看远程仓库
git remote -v

# 修改远程仓库地址
git remote set-url origin git@github.com:username/NewName.git

# 添加多个远程仓库
git remote add upstream git@github.com:original-owner/Repo.git

# 删除远程仓库关联
git remote remove origin
```

### Fork：参与别人的项目

Fork 是 GitHub 上的核心协作功能，让你能在别人的项目上自由修改，不影响原项目。

**Fork 工作流：**

```
1. Fork 原仓库 → 你的 GitHub 账号下会有一份副本
2. Clone 你 Fork 的仓库到本地
3. 创建功能分支进行开发
4. Push 到你的 Fork 仓库
5. 创建 Pull Request 到原仓库
6. 原仓库作者审核并合并
```

**具体操作：**

```bash
# 1. 在 GitHub 网页上点击 Fork 按钮

# 2. 克隆你 Fork 的仓库
git clone git@github.com:your-username/Repo.git
cd Repo

# 3. 添加原仓库为上游（方便同步更新）
git remote add upstream git@github.com:original-owner/Repo.git

# 4. 创建功能分支
git checkout -b feature/new-feature

# 5. 开发完成后推送
git push -u origin feature/new-feature

# 6. 在 GitHub 网页上创建 Pull Request
```

**同步原仓库的更新：**

```bash
# 获取原仓库的最新代码
git fetch upstream

# 切换到主分支
git checkout main

# 合并原仓库的更新
git merge upstream/main

# 推送到你的 Fork
git push origin main
```

### Pull Request 流程

Pull Request（简称 PR）是 GitHub 上代码审查和合并的核心机制。

**创建 PR：**

1. 在 GitHub 仓库页面点击 **Pull requests** 标签
2. 点击 **New pull request**
3. 选择 base 分支（目标分支）和 compare 分支（源分支）
4. 点击 **Create pull request**
5. 填写 PR 标题和描述：

```markdown
## 变更说明
添加了用户登录功能，支持邮箱和手机号登录。

## 变更类型
- [x] 新功能
- [ ] Bug 修复
- [ ] 重构

## 测试
- [x] 单元测试通过
- [x] 真机测试通过
- [x] 模拟器测试通过

## 截图
（如有 UI 变更，附上截图）
```

6. 点击 **Create pull request**

**使用 GitHub CLI 创建 PR（更高效）：**

```bash
# 安装 GitHub CLI
brew install gh

# 登录
gh auth login

# 创建 PR
gh pr create --title "feat: 添加用户登录功能" --body "实现了邮箱和手机号登录"

# 查看PR列表
gh pr list

# 检出某个PR到本地
gh pr checkout 123

# 合并PR
gh pr merge 123
```

---

## .gitignore 配置

### 什么是 .gitignore

`.gitignore` 文件告诉 Git 哪些文件不需要跟踪。有些文件不应该提交到仓库中：

| 类型 | 示例 | 原因 |
|------|------|------|
| 编译产物 | `build/`、`DerivedData/` | 可以重新生成，且体积大 |
| 用户配置 | `*.xcuserdata`、`UserInterfaceState.xcuserstate` | 包含个人偏好，因人而异 |
| 敏感信息 | `APIKeys.swift`、`.env` | 包含密钥和密码 |
| 系统文件 | `.DS_Store`、`Thumbs.db` | macOS/Windows 自动生成的文件 |
| 依赖目录 | `Pods/`、`node_modules/` | 可以通过包管理器重新安装 |

### Xcode 项目专用 .gitignore

在项目根目录创建 `.gitignore` 文件，内容如下：

```gitignore
# Xcode
#
# gitignore contributors: remember to update Global/Xcode.gitignore, Objective-C.gitignore & Swift.gitignore

## User settings
xcuserdata/

## compatibility with Xcode 8 and earlier (ignoring not required starting Xcode 9)
*.xcscmblueprint
*.xccheckout

## compatibility with Xcode 3 and earlier (ignoring not required starting Xcode 4)
build/
DerivedData/
*.moved-aside
*.pbxuser
!default.pbxuser
*.mode1v3
!default.mode1v3
*.mode2v3
!default.mode2v3
*.perspectivev3
!default.perspectivev3

## Obj-C/Swift specific
*.hmap

## App packaging
*.ipa
*.dSYM.zip
*.dSYM

## Playgrounds
timeline.xctimeline
playground.xcworkspace

# Swift Package Manager
#
# Add this line if you want to avoid checking in source code from Swift Package Manager dependencies.
# Packages/
# Package.pins
# Package.resolved
# *.xcodeproj
#
# Xcode automatically generates this directory with a .xcworkspacedata file and xcuserdata
# hence it is not needed unless you have added a package configuration file to your project
# .swiftpm

.build/

# CocoaPods
#
# We recommend against adding the Pods directory to your .gitignore. However
# you should judge for yourself, the pros and cons are mentioned at:
# https://guides.cocoapods.org/using/using-cocoapods.html#should-i-check-the-pods-directory-into-source-control
#
# Pods/
#
# Add this line if you want to avoid checking in source code from the Xcode workspace
# *.xcworkspace

# Carthage
#
# Add this line if you want to avoid checking in source code from Carthage dependencies.
# Carthage/Checkouts

Carthage/Build/

# Accio dependency management
Dependencies/
.accio/

# fastlane
#
# It is recommended to not store the screenshots in the git repo.
# Instead, use fastlane to re-generate the screenshots whenever they are needed.
# For more information about the recommended setup visit:
# https://docs.fastlane.tools/best-practices/source-control/#source-control

fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots/**/*.png
fastlane/test_output

# Code Injection
#
# After new code Injection tools there's a generated folder /iOSInjectionProject
# https://github.com/johnno1962/injectionforxcode

iOSInjectionProject/

# macOS
.DS_Store
.AppleDouble
.LSOverride

# Thumbnails
._*

# Files that might appear in the root of a volume
.DocumentRevisions-V100
.fseventsd
.Spotlight-V100
.TemporaryItems
.Trashes
.VolumeIcon.icns
.com.apple.timemachine.donotpresent

# Directories potentially created on remote AFP share
.AppleDB
.AppleDesktop
Network Trash Folder
Temporary Items
.apdisk

# Sensitive files - 绝不要提交以下文件
*.env
*.env.local
APIKeys.swift
Secrets.swift
GoogleService-Info.plist
```

> ⚠️ **警告**：包含 API Key、密码、证书等敏感信息的文件**绝不能**提交到 Git 仓库。即使你后来删除了这些文件并提交，它们仍然存在于 Git 历史中。如果已经误提交了敏感信息，请参考后面的"常见问题"部分。

### 创建 .gitignore 的快捷方式

GitHub 提供了常用项目的 .gitignore 模板：

1. 在创建仓库时选择 Swift 或 Objective-C 模板
2. 或访问 [github.com/github/gitignore](https://github.com/github/gitignore) 获取模板

也可以使用 `gitignore.io` 服务：

```bash
# 安装 gi 命令
echo "function gi() { curl -sLw \"\\n\" https://www.toptal.com/developers/gitignore/api/\$@ ;}" >> ~/.zshrc
source ~/.zshrc

# 生成 Xcode + Swift + macOS 的 .gitignore
gi xcode,swift,macos > .gitignore
```

---

## 结合 AI 工具使用 Git

### Claude Code 的 Git 集成

Claude Code 是一个终端 AI 工具，它内置了 Git 支持，可以帮你更高效地使用 Git：

**自动生成提交信息：**

```bash
# Claude Code 会分析你的修改，自动生成规范的提交信息
claude "帮我提交当前的修改"
```

Claude Code 会执行 `git diff` 分析修改内容，然后生成类似这样的提交：

```
feat: 添加用户注册页面

- 创建 RegisterViewController
- 添加表单验证逻辑
- 集成网络请求接口
```

**智能代码审查：**

```bash
# 让 Claude Code 审查你的修改
claude "审查我当前的代码修改，指出潜在问题"
```

**解决冲突辅助：**

```bash
# 让 Claude Code 帮你理解和解决冲突
claude "帮我解决当前的 Git 合并冲突"
```

**批量操作：**

```bash
# 创建分支、修改代码、提交、推送，一步到位
claude "创建一个 feature/settings 分支，添加设置页面，然后提交并推送"
```

### Cursor / Trae 中的 Git 操作

AI IDE 也提供了便捷的 Git 操作：

| 操作 | Cursor | Trae |
|------|--------|------|
| 查看修改 | 左侧 Source Control 面板 | 左侧源代码管理面板 |
| 提交 | 面板中输入信息 → ✅ | 面板中输入信息 → 提交 |
| AI 生成提交信息 | 点击 ✨ 图标 | 点击 AI 生成按钮 |
| 查看历史 | Source Control → History | 源代码管理 → 历史记录 |
| 分支管理 | 左下角分支名 → 创建/切换 | 左下角分支管理 |

### GitHub Copilot 与 Git

GitHub Copilot Chat 可以回答 Git 相关问题：

```
你: 如何撤销上一次提交？
Copilot: 你可以使用 git reset --soft HEAD~1 撤销上一次提交，
但保留修改在暂存区。如果你想完全丢弃修改，使用 git reset --hard HEAD~1。
注意：如果已经推送到远程，建议使用 git revert 代替。
```

### AI 辅助 Git 工作流最佳实践

| 场景 | AI 工具 | 操作 |
|------|---------|------|
| 生成提交信息 | Claude Code / Cursor | 分析 diff 自动生成规范提交信息 |
| 代码审查 | Claude Code | 审查修改，指出问题和改进建议 |
| 解决冲突 | Claude Code / ChatGPT | 分析冲突内容，给出合并建议 |
| 编写 .gitignore | ChatGPT / Claude | 描述项目类型，生成对应的 .gitignore |
| Git 命令查询 | Copilot Chat | 用自然语言询问 Git 操作方法 |
| PR 描述生成 | Claude Code / gh | 分析修改内容，生成 PR 描述 |

> 💡 **提示**：AI 工具可以帮你更高效地使用 Git，但你需要理解每个 Git 操作的含义。不要盲目执行 AI 生成的 Git 命令，尤其是 `reset`、`push --force` 等危险操作。

---

## 常见问题与排错

### 问题 1：提交了敏感信息

**症状**：不小心把 API Key 或密码提交到了仓库。

**解决方法**：

```bash
# 方法一：从历史中彻底删除文件（危险操作，会重写历史）
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch 敏感文件路径" \
  --prune-empty --tag-name-filter cat -- --all

# 方法二：使用 BFG Repo-Cleaner（更安全高效）
brew install bfg
bfg --delete-files 敏感文件名
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 强制推送更新远程仓库
git push origin --force --all
```

> ⚠️ **警告**：如果敏感信息已经推送到公开仓库，即使从 Git 历史中删除，也可能已经被他人获取。**必须立即更换泄露的密钥和密码！**

### 问题 2：push 被拒绝

**症状**：`git push` 时提示 `Updates were rejected because the remote contains work that you do not have locally.`

**原因**：远程仓库有你本地没有的新提交。

**解决方法**：

```bash
# 先拉取远程更新，再推送
git pull --rebase origin main
git push origin main
```

### 问题 3：合并冲突不知道怎么解决

**症状**：`git merge` 后出现 `CONFLICT` 提示。

**解决方法**：

```bash
# 查看冲突文件
git status

# 如果想放弃合并
git merge --abort

# 如果想使用某一方的版本
git checkout --theirs 冲突文件    # 使用对方分支的版本
git checkout --ours 冲突文件      # 使用当前分支的版本
```

### 问题 4：误用了 git reset --hard

**症状**：执行 `git reset --hard` 后发现丢失了重要的修改。

**解决方法**：

```bash
# 查看所有操作记录（包括已删除的提交）
git reflog

# 找到想恢复的提交哈希，然后恢复
git reset --hard a1b2c3d
```

> 💡 **提示**：`git reflog` 是 Git 的"后悔药"，它记录了所有 HEAD 的移动历史。即使你用 `reset --hard` 删除了提交，只要还没被垃圾回收（默认 30 天），都可以通过 reflog 恢复。

### 问题 5：中文文件名乱码

**症状**：`git status` 中中文文件名显示为 `\345\274\200\345\217\221`。

**解决方法**：

```bash
git config --global core.quotepath false
```

### 问题 6：提交者信息错误

**症状**：提交记录显示的用户名或邮箱不是你想要的。

**解决方法**：

```bash
# 修改最近一次提交的作者信息
git commit --amend --author="正确的名字 <正确的邮箱@example.com>"

# 修改全局配置
git config --global user.name "正确的名字"
git config --global user.email "正确的邮箱@example.com"
```

### 问题 7：仓库体积过大

**症状**：仓库占用空间很大，clone 很慢。

**解决方法**：

```bash
# 查看仓库中哪些文件/目录占用空间最大
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print substr($0,6)}' | \
  sort -n -k2 | \
  tail -20

# 浅克隆（只获取最近的提交历史）
git clone --depth 1 git@github.com:username/Repo.git

# 单分支克隆
git clone --single-branch --branch main git@github.com:username/Repo.git
```

### 问题 8：SSH 连接超时

**症状**：`ssh -T git@github.com` 超时或连接失败。

**解决方法**：

```bash
# 测试 SSH 连接（显示调试信息）
ssh -vT git@github.com

# 如果是国内网络问题，配置 SSH 代理
# 在 ~/.ssh/config 中添加
Host github.com
  Hostname github.com
  User git
  ProxyCommand nc -X 5 -x 127.0.0.1:7890 %h %p
```

### 问题 9：Xcode 项目文件冲突

**症状**：`.pbxproj` 文件合并冲突，Xcode 无法打开项目。

**解决方法**：

```bash
# 方法一：使用 Xcode 打开项目，Xcode 会提示冲突并帮助解决

# 方法二：如果冲突简单，手动编辑 .pbxproj 文件
# 注意：这个文件结构复杂，手动编辑需谨慎

# 方法三：如果无法解决，选择一方的版本，然后在 Xcode 中重新添加文件
git checkout --ours ProjectName.xcodeproj/project.pbxproj
# 然后在 Xcode 中重新添加缺失的文件引用
```

> 💡 **提示**：减少 `.pbxproj` 冲突的最佳实践是——团队成员避免同时修改项目结构（添加/删除文件）。使用 SPM 管理依赖也可以减少这类冲突。

---

## Git 命令速查表

### 基础操作

| 命令 | 说明 |
|------|------|
| `git init` | 初始化仓库 |
| `git clone <url>` | 克隆远程仓库 |
| `git status` | 查看当前状态 |
| `git add .` | 添加所有修改到暂存区 |
| `git add <file>` | 添加指定文件到暂存区 |
| `git commit -m "msg"` | 提交暂存区的修改 |
| `git push` | 推送到远程仓库 |
| `git pull` | 拉取远程最新代码 |
| `git pull --rebase` | 拉取并 rebase 合并 |
| `git fetch` | 获取远程更新（不合并） |
| `git log` | 查看提交历史 |
| `git log --oneline` | 单行查看历史 |
| `git diff` | 查看修改内容 |
| `git stash` | 暂存当前修改 |
| `git stash pop` | 恢复暂存的修改 |

### 分支操作

| 命令 | 说明 |
|------|------|
| `git branch` | 查看本地分支 |
| `git branch -a` | 查看所有分支 |
| `git branch <name>` | 创建新分支 |
| `git checkout -b <name>` | 创建并切换分支 |
| `git switch -c <name>` | 创建并切换分支（新语法） |
| `git checkout <name>` | 切换分支 |
| `git switch <name>` | 切换分支（新语法） |
| `git merge <name>` | 合并分支 |
| `git branch -d <name>` | 删除已合并分支 |
| `git branch -D <name>` | 强制删除分支 |

### 撤销操作

| 命令 | 说明 |
|------|------|
| `git restore <file>` | 撤销工作区修改 |
| `git restore --staged <file>` | 撤销暂存区修改 |
| `git reset --soft HEAD~1` | 回退提交，保留修改 |
| `git reset --hard HEAD~1` | 回退提交，丢弃修改 |
| `git commit --amend` | 修改最近一次提交 |
| `git revert <hash>` | 创建新提交来撤销指定提交 |
| `git reflog` | 查看操作历史（后悔药） |

### 远程操作

| 命令 | 说明 |
|------|------|
| `git remote -v` | 查看远程仓库 |
| `git remote add origin <url>` | 添加远程仓库 |
| `git remote set-url origin <url>` | 修改远程仓库地址 |
| `git push -u origin <branch>` | 首次推送并设置上游 |
| `git push origin --delete <branch>` | 删除远程分支 |

---

## Git 工作流推荐

### 个人项目工作流

对于个人项目或学习项目，使用简单的工作流即可：

```
1. 在 main 分支上开发
2. 需要加新功能时，创建 feature 分支
3. 功能完成后，合并回 main
4. 定期 push 到 GitHub 备份
```

```bash
# 日常工作流
git switch main
git pull
git switch -c feature/new-feature
# ... 编写代码 ...
git add .
git commit -m "feat: 新功能描述"
git switch main
git merge feature/new-feature
git push
git branch -d feature/new-feature
```

### 团队项目工作流

团队项目推荐使用 GitHub Flow：

```
1. main 分支始终可部署
2. 新功能从 main 创建 feature 分支
3. 在 feature 分支上开发，定期 push
4. 开发完成后创建 Pull Request
5. 团队成员 Code Review
6. Review 通过后合并到 main
7. 删除 feature 分支
```

### iOS 项目 Git 使用建议

| 建议 | 说明 |
|------|------|
| 不要提交 `DerivedData` | 编译产物，可重新生成 |
| 不要提交 `xcuserdata` | 个人偏好设置 |
| 不要提交 `.DS_Store` | macOS 自动生成 |
| 不要提交 API Key | 使用环境变量或配置文件 |
| 使用 `.gitignore` | 参考前面的模板 |
| 提交前编译 | 确保代码能编译通过 |
| 频繁提交 | 小步提交比大步提交更安全 |
| 写好提交信息 | 方便回溯和协作 |

---

## 小结

本章我们系统学习了 Git 版本控制的基础知识：

| 主题 | 要点 |
|------|------|
| **版本控制概念** | Git 是分布式版本控制系统，能记录修改、随时回退、并行开发 |
| **安装配置** | 安装 Git、配置用户名邮箱、生成 SSH Key |
| **核心概念** | 工作区、暂存区、本地仓库、远程仓库四个区域 |
| **日常操作** | init、clone、add、commit、push、pull、status、log |
| **分支管理** | 创建、切换、合并分支，解决冲突 |
| **GitHub** | 创建仓库、Fork、Pull Request 协作流程 |
| **.gitignore** | 配置忽略文件，避免提交编译产物和敏感信息 |
| **AI + Git** | 利用 Claude Code 等 AI 工具提升 Git 使用效率 |

✅ 到此为止，你已经掌握了 Git 的基本使用，可以开始用 Git 管理你的 iOS 项目了！

← [开发工具链配置](./开发工具链配置.md) | [终端基础操作](./终端基础操作.md) →
