# 值得分析的提交清单 · phosphor-pid-control

> **数据源**：本目录 [`commits.tsv`](commits.tsv) / [`commits.jsonl`](commits.jsonl) / [`commit-files.tsv`](commit-files.tsv)
> （443 个提交、1988 条「提交×文件」记录，2018-02-27 → 2026-10-21，51 位作者，无合并提交）
> **结构信号**：用 `git log --all --name-status` 补齐每提交的新增(A)/修改(M)/删除(D)文件数，与 churn 交叉。
> **生成时间**：2026-11-01。仓库当前 detached 在 `d801218`，`commit-history/` 未纳入版本控制。

---

## 0. 结论先行

| 处置 | 数量 | 说明 |
|---|---|---|
| **精读（S 级）** | 6 | 定义骨架或整体换代的提交，读 diff 全文 |
| **精读（A 级）** | 13 | 机制/语义转折点，读 diff + 追前后各 1 个提交 |
| **略读（B 级）** | 13 | 只读提交信息 + 受影响接口的函数签名（下表 12 行，构建换代两篇合一行） |
| **跳过** | 411 | 其中 68 个是格式化/许可/拼写/构建小修（占数量 15.3%，却吃掉 7220 行改动） |

比例约 **6.5% 值得精读**。这个仓库的分布极度长尾：中位数每提交只改 **26 行**，而 60 个提交 ≥200 行、16 个 ≥500 行——**信息密度集中在少数几个提交里，读错提交等于白读**。

---

## 1. 筛选规则（可复用到别的仓库）

**硬信号（满足其一进候选池）**

| 编号 | 判据 | 本仓库命中 | 含义 |
|---|---|---|---|
| R1 | 单提交新建文件 A≥3 或删除 D≥3 | 15 / 7 | 架构动作：加子系统 / 拆子系统 |
| R2 | churn（增+删）≥500 | 16 | 大手术，通常是换代而非修补 |
| R3 | 触及语义热点：`pid/ec/`（算法）、`conf.hpp`+`buildjson/`（配置 schema）、`failsafe`、并发、执行器写入路径 | 25 个碰 `conf.hpp` | 改这些 = 改系统契约 |
| R4 | 同时改 ≥3 个顶层目录 | 90 | 跨层改动，多为"接口搬家"；信号偏弱，只用于辅助排序 |
| R5 | 里程碑：构建系统、配置格式、命名空间、日志、许可证 | 人工判定 | 语境价值 > 代码价值，适合学"演化史" |

**反信号（命中即跳过，优先于硬信号）**

| 编号 | 判据 | 例证 |
|---|---|---|
| X1 | 主题命中格式化/许可/拼写/include 排序 | `da4a5dd` 2522 行，**只新增了 `.clang-format` 一个文件**——行数第一，信息量倒数第一 |
| X2 | churn ≤ 5 且只碰 1 个文件 | 多为编译修补、`chmod`（`7494ce0` churn=0，只改权限位） |
| X3 | 纯注释/文档排版 | 除非是设计文档（`cc74ae1` 是例外，见 A1） |

> ⚠️ 数据口径提醒：`commit-files.tsv` 生成时用了 `--no-renames`，**重命名会显示成"删+增"**（如 `22c257a` 里 `controller.cpp → pidcontroller.cpp`）。判断结构动作要结合 `git log --name-status` 的 `R` 记录，不能只看 tsv。

---

## 2. S 级：必读（6 个，定义骨架或整体换代）

