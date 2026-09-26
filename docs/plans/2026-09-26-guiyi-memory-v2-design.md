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
| 5 | 情绪聚合 | **时间衰减加权聚合**，24h 窗口，**查询期计算、不落库** | 取最新一条观测（被最后一句随口抱怨带偏，且与 v1 §5.1 的"滑动聚合"冲突）；窗口内简单平均（无法体现"越近越重要"） |
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

---

## 三、架构变更

**模块树新增四个目录**

```text
guiyi/
  api.py       # 门面：新增 2 个方法，recall 返回扩展
  emo/         # 新增：情感状态读写、饱和累积、衰减、重算
  proc/        # 新增：程序性证据累加、top-N 选取、注入块生成
  entity/      # 新增：实体消解（归一化/别名/拼音/向量/共现/候选/拆分）
  forget/      # 扩充：巩固完整实现（聚类/提炼/归档/回滚）
  judge/       # 扩充：新增 JevJudge（可选后端，实现同一协议）
```

依赖方向仍单向：`api → {retrieve, extract, judge, forget, emo, proc, entity} → store → schema`。

**门面：八方法**

| 方法 | 变化 | 说明 |
| --- | --- | --- |
| `remember` | 不变 | 写入一轮对话 |
| `recall` | **返回扩展** | 新增 `procedural_block`、`state_block`、`emotional_state` 三个字段 |
| `forget` | 不变 | 级联范围扩到十一张表 |
| `consolidate` | **从"只归档"变为完整实现** | 新增 `dry_run: bool = False` |
| `get_state` | **新增** | 读某 scope 的当前情感状态 |
| `procedural_block` | **新增** | 生成程序性注入块 |
| `split_entity` | **新增** | 拆分误合并的实体 |
| `rollback_derived` | **新增** | 回滚派生产物并重算受影响状态 |

**一致性约束**：`recall()` 内部**调用** `get_state()` 与 `procedural_block()`，不另写实现——否则迟早出现"界面显示的状态"与"实际注入的状态"不一致。

**新增依赖**：`pypinyin`（纯 Python，体积小），供实体消解的拼音层使用。

---

## 四、数据模型变更

**迁移是纯增量：新增 3 张表，无 ALTER、无索引变更。表数 8 → 11。**

| 新表 | 作用 | 关键字段 |
| --- | --- | --- |
| `emotional_state` | 情感状态，**每 scope 一行** | `scope_type` / `scope_key`（联合主键）、`affinity` REAL、`relationship_stage` TEXT、`stage_since`、`user_mood`、`user_mood_intensity`、`user_mood_updated_at`、`aliya_mood`、`aliya_mood_intensity`、`aliya_mood_updated_at`、`interaction_count`、`updated_at` |
| `derived_link` | 派生 → 源 的双向链接，支撑"一键回滚" | `derived_memory_id`、`source_memory_id`（联合主键）、`cluster_key`、`created_at` |
| `entity_merge_candidate` | 待复核的实体合并候选 | `left_entity_id`、`right_entity_id`（联合主键）、`confidence`、`signals` JSON、`status`(pending/accepted/rejected)、`created_at`、`decided_at` |

**三个关键决策**

1. **不建"状态证据"表**。`affinity` 的证据直接是 `memory_item` 中 `kind='emotional'` 的条目（`payload.intensity` + `payload.target`）。重算 = 重新聚合这些条目，天然满足"删源即可重算"，且少一张表。重算纳入 `status IN ('active','archived')` 的事件——**归档只影响检索，不改变"这件事发生过"**。
2. **`relationship_stage` 是派生缓存，不是权威数据**。权威只有 `affinity` 与证据；阶段由双阈值 + 滞后算出后缓存，`stage_since` 供滞后判定。这点必须写明，否则将来会出现"状态表与证据对不上"的排查噩梦。
3. **派生关系用 `derived_link`，不塞进 `memory_relation`**。`relation_type` 描述**内容语义**关系（语义/时间/因果/实体），而"由谁派生"是**结构**关系；混在一起会让 v3 图检索把派生边当成内容边。

**迁移**

