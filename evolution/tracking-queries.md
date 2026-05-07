# Tracking Queries

> 给 2026 部门 AI 知识管理 talk 用的演化追踪——每次 ingest 前后各跑一次 query workflow，结果落到 `evolution/query-snapshots/`。
> 详见 [`CLAUDE.md` 附录 A](../CLAUDE.md)。Talk 结束后本目录整体删除。

每条 query 用 `query-key` 唯一标识，和快照文件名直接拼接（`<date>-<ingest-context>-<query-key>-<pre|post>.md`）。

---

## 1. giga-scale-sharding

**Question**:
当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？
Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。

**Why track**: 这是 talk "成果展示" 段（15 min）的主问题。从 D1 几乎答不出 → D14 给出多策略对比 + 5+ page citation，是开场 promise 最强证据。

**Cover area**: sharding strategies, system architecture, scale trade-offs

---

## 2. hnsw-vs-nsg-selection

**Question**:
HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？

**Why track**: 当前 wiki 已有 HNSW、NSW、NSG 三个 page，这道题现在就有体面答案——D-now 的快照已经很有内容；后续 ingest 加深会让 trade-off 表格更细。**适合做开场暖场对比**（Week 1 早期 vs Week 3 末）。

**Cover area**: graph-based ANN, algorithm comparison

---

## 3. quantization-landscape

**Question**:
PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？

**Why track**: 当前只有 PQ 一个 page。后续 ingest OPQ / RaBitQ / SPANN（含 PQ-OPQ 应用）会让答案**渐进增厚**——展示"知识沉淀曲线"的典型例子。

**Cover area**: quantization, compression-accuracy trade-offs

---

## 4. memory-vs-disk-large-scale

**Question**:
当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？

**Why track**: 当前 wiki **完全没覆盖** disk-based 方案——pre 答案会接近空白，等 SPANN / DiskANN ingest 后突然变厚。**戏剧性最强的"突变"对比**，是 talk 中段引出"为什么需要 ingest"的好素材。

**Cover area**: disk-based ANN, large-scale systems

---

## 5. embedding-update-handling

**Question**:
当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？

**Why track**: 这正是 talk 当天 live demo 那篇 2026 SIGMOD 论文（*Integrating Vector Databases across Embedding Models*）的主题。Live demo 后这道题答案会跳一大格，是**收束环节的绝佳"现场学习"证据**。

**Cover area**: embedding lifecycle, cross-model compatibility

---

## 6. scale-tier-shifts

**Question**:
向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在**索引选择、存储介质、并行能力**这三个维度上，**哪些决策在哪一档发生"质变"**？这里"质变"指必须换方案（如 HNSW → IVF+量化、内存 → SSD），而不是同方案下的参数微调。

**Why track**: 强迫答案以"迁移点"呈现，而不是 4×3 静态表格——更能体现 wiki 累积知识的判断力。
- Pre 阶段（wiki 薄）：可能只能粗略指出"内存 → 磁盘"一档质变，且无具体阈值
- Post 阶段（多轮 ingest）：能定位具体阈值（如 SPANN 主张的 ≥ 百亿、DiskANN 的 SSD 切换点）和具体替代算法

**与 #1 的分工**：本 query 是**全景质变图**（10亿 → 万亿 跨档的迁移点）；#1 (`giga-scale-sharding`) 是其中千亿/万亿一档的**具体工程落地**（节点数、内存、维度、P99）。两者形成"概览 → 深挖"递进。

**Cover area**: scale-driven design transitions, capacity planning, regime shifts

---

## 7. index-architecture-global-vs-routed

**Question**:
在千亿/万亿规模下，索引架构应选：
(a) **全局单一索引**——所有向量进同一个索引（如单个 HNSW），无路由层
(c) **层次路由结构**——粗粒度聚类先定位到少数分区，再走那些分区的局部索引（IVF / SPANN / Milvus segment 模型为代表）

两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？

**Why track**:
- Pre 阶段：wiki 单薄时大概率只能给"全局太大装不下，所以分"这种笼统理由，无法展开 (c) 的具体机制
- Post 阶段（SPANN / DiskANN / Milvus / Pinecone scaling 之后）：能展开 (c) 的具体路由算法、构建成本量级、recall floor、增量更新策略

**与 #1、#6 的分工**：
- #1 锁定 16 节点的**物理排布**（hardware allocation）
- 本 query 锁定**逻辑架构选型**——"单一图 vs 路由+局部"（topology choice）
- #6 跨规模看质变点；本 query 锁定 giga/tera 一档的架构二选一

**Cover area**: index topology, routing layer, hierarchical clustering, partitioned ANN

---

## 使用说明

- **新增 query**：直接在本文件追加一节，确保 `query-key` 唯一
- **删除 query**：直接删该节即可（已有的快照保留作历史）
- **暂停整个机制**：删除本文件——schema 里的 hook 失效，ingest 不再触发快照