### S1 · `d801218` — PID Objects & Algo · 2018-03-08 · Patrick Venture
- **规模**：+1463 / −0，16 个新文件（`pid/` 一次性成型）
- **信号**：A=16（全新建），占全仓库总改动约 3.5%
- **为什么必读**：整个系统的**算法 + 对象模型**都在这一刀里：`ec::pid` 的控制律、`PIDController` 的三钩子抽象（`input_proc`/`setpt_proc`/`output_proc`）、`PIDZone` 的缓存/failsafe/调度、配置装配。此后 8 年的所有提交都是在这个骨架上的增补与修补。
- **建议追问**：① 为什么把"风扇聚合策略（min）"写死，却把 PID 参数全部外置？② failsafe 的 90% 是"安全侧"还是"噪声侧"，谁定的？③ 抗饱和回算 `i = out − p` 为何不减前馈？
- **对照读**：`0e8fc39`（2022 补上 D 项）、`ba8ffa7`（2019 换掉配置格式）——看哪些决定活了 8 年，哪些 11 个月就被推翻。

### S2 · `863b924` — Sensor Objects · 2018-03-08 · Patrick Venture
- **规模**：+1297 / −0，26 个新文件（`dbus/`、`sysfs/`、`sensors/`、`notimpl/` 同时出现）
- **信号**：A=26，单提交建了 4 个子系统
- **为什么必读**：这是**依赖倒置第一次落地**——`Sensor` 接口 + `ReadInterface`/`WriteInterface` 分离 + 三套实现（D-Bus 被动读、sysfs 读写、`notimpl/` 空实现）。后面所有单元测试（123 个提交碰 `test/`）都建立在这层可替换性上。
- **建议追问**：为什么用"继承 + 空实现"而不是模板或函数对象？`notimpl` 的 `read()` 返回什么值——它直接决定 failsafe 会不会被触发。
- **对照读**：`5e92909`（把 builder 从 manager 拆出去）、`0df7c0f`（为测试引入接口）——看"可测性"是怎么一步步倒逼设计的。

### S3 · `e620656` — Scripts and main daemon · 2018-03-08 · Patrick Venture
- **规模**：+1582 / −0，30 个新文件（`main.cpp`、`Makefile.am`、`configure.ac`、`conf.hpp`、`examples/`、`scripts/*.py`）
- **信号**：与 S1/S2 同日，是补丁系列第 5 篇；首次「可运行」
- **为什么必读**：系统的**组装说明书**：线程模型（`threads/busthread`）、配置样例（`examples/swampd.conf`）、以及最值得玩味的一点——**用 Python + Mako 模板生成 C++ 代码**（`scripts/writepid.mako.cpp`、`zone_gen.py` 等），试图在编译期拿到配置的类型校验。这条路线 11 个月后被 JSON 解析取代。
- **建议追问**：代码生成 vs 运行期解析，各自的校验能力与代价？为什么放弃前者（提示：改配置要重编译，板厂不接受）？
- **系列读法**：`a788899`（初提交 202 行）→ `d801218` → `863b924` → `391b8b0`（IPMI 手动模式）→ `e620656`，**五个一起读**才算一个完整系统；单片不可编译，评审单位是 series 而非 commit。

### S4 · `7136a5a` — Add ability to configure from dbus · 2018-07-19 · James Feist
- **规模**：+560 / −1，10 个文件（新增 `dbus/dbusconfiguration.cpp` 341 行）
- **信号**：A=4；首次让"配置"脱离文件
- **为什么必读**：**部署模型的转折**：配置从"随镜像发的静态文件"变成"由 D-Bus 服务在运行时提供"。这既是 OpenBMC 生态的强约束（配置统一走 D-Bus），也埋下了 `1fe0895`（热重载）的根需求。
- **建议追问**：两套配置源并存期如何定优先级？D-Bus 不可用时降级到哪条路径？**配置的所有权归谁**（BMC 服务 vs 板厂）？

### S5 · `22c257a` — Add stepwise controller · 2018-08-31 · James Feist
- **规模**：+636 / −258，19 个文件（新增 `pid/ec/stepwise.*`、`pid/pidcontroller.*`）
- **信号**：A=5；首次出现"非 PID 控制律"
- **为什么必读**：这是对 S1 抽象的一次**压力测试**：三钩子抽象能不能容纳阶跃/查表控制器？结果是把基类从 `controller.*` 改名成 `pidcontroller.*`，抽象被"步骤控制器"重新划界——典型的"第二个用例逼出真抽象"。
- **建议追问**：Stepwise 与 PID 的共享点/差异点分别是什么？为什么不做成"同一个类的模式参数"，而要新开类？改名这个动作泄露了哪些早期设计假设？

