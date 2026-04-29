# Linux 常用命令与 Shell 技巧

> **文章类型**：速查手册 + 实用脚本  
> **适用读者**：嵌入式开发者、想提升 Linux 命令行效率的工程师  
> **最后更新**：2026-04  
> **标签**：`Linux` `Shell` `Bash` `命令行` `效率` `工具`

---

## 目录

1. [文件与目录操作](#1-文件与目录操作)
2. [文本处理三剑客](#2-文本处理三剑客)
3. [进程与资源管理](#3-进程与资源管理)
4. [网络命令](#4-网络命令)
5. [Shell 脚本实用模板](#5-shell-脚本实用模板)
6. [嵌入式开发专用命令](#6-嵌入式开发专用命令)
7. [效率提升技巧](#7-效率提升技巧)
8. [TODO](#8-todo)

---

## 1. 文件与目录操作

```bash
# ── 查找文件 ────────────────────────────────────────────
find . -name "*.c"                    # 当前目录下所有 .c 文件
find . -name "*.c" -newer Makefile    # 比 Makefile 新的 .c 文件
find . -size +1M                      # 大于 1MB 的文件
find . -type d -name "build"          # 名为 build 的目录
find . -name "*.o" -delete            # 删除所有 .o 文件

# ── 文件权限 ────────────────────────────────────────────
chmod +x script.sh                    # 添加执行权限
chmod 644 config.h                    # rw-r--r--
chmod -R 755 directory/               # 递归设置目录权限

# ── 文件内容查看 ─────────────────────────────────────────
cat file.txt                          # 完整输出
head -20 file.txt                     # 前 20 行
tail -20 file.txt                     # 后 20 行
tail -f /var/log/syslog               # 实时追踪日志
less file.txt                         # 分页查看（q 退出，/ 搜索）

# ── 文件比较 ────────────────────────────────────────────
diff file1.c file2.c                  # 差异对比
diff -r dir1/ dir2/                   # 目录递归对比
vimdiff file1.c file2.c              # 可视化对比

# ── 压缩与解压 ───────────────────────────────────────────
tar -czvf archive.tar.gz directory/   # 压缩
tar -xzvf archive.tar.gz              # 解压
tar -xzvf archive.tar.gz -C /tmp/    # 解压到指定目录
zip -r archive.zip directory/
unzip archive.zip

# ── 软硬链接 ────────────────────────────────────────────
ln -s /usr/bin/arm-none-eabi-gcc gcc  # 软链接
ln source.c hardlink.c               # 硬链接
```

---

## 2. 文本处理三剑客

### grep — 文本搜索

```bash
# 基本搜索
grep "TODO" *.c                       # 在所有 .c 文件中搜索
grep -r "TODO" src/                   # 递归搜索目录
grep -i "error" log.txt               # 忽略大小写
grep -n "void task_" *.c             # 显示行号
grep -l "FreeRTOS" *                  # 只显示文件名

# 正则表达式
grep -E "int [a-z_]+\s*\(" *.c       # 查找函数定义
grep -E "0x[0-9A-Fa-f]{8}" log.txt   # 查找 8 位十六进制数

# 上下文
grep -A 3 "HardFault" *.c            # 匹配行后 3 行
grep -B 3 "HardFault" *.c            # 匹配行前 3 行
grep -C 3 "HardFault" *.c            # 前后各 3 行

# 反向搜索
grep -v "^//" *.c                     # 排除注释行
grep -v "^$" file.txt                 # 排除空行

# 统计
grep -c "Error" log.txt              # 统计匹配行数
```

### awk — 结构化文本处理

```bash
# 打印指定列（字段分隔符默认为空白）
awk '{print $1, $3}' data.txt
awk -F',' '{print $2}' data.csv      # 逗号分隔

# 条件过滤
awk '$3 > 100 {print $0}' data.txt   # 第3列 > 100 的行
awk '/ERROR/ {print NR, $0}' log.txt # 包含 ERROR 的行（带行号）

# 统计求和
awk '{sum += $2} END {print "Total:", sum}' data.txt

# 处理 CAN 日志（实际工作中常用）
# 筛选 CAN ID = 0x18FF50E5 的报文，并提取第4字段（数据）
awk -F',' '$2 == "0x18FF50E5" {print $1, $4}' can_log.csv
```

### sed — 流式文本编辑

```bash
# 替换
sed 's/旧字符串/新字符串/g' file.txt         # 全局替换，输出到终端
sed -i 's/旧字符串/新字符串/g' file.txt      # 原地替换（修改文件）
sed -i.bak 's/foo/bar/g' file.txt            # 替换并备份原文件

# 删除
sed '/^$/d' file.txt                          # 删除空行
sed '/^#/d' file.txt                          # 删除注释行
sed '10,20d' file.txt                         # 删除 10-20 行

# 打印指定行
sed -n '10,20p' file.txt                      # 只打印 10-20 行

# 批量重命名（结合 find）
find . -name "*.txt" -exec sed -i 's/old/new/g' {} \;
```

---

## 3. 进程与资源管理

```bash
# ── 进程查看 ────────────────────────────────────────────
ps aux                               # 所有进程
ps aux | grep openocd                # 过滤进程
top                                  # 实时进程监控
htop                                 # 更友好的 top（需安装）

# ── 进程控制 ────────────────────────────────────────────
kill 1234                            # 发送 SIGTERM（优雅停止）
kill -9 1234                         # 发送 SIGKILL（强制终止）
killall openocd                      # 按名称终止
pkill -f "python script.py"          # 按命令行匹配

# ── 后台运行 ────────────────────────────────────────────
command &                            # 后台运行
nohup command &                      # 即使终端关闭也继续运行
screen -S session_name               # 创建持久会话
tmux new -s session_name             # tmux 会话（推荐）

# ── 资源使用 ────────────────────────────────────────────
df -h                                # 磁盘空间
du -sh directory/                    # 目录大小
du -sh * | sort -rh | head -10      # 最大的 10 个目录/文件
free -h                              # 内存使用
lscpu                                # CPU 信息

# ── 服务管理（systemd）────────────────────────────────────
systemctl start   service_name
systemctl stop    service_name
systemctl restart service_name
systemctl enable  service_name       # 开机自启
systemctl status  service_name
journalctl -u service_name -f        # 实时查看服务日志
```

---

## 4. 网络命令

```bash
# ── 连通性测试 ──────────────────────────────────────────
ping -c 4 192.168.1.1
traceroute google.com

# ── 端口与连接 ──────────────────────────────────────────
netstat -tlnp                        # 所有监听端口
ss -tlnp                             # 同上（更快）
ss -tp                               # 所有 TCP 连接

# ── 文件传输 ────────────────────────────────────────────
scp file.bin user@192.168.1.100:/tmp/        # 上传文件
scp user@192.168.1.100:/tmp/log.txt .        # 下载文件
scp -r directory/ user@host:/path/           # 上传目录

rsync -avz --progress source/ user@host:dest/  # 增量同步（推荐）

# ── 串口工具 ────────────────────────────────────────────
minicom -D /dev/ttyUSB0 -b 115200    # 串口终端
screen /dev/ttyUSB0 115200           # 轻量串口终端
picocom -b 115200 /dev/ttyUSB0       # 推荐：功能完善
stty -F /dev/ttyUSB0 115200 raw      # 设置串口参数

# ── HTTP 测试 ────────────────────────────────────────────
curl -X GET http://localhost:8080/status
curl -X POST -H "Content-Type: application/json" \
     -d '{"key":"value"}' http://api.example.com/endpoint
wget -O firmware.bin http://server/firmware.bin

# ── SSH 技巧 ────────────────────────────────────────────
ssh -L 8080:localhost:8080 user@remote_host   # 本地端口转发
ssh -N -f -L 9000:csms.internal:9000 jump_host  # 后台端口转发
```

---

## 5. Shell 脚本实用模板

### 通用脚本模板

```bash
#!/bin/bash
# ============================================================
# 脚本名：deploy_firmware.sh
# 功能  ：自动编译并烧录固件
# 用法  ：./deploy_firmware.sh [clean]
# 作者  ：lin-JHTP
# 日期  ：2026-04
# ============================================================

set -euo pipefail   # 遇错退出(-e)，未定义变量报错(-u)，管道错误处理(-o pipefail)
IFS=$'\n\t'         # 安全的字段分隔符

# ── 颜色定义 ────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # 恢复默认

log_info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
log_error() { echo -e "${RED}[ERROR]${NC} $*" >&2; }

# ── 主逻辑 ──────────────────────────────────────────────────
BUILD_DIR="build"
TARGET="firmware"
OPENOCD_CFG="openocd.cfg"

main() {
    log_info "开始编译固件..."

    if [[ "${1:-}" == "clean" ]]; then
        log_warn "清理编译目录..."
        make clean
    fi

    if ! make -j"$(nproc)" 2>&1; then
        log_error "编译失败！"
        exit 1
    fi

    log_info "编译成功！固件大小："
    arm-none-eabi-size "${BUILD_DIR}/${TARGET}.elf"

    log_info "开始烧录..."
    openocd -f "${OPENOCD_CFG}" \
            -c "program ${BUILD_DIR}/${TARGET}.elf verify reset exit"

    log_info "烧录完成！"
}

main "$@"
```

### 批量处理 CAN 日志

```bash
#!/bin/bash
# 批量分析 CAN 日志，提取充电会话数据

LOG_DIR="./can_logs"
OUTPUT_DIR="./analysis"
mkdir -p "$OUTPUT_DIR"

for log_file in "$LOG_DIR"/*.csv; do
    filename=$(basename "$log_file" .csv)
    log_info "处理: $filename"

    # 提取 BCS 报文（0x1882D6F4）并计算充电时长
    awk -F',' '
        $2 == "0x1882D6F4" {
            if (NR == 1) start_time = $1
            end_time = $1
            count++
        }
        END {
            printf "会话时长: %.1f 分钟，报文数: %d\n",
                   (end_time - start_time) / 60, count
        }
    ' "$log_file" > "$OUTPUT_DIR/${filename}_summary.txt"
done

log_info "分析完成，结果在 $OUTPUT_DIR"
```

---

## 6. 嵌入式开发专用命令

```bash
# ── 工具链 ──────────────────────────────────────────────────
arm-none-eabi-gcc --version
arm-none-eabi-size firmware.elf           # 查看各段大小（text/data/bss）
arm-none-eabi-objdump -d firmware.elf     # 反汇编
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin  # 转 binary
arm-none-eabi-nm --size-sort firmware.elf | tail -20        # 最大的符号

# 查看固件大小详情
arm-none-eabi-size -A firmware.elf | sort -k2 -rn | head -20

# ── OpenOCD 操作 ──────────────────────────────────────────────
# 只烧录不运行
openocd -f interface/jlink.cfg -f target/stm32f3x.cfg \
        -c "init; halt; program firmware.elf; exit"

# 只复位目标
openocd -f interface/jlink.cfg -f target/stm32f3x.cfg \
        -c "init; reset; exit"

# 读取 Flash 内容到文件
openocd -f interface/jlink.cfg -f target/stm32f3x.cfg \
        -c "init; halt; dump_image backup.bin 0x08000000 0x100000; exit"

# ── 串口快速收发测试 ────────────────────────────────────────
# 发送数据到串口
echo -ne "\x01\x02\x03" > /dev/ttyUSB0

# 脚本监听串口并记录日志
while true; do
    cat /dev/ttyUSB0 | tee -a serial_log_$(date +%Y%m%d).txt
done

# ── 二进制文件分析 ────────────────────────────────────────────
xxd firmware.bin | head -20            # 十六进制查看
strings firmware.bin | grep -i "version"  # 提取字符串
hexdump -C firmware.bin | head -10     # 带 ASCII 的十六进制
```

---

## 7. 效率提升技巧

### 快捷键（Bash）

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+R` | 搜索历史命令 |
| `Ctrl+A` | 跳到行首 |
| `Ctrl+E` | 跳到行尾 |
| `Ctrl+W` | 删除前一个单词 |
| `Ctrl+L` | 清屏 |
| `!!` | 重复上一条命令 |
| `!$` | 上一条命令的最后一个参数 |
| `Alt+.` | 插入上一条命令的最后参数 |

### 有用的 Bash 配置（~/.bashrc）

```bash
# ── 常用别名 ────────────────────────────────────────────────
alias ll='ls -alFh --color=auto'
alias la='ls -Ah'
alias cls='clear'
alias grep='grep --color=auto'

# 嵌入式开发别名
alias build='make -j$(nproc)'
alias flash='openocd -f openocd.cfg -c "program build/firmware.elf verify reset exit"'
alias monitor='picocom -b 115200 /dev/ttyUSB0 --imap lfcrlf'

# ── Git 别名 ────────────────────────────────────────────────
alias gs='git status'
alias gl='git log --oneline --graph --all -15'
alias gd='git diff'
alias ga='git add -p'  # 交互式暂存

# ── 工具函数 ────────────────────────────────────────────────
# 快速进入项目目录
cdp() { cd ~/projects/"$1" || return; }

# 查找并 kill 进程
fkill() { kill -9 "$(pgrep -f "$1")"; }

# 快速备份文件
bak() { cp "$1" "$1.bak.$(date +%Y%m%d_%H%M%S)"; }
```

---

## 8. TODO

- [ ] tmux 会话管理详细教程（窗口/面板/会话恢复）
- [ ] 正则表达式速查手册（grep/awk/sed 通用）
- [ ] Shell 脚本调试技巧（set -x/bashdb）
- [ ] Makefile 从入门到实战（依赖分析/自动变量）
- [ ] Python 替代 Shell 脚本的场景（串口自动测试）
