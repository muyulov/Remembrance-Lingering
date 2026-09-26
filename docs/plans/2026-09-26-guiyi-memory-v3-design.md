# 归忆 v3 设计（关系图多跳检索 · 向量索引 · GPU embedding 与重排）

日期：2026-09-26
状态：设计已确认，待实施
基线：本仓库 `main @ 7c6a2b6`
上游：`docs/plans/2026-09-25-guiyi-memory-design.md`（**v1 设计**）、`docs/plans/2026-09-26-guiyi-memory-v2-design.md`（**v2 设计**）；本设计是增量，不重述已定内容
覆盖范围：仅本仓库（`guiyi` 包）；本体侧变更见第十一章

---

## 一、目标与范围

**目标**

- 交付**关系图多跳检索**（作为第四路召回），含因果链利用。
- 交付 **`sqlite-vec` 向量索引**：突破"全部向量常驻内存"的规模上限。
- 交付 **bge-m3（GPU / Vulkan + llama.cpp）** embedding，并引入 **cross-encoder 重排**。

**非目标**

- 不做多模态记忆。
- 不做"后置图扩展"（两段式，种子出发的邻域补全）——见决策 1 的否决理由。
- 不做向量数据库、不做独立进程/HTTP 层。
- 不改动 v1/v2 的架构、四条硬约束、三层作用域模型与错误哲学。

**结构性好处：v3 对本体几乎零契约变更**——八个方法签名不变，`recall()` 返回值不变，唯一可感知变化是 `RecalledMemory.hit_routes` 中会多出 `"graph"` 值（该字段类型 `tuple[str, ...]` 已支持）。

---

## 二、决策记录

| # | 议题 | 结论 | 被否决的方案与原因 |
| --- | --- | --- | --- |
| 1 | 图检索如何接入 | **作为第四路加入 RRF**：query 识别实体 → scope 内图扩展 → 独立排名，与向量 / BM25 / 结构化过滤一起融合 | 后置扩展（先用混合检索取 top-K，再以实体为种子沿边补全）：精度高但**价值受种子质量限制**，解决的是"相关的事一起捞"而不是多跳问题，属 v4 候选；替换 BM25 路：会丢掉 BM25 的专有名词能力（v1 特意保留） |
| 2 | 图遍历走哪些边 | **以 `causal` 为核心**，辅以 `entity` 连接（由 `memory_entity` 自连接**即时算**，不读显式边）；**`semantic` 边不参与**；`temporal` 边默认关 | 四类边全走 + 按类型加权（原方案）：`semantic` 边会让图变成稠密图，预算被低价值连接吃掉。**关键判据：四类关系里只有 `causal` 不可派生**——`semantic` 已被向量路覆盖、`temporal` 与 `entity` 都能即时算出 |
| 3 | 图遍历的 scope 边界 | **严格限定在允许的 scope 集合内，结构上不可能跨出**：每跳重新校验、实体匹配只在允许 scope 内查、边两端 scope 不一致即拒绝走该边 | 允许跨 scope 后靠"检索时过滤"：图遍历**逐跳进行**，过滤点极多，任何一处漏掉即泄露 |
| 4 | bge-m3 向量升级的迁移 | **离线迁移命令**（`guiyi-migrate`）：停服 → 改配置 → 迁移 → 启动。**单向，不提供回退流程**；旧向量清理放在最后一步 | 启动时后台重算 + BM25-only 降级：引入"检索质量悄悄降级"的窗口期，用户只感觉"它突然记性变差"，极难归因；双模型并存 + 双路检索融合：**新旧模型向量空间不可混用**（新 query 编码去匹配旧向量在数学上无意义），要么双路编码使成本翻倍，要么就是错的，且属过渡态专用（YAGNI） |
| 5 | `sqlite-vec` 的定位 | **`memory_vector`（BLOB）是唯一真源；`sqlite-vec` 是「可丢弃的索引」**——写入同事务双写，删除只删权威，孤儿由一致性校验清理，异常时 `DROP` 后从权威 BLOB 重建（**不需重新编码**） | `sqlite-vec` 直接取代 `memory_vector` 作为唯一存储：降级回 numpy 时需转换，向量数据变成对扩展的强依赖；`memory_vector` 权威 + 索引但无重建机制：两处不一致无法自愈 |
| 6 | 是否引入 cross-encoder 重排 | **引入**（`bge-reranker-v2-m3`，本地 llama.cpp）。**三重启用条件**：配置开启、**GPU 可用**、不在降级态 | 不做（原 v1 的 RRF + 规则重排）：用户明确要求引入；**CPU 模式硬跑**：cross-encoder 是 O(N) 次前向，CPU 上对 20 个候选是秒级事故，故 CPU 模式**直接跳过**而非硬跑 |
| 7 | reranker 的作用范围 | **精排 RRF 后的前 `rerank_top_k` 个，设为与 `recall_top_k` 同值（默认 20）= 全量精排** | 只精排 top-8（更省延迟）：用户要求扩大范围；但需连带把 `rerank_timeout_ms` 从 300 提到 800，否则频繁超时→频繁降级→reranker 形同虚设 |
| 8 | reranker 分数的组合方式 | **插值形式** `final = base × (1 − w + w × rerank_norm)`，`w=0` 时**精确退化**为 v1 原式 | 纯乘积：cross-encoder 分数集中在 0.5–0.9，直接相乘会压扁既有因子（重要性/新近度/置信度）的区分度；纯替换：会丢掉"重要的事该被记住"这一产品语义 |
| 9 | llama.cpp 的进程归属 | **外部管理**，归忆只要求 `embed_endpoint` / `rerank_endpoint` 可达；不可达即降级 | 归忆拉起子进程：违反 v1 §3 硬约束"纯 Python 对象、无后台生命周期"（与 v2 决策 9 一致） |
| 10 | query 无已知实体时图路的行为 | **正常返回空**，不报错 | 报错或退化告警：这不是异常——图路对"我记得你上次说的那个人的事"有用，对"今天天气"没用 |
| 11 | GPU 不可用时的行为总则 | **逐条降级、绝不硬跑**：embedding 回落 ONNX CPU；reranker **跳过**（`w=0`）；`sqlite-vec` 回落 numpy | 统一"要么全开要么全关"：会让一个次要组件的能力缺失拖垮整体可用性 |

