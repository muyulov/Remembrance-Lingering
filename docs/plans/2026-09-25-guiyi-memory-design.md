# 归忆（Remembrance-Lingering）记忆系统 设计

日期：2026-09-25
状态：设计已确认，待实施
基线：本仓库 `main @ 53126cb`（仓库仅有 README 与 LICENSE，本设计从零起步）
覆盖范围：**仅本仓库**（`guiyi` 包）。`Aliya-cosmos` 本体侧的改动只在第八章列出接口契约与接入清单，不展开实现。

---

## 一、目标与范围

归忆是 **Aliya（个人 AI 伙伴）的长期记忆系统**，跨会话记住用户、经历、偏好与情感。

**目标**

- 记住"发生过什么"（情景）、"这个人是谁"（语义）、"关系与情绪现在如何"（情感）、"Aliya 自己怎么说话"（程序性）。
- 检索到的记忆能在对话中被正确引用，且**不串人、不串群**。
- 数据可回溯、可重建、可删除。

**非目标（本版不做）**

- 不提供 HTTP / RPC 层——若将来要独立部署，由使用方自行包一层。
- 不做实体图多跳检索（v2 规划）。
- 不做多模态记忆（图片、语音）。
- 不做跨用户共享记忆（除群记忆这一层）。

---

## 二、决策记录

| # | 议题 | 结论 | 被否决的方案与原因 |
| --- | --- | --- | --- |
| 1 | 与本体关系 | 独立、框架无关的 Python 包，本体侧写薄适配层 | 并入本体 `core/`（强耦合，独立仓库失去意义）；独立进程 + API（对个人伙伴是过度设计） |
| 2 | 是否继承本体 `Service` | **不继承**，归忆定义自己的对象模型 | 继承会造成 `本体 ⇄ 归忆` 循环依赖，且违反本体"service 层不引用框架"的既有约定 |
| 3 | 记忆类型 | 四轨：情景 + 语义 + 情感 + 程序性 | 双轨（能力不足）；极简单轨（"了解这个人"的能力弱） |
| 4 | 存储 | 单文件 SQLite（WAL） | 嵌入式向量库（多组件 + 两套存储一致性负担）；外部向量服务（内容出本机） |
| 5 | Embedding | v1 CPU ONNX（bge-base-zh），`Embedder` 协议化 | bge-m3 走 CPU 延迟高一个数量级；ROCm + torch 需 `HSA_OVERRIDE_GFX_VERSION` 伪装 gfx1031，与 fail-fast 风格冲突 |
| 6 | 写入机制 | 两层：原文全量 + LLM 实时派生 | 只存抽取结果（不可恢复）；实时 vs 异步的选择见 #7 |
| 7 | 实时抽取实现 | **并发抽取、回复不等**，原文先落库 | 串行叠在回复链路（响应从约 1s 拉到约 3s）；单次调用双输出（把"怎么撩"与"记什么"耦合，难调） |
| 8 | 作用域 | 三层：个人 / 群 / Aliya 自我 | 严格单用户（无法被拉群）；不分层（群内会泄露个人隐私） |
| 9 | 检索 | 混合检索（向量 + BM25 + 结构化过滤）+ RRF + 重排 + 类型配额 | 纯向量（专有名词区分度差）；v1 上实体图（依赖实体消解质量，先做对再开） |
| 10 | 四轨存储形态 | **单表** + `payload` JSON | 四张表（跨类型召回与统一排序复杂，向量索引要维护四份） |
| 11 | 矛盾事实 | supersede 链，旧值保留 | 直接覆盖（丢失"你以前喜欢 X"这类有价值的历史） |
| 12 | 遗忘 | 三档：排序衰减 / 归档 / 硬删除 | 单一"删除"策略（要么丢数据，要么无法真正删除） |
| 13 | 删除失败 | `forget()` **绝不降级**，失败即抛并重试 | 记日志后继续（删除没有"部分成功"，遗留数据即隐私事故） |
| 14 | 作用域缺失 | **fail fast** 抛 `ScopeError` | 默认放行查全库（一旦漏传 `scope` 就是泄露） |

---

## 三、总体架构与模块划分

**包名**：`guiyi`（归忆）。独立 Python 包，自持 `pyproject.toml`，不依赖本体任何模块。

```text
guiyi/
  api.py          # 唯一对外门面：MemoryEngine，框架无关
  schema.py       # 领域模型（pydantic）：Scope / MemoryKind / MemoryItem / Provenance
  store/          # SQLite 建表与迁移、DAO、FtsIndex(FTS5)、VectorIndex 协议及实现
  embed/          # Embedder 协议 + OnnxEmbedder(v1) + NullEmbedder(降级)
  extract/        # Extractor 协议 + LlmExtractor（四轨结构化抽取）
  retrieve/       # 三路召回 + RRF 融合 + 重排 + token 配额组装
  forget/         # 衰减、巩固、作用域删除
  errors.py       # 异常层次
```

