---
name: legacy-test-suite-revival
description: 把坏掉的测试套件修复到全绿，并让 E2E / QA 脚本能在任意环境（含全新空库）重复执行。当用户说「测试跑不起来」「mvn test 失败」「让测试全绿」「修测试」「闭环测试」「新环境跑一遍流程」「QA 脚本只能在这台机器跑」时使用。特别适用于没有 Git 历史、测试与生产代码已经脱节的老项目。
agent_created: true
---

# 老项目测试复活与 E2E 可移植化

核心难点不是改代码，而是**判断「是测试过期，还是生产代码有缺陷」**。判错的代价不对称：
把缺陷当过期测试改绿，等于把 bug 埋进代码库并盖上「已验证」的章。

## 铁律：先建安全网，再动代码

没有 Git 历史的项目**必须先**：

```bash
tar czf archive/fix-baseline-$(date +%Y%m%dT%H%M%S).tar.gz --exclude=target --exclude=node_modules -C app .
cd app && git init -q && git add -A && git commit -m "baseline: 修复前基线"
```

提交前**必须**确认密钥与构建产物没进去（`git diff --cached --name-only | grep -E 'target/|node_modules|\.env$'`）。
很多老项目其实早有 `.gitignore`，只是从没初始化过仓库。

同时确认**测试是否连生产库**：查 `src/test/resources/application.yml` 的数据源、Testcontainers 配置，
并 `ss -tlnp | grep -E ":3306|:6379"` 确认宿主机无监听。不安全就先隔离，再跑。

## 第一步：拿到「去重后」的真实错误清单

Maven 的 `[ERROR]` 行会算上续行，**按文件路径统计会虚高**（实测把 12 处报成 24 处）。

```bash
grep '^\[ERROR\]' build.log | grep -E '\.java' | sort -u
```

## 第二步：给每个错误定性 —— 三种，处理方式完全相反

| 定性 | 判据 | 处理 |
|---|---|---|
| **测试过期** | 生产代码改名/改返回类型/加参数，测试没跟上 | 改测试，逐条对齐真实签名 |
| **测试夹具缺失** | 新增了前置校验或策略调用，mock 没 stub → 返回 null → NPE | **补 mock**，不要删校验 |
| **生产代码缺陷** | 见下节判据 | 改生产代码 |

定性证据优先级（从高到低）：

1. **方法签名的自相矛盾** —— 形参声明了却**完全没被使用**，或实现与自身 javadoc 相反。
   这是最强的缺陷信号（例：`canManage(ctx, orgId)` 里从没出现 `orgId`；
   `isQuarterHrOperator(quarter)` 里从没出现 `quarter`）。
2. **历史快照二分定位** —— 用项目里的 tar 快照/dump 对比改动前后，判断「测试先有还是代码先有」。
   - 测试在两个快照里都存在但一直失败 → **测试从没绿过**，是编码期望的规格，改代码。
   - 代码先有、测试后加 → 可能是追认缺陷，继续看第 3 点。
3. **项目自己的验收文档**（`VALIDATION-REPORT.md` 之类）—— 最接近权威规格。
4. **同一规格上的矛盾用例** —— 若同时存在 `xCan...` 与 `xCannot...` 两个用例，
   说明历史上被反向改过，**不要**单方面决定，列出双方证据交给用户。

### 动生产行为前，先 grep 那个行为的测试（血泪教训）

这条单独列出来，因为它造成的损失最大。

我按「形参未被使用」判定了 1 处真实缺陷后，顺着同一个特征去扫全仓，
得到 20+ 处「疑似同型」，差点一起改掉。逐站读完才发现在那些模块里：

```java
@Test void hrMayQueryAllOrganizations()      // 断言 HR 不传组织 = 不加过滤
@Test void hrMayRequestAnotherOrganization() // 断言 HR 传其他组织 = 放行
```

**这些测试名就是规格，而且写得非常清楚。** 我差点把它们当缺陷「修」掉。

规则：判定某行为是缺陷之前，先跑

```bash
grep -rn "<行为关键词>\|Cannot\|May" src/test/ | head -30
```

**只要有测试明确断言该行为应当存在，它就不是缺陷，而是规格** —— 哪怕它与
javadoc、与别的模块、与你的直觉都相反。此时正确动作是报告冲突，不是改代码。

更隐蔽的一种：**同一个类里 javadoc 与自己的测试相反**
（实测 `PublicNoticeQueryService` 的 javadoc 写 "an HR user is constrained to its own
organization"，而它自己的测试 `hrMaySelectAnotherOrganization` 断言 HR 跨组织放行）。
遇到这种，说明规格本身没收敛，**停下来出决策文档**。

