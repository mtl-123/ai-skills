---
name: docker-release-verify
description: 把源码改动安全发布成 docker 镜像并上线，重点解决「构建缓存导致产物不来自当前源码」「发布前没有回滚点」「上生产的镜像不是被测过的那个」三类陷阱。当用户说「部署到生产」「上线新版本」「重建镜像」「发布这次修复」「docker compose build 后重启」，或需要在无 Git/无 CI 的老项目里做可回滚的发布时使用。
agent_created: true
---

# 可回滚、可验证的 docker 发布

面向「没有 CI、基础镜像靠手工 load、发布靠人手敲命令」的项目。
核心不是把命令跑通，而是**证明上线的那个产物确实是你要上的那份**，并且随时能退回来。

## 第一原则：构建成功 ≠ 产物是新的

BuildKit 命中缓存时，`docker compose build` 可能在 **0.3 秒**内返回 `Built`，
输出里没有任何报错，但镜像还是上一版。日志里只有 `CACHED` 字样能看出来。

**发布前必须做产物校验，两条都要做：**

```bash
# 1) 强制重编编译层（保留运行阶段缓存，代价几分钟）
docker build --no-cache-filter=build -f backend/Dockerfile -t <repo>:<tag> .
#    日志里必须能看到 [build N/M] RUN ... 是跑过的，而不是 CACHED

# 2) 从镜像里把产物抽出来核对内容，不能只看时间戳
docker create --name probe-img <repo>:<tag> >/dev/null
docker cp probe-img:/app/app.jar /tmp/probe.jar && docker rm -f probe-img
python3 check_jar.py /tmp/probe.jar    # 见下节
```

容器里往往**没有 `unzip`**（`jre`/`alpine` 基础镜像都不带），所以要预先准备
一个用 `python3 -m zipfile` 或 `zipfile.ZipFile` 读的校验脚本。

### 产物校验脚本的写法

对每个本次修改点，取一个**必然出现在编译产物里的字面量**（新增的方法名、改动的常量、
新引用的类名），在 class 文件字节里搜：

```python
import zipfile
z = zipfile.ZipFile("/tmp/probe.jar")
data = z.read("BOOT-INF/classes/com/x/Y.class")
print(b"ForWrite" in data, z.getinfo("BOOT-INF/classes/com/x/Y.class").date_time)
```

要点：
- **必须有负向对照**：同时断言「本次已删除的东西不在里面」，否则"全部命中"可能只是脚本在自欺。
- 打印 `date_time` 确认是当天编译，避免旧 class 混入。
- 校验脚本以退出码表达结果，方便串进发布脚本。

## 第二原则：上生产的镜像，必须是被测过的那个镜像 ID

顺序固定：

1. 构建 → 得到镜像 ID（如 `555e292f…`）
2. 打一个候选标签 `repo:candidate-<短commit>`，把**隔离环境**指到它
3. 在隔离环境跑完整回归（端到端，不只单测）
4. 通过后，把同一个镜像推上线

**不要**「先在测试环境验证 A 镜像，再重新构建 B 镜像上线」——那等于没验证。

## 第三原则：回滚点是发布的一部分，不是可选项

发布脚本第一步必须做这四件事：

| 动作 | 目的 |
|---|---|
| `docker tag <当前镜像> <repo>:pre-deploy-<日期>` | 秒级切回（约 20 秒恢复） |
| 备份 compose / env 原文 | 配置层可回退 |
| 记录**数据基线数字**（表数、关键表行数、迁移版本） | 事后能逐项对照，而不是"看起来正常" |
| `mysqldump --single-transaction --routines --triggers` 全量落盘 | 数据层兜底 |

基线要和发布后的数字**逐项对照并打印在一起**，不要只打印新的。

## 第四原则：验证矩阵要覆盖协议层与数据层

只跑 `docker ps` 看到 healthy 是不够的。固定清单：

```bash
# 容器与镜像 ID（确认容器真的换了镜像）
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
docker inspect <容器> --format '{{.Image}}'

# 挂载源（数据目录必须仍在预期位置）
docker inspect <容器> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'

# 数据行数（对照基线）
# 启动日志（Flyway 版本 / 启动耗时 / ERROR 行数）
docker logs <容器> 2>&1 | grep -E "Started .*Application in|Current version of schema|No migration necessary|ERROR"
docker logs <容器> 2>&1 | grep -c ERROR

# 协议层：经真实入口（nginx）而不是容器直连
#   方法不匹配应 405、路径不存在应 404、Content-Type 不支持应 415、无令牌应 401
# 受权只读冒烟：带令牌 GET 一个真实业务列表接口，确认能取到数据
```

协议层的值要**经用户真实访问路径**（如 `http://host:5173/api/v1/...`）测，
因为生产覆盖文件常把后端端口从宿主机收回（`ports: !reset []`），直连会失败。

## 第五原则：不打印密钥，但可以做受权冒烟

需要管理员令牌时，把「派生 → 使用 → 清理」放在**同一个远端 shell** 里完成：

```bash
TMP=$(mktemp /tmp/env.XXXXXX); chmod 600 "$TMP"
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' <容器> | grep -E '^(JWT_SECRET|DB_PASSWORD)=' > "$TMP"
TOK=$(... 用该 env 派生 ...); rm -f "$TMP"    # 变量本身不 echo，只打印长度和 HTTP 状态码
```

只做 GET，不做任何写操作。

## 容易踩的坑

