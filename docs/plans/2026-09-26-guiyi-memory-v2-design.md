# 归忆 v2 设计（情感轨 · 程序性轨 · 离线巩固 · 实体消解）

日期：2026-09-26
状态：设计已确认，待实施
基线：本仓库 `main @ 2491e9c`
上游：`docs/plans/2026-09-25-guiyi-memory-design.md`（下称 **v1 设计**）；本设计是其增量，不重述 v1 已定内容
覆盖范围：仅本仓库（`guiyi` 包）；本体侧变更见第九章

---

## 一、目标与范围

**目标**

- 补全四轨中的**情感轨**与**程序性轨**（v1 只交付了情景与语义）。
- 交付**离线巩固**：从多条情景记忆中提炼模式。
- 交付**实体消解**：为 v3 的关系图多跳检索铺路。
- `Judge` 可选切换为 Jev（后端可换，协议不变）。

**非目标**

- 不做关系图多跳检索（v3）。
- 不引入 `sqlite-vec`（v3）。
- 不做多模态记忆。
- **不改动 v1 的架构、四条硬约束、三层作用域模型与错误哲学**。
- v2 **不强制**切换 Jev——先自测中文一致率，达标才切。

---

## 二、决策记录

| # | 议题 | 结论 | 被否决的方案与原因 |
| --- | --- | --- | --- |
| 1 | 情感「状态」如何落地 | 情绪**事件**进 `memory_item`；**当前状态**另存 `emotional_state` 表 | 只建状态表（丢掉情绪事件时间线）；复用 `memory_item` 以 `payload.state_key` 模拟（"当前值"要靠 supersede 链模拟，语义绕且查询需过滤） |
| 2 | `emotional_state` 内容 | 关系状态（`affinity` + `relationship_stage`）+ 双方当前情绪 | 只维护关系（无法感知"她今天不太开心"）；加"情绪趋势"（与巩固重复，趋势是派生量应算不应存）；加"对事物的态度"（**与语义轨偏好重复**，两处会打架） |
| 3 | `affinity` 驱动信号 | **只由用户对 Aliya 的明确态度驱动**（表扬/责备/感谢/亲昵）；用户自身情绪只进 `user_mood`；共同经历经互动量门槛影响阶段 | 把"用户倾诉痛苦"算作关系加分（把痛苦编码为加分项，可能产生"她越难受我们越亲近"的**病态偏置**，且该偏置不报错、只在长期相处中显形）；只保留离散档位（表达力弱、无法重算细微变化） |
| 4 | `affinity` 计算 | **饱和累积** `tanh(Σᵢ wᵢ·dᵢ / S)` + 阶段滞后 | 纯累加线性（会漂移到极值，单次事件无界）；EMA 指数滑动（状态只依赖上一步，**删记忆后无法重算**，与"派生层可重建"冲突，且无法审计） |
| 5 | 情绪聚合 | **二维连续量**（效价 + 强度）的**时间衰减加权聚合**，查询期计算、不落库 | 离散标签直接加权平均（**数学上不成立**）；取最新一条观测（被最后一句随口抱怨带偏，且与 v1 §5.1 的"滑动聚合"冲突）；窗口内简单平均（无法体现"越近越重要"） |
| 6 | `aliya_mood` 来源 | **本体传入优先**（`Utterance.mood` 可选）→ 缺失时抽取器反推 → 都没有则保持上次值 | 仅自我观察（归忆从文本**事后反推**生成侧的情绪，必然失真）；取消 `aliya_mood`（与决策 2 冲突） |
| 7 | 程序性 `trait` 表示 | **受控维度表**（初始 8 维）+ 明确祈使**即时生效** + 生效值**切换滞后** | 开放词表（近义维度必然碎片化，`语气词`/`语气词使用`/`口头禅` 各自累加，`top-N` 被同义项占满）；受控 + `misc` 自由槽（`misc` 迟早成垃圾场） |
| 8 | 巩固提炼产物的约束 | **受约束归纳**：只允许「偏好/事实」与「频次/趋势」两类输出；**禁人格断言与情绪评价**；每条派生必须可溯源；派生走同一套 `supersede` 判定 | 自由归纳（必然产出"她是焦虑的人"这类**人格标签**，入库后固化成偏见，且用户看不到、难纠正）；只产频次/趋势（最安全但巩固价值大减） |
| 9 | 巩固的调度归属 | **本体调度**，归忆只提供幂等的 `consolidate()` | 归忆内部起定时任务（破坏 v1 §3 硬约束"纯 Python 对象、无后台生命周期"；且本体已有优雅关闭机制，能正确取消任务） |
| 10 | 实体消解时机 | **本地信号写入即时 + 灰区按需 + 巩固复核** | 全部推到巩固（白天新别名要等一夜，期间表现为"你说的人我没印象"）；全部即时（每轮多一次出网调用，延迟与隐私成本叠加） |
| 11 | 实体合并策略 | 高置信度才自动 + `split_entity` 保证可逆 | 全自动（**误合并是隐私级损害**——把 A 的事说给 B 听，且不报错、难发现）；只做精确匹配（v3 图检索会因实体碎片化基本失效） |
| 12 | 派生关系存哪 | 独立 `derived_link` 表 | 塞进 `memory_relation.relation_type`（**内容语义关系**与**结构派生关系**混淆，会让 v3 图检索把派生边当内容边）；`memory_item` 加 JSON 数组列（回滚需按源反查，必须可索引） |
| 13 | 状态证据是否单独建表 | **不建**，直接复用 `kind='emotional'` 的条目 | 独立 `state_evidence` 表（与 `memory_item` 冗余，且可能两处不一致） |
| 14 | `relationship_stage` 的地位 | **派生缓存**，可随时由 `affinity` + 互动量重算 | 作为权威存储（会与证据不一致，制造排查噩梦） |
| 15 | 门面方法数 | 四 → **八**（新增 `split_entity` / `rollback_derived` / `get_state` / `procedural_block`） | 状态只经 `recall()` 返回（无法在没有检索意图时读状态，调试与界面展示不便） |
| 16 | 阈值与权重放哪 | **全部外置为 `MemorySettings` 配置项**，文档标注「初始值待实测调优」 | 写死在文档（它们本就是**未经验证的经验值**，写死会以"设计结论"的面目流传） |
| 17 | 写入的 scope 约束 | **程序性只写 `self`；情感状态只更新 `personal` 与 `self`；`group` 仅产出 `episodic`**（5.0 红线） | 不加约束（群成员可通过群消息投毒 Aliya 人设、或集体操纵某人的 `affinity`——跨 scope 污染且可被群体利用） |
| 18 | 回滚/拆分是否可被自动撤销 | **不可**：写 `veto_record` 终态否决，巩固先排除 | 只删派生记忆（下次巩固会**重新提炼出来**，用户回滚等于白做）；只把候选置 `rejected`（巩固复核灰区时会再次合并） |
| 19 | 巩固幂等的判定依据 | **源集合重叠率** ≥ `consolidate_idempotent_overlap` 即视为已提炼 | 靠 `cluster_key` 全等（新增一条成员后 key 变化 → 幂等失效 → 重复提炼）；`cluster_key` 仅保留作回滚追溯 |
| 20 | 情绪事件的归档 | 归档条件按「长期未访问 + 低重要度」判定，**不限 `kind`** | 只归档 `episodic`（情绪事件将**永不被归档**、无限增长，且与"重算纳入 archived"自相矛盾） |