- `PRAGMA user_version` 1 → 2；**纯增量建 3 张表**。
- 在 `start()` 内**单事务**执行；库版本高于代码已知版本、或迁移失败 → **拒绝启动**（沿用 v1 §12.3 定案）。
- **不支持降级**：版本超前即拒绝启动，降级需人工处理（先备份，再手工脚本或重建）。
- 新增 `backup_before_migrate`（默认 `true`）：迁移前自动 `VACUUM INTO` 备份一次。

---

## 五、v2 数据流

### 5.1 情感状态更新（写入时）

抽取器产出 `kind='emotional'` 的 draft 后，更新该 scope 的 `emotional_state`。

**`affinity` 只看「用户对 Aliya 的态度」**：

| 情形 | 对 `affinity` 的影响 |
| --- | --- |
| 用户表扬 / 感谢 / 亲昵（`target=aliya`，正性） | 上升 |
| 用户责备 / 表达不满（`target=aliya`，负性） | 下降 |
| 用户讲述自己的遭遇（`target=user`） | **不影响**，只进 `user_mood` |

`affinity = tanh(Σᵢ wᵢ·dᵢ / S)`，`dᵢ` 为第 i 条事件对关系的贡献（带符号），`S` 为饱和尺度。共同经历通过**互动量门槛**影响 `relationship_stage`，不直接进 `affinity`。

**当前情绪 = 时间衰减加权聚合**（**不是取最新一条**）：

- `mood = Σᵢ wᵢ·dᵢ / Σᵢ wᵢ`，`wᵢ = 0.5^(Δtᵢ / mood_half_life_hours)`
- 窗口为最近 `mood_window_hours`（默认 24h）
- **查询期计算、不落库**（与 v1"排序衰减不落库"一致）
- 聚合强度低于 `mood_neutral_threshold` → 输出**中性情绪** `neutral`（不是"平静"这类自造状态）

**`aliya_mood` 来源分层**：① 本体传入的 `Utterance.mood` / `mood_intensity` 优先 ② 缺失时抽取器从 `role="assistant"` 文本反推 ③ 两者都无则**保持上次值不更新**。

**`relationship_stage`** 由 `affinity` + `interaction_count` 派生，带**滞后**：升级需 ≥ 上阈值、降级需 ≤ 下阈值，避免阈值附近抖动。

`recompute_state(scope)` 是**唯一重算入口**，供 `forget()` / `rollback_derived()` 调用。

### 5.2 程序性证据累加与注入

- 抽取器产出 `kind='procedural'` 的 draft，维度必须落在**受控维度表**内，否则**丢弃**（不入库）：
  `称呼方式`、`语气词`、`句长偏好`、`emoji/颜文字`、`话题回避`、`回复节奏`、`幽默程度`、`主动发问频率`
- 取值受控（如 `常用` / `中性` / `避免`）；同维度不同取值各自累积证据，**生效值取权重最高者**（风格可并存，不做 supersede）
- **权重不对称**：`user_feedback` > `self_observation`（具体数值见配置，标注待实测）——用户的直接反馈远比自我观察可靠
- **明确祈使即时生效**：带明确祈使的 `user_feedback`（如"以后别叫我宝贝"）**一次即置为生效值并给高权重**，不等累加
- **切换滞后**：生效值切换需新取值权重超出当前生效值 `procedural_switch_hysteresis` 才发生，防抖动
- 注入 `procedural_block()`：取权重 top-N（`procedural_top_n`），`pinned` 无条件优先且**不占 N 名额**；按 `procedural_token_budget` 截断

### 5.3 巩固 `consolidate()`（幂等、可中断）

1. **扫描**：按 scope 分批取 `kind='episodic'` 的 `active`/`archived` 候选（`consolidate_batch` 限流）
2. **聚类**：向量相似度 ≥ `consolidate_cluster_similarity` + 同 scope
3. **门槛**：簇内 ≥ `consolidate_min_cluster` 条、跨 ≥ `consolidate_min_sessions` 个会话、时间跨度 ≥ `consolidate_min_span_days` 天
4. **提炼**：每簇一次 LLM 调用，产出 1 条 `semantic` 记忆（受 `consolidate_max_llm_calls` 每夜上限约束）
5. **落库**：写派生记忆，`derived_link` 连到每个源
6. **归档**：满足条件的 episodic 置 `archived`
7. **重算**：重算受影响 scope 的 `emotional_state`

