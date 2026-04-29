# Git 高级用法与踩坑记录

> **文章类型**：经验清单 + 问题与解答  
> **适用读者**：日常使用 Git，想掌握高级操作和避免常见问题的开发者  
> **最后更新**：2026-04  
> **标签**：`Git` `版本控制` `rebase` `cherry-pick` `工具` `效率`

---

## 目录

1. [Git 工作原理简图](#1-git-工作原理简图)
2. [分支策略](#2-分支策略)
3. [高频进阶操作](#3-高频进阶操作)
4. [撤销与修复操作](#4-撤销与修复操作)
5. [子模块（submodule）](#5-子模块submodule)
6. [Git Hooks 自动化](#6-git-hooks-自动化)
7. [嵌入式项目的 .gitignore 模板](#7-嵌入式项目的-gitignore-模板)
8. [常见踩坑记录](#8-常见踩坑记录)
9. [TODO](#9-todo)

---

## 1. Git 工作原理简图

```
工作目录（Working Directory）
    │ git add
    ▼
暂存区（Index/Staging Area）
    │ git commit
    ▼
本地仓库（Local Repository）
    │ git push
    ▼
远程仓库（Remote Repository，如 GitHub）
```

**关键命令方向：**

```bash
git status        # 查看工作目录和暂存区状态
git add .         # 工作目录 → 暂存区
git commit        # 暂存区 → 本地仓库
git push          # 本地仓库 → 远程仓库
git pull          # 远程仓库 → 本地仓库（= git fetch + git merge）
git fetch         # 远程仓库 → 本地仓库（只下载，不合并）
```

---

## 2. 分支策略

### 推荐：Git Flow（适合嵌入式发版项目）

```
main（生产环境，只接受 release 和 hotfix 合并）
    │
release/v1.2（发版准备，仅 bugfix）
    │
develop（集成分支，接受 feature 合并）
    ├── feature/ocpp-2.0-upgrade
    ├── feature/insulation-detection
    └── feature/ota-update
```

### 简化策略（小团队推荐）

```
main（主分支，发版打 tag）
    ├── develop（日常开发）
    └── hotfix/xxx（紧急修复）
```

### 提交信息规范（Conventional Commits）

```
格式：<type>(<scope>): <description>

类型：
  feat:     新功能
  fix:      Bug 修复
  refactor: 重构（非 bug，非新功能）
  docs:     文档变更
  test:     测试相关
  chore:    构建/工具/依赖变更
  perf:     性能优化

示例：
  feat(ocpp): implement RemoteStartTransaction support
  fix(uart): fix DMA buffer overflow on 921600 baud
  docs(readme): update flashing instructions for GD32F303
```

---

## 3. 高频进阶操作

### rebase — 保持整洁的线性历史

```bash
# 场景：feature 分支开发时，develop 有新提交，需要同步

# 方法一：merge（产生 merge commit，历史有分叉）
git checkout feature/xxx
git merge develop

# 方法二：rebase（历史保持线性，更整洁）
git checkout feature/xxx
git rebase develop

# 交互式 rebase：整理提交记录（合并/拆分/修改提交信息）
git rebase -i HEAD~5  # 整理最近 5 个提交
```

**交互式 rebase 常用命令：**

```
pick   保留提交
reword 保留提交但修改提交信息
squash 合并到前一个提交（保留提交信息）
fixup  合并到前一个提交（丢弃提交信息）
drop   删除该提交
```

!!! warning "rebase 黄金法则"
    **不要对已推送到远程的公共分支执行 rebase！**  
    rebase 会改变提交历史，会让其他人的本地分支产生冲突。只在自己的私有分支上使用。

### cherry-pick — 从其他分支摘取提交

```bash
# 场景：hotfix 修复了 develop 上的 bug，需要也应用到 main

# 1. 找到需要的提交哈希
git log --oneline develop  # 找到 abc1234

# 2. 在 main 上应用该提交
git checkout main
git cherry-pick abc1234     # 单个提交
git cherry-pick abc1234..def5678  # 连续多个提交

# 遇到冲突时
git cherry-pick --continue  # 解决冲突后继续
git cherry-pick --abort     # 放弃 cherry-pick
```

### stash — 临时保存未完成的工作

```bash
# 场景：正在开发 feature，突然需要切换到其他分支修 bug

git stash               # 保存当前工作（包括暂存区）
git stash push -m "WIP: add DMA rx implementation"  # 带备注

git stash list          # 查看所有 stash
git stash pop           # 恢复最新 stash 并删除
git stash apply stash@{2}  # 恢复指定 stash（不删除）
git stash drop stash@{2}   # 删除指定 stash
git stash clear            # 清空所有 stash
```

### bisect — 二分法查找引入 bug 的提交

```bash
# 场景：某个版本引入了 bug，但不知道是哪个提交

git bisect start
git bisect bad          # 标记当前版本有 bug
git bisect good v1.2    # 标记某个已知正常的版本

# Git 自动切换到中间提交，测试后标记
git bisect good         # 当前没有 bug
git bisect bad          # 当前有 bug

# Git 自动定位到引入 bug 的提交
git bisect reset        # 结束 bisect，回到原来的分支
```

---

## 4. 撤销与修复操作

### 常用撤销命令对比

| 命令 | 作用 | 是否修改历史 |
|------|------|------------|
| `git checkout -- file` | 丢弃工作目录的修改 | 否 |
| `git restore file` | 同上（新版语法） | 否 |
| `git reset HEAD file` | 撤销 add（退出暂存区） | 否 |
| `git commit --amend` | 修改最近一次提交 | 是（改历史） |
| `git reset --soft HEAD~1` | 撤销最近提交，保留暂存区 | 是 |
| `git reset --mixed HEAD~1` | 撤销最近提交，修改退回工作目录 | 是 |
| `git reset --hard HEAD~1` | 撤销最近提交，丢弃所有修改 | 是 |
| `git revert HEAD` | 创建一个新提交来撤销 | 否（安全） |

!!! danger "git reset --hard 危险操作"
    `git reset --hard` 会丢失未提交的修改，无法通过 Git 恢复！  
    操作前先 `git stash` 或确认不需要这些修改。

### 恢复误删的提交（reflog）

```bash
# 即使 reset --hard 了，只要没过期（默认 90 天），还能找回来

git reflog              # 查看所有操作历史，找到目标 SHA

git reset --hard HEAD@{3}   # 回到 reflog 中某个状态
# 或
git checkout abc1234    # 创建临时分支保存
```

---

## 5. 子模块（submodule）

适合管理嵌入式项目中的第三方库（如 FreeRTOS、LWIP）。

```bash
# 添加子模块
git submodule add https://github.com/FreeRTOS/FreeRTOS-Kernel.git Middleware/FreeRTOS

# 克隆包含子模块的项目
git clone --recurse-submodules https://github.com/xxx/project.git
# 或
git clone https://github.com/xxx/project.git
git submodule update --init --recursive

# 更新子模块到最新版本
git submodule update --remote

# 删除子模块（稍微麻烦）
git submodule deinit Middleware/FreeRTOS
git rm Middleware/FreeRTOS
rm -rf .git/modules/Middleware/FreeRTOS
```

---

## 6. Git Hooks 自动化

Git Hooks 可以在特定事件时自动执行脚本。

### pre-commit：提交前检查

```bash
# .git/hooks/pre-commit（赋予执行权限：chmod +x）
#!/bin/bash

# 检查是否有调试代码残留
if git diff --cached | grep -E "printf\s*\(|DEBUG_PRINT|TODO_REMOVE" > /dev/null; then
    echo "错误：发现调试打印语句，请清理后再提交"
    exit 1
fi

# 运行 cppcheck 静态检查（如有安装）
if command -v cppcheck &> /dev/null; then
    cppcheck --error-exitcode=1 Application/ BSP/ 2>&1
fi

exit 0
```

### commit-msg：检查提交信息格式

```bash
# .git/hooks/commit-msg
#!/bin/bash

commit_msg=$(cat "$1")
pattern="^(feat|fix|refactor|docs|test|chore|perf)(\(.+\))?: .{5,}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "提交信息格式错误！"
    echo "正确格式：feat(scope): description"
    exit 1
fi
```

---

## 7. 嵌入式项目的 .gitignore 模板

```gitignore
# ── Keil 工程文件 ──────────────────────────────────────
*.uvguix.*
*.uvgui.*
*.dbgconf
*.dep
*.d
*.axf
*.htm
*.map
*.lnp
*.sct
*.lst
*.o
*.lib
RTE/

# ── 编译输出 ───────────────────────────────────────────
Objects/
Listings/
build/
out/
*.elf
*.bin
*.hex
*.s19

# ── IDE 配置（可选保留，视团队情况） ───────────────────
.vscode/c_cpp_properties.json   # 本地 VSCode 配置
.vscode/launch.json             # 本地调试配置

# ── 日志与临时文件 ────────────────────────────────────
*.log
*.tmp
*.bak
*.orig
*.swp
*.swo

# ── macOS 系统文件 ────────────────────────────────────
.DS_Store
__MACOSX/

# ── Windows 系统文件 ──────────────────────────────────
Thumbs.db
desktop.ini
```

---

## 8. 常见踩坑记录

### 踩坑1：推送时被拒绝（non-fast-forward）

```
错误：! [rejected] main -> main (non-fast-forward)
原因：远程有你本地没有的提交

正确做法：
git pull --rebase origin main  # 优先用 rebase 保持历史整洁
# 解决冲突后
git push origin main

错误做法：
git push --force  # ❌ 可能覆盖别人的提交！
```

### 踩坑2：意外将 secrets 提交到仓库

```bash
# 立即删除敏感文件
git rm --cached config_with_secret.h
echo "config_with_secret.h" >> .gitignore
git commit -m "chore: remove accidentally committed secret file"

# 如果已经推送，需要彻底清理历史（危险操作，需团队协调）
# 使用 git filter-repo 工具清理历史
pip install git-filter-repo
git filter-repo --path config_with_secret.h --invert-paths

# 同时：立即更换泄露的密钥/密码！
```

### 踩坑3：中文文件名乱码

```bash
# 解决 git status 中文文件名显示为 \342\201\... 的问题
git config --global core.quotepath false
```

### 踩坑4：行尾换行符问题（Windows/Linux协作）

```bash
# 在 .gitattributes 中统一配置
echo "* text=auto eol=lf" > .gitattributes
echo "*.bat text eol=crlf" >> .gitattributes
echo "*.ps1 text eol=crlf" >> .gitattributes
```

---

## 9. TODO

- [ ] GitHub Actions CI/CD 在嵌入式项目中的应用
- [ ] Git Large File Storage（LFS）管理固件二进制文件
- [ ] Gerrit 代码审查流程（企业内部协作）
- [ ] 语义化版本（semver）自动化打 tag 脚本