---

## 三、架构变更

**模块变更**

```text
guiyi/
  graph/     # 新增：实体识别 → scope 内图扩展 → 独立排名
  embed/     # 新增 LlamaCppEmbedder（HTTP）；OnnxEmbedder 保留为降级实现
  store/     # 新增 VecIndex 的 sqlite-vec 实现；保留 numpy 实现
  cli/       # 新增 migrate 入口（guiyi-migrate）
```

依赖方向不变：`api → {retrieve, extract, judge, forget, emo, proc, entity, graph} → store → schema`。

**门面与方法签名：全部不变**（八方法）。`recall()` 返回值不变，仅 `RecalledMemory.hit_routes` 会新增 `"graph"` 取值。

**`VectorIndex` 协议不变**——numpy 与 sqlite-vec 是两个实现，上层无感。

**新增依赖**：`sqlite-vec`（条件启用）；两套 GGUF 模型文件（bge-m3、bge-reranker-v2-m3）；一个本地 llama.cpp 服务（提供 embedding 与 rerank 两个端点）。**无新增 Python 运行时依赖**（llama.cpp 走 HTTP）。

**llama.cpp 部署形态**：embedding 与 rerank 是**两个不同模型**，而 llama.cpp server 一次只加载一个模型——因此需要**两个实例 / 两个端口**。FP16 下两者合计约 2.3GB，12GB 显存充裕。

---

## 四、图路设计

### 4.1 `recall()` 的新五步

| 步 | 内容 |
| --- | --- |
| 1 | **四路并行召回**：向量 / BM25 / 结构化过滤 / **图路**——全部在 **SQL 层绑定 scope** |
| 2 | **RRF 融合**（`k=60`，沿用 v1）→ 取 `recall_top_k`(20) |
| 3 | **reranker 精排**（仅三方条件满足时；见第五章） |
| 4 | **规则重排**：v1 既有因子与 reranker 分数按插值组合 |
| 5 | 类型配额 + token 预算组装（沿用 v1/v2） |