**提炼产物的硬约束**（由代码校验，不能交给模型自觉）：

- 只允许两类输出：① 偏好 / 事实（如"喜欢猫"）② 频次 / 趋势描述（如"近一个月多次提到加班"）
- **禁止人格断言与情绪评价**（不得产出"焦虑""冷漠""工作狂"这类标签）
- 产出必须能列出支撑它的**源 id 列表**（`derived_link` 已具备）
- 派生语义记忆**必须与已有语义记忆走同一套 `supersede` 判定**，否则会出现"即时抽取说 A、巩固说 not A"的两条 active

**幂等**：簇内已有派生记忆（经 `derived_link` 的 `cluster_key` 可查）→ 跳过。`dry_run=true` 只报告不落库。**单个簇的处理必须在一个事务内**，不允许半落库。

### 5.4 实体消解

**分层，按信号成本分配时机**：

| 时机 | 信号 | 动作 |
| --- | --- | --- |
| 写入时（本地，零出网） | 归一化后精确相等、别名表命中、**向量相似度 ≥ `entity_vector_merge_threshold`** | **自动归并** |
| 写入时（出网，按需） | 向量落在**灰区**（`entity_gray_zone`） | 调一次 `Judge.holds` 判定；受 `entity_judge_max_per_turn` 限制；达标才合并 |
| 巩固时（批量） | 拼音 / 向量 / `Judge.holds` | 清理遗留候选、复核灰区；达标自动合并，否则写 `entity_merge_candidate`(pending) |

**信号叠加共现上下文**：判断两个称呼是否同一实体时，除名字相似度外还计入**共现**（是否出现在同 scope、相近时间段、相近话题）——单纯看名字不可靠。

`split_entity(entity_id, alias)`：把别名拆成新实体、重挂 `memory_entity`、将相关候选置 `rejected`。

---

## 六、错误处理

延续 v1 §九 的"区分致命与可降级"，v2 新增链路的表态：

| 链路 | 态度 | 理由 |
| --- | --- | --- |
| 情感状态更新失败 | **可降级** | 状态可由证据重算，下次重算即修复；不阻塞对话 |
| 程序性注入失败 | **可降级** | 返回空块，对话继续 |
| 巩固失败 | **可降级且幂等** | 下次接着跑；但**单个簇必须在一个事务内**，不可半落库 |
| 实体合并 | **不可"降级为随便合并"** | 误合并是隐私级损害；`Judge` 不可用时灰区一律留候选，宁漏不误 |
| `split_entity` / `rollback_derived` | **绝不降级** | 与 `forget()` 同级——都在纠正错误的记忆 |
| `Judge` 不可用 | **回落 `RuleJudge`** | 功能不中断；Jev 挂掉或 `local_only=true` 时自动降级 |

**新增异常**：`ConsolidateError`（巩固阶段失败，可重试）、`EntityError`（非法操作，如拆解不存在的别名）。

---

## 七、测试策略

沿用 v1 §十 的手段（假 `Embedder` / `Extractor` / `Judge`，`:memory:` SQLite，全程不触网）。**v2 新增 11 条，编号 16–26**：

| # | 用例 | 断言 |
| --- | --- | --- |
| 16 | 情感符号 | 用户倾诉负面情绪 → `affinity` **不变**；用户对 Aliya 表达不满 → 下降 |
| 17 | 噪声抑制 | 连续 10 条平静 + 1 条低强度抱怨 → `user_mood` 不变（加权聚合压住噪声） |
| 18 | **状态可重算** | 删掉某条情绪事件后重算，结果与"从未写入该事件"一致 |
| 19 | 程序性即时生效 | 「别叫我宝贝」一次即置为生效值 |
| 20 | 受控维度 | 抽取器产出未知维度 → 丢弃，不入库 |
| 21 | 人格断言拦截 | 提炼产出「她是焦虑的人」→ 被规则校验丢弃 |
| 22 | 派生可回滚 | `rollback_derived` 后派生消失、源情景记忆仍在、状态已重算 |
| 23 | 巩固幂等 | 同一簇跑两次只产生一条派生 |
| 24 | 实体可拆 | `split_entity` 后两实体独立、`memory_entity` 重新挂载、候选置 `rejected` |
| 25 | 灰区不误合并 | `Judge` 不可用时灰区留候选，不合并 |
| 26 | `Judge` 降级 | `Judge` 抛错 → `RuleJudge` 接管，功能不中断 |