### 矛盾的规格不是「多处写错」，而是「口径未收口」

同一类行为在 10 个模块里呈现 3 种不同规则、且每种都有测试背书时，
正确产出是：

1. 证据矩阵（每处列出：背书形式 = 测试名 / javadoc 原文 / 文档行号）
2. 指出**直接对照的矛盾对**（形状相同、期望相反的两个用例）
3. 给出选项与代价表（选 A 要改哪些、选 B 要改哪些）
4. 明确你现在**没有**改任何一侧，以及为什么

**这比"修了 22 处"有价值得多** —— 因为无论选哪个方向，单向收口都需要一次决策，
而改错方向的代价远高于等待。

## 第三步：修复一定要「最小且可解释」

- **只修有测试或 javadoc 背书的站点**。同一类错误往往被复制到几十处
  （实测：`!hasRole("HR")` 15 处 + 正向 `|| hasRole("HR")` 22 处），
  其中相当一部分是**有意设计**。盲改会打断合法业务。
  产出「修复清单」+「明确保留清单（含保留理由）」两份，后者的价值不亚于前者。
- 测试夹具的修复要**说明为什么这么填**（依据哪个 record 的字段顺序、哪个重载把谁置 null），
  不要靠试错凑绿。
- 一个新缺陷修好后，**可能暴露下一个被它掩盖的缺陷**（NPE 挡住了后面的
  `InvalidUseOfMatchersException`）。每轮修完必须重跑全量，不要只跑单文件。

## 第四步：让 E2E / QA 脚本可移植

典型症状：脚本断言一套**任何环境都不存在**的固定 ID。先验证再动手：

```bash
# 这些 ID 在目标环境真的存在吗？
SELECT COUNT(*) FROM members WHERE id IN (<脚本里写死的那批>);
```

**先找已有的引导脚本。** 很多项目其实已经有过一个可移植的身份准备脚本
（用保留号段、幂等、管理员从 `platform_admins` 推导），只是从没被 seed 脚本调用。
这时正确做法是**接入它**，而不是新写一个。

改造清单：

1. seed 的 SQL 只保留**变量赋值块**在写死 ID 上（通常就 6~7 行），把它改成保留号段 + 派生。
2. seed 的 shell 断言改为**对同一套身份集**校验，并在执行前幂等调用身份准备脚本。
3. 编排脚本（新增 `run_*_closure.sh`）：
   - 环境自检（容器 healthy + 后端 `/actuator/health`）
   - 迁移基线（`flyway_schema_history` 条数/最大版本/失败数）
   - 平台初始化（幂等：`initializationRequired` 为 true 才 POST）
   - 造身份 → 造数据 → 跑 verify，**逐项记 PASS/FAIL 并最后汇总**
   - **令牌密钥与 DB 口令从目标容器读**（`docker inspect -f '{{range .Config.Env}}...'`）写入临时文件，
     不依赖也不读取生产敏感环境文件
   - **破坏性用例（备份→重置→恢复）默认跳过**，用开关显式开启
4. 走一遍就走通是不可能的。**预期 3~5 轮**：每轮读失败点 → 定性 → 补丁 → 重跑。

### 顺序陷阱（踩过）

补前置步骤时，**一定要确认插在了被依赖调用的前面**。
实测把「结果方案确认」插到了「生成候选名单」之后，导致同一个 409 又出现一次，白跑一轮。
补完立刻打印目标片段核对顺序，别只看「补丁已应用」。

### 接口演进导致的缺失前置

老 E2E 脚本常在流程中途 409，报错信息本身就是规格，直接照做：

```
"请先由 HR 确认本季度每个奖项的结果方案，再生成候选名单"
  → PUT /quarters/{id}/result-awards/{quarterAwardId}
"请先由 HR 按复审得分生成本季度候选获奖名单"
  → POST /quarters/{id}/candidate-award-lists/actions/generate
```

配额要和脚本末尾的分布断言对齐（例：6 个项目 / 2 个奖项、期望 2/2/2 → 每奖项 1/1/1）。

## 验收：必须给出前后对照

```
mvn test-compile   BUILD FAILURE → BUILD SUCCESS
可执行测试         0 → 456
失败 + 错误        21 → 0
```

并且**全程核对隔离**：生产库表数 / 关键表行数 / 是否有 QA 前缀数据混入，
以及生产容器是否被重启。这些数字要写进报告，不能只说「没影响」。