**依赖方向**：单向 `api → {retrieve, extract, forget} → store → schema`。子模块之间不横向引用（`retrieve` 不 import `extract`），保证各自可独立测试。

**三条硬约束**

1. **不 import 本体任何模块**，也不继承 `core.service.Service`——避免循环依赖。
2. **外部依赖全部经协议注入**：`Embedder`、`Extractor`、`LlmClient` 均为 `Protocol`；v1 实现是 ONNX、LLM 抽取、DeepSeek 客户端。测试可注入假实现，换模型不改上层。
3. **不引入 web 框架**。`MemoryEngine` 是纯 Python 对象。

**唯一门面**：`MemoryEngine` 暴露四个方法——`remember()`、`recall()`、`forget()`、`consolidate()`。本体侧适配层只做转发。

---

## 四、数据模型

**五张表，四轨统一存储。**

| 表 | 作用 | 关键字段 |
| --- | --- | --- |
| `utterance` | 原文层（**不可再生**） | `id`、`scope_type`/`scope_key`、`session_id`、`speaker_id`、`role`、`text`、`created_at` |
| `memory_item` | 记忆条目层（四轨统一表） | `id`、`kind`、`scope_type`/`scope_key`、`text`、`payload` JSON、`importance`、`confidence`、`status`、`supersedes_id`、`pinned`、`access_count`、`last_accessed_at`、`created_at`、`updated_at` |
| `memory_vector` | 向量（独立表） | `item_id` PK、`dim`、`vec` BLOB(float32)、`model` |
| `entity` / `entity_alias` / `memory_entity` | 实体与关系 | `canonical_name`、`kind`、`alias`、`memory_id` |
| `memory_provenance` | 来源指针 | `memory_id`、`utterance_id`、`extractor_version`、`created_at` |

**枚举取值**

- `kind`：`episodic` / `semantic` / `emotional` / `procedural`
- `scope_type`：`personal` / `group` / `self`；`scope_key` 分别为 QQ 号 / 群号 / 字面量 `'self'`
- `status`：`active` / `superseded` / `archived`

**三个关键决策**

1. **四轨单表**。检索是核心路径——单表让"跨类型召回 + 统一 RRF 重排 + 一份向量索引"变简单。类型差异用 `payload` JSON 承载：
   - 语义：`subject` / `predicate` / `object`
   - 情感：`emotion` / `target` / `intensity`
   - 程序性：`style` / `trait`
   - 情景：`event_time` / `participants`
2. **矛盾更新用链**。新事实把旧事实置 `status=superseded` 并用 `supersedes_id` 串联，历史保留。
3. **实体 v1 写入但不检索**。v2 开图时不必回填历史；`entity_alias` 从第一天存在，正是实体消解的基础。

**索引**

- `(scope_type, scope_key, kind, status)`——覆盖检索主要过滤组合
- `(supersedes_id)`——冲突链查询
- `memory_vector.item_id` 主键

**约定**

- 时间一律存 **UTC**。
- `memory_vector.model` 记录生成向量的模型名，换模型时据此识别需重算的条目。
- 向量以 `BLOB`（float32）存储，v1 用 numpy 暴力检索；v2 可切 `sqlite-vec` 虚拟表，`VectorIndex` 接口不变。

---

## 五、核心数据流

### 5.1 写入链路 `remember()`

1. **原文先落库并提交**——不可再生数据绝不丢，这是并发抽取能安全进行的前提。
2. **并发启动抽取**，与回复生成并行，互不等待。
3. **抽取**：`LlmExtractor` 输入本轮对话，输出四轨候选条目 + 重要度 + 置信度 + 实体。
4. **归一与去重**：实体先与别名表比对做消解；语义记忆与同作用域已有条目做相似度比对，命中阈值则走 `supersede` 而非新增。
5. **落库**：`memory_item` + `memory_vector` + `memory_entity` + `memory_provenance`（记 `extractor_version`）。
6. **失败只记日志**。原文已存，可离线用新模型重跑。

**四轨的写入语义各不相同**，不能一套逻辑套用：

| 轨 | 写入语义 |
| --- | --- |
| 情景 | **追加**（append-only），发生过就是发生过 |
| 语义 | **更新**（supersede 链），事实会变 |
| 情感 | **状态覆盖 + 滑动聚合**——"当前情绪""关系阶段"是状态，不是条目堆砌 |
| 程序性 | **证据累加**——同一 trait 的多次观察加权成稳定风格档案，不逐条检索 |

### 5.2 检索链路 `recall()`

