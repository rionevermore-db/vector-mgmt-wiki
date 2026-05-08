---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: guo-2022-manu
wiki-pages-total: 39
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md, systems/milvus.md, concepts/delta-consistency.md, concepts/manu-ssd-hierarchical-kmeans.md, benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md]
cited-count: 7
---

# Post-snapshot (guo-2022-manu): scale-tier-shifts

## TL;DR (delta from pinecone-docs post)

**新增第 7 个质变点：strong consistency → tunable delta consistency**。Manu 论文形式化的 delta consistency 模型把"读必看到最新写"重新表述为"τ-bounded staleness"——任何规模档下都可触发的工程哲学切换。**与第 4（DBMS 形态）+ 第 5（in-place update）+ 第 6（adaptive index）的质变同代**——cloud-native vector DBMS 设计的 4 个关键维度。

## Answer

### 七个质变点（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| **7（NEW）** | **strong/eventual binary → tunable delta τ** | **consistency 模型** | **vector DB 应用的多样化容忍度** |

### 第 7 质变的工程含义（NEW）

[per concepts/delta-consistency.md "提出背景"]

之前所有 wiki ingest 的系统**隐式选择**一种 consistency 模型：
- Faiss / Milvus 1.x：static index → strong consistency（query 总看到最新已 apply 的写）
- DiskANN / SPANN：static + 周期 rebuild → 实际是 eventual（rebuild 期内看到旧 view）
- Milvus 2.x LSM segment：snapshot isolation 在 segment 级
- Pinecone：docs 不深入 consistency，best-effort
- SPFresh：tombstone 标记 + version map → eventual + read-your-writes

**Manu (VLDB 2022) 显式形式化**：
- User 指定 τ
- τ=0 → strong；τ=∞ → eventual；中间 → bounded staleness
- "First to support delta consistency in a vector database"

→ consistency 从**隐式系统选择** 升级为**显式用户参数**——质变。

### 第 4-7 质变的同代浪潮（NEW）

[per topics/disk-vs-memory-ann.md "v2.6.x 把 WAL 也搬到 object storage" + concepts/delta-consistency.md]

四个质变点共属 **2021-2024 cloud-native vector DBMS 设计浪潮**：

| 质变 | 代表论文 | 时间 |
|---|---|---|
| 4: DBMS 形态 | wang-2021-milvus（SIGMOD）| 2021 |
| 5: in-place update | xu-2023-spfresh（SOSP）| 2023 |
| 6: adaptive indexing | pinecone-docs / Pinecone serverless | 2024 |
| **7: tunable consistency** | **guo-2022-manu（VLDB）** | **2022** |

→ 这四个维度独立但相关——共同把 vector DB 从"算法库"重塑为"DBMS"。

### Manu 实测 elasticity 的质变性（NEW）

[per benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md "Elasticity Test"]

Manu §5 的 24h e-commerce workload 是 wiki 内**首次**测了"动态 elasticity"——之前所有 benchmark 都是固定资源 + 固定 workload。

→ "elasticity"从**口号** 变成**实证**的质变：可量化（latency thresholds 触发 query node 数翻倍 / 减半）+ 可重现（K8s pod scaling）。这是 cloud-native 设计的关键收益。

### 与之前 ingest 的演进

| | pinecone-docs post | **guo-2022-manu post (NEW)** |
|---|---|---|
| 质变点数 | 6 | **7** |
| Consistency 维度 | 未显式 | **显式 delta consistency τ** |
| Elasticity 实证 | n/a | **24h 实测** |
| Cloud-native 浪潮代表论文数 | 3-4 | **+ Manu 完整 4 论文集** |

### 不算质变（参数微调）

- 同 delta τ 内的细微调（10s vs 30s）
- Time-tick 间隔 200ms vs 800ms（影响 latency 但非语义质变）
- LSH replication 4× vs 8×

### 已知盲区

- **Token-level / 多字段 delta consistency**：late interaction 检索每文档多 token vectors，是否每 vector 独立 τ？paper 未涉及
- **跨 region delta**：跨云 region τ 的实际 staleness 上限
- **2026 SIGMOD 跨 model 整合**：embedding lifecycle 是另一种"事件型质变"

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/delta-consistency.md](../../concepts/delta-consistency.md)
- [concepts/manu-ssd-hierarchical-kmeans.md](../../concepts/manu-ssd-hierarchical-kmeans.md)
- [benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md](../../benchmarks/manu-vs-elasticsearch-vearch-vald-vespa.md)