## 附带排查：依赖与 JDK 版本不匹配（老项目常见）

测试能跑但日志里刷满告警时，先数告警条数再决定要不要管。判据是这一句：

```
Unsupported class file major version NN
```

`NN` 是 class file 主版本号：61=Java 17、65=21、68=24、**69=25**。
出现它说明某个字节码工具比运行用的 JDK 老一代。实测两例：

| 依赖 | 症状 | 处理 |
|---|---|---|
| `jacoco-maven-plugin` 0.8.12 | **1878 条** `IllegalClassFormatException`，2077 处 class file 69。JDK 平台类与 Mockito 在 JDK 25 上生成的 mock 类都无法插桩 —— **覆盖率数据被静默丢弃** | 升到 0.8.15 → 告警归零 |
| `archunit` 1.3.0 | `ClassFileProcessor` 读不了 Java 25，只做 `falling back to simple import`，**架构规则被静默弱化**（测试仍"通过"） | 升到 1.5.0 → 降级告警归零 |

**注意「静默弱化」比「报错」危险**：ArchUnit 的降级导入会让架构测试变成安慰剂。
升级后必须重跑该测试，确认它是真的通过。

查最新版：`https://repo1.maven.org/maven2/<groupId路径>/<artifactId>/maven-metadata.xml`。

## 别为了验证而在共享环境上启第二个应用实例

想端到端验证一个 HTTP 层改动时，如果镜像重建不了，很容易想到
「直接 `java -jar` 连现有的库」。**先想清楚副作用**：

- 定时任务会**重复执行**
- 通知/消息队列会**重复发送**（本项目有 `notification_outbox` 与企业微信集成）
- 迁移脚本可能重复执行

代价高于收益时，退回契约级验证并在报告里**明确说明这一层没验**，
比制造两个实例抢同一份副作用安全得多。

## 镜像重建能力要提前确认

改完源码要部署时才发现构建不了，是老项目的高频坑。检查顺序：

```bash
docker images | grep -E "maven|temurin|<基础镜像名>"   # 本地有没有基础镜像
docker pull <基础镜像>                                  # 能不能拉
```

要点：
- Dockerfile 首行 `# syntax=docker/dockerfile:1` 会让 BuildKit **额外去 Hub 拉前端镜像**；
  删掉该行可绕过（BuildKit 内置前端同样支持 `RUN --mount=type=cache`），
  但 `FROM` 那一层仍要能拉到基础镜像。
- Maven 能连 **不等于** Docker 镜像仓库能连 —— 两者常被不同的网络策略放行。
  实测该环境 Maven Central 通、`registry-1.docker.io` TCP 超时。
- 结论要写进报告：**「源码已改好但当前无法发布」本身就是必须上报的阻塞项。**

### 应急出镜像：叠层法（不需要任何 registry）

若旧镜像已经存在，且它的最后一层就是把 jar 复制进去，可以直接叠一层：

```bash
docker image inspect <旧镜像> --format "{{.Config.Entrypoint}} {{.Config.WorkingDir}}"
docker history <旧镜像> --no-trunc --format "{{.CreatedBy}}" | head -6

printf "FROM <旧镜像>\nCOPY target/<jar> /app/app.jar\n" > /tmp/Dockerfile.jarswap
docker build -f /tmp/Dockerfile.jarswap -t <新标签> <模块目录>
```

- 优点：秒级完成，可用来**验证源码、跑 E2E、出热修**
- 缺点：**不更新基础镜像的安全补丁**，永远以旧镜像为底座 —— 只能应急。
  正解是把基础镜像 `docker save` / `docker load` 搬进来（零停机）。

## 删除/清理路径：用 information_schema 做外键审计

E2E 测试开始跑「以前没人跑过的分支」时，最容易暴露的就是**清理清单漏表**。
现象通常是接口从 200 变 500：

```
Cannot delete or update a parent row: a foreign key constraint fails
  (<child_table>, CONSTRAINT <fk> FOREIGN KEY (...) REFERENCES <parent> (id))
```

**别只修报错的那一张**，把所有指向父表的子表一次性列出来再逐条对照：

```sql
SELECT TABLE_NAME, COLUMN_NAME, REFERENCED_TABLE_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = :db AND REFERENCED_TABLE_NAME IN ('projects','quarters')
ORDER BY TABLE_NAME;
```

然后核对三点：