### 4.2 图路算法

1. **query 实体识别**：在给定 scope 内查 `entity` / `entity_alias`（归一化精确 → 别名命中 → 向量近邻，最多 `graph_entity_max_matches` 个）。**只读不写**——检索不该改数据。
2. **种子记忆**：经 `memory_entity` 反查这些实体关联的条目。
3. **有界扩展**（BFS）：
   - 边：**`causal`**（读 `memory_relation`）+ **`entity` 连接**（`memory_entity` 自连接即时算）；`semantic` 不参与；`temporal` 仅在 `graph_temporal_edges_enabled=true` 时参与（默认关）
   - 预算：`graph_max_hops`(2)、每节点分支 `graph_max_branch`(20)、总节点 `graph_max_nodes`(200)
   - 权重：`hop_decay = graph_hop_decay^hop`（默认 0.5，逐跳衰减）
   - `visited` 集合去重防环
   - **每跳重新校验 scope**（第七章红线）
4. **排名**：`graph_score = Σ_路径 (hop_decay × edge_confidence)`，在候选集内归一化，作为图路的独立排名。
5. 作为**第四路**并入 RRF，`graph_hit_weight` 默认 `1.0`（待实测）。

### 4.3 两条行为约定

- **query 里识别不出任何已知实体 → 图路返回空、不报错**（正常降级）。
- **数据异常兜底**：若一条边的两端 scope 不同，**拒绝走这条边**（以严格者为准）。

---

## 五、reranker 设计

**启用条件（三重，全满足才生效）**

1. `rerank_enabled = true`
2. **GPU 可用**——`rerank_endpoint` 可达；**CPU 模式直接跳过，不硬跑**
3. 不处于降级态

**模型**：`bge-reranker-v2-m3`（多语言，与 bge-m3 同源），走 llama.cpp `--reranking`，GGUF。**本地推理，不出网。**

**作用范围**：RRF 后的前 `rerank_top_k`（默认 20，与 `recall_top_k` 同值）。**精排范围不得超过召回范围**——`rerank_top_k > recall_top_k` 时抛 `ConfigError`（防呆）。

**组合公式**

```
rerank_norm = minmax(rerank_score)                 # 候选集内归一化（cross-encoder 分数分布窄）
base        = rrf_norm × hit_weight × log1p(access_count) × importance × confidence × recency
final       = base × (1 − w + w × rerank_norm)     # w = rerank_weight，默认 0.5
```

三个设计点：

- **`w = 0` 时精确退化为 v1 原式**——reranker 是"可完全摘除的增益项"，不是必需环节。
- **不做纯乘积**——避免压扁既有因子的区分度。
- **减半保护**——`rerank_norm = 0` 的条目仅被 ×0.5，**不会被清零**；高重要度记忆不该因一次相关性判断被彻底挤出。

**输入口径**：`(query, memory.text)`，**不含 payload 结构化字段**——与向量路输入一致，避免两路"看的东西不一样"导致排序不可解释。

**降级与恢复**（沿用 v2 `Judge` 的模式；**各自的失败计数与探测独立**，避免两个服务互相牵连）

| 情形 | 处理 |
| --- | --- |
| 单次超时（`rerank_timeout_ms`=800） | 本次 `w=0`，用原序 |
| 连续 `rerank_failure_threshold`(3) 次失败 | 进入降级态：**零额外延迟**，直接 `w=0` |
| 降级态下每 `rerank_probe_interval_min`(10) 探测一次 | 成功即恢复 |
| `local_only = true` | 永久跳过 |

**一条须实测的风险**：cross-encoder 延迟随候选数线性增长。**批量前向**（一次提交 20 个文档对）与**逐条前向**的性能可能相差一个数量级。llama.cpp 的 rerank 端点接受 documents 数组，理论上可批量，但**是否真批量须在落地时实测**；若只能逐条，20 个候选在 GPU 上也可能到秒级，此时应下调 `rerank_top_k` 或关闭。

---

## 六、向量索引与迁移