- **前端重建：不是"能不能重建"，而是"参数带齐没有"**——详见下一节。
  参数带不全比不重建危险得多：故障是静默的。源码确实变了就该重建，但要带参数 + 构完做产物对比。
- **镜像仓库不可达 ≠ Maven 不可达**：两条网络策略常常独立。基线镜像拉不下来时，
  可以先从可达的国内镜像源 `docker pull` 再 `docker tag` 成本地期望的名字。
  注意带后缀与不带后缀（`-noble` / 默认）是不同 tag，**两个都打上**才不会让 Dockerfile 解析失败。
- **`docker compose up -d --no-build` 只会重建镜像/配置有变化的服务**：
  实测只重建后端约 17 秒，MySQL/Redis/前端全程未重启，停机窗口远小于"整栈重启"。
- **永远不要 `docker compose down -v`**，也不要 `docker system prune --volumes`：
  机器上常存有早期版本遗留的**无主命名卷**，里面可能是某段历史数据。
  清理要按名字逐个来，且**先看内容**（`sudo ls -A /var/lib/docker/volumes/<卷>/_data`）再决定。
- **清理镜像优先于清理数据**：`docker image rm` 掉自己的实验产物（`docker system df` 看可回收量）
  比删历史数据安全得多。保留人工打的历史快照标签和最近一次 `pre-deploy-*`。
- **构建缓存不要随手 prune**：BuildKit 的 `--mount=type=cache,target=/root/.m2`
  会随构建缓存一起被清掉，之后每次构建都要重新拉全套 Maven 依赖。
- Windows 上写好的脚本推到 Linux 前先 `sed -i 's/\r$//'`，否则 `bash` 报
  `\r: command not found`。用 `tar czf - -C dir . | ssh host 'tar xzf - -C dest'` 传输最省事。

## 前端镜像：构建参数与产物对比（最容易静默出事的两处）

### 1. 构建期参数必须带齐，否则故障是静默的

很多前端把配置（企业微信 corp id、OAuth 回调域、API 基址）作为 **构建期** 注入
（`ARG` + `ENV` + `vite.config.ts` 里的 `define`）。漏传**不会报错**：
构建成功、页面照常打开、静态资源 200 —— 但登录/扫码功能整块不可用。

发布前固定做两件事：

```bash
# ① 从上一版镜像里反查参数实际值（产物里是明文内联的）
#    先确认上一版是否带上了参数：搜占位符
docker run --rm --entrypoint sh <镜像> -c \
  'grep -rl "replace_with\|REPLACE_" /usr/share/nginx/html || echo 无占位符'
#    再搜真实值形态（corp id 通常形如 ww + 12 位以上十六进制）
docker run --rm --entrypoint sh <镜像> -c \
  'grep -roh "ww[0-9a-zA-Z]\{12,\}" /usr/share/nginx/html/assets | sort -u'
# ② 把参数固化成文件（如 archive/<service>-build.env）并让构建脚本读它
```

新产物构建完，**再用同样两条命令验证一次**：真实值命中数 > 0、占位符命中 0。

### 2. 产物对比不能用文件名，要用「归一化内容指纹」

两个陷阱会同时出现：

- **残留堆积**：若历史镜像是用 `docker cp dist + docker commit` 增量更新的，
  目录合并会让旧 chunk 全部留下。实测某镜像 339 个文件里 **172 个是残留**
  （同一组件有 4~6 个不同 hash 的副本）。按文件名 diff，会把残留当成"线上仍在使用的功能"。
- **级联改 hash**：一个 chunk 变了，所有 import 它的 chunk 的引用字符串跟着变，
  于是大量文件"变了"，但内容其实没变（实测 36 个变化文件里 34 个属此类）。

正确做法：

1. 只保留 **`index.html` 的入口依赖图**（浏览器真正会加载的部分），排除残留；
2. 把形如 `-AbC123xY.js` 的 hash 段归一化为 `-HASH.js`，消除级联；
3. 用**归一化后的内容指纹**做集合比较 —— 与文件名完全无关。

```
✅ 内容完全相同: 166      🔴 旧有新无: 1      🟢 新有旧无: 5
```

判读：`旧有新无` 是唯一危险信号（功能消失），必须逐个人工确认；
`新有旧无` 多为字体/图片等非 js/css 资源（依赖图默认不跟踪），核对名字（=内容 hash）即可。

### 3. 不要用镜像 ID 判断「是否同一份代码」

`docker build` 与 `docker compose build` 产出的镜像 **ID 不同**，差异只在元数据
（provenance / config）。实测同一份源码：jar md5 相同、RootFS 8 层摘要相同，
但 `555e292f`（docker build）≠ `caf9937e`（compose build）。

判断是否同一份代码，要比 **产物文件 md5** 或 **`.RootFS.Layers` 摘要**：

```bash
docker image inspect <img> --format '{{.RootFS.Layers}}'
```



## 交付物（写进仓库，别只留在临时目录）

发布流程本身要固化进项目，否则下次又从零开始。建议落在 `demo/deploy/`：

```
demo/deploy/
├── README.md              流程说明 + 关键约束 + 回滚方式
├── 01_backup.sh           回滚点
├── check_jar.py           后端产物校验（含负向对照）
├── 02_deploy.sh           部署 + 验证矩阵（前后端联合判定）
├── 03_prod_smoke.sh       受权只读冒烟
├── 05_frontend_build.sh   前端构建（带齐构建期参数）+ 打回滚标签 + 抽旧产物
└── check_dist_diff.py     前端产物等价性校验（入口依赖图 + 归一化指纹）
```