1. **有没有整张表漏掉**（实测漏了 3 张）
2. **删除顺序对不对** —— 孙表必须早于子表、子表早于父表。
   例：某表同时引用 `quarters` 与 `quarter_awards`，就必须早于两者；
   而引用 `candidate_award_lists` 的两张表要先删，后者才能删。
3. **补一条顺序回归测试**（Mockito `InOrder` 即可，不必真连库）：

```java
InOrder order = inOrder(jdbcTemplate);
order.verify(jdbcTemplate).update(eq("DELETE FROM <child> ..."), any(Object[].class));
order.verify(jdbcTemplate).update(eq("DELETE FROM <parent> ..."), any(Object[].class));
```

**为什么这类缺陷能潜伏很久**：它只在「父表下存在子表数据」时才触发。
某张表在生产库里可能一直是 0 行，所以删除一直成功；
直到新的测试流程第一次生成了这类数据，缺陷才浮出来。
**这是测试补全的价值，不是新引入的 bug —— 报告里要这么写。**

### 不要顺手把这类 500 改成 409

`DataIntegrityViolationException` 按 HTTP 语义确实该是 409。但那个显眼的 **500 恰好
帮你抓到了漏删的表**；改成 409 后，将来再漏一张就变成静默的 409。修根因、保留 500 的可见性。

### 顺带一个会让结论出错的 shell 坑

```bash
probe() { curl ... ${4:+-H "Authorization: Bearer $TOKEN"} "$URL"; }
```

参数展开（`${var:+...}`）里的引号**不会被重新解析**，header 会被拆成多个参数，
令牌实际没带上 —— 于是把「未认证 401」误读成「404 未生效」。
验证类脚本里要**显式传** `-H`，不要在条件展开里拼参数。

## 改策略时：共享鉴权方法会有「隐式依赖」

当一条策略（例：「HR 可以查看另一个组织的数据」）需要落到共享的鉴权方法上时，
**不要只改方法本身，也不要只找出语义上属于该策略的调用点**。

实测踩到的坑：某服务只调了 `requireVisibleQuarter(id)` 来做组织校验，
而它内部走的是 `requireSameOrg`。把 `requireSameOrg` 放宽为「查看」语义后，
**这条写路径就静默失去了组织校验** —— 普通 HR 能修改其他组织的配置。
名字里带 "visible" 让人以为只影响读取，实际上它是唯一的检查点。

**正确做法：**

1. **先把调用点按读/写分类**，别按名字猜。写一个小脚本枚举所有调用点，
   打印其所属方法名，再据方法语义归类（`submit`/`issue`/`assign`/`archive`… vs `list`/`get`/`summary`…）。
2. 共享方法拆成**两个**：保留原名承载「查看」语义，新增 `...ForWrite` 承载「变更」语义。
3. **逐个核对原方法的调用点是否「依赖它附带做检查」**：

   ```bash
   grep -n "<sharedMethod>" src/main/java/**/*.java     # 拿到全部调用点
   ```

   对每个写路径问一句：它自己有没有独立的组织校验？没有就必须显式补。
   实测 5 个调用点里只有 1 个缺 —— 但那 1 个就是漏洞。
4. 混合用途的私有辅助（如 `requireManageable(quarter)` 同时被 `list` 和 `update` 调用）
   **不要改名**，而是保留读取语义，在写入口显式加一行 `requireSameOrgForWrite(...)`。

### 顺带：这类改动最容易撞上的两类测试假通过

- **裸 mock 不执行 lambda**：`mock(DistributedLockExecutor.class)` 会让
  `lockExecutor.execute(key, code, msg, supplier)` 直接返回 null，**lambda 体永不执行**。
  于是「断言 403」的用例可能因为压根没走到校验而"通过/失败"。修法是让 mock 同步执行传入的
  `Supplier`：`thenAnswer(inv -> ((Supplier<?>) inv.getArgument(3)).get())`。
- **桩打在被改名的旧方法上**：服务改调 `...ForWrite` 后，mock 未命中返回 null → NPE。
  逐个把桩改成新名（或在 setUp 里**补**一份新名的桩，读取路径的桩保留）。

## 报告结构

1. 结论摘要表（起点 → 终点）
2. 按定性分类的修复清单（生产缺陷 / 测试过期 / 夹具缺失）
3. **明确保留清单**及理由
4. 我自己判断错、后来修正的地方（老项目里规格常自相矛盾，如实写出来比装作笃定更有价值）
5. 未修项与原因
6. 回退命令
7. 新环境闭环的执行方式与结果