### 6.1 `sqlite-vec` 接入

| 项 | 结论 |
| --- | --- |
| 定位 | **可丢弃索引**；唯一真源是 `memory_vector`（BLOB） |
| 启用 | `vec_index_enabled`；或向量数 > `vector_in_memory_max`(30 万) 时告警提示开启 |
| 环境检测 | `start()` 检测 `load_extension` 可用性（Python 标准库 `sqlite3` 需编译期开启 `--enable-loadable-sqlite-extensions`）；**不可用 → 自动回落 numpy + 告警，不拒绝启动** |
| 一致性 | `vec_index_check()` 清理孤儿行、补建缺失行——**缺失行从权威 BLOB 重建，不触发重新编码** |
| 恢复 | 任何异常下 `DROP` 索引表后重建 |
| 存储代价 | 翻倍（10 万 × 1024 维 × 4B × 2 ≈ 820MB） |

**如实标注**：它的收益是**内存上限**（不必把全部向量读进 Python 进程），**不是速度**——sqlite-vec 提供的仍是暴力扫描，可能比 numpy 常驻数组更慢。**须实测对比后**才决定是否默认开启（见验收 21）。

### 6.2 bge-m3 迁移（单向）

**顺序：停服 → 改配置 → 跑 `guiyi-migrate` → 启动**（配置是唯一真源，迁移命令从配置读目标模型）

| 步 | 动作 |
| --- | --- |
| 1 | **前置校验**：目标模型存在且可加载、`embed_dim` 与模型输出维度一致、embedding 端点可达、**服务未在运行**（取独占写锁，失败即报"服务仍在运行"）；打印即将发生的事并要求 `--yes` 确认 |
| 2 | **自动备份**：`VACUUM INTO`（沿用 `backup_before_migrate`） |
| 3 | **分批重算**：按 `memory_item` 分批重算向量并写入 `memory_vector`（`model = 新模型`） |
| 4 | **清理旧向量**：`DELETE FROM memory_vector WHERE model != 新模型` |
| 5 | **更新版本**：`PRAGMA user_version` → 3 |

**幂等续跑靠已有的 `model` 列**，不需要迁移状态表：

```sql
SELECT id FROM memory_item
WHERE id NOT IN (SELECT item_id FROM memory_vector WHERE model = :new_model)
```

中断后重跑只处理缺失行。**清理旧向量放在最后一步**：中途失败时旧向量仍在，库不会经历"零向量"状态——**不可逆操作尽量靠后**。

**`start()` 增校验**：`memory_vector.model` 与 `embed_model` 不一致 → **拒绝启动**并提示"请先运行 `guiyi-migrate`"，防止"改了配置忘了迁移"的静默错配。

**不提供回退流程**。因此 **v2 §12 的中文一致率自测是硬门槛**——切之前必须测，切完若质量不达标只能再改配置 + 再跑一次迁移。迁移命令**不删除任何模型文件**（模型文件的清理不属归忆职责）。

### 6.3 迁移顺序

- `user_version` 2 → 3。
- **若同时换模型 + 开 `sqlite-vec`：先重算（换模型），再建索引（搬运）**——顺序不能反，否则建好的索引立刻失效。

---

## 七、安全红线

> **图遍历严格限定在允许的 scope 集合内，结构上不可能跨出。**

v2 的 scope 过滤是在**召回时一次性**做的；而图遍历是**逐跳进行**的，过滤点极多，任何一处漏掉就是隐私泄露。机制四条：

1. 图遍历的**每一跳**都重新校验节点 scope 是否在允许集合内；
2. 实体匹配**只在允许的 scope 内查**（不跨 scope 找同名实体）；
3. **边的 scope 一致性校验**：一条边两端 scope 不同（数据异常）时拒绝走该边；
4. 单测构造"跨 scope 的边"，断言遍历**不会跨出**（测试 29）。

**为什么这是 v3 头号风险**：v1/v2 建立的隔离保证都是"单点过滤"，而图路把过滤点数量从 1 变成了 O(跳数 × 分支数)。代价是 Aliya 永远不会把"群里的某人"和"私聊里的某人"认成同一实体——但"认出来"这件事本身即泄露，这正是 v1 把实体按 scope 隔离的原因。