---

## 八、配置项（v2 新增）

**全部阈值与权重均为外置配置，且标注「初始值待实测调优」——它们是未经验证的经验值，不得以"设计结论"对待。**

| 配置项 | 默认 | 标注 |
| --- | --- | --- |
| `mood_window_hours` | `24` | 待实测 |
| `mood_half_life_hours` | `6` | 待实测 |
| `mood_neutral_threshold` | `0.2` | 待实测 |
| `affinity_scale`（公式中的 `S`） | `10` | 待实测 |
| `stage_up_threshold` / `stage_down_threshold` | `0.6` / `0.45`（滞后双阈值） | 待实测 |
| `procedural_top_n` | `8` | — |
| `procedural_token_budget` | `200` | — |
| `procedural_weight_user_feedback` | `3.0` | 待实测 |
| `procedural_weight_self_observation` | `1.0` | 待实测 |
| `procedural_switch_hysteresis` | `0.2` | 待实测 |
| `consolidate_batch` | `200` | — |
| `consolidate_cluster_similarity` | `0.82` | 待实测 |
| `consolidate_min_cluster` | `3` | 待实测 |
| `consolidate_min_sessions` | `2` | 待实测 |
| `consolidate_min_span_days` | `7` | 待实测 |
| `consolidate_max_llm_calls` | `200` | 待实测（每夜上限） |
| `consolidate_cron` | 已有（v1） | 由本体读取并调度 |
| `entity_vector_merge_threshold` | `0.95` | 待实测 |
| `entity_gray_zone` | `[0.85, 0.95]` | 待实测 |
| `entity_merge_confidence` | `0.95` | 待实测 |
| `entity_judge_max_per_turn` | `1` | 待实测 |
| `backup_before_migrate` | `true` | — |

---

## 九、本体侧契约变更

> 本章只列契约与清单，不展开实现。

### 9.1 归忆导出增量

- `MemoryEngine` 新增四方法：`get_state` / `procedural_block` / `split_entity` / `rollback_derived`
- `recall()` 返回值新增 `procedural_block`、`state_block`、`emotional_state`
- 新增类型：`EmotionalState`、`ProceduralTrait`、`ConsolidateReport` 扩展字段
- 新增异常：`ConsolidateError`、`EntityError`

### 9.2 本体改动清单（在 v1 的 6 处之外）

| 文件 | 动作 |
| --- | --- |
| `core/service/memory_service.py` | 新增定时任务：读 `consolidate_cron`，到点调 `MemoryService.consolidate()`；随 Service 生命周期启停与取消 |
| `core/service/memory_service.py` | 组装 prompt 时取用 `recall()` 返回的 `procedural_block` 与 `state_block` |
| 回复生成侧 | 在 `Utterance` 上填可选 `mood` / `mood_intensity`（不填也能跑，归忆会降级） |
| `core/config/settings.py` | `MemorySettings` 增 v2 的 22 项配置 |
| `data/config/app.yaml` | 补对应默认段 |

### 9.3 契约约束

- `consolidate()` 与两个定时任务相关的**调度全部在本体**；归忆不持有后台任务。
- `split_entity` / `rollback_derived` **同样需要调用方鉴权**（与 `forget()` 同级）。
- 参数一律经 `MemorySettings` 传入，归忆不读环境变量。

---

