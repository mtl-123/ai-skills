---
name: docker-bind-mount-migration
description: 安全迁移被运行中容器 bind mount 的数据目录（改名、挪位置、重组目录结构），做到零数据丢失、属主权限不变。当用户说「把数据目录挪一下」「重组项目目录」「整合成一个项目目录」「docker 数据卷换路径」，或任何需要移动正在被容器挂载的目录时使用。
agent_created: true
---

# 迁移被容器挂载的数据目录

在**不停业务、不丢数据、权限不变**的前提下，把被运行中容器 bind mount 的目录挪到新位置。

## 第一原则：理解 bind mount 绑的是 inode，不是路径

这是整个任务的核心，也是唯一会让人误判「数据全丢了」的地方。

- **同文件系统内 `mv` 不影响正在运行的容器** —— 容器已经把 inode 绑住了，继续读写同一份数据。
- **但容器一旦重启/重建**，Docker 会按配置里的**旧路径重新解析**。
  旧路径不存在时，**Docker 会静默新建一个空目录挂上去**。
  MySQL 看到空数据目录 → 初始化一个全新空库 → 界面上业务数据全空。

**最危险的组合**：数据其实完好躺在新位置，但应用在空库上跑，此时若有人写入就出现两套数据。

**因此：移动后必须立刻让旧路径仍然可达（补软链），否则必须同步重建容器。中间不能留空档。**

## 执行前必须查清的 5 件事

```bash
# 1) 谁挂载了什么
for c in $(docker ps --format "{{.Names}}"); do
  echo "--- $c"; docker inspect "$c" --format "{{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}}{{println}}{{end}}"
done

# 2) 属主权限（必须原样保留，抄下来做基线）
stat -c "%n %U(%u):%G(%g) %a" <数据目录>/*/

# 3) 是否同一文件系统（决定 mv 是否安全）
stat -c "%n 设备=%d" ~ <源目录> <目标父目录>

# 4) 配置里的硬编码路径
grep -rn "旧路径" --include=*.yml --include=*.env --include=*.sh .

# 5) compose 项目名是否被钉死（决定容器名会不会变）
head -3 docker-compose.yml   # 看有没有 name: xxx
```

## 权限铁律

**只能用 `mv`。** 同一文件系统内 `mv` 是改名操作，属主、属组、权限位、时间戳一个都不变，且瞬间完成。

- **禁止 `cp` 不带 `-a`、禁止 `rsync` 不带 `-a`** —— 会重建成当前用户，
  容器内的 uid（常见 999）变成宿主 uid（常见 1000），MySQL 会**拒绝启动**。
- **不要用 `chown -R` 去「修正」权限**。数据目录上的奇怪权限（如 `1777`）往往有原因。
- 移动 root 属主的目录：`rename(2)` 只需**父目录**可写，不需要对目录本身有权限，
  但实践中可能仍被拒 → 用 `sudo -n mv`。

## 标准两阶段流程

### 阶段一：内容归位 + 兼容软链（零停机）

```bash
set -e
mkdir -p <新根>/archive
mv <数据目录> <新根>/data
ln -s <新根>/data <旧数据目录名>     # ★ 紧接上一行，不能拖
# ... 移动其余内容 ...
rmdir <旧项目目录> && ln -s <新根> <旧项目目录名>   # 旧脚本里写死的路径继续可用
```

做完立即验证容器与数据。此时**容器从未停止**，业务无中断。

### 阶段二：配置收敛（有停机，约 1 分钟）

```bash
cd <新根>/app
docker compose -f docker-compose.yml -f docker-compose.production.yml down -t 60   # ★ 绝不加 -v
sed -i "s#旧路径#新路径#g" docker-compose.production.yml
echo "INNOVATION_DATA_ROOT=<新根>/data" > .env     # 用 .env 固定，避免两个 compose 解析不一致
sed -i -e "s#旧路径#新路径#g" -e "s#旧项目路径#新根#g" <运维脚本>
rm <旧软链>
docker compose -f docker-compose.yml -f docker-compose.production.yml up -d --no-build
docker compose ... ps
```

**`-t 60`**：给数据库足够的优雅关闭时间，避免 SIGKILL 后走 InnoDB 恢复。
**`--no-build`**：镜像本地已有时不要重新构建。
**`.env` 优于改 defaults**：compose 自动加载项目目录下的 `.env`，
能让「只传主文件」和「主文件+生产覆盖」两种组合解析到同一个数据根。

## 别忘了运维脚本

移动路径后，用 `grep -rn "旧路径"` 全量搜一遍脚本。特别注意**带路径守卫**的脚本：

```bash
case "$DATA" in /home/xxx/old-data) ;; *) echo "Unexpected data root"; exit 1;; esac
```

路径一改就会直接拒绝执行。这是好的防御设计，但迁移时必须一起改。

## 验收清单

| 检查项 | 命令 | 期望 |
| --- | --- | --- |
| 容器 | `docker ps` | 全部 healthy |
| 挂载源 | `docker inspect <c> --format "{{range .Mounts}}{{.Source}}{{"\n"}}{{end}}"` | 指向新路径 |
| 属主 | `diff` 迁移前快照 | **完全一致** |
| 应用 | `curl` 首页 + actuator/health | 200 / UP |
| **数据行数** | 关键表 COUNT | **与迁移前逐项一致** |
| 旧路径残留 | `[ -L 旧路径 ]` | 已清除 |

## 硬约束

- **`docker compose down` 永远不加 `-v`**。`-v` 会删命名卷，
  机器上可能还留着早期版本产生的无主命名卷（几十到几百 MB），会连带蒸发。
- **迁移前一定做备份**：`mysqldump --single-transaction --databases <db> > dump.sql`，
  外加 compose 文件与权限快照。这是唯一的兜底。
- **发现「数据好像没了」时，第一件事是停栈，不是重启或重新初始化。**
  数据几乎不可能真丢，大概率是容器挂到了错误的空目录上。
- 用 `set -e` 时注意：脚本中途失败会留下**半完成状态**。
  失败后先做一次即时体检（容器/数据/权限），确认关键步骤已完成，再续做剩余步骤，
  **不要从头重跑**。