---

## 八、错误处理

延续 v1 §九 / v2 §六 的"区分致命与可降级"：

| 链路 | 态度 | 理由 |
| --- | --- | --- |
| 图路失败（识别或遍历出错） | **可降级**：该路返回空，其余三路照常 | 多一路是增益，不该成为单点 |
| query 无已知实体 | **正常返回空** | 不是错误（决策 10） |
| reranker 超时 / 失败 | 本次 `w=0`；连续失败进降级态 | 检索是热路径，必须有零延迟兜底 |
| GPU / llama.cpp embedding 不可用 | **回落 ONNX CPU** | 沿用 v1 的降级链 |
| `sqlite-vec` 不可用 | **回落 numpy + 告警，不拒绝启动** | 索引是可选加速，不是必需 |
| 向量 `model` 与配置不一致 | **fail fast 拒绝启动**，提示运行 `guiyi-migrate` | 静默错配会导致检索质量莫名下降，且无报错 |
| `guiyi-migrate` 前置校验失败 | **拒绝开始，不动任何数据** | 迁移是单向的，前置必须严 |
| 迁移中途失败 | 已重算行保留、重跑续传；**库不处于"零向量"状态** | 清理在最后一步 |

**新增异常**：`MigrationError`。

---

## 九、测试策略

沿用 v1 §十 / v2 §七 的手段（假 `Embedder` / `Extractor` / `Judge` / **假 reranker 与假图扩展**，`:memory:` SQLite，全程不触网）。**v3 新增 12 条，编号 29–40**：

| # | 用例 | 断言 |
| --- | --- | --- |
| 29 | **图路不跨 scope** | 构造 scope A 的边指向 scope B 的记忆 → 遍历结果**不含** B；边两端 scope 不一致时拒绝走该边 |
| 30 | 图路预算 | 构造稠密图 → 扩展节点数 ≤ `graph_max_nodes`，跳数 ≤ `graph_max_hops`，每节点分支 ≤ `graph_max_branch` |
| 31 | 图路环路 | 构造环 → 遍历终止，不重复访问 |
| 32 | 无实体 query | 图路返回空，其余三路正常，**不报错** |
| 33 | reranker 精确退化 | `w=0` 时 `final` 与 v1 原式**逐位相同** |
| 34 | reranker 超时 | 超时 → 本次用原序（`w=0`） |
| 35 | reranker 降级与恢复 | 连续 3 次失败 → 降级态零额外延迟；探测成功后恢复 |
| 36 | 精排范围校验 | `rerank_top_k > recall_top_k` → `ConfigError` |
| 37 | `sqlite-vec` 不可用 | `load_extension` 失败 → 自动回落 numpy，**服务正常启动** |
| 38 | **索引可重建** | 删除索引表 → `vec_index_check()` 从权威 BLOB 重建，**断言 Embedder 未被调用**（不需重新编码） |
| 39 | **迁移幂等续跑** | 迁移中断后重跑，只处理缺失行；最终向量数 == `memory_item` 数 |
| 40 | **模型不符拒绝启动** | `memory_vector.model` ≠ `embed_model` → 启动抛 `MigrationError` 并提示 `guiyi-migrate` |

累计 40 条（v1 15 + v2 13 + v3 12）。

---

## 十、配置项（v3 新增）

**全部阈值与权重均为外置配置，标注「初始值待实测调优」。**

| 配置项 | 默认 | 标注 |
| --- | --- | --- |
| `graph_enabled` | `true` | — |
| `graph_max_hops` | `2` | 待实测 |
| `graph_max_branch` | `20` | 待实测 |
| `graph_max_nodes` | `200` | 待实测 |
| `graph_hop_decay` | `0.5` | 待实测 |
| `graph_hit_weight` | `1.0` | 待实测 |
| `graph_entity_max_matches` | `5` | 待实测 |
| `graph_temporal_edges_enabled` | `false` | — |
| `rerank_enabled` | `true` | — |
| `rerank_endpoint` | — | — |
| `rerank_model` | `bge-reranker-v2-m3` | — |
| `rerank_top_k` | `20`（≤ `recall_top_k`） | 待实测 |
| `rerank_timeout_ms` | `800` | 待实测 |
| `rerank_weight`（公式中的 `w`） | `0.5` | 待实测 |
| `rerank_failure_threshold` | `3` | 待实测 |
| `rerank_probe_interval_min` | `10` | 待实测 |
| `vec_index_enabled` | `false`（条件启用） | 待实测（须先证明比 numpy 有净收益） |
| `vec_index_check_on_start` | `true` | — |
| `embed_backend` | `onnx`（可切 `llamacpp`） | — |
| `embed_endpoint` | — | — |

