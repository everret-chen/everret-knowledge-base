# Git 配置与日常命令手册

> 适用对象：**everret-knowledge-base**（CTF 知识库 / payload 库 / 比赛复盘）
> 实测环境：`git version 2.45.1.windows.1` + Windows PowerShell 7 + Git Credential Manager
> 仓库地址：`https://github.com/everret-chen/everret-knowledge-base.git`
>
> **约定**：命令块都可以直接复制粘贴执行。标 `[PS]` 的只在 PowerShell 里有效，标 `[仅首次]` 的做一次就不用再管。

---

## 目录

- [0. 一分钟上手](#0-一分钟上手)
- [1. 安装与自检](#1-安装与自检)
- [2. 一次性配置（全局）](#2-一次性配置全局)
- [3. 认证：HTTPS 与 SSH 两套方案](#3-认证https-与-ssh-两套方案)
- [4. 接入远程仓库（三种场景）](#4-接入远程仓库三种场景)
- [5. 日常命令](#5-日常命令)
- [6. 撤销与后悔药](#6-撤销与后悔药)
- [7. 本仓库工作流](#7-本仓库工作流)
- [8. 排错速查](#8-排错速查)
- [9. 进阶但不复杂](#9-进阶但不复杂)
- [10. 命令速查表](#10-命令速查表)
- [附录 A. 本机实测的两个坑](#附录-a-本机实测的两个坑)
- [附录 B. 配置文件放在哪](#附录-b-配置文件放在哪)

---

## 0. 一分钟上手

**每天开工**（同步最新内容）：

```bash
git pull --rebase
```

**写完一篇笔记 / 一个 payload**（三连）：

```bash
git status                 # 先看改了哪些文件
git add -A                 # 全部加入暂存区
git commit -m "feat(web/sqli): 补充报错注入常用函数"
```

**收工**（推上去备份）：

```bash
git push
```

**就这 3 组命令，能覆盖你 90% 的使用场景。** 后面章节是把这三组命令拆开讲清楚，以及出问题时怎么办。

---

## 1. 安装与自检

### 1.1 检查是否已安装

```bash
git --version
git config --list --show-origin   # 看现有配置都写在哪
```

本机实际输出：`git version 2.45.1.windows.1`，安装在 **`D:\software\github\Git`**（不是默认的 `C:\Program Files\Git`）。这会影响查找配置文件的位置，见 [附录 B](#附录-b-配置文件放在哪)。

### 1.2 没装的话

```powershell
winget install --id Git.Git -e --source winget
```

装完**重开终端**再执行 `git --version`。Git for Windows 自带三样东西，都很重要：

| 组件 | 作用 | 检查方式 |
| --- | --- | --- |
| Git Bash | 提供 `git` 命令 | `git --version` |
| Git Credential Manager (GCM) | HTTPS 免密登录，第一次推送会弹浏览器授权 | `git config --global credential.helper` 应输出 `manager` |
| OpenSSH | 用 SSH 方式连接 GitHub | `ssh -V` |

### 1.3 装上之后第一件事：配身份

不配身份，第一次 `git commit` 会直接拒绝你：

```bash
git config --global user.name "everret-chen"
git config --global user.email "2710895214@qq.com"
git config --global --list | findstr user
```

> **提交邮箱的意义**：GitHub 靠邮箱把你提交的 commit 算进你的贡献图。
> - 想让贡献图正常显示 → 用 GitHub 账号绑定的邮箱（本机就是 `2710895214@qq.com`）。
> - 不想公开真实邮箱 → 用 GitHub 给的隐私地址 `<用户名>@users.noreply.github.com`，在 GitHub 的 Settings → Emails 里能看到自己专属的那个。

---

## 2. 一次性配置（全局）

### 2.1 推荐配置块（复制一次执行完）

`[仅首次]` 这段做一遍，之后所有仓库都受益：

```bash
# ── 身份 ──
git config --global user.name "everret-chen"
git config --global user.email "2710895214@qq.com"

# ── 默认分支名：新建仓库默认用 main（GitHub 现在的主分支名）──
git config --global init.defaultBranch main

# ── 中文与编码：不加这几项，中文文件名会显示成 \346\226\207 这种转义 ──
git config --global core.quotepath false
git config --global i18n.commitEncoding utf-8
git config --global i18n.logOutputEncoding utf-8

# ── 换行符：Windows 上检出为 CRLF，提交进仓库一律 LF ──
git config --global core.autocrlf true

# ── 明确 pull 行为，消除每次 pull 的警告 ──
git config --global pull.rebase false

# ── 只推当前分支到同名远程分支（现代默认值，写出来更直观）──
git config --global push.default simple

# ── HTTPS 免密：用 Git Credential Manager 记住凭据 ──
git config --global credential.helper manager

# ── Windows 上文件权限位无意义，关掉免得平白出现 mode change ──
git config --global core.fileMode false

# ── Windows 上用系统证书库（能认公司/安全软件的根证书，见附录 A）──
git config --global http.sslBackend schannel
```

### 2.2 每一项在干什么（以及本机现状）

| 配置项 | 作用 | 本机当前值 | 来源 |
| --- | --- | --- | --- |
| `user.name` / `user.email` | 提交记录的作者信息 | `everret` / `2710895214@qq.com` | 全局 `C:/Users/win11/.gitconfig` |
| `init.defaultBranch` | 新建仓库的默认分支名 | `master`（系统级）→ **建议改成 `main`** | 系统级 `D:/software/github/Git/etc/gitconfig` |
| `core.autocrlf` | 换行符自动转换 | `true` | 系统级（Git for Windows 装机默认） |
| `pull.rebase` | `git pull` 用合并还是变基 | `false`（合并） | 系统级 |
| `credential.helper` | 记住 HTTPS 密码 | `manager` | 系统级 |
| `core.quotepath` | 中文路径显示 | 未设置 → 会转义显示，**建议设成 false** | — |
| `http.sslBackend` | TLS 实现 | `schannel` | 全局 |

查看某一项到底从哪来（排查「我明明设了为什么没生效」的利器）：

```bash
git config --show-origin --get core.autocrlf
git config --list --show-origin
```

### 2.3 配置优先级（为什么"我设了却不生效"）

从低到高，**后者覆盖前者**：

```
系统级 (--system)  →  全局级 (--global)  →  仓库级 (--local)
```

所以：某个仓库里单独设过的值，会盖掉全局。查看某个仓库自己的设置：

```bash
git config --local --list      # 只在这个仓库里执行
```

### 2.4 常用别名（省一半敲键盘时间）

```bash
git config --global alias.st  status
git config --global alias.stb "status -sb"
git config --global alias.co  checkout
git config --global alias.sw  switch
git config --global alias.br  branch
git config --global alias.lg  "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 --stat"
git config --global alias.unstage "restore --staged"
git config --global alias.uncommit "reset --soft HEAD~1"
git config --global alias.amend "commit --amend --no-edit"
git config --global alias.undo "restore"
```

之后 `git stb` 就等于 `git status -sb`，`git lg` 就是带图形的历史。

### 2.5 代理（国内网络常需要）

> 本机通过 WinHTTP 检测显示 **Direct access（无代理）**，实测也不需要在 git 里配代理。下面的命令留着备用。

```bash
# 走本地代理软件（端口按你软件的实际端口改，常见 7890 / 7897 / 10809）
git config --global http.proxy  http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

# 只给 GitHub 走代理，其他仓库走直连
git config --global http.https://github.com.proxy http://127.0.0.1:7890

# 取消代理（症状：关掉代理软件后 git 一直连不上/超时）
git config --global --unset http.proxy
git config --global --unset https.proxy
git config --global --unset http.https://github.com.proxy

# 查看当前有没有设代理
git config --global --get-regexp proxy
```

---

## 3. 认证：HTTPS 与 SSH 两套方案

两种情况 **选一种即可**，不要混着配。

### 3.1 HTTPS + Git Credential Manager（推荐，本项目正在用）

**优点**：不用管密钥，第一次推送弹一次浏览器授权，之后自动免密。

首次 `git push` 时 GCM 会自动弹出窗口让你登录 GitHub（浏览器授权或粘贴 token），登录一次之后凭据存进 Windows 凭据管理器，后续静默使用。

检查与配置：

```bash
git config --global credential.helper manager        # 应输出 manager
git config --global credential.https://github.com.username everret-chen
```

**换账号 / 认证一直失败时，清掉旧凭据**（这一步能解决绝大多数"密码明明对却推不上去"）：

1. 图形界面：`Win + R` → 输入 `control /name Microsoft.CredentialManager` → **Windows 凭据** → 找到 `git:https://github.com` → 删除。
2. 或命令行看 GCM 能做什么：

```bash
git credential-manager --help
git credential-manager diagnose
```

> **GitHub 早就不允许用登录密码做 git 操作了。** 如果需要手动输密码，必须用 **Personal Access Token（PAT）**：
> GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → 生成时勾选 **Contents: Read and write** → 把生成的 token 当密码粘贴。
> 用户名填 GitHub 用户名，密码填这个 token。

**改远程地址（HTTPS 换成别的写法 / 带用户名）**：

```bash
git remote -v
git remote set-url origin https://github.com/everret-chen/everret-knowledge-base.git
```

### 3.2 SSH（备用方案）

**优点**：不用存 token，一次配好后非常稳。

```bash
# 1) 生成密钥（邮箱换成你的；一路回车即可，密码短语可留空）
ssh-keygen -t ed25519 -C "2710895214@qq.com"
# 公钥在 %USERPROFILE%\.ssh\id_ed25519.pub，私钥 id_ed25519 绝不能外传

# 2) Windows 上启动 ssh-agent 并加入私钥 [PS]
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519

# 3) 查看并复制公钥 [PS]
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
# 粘贴到 GitHub → Settings → SSH and GPG keys → New SSH key

# 4) 验证（出现 "Hi 用户名! You've successfully authenticated" 即成功）
ssh -T git@github.com
```

把本仓库从 HTTPS 切到 SSH：

```bash
git remote set-url origin git@github.com:everret-chen/everret-knowledge-base.git
git remote -v
```

切回 HTTPS：

```bash
git remote set-url origin https://github.com/everret-chen/everret-knowledge-base.git
```

**`~/.ssh/config` 小技巧**（多账号或公司网络需要走 443 端口时）：

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    # 公司/学校封了 22 端口时启用下面这行，走 443
    # Port 443
```

---

## 4. 接入远程仓库（三种场景）

### 场景 A：GitHub 上已有仓库，本地目录是空的 ⭐ **本项目走的就是这条**

```bash
cd E:\everret-knowledge-base
git clone https://github.com/everret-chen/everret-knowledge-base.git .
```

- 末尾那个 **`.`** 表示"克隆到当前目录"，而不是新建一个同名文件夹。
- 要求当前目录**必须为空**（包括不能有隐藏的 `.git`），否则 git 会拒绝。
- 好处：一步同时拿到远程内容、`origin` 地址、`main` 分支跟踪关系，不会出现历史冲突。

克隆完立刻验证：

```bash
git remote -v
git status -sb          # 期望看到 ## main...origin/main
git log --oneline
```

### 场景 B：本地已经写了一堆笔记，GitHub 上是空仓库

```bash
cd E:\everret-knowledge-base
git init -b main                            # 初始化并把默认分支定为 main
git add -A
git commit -m "chore: 初始化知识库"
git remote add origin https://github.com/everret-chen/everret-knowledge-base.git
git push -u origin main                     # -u 记住上游，之后直接 git push 即可
```

> ⚠️ 在 GitHub 网页上新建仓库时，**不要勾选** "Add a README file" / ".gitignore" / "license"，否则远程会先有一个提交，push 时你会被拒绝（`! [rejected]`），得先 pull 再推。

### 场景 C：本地已有内容，远程也有 README / 初始提交（两边都有东西）

最安全的两条路：

```bash
# 路线 1：直接合并远程内容（保留两边历史，会产生一个合并提交）
git init -b main
git remote add origin https://github.com/everret-chen/everret-knowledge-base.git
git fetch origin
git pull origin main --allow-unrelated-histories    # 两边历史无关时必须加这个参数
# 处理冲突 → git add 冲突文件 → git commit
git push -u origin main

# 路线 2：线性历史（推荐，无意义的合并提交更少）
git pull --rebase origin main
git push -u origin main
```

**远程管理命令**：

```bash
git remote -v                                        # 查看
git remote add origin <url>                          # 新增
git remote set-url origin <url>                      # 改地址（换 HTTPS/SSH 就靠它）
git remote rename origin upstream                    # 改名
git remote remove upstream                           # 删除
git remote show origin                               # 看分支跟踪详情
```

---

## 5. 日常命令

> 这一章是核心。每条命令都注明「什么时候用」。

### 5.1 看状态：动手前先看

```bash
git status                  # 完整状态
git status -sb              # 简洁版（推荐，日常就用这个）
git status --ignored        # 连被忽略的文件一起显示
git diff                    # 工作区 vs 暂存区：我还没 add 的改了什么
git diff --staged           # 暂存区 vs 上次提交：我 add 了什么准备提交
git diff HEAD               # 工作区+暂存区 vs 上次提交：全部改动
git diff main origin/main   # 本地 main vs 远程 main：我领先/落后多少
```

**记不住就看这两条**：`git status -sb` 看大局，`git diff` 看细节。

### 5.2 看历史

```bash
git log --oneline                          # 每行一条，最常用
git log --oneline --graph --all --decorate # 带分支图形的全貌（= 别名 git lg）
git log -3 -p                              # 最近 3 条提交的完整改动
git log --author="everret"                 # 按作者过滤
git log --since="2 weeks ago" --oneline    # 按时间过滤
git log --stat -1                          # 最近一次提交改了哪些文件
git show <commit>                          # 看某次提交的全部内容
git show HEAD~1:web/sqli/payload.md        # 看某个文件在上一版的样子
```

### 5.3 提交改动

```bash
# ① 加入暂存区
git add web/sqli/payload.md      # 指定文件
git add web/sqli/                # 指定目录
git add -A                       # 全部（新增 + 修改 + 删除），最省事
git add -p                       # 逐块挑选要提交的改动（改动很杂时用，强烈推荐）

# ② 撤销暂存（add 错了，还没 commit）
git restore --staged web/sqli/payload.md    # 从暂存区拿出来，改动仍在
git restore --staged .                      # 全部撤出暂存区

# ③ 提交
git commit -m "feat(web/sqli): 增加报错注入函数清单"
git commit -m "标题" -m "正文：写详细说明"   # 多段提交信息
git commit -am "fix: 修正 payload 里的错别字" # 已跟踪文件的改动，跳过 add（新增文件仍需 add）

# ④ 提交信息写错了 / 漏了文件（⚠️ 只在还没 push 时用）
git add 漏掉的文件
git commit --amend --no-edit        # 把漏的文件并进上一条提交，信息不动
git commit --amend -m "新的信息"     # 顺便改提交信息
```

> ⚠️ **`--amend` 会重写提交历史。已经 push 过的提交不要 amend**，否则本地和远程历史分叉，下次推送会被拒。已推送的写错了就用 `git revert`（见 [6.5](#65-revert把错误提交反向抵消)）或新提交一次修正。

### 5.4 分支

知识库是个人仓库，**日常其实用不到分支**。但有两个场景值得用：

- 想大改目录结构，怕改崩 → 开分支试，不满意直接删。
- 想试新的组织方式 → 开分支，满意了再合并回来。

```bash
git branch                          # 本地分支，* 表示当前所在
git branch -a                       # 连远程分支一起看
git switch -c restructure           # 新建并切到 restructure
git switch main                     # 切回 main
git merge restructure               # 把 restructure 合并进当前分支
git merge --no-ff restructure       # 强制留一个合并节点（历史更清楚）
git branch -d restructure           # 删除已合并的分支
git branch -D restructure           # 强制删除（内容没合并也删，⚠️ 会丢）
git push -u origin restructure      # 把分支推上去
git push origin --delete restructure # 删除远程分支

# 变基：把自己分支的提交"搬"到最新 main 之上，历史成一条直线
git switch restructure
git rebase main                     # 冲突时改文件 → git add → git rebase --continue
git rebase --abort                  # 玩坏了，整体放弃回到变基前
```

### 5.5 同步：fetch / pull / push

```bash
git fetch                      # 只下载远程更新，不改你的工作区（最安全，先看看）
git fetch --all --prune        # 下载全部远程更新，并清掉远程已删的分支
git pull                       # = fetch + merge
git pull --rebase              # = fetch + rebase（历史线性，推荐日常用）
git push                       # 推送当前分支（前提：已 -u 设过上游）
git push -u origin main        # 首次推送并记住上游
git push origin main           # 指定分支推送
git push origin --tags         # 推送所有标签
git log origin/main --oneline -1   # 看远程分支最新提交
```

**推送被拒绝（`! [rejected] ... fetch first`）**：说明远程有你本地没有的提交（例如你在 GitHub 网页上直接改过文件）。处理：

```bash
git pull --rebase origin main      # 先把远程的拉下来，把你的提交叠在上面
git push                           # 再推
```

**绝不使用 `git push --force`。** 它会把远程历史直接覆盖掉，别人（以及你的另一台设备）的提交会永久消失。真要覆盖也用安全的那个版本：

```bash
git push --force-with-lease        # 只有远程没有新提交时才允许覆盖，防止误删
```

### 5.6 标签：给重要节点打标记

```bash
git tag -a v1.0-2025 -m "2025 赛季知识库整理完成"
git push origin v1.0-2025          # 推单个标签
git push origin --tags             # 推全部标签
git tag                            # 列出本地标签
git tag -d v1.0-2025               # 删本地标签
git push origin --delete v1.0-2025 # 删远程标签
```

### 5.7 忽略规则

```bash
git check-ignore -v 附件.zip       # 查这个文件是被哪一条规则忽略的
git status --ignored               # 列出所有被忽略的文件
git rm -r --cached 目录名           # 停止跟踪（保留本地文件！只从 git 里移除）
git rm -r --cached . && git add -A # 规则改完后重新应用一遍
git add -f 被忽略的文件             # 强行提交一个被 .gitignore 忽略的文件
```

> `git rm -r --cached` **不加 `--cached` 就会连本地文件一起删掉**，务必带上。

### 5.8 暂存现场（写到一半要去干别的）

```bash
git stash push -u -m "sqli 笔记写一半"   # -u 连未跟踪的新文件一起收起来
git stash list                          # 看有哪些暂存
git stash show -p stash@{0}              # 看内容
git stash pop                            # 恢复并删除这条暂存（常用）
git stash apply                          # 恢复但保留暂存（想应用到多个分支时用）
git stash drop stash@{0}                 # 删掉一条
git stash clear                          # 全清（⚠️ 不可恢复）
```

---

## 6. 撤销与后悔药

按「错到哪一步」选命令，**从上往下危害递增**：

| 我想撤销…… | 命令 | 影响范围 |
| --- | --- | --- |
| 工作区里某个文件的改动 | `git restore 文件` | 丢掉该文件未提交的修改 ⚠️ |
| 刚 `git add` 的东西（不丢改动） | `git restore --staged 文件` | 只退出暂存区 |
| 最近一次 commit（保留改动在暂存区） | `git reset --soft HEAD~1` | 改动都还在 |
| 最近一次 commit（改动回到工作区） | `git reset HEAD~1` | 改动都还在 |
| 最近一次 commit（改动全丢） | `git reset --hard HEAD~1` | ⚠️⚠️ 不可恢复（reflog 可救） |
| 已推送的错误提交 | `git revert <commit>` | 生成一个反向提交，历史不改写 |

### 6.1 丢弃工作区改动

```bash
git restore web/sqli/payload.md      # 单个文件
git restore .                        # 全部（⚠️ 未提交的改动直接消失）
git clean -nd                        # 先预览：哪些未跟踪文件会被删（安全，只显示）
git clean -fd                        # 真删（⚠️ 新写的、还没 add 的文件会没）
```

### 6.2 reset 三种模式（关键区别）

```bash
git reset --soft  HEAD~1   # 撤销提交，改动留在「暂存区」→ 想重新组织提交时用
git reset --mixed HEAD~1   # 撤销提交和暂存，改动留在「工作区」（默认）→ 最常用
git reset --hard  HEAD~1   # 撤销提交，改动全丢（⚠️）
git reset --hard origin/main   # 让本地分支和远程完全一致（⚠️ 丢弃本地所有未推送提交）
```

### 6.3 reflog：reset 之后的反悔药

`reset --hard` 丢掉的东西，**在 git 里通常还能找回来**（默认保留 30~90 天）：

```bash
git reflog                       # 列出 HEAD 走过的每一步，找到丢掉的那个提交 hash
git show <hash>                  # 确认就是它
git reset --hard <hash>          # 回到那个状态
git switch -c rescue <hash>      # 或者开个新分支把它捞出来，更安全
```

### 6.4 找回误删的分支

```bash
git reflog                                  # 找到被删分支最后的 hash
git switch -c 分支名 <hash>                  # 用那个 hash 重建分支
```

### 6.5 revert：把错误提交反向抵消

已经推送出去、又不想改写历史的场合：

```bash
git revert <commit>              # 生成一个"反向操作"的新提交
git revert HEAD                  # 撤销最近一次提交
git revert --no-commit <commit>  # 只改工作区，不自动提交，方便手工调整
```

---

## 7. 本仓库工作流

### 7.1 目录约定

```
everret-knowledge-base/
├── README.md                  # 总入口：目录导航 + 使用说明
├── .gitignore                 # 忽略规则（大附件、密钥、草稿）
├── .gitattributes             # 换行符规则（跨设备一致）
├── docs/
│   └── Git使用手册.md          # 本文档
├── web/                       # Web 方向
│   ├── README.md              # 方向总览 + 每个分类的进度
│   ├── sqli/                  # ┐
│   │   ├── 原理笔记.md         # │ 每个考点固定三件套：
│   │   ├── payload.md         # │ 原理 / payload / 做过的题
│   │   └── 做过的题.md         # ┘
│   ├── xss/                   # （同上三件套）
│   ├── ssti/
│   ├── ssrf/
│   ├── upload/                # 文件上传
│   └── rce/
├── crypto/                    # 密码学
├── misc/                      # 杂项（流量分析、隐写、取证…）
└── review/                    # 比赛复盘，一场一个文件
    ├── README.md              # 复盘索引
    ├── 复盘模板.md             # 复制它来写新复盘
    └── 2025-XX杯.md           # 实际复盘
```

**新增一个考点**：在对应方向下建子目录，放「原理笔记.md / payload.md / 做过的题.md」三件套（照抄现有目录的结构即可），然后在 `web/README.md` 登记一行。

### 7.2 提交信息规范

格式：`类型(范围): 简短描述`

| 类型 | 用在哪 | 示例 |
| --- | --- | --- |
| `feat` | 新增笔记 / payload / 考点 | `feat(web/sqli): 新增二次注入原理与例题` |
| `fix` | 修正错误、错别字、失效的 payload | `fix(web/ssti): 修正 Jinja2 沙箱逃逸链的错误函数名` |
| `docs` | 文档、模板、README 调整 | `docs: 补充 Git 手册的撤销章节` |
| `review` | 比赛复盘归档 | `review: 2025-XX杯复盘` |
| `chore` | 杂项（忽略规则、目录调整、配置） | `chore: 忽略 pcap 附件与密钥文件` |
| `refactor` | 重组目录结构、拆分文件 | `refactor(misc): 流量分析拆成独立考点` |

### 7.3 一次完整的写作流程

```bash
git pull --rebase                       # ① 先同步（多设备时必做）
# ② 写笔记：改原理笔记.md / payload.md / 做过的题.md
git status -sb                          # ③ 看改了啥
git add -A
git commit -m "feat(web/sqli): 补充时间盲注 payload 与例题"
git push                                # ④ 推上去
```

### 7.4 比赛复盘归档

```bash
cp review/复盘模板.md "review/2025-XX杯.md"    # [PS] Copy-Item 也行
# 填内容，再把关键题的 payload 顺手归档到对应 web/*/payload.md
git add -A && git commit -m "review: 2025-XX杯复盘" && git push
```

### 7.5 收工习惯（这条最重要）

知识库的价值在于**不丢**。养成两个习惯：

1. **每天收工前 `git push`**（多设备时先 `git pull --rebase`）。
2. **不确定要不要提交的临时草稿丢进 `scratch/`**（已被 `.gitignore` 忽略，不会污染仓库，也不会误提交）。

---

## 8. 排错速查

### 8.1 `fatal: refusing to merge unrelated histories`

本地和远程是两条互不相关的历史（常见于"远程先建了 README，本地又 `git init` 了一遍"）。

```bash
git pull origin main --allow-unrelated-histories
```

### 8.2 `! [rejected] main -> main (fetch first)`

远程有你没有的提交。**不要 `--force`**：

```bash
git pull --rebase origin main
git push
```

### 8.3 `LF will be replaced by CRLF` 警告

换行符转换提示，**不是错误**，可以安全忽略。想统一成一套规则，靠仓库里的 `.gitattributes`：

```bash
git add --renormalize .        # 按新规则把所有文件重新规范化一遍
git commit -m "chore: 统一换行符为 LF"
```

### 8.4 中文文件名显示成一串 `\346\226\207`

```bash
git config --global core.quotepath false
```

如果连输出内容都是乱码（终端问题）：

```powershell
chcp 65001                                              # CMD
[Console]::OutputEncoding = [Text.Encoding]::UTF8        # [PS]
git config --global i18n.logOutputEncoding utf-8
```

### 8.5 `Authentication failed` / 一直弹登录框

1. 先确认 GitHub 密码不能用于 git（必须用 PAT 或 SSH）。
2. 清掉旧凭据：凭据管理器 → Windows 凭据 → 删除 `git:https://github.com`。
3. 检查凭据助手：`git config --global credential.helper` 应为 `manager`。
4. 实在不行切 SSH：`git remote set-url origin git@github.com:everret-chen/everret-knowledge-base.git`。

### 8.6 `unable to get local issuer certificate` / `SSL certificate problem`

证书链不被信任。**按顺序试，不要一上来就关校验**：

```bash
# ① 改用 Windows 证书库（能认本机安全软件/公司网关的根证书）
git config --global http.sslBackend schannel

# ② 指定一个可信 CA 文件
git config --global http.sslCAInfo "D:/software/github/Git/mingw64/etc/ssl/certs/ca-bundle.crt"

# ③ 退出会对 HTTPS 做中转的安全软件 / VPN / 抓包工具后重试

# ④ 最后手段（⚠️ 关闭证书校验 = 放弃中间人攻击防护，仅限临时排障，用完立刻撤销）
git config --global http.sslVerify false
git config --global --unset http.sslVerify       # 恢复
```

本机就踩过这个坑，见 [附录 A](#附录-a-本机实测的两个坑)。

### 8.7 连不上 / 超时 / `Failed to connect to github.com port 443`

```bash
git config --global --get-regexp proxy     # 先看有没有残留代理配置
git config --global --unset http.proxy     # 关掉代理软件后必须清掉，否则一直超时
git config --global --unset https.proxy
ping github.com
ssh -T git@github.com                      # 换 SSH 通道试试
```

### 8.8 提交时提示 `Please tell me who you are`

```bash
git config --global user.name "everret-chen"
git config --global user.email "2710895214@qq.com"
```

### 8.9 不小心提交了大文件 / 密钥

```bash
git count-objects -vH              # 看仓库体积
```

- **刚提交、还没 push** → `git reset --soft HEAD~1`，把文件加进 `.gitignore`，重新提交。
- **已经 push 了** → 普通 `git rm` 删不掉历史里的内容，需要用 `git filter-repo` 改写历史（属于高风险操作），或者直接用 Git LFS。
- **密钥泄漏了** → **第一优先级是去 GitHub 把那个 token / key 撤销作废**，历史清理是第二件事。

### 8.10 `error: open("..."): Invalid argument`（文件被杀毒软件锁死）

**症状**：`git add` 卡在某一个文件上，其余文件正常：

```
error: open("web/rce/payload.md"): Invalid argument
error: unable to index file 'web/rce/payload.md'
fatal: adding files failed
```

同时用 PowerShell 读这个文件会报：

```
IOException: Operation did not complete successfully because the file
contains a virus or potentially unwanted software.
```

**原因**：Windows Defender（或第三方杀软）的实时防护把文件内容判定为病毒 / 黑客工具，从而**阻止任何程序打开它**。

> ⚠️ **别被报错误导**：`Invalid argument` 看起来像路径或编码问题，很容易让人去折腾文件名和 UTF-8，实际是杀软。
> 本知识库里的**一字不差的一句话木马**（PHP 那种 `@ev` + `al($_POST[1])` 的写法）必然命中特征——写 payload 库一定会遇到，属于日常问题。

**为什么"能创建却不能读"**：写入时杀软先放行、扫描后才拉黑；之后读取就被拦。所以会出现"我明明刚写进去"的错觉。

**处理步骤**：

1️⃣ **先确认是不是杀软**（关键一步）：

```powershell
# [PS] 逐个文件试读，读不出来的就是被锁的
Get-ChildItem -Recurse -File | ForEach-Object {
  $p = $_.FullName
  try { [void](Get-Content -LiteralPath $p -Raw -ErrorAction Stop) }
  catch { Write-Output "BLOCKED: $p" }
}
```

看看 Defender 的拦截记录：

```powershell
Get-MpThreatDetection | Select-Object -Last 5 InitialDetectionTime, Resources
```

2️⃣ **删除被锁文件**（删除通常不受影响），然后用不触发特征的写法重写：

```bash
Remove-Item 'web\rce\payload.md' -Force
```

3️⃣ **本仓库约定的替代写法**：一句话木马统一记成 **base64**，旁边给解码命令——既能直接用，又不会被锁。

```powershell
# [PS] 本地算好 base64 再粘进笔记
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('要记录的内容'))
```

```bash
# 使用时解码回来
echo <base64串> | base64 -d > /var/www/html/s.php
```

4️⃣ **如果整个库都要放各种 payload**，可以给仓库目录加 Defender 排除项（**需要管理员权限，等于降低本机防护，自行决定**）：

```powershell
# [PS，需管理员] 
Add-MpPreference -ExclusionPath 'E:\everret-knowledge-base'
# 撤销
Remove-MpPreference -ExclusionPath 'E:\everret-knowledge-base'
```

> **教训**：git 报 `Invalid argument` / `unable to index file` 这类"莫名其妙"的错误时，**先怀疑杀软，再怀疑编码和路径**。

---

## 9. 进阶但不复杂

### 9.1 `git add -p`：把一堆改动拆成多个干净的提交

改动很杂（比如同时改了 sqli 和 xss）时，用 `git add -p` 逐块决定，把提交拆得干净好读。

### 9.2 `git bisect`：定位"是哪次提交把东西改坏了"

```bash
git bisect start
git bisect bad                 # 当前版本有问题
git bisect good <好提交hash>    # 已知没问题的版本
# git 会自动切到中间版本，你测试后告诉它 good / bad，最后：
git bisect reset
```

### 9.3 查看某一行是谁什么时候写的

```bash
git blame web/sqli/payload.md
git log -L 10,20:web/sqli/payload.md    # 看某几行的完整变更历史
```

### 9.4 Git LFS：确实要存大附件时

```bash
git lfs install
git lfs track "*.pcap"          # 生成 .gitattributes 规则
git add .gitattributes
git add 大文件.pcap
git commit -m "chore: 用 LFS 管理 pcap 附件"
```

> LFS 有免费额度（存储 1GB / 流量 1GB·月），超了要付费。**首选方案仍然是附件放网盘，仓库里只存链接。**

### 9.5 多个设备同步的纪律

```
开工：git pull --rebase
收工：git add -A && git commit -m "..." && git push
```

两台设备同时改同一个文件才会冲突。冲突时打开文件，处理 `<<<<<<<` / `=======` / `>>>>>>>` 标记，然后：

```bash
git add 冲突文件
git rebase --continue      # 或 git commit（合并场景）
```

---

## 10. 命令速查表

| 目的 | 命令 |
| --- | --- |
| 看简洁状态 | `git status -sb` |
| 看未暂存的改动 | `git diff` |
| 看已暂存的改动 | `git diff --staged` |
| 看提交历史（图形） | `git log --oneline --graph --all --decorate` |
| 看最近一次提交详情 | `git show --stat HEAD` |
| 全部加入暂存区 | `git add -A` |
| 逐块挑选改动 | `git add -p` |
| 撤出暂存区 | `git restore --staged 文件` |
| 提交 | `git commit -m "feat(范围): 描述"` |
| 补进上次提交（未推送） | `git commit --amend --no-edit` |
| 拉取并变基 | `git pull --rebase` |
| 推送 | `git push` |
| 首次推送并设上游 | `git push -u origin main` |
| 看远程地址 | `git remote -v` |
| 改远程地址 | `git remote set-url origin <url>` |
| 丢弃某文件改动 | `git restore 文件` |
| 撤销最后一次提交（留改动） | `git reset --soft HEAD~1` |
| 撤销最后一次提交（改动回工作区） | `git reset HEAD~1` |
| 撤销最后一次提交（全丢） | `git reset --hard HEAD~1` |
| 已推送的错误提交 | `git revert <commit>` |
| 找回 reset 丢掉的东西 | `git reflog` → `git reset --hard <hash>` |
| 临时收起改动 | `git stash push -u -m "说明"` |
| 恢复暂存的改动 | `git stash pop` |
| 查文件被哪条规则忽略 | `git check-ignore -v 文件` |
| 停止跟踪（保留文件） | `git rm -r --cached 目录` |
| 强行提交被忽略的文件 | `git add -f 文件` |
| 打标签 | `git tag -a v1.0 -m "说明"` → `git push origin --tags` |
| 新建并切分支 | `git switch -c 分支名` |
| 切分支 | `git switch 分支名` |
| 合并分支 | `git merge 分支名` |
| 删本地分支 | `git branch -d 分支名` |
| 清掉未跟踪文件（先预览） | `git clean -nd` |
| 看仓库体积 | `git count-objects -vH` |

---

## 附录 A. 本机实测的两个坑

这两条是配置本仓库时**真实遇到并解决**的，记下来省得下次再查。

### A.1 受限终端里所有 HTTPS 连接都失败

**症状**：`git ls-remote` / `git clone` 报

```
schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS (0x8009030e)
```

连百度也一样失败，**所以不是 GitHub 的问题**，也不是网络被墙。原因是 Windows 的 TLS 组件（schannel）在受限权限的进程里拿不到凭据。

**解决**：网络相关的 git 命令（`clone` / `fetch` / `pull` / `push`）**在普通 PowerShell 窗口里执行**，不要在受限/沙箱终端里跑。本地命令（`add` / `commit` / `log` / `status`）不受影响。

### A.2 装了 TLS 中转软件，git 不认它的证书

**症状**：改用其他 TLS 后端后报

```
SSL certificate problem: unable to get local issuer certificate
```

**原因**：本机 `Cert:\CurrentUser\Root` 里存在 `Sangfor Technologies Inc.`、`DO_NOT_TRUST_FiddlerRoot` 之类的根证书——说明有安全网关 / 抓包软件在对 HTTPS 做中转并重签证书。**Windows 证书库信任它，但 Git 自带的公共 CA 列表不认它**，于是就报"找不到颁发者"。

**解决**（按推荐顺序）：

```bash
# ① 用 Windows 证书库，而不是 git 自带的 CA 列表
git config --global http.sslBackend schannel
```

```bash
# ② 或者临时退出抓包工具 / 公司安全客户端后重试
```

**不要**图省事设 `http.sslVerify false` —— 那等于放弃对中间人的防护。详见 [8.6](#86-unable-to-get-local-issuer-certificate--ssl-certificate-problem)。

---

## 附录 B. 配置文件放在哪

| 级别 | 本机路径 | 作用范围 | 查看命令 |
| --- | --- | --- | --- |
| 系统级 | `D:\software\github\Git\etc\gitconfig` | 本机所有用户所有仓库 | `git config --system --list` |
| 全局级 | `C:\Users\win11\.gitconfig` | 当前用户所有仓库 | `git config --global --list` |
| 仓库级 | `E:\everret-knowledge-base\.git\config` | 仅本仓库（优先级最高） | `git config --local --list` |

**本仓库的仓库级配置**（只影响这个仓库，换电脑 clone 后需要重设）：

```bash
git config user.name  "everret-chen"
git config user.email "2710895214@qq.com"
```

写入与删除：

```bash
git config --global 键 值          # 写
git config --global --unset 键      # 删
git config --global --list          # 列
git config --global --edit          # 直接用编辑器打开配置文件
```

> 注意：`.git/config` **不会**被 push 到 GitHub（`.git` 目录是本地数据），所以团队协作时把约定写进仓库里的 `.gitattributes` / `.gitignore`，而不是写进个人配置。
