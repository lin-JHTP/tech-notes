# 工具集锦

> 开发效率工具速查手册：Git、Linux、VSCode、Docker 等常用工具的核心命令和配置技巧。

---

## 🔧 Git 常用命令

```bash
# ── 基础操作 ──
git init                          # 初始化仓库
git clone <url>                   # 克隆仓库
git status                        # 查看状态
git add .                         # 暂存所有修改
git commit -m "feat: 添加xx功能"   # 提交（建议使用约定式提交信息）
git push origin main              # 推送到远端

# ── 分支管理 ──
git branch feature/my-feature    # 创建分支
git checkout feature/my-feature  # 切换分支
git merge feature/my-feature     # 合并分支
git branch -d feature/my-feature # 删除分支

# ── 日志与回退 ──
git log --oneline --graph         # 图形化查看提交历史
git diff HEAD~1                   # 与上一次提交对比
git reset --soft HEAD~1           # 撤销最近一次提交（保留文件修改）
```

---

## 🐧 Linux 常用命令

```bash
# ── 文件操作 ──
ls -la           # 列出文件（含隐藏文件）
find . -name "*.c"  # 递归查找 .c 文件
grep -rn "TODO" src/  # 递归搜索关键字

# ── 进程管理 ──
ps aux | grep python   # 查找进程
kill -9 <PID>          # 强制终止进程
top / htop             # 实时进程监控

# ── 串口调试（嵌入式常用）──
minicom -b 115200 -D /dev/ttyUSB0   # 串口终端
screen /dev/ttyUSB0 115200          # 另一种串口终端

# ── 权限管理 ──
chmod +x script.sh    # 添加执行权限
sudo chown user file  # 修改文件所有者
```

---

## 💻 VSCode 实用技巧

| 快捷键（Windows/Linux） | 功能 |
|------------------------|------|
| `Ctrl+Shift+P` | 命令面板（最重要！） |
| `Ctrl+P` | 快速打开文件 |
| `Ctrl+G` | 跳转到指定行 |
| `F12` | 跳转到定义 |
| `Ctrl+Shift+F` | 全局搜索 |
| `Alt+Click` | 多光标编辑 |

### 推荐嵌入式开发扩展

- **C/C++**（ms-vscode.cpptools）：C 语言智能提示
- **Cortex-Debug**：ARM 嵌入式调试
- **GitLens**：Git 增强
- **Serial Monitor**：串口监视

---

## 🐳 Docker 基础（AI 开发常用）

```bash
# 拉取镜像
docker pull pytorch/pytorch:latest

# 运行容器（挂载当前目录）
docker run -it --gpus all \
    -v $(pwd):/workspace \
    pytorch/pytorch:latest bash

# 常用命令
docker ps              # 查看运行中的容器
docker images          # 查看本地镜像
docker stop <id>       # 停止容器
```

---

## 📝 待补充内容

- Git 进阶（rebase、cherry-pick、stash）
- Linux Shell 脚本编写
- Make / CMake 构建系统
- SSH 远程开发配置