### S6 · `ce6a3f3` — Remove threads · 2019-03-12 · James Feist
- **规模**：+167 / −197，9 个文件（删 `pid/pidthread.*`、`threads/busthread.*`，加 `pid/pidloop.*`）
- **信号**：D=4；`configure.ac` 新增 boost/asio 检查
- **为什么必读**：**并发模型整体换代**：从"每 zone 一个 `std::thread` + `sleep_for`"改成 `boost::asio::steady_timer` + `io_context` 事件循环。它同时改变了控制节拍、启动顺序、failsafe 时基——影响面远超行数。
- **建议追问**：定时器漂移/错过触发怎么办？原先"每 zone 一线程"的隔离性丢失后，zone 之间互相阻塞如何避免？这次改动是否改变了 S1 里 `ts`（采样周期）的语义？

---

## 3. A 级：强烈建议（13 个，机制与语义转折）

| # | 提交 | 日期 | 标题 | 规模 | 类型 | 值得分析的点 & 追问 |
|---|---|---|---|---|---|---|
| A1 | `cc74ae1` | 2018-10-10 | docs: Ported Design Doc to OpenBMC | +319 / −1，2 文件 | 文档 | **唯一的设计文档提交**（README 扩到 320 行 + 架构图）。语用/语境金矿：作者亲口写的设计意图，可与代码逐条对账。追问：文档里承诺的机制，代码里兑现了几个？ |
| A2 | `5c7cc54` | 2018-06-11 | pid: zone split out builders | +332 / −257 | 分层/可测性 | 把"构造"从"运行"里拆出来（`builder.*`），zone 才变得可单测。追问：拆完之后 zone 还剩哪些"不该它管"的职责？ |
| A3 | `a58197c` | 2018-06-11 | test: pid: zone | +486 / −37 | 测试体系 | 测试基建起点（配 gmock mock 掉传感器与 D-Bus）。追问：为了可测，生产代码被迫改了什么？mock 的边界是否暴露了耦合？ |
| A4 | `ba8ffa7` | 2019-02-11 | swap libconfig++ for json | +178 / −404 | 配置换代 | 库依赖、schema、错误校验全部换掉；删的比加的多，说明旧方案的设计被废弃。追问：JSON 带来了什么、丢失了什么（类型安全、注释、默认值）？ |
| A5 | `18b1311` | 2019-02-14 | build: drop support for YAML configurations | +107 / −527 | 删除 | 三天后删掉 YAML 与代码生成路线（D=13）。追问：一条路线要多久被判死刑（11 个月）？判据是什么？ |
| A6 | `1fe0895` | 2019-05-07 | Make dbusconfiguration reloadable without reboot | +161 / −112 | 生命周期 | 配置热重载：对象重建、状态迁移、zone 存活期与配置的关系。追问：重载瞬间 PID 的积分项/上一输出怎么处理？ |
| A7 | `98b704e` | 2019-06-03 | Add Fan Redundancy Support | +276 / −18 | 传感器语义 | 新增 `dbuspassiveredundancy`：一个"风扇"背后是双 tach（冗余）。追问：冗余读数的聚合策略（max/min/投票）是什么？与 S1 的 min 聚合如何叠加？ |
| A8 | `8f73ad7` | 2021-10-07 | Make specific UNA sensors not trigger failsafe | +484 / −4 | 监控语义 | "传感器读不到"从"一律 failsafe"变成"按配置分类"。追问：UNA/不可读/掉线的语义边界如何划分？这个补丁暴露了 S1 failsafe 定义的哪条漏洞？ |
| A9 | `0e8fc39` | 2022-10-05 | Support derivative term in PID algorithm… | +174 / −24 | 算法 | 4 年半后补齐 D 项，并把周期从 fan table 可配。追问：为什么当年敢叫 PID 却没有 D？补 D 之后抗饱和公式是否仍缺前馈项？ |
| A10 | `31058fd` | 2023-01-13 | Implementing the TempToMargin feature | +294 / −42 | 领域模型 | 引入"温度→裕量"变换，热控制器的输入语义变了（配合测试改动）。追问：变换放在哪一层（sensor 还是 controller）？为什么？ |
| A11 | `9788963` | 2023-11-06 | Support to accumulate PWM of different controllers for same sensor | +217 / −54 | 执行器冲突 | 多个 controller 写同一 PWM 时从"后写覆盖"变成"叠加"，并且要限幅。追问：叠加的上限由谁定？这是 `max` 仲裁之外的第二套冲突规则，二者会打架吗？ |
| A12 | `aadb30d` | 2020-08-10 | move dbus helper interface into its own file | +401 / −318 | 依赖方向 | 把 D-Bus 依赖从核心剥离成 `dbushelper_interface`，为后续"核心不依赖 D-Bus"铺路（配合 `7a98c19` 改用 `ZoneInterface*`）。追问：依赖方向修正后，单测里还剩哪些 D-Bus 泄漏？ |
| A13 | `de74542` | 2020-11-07 | Adding new feature of core PID loop logging | +533 / −13 | 可观测性 | 把日志埋进控制回路本身（`pid/ec/logging.*`），`ec::pid()` 增加 `name` 参数。追问：在 10Hz 回路里做 I/O 的代价与开关策略？日志字段如何支撑现场调参？ |

