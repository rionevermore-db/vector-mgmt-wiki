---
title: DistributedANN-Cited 4 Frontier Papers（CXL-ANNS / LM-DiskANN / AiSAQ / Graph Partitioning）
type: concept
sources: [jang-2023-cxl-anns, pan-2023-lm-diskann, tatsuno-2024-aisaq, gottesbueren-2024-graph-partitioning, adams-2025-distributedann]
related: [../systems/distributedann.md, ../systems/diskann.md, ../systems/spann.md, ../systems/spfresh.md, ../systems/freshdiskann.md, ../systems/turbopuffer.md, ../systems/chroma.md, ../systems/cxl-anns.md, vamana.md, hnsw.md, product-quantization.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md]
created: 2026-05-12
updated: 2026-05-19 (CXL-ANNS deepen → 独立 systems/cxl-anns.md, 本段收窄为指针)
---

# DistributedANN-Cited 4 Frontier Papers

**TL;DR**: 4 篇与 [DistributedANN](../systems/distributedann.md) (Bing 2025 ICML Workshop) 同时期或被 paper 显式 cite 的 ANN frontier 研究, 共同**填 wiki citation chain**——之前 DistributedANN page 引用这些 paper 但 source 未 ingest. 4 paper 涵盖**4 个不同 hardware / 系统级 ANN 创新维度**: (1) **CXL-ANNS** (Jang 2023 USENIX ATC) — CXL memory disaggregation, 软硬件协同, **111× higher QPS / 93.3% lower latency vs SOTA**; (2) **LM-DiskANN** (Pan 2023 IEEE BigData) — low memory footprint disk-native dynamic graph (~10 MB DRAM for billion-scale, **storage 换 memory**, trade-off opposite to standard DiskANN); (3) **AiSAQ** (Tatsuno 2024 Kioxia) — all-in-storage ANN with PQ, **DRAM-free** (~10 MB memory at billion-scale), sub-millisecond multi-dataset switching for RAG; (4) **Graph Partitioning for NN Search** (Gottesbüren 2024 PVLDB 2025 Google Research) — neighborhood-preserving sharding + modular routing for large-scale ANNS, DistributedANN §4.4 显式 competitor. **对 wiki 内 vector DBs 的核心价值**: (a) **citation chain 完整闭合**——之前 wiki DistributedANN page 引用 4 paper 但 source 缺, 现完整 ingest; (b) **Hardware tier landscape 完整化**——之前 wiki disk philosophy 8 类涵盖 software-side, 4 paper 添加**hardware-side tier** (CXL disaggregated memory / SSD-native DRAM-free / etc.); (c) **3 类 ANN search hardware deployment paradigm** identified: DRAM-resident (HNSW classic) → DRAM + SSD hybrid (DiskANN/SPANN/Starling) → **DRAM-free / Memory disaggregated** (AiSAQ / CXL-ANNS) → **distributed shared** (DistributedANN); (d) **Graph partitioning for ANNS** 是 DistributedANN single-graph approach 直接对比 paper, fair benchmark methodology 重要参考.

## 4 paper 各自概述

### 1. CXL-ANNS (Jang 2023 USENIX ATC, KAIST + Panmnesia)

> **已 deepen 为独立 system page**：[systems/cxl-anns.md](../systems/cxl-anns.md)（2026-05-19 一手论文 deepen-ingest）。本段保留为指针，详细技术深度（CXL 三 sub-protocol / Type-3 HDM、4 个机制、FPGA+gem5 原型、分层 eval、§7 anti-GPU 反论）见该 page。

**一句话**：把全量 billion-point 数据集放进 **CXL 解耦内存池**（不压缩、不下放 SSD）→ billion-scale **无精度损失**，用 relationship-aware caching + ANNS-aware prefetch + EP-side 近数据距离计算 + 依赖松弛把 CXL far-memory（naive 比 oracle 慢 3.9×）藏掉，最终 **111.1× higher QPS / 93.3% lower latency vs SOTA**（PQ/DiskANN/HM-ANN），且比无限 DRAM oracle 还快 3.8× throughput。wiki 内**首个 hardware-level memory disaggregation ANN 系统**；§7 显式论证 GPU 对 ANN 距离计算不经济——对 [CAGRA](../systems/cagra.md) GPU-native 路线的哲学反论。[jang-2023-cxl-anns]

### 2. LM-DiskANN (Pan 2023 IEEE BigData, Univ. Nebraska-Lincoln)

[per pan-2023-lm-diskann]

**Problem**: 标准 DiskANN 需要 PQ vectors 全在 DRAM (1B vectors × 64 byte = 64 GB DRAM). 仍 DRAM-bound.