## 十、风险与缓解

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| **巩固产出人格标签** | 偏见固化，用户看不到、难纠正 | 受约束归纳（只两类输出）+ 禁人格断言 + 规则二次校验（七） |
| **误合并实体** | 隐私级损害（把 A 的事说给 B 听），且不报错、难发现 | 高置信度才自动 + 灰区留候选 + `split_entity` 可逆（5.4） |
| **情感的病态偏置** | "她越难受关系越近"，长期显形 | `affinity` 只由用户对 Aliya 的明确态度驱动（决策 3） |
| **刷分** | 反复说"我最喜欢你"抬高 `affinity` | 饱和函数限制上限 + 同因子每日计数上限 + 即时生效通道**仅对明确祈使开放** |
| 巩固成本随数据量增长 | 每夜 LLM 调用线性增长 | `consolidate_max_llm_calls` 上限 + 分批 + 按优先级（新近、高重要度优先） |
| **v2 迁移不支持降级** | 回退版本需人工处理 | `backup_before_migrate` 自动备份；版本超前即拒绝启动并提示处理方式 |

---

## 十一、验收标准（v2 增量，承接 v1 的 1–9）

10. **可重算性**：删任一条情绪事件后重算，`affinity` 与 `mood` 与"从未写入该事件"一致。
11. **可溯源**：巩固产出的每条派生记忆都能列出全部源 `memory_id`。
12. **可逆**：`rollback_derived` 后源情景记忆完好，且受影响状态已重算。
13. **可拆**：`split_entity` 后无残留挂载，相关候选置 `rejected`。
14. **幂等**：巩固连续执行两次，不产生重复派生记忆。

---

## 十二、实施顺序建议

| 切片 | 内容 | 说明 |
| --- | --- | --- |
| **v2a** | 程序性轨：受控维度表 + 证据累加 + 即时生效 + top-N 注入 | 最独立、风险最低，不依赖新表（复用 `memory_item`） |
| **v2b** | 情感轨：`emotional_state` 表 + 饱和累积 + 加权聚合 + 状态重算 | 引入第一张新表与 `get_state` |
| **v2c** | 巩固：聚类 + 受约束提炼 + `derived_link` + `rollback_derived` | 依赖 v2b 的状态重算入口 |
| **v2d** | 实体消解：分层信号 + 灰区按需 + `entity_merge_candidate` + `split_entity` | 依赖 `Judge` 与向量 |
| **v2e**（可选） | `Judge` 切 Jev | 前置条件：自测中文一致率达标；否则不启动 |

**v2 的取舍**：v2a/v2b 可先交付并独立验收——情感与程序性不需要巩固就能生效；巩固与消解价值更高但复杂度也更高，放在后面。

---

## 十三、与 v1 的差异清单

实施时需同步回改 v1 设计的以下位置（**以免两版文档矛盾**）：

| v1 位置 | v2 起的变化 |
| --- | --- |
| §3「唯一门面」四个方法 | → **八个方法**（新增 `get_state` / `procedural_block` / `split_entity` / `rollback_derived`） |
| §4 八张表 | → **十一张表**（新增 `emotional_state` / `derived_link` / `entity_merge_candidate`） |
| §5.1 情感轨写入语义 | 措辞"状态覆盖 + 滑动聚合"**保持有效**；v2 明确了聚合公式与不落库 |
| §5.3 与 §15.1「v1 的 `consolidate()` 只做归档」 | v2 起为**完整实现** |
| §9.1 错误处理表 | v2 起增四条链路态度（5 条 → 9 条） |
| §10 测试策略 10 条 | v2 起增 11 条（共 26 条） |
| §12.1 配置项 17 项 | v2 起增 22 项 |
| §13 风险表 12 条 | v2 起增 6 条 |
| §14 验收标准 9 条 | v2 起增 5 条（共 14 条） |
| §14 第 4 条「八张表零残留」 | v2 起为**十一张表** |
| §8.3 / §11.3 契约约束 | v2 起增"调度在本体""`split_entity`/`rollback_derived` 需鉴权" |
| §15.2 Jev 评估 | v2 起进入**可选实施**（v2e），前置自测中文一致率 |

---

*本设计由头脑风暴流程逐段确认后固化。所有阈值均为初始值，须以实测数据调优；实施中若发现与 v1 或仓库实际代码冲突，以代码为准并回改本文档。*