---

## 三、架构变更

**模块树新增四个目录**

```text
guiyi/
  api.py       # 门面：新增 4 个方法，recall 返回扩展
  emo/         # 新增：情感状态读写、饱和累积、衰减、重算
  proc/        # 新增：程序性证据累加、top-N 选取、注入块生成
  entity/      # 新增：实体消解（归一化/别名/拼音/向量/共现/候选/拆分）
  forget/      # 扩充：巩固完整实现（聚类/提炼/归档/回滚/否决）
  judge/       # 扩充：新增 JevJudge（可选后端，实现同一协议）
```

依赖方向仍单向：`api → {retrieve, extract, judge, forget, emo, proc, entity} → store → schema`。

**门面：八方法**

| 方法 | 变化 | 说明 |
| --- | --- | --- |
| `remember` | 不变 | 写入一轮对话 |
| `recall` | **返回扩展** | 新增 `procedural_block`、`state_block`、`emotional_state` 三个字段 |
| `forget` | 不变 | 级联范围扩到十二张表 |
| `consolidate` | **从"只归档"变为完整实现** | 新增 `dry_run: bool = False` |
| `get_state` | **新增** | 读某 `personal` scope 的当前情感状态 |
| `procedural_block` | **新增** | 生成程序性注入块（`self` scope） |
| `split_entity` | **新增** | 拆分误合并的实体，并写 `veto_record` |
| `rollback_derived` | **新增** | 回滚派生产物、写 `veto_record`、重算受影响状态 |

**一致性约束**：`recall()` 内部**调用** `get_state()` 与 `procedural_block()`，不另写实现——否则迟早出现"界面显示的状态"与"实际注入的状态"不一致。

---

## 四、数据模型变更

**迁移是纯增量：新增 4 张表，无 ALTER、无索引变更。表数 8 → 12。**

