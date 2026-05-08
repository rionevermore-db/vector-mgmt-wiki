---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md, systems/analyticdb-v.md, concepts/vgpq.md, systems/milvus.md]
cited-count: 6
---

# Post-snapshot (wei-2020-analyticdb-v): scale-tier-shifts

## TL;DR (delta from guo-2022-manu post)

**新增第 8 个质变维度：vector-first DBMS → OLAP-extended-vector**。ADBV 走的是"OLAP 加 vector"路径——与 Milvus / Pinecone / Faiss "vector-first 系统加 SQL" 路径**正交**。这是规模轴之外的**架构起点选择维度**——可在任何规模下触发。

## Answer

### 八个质变点（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化容忍度 |
| **8（NEW）** | **vector-first DBMS → OLAP-extended-vector** | **架构起点** | **OLAP workload 多 / SQL 用户基础大 / 既有 OLAP 投资重** |

### 第 8 质变的起点对立（NEW）

[per systems/analyticdb-v.md "与 wiki 现有系统的定位差异"]

| 起点 | 代表 | 对待 SQL 的态度 | 对待 OLAP query 的态度 |
|---|---|---|---|
| ANN library | Faiss | n/a | 用户胶合 |
| 算法系统 | DiskANN/SPANN/SPFresh | n/a | 不直接支持 |
| **Vector-first DBMS** | **Milvus / Pinecone** | "schema 模仿 SQL 但 API 自己定" | 部分支持（filter / hybrid） |
| **OLAP-extended-vector** | **AnalyticDB-V / Vespa** | "SQL native，vector 是 first-class field" | **完整 OLAP（join / aggregation / window）+ vector** |

→ 这两条路径**起点不同**：
- Vector-first：从 ANN 算法出发；为大量 vector workload 优化；OLAP 作 secondary
- OLAP-extended：从关系 OLAP 出发；为 SQL workload 优化；vector 加进去

**触发条件**：
- 用户主要用 SQL + OLAP（推荐 ADBV 路径）
- 用户主要用 vector search（推荐 vector-first 路径）
- 混合（depending on workload mix）

### ADBV 的 13B production 给质变 2-3 的延展（NEW）

[per benchmarks/analyticdb-v-vs-twostep.md "结果 7"]

ADBV Smart City production 13B records / 30 TB / 70 节点：
- 单节点 ~185M records——已经过了"质变 2"（单机 RAM 触顶）
- 走"分布式 + 分区 + Pangu"——走过"质变 3"（SSD 触顶 + 分布式存储）

→ 13B 这个尺度 wiki 内已 ingest 实测部署：
- Faiss 1.5T (Meta) - 不公开 latency
- ADBV 13B Smart City - "ms 级"
- Bing SPANN 千亿+ - ~1 ms

→ 13B 是 wiki 内**少数有"具体 latency + cost + production case"完整记录**的规模。

### 与之前 ingest 的演进

| | guo-2022-manu post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| 质变点数 | 7 | **8** |
| 架构起点维度 | 隐含（vector-first 一统） | **显式：vector-first vs OLAP-extended 二选一** |
| OLAP 路径 | 未涉及 | **首次显式** |
| Production 大规模实测 | Manu 100M / Bing SPANN 千亿+ | **+ ADBV 13B** |

### 不算质变（参数微调）

- ADBV 4 plan 内的超参 σ/β/γ 调
- VGPQ n_clusters / n_subcells 选择
- Streaming/batching merge 频率

### 已知盲区

- **OLAP-extended-vector 在 Pinecone / Milvus 上的可行性**：理论可行（在 vector-first 系统加 SQL 解析器），实际工程深度差距大
- **Vector-first 与 OLAP-extended 路径的最终融合**：是否会汇聚？或永远分支？
- **2026 SIGMOD 跨 model 整合**：embedding lifecycle 是另一种"事件型质变"

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [systems/milvus.md](../../systems/milvus.md)
