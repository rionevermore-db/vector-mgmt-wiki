---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-09
phase: post
ingest-context: wang-2024-starling
wiki-pages-total: 58
cited-pages: [systems/starling.md, systems/spann.md, systems/diskann.md, concepts/block-shuffling.md, topics/disk-vs-memory-ann.md]
cited-count: 5
---

# Post-snapshot (wang-2024-starling): memory-vs-disk-large-scale

## TL;DR (delta from gao-2024-rabitq post)

**Starling ingest 暴露 wiki 内一个长期 hidden assumption：DiskANN/SPANN 假设的"single-server 大磁盘"在 vector DBMS 工程现实里不成立**——大多数现代 vector DB（Milvus / Manu）用 **segment 抽象**（~2GB RAM + ~10GB disk per segment）。在 segment 约束下：(a) **SPANN 直接不可行**（每向量 8× 复制 → 33M × 8 = 264M storage 远超 10GB），(b) **DiskANN 高 latency**（OR(G)≈0、94% block 浪费、362 hops search path）。Starling 是 wiki 内**首个明确接受 segment-level 约束作为 design first principle** 的 disk graph framework——ANNS 2× 快于 DiskANN，RS 43.9× 快。**memory-vs-disk landscape 现在有第三层划分**："single-server"（DiskANN/SPANN）vs "segment-level"（Starling）。

## Answer

### 与之前 ingest 的演进

| | gao-2024-rabitq post | **wang-2024-starling post (NEW)** |
|---|---|---|
| 全内存 graph 选项 | 不变 | 不变 |
| Memory + SSD（single-server）| DiskANN / SPANN | **不变（仍是 single-server 路径）** |
| Memory + SSD（**segment-level**） | n/a | **Starling（首次出现）** |
| RaBitQ + SSD 实证 | 完全空白 | 仍空白（RaBitQ + graph 集成开放）|
| Wiki 内最大 segment-level 实证 | n/a | **Starling 33M × 4 datasets + 1B BIGANN** |

### 路线对比表（updated with Starling segment-level 路径）

| 路线 | 数据驻留 | budget 假设 | 量化 | scale 实证 |
|---|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | single-server 大内存 | ✗ | 1B OOM |
| 磁盘 + 量化 + SSD re-rank（DiskANN） | DRAM (PQ) + SSD | **single-server 64GB+ RAM, 几 TB SSD** | PQ | 1B SIFT |
| 磁盘 + IVF + SSD 全精度（SPANN） | DRAM (centroids) + SSD | **single-server 大磁盘 + 跨机 replicate** | ✗ | 1B+ Bing |
| 磁盘 + Filter-aware Vamana + PQ DRAM（Filtered-DiskANN） | DRAM + SSD | single-server | PQ | 28M DANN |
| 全内存 + Predicate-Agnostic HNSW（ACORN） | DRAM only | single-server | ✗ | 25M LAION |
| VBASE + HNSW (in-mem) | DRAM only | single-instance PG | ✗ | 330K Recipe1M |
| VBASE + SPANN | DRAM + SSD | single-instance PG | ✗ | 330K Recipe1M |
| RaBitQ + IVF (RAM only) | DRAM only | single-server | RaBitQ | 6 dataset, 2.34M |
| **Starling-Vamana / NSG / HNSW (NEW)** | **DRAM (nav graph + PQ) + SSD (reordered graph)** | **segment ≤2GB RAM + ≤10GB disk** | PQ short codes for routing | **33M × 4 datasets + 1B BIGANN via 31 segments** |

→ Starling 是**唯一明确为 vector DBMS segment 设计的 disk graph framework**。其他系统假设 single-server budget。

### Starling 的 segment-level 优化（NEW）

[per systems/starling.md "架构图" + benchmarks/starling-vs-diskann-spann-on-segment.md]

三个核心设计：
1. **[Block shuffling](../../concepts/block-shuffling.md)**（NP-hard）：3 个启发式 BNP/BNF/BNS，OR(G) 从 DiskANN ≈0 提升到 0.34-0.87
2. **In-memory navigation graph**：采样 <10% vector 建 in-memory graph 找 query-aware entry → 搜索路径 ℓ 减半
3. **Block search**：4KB block 内一次性处理所有 vertex + 三个 computation 优化（block pruning σ=0.3 / I/O+computation pipeline / PQ approximate distance）

实测 BIGANN 33M segment：
- ANNS recall=0.95: 5ms (vs DiskANN 10ms, 2× 快)
- RS AP=0.9 QPS: 8690 (vs DiskANN 181, **43.9× 快**)
- Vertex utilization: 0.34 (vs DiskANN 0.06, 5.5× 提升)

### Single-server vs segment-level 路径选择（NEW）

| 决策因素 | Single-server 路径（DiskANN / SPANN） | Segment-level 路径（Starling） |
|---|---|---|
| 部署形态 | 一台机器一份大索引 | 一台机器多 segment 共享 |
| 工业典型 | Bing 大磁盘服务器 | Milvus / Zilliz Cloud |
| Fault tolerance | 跨机 replicate 整个索引 | per-segment replicate（更细粒度） |
| Load balancing | 整索引 dispatch | per-segment dispatch |
| 单机 1B | DiskANN 64 GB RAM + 1.1 TB peak build | Starling 31 segments × 32GB |
| Build time（1B） | 5+ 天 | 31 × per-segment build（并行可能） |

→ **vector DBMS 走 segment-level 是工业现实**——这个观察打破"单 server 大磁盘"作为 default assumption。

### Billion-scale 通过 segmentation（NEW）

[per benchmarks/starling-vs-diskann-spann-on-segment.md "Result 7"]

BIGANN 1B 实证：
- 31 segments × ~32 GB RAM allocation × 10 GB disk per segment
- 2 query nodes total（segments 分摊）
- Starling **>2× DiskANN at recall>0.96**

→ Single-server 1B (DiskANN) vs 31 segments × ~32GB (Starling) 是两种不同 scaling 哲学——后者契合 vector DBMS 工程现实。

### "HCPS + SSD" 仍是最大盲区

| Workload | LCPS（≤1000 filters） | HCPS（10^8+ filters） |
|---|---|---|
| memory-only | FilteredVamana / NHQ / ACORN / VBASE-HNSW / RaBitQ+IVF | ACORN 唯一 |
| memory + SSD（single-server） | Filtered-DiskANN / VBASE+SPANN | **完全空白** |
| memory + SSD（**segment-level**） | **Starling 不支持 filter（NEW gap）** | **完全空白** |

→ Starling 加 filter / multi-vector 集成是新 frontier——论文 §7 提及 future work。

### 已知盲区

- **Starling + filter / multi-vector**：完全空白
- **Starling + GPU**：§8 future work
- **Starling + RaBitQ**：两个论文同年发表未联合实证
- **Starling 多 query node 间协调**：单 query node 内 OK；跨 node coordinator 模式未深入
- **HCPS + segment-level + SSD**：完全空白

## Cited Pages

- [systems/starling.md](../../systems/starling.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [concepts/block-shuffling.md](../../concepts/block-shuffling.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
