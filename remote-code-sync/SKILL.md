---
name: remote-code-sync
description: 把本地（Windows + Git Bash）写好的代码/项目同步到远程 Linux 主机并验证构建与测试。当用户说「推到远程」「同步到 X 机器」「在远程跑一下测试」「远程编译验证」，或者需要在远程主机上落地一个本地编写的项目时使用。
agent_created: true
---

# 本地代码同步到远程并验证

Windows 上写代码、Linux 上跑——这是本机的常规工作方式。本文记录这条链路上的坑。

## 前置：修 PATH

本机 Git Bash 默认 PATH 缺 `dirname`，任何命令都会报
`dirname: command not found` 且部分命令静默失败。**每条 bash 命令开头必加**：

```bash
export PATH="/usr/bin:/bin:/c/Windows/System32:/c/Windows/System32/OpenSSH:$PATH"
```

## 同步：tar over ssh（不依赖 rsync/scp 配置）

在本机项目根目录执行：

```bash
cd /c/Users/MTL/WorkBuddy/<工作区>/<项目> || exit 1
tar czf - . | ssh -o BatchMode=yes -o ConnectTimeout=10 <host> \
  'mkdir -p ~/<项目> && tar xzf - -C ~/<项目>'
```

**注意**：本机与远程常有约 1 分钟时钟偏差，`tar` 会刷一片
`time stamp ... is N s in the future` 警告。**无害，可忽略**，
需要安静时在管道末尾加 `2>&1 | grep -v "time stamp"`。

只同步子目录（改单个模块时更快）：`tar czf - ./src | ssh <host> 'cd ~/<项目> && tar xzf -'`

## 必做：CRLF 修正

本机写出的 shell 脚本可能是 CRLF 换行，在 Linux 上会以
`bad interpreter: No such file or directory` 失败。同步后立即修：

```bash
ssh <host> 'cd ~/<项目> && sed -i "s/\r$//" *.sh && chmod +x *.sh'
```

## 验证：编译 + 测试一次跑完

零依赖 JDK 项目（无 Maven/Gradle，直接 `javac`）：

```bash
ssh <host> 'cd ~/<项目> && rm -rf out && mkdir -p out && \
  javac -encoding UTF-8 -d out $(find src -name "*.java") && \
  java -Dfile.encoding=UTF-8 -cp out <主类全限定名>'
```

中文源码必须同时给 `javac -encoding UTF-8` 与 `java -Dfile.encoding=UTF-8`，
否则输出乱码或编译报「不可映射字符」。

**优先做成 `run-tests.sh`**，远程只调一次，本地也能复用，避免把长命令塞进 ssh。

## 本机各远程主机的运行时（2026-09-17 核实）

- SGD-83：openjdk 25 + javac 25、python3 3.14.6（含 sqlite3）、git 2.53、docker 28.5。
  `mvn` / `node` **不在 PATH**，分别走项目内 `./mvnw` 与 `source ~/.nvm/nvm.sh`（v24.19.0）。
- 非交互式 ssh 不会加载 `.bashrc`，所以 nvm 里的 node 一定不可见。要用就先显式 source。

## 交付确认

同步后列一遍远端文件与行数，确认没有漏传、没有多传 `out/` 之类的产物：

```bash
ssh <host> 'cd ~/<项目> && find . -type f -not -path "./out/*" | sort | \
  while read f; do printf "%6s  %s\n" "$(wc -l < "$f")" "$f"; done'
```

## 硬约束

- **不改业务机器的运行中数据**。同步只写到用户家目录下的独立项目目录。
- 远程编译产物放 `out/`，不要写进仓库目录或其它项目目录。
- 首次往新主机推代码前，先把项目名与用户确认，避免撞上已有目录。