| 新表 | 作用 | 关键字段 |
| --- | --- | --- |
| `emotional_state` | 情感状态。**只存在 `personal` 与 `self` 两种行，`group` 不建行**（列的适用 scope 见下） | `scope_type` / `scope_key`（联合主键）、`affinity`、`relationship_stage`、`stage_since`、`user_mood`、`user_mood_valence`、`user_mood_intensity`、`user_mood_updated_at`、`aliya_mood`、`aliya_mood_valence`、`aliya_mood_intensity`、`aliya_mood_updated_at`、`interaction_count`、`updated_at` |
| `derived_link` | 派生 → 源 的双向链接，支撑"一键回滚" | `derived_memory_id`、`source_memory_id`（联合主键）、`cluster_key`、`created_at` |
| `veto_record` | **终态否决记录**，防止巩固自动撤销用户的回滚/拆分 | `kind`(`consolidate_cluster` / `entity_merge`)、`target_key`、`reason`、`created_at`（联合主键 `kind` + `target_key`） |
| `entity_merge_candidate` | 待复核的实体合并候选 | `left_entity_id`、`right_entity_id`（联合主键）、`confidence`、`signals` JSON、`status`(pending/accepted/**rejected 为终态**)、`created_at`、`decided_at` |

**列的适用 scope**（避免"每 scope 一行"与内部量粒度不匹配）：

| 列 | 适用 scope |
| --- | --- |
| `affinity` / `relationship_stage` / `stage_since` / `interaction_count` | `personal` |
| `user_mood` / `user_mood_valence` / `user_mood_intensity` / `user_mood_updated_at` | `personal` |
| `aliya_mood` / `aliya_mood_valence` / `aliya_mood_intensity` / `aliya_mood_updated_at` | `self`（**全局唯一一行**） |

非适用 scope 的列为 `NULL`；写入时由代码校验，违反即抛 `EmotionError`。

**四个关键决策**

1. **不建"状态证据"表**。`affinity` 的证据直接是 `memory_item` 中 `kind='emotional'` 的条目（`payload.target` + `payload.valence` + `payload.intensity`）。重算 = 重新聚合这些条目，天然满足"删源即可重算"。重算纳入 `status IN ('active','archived')` 的事件——**归档只影响检索，不改变"这件事发生过"**。
2. **`relationship_stage` 是派生缓存，不是权威数据**。权威只有 `affinity`、`interaction_count` 与证据；阶段由档位门槛 + 滞后算出后缓存，`stage_since` 供滞后判定。
3. **派生关系用 `derived_link`，不塞进 `memory_relation`**。`relation_type` 描述**内容语义**关系（语义/时间/因果/实体），而"由谁派生"是**结构**关系；混在一起会让 v3 图检索把派生边当成内容边。
4. **否决必须落库为终态**。`veto_record` 是"用户已明确纠正过"的持久记录，巩固与消解**先读它再决定**；它只能由人工/显式接口清除，**系统不得自动过期**。

**迁移**

- `PRAGMA user_version` 1 → 2；**纯增量建 4 张表**。
- 在 `start()` 内**单事务**执行；库版本高于代码已知版本、或迁移失败 → **拒绝启动**（沿用 v1 §12.3 定案）。
- **不支持降级**：版本超前即拒绝启动，降级需人工处理（先备份，再手工脚本或重建）。
- 新增 `backup_before_migrate`（默认 `true`）：迁移前自动 `VACUUM INTO` 备份一次。

---

## 五、v2 数据流

### 5.0 作用域约束（安全红线，先于以下所有链路）

| 输入 scope | 允许产出的 `kind` |
| --- | --- |
| `personal` | `episodic` / `semantic` / `emotional` / `procedural` |
| `self` | `episodic` / `semantic` / `procedural` |
| `group` | **仅 `episodic`** |

三条硬规则：

1. **程序性记忆只写 `self` scope**，全局一份，**不按参与者或会话分片**。
2. **情感状态只更新 `personal`（对某个具体的人）与 `self`（`aliya_mood` 全局一行）**；`group` 不建 `emotional_state` 行。
3. **群消息不得更新任何情感状态或程序性记忆**——否则群成员可投毒 Aliya 人设（人设是固定注入的，影响面全局），或集体操纵某人的 `affinity`（跨 scope 污染）。

违反即抛 `ScopeError`，与 v1 "默认拒绝"一致。

### 5.1 情感状态更新（写入时）

**情绪统一表示为二维连续量**：`valence ∈ [-1, 1]`（效价）+ `intensity ∈ [0, 1]`（强度）。情绪**标签**收敛到受控词表（见 5.1.1），每个标签带默认效价；标签是**派生展示值**，聚合运算一律在连续量上进行。

**`affinity` 只看「用户对 Aliya 的态度」**：

| 情形 | 对 `affinity` 的影响 |
| --- | --- |
| `target=aliya`，效价为正（表扬 / 感谢 / 亲昵） | 上升 |
| `target=aliya`，效价为负（责备 / 表达不满） | 下降 |
| `target=user`（用户讲述自己的遭遇） | **不影响**，只进 `user_mood` |
| `target=third_party`（提到第三方） | **不影响任何状态**，只作情景记录 |

`affinity = tanh(Σᵢ wᵢ·dᵢ / S)`，其中 `dᵢ = valenceᵢ × intensityᵢ`（**仅 `target=aliya` 的事件计入**），`wᵢ` 为时间衰减权重，`S` 为饱和尺度。共同经历通过**互动量门槛**影响 `relationship_stage`，不直接进 `affinity`。

**当前情绪 = 二维时间衰减加权聚合**（**不是取最新一条**）：

- `valence = Σᵢ wᵢ·vᵢ / Σᵢ wᵢ`，`intensity = Σᵢ wᵢ·iᵢ / Σᵢ wᵢ`，`wᵢ = 0.5^(Δtᵢ / mood_half_life_hours)`
- 窗口为最近 `mood_window_hours`（默认 24h）
- **查询期计算、不落库**（与 v1"排序衰减不落库"一致）
- 标签由聚合后的 `valence` 分档得出：`|valence| < mood_neutral_threshold` → `neutral`；否则取窗口内**该极性下 `wᵢ·intensityᵢ` 最大**的情绪标签

**`aliya_mood` 来源分层**：① 本体传入的 `Utterance.mood` / `mood_intensity` 优先 ② 缺失时抽取器从 `role="assistant"` 文本反推 ③ 两者都无则**保持上次值不更新**。写入 `self` scope 的 `emotional_state` 行。

**`relationship_stage`** 四档：`陌生` → `熟悉` → `亲近` → `亲密`。**升级**需 `affinity ≥ 该档上阈值` **且** `interaction_count ≥ 该档互动量门槛`；**降级**需 `affinity ≤ 下阈值`（滞后，防抖动）。档位与阈值见配置 `stage_levels`。

`interaction_count` 语义：**每次 `remember()` 成功提交一轮对话，对相应 `personal` scope +1**（同一轮内多条 utterance 只计 1）。

`recompute_state(scope)` 是**唯一重算入口**，供 `forget()` / `rollback_derived()` 调用。

#### 5.1.1 受控情绪词表

情绪标签收敛为受控集合（代码常量，随版本演进），每个带默认效价：

| 标签 | 效价 | 标签 | 效价 |
| --- | --- | --- | --- |
| 开心 | +0.8 | 难过 | −0.7 |
| 感动 | +0.7 | 生气 | −0.8 |
| 期待 | +0.5 | 焦虑 | −0.6 |
| 亲昵 | +0.6 | 委屈 | −0.6 |
| 平静 | 0.0 | 疲惫 | −0.4 |
| 无聊 | −0.2 | 害怕 | −0.7 |

抽取器产出词表外的标签 → 归一到 `neutral`（不丢弃，因为情绪事件本身仍应留存）。**词表新增标签属代码变更**，与 `trait` 维度同性质。

### 5.2 程序性证据累加与注入

- 抽取器产出 `kind='procedural'` 的 draft（**仅 `self` scope**），维度必须落在**受控维度表**内，否则**丢弃**（不入库）：
  `称呼方式`、`语气词`、`句长偏好`、`emoji/颜文字`、`话题回避`、`回复节奏`、`幽默程度`、`主动发问频率`
- 取值受控（如 `常用` / `中性` / `避免`）；同维度不同取值各自累积证据，**生效值取权重最高者**（风格可并存，不做 supersede）
- **权重不对称**：`user_feedback` > `self_observation`（具体数值见配置，标注待实测）——用户的直接反馈远比自我观察可靠
- **明确祈使即时生效**：带明确祈使的 `user_feedback`（如"以后别叫我宝贝"）**一次即置为生效值并给高权重**，不等累加。受 5.0 红线约束，**该通道只接受 `personal` scope 的私聊输入**
- **切换滞后**：生效值切换需新取值权重超出当前生效值 `procedural_switch_hysteresis` 才发生，防抖动
- 注入 `procedural_block()`：取权重 top-N（`procedural_top_n`），`pinned` 无条件优先且**不占 N 名额**；按 `procedural_token_budget` 截断
- **排序依据落在 `payload.evidence_weight`（JSON，无索引）**：因受控维度有限，程序性条目数量有天然上限（≤ 维度数 × 取值数），**全表扫描 + Python 排序的代价可忽略**——这是不为其加列/索引的依据

### 5.3 巩固 `consolidate()`（幂等、可中断）

1. **扫描**：按 scope 分批取 `kind IN ('episodic', 'emotional')` 的 `active`/`archived` 候选（`consolidate_batch` 限流）。**扫描前先取一次快照**：本次只处理快照内的条目，期间新增/修改的留给下次——巩固不阻塞 `remember`，两者共用 v1 的全局写锁（排队而非破坏）
2. **排除否决**：**先读 `veto_record` 与 `entity_merge_candidate.status='rejected'`，命中的簇/实体对直接跳过**
3. **聚类**：向量相似度 ≥ `consolidate_cluster_similarity` + 同 scope
4. **门槛**：簇内 ≥ `consolidate_min_cluster` 条、跨 ≥ `consolidate_min_sessions` 个会话、时间跨度 ≥ `consolidate_min_span_days` 天
5. **提炼**：每簇一次 LLM 调用，产出 1 条 `semantic` 记忆（受 `consolidate_max_llm_calls` 每夜上限约束）
6. **落库**：写派生记忆，`derived_link` 连到每个源并记 `cluster_key`
7. **归档**：满足「长期未访问 + 低重要度」的条目置 `archived`，**不限 `kind`**（情绪事件同样可归档，但归档后仍计入状态重算）
8. **重算**：重算受影响 scope 的 `emotional_state`

**提炼产物的硬约束**（由代码校验，不能交给模型自觉）：

- 只允许两类输出：① 偏好 / 事实（如"喜欢猫"）② 频次 / 趋势描述（如"近一个月多次提到加班"）
- **禁止人格断言与情绪评价**（不得产出"焦虑""冷漠""工作狂"这类标签）
- 产出必须能列出支撑它的**源 id 列表**（`derived_link` 已具备）
- 派生语义记忆**必须与已有语义记忆走同一套 `supersede` 判定**，否则会出现"即时抽取说 A、巩固说 not A"的两条 active

**幂等**：不以 `cluster_key` 全等判定（新增成员会让 key 变化），而是**比对源集合重叠率**——若某簇与既有派生记忆的源集合**重叠 ≥ `consolidate_idempotent_overlap`**，视为已提炼，跳过。`cluster_key` 仅用于回滚追溯。`dry_run=true` 只报告不落库。**单个簇的处理必须在一个事务内**，不允许半落库。

### 5.4 实体消解

**分层，按信号成本分配时机**：

| 时机 | 信号 | 动作 |
| --- | --- | --- |
| 写入时（本地，零出网） | 归一化后精确相等、别名表命中、**向量相似度 ≥ `entity_vector_merge_threshold`** | **自动归并**（先查 `veto_record`） |
| 写入时（出网，按需） | 向量落在**灰区**（`entity_gray_zone`） | 调一次 `Judge.holds` 判定；受 `entity_judge_max_per_turn` 限制；达标才合并 |
| 巩固时（批量） | 拼音 / 向量 / `Judge.holds` | 清理遗留候选、复核灰区；达标自动合并，否则写 `entity_merge_candidate`(pending) |

**信号叠加共现上下文**：判断两个称呼是否同一实体时，除名字相似度外还计入**共现**（是否出现在同 scope、相近时间段、相近话题），权重 `entity_cooccurrence_weight`。单纯看名字不可靠。

**否决优先**：任何阶段合并前都要查 `veto_record`；`split_entity` 写入的否决对**永不被自动撤销**。

`split_entity(entity_id, alias)`：把别名拆成新实体、重挂 `memory_entity`、将相关候选置 `rejected`（终态）、写 `veto_record(kind='entity_merge', target_key=规范化排序后的实体对)`。

---

## 六、错误处理

延续 v1 §九 的"区分致命与可降级"，v2 新增链路的表态：

| 链路 | 态度 | 理由 |
| --- | --- | --- |
| 情感状态更新失败 | **可降级** | 状态可由证据重算，下次重算即修复；不阻塞对话 |
| 程序性注入失败 | **可降级** | 返回空块，对话继续 |
| 巩固失败 | **可降级且幂等** | 下次接着跑；但**单个簇必须在一个事务内**，不可半落库 |
| 实体合并 | **不可"降级为随便合并"** | 误合并是隐私级损害；`Judge` 不可用时灰区一律留候选，宁漏不误 |
| `split_entity` / `rollback_derived` | **绝不降级** | 与 `forget()` 同级——都在纠正错误的记忆；失败需重试，且必须写入 `veto_record` |
| `Judge` 不可用 | **回落 `RuleJudge`** | 见下方降级与恢复规则 |

**`Judge` 降级与恢复**：连续 `judge_failure_threshold`（默认 3）次失败（超时 `judge_timeout_ms` 默认 800ms，或 HTTP 错误）→ 进入降级态，改用 `RuleJudge`；降级态下每 `judge_probe_interval_min`（默认 30 分钟）**探测一次**，成功即恢复。`local_only=true` 时为**永久降级**，不探测。降级期间实体消解的灰区**一律留候选**。

**新增异常**：`ConsolidateError`（巩固阶段失败，可重试）、`EntityError`（非法操作，如拆解不存在的别名）、`EmotionError`（状态列与 scope 不匹配）。

---

## 七、测试策略

沿用 v1 §十 的手段（假 `Embedder` / `Extractor` / `Judge`，`:memory:` SQLite，全程不触网）。**v2 新增 11 条，编号 16–26**：

| # | 用例 | 断言 |
| --- | --- | --- |
| 16 | 情感符号 | `target=user` 的负面事件 → `affinity` **不变**；`target=aliya` 的负面事件 → 下降 |
| 17 | 噪声抑制 | 连续 10 条平静 + 1 条低强度抱怨 → `|Δvalence| < 0.05`（**断言变化幅度，不写"不变"**，避免与 half_life 参数耦合） |
| 18 | **状态可重算** | 删掉某条情绪事件后重算，结果与"从未写入该事件"一致；**重算两次结果逐位相同**（幂等）；**打乱证据顺序结果不变**（顺序无关） |
| 19 | 程序性即时生效 | 「别叫我宝贝」一次即置为生效值 |
| 20 | 受控维度 | 抽取器产出未知维度 → 丢弃，不入库 |
| 21 | 人格断言拦截 | 提炼产出「她是焦虑的人」→ 被规则校验丢弃 |
| 22 | 派生可回滚 | `rollback_derived` 后派生消失、源情景记忆仍在、状态已重算、**`veto_record` 已写入** |
| 23 | 巩固幂等 | 同一簇跑两次只产生一条派生；**向该簇新增一条成员后再跑，仍不产生第二条**（验证重叠率判定） |
| 24 | 实体可拆 | `split_entity` 后两实体独立、`memory_entity` 重新挂载、候选置 `rejected` |
| 25 | 灰区不误合并 | `Judge` 不可用时灰区留候选，不合并 |
| 26 | `Judge` 降级与恢复 | 连续 3 次失败 → 降级为 `RuleJudge`；探测成功后恢复为 `JevJudge` |
| 27 | **scope 红线** | `group` scope 的输入产出 `kind='procedural'` 或 `emotional` → 被拒（`ScopeError`），状态不变 |
| 28 | **否决终态** | 回滚某簇后再次执行巩固 → 跳过该簇，不产生派生 |

（原计划 11 条，因补入 scope 红线与否决终态两条，**实为 13 条，编号 16–28**。）

---

## 八、配置项（v2 新增）

**全部阈值与权重均为外置配置，且标注「初始值待实测调优」——它们是未经验证的经验值，不得以"设计结论"对待。**

| 配置项 | 默认 | 标注 |
| --- | --- | --- |
| `mood_window_hours` | `24` | 待实测 |
| `mood_half_life_hours` | `6` | 待实测 |
| `mood_neutral_threshold` | `0.2` | 待实测 |
| `affinity_scale`（公式中的 `S`） | `10` | 待实测 |
| `stage_levels`（档位、双阈值、互动量门槛表） | 见代码常量初始值 | 待实测 |
| `procedural_top_n` | `8` | — |
| `procedural_token_budget` | `200` | — |
| `procedural_weight_user_feedback` | `3.0` | 待实测 |
| `procedural_weight_self_observation` | `1.0` | 待实测 |
| `procedural_switch_hysteresis` | `0.2` | 待实测 |
| `state_block_token_budget` | `200` | — |
| `injection_total_budget` | `1600` | 待实测 |
| `consolidate_batch` | `200` | — |
| `consolidate_cluster_similarity` | `0.82` | 待实测 |
| `consolidate_min_cluster` | `3` | 待实测 |
| `consolidate_min_sessions` | `2` | 待实测 |
| `consolidate_min_span_days` | `7` | 待实测 |
| `consolidate_max_llm_calls` | `200` | 待实测（每夜上限） |
| `consolidate_idempotent_overlap` | `0.8` | 待实测 |
| `entity_vector_merge_threshold` | `0.95` | 待实测 |
| `entity_gray_zone` | `[0.85, 0.95]` | 待实测 |
| `entity_merge_confidence` | `0.95` | 待实测 |
| `entity_judge_max_per_turn` | `1` | 待实测 |
| `entity_cooccurrence_weight` | `0.3` | 待实测 |
| `judge_timeout_ms` | `800` | 待实测 |
| `judge_failure_threshold` | `3` | 待实测 |
| `judge_probe_interval_min` | `30` | 待实测 |
| `backup_before_migrate` | `true` | — |

（表内 28 行；其中 `stage_levels` 以配置结构承载多值。`consolidate_cron` 为 v1 已有项，不重复计入。）

**三块注入内容的预算与拼接**

- `prompt_block` ≤ `recall_token_budget`（v1，1200）；`procedural_block` ≤ `procedural_token_budget`（200）；`state_block` ≤ `state_block_token_budget`（200）
- **归忆保证三块合计 ≤ `injection_total_budget`**；超出时按 **`prompt_block` → `procedural_block` → `state_block` 的顺序削减**（先砍记忆，最后砍情绪——情绪块最小且对语气影响最直接）
- **拼接由本体完成**，建议顺序：`state_block` → `procedural_block` → `prompt_block`

**依赖清单变更（v2）**

- 运行时新增：`pypinyin`（纯 Python，体积小，供实体消解拼音层）
- 其余不变：`pydantic>=2`、`numpy`、`onnxruntime`、`loguru>=0.7`
- 仍**明确不引入**：torch、langchain、任何 web 框架、任何向量数据库 SDK

---

## 九、本体侧契约变更

> 本章只列契约与清单，不展开实现。

### 9.1 归忆导出增量

**新增四方法**：`get_state` / `procedural_block` / `split_entity` / `rollback_derived`。`recall()` 返回值新增 `procedural_block`、`state_block`、`emotional_state`。

**新增类型**（字段是契约的一部分，必须冻结）：

```python
class Mood(StrEnum):                    # 受控情绪词表，见 5.1.1
    HAPPY = "开心"; MOVED = "感动"; EAGER = "期待"; AFFECTIONATE = "亲昵"
    CALM = "平静"; BORED = "无聊"
    SAD = "难过"; ANGRY = "生气"; ANXIOUS = "焦虑"
    WRONGED = "委屈"; TIRED = "疲惫"; AFRAID = "害怕"
    NEUTRAL = "neutral"


class RelationshipStage(StrEnum):
    STRANGER = "陌生"; ACQUAINTED = "熟悉"; CLOSE = "亲近"; INTIMATE = "亲密"


@dataclass(frozen=True)
class EmotionalState:                   # personal scope
    scope: Scope
    affinity: float                     # -1..1
    relationship_stage: RelationshipStage
    stage_since: datetime
    user_mood: Mood                     # 派生展示值
    user_mood_valence: float            # -1..1
    user_mood_intensity: float          # 0..1
    interaction_count: int
    updated_at: datetime


@dataclass(frozen=True)
class AliyaMood:                        # self scope（全局唯一）
    mood: Mood
    valence: float
    intensity: float
    updated_at: datetime


class TraitDimension(StrEnum):          # 受控维度表，见 5.2
    ADDRESS_STYLE = "称呼方式"; TONE_PARTICLES = "语气词"; SENTENCE_LENGTH = "句长偏好"
    EMOJI = "emoji/颜文字"; TOPIC_AVOID = "话题回避"; REPLY_PACE = "回复节奏"
    HUMOR = "幽默程度"; PROACTIVE_ASK = "主动发问频率"


@dataclass(frozen=True)
class ProceduralTrait:                  # self scope
    dimension: TraitDimension
    value: str                          # 受控取值：常用 / 中性 / 避免 …
    evidence_weight: float
    pinned: bool


@dataclass(frozen=True)
class ConsolidateReport:
    scanned: int
    excluded_by_veto: int               # 被 veto_record 排除
    clustered: int
    distilled: int
    skipped_overlap: int                # 因重叠率幂等而跳过
    archived: int
    merged: int
    superseded: int
    llm_calls: int
    dry_run: bool
```

**新增异常**：`ConsolidateError`、`EntityError`、`EmotionError`。

### 9.2 本体改动清单（在 v1 的 6 处之外）

| 文件 | 动作 |
| --- | --- |
| `core/service/memory_service.py` | 新增定时任务：读 `consolidate_cron`，到点调 `MemoryService.consolidate()`；随 Service 生命周期启停与取消 |
| `core/service/memory_service.py` | 组装 prompt 时取用 `recall()` 返回的 `procedural_block` 与 `state_block`，按 §八 的顺序与总预算拼接 |
| 回复生成侧 | 在 `Utterance` 上填可选 `mood` / `mood_intensity`（不填也能跑，归忆会降级） |
| `core/config/settings.py` | `MemorySettings` 增 v2 的 28 项配置 |
| `data/config/app.yaml` | 补对应默认段 |
| 依赖 | `pyproject.toml` 增 `pypinyin` |

### 9.3 契约约束

- `consolidate()` 与定时任务的**调度全部在本体**；归忆不持有后台任务。
- `split_entity` / `rollback_derived` **同样需要调用方鉴权**（与 `forget()` 同级）。
- **`group` scope 的调用不得写入情感状态与程序性记忆**（5.0 红线），归忆会拒；本体不应依赖捕获异常来兜底，而应在调用前就不构造该输入。
- 参数一律经 `MemorySettings` 传入，归忆不读环境变量。

---

## 十、风险与缓解

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| **巩固产出人格标签** | 偏见固化，用户看不到、难纠正 | 受约束归纳（只两类输出）+ 禁人格断言 + 规则二次校验（5.3） |
| **误合并实体** | 隐私级损害（把 A 的事说给 B 听），且不报错、难发现 | 高置信度才自动 + 灰区留候选 + `split_entity` 可逆 + `veto_record` 终态（5.4） |
| **群消息投毒人设 / 操纵关系值** | 人设全局影响；`affinity` 被群体操纵 | 5.0 作用域红线：`group` 只产 `episodic`；程序性只写 `self`；情感状态只受 `personal`/`self` 影响 |
| **回滚被巩固自动撤销** | 用户纠错无效，"删了又生" | `veto_record` 终态 + 巩固扫描前排除（决策 18） |
| **情感的病态偏置** | "她越难受关系越近"，长期显形 | `affinity` 只由 `target=aliya` 的事件驱动（决策 3、5.1） |
| **刷分** | 反复说"我最喜欢你"抬高 `affinity` | 饱和函数限制上限 + 同因子每日计数上限 + 即时生效通道**仅对明确祈使开放**且仅限私聊 |
| 巩固成本随数据量增长 | 每夜 LLM 调用线性增长 | `consolidate_max_llm_calls` 上限 + 快照分批 + 按优先级（新近、高重要度优先） |
| **v2 迁移不支持降级** | 回退版本需人工处理 | `backup_before_migrate` 自动备份；版本超前即拒绝启动并提示处理方式 |

---

## 十一、验收标准（v2 增量，承接 v1 的 1–9）

10. **可重算性**：删任一条情绪事件后重算，`affinity` 与 `user_mood_valence` 与"从未写入该事件"一致；重算两次结果逐位相同。
11. **可溯源**：巩固产出的每条派生记忆都能列出全部源 `memory_id`。
12. **可逆**：`rollback_derived` 后源情景记忆完好、受影响状态已重算，且**再次执行巩固不会重建该派生**。
13. **可拆**：`split_entity` 后无残留挂载、相关候选为终态 `rejected`，且**再次执行巩固/消解不会重新合并**。
14. **幂等**：巩固连续执行两次，不产生重复派生记忆；向已提炼的簇新增成员后再执行，仍不产生第二条。
15. **作用域红线**：`group` scope 的输入**不产生任何情感状态变化与程序性记忆**（自动化断言）。
16. **成本上限**：对 1 万条情景记忆执行一次 `consolidate()`，LLM 调用次数 ≤ `consolidate_max_llm_calls`，且可在 `dry_run` 下完整报告而不落库。

---

## 十二、实施顺序建议

| 切片 | 内容 | 说明 |
| --- | --- | --- |
| **v2a** | 程序性轨：受控维度表 + 证据累加 + 即时生效 + top-N 注入 | 最独立、风险最低，不依赖新表（复用 `memory_item`；排序依据见 5.2 末条） |
| **v2b** | 情感轨：`emotional_state` 表 + 二维情绪 + 饱和累积 + 状态重算 | 引入第一张新表与 `get_state`；必须同时落地 5.0 红线 |
| **v2c** | 巩固：聚类 + 受约束提炼 + `derived_link` + `veto_record` + `rollback_derived` | 依赖 v2b 的状态重算入口 |
| **v2d** | 实体消解：分层信号 + 灰区按需 + `entity_merge_candidate` + `split_entity` | 依赖 `Judge` 与向量 |
| **v2e**（可选） | `Judge` 切 Jev | 前置条件：自测中文一致率达标；否则不启动 |

**v2 的取舍**：v2a/v2b 可先交付并独立验收——情感与程序性不需要巩固就能生效；巩固与消解价值更高但复杂度也更高，放在后面。

---

## 十三、与 v1 的差异清单

实施时需同步回改 v1 设计的以下位置（**以免两版文档矛盾**）：

| v1 位置 | v2 起的变化 |
| --- | --- |
| §3「唯一门面」四个方法 | → **八个方法**（新增 `get_state` / `procedural_block` / `split_entity` / `rollback_derived`） |
| §4 八张表 | → **十二张表**（新增 `emotional_state` / `derived_link` / `veto_record` / `entity_merge_candidate`） |
| §5.1 情感轨写入语义 | 措辞"状态覆盖 + 滑动聚合"**保持有效**；v2 明确了二维连续量、聚合公式与不落库 |
| §5.3 与 §15.1「v1 的 `consolidate()` 只做归档」 | v2 起为**完整实现** |
| §9.1 错误处理表 | v2 起增 6 条链路态度（5 条 → 11 条） |
| §10 测试策略 10 条 | v2 起增 **13** 条（共 23 条） |
| §12.1 配置项 **25** 行 | v2 起增 **28** 行 |
| §12.2 依赖清单 | v2 起增 `pypinyin` |
| §12.1 `enabled_kinds` 默认值 | v1 默认 `["episodic","semantic"]` → v2 起默认 **四轨全开** |
| §12.1 `type_quota` | v1 四轨 40/35/15/10 → v2 起为**只覆盖参与检索的三类**：`episodic 40 / semantic 45 / emotional 15`（`procedural` 不参与检索，见 5.2） |
| §13 风险表 12 条 | v2 起增 8 条 |
| §14 验收标准 9 条 | v2 起增 7 条（共 16 条） |
| §14 第 4 条「八张表零残留」 | v2 起为**十二张表** |
| §11.3 契约约束 | v2 起增"调度在本体""`split_entity`/`rollback_derived` 需鉴权""`group` scope 不得写入情感与程序性" |
| §15.2 Jev 评估 | v2 起进入**可选实施**（v2e），前置自测中文一致率 |

---

*本设计由头脑风暴流程逐段确认后固化；经一轮完整审查补入 5 条 P0、12 条 P1 与全部计数修正。所有阈值均为初始值，须以实测数据调优；实施中若发现与 v1 或仓库实际代码冲突，以代码为准并回改本文档。*