1. **作用域解析**：调用方给出当前场景（私聊某人 / 某群）→ 允许的 scope 集合 `{personal(该人), group(该群), self}`。
2. **三路并行召回**，过滤条件在 **SQL 层绑定 scope**，不靠上层记得传参。
3. **RRF 融合**三路排名（避免各家分数量纲不可比）。
4. **重排**：`importance × 新近度衰减 × confidence × 命中权重`。
5. **类型配额 + token 预算**组装上下文：语义 40% / 情景 35% / 情感 15% / 程序性 10%。
6. **回写** `access_count` / `last_accessed_at`——巩固与衰减的输入信号。

### 5.3 遗忘与巩固

**三档处理，绝不混为一谈：**

| 档 | 触发 | 效果 |
| --- | --- | --- |
| **排序衰减** | 持续 | 只降排序权重，**不丢数据**；`pinned` 与程序性不衰减 |
| **归档** | 离线巩固 | 长期未访问的低重要度情景记忆置 `archived`，退出检索但可被巩固引用 |
| **硬删除** | 仅"忘掉我" | 按 scope 级联删除五张表相关行 |

**巩固 `consolidate()`**（离线任务，建议凌晨执行）：把相似情景记忆聚类，提炼成语义记忆（"多次提到加班" → "工作压力大"）；更新情感与关系状态。该步骤需要 LLM，但离线执行，成本与延迟可控。

---

## 六、错误处理

### 6.1 错误哲学：区分"致命"与"可降级"

| 链路 | 态度 | 理由 |
| --- | --- | --- |
| `recall()` | **可降级**：embedding 失败 → 退化 BM25-only，返回结果并置 `degraded=True` | 绝不允许"向量化挂了就不回话" |
| `remember()` | **可降级**：抽取失败 → 原文已落库，标记待重抽 | 对话不该被记忆系统拖垮 |
| `forget()` | **绝不降级**：删不干净即隐私事故，失败必须抛出并可重试 | 数据删除没有"部分成功" |
| 作用域缺失 | **fail fast** 抛 `ScopeError` | 宁可报错，也绝不在缺少隔离条件时查全库 |

**核心原则：默认拒绝，而非默认放行。**

### 6.2 异常层次

```text
GuiyiError
  ├── ConfigError      # 配置非法（如 embed_dim 与模型不符）
  ├── StoreError       # 存储层（含 database is locked）
  ├── EmbedError       # 向量化失败
  ├── ExtractError     # 抽取失败
  └── ScopeError       # 作用域缺失或非法
```

### 6.3 日志

归忆**不打印日志到 stdout**，也不依赖本体的 loguru。引擎通过注入的 `Logger` 协议回调上报（默认可传 `None` 静默），由本体适配层桥接到本体的 `log`。这样归忆不绑定任何日志框架。

---

## 七、测试策略

**核心手段：全程不碰真模型、不碰网络。** 注入确定性假 `Embedder`（返回预设向量）与假 `Extractor`（返回固定结构），存储用 `:memory:` SQLite。测试快、稳、可离线跑 CI。

**必测用例（按重要性排序）：**

| # | 用例 | 断言 |
| --- | --- | --- |
| 1 | **作用域隔离** | A 的私聊记忆，在群聊上下文的 `recall()` 结果中**查不到**。断言落在**查询结果**上，而非白盒检查 SQL 字符串 |
| 2 | 矛盾更新 | 先后写入矛盾事实 → `recall()` 只返回新值；旧值可按 id 查到且 `status=superseded` |
| 3 | 降级路径 | `Embedder` 抛错时 `recall()` 仍返回 BM25 结果并置 `degraded` |
| 4 | 删除彻底性 | `forget()` 后五张表零残留 |
| 5 | 去重幂等 | 同一事实重复抽取两次，不产生两条 `active` 记忆 |
| 6 | 写入不阻塞 | 抽取失败时，原文仍已落库 |
| 7 | RRF 融合 | 构造已知排名，断言融合顺序符合预期 |
| 8 | 作用域缺失 | 不传 `scope` 调 `recall()` → `ScopeError` |
| 9 | 向量维度校验 | `embed_dim` 与模型输出不符 → `ConfigError` |

---

## 八、本体侧接口契约

> 本章只列契约与改动清单，不展开实现。落地时在 `Aliya-cosmos` 仓库另开设计。

### 8.1 归忆导出

- `MemoryEngine`：`remember()` / `recall()` / `forget()` / `consolidate()`
- `MemorySettings`（pydantic `BaseModel`）
- `Scope` / `MemoryKind` / `MemoryItem` / `RecallResult` 等类型
- `GuiyiError` 及其子类

### 8.2 本体改动清单（6 处）

