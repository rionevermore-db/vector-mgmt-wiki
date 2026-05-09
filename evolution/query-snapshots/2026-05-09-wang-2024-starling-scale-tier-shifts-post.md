---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, concepts/block-shuffling.md, topics/disk-vs-memory-ann.md, topics/index-selection.md]
cited-count: 4
---

# Post-snapshot (wang-2024-starling): scale-tier-shifts

## TL;DR (delta from gao-2024-rabitq post)

**Starling 不新增第 12 质变维度——而是把第 4 维度（library → DBMS）细化出 sub-dimension**：原 wiki 内 "DBMS" 抽象隐含 "single-server 大磁盘" 假设（DiskANN/SPANN 都假设 ≥64GB RAM + 几 TB SSD）；Starling 揭示 **vector DBMS 工程现实是"per-machine 多 segment + 每 segment ≤2GB RAM + ≤10GB disk"**——这对 disk-resident 索引的设计约束完全不同。**所以第 4 维度细化为 4a（single-server DBMS）vs 4b（segment-level DBMS）**。这影响维度 2/3（RAM/SSD 触顶）的实际形态——不是"单机内存触顶"，而是"per-segment 内存预算 ~2GB"。

## Answer

### 十一个质变点（updated with sub-dimension）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B（per-machine 视角；per-segment 是 33M）|
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+（同上）|
| **4** | **library → DBMS（cloud-native）** | 工程形态 | dynamic + 分布式 + filter |
| **4a（NEW sub）** | **single-server DBMS（DiskANN-style budget）** | 部署假设 | single-server 大磁盘环境（如 Bing） |
| **4b（NEW sub）** | **segment-level DBMS（Starling 形态）** | 部署假设 | **vector DBMS 工程主流（Milvus / Zilliz / Manu）** |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化容忍度 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a (OLAP) / 8b (OLTP) 子分化 |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity 触发 |
| 10 | filter-aware build → predicate-agnostic build | filter cardinality | >1000 filters / 任意 operator |
| 11 | TopK interface → Iterator + RM | query 复杂度 | multi-column / range / Join 出现 |
| 11b | Biased PQ → Unbiased + sharp error bound | quantizer 自身的理论保证 | 同样攻击 K' 问题，但 quantizer 层 |

### 第 4a vs 4b sub-dimension 的特征（NEW）

[per topics/disk-vs-memory-ann.md "关键洞见 4: vector DBMS segment 模型 ≠ single-server"]

| | **4a Single-server DBMS** | **4b Segment-level DBMS** |
|---|---|---|
| 代表系统 | DiskANN / SPANN（single-machine deployment） | Milvus / Zilliz / Manu / Starling |
| Per-instance budget | 64 GB RAM + 几 TB SSD | **~2 GB RAM + ~10 GB disk** |
| 主要部署场景 | 公司内部大磁盘服务器 | cloud-native vector DBMS |
| Fault tolerance 单元 | 整 server replica | **per-segment replica** |
| Scale 路径 | 单 server 大索引 + cross-server replicate | **多 server × 多 segment per server** |
| 1B SIFT 实证 | DiskANN 64GB + 1.1TB peak build | **Starling 31 segments × 32GB** |
| Build time 1B | 5+ 天 | per-segment 1200s × parallel ≈ 2 小时（16 nodes） |
| 工业 default | Bing / 自建大磁盘 | **现代 vector DBMS** |

→ **4b 是 vector DBMS 工程主流**——4a 仅在特定大磁盘环境（Bing-like）适用。

### 维度 2/3 在 4b 形态下的实际触发（NEW）

[per topics/disk-vs-memory-ann.md + Starling §6.9]

| 维度 2 (RAM 触顶) | 4a 视角 | **4b 视角（NEW）** |
|---|---|---|
| 触发规模 | 1B vector | **per-segment 33M vector** |
| 解决方案 | DiskANN PQ | **Starling block shuffling + nav graph** |
| Per-machine 总容量 | ~1B vec/node | **~10B vec/node（30 segments × 33M）** |

| 维度 3 (SSD 触顶) | 4a 视角 | **4b 视角** |
|---|---|---|
| 触发规模 | 几百亿 | **per-segment 33M × 25GB raw** |
| 解决方案 | SPANN bin-packing | **per-segment ≤10GB disk + 多 segment per node** |

→ 维度 2/3 的"触顶"含义在 4b 下不同——不是单机内存/磁盘上限，而是 per-segment budget。

### 与之前 ingest 的累积演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| 质变维度数 | 11（with 11b sub） | **11（with 11b + 4a/4b sub-dimensions）** |
| 第 4 维度 | 单一 "library → DBMS" | **细化为 4a single-server vs 4b segment-level** |
| Per-machine RAM 利用 | RaBitQ 推到 ~10B/node | **Starling 多 segment 共享 1TB → 多 segment 路径** |
| Wiki 内 segment-level 实证 | n/a | **首次明确（Starling）** |

### 不算质变（参数微调）

- Starling block size 4KB vs 8KB vs 16KB
- BNF iteration count β = 8
- Block pruning σ = 0.3
- Memory navigation graph sample ratio (<10%)

### 已知盲区

- **维度 4a vs 4b production A/B 实证**：Starling 学术 only；Milvus 当前 release 仍用 DiskANN（非 Starling）
- **维度 4b + scale 100B**：BIGANN 1B 实证 OK，更大未实证
- **维度 4b + filter / multi-vector**：完全空白
- **维度 11 + 4b 叠加（VBASE engine + Starling segment）**：完全空白
- **维度 11b + 4b 叠加（RaBitQ + Starling segment）**：完全空白
- **维度 12+**：未来 ingest 是否暴露更多维度

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [concepts/block-shuffling.md](../../concepts/block-shuffling.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/index-selection.md](../../topics/index-selection.md)