（20 项。`embed_model` / `embed_dim` / `vector_in_memory_max` / `backup_before_migrate` 为 v1/v2 已有项，不重复计入。）

---

## 十一、本体侧契约变更

### 11.1 归忆导出增量

- **门面方法签名：无变化**（仍为八方法）。
- `recall()` 返回值：无新字段；`RecalledMemory.hit_routes` 可能含 `"graph"`。
- 新增异常：`MigrationError`。
- 新增 CLI 入口：`guiyi-migrate`（归忆自持，本体无需包装）。

### 11.2 本体改动清单（在 v1 的 6 处 + v2 的 5 处之外）

| 文件 | 动作 |
| --- | --- |
| 部署侧（非代码） | 起两个 llama.cpp 实例（embedding / rerank），配 `embed_endpoint` 与 `rerank_endpoint`；放置 bge-m3 与 bge-reranker-v2-m3 的 GGUF |
| `core/config/settings.py` | `MemorySettings` 增 v3 的 20 项配置 |
| `data/config/app.yaml` | 补对应默认段 |
| 运维流程 | 升级 bge-m3 时按 6.2 的顺序执行（停服 → 改配置 → `guiyi-migrate` → 启动） |

### 11.3 契约约束

- **调度与进程管理全在本体/外部**：归忆不持有后台任务，也不拉起 llama.cpp。
- 归忆**不删除任何模型文件**（迁移不清理旧模型）。
- 参数一律经 `MemorySettings` 传入，归忆不读环境变量。

---

## 十二、风险与缓解

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| **图遍历跨 scope 泄露**（头号） | 把 A 的事说给 B 听，且不报错、难发现——隔离保证从"单点过滤"变 O(跳数×分支) | 第七章四条机制（每跳校验 / 实体只在允许 scope 查 / 边 scope 一致性 / 结构性测试 29） |
| **reranker 拖垮延迟** | cross-encoder 是 O(N) 次前向，候选多时到秒级 | 仅 GPU 模式启用（CPU 直接跳过）+ 超时 800ms + 降级态零延迟 + 批量前向须实测（第五章） |
| **bge-m3 切换单向不可逆** | 质量不达标时只能再迁移一次 | **前置硬门槛**：v2 §12 中文一致率自测；迁移前自动备份；清理放最后 |
| `sqlite-vec` 的 `load_extension` 不可用 | 环境不支持，索引无法启用 | `start()` 前置检测 → 自动回落 numpy + 告警，不拒绝启动 |
| **`sqlite-vec` 可能更慢** | 误以为"上了索引就更快"，实际是暴力扫描 | 文档如实标注收益是**内存上限**；验收 21 要求实测对比后才默认开启 |
| 存储翻倍 | 磁盘占用增加（10 万 × 1024 维 ≈ 820MB） | 索引可 `DROP` 释放；不启用即无此开销 |
| 图路质量依赖 v2 实体消解 | 消解碎片化严重则图路价值被削 | 列为 v3a 的前置条件；消解质量不达标时图路保持关闭 |
| 双 llama.cpp 实例的显存与运维 | 两个模型常驻（约 2.3GB） | 12GB 显存充裕；两个端点不可达时逐条降级，不影响核心链路 |

---

## 十三、验收标准（v3 增量，承接 v1 的 1–9、v2 的 10–16）

