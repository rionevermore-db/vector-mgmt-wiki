---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-09
phase: post
ingest-context: ootomo-2023-cagra
wiki-pages-total: 64
cited-pages: [systems/cagra.md, concepts/cagra-graph.md, topics/gpu-vs-cpu-ann.md, topics/index-selection.md]
cited-count: 4
---

# Post-snapshot (ootomo-2023-cagra): scale-tier-shifts

## TL;DR (delta from singh-2021-freshdiskann post)

**CAGRA 把第 4 质变维度（library → DBMS）的 4a/4b sub-dimensions 拓展到 hardware path 维度——产生新 sub-dimension 4c (GPU memory-resident DBMS)**。之前 wiki 内 vector DBMS 部署假设 **CPU + SSD** (4a single-server / 4b segment-level)；CAGRA 揭示 **GPU memory-resident** 是第三种 deployment path——recall ceiling + single-query latency 上限突破，但 dataset size 上限受 GPU memory 限制 (~100M for 96-d, ~24M for 768-d on A100)。**对 scale tier shift 的影响**：GPU 是"低 dataset / 高 latency budget"的 dominator，不是 large-scale solution——**hardware 选择维度新增**而非取代既有维度。

## Answer

### 十一个质变点 + sub-dimensions（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| **4** | **library → DBMS（cloud-native）** | 工程形态 | dynamic + 分布式 + filter |
| 4a (sub) | single-server CPU+SSD DBMS | 部署假设 | DiskANN/SPANN single-server |
| 4b (sub) | segment-level CPU+SSD DBMS | 部署假设 | Milvus / Manu / Starling |
| **4c (NEW sub)** | **GPU memory-resident DBMS** | 部署假设 | NVIDIA RAPIDS / Milvus GPU_CAGRA |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 5a (sub) | cluster path (SPFresh/LIRE) | memory budget ≤ ~10 GB for 1B | cost-sensitive streaming |
| 5b (sub) | graph path (FreshDiskANN/FreshVamana) | memory budget ~128 GB for 1B | recall-sensitive streaming |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 多样化 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a OLAP / 8b OLTP |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity |
| 10 | filter-aware build → predicate-agnostic build | filter cardinality | >1000 filters / 任意 op |
| 11 | TopK interface → Iterator + RM | query 复杂度 | multi-column / range / Join |
| 11b | Biased PQ → Unbiased + sharp error bound | quantizer 理论保证 | quantizer 层 K' 攻击 |

### 第 4c sub-dimension 的特征（NEW）

[per topics/gpu-vs-cpu-ann.md "工业方案对比" + systems/cagra.md "Scale 边界"]

| | 4a Single-server CPU+SSD | 4b Segment-level CPU+SSD | **4c GPU memory-resident** |
|---|---|---|---|
| 代表系统 | DiskANN / SPANN | Milvus / Manu / Starling | **CAGRA / NVIDIA RAPIDS / Milvus GPU_CAGRA** |
| Per-instance budget | 64 GB RAM + TB SSD | 2 GB RAM + 10 GB disk | **80 GB GPU HBM (A100)** |
| 主要部署场景 | 公司内部大磁盘服务器 | cloud-native vector DBMS | **GPU-equipped 节点 (RAPIDS / NVIDIA AI infrastructure)** |
| 1B SIFT 实证 | DiskANN 64GB CPU+SSD | per-segment OK | **multi-GPU shard needed** |
| Recall ceiling | ~95% | ~95% | **~99%+** |
| Single-query latency | 5-10 ms | 5 ms | **<1 ms (multi-CTA)** |
| Dataset size limit per node | 几 TB | per-segment | **~100M (96-d) / ~24M (768-d) per A100** |
| 工业 default | Bing / 自建大磁盘 | 现代 vector DBMS (Milvus / Zilliz Cloud) | **NVIDIA AI infrastructure / Milvus + RAFT** |

→ **4c 是"低 dataset + 高 recall + 低 latency"的 dominator**——不取代 4a/4b，是 complementary 路径。

### Scale tier vs hardware 选择 matrix（NEW）

[per concepts/cagra-graph.md "vs HNSW/NSG/Vamana"]

| Scale | CPU + RAM only | CPU + SSD | **GPU memory** | Multi-GPU |
|---|---|---|---|---|
| < 10M | HNSW / NSG | overkill | **CAGRA single A100 hot path** | overkill |
| 10M - 100M | HNSW / NSG | DiskANN if memory budget 严 | **CAGRA single A100** | overkill |
| 100M - 1B | RaBitQ + IVF | DiskANN / SPANN / Starling | **CAGRA needs FP16 / multi-GPU** | **CAGRA 4-8 GPU shard** |
| 1B+ | OOM | DiskANN / SPANN / Starling | **needs PQ + multi-GPU + future work** | **未实证** |
| 千亿 | OOM | (c) routing + per-segment | **不竞争** | **完全空白** |

→ GPU 路径在 scale axis 上**适合 sweet spot 10M-100M (96-d) / 1M-25M (768-d)**——超过需 multi-GPU，不及在 CPU 上更经济。

### 与之前 ingest 的累积演进

| | gao-2024-rabitq post | wang-2024-starling post | singh-2021-freshdiskann post | **ootomo-2023-cagra post (NEW)** |
|---|---|---|---|---|
| 质变维度数 | 11 (with 11b) | 11 (with 11b + 4a/4b) | 11 (with 11b + 4a/4b + 5a/5b) | **11 (with 11b + 4a/4b/4c + 5a/5b)** |
| Hardware path 维度 | 同前 | 同前 | 同前 | **+ 4c GPU memory-resident** |
| Recall ceiling 维度 | 同前 | 同前 | 同前 | **+ GPU graph 99%+ ceiling** |
| Single-query latency 维度 | 同前 | 同前 | 同前 | **+ CAGRA multi-CTA <1 ms** |

### 不算质变（参数微调）

- CAGRA team size 4-32 (per dataset)
- CAGRA p (parents per iteration)
- FP16 vs FP32 dataset
- Single-CTA vs Multi-CTA dispatch threshold

### 已知盲区

- **维度 4a vs 4b vs 4c production benchmark**：完全空白
- **维度 4c + scale 100M+**：multi-GPU sharding 仅论文提及未实证
- **维度 4c + filter / multi-vector**：完全空白
- **维度 4c + streaming (FreshCAGRA)**：完全空白
- **维度 11 + 4c (VBASE iterator + GPU)**：理论可行未实证
- **维度 11b + 4c (RaBitQ + GPU)**：完全空白
- **维度 12+**：未来 ingest 是否暴露更多

## Cited Pages

- [systems/cagra.md](../../systems/cagra.md)
- [concepts/cagra-graph.md](../../concepts/cagra-graph.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
- [topics/index-selection.md](../../topics/index-selection.md)
