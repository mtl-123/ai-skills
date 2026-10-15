---
name: remote-host-recon
description: 通过 SSH 登录远程 Linux 主机，一次性摸清「谁在用、跑着什么、资源够不够、工具链齐不齐」，产出一份环境勘察报告。当用户说「登录 X 机器看看」「读一下这台机器的内容」「远程主机环境勘察」「这台服务器上跑的是什么」，或者准备在陌生远程主机上做开发/排障前需要先建立环境认知时使用。
agent_created: true
---

# 远程主机环境勘察

把一台陌生（或久未登录）的远程 Linux 主机，在 3~5 轮内摸清底细，并落成一份可交付的报告。

## 何时用

- 用户给了 IP / 主机别名，让你「连上去看看」「读一下内容」
- 准备在该机器上开发或排障，但不知道环境现状
- 需要判断某台机器能否承担某个任务（资源、工具链是否够）

## 前置：先修 PATH（Windows + Git Bash 必做）

本机 Git Bash 的默认 PATH 可能缺少 `dirname` 等工具，任何 `&&` / 子命令都会报
`dirname: command not found`。**每条命令开头都加**：

```bash
export PATH="/usr/bin:/bin:/c/Windows/System32:/c/Windows/System32/OpenSSH:$PATH"
```

同一环境下 PowerShell 工具的输出偶发不回显，**优先用 Bash 工具跑命令**。

## 步骤

### 第 1 轮：连通性与身份

```bash
ssh -o BatchMode=yes -o ConnectTimeout=8 -o LogLevel=ERROR <host> \
  'whoami; id; echo $HOME; hostname; uname -a; cat /etc/os-release | head -3'
```

`BatchMode=yes` 关键：密钥不通时立刻失败，不会卡在密码提示上。

### 第 2 轮：批量探活（多主机时）

一次循环 + 后台并发，`ConnectTimeout=3`。**必须区分报错类型**：

- `Connection refused` → 端口没监听（sshd 没起 / 端口不对）
- `Connection timed out` → 防火墙、安全组或机器不在线
- `Your password has expired ... no TTY available` → 密钥认证其实已通过，
  是账号密码过期被 sshd 拦截 → 交互式改密：`ssh -t <host> passwd`

判断成功**不要**用 `grep "|"` 之类的宽松匹配，会把错误信息也判成成功。
正确做法：显式检查是否拿到 hostname，例如
`r=$(ssh ... 'hostname' 2>&1); case "$r" in *"timed out"*|*"refused"*|*"Permission denied"*) DOWN=1;; esac`

### 第 3 轮：家目录与资源

```bash
ls -la $HOME
df -h / ; free -h ; uptime
for c in java mvn node npm git docker python3; do
  printf "%-8s " "$c"; command -v $c >/dev/null && $c --version 2>&1 | head -1 || echo "(未安装)"
done
```

注意：`mvn` / `node` 常不在 PATH，而是通过项目内 `./mvnw` 或 `~/.nvm` 提供。
装成「未安装」容易误判，要结合 `ls ~/.nvm ~/.m2` 一起看。

### 第 4 轮：在跑什么

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
systemctl list-units --type=service --state=running | head -20
```

若发现 compose 项目，把 `docker-compose.yml` 与 `*-production.yml` 都读出来——
生产覆盖文件往往用 `!reset []` 收敛端口暴露，只看主文件会误判暴露面。

### 第 5 轮：项目定位

先确认是不是 Git 仓库：`git rev-parse --is-inside-work-tree`。
**不是 Git 仓库是重要风险信号**（无法 diff/回滚，往往靠 tar 快照留痕），要在报告里明确点出。

再看规模与结构：`find src -name "*.java" | wc -l`、顶层包/模块列表。

## 报告结构

按此顺序组织，最后用 `present_files` 交付 Markdown：

1. 账号与系统（用户、家目录、发行版、glibc、uptime）
2. 家目录内容清单（分「项目部署 / 备份快照 / 配置文件 / 工具缓存」四类）
3. 主项目：业务定位 + 目录结构 + 技术栈 + 质量基线
4. 运行中的服务（容器 / 端口 / 状态）
5. 系统资源（磁盘、内存、负载）
6. 工具链版本
7. **值得注意的点**（风险与隐患，这是报告最有价值的部分）
8. **未读取的敏感文件**（列出路径 + 说明故意未展开内容并给出原因）

## 硬约束

- **只读**。勘察阶段不修改远端任何文件；要改动必须先向用户说明并取得确认。
- **敏感文件不展开**：`*.env`、`production.env`、`wecom.local.env`、`credentials*`、
  测试身份 SQL —— 识别到即列出路径，不 `cat` 内容。用户明确要求且说明用途后再读。
- 命令统一用 `ssh <host> '...'` 单次批量下发，减少往返；输出用 `=== 标题 ===` 分段，
  否则多段输出回来无法区分。

## 收尾

- 把环境结论写入记忆：主机画像、用户名、端口、项目定位、待修项。
- 若后续要做远程开发，顺手查 glibc：`ldd --version | head -1`。
  **VS Code / CodeBuddy IDE 的 Remote-SSH 要求远端 glibc ≥ 2.28**，不达标会报
  `Could not establish connection to`。