17. **多跳优于基线**（v1 §15.1 定下的出口条件）：在构造的多跳问题上，开启图路后的命中率 / LLM-as-Judge 分数**显著优于仅混合检索基线**。
18. **图路不跨 scope**：构造跨 scope 的边，遍历结果不含越界记忆（结构性断言）。
19. **reranker 增益**：开放域题目上优于"仅 RRF"基线；且 `w=0` 时逐位等于基线。
20. **延迟预算**：图路 + reranker 全开时 `recall()` 的 P95 增量 ≤ 300ms（GPU 模式）。**该项为初始目标，须实测后回填真实值**；若批量前向不可用导致超标，则下调 `rerank_top_k`。
21. **内存上限**：启用 `sqlite-vec` 后常驻内存**不随向量数线性增长**；且须提交与 numpy 模式的**实测对比数据**（含 P50/P95 延迟与内存峰值），据此决定是否默认开启。
22. **迁移可续跑**：迁移中断后重跑最终一致（向量数 == 条目数）；`memory_vector.model` 与配置不符时服务**拒绝启动**。

---

## 十四、实施顺序建议

| 切片 | 内容 | 前置条件 |
| --- | --- | --- |
| **v3a** | 图路：实体识别 + 有界 BFS + 第四路接入 | **v2 实体消解质量达标**（碎片化严重则图路价值被削） |
| **v3b** | reranker：三重启用条件 + 插值组合 + 降级态 | GPU 可用；**批量前向性能实测** |
| **v3c** | bge-m3 迁移：`guiyi-migrate` + `start()` 校验 | **v2 §12 中文一致率自测达标**（单向不可逆） |
| **v3d** | `sqlite-vec`：可丢弃索引 + 一致性校验 | 向量数达阈值；**实测证明比 numpy 有净收益** |

**v3 的取舍**：v3a/v3b 是"检索质量"的增益，可独立交付与验收；v3c 是"不可逆的基础设施变更"，必须最后做且门槛最严；v3d 是纯规模问题，**数据量没到就不做**（YAGNI）。

---

## 十五、与 v1 / v2 的差异清单

实施时需同步回改前两版设计的以下位置（**以免版本间矛盾**）：

| 前置文档位置 | v3 起的变化 |
| --- | --- |
| v1 §15.1 的 v3 行 | 指向本文档；出口条件细化为验收 17–22 |
| v1 §四 八张表（v2 起十二张） | **v3 不新增业务表**；`sqlite-vec` 虚拟表是索引，不计入表数 |
| v1 §六「唯一门面」八个方法 | **不变**（v3 无契约变更）；`RecalledMemory.hit_routes` 增 `"graph"` 取值 |
| v1 §八 检索参数 | v3 起增图路（`graph_hit_weight`）与 reranker（`rerank_weight`）两组参数 |
| v1 §9.2 异常层次 | v3 起增 `MigrationError` |
| v1 §12.2 依赖清单 | v3 起增 `sqlite-vec`（条件启用）；GGUF 模型与 llama.cpp 走外部服务、不进 pip 依赖 |
| v1 §12.3 实施约定 | v3 起增两条：**向量 `model` 与配置不符即拒绝启动**；`rerank_top_k ≤ recall_top_k` 校验 |
| v1 §13 风险表（v2 起 20 条） | v3 起增 8 条，头号为**图遍历跨 scope 泄露** |
| v1 §14 验收（v2 起 16 条） | v3 起增 6 条（共 22 条）；**第 5 条的 `P95 < 100ms` 须在图路 + reranker 全开下复测并回填真实值** |
| v1 §10 测试策略（v2 起 23 条） | v3 起增 12 条（共 40 条） |
| v2 §12.1 配置项 28 行 | v3 起增 20 行（共 48 行） |
| v2 §12 中 Jev（v2e） | **不冲突**：v3b 的 reranker 与 v2e 的 Jev 是两件独立的事，可分别启用 |

---

*本设计由头脑风暴流程逐段确认后固化。所有阈值均为初始值，须以实测数据调优；其中 `rerank_top_k`、`sqlite-vec` 的默认是否开启、以及验收 20 的延迟目标，均依赖落地实测结果回填。实施中若发现与 v1/v2 或仓库实际代码冲突，以代码为准并回改本文档。*
