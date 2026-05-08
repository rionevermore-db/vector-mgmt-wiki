---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: zhang-2023-vbase
wiki-pages-total: 53
cited-pages: [systems/spann.md, systems/diskann.md, systems/vbase.md, topics/disk-vs-memory-ann.md, concepts/relaxed-monotonicity.md]
cited-count: 5
---

# Post-snapshot (zhang-2023-vbase): memory-vs-disk-large-scale

## TL;DR (delta from patel-2024-acorn post)

**VBASE+SPANN 集成 (§5.4) 是 wiki 内首次实证 in-memory graph (HNSW) + on-disk partition (SPANN) 共用 query engine**——证明 [Relaxed Monotonicity](../../concepts/relaxed-monotonicity.md) 抽象在 partition-based + SSD 索引上仍 hold。**memory-vs-disk landscape 的工程意义**：之前 HNSW（memory）与 SPANN（SSD）必须**通过应用层胶合**才能在同一服务里共存；VBASE 让两者**在同一 query engine 内 first-class 共存**。SSD 层 latency 仍偏高（VBASE+SPANN Q5 99p 519.7 ms vs HNSW in-memory 160.7 ms）但**统一编程模型**是质的进展。

## Answer

### VBASE+SPANN 实测（NEW）

[per systems/vbase.md "VBASE+SPANN" + zhang-2023-vbase Table 8]

Azure Standard_L16s_v3 NVMe 上 VBASE 集成 SPANN：

| Query | Recall | avg latency | 99p |
|---|---|---|---|
| Q1 (TopK) | 0.9911 | 9.4 | 11.6 |
| Q2 (TopK + numeric) | 0.9214 | 10.7 | 44.9 |
| Q5 (multi + numeric) | 0.9757 | 87.4 | **519.7** |
| Q7 (range) | 0.9923 | 17.8 | 283.5 |
| Q8 (Join) | 0.9638 | 87,729.3 | (single param) |

→ 全 8 query 类型可行；99p 高（SSD 随机 IO 放大）但**统一 query engine 实证 + 全 query 类型支持**是新成就。

### 路线对比表（updated with VBASE+SPANN）

| 路线 | 数据驻留 | filter capability | query engine | scale 实证 |
|---|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | ✗ | per-system TopK | 1B OOM |
| 磁盘 + 量化 + SSD re-rank（DiskANN）| DRAM (PQ) + SSD | ✗ | per-system TopK | 1B SIFT |
| 磁盘 + IVF + SSD 全精度（SPANN）| DRAM (centroids) + SSD | ✗ | per-system TopK | 1B+ Bing |
| 磁盘 + Filter-aware Vamana + PQ DRAM（Filtered-DiskANN）| DRAM (PQ + medoid) + SSD | ✓ build-time, LCPS | per-system TopK | 28M DANN |
| 全内存 + Predicate-Agnostic HNSW（ACORN）| DRAM only | ✓ build-time, HCPS unbounded | per-system TopK | 25M LAION |
| **VBASE + HNSW (in-mem)** | **DRAM only** | **filter via SQL WHERE** | **iterator + RM** | 330K Recipe1M |
| **VBASE + SPANN (in-mem + SSD, NEW)** | **DRAM (centroids) + SSD (posting)** | **filter via SQL WHERE** | **iterator + RM** | **330K Recipe1M (单实例 PG)** |

→ VBASE+SPANN 是**唯一已实证"统一 iterator + memory + SSD"的系统**。其他 SSD 系统（DiskANN / SPANN / Filtered-DiskANN）都是 per-system TopK 接口。

### 与之前 ingest 的演进

| | patel-2024-acorn post | **zhang-2023-vbase post (NEW)** |
|---|---|---|
| Memory-only filter | + ACORN HCPS 25M LAION | **+ VBASE-HNSW iterator 330K** |
| Memory + SSD filter | Filtered-DiskANN 28M DANN (LCPS) | **+ VBASE+SPANN unified iterator 330K** |
| Unified query engine 跨 mem + SSD | n/a | **首次实证（VBASE）** |
| HCPS + SSD 路径 | 完全空白 | **不变** |

### "HCPS + SSD" 仍是最大盲区（不变）

| Workload | LCPS | HCPS |
|---|---|---|
| memory-only | FilteredVamana / NHQ / ACORN / **VBASE-HNSW** | ACORN 唯一 |
| memory + SSD | Filtered-DiskANN ✓ / **VBASE+SPANN ✓** | **完全空白** |

→ VBASE 的 unified query engine 抽象**理论上**可以承载 ACORN+SSD 路径（HNSW base + SSD partition + RM iterator），但论文 §5.4 仅实证 SPANN（无 filter-aware build）。HCPS + SSD + Iterator 是 wiki 全新 frontier。

### 已知盲区

- **VBASE+DiskANN（Vamana on SSD）**：未实证（VBASE 集成 SPANN 走 SSD，DiskANN 走 SSD 但未集成）
- **HCPS + SSD via VBASE**：理论可行未实证
- **VBASE billion-scale**：单实例 PG 限制 + 论文止于 330K
- **CXL / 持久内存 + RM iterator**：仍未覆盖
- **Distributed RM iterator across mem + SSD shards**：开放问题

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/vbase.md](../../systems/vbase.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [concepts/relaxed-monotonicity.md](../../concepts/relaxed-monotonicity.md)