**Approach**: 每 node 存 **complete routing info** (PQ-compressed copies of all neighbors) immediately following node 自身 vector. Reading 一 node = 同时获取所有 neighbor PQ codes.
- DRAM footprint: ~10 MB 仅 entry point + 少 metadata
- Disk: store more (vector + neighbor PQ codes interleaved)
- **Trade DRAM for disk**: opposite of standard DiskANN philosophy

**Results**: Similar recall-latency curve vs SOTA graph-based ANN, **much less memory**. Production case: libSQL (Turso) 集成 LM-DiskANN.

→ wiki 内**首个明示 storage-for-memory trade-off 的 DiskANN variant**.

### 3. AiSAQ (Tatsuno 2024 arXiv 2404.06004, Kioxia)

[per tatsuno-2024-aisaq]

**Problem**: DiskANN PQ DRAM 仍 scales linearly with corpus size. 多 dataset (RAG over多 corpus) 时 DRAM 切换 cost 高.

**Approach**: **All-in-storage**——PQ vectors 也放 SSD (不在 DRAM). 仅 metadata (entry point, codebook header) 在 DRAM.
- DRAM footprint: ~10 MB at billion-scale
- Negligible index load time before query (vs DiskANN 须 load PQ to DRAM)
- **Sub-millisecond multi-dataset switching** for RAG workload (switch between billion-scale corpora)

**Production**: Kioxia open-source (github.com/kioxia-jp/aisaq-diskann).

→ DistributedANN §4 cited as alternative DRAM-free path.

### 4. Graph Partitioning for ANNS (Gottesbüren 2024 PVLDB 2025, Google Research)

[per gottesbueren-2024-graph-partitioning]

**Problem**: Large-scale ANNS 分 partitions (类 SPANN), prior routing methods tightly coupled to specific partitioning method.

**Approach**: 
1. **Neighborhood-preserving sharding**: partition input points 使 nearest neighbors of any point in few shards
2. **Modular routing**: 任何 partitioning method 都可与 routing algorithm 解耦
3. 强 theoretical guarantee on routing performance

**Results**: Competitive with single-graph approach (e.g., DistributedANN) on quality, with **lower per-shard search cost**.

→ DistributedANN paper §4.4 显式列 as alternative architecture; "lower multi-hop network overhead but DistributedANN 因 single-graph quality 更高 production 选 single-graph"——但 Gottesbüren approach 在 latency-critical case **更优 candidate** (per DistributedANN §4.4 admission).

## 4 paper 共同主题: ANN 系统级创新 axis

[per 4 paper + wiki tier-shift / disk philosophy]

| Paper | 主题 axis | Production impact |
|---|---|---|
| **CXL-ANNS** | Hardware: CXL memory disaggregation | DRAM expansion via interconnect, multi-device pool |
| **LM-DiskANN** | Software: store routing info per node | DRAM 10 MB at billion-scale, libSQL adoption |
| **AiSAQ** | Software: all-in-storage with PQ | DRAM-free + multi-dataset switching |
| **Gottesbüren** | Software: graph partitioning + routing decoupling | Modular sharding alternative to single-graph |

→ 4 paper 不是单一突破, 而是**4 个独立 axis 的 ANN system-level frontier** explorations 同时发生.

## DistributedANN 与 4 paper 关系

[per adams-2025-distributedann §4.3-4.4]

- **DistributedANN vs CXL-ANNS**: 都解决 memory bottleneck, 但路径不同. DistributedANN distributed graph + KV store; CXL-ANNS CXL memory disaggregation 单机内
- **DistributedANN vs LM-DiskANN / AiSAQ**: 都减 DRAM, 但 DistributedANN 是 distributed scale; LM-DiskANN / AiSAQ 是 single-node 路径
- **DistributedANN vs Gottesbüren partitioning**: DistributedANN single-graph distributed; Gottesbüren 是改进 partitioning. Paper §4.4: "for our scenario, DistributedANN scalability outweighs partition latency advantages" — 但 latency-critical case 仍是 Gottesbüren approach

## Open Questions

- **CXL-ANNS vs DistributedANN scale ceiling**: CXL 是 single-cluster memory pool, DistributedANN 是 distributed graph; 实际产线选哪种 unclear
- **LM-DiskANN production case**: libSQL (Turso) 集成是开始, 其他 vendor adoption 不公开
- **AiSAQ vs Turbopuffer / Chroma Cloud SPANN object-storage path**: 都 SSD-resident DRAM-light path, 实测 head-to-head 不公开
- **Gottesbüren partitioning + neural sparse / multimodal**: 论文 dense ANN only, 不涉及 sparse / late-interaction
- **4 paper 在 vendor production 实际 deployment**: 都是研究 paper, vendor 引用程度不公开
- **CXL hardware availability**: CXL 2.0+ memory disaggregation hardware market 成熟度不公开

Cited by: 待 query 引用