---

## 4. B 级：略读（13 个提交，只读提交信息 + 接口签名）

| 提交 | 日期 | 标题 | 一句话理由 |
|---|---|---|---|
| `a788899` | 2018-02-27 | Initial commit | 只有 LICENSE + README 空壳，作为时间线起点 |
| `391b8b0` | 2018-03-08 | The OEM IPMI interface for status | 手动模式的**外部控制面**，读接口即可 |
| `5e92909` | 2018-06-08 | sensors: split building from manager object | 与 A2 同构（构造/运行分离），看签名差异即可 |
| `5f59c0f` | 2018-11-11 | Move all floats to doubles | 精度决策，29 个文件的机械替换；只有"为什么是 now"值得看一眼 |
| `36b7d8e` | 2018-10-05 | [dbus-passive] Add threshold fan failure | 风扇失效判定阈值，是 A8 的前置 |
| `5426c34` | 2019-02-11 | add json verification for configurations | JSON schema 校验，配合 A4 读 |
| `7442c37` | 2019-02-11 | conf change: pid: all pid details are now camelCase | 配置字段命名规范化，机械但改变外部契约 |
| `3f0f7bc` | 2023-02-13 | Add MissingIsAcceptable feature to avoid failsafe | A8 的续篇：允许"缺失可接受" |
| `92f9f3c` | 2023-11-07 | Auto determine failsafe duty according sensor fail | failsafe 占空比自动推导，终结 S1 里"90% 硬编码" |
| `6df8bb5` | 2024-11-27 | Add failsafe logger for zones | 新的可观测性子系统（`failsafeloggers/`），与 A13 对照 |
| `aacd568` / `b1225b2` | 2022-12 | Remove autotools / Add meson | 构建系统换代（删 508 行），语境价值 |
| `b2a20e3` | 2026-10-21 | refactor: migrate to lg2 | 日志框架现代化，收尾阶段代表 |

---

## 5. 跳过清单（举例：为什么"行数多"不等于"值得读"）