| 文件 | 动作 |
| --- | --- |
| `pyproject.toml` | 加归忆依赖（开发期用 uv path 依赖指向本仓库） |
| `core/config/settings.py` | `Settings` 增顶层字段 `memory: MemorySettings` |
| `core/config/__init__.py` | 导出 `MemorySettings` |
| `core/service/memory_service.py` | 新增 `MemoryService(Service)`：构造器接 `MemorySettings` + `ClockService`；`start()` 建表与预热 Embedder；`stop()` 关连接；方法转发给引擎 |
| `core/service/registry.py` | `register(MemoryService)` |
| `data/config/app.yaml` | 补 `memory:` 段 |

### 8.3 契约约束

- `MemoryEngine` 的方法**必须显式接收 `scope` 与时间**，不读任何隐式全局状态。
- 归忆不打印日志、不抛框架异常，只用自己的异常层次。
- 归忆**不反向依赖**本体，也不持有任何密钥。API key 由本体注入。

---

## 九、配置与依赖

### 9.1 `MemorySettings` 配置项

| 字段 | 说明 |
| --- | --- |
| `db_path` | 记忆库路径，默认 `data/memory/guiyi.sqlite3` |
| `embed_model` | 模型名或路径，默认 bge-base-zh ONNX |
| `embed_dim` | 向量维度，用于校验模型与向量表一致 |
| `embed_timeout_ms` | 单次向量化超时 |
| `recall_top_k` | 召回条数 |
| `recall_token_budget` | 组装上下文 token 上限 |
| `type_quota` | 四轨配额比例（默认 40/35/15/10） |
| `decay_half_life_days` | 新近度半衰期 |
| `consolidate_cron` | 巩固任务时间 |
| `llm_extract_model` | 抽取用模型名 |

**API key 由本体注入，不落归忆配置**——归忆不持有密钥。

### 9.2 依赖清单

- 运行时：`pydantic>=2`、`numpy`、`onnxruntime`
- 可选（v2）：`sqlite-vec`
- 开发：`pytest`、`ruff`、`basedpyright`
- **明确不引入**：torch、langchain、任何 web 框架、任何向量数据库 SDK
- Python **pin 到 3.12**（系统 Python 3.14 下 onnxruntime 轮子未必齐）

---

## 十、风险与缓解

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| **SQLite 并发写** | WAL 下单写多读，并发抽取易撞 `database is locked` | 抽取写入走**单写队列**串行化；捕获 `StoreError` 后重试 |
| 实时抽取的成本与失败率 | 每轮多一次 LLM 调用 | 并发执行不阻塞回复；原文已存，失败可离线重抽 |
| **抽取走外部 API，内容离开本机** | 隐私 | 在文档与配置说明中**显著声明**；提供"仅本地"模式开关（关闭抽取则退化为纯原文 + BM25） |
| 实体消解质量 | 决定 v2 图检索成败 | v1 先写入实体与别名，不检索；消解做对后再开图 |
| 四轨一次上齐复杂度高 | 实现与调试成本 | 提供**功能开关**，第一版允许先只开 情景 + 语义 |
| 小 embedding 对专有名词区分度弱 | 检索漏召 | 混合检索中的 BM25 路专门兜底；`entity_alias` 辅助 |
| 群聊回复全群可见 | 检索漏加 scope 即泄露 | scope 过滤在存储层强制；缺 scope 直接 `ScopeError` |

---

## 十一、验收标准

1. 私聊能正确引用 30 天前的用户偏好，且该记忆**在群聊中绝不复述**（隔离回归测试通过）。
2. 矛盾事实更新后 `recall()` 只返回最新值，历史可追溯。
3. embedding 不可用时仍能按关键词召回，并显著标记 `degraded`。
4. `forget()` 后五张表零残留（自动化断言）。
5. 10 万条记忆下单次 `recall()` P95 < 100ms（CPU）。
6. 抽取失败不影响对话，且原文可离线重抽。
7. `uv run pytest` 全绿；`uv run ruff check .` 与 `basedpyright` 零告警。

---

## 十二、分阶段实施建议

| 阶段 | 内容 | 出口条件 |
| --- | --- | --- |
| **v1** | SQLite 五表 + 作用域强制 + 混合检索（向量 + BM25）+ 情景/语义两轨 + 并发抽取 + 降级路径 | 验收标准 1–7 通过 |
| **v2** | 情感与程序性两轨 + 离线巩固 + 实体消解 | 情感状态能稳定更新；聚类提炼不产生明显噪声 |
| **v3** | 实体图多跳检索 + `sqlite-vec` 向量索引 + Vulkan/llama.cpp 升级 bge-m3 | 图检索在多跳问题上优于混合检索基线 |

**v1 的取舍**：先把"不串人、不丢数据、能降级"这三件正确性的事做对，再谈能力上限。能力可以后加，正确性一旦出问题就是信任损失。

---

*本设计由头脑风暴流程逐段确认后固化。实施中若发现与既有约定冲突，以本仓库实际代码为准，并回改本文档。*