| 提交 | 规模 | 实际内容 | 跳过理由 |
|---|---|---|---|
| `da4a5dd` | 2522 行，72 文件 | 新增 `.clang-format`（1 个文件）并**按其重排全仓库** | 全仓 churn 第一，语义改动为 0 |
| `bd63bca` | 723 行，29 文件 | clang-format 18 重新格式化 | 同上，X1 |
| `46a755f` | 513 行，30 文件 | 版权头换 SPDX | 许可证样板，X1 |
| `f8b6e55` | 383 行，86 文件 | "Fix includes" | 头文件整理，X1 |
| `d2768c5` | 212 行，35 文件 | clang-tidy 修复 | 静态检查清理，X1 |
| `7494ce0` | 0 行 | `bootstrap.sh` 加可执行位 | churn=0，X2 |

> 经验值：本仓库 68 个机械提交合计 7220 行改动，占全部 churn 的约 17%——**用行数排序找重点，会先撞上这批**。

---

## 6. 三条阅读路线（按目标选，不必全读）

**路线 A · 学设计（骨架如何长成）**
`a788899` → **S1 `d801218`** → **S2 `863b924`** → `391b8b0` → **S3 `e620656`** → A1 `cc74ae1`（对账设计文档）→ **S5 `22c257a`** → A2 `5c7cc54`
→ 收获：一个 PID 守护进程从 0 到可运行的全部关键决定。

**路线 B · 看演化（配置与部署模型）**
`e620656`（YAML + 代码生成）→ **S4 `7136a5a`**（搬到 D-Bus）→ A4 `ba8ffa7` → A5 `18b1311`（删 YAML）→ A6 `1fe0895`（热重载）→ `5426c34`（校验）→ `aacd568`（meson）
→ 收获：一条"配置方案试错 4 年"的完整证据链，适合做工程决策案例。

**路线 C · 看缺陷与修补（语义如何被现场需求拽出来）**
**S1 `d801218`**（failsafe 定义、无 D 项、抗饱和窟窿）→ `36b7d8e` → **A8 `8f73ad7`** → `3f0f7bc` → `92f9f3c` → **A9 `0e8fc39`**（补 D）→ **A11 `9788963`**（PWM 叠加）
→ 收获：同一个语义漏洞被分 5 次、跨 5 年修补——这就是"设计缺陷的寿命"的最好处方。

---

## 7. 统计附录

**按年**：2018:115 · 2019:90 · 2020:66 · 2021:21 · 2022:30 · 2023:33 · 2024:20 · 2025:44 · 2026:24

**规模分布**：中位数 26 行 / 均值 92.8 行；≥200 行 60 个；≥500 行 16 个；churn=0 的 1 个

**结构动作**：新建文件 ≥3 的提交 15 个；删除文件 ≥3 的 7 个——真正的"架构搬迁"只有二十来次

**模块热度**（触及该目录的提交数）：`dbus/` 160 · `pid/` 129 · `test/` 123 · `conf.hpp` 25 · `meson.build` 24

**机械提交**：68 个（15.3%），合计 7220 行

---

## 8. 复现与继续

```powershell
# 看单个提交的完整画像
git -C .. show --stat --summary <hash>
git -C .. diff-tree --no-commit-id --name-status -r <hash>

# 看某关键词的生死
git -C .. log --reverse --date=short --pretty=format:'%h %ad %s' -S'<keyword>' master -- .

# 按本清单筛表格
$want = Import-Csv .\commits-to-analyze.tsv -Delimiter "`t"
Import-Csv .\commits.tsv -Delimiter "`t" |
  Where-Object { $want.commit -contains $_.short_id } |
  Select-Object short_id, author_date, subject

# 只看 S/A 级
$want | Where-Object tier -ne 'B' | Select-Object tier, commit, churn, type, why
```

**下一步建议**：按路线 A 走一遍（约 2 天），每个 S 级提交产出「设计承诺卡 + 留白表 + 反事实表」；A 级只产出"一句话结论 + 一个可验证断言"。`commits-to-analyze.tsv` 是同名单的机器可读版本（15 列：tier / commit / date / author / subject / files / added / deleted / churn / A / M / D / type / why / ask），便于排序与检索。
